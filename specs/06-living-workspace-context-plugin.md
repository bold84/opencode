# Living Workspace Context Plugin (OpenCode)

This document is a detailed implementation plan for a **single OpenCode plugin** that adds “Living Workspace” context management:
- relevance scoring
- 3-tier context fading (full / structural / purpose)
- per-session + per-subagent scoping
- improved compaction prompts
- caching + optional dependency-aware refinements

The plan assumes **no core OpenCode changes** are required (plugin-only), and leverages the existing plugin hooks:
- `experimental.chat.messages.transform`
- `experimental.chat.system.transform`
- `chat.params`
- `tool.execute.before` / `tool.execute.after`
- `experimental.session.compacting`

---

## 1) Goals

### Primary goals
1. **Keep the main LLM on-track** across multi-step work by injecting an always-fresh, relevance-ranked “workspace context pack” each step.
2. **Scale to large repos** by never relying on stuffing whole codebases into the LLM context; instead inject only what’s likely relevant.
3. **Work for subagents automatically** (child sessions) without additional wiring.
4. **Respect OpenCode’s safety model** by default: no silent scanning/reading of arbitrary files unless explicitly enabled.

### High-value features included
- **Relevance scoring** based on actual tool usage + recency + edits.
- **Three-tier fading**:
  - Full content (hot)
  - Structural summaries (warm)
  - Purpose-only (cold)
- **Per-session scoping** + sensible inheritance for subagent sessions.
- **Basic history condensation improvements** by enhancing compaction prompt.

### Refinements included
- Caching (content/summaries/token estimates)
- Lightweight dependency / co-access graph signals
- “Value per token” packing (greedy; optional knapsack later)
- Debug tooling for observability

### Non-goals
- Replace OpenCode’s native compaction pipeline.
- Build a full universal multi-language indexer inside the plugin.
- Guarantee perfect symbol extraction for every language.

---

## 2) Constraints and Integration Points

### What the plugin can reliably do
- **Inject context into every LLM step** via `experimental.chat.messages.transform` (called in the main prompt loop).
- **Adjust system prompt** via `experimental.chat.system.transform`.
- **Track behavior** via tool hooks (`tool.execute.*`) and message hook (`chat.message`).
- **Customize compaction prompts** via `experimental.session.compacting`.

### Important caveat
Some internal “small model” helper calls in OpenCode may not pass through `experimental.chat.messages.transform` (depending on the call site). This does **not** block the core feature set, because:
- The main agent + subagents (the primary loops) do pass through the hook.
- The compaction call is separately hookable.
- System prompt transform can still provide stable “rules” even when per-step injected context is absent.

---

## 3) One-Plugin Architecture

### Single package layout
One npm package (or one `file://` plugin) that exports one plugin entrypoint.

Recommended internal structure (within the single package):
- `src/index.ts` — plugin entrypoint, returns hooks
- `src/state.ts` — in-memory state + caches
- `src/scoring.ts` — relevance scoring
- `src/summarize.ts` — structural/purpose summary generators
- `src/context-pack.ts` — builds the injected context block
- `src/hooks/*.ts` — hook implementations
- `src/tools/*.ts` — optional debug tools

This is still **one plugin** (one published package), just organized internally.

---

## 4) Data Model (“Workspace Memory”)

### Key idea
Maintain a lightweight “workspace memory” per **project/worktree**, with a per-session view:
- **Global memory** (shared across sessions in same worktree): learned file purpose, cached structural summaries, co-access statistics.
- **Session memory** (per `sessionID`): recency, what’s being touched right now, current task anchors.

### Core entities

**FileRecord** (keyed by absolute or worktree-relative path)
- identity: `path`, `language`, `sizeBytes`
- access signals:
  - `lastReadAt`, `readCount`
  - `lastGrepAt`, `grepHitCount`
  - `lastListedAt`
- edit signals:
  - `lastEditAt`, `editCount`
  - `lastPatchAt`, `patchCount`
- conversational signals:
  - `lastMentionAt`, `mentionCount`
- derived:
  - `relevanceScore` (computed)
  - `tier` (full/structural/purpose)

**SessionRecord**
- `sessionID`
- optional `parentSessionID` (detected lazily)
- `lastModel` (provider/model + context limit if available)
- recent “anchors”:
  - `recentFilesTouched[]`
  - `recentSymbolsQueried[]`
  - `recentUserMentions[]`

**Caches**
- `contentCache`: file -> last fetched content (bounded + TTL)
- `structCache`: file -> structural summary (bounded + invalidation by mtime/hash)
- `purposeCache`: file -> purpose string
- `tokenEstimateCache`: string -> estimated tokens

---

## 5) Signal Collection (Hooks)

### Hook: `tool.execute.after`
Use tool results to update file activity without additional IO.

Track at minimum:
- `read`: update `lastReadAt`, `readCount`, add file to session anchors
- `grep`: update `lastGrepAt`, add matched file paths to anchors, increment `grepHitCount`
- `glob` / `ls`: update discovery signals (light weight)
- `edit` / `write` / `patch`: update `lastEditAt`, `editCount`, add to anchors
- `bash`: optionally parse output for file paths (off by default; noisy)

Why this works well in OpenCode:
- It reflects what the agent actually looked at.
- It stays aligned with the permission model.

### Hook: `chat.message`
Use user text to:
- detect explicit file mentions (e.g., `src/foo.ts`, backticked paths)
- detect explicit symbols (e.g., `FooBar`, `handleSubmit`)
- update `lastMentionAt` / `mentionCount` and anchors

### Hook: `chat.params`
Capture model/provider context that is useful for packing:
- model ID/provider ID
- (if available in the model metadata) context window size

Store this on the session record so the packer can pick a safe budget.

---

## 6) Relevance Scoring

### Design goals
- Cheap to compute
- Deterministic and explainable
- Driven by real user/agent actions

### Recommended scoring function (MVP + refinements)
Score each file `f` per session:

**A. Recency of access (strong)**
- Exponential decay based on `now - lastAccessAt` (read/grep/edit/mention all count as access)

**B. Edit emphasis (very strong)**
- Recently edited files get a large boost; they are usually central.

**C. Frequency (medium)**
- Files frequently accessed in the session rise.

**D. Task alignment (lightweight, optional)**
- If user explicitly mentions a path/symbol, boost matching files.

**E. Dependency / co-access proximity (refinement)**
- If file A and B are often accessed in the same time window, boost them together.
- If import statements are detected in a hot file and reference another file, boost that target.

### Weight tuning
Expose weights in plugin config with good defaults, e.g.
- editRecencyWeight > readRecencyWeight
- mentionWeight modest (avoids spurious boosts)

---

## 7) Three-Tier Context Fading

### Tier definitions

**Tier 1: Full (hot)**
- Include full content or (preferably) the most relevant slice:
  - If the file was read recently: reuse last read slice
  - If the file was edited: include a diff/patch-like snippet if available
  - Otherwise: include head + key regions only (bounded)

**Tier 2: Structural (warm)**
- Include a compact “shape of file” summary:
  - classes/functions signatures
  - exported symbols
  - important constants
  - imports (for dependency hints)

**Tier 3: Purpose (cold)**
- Include path + 1-line purpose + last touched time.
- Purpose can be derived from:
  - filename heuristics (`router.ts` → “routing”)
  - top-of-file docstring/comment
  - prior cached purpose

### How to implement structural summaries (practical approach)
Because OpenCode’s public HTTP API doesn’t expose document-level symbols directly:
- **Base approach (reliable):** parse using lightweight, language-aware regex for common languages (TS/JS/Python/Go).
- **Refinement:** optional Tree-sitter/WebTreeSitter inside the plugin for higher fidelity.
  - Keep it optional and lazy-loaded by language.

Either way stays within a single plugin package.

---

## 8) Context Packing and Injection

### Where to inject
Use `experimental.chat.messages.transform` to append a synthetic context block to the **last user message**.

Rationale:
- Doesn’t permanently pollute system prompt.
- Keeps OpenCode’s system prompt caching behavior stable.

### Idempotence / de-duplication
Every step, before injecting, remove any prior injected blocks by:
- tagging injected text with a distinctive header marker, e.g. `<!-- LW_CONTEXT -->`
- or using part metadata (if preserved through hook types)

### Token budgeting
Implement a budget policy:
- Determine an overall budget for injected context, e.g. 10–25% of model context.
- If model context limit is unknown, default to a conservative budget (e.g. 8k tokens).

Packing algorithm (recommended):
1. Build candidate list from anchors + top scored files.
2. Assign each candidate a tier and estimate cost.
3. Sort by “value per token” (relevance / estimatedTokens).
4. Add until budget exhausted.

Knapsack DP is optional; greedy is usually sufficient.

### Output format
Inject a single block:
- `Workspace Summary` (very short)
- `Hot Files` (full)
- `Warm Files` (structural)
- `Cold Files` (purpose)
- `Open Threads` (if available from recent steps)

Use consistent delimiters so the LLM can reliably parse:
- `<workspace_context>` … `</workspace_context>`

---

## 9) Subagent Support (Child Sessions)

OpenCode creates subagents as child sessions via the `task` tool.

### Strategy
- Treat each `sessionID` independently for scoring.
- Inherit global caches across sessions in the same worktree.
- Optionally seed child session anchors from parent:
  - When a new session is first seen, lazily fetch session info to detect `parentID`.
  - Copy a small set of top anchors (top N file paths) into child session anchors.

This provides subagents with immediate context without over-sharing everything.

---

## 10) Compaction Improvements

### Hook: `experimental.session.compacting`
Replace or augment the compaction prompt so summaries preserve:
- current objective
- decisions made and why
- files modified (paths)
- errors encountered and resolutions
- remaining TODOs / next steps

### Goal
When OpenCode compacts history, the resulting prompt should retain:
- *what matters for continuing work*
- while dropping verbose tool outputs

---

## 11) Caching and Performance

### Caches (refinement)
- Structural summary cache keyed by `(path, mtime/hash)`
- Content slice cache keyed by `(path, lastReadAt)`
- Token estimate cache keyed by string hash

### Bounds
- Use LRU (by count) + TTL (by time)
- Hard cap memory usage by limiting stored content slices

### Work avoidance
- Only re-score files touched recently or in candidate sets.
- Only re-generate summaries when file changes.

---

## 12) Optional Debug Tools (Still One Plugin)

Add plugin-defined tools (via plugin `tool` hook) for observability:
- `lw.status` — show top ranked files + scores + tiers
- `lw.config` — print effective config
- `lw.explain <path>` — explain why a file is ranked

These tools help iterate on tuning and prove behavior.

---

## 13) Configuration

Expose config under a single namespace, e.g. `livingWorkspace`.

Suggested config options:
- `enabled: boolean`
- `maxInjectedTokens: number`
- `tierThresholds: { full: number; structural: number }`
- `maxFullFiles: number`
- `maxStructuralFiles: number`
- `maxPurposeFiles: number`
- weights for scoring (recency/edit/frequency/mentions/deps)
- `allowImplicitReads: boolean` (default false)
- `debug: boolean`

---

## 14) Testing Plan

### Unit tests (plugin repo)
- scoring: recency decay, weighting, deterministic ordering
- tier selection: thresholds and caps
- packing: respects token budget, stable output shape
- de-duplication: injected blocks don’t accumulate

### Integration tests (manual + scripted)
- Run OpenCode with plugin enabled and:
  - read/edit multiple files → ensure hot files rise
  - spawn subagent task → verify child session gets scoped context
  - force compaction → ensure improved summary prompt is used

### Acceptance criteria
- Context block appears on every normal LLM step.
- Block stays within configured budget.
- Subagent sessions show different (scoped) context vs main.
- No noticeable latency spikes for typical projects.

---

## 15) Implementation Milestones

### Milestone 1 — Skeleton + hooks (1–2 days)
- Plugin package scaffolding
- Implement hooks: `tool.execute.after`, `chat.message`, `chat.params`
- In-memory session/file records

### Milestone 2 — Context injection MVP (2–4 days)
- Candidate selection from anchors
- Relevance scoring v1
- Inject `<workspace_context>` into last user message
- De-duplication + budgeting

### Milestone 3 — Three-tier fading (3–6 days)
- Purpose summaries
- Structural summaries (regex-based)
- Full-content policy (prefer slices/diffs)

### Milestone 4 — Compaction improvements (1–2 days)
- Replace/augment compaction prompt via hook

### Milestone 5 — Refinements (3–7 days)
- caching + LRU/TTL
- dependency/co-access signals
- debug tools
- tuning + docs

---

## 16) Risks and Mitigations

- **Risk: plugin reads too much / breaks safety expectations**
  - Mitigation: default to no implicit reads; build from observed tool usage.

- **Risk: context block grows and causes overflow**
  - Mitigation: strict token budget + tier caps + greedy value-per-token packing.

- **Risk: structural summaries inaccurate across languages**
  - Mitigation: start with best-effort regex for common languages; optionally enable Tree-sitter.

- **Risk: noisy file mention parsing**
  - Mitigation: conservative path detection; require explicit-looking paths.

---

## 17) Deliverable

A single OpenCode plugin package (one installable entry in `config.plugin`) that:
- continuously maintains session-aware workspace memory
- injects a relevance-ranked context pack each LLM step
- improves compaction summaries
- supports subagents automatically

