# Lazy MCP Loading

## TL;DR

> **Summary**: Add a per-server MCP `mode` (`eager | lazy | disabled`) with backward-compatible config parsing, a searchable MCP meta-tool, and session-scoped lazy tool activation while keeping all real MCP execution on the existing `session/prompt.ts` wrapper path.
> **Estimated Effort**: Large

## Context

### Original Request

Create a detailed implementation plan for lazy MCP loading in this repository using the recommended architecture: per-server MCP mode with eager + lazy + disabled behavior; v1 scoped to MCP tools only; lazy tools discoverable through a compact prompt index plus a meta-tool; no direct MCP execution from the meta-tool; and no auto-thresholds, unload/TTL, or prompt/resource deferral in v1.

### Key Findings

- MCP server init is currently eager inside `packages/opencode/src/mcp/index.ts` via `InstanceState.make`, which connects all configured non-disabled servers and caches tool defs in `s.defs`.
- Real MCP tool execution is wrapped in `packages/opencode/src/session/prompt.ts`; that wrapper already handles permissions, plugin hooks, output truncation, attachments/resources, and metadata. That path should remain the only execution path.
- MCP prompts are surfaced as commands in `packages/opencode/src/command/index.ts`; because v1 is MCP-tools-only, prompt behavior should stay as-is unless implementation proves lazy mode leaks unwanted prompt semantics.
- Current config schema in `packages/opencode/src/config/config.ts` only distinguishes enabled vs disabled. There is no `mode`, `mcp_lazy`, or `mcp_search` in this checkout.
- Existing MCP lifecycle tests in `packages/opencode/test/mcp/lifecycle.test.ts` already mock MCP transports/clients and are the best place to cover mode/init/index behavior.
- Existing OAuth tests in `packages/opencode/test/mcp/oauth-auto-connect.test.ts` and `packages/opencode/test/mcp/oauth-browser.test.ts` are the right places to pin remote/OAuth lazy-server edge cases if mode handling changes auth startup behavior.
- `packages/opencode/test/session/prompt-effect.test.ts` already provides a mocked `MCP.Service`, making it the right place to verify session-scoped activation and that loaded MCP tools still execute through the shared wrapper path.
- Existing session tests in `packages/opencode/test/session/session.test.ts` are available if implementation needs to verify cleanup around session teardown beyond prompt-loop tests.
- Earlier review guidance matters: do not duplicate MCP execution/result formatting in the search tool, and search must match tool names/descriptions, not just server names.
- Current architecture naturally supports **lazy exposure** more easily than **true lazy connection**: tool metadata comes from `client.listTools()` after connect, so v1 should explicitly keep connecting lazy servers eagerly at startup while only exposing their tools on demand.

## Objectives

### Core Objective

Implement a backward-compatible lazy MCP tool exposure model where each server can be configured as `eager`, `lazy`, or `disabled`, lazy tools can be searched/described/activated per session, and once activated they run through the exact same MCP execution wrapper used by eager tools today.

### Deliverables

- [ ] Add backward-compatible MCP config support for per-server `mode`.
- [ ] Add MCP catalog/index plumbing for eager vs lazy tool exposure.
- [ ] Add a session-scoped MCP meta-tool for `search`, `describe`, and `load`.
- [ ] Preserve existing MCP execution path and behavior for loaded tools.
- [ ] Add targeted config, MCP, and session tests.
- [ ] Complete a post-implementation security review focused on catalog exposure, input validation, and permission invariants.

### Definition of Done

- [ ] `cd packages/opencode && bun typecheck`
- [ ] `cd packages/opencode && bun test test/config/config.test.ts`
- [ ] `cd packages/opencode && bun test test/mcp/lifecycle.test.ts`
- [ ] `cd packages/opencode && bun test test/session/prompt-effect.test.ts`
- [ ] Lazy servers are not directly exposed as callable tools until loaded, eager servers behave exactly as today, and disabled servers remain excluded.

### Guardrails (Must NOT)

- Must NOT change real MCP tool execution to bypass `packages/opencode/src/session/prompt.ts`.
- Must NOT make the meta-tool directly call arbitrary MCP tools in v1.
- Must NOT conflate `disabled` with `lazy`.
- Must NOT implement true lazy connection in v1.
- Must NOT add auto-threshold heuristics, unload/TTL, or prompt/resource deferral in v1.
- Must NOT run tests from the repo root.

### Security & Guardrails

- [ ] Catalog and prompt index metadata must be minimized to `{ server, toolName, sanitizedToolId, short description, loaded }`; do not expose argument schemas, headers, secrets, OAuth tokens, environment values, or raw connection details in search results.
- [ ] Search and prompt summaries must redact or omit sensitive server metadata by default; descriptions should come from MCP tool defs only.
- [ ] Meta-tool input validation must reject unknown servers, empty/oversized queries, malformed tool selectors, and duplicate load targets safely.
- [ ] Validate `server` and `name` before catalog lookup with a conservative allowlist regex and max-length checks so invalid selectors never trigger deeper search/work.
- [ ] Meta-tool `search` and `load` must not bypass permission checks for eventual execution; loaded tools must still run through the same `ctx.ask(...)` permission path as eager tools.
- [ ] Add fixed caps for search result count, query length, and per-call load batch size to avoid prompt bloat or catalog-scan abuse.
- [ ] Add per-session rate limits for meta-tool `search` and `load` so repeated calls cannot flood catalog scans or activation churn within a single session.
- [ ] Catalog/search responses must be permission-filtered for the requesting session before results are returned.
- [ ] Keep eager startup connection behavior under explicit concurrency and failure guardrails; v1 should preserve or lower current fan-out and avoid retry storms.
- [ ] Include a post-implementation security review covering catalog leakage, startup connection risk, and permission invariants.

## TODOs

- [ ] 1. Normalize MCP config around `mode`
     **What**: Extend both strict schemas in `packages/opencode/src/config/config.ts` — `McpLocal` and `McpRemote` — by adding `mode: z.enum(["eager", "lazy", "disabled"]).optional()` while retaining `enabled: z.boolean().optional()` for backward compatibility. Add a shared normalization helper, used by MCP runtime code, that resolves one final mode with explicit precedence: `mode` wins if present; otherwise `enabled === false` resolves to `disabled`; otherwise `enabled === true` or `undefined` resolves to `eager`. When both `mode` and `enabled` are present and disagree, preserve `mode`, emit one warning-level log, and continue.
     **Files**: `packages/opencode/src/config/config.ts`, `packages/opencode/test/config/config.test.ts`
     **Acceptance**: Config tests in `packages/opencode/test/config/config.test.ts` prove legacy `{ enabled: false }` still disables, omitted `mode/enabled` stays eager, explicit `mode: "lazy"` parses on both local and remote entries, and conflicting `mode` + `enabled` keeps `mode` while logging a warning.

- [ ] 2. Introduce an MCP catalog/index model distinct from exposed tools
     **What**: Refactor MCP state so tool defs remain cached per server, but exposure is filtered by resolved mode. Add internal helpers that return: (a) eager callable tools, (b) lazy tool catalog entries, and (c) server mode metadata. Keep prompts/resources behavior unchanged for v1. The catalog should be derived from cached defs and include only minimized metadata such as `{ server, mode, toolName, toolId, description }`, where `toolId` is the existing sanitized runtime id and `server + toolName` remains the canonical selector to avoid collisions. Do not include schemas, auth state, headers, env, or transport details in the catalog. Plan the catalog API to accept session/user permission context so results can be filtered before the meta-tool returns them.
     **Files**: `packages/opencode/src/mcp/index.ts`, `packages/opencode/test/mcp/lifecycle.test.ts`
     **Acceptance**: MCP tests in `packages/opencode/test/mcp/lifecycle.test.ts` prove eager servers still appear in `MCP.tools()`, lazy servers do not appear there until requested by session logic, disabled servers remain absent, and search data includes tool names/descriptions from lazy servers without leaking connection metadata or permission-hidden entries.

- [ ] 3. Decide and document the v1 lazy-connect tradeoff in code comments and plan notes
     **What**: Make the v1 tradeoff explicit in code comments, tests, and user-facing behavior: this feature is **lazy exposure only**, not true lazy connection. Non-disabled servers, including `mode: "lazy"`, may still connect during `InstanceState.make` so `listTools()` can build the catalog. Add startup/resource guardrails around that choice: preserve existing timeouts, do not increase default concurrency, avoid extra list calls beyond current caching/watch behavior, ensure lazy mode does not reconnect more often than eager mode, and add simple circuit-breaker guidance for v1 so repeatedly failing servers are not retried in a tight loop during one startup lifecycle.
     **Files**: `packages/opencode/src/mcp/index.ts`, `packages/opencode/test/mcp/lifecycle.test.ts`
     **Acceptance**: The implementation makes the tradeoff explicit in code and behavior, startup semantics remain stable, lazy mode does not imply metadata-only startup, and tests verify lazy servers may connect eagerly while staying unexposed as callable tools under bounded startup concurrency/failure behavior.

- [ ] 4. Add MCP service APIs for lazy discovery and selective exposure
     **What**: Extend `MCP.Interface` with catalog-oriented methods instead of teaching the meta-tool to inspect internal state itself. Recommended shape:
  - `catalog(): Effect<Record<string, CatalogEntry>>` or `search(query, opts)` for all lazy-capable entries
  - `tools(input?: { includeLazy?: string[] | Set<string> }): Effect<Record<string, Tool>>` so eager tools are always included and session-loaded lazy tool ids can be merged in
    Keep the returned `Tool` objects backed by the same `convertMcpTool()`/client path used today. Add internal caps to catalog/search helpers (for example a default/max limit) so large MCP catalogs cannot flood prompt assembly or tool responses. Include a permission-filtering step in the API contract so the session/meta-tool layer can request catalog entries already filtered for the current session ruleset.
    **Files**: `packages/opencode/src/mcp/index.ts`, `packages/opencode/test/mcp/lifecycle.test.ts`
    **Acceptance**: Session code can ask MCP for an index without exposing tools, request a merged tool map for the current session’s activated lazy tools, and catalog APIs enforce bounded result sizes plus permission filtering.

- [ ] 5. Add session-scoped lazy activation state
     **What**: Extend `SessionPrompt` instance state with a per-session set of activated lazy MCP selectors, e.g. `Map<SessionID, Set<string>>`, plus lightweight per-session meta-tool rate-limit counters/windows for `search` and `load`. Clean both up with the existing runner lifecycle/finalizers. Activation should be in-memory and session-scoped for v1; do not add DB schema or migration work unless implementation proves persistence is required. Define cleanup guarantees: remove activation state and rate-limit state when the session runner is finalized, when the instance is disposed, and on interrupted/cancelled prompt loops; after process crash/restart the state is intentionally lost.
     **Files**: `packages/opencode/src/session/prompt.ts`, `packages/opencode/test/session/prompt-effect.test.ts`, `packages/opencode/test/session/session.test.ts`
     **Acceptance**: A loaded lazy tool becomes available for later turns in the same session, is not globally activated for other sessions, per-session meta-tool rate limiting is enforced and cleared on runner/instance cleanup, and tests document that crash/restart drops activation state by design.

- [ ] 6. Add the MCP meta-tool without duplicating execution logic
     **What**: Add one meta-tool, preferably named `mcp`, to the tool set assembled in `SessionPrompt`. It should support three v1 actions:
  - `search`: fuzzy/substring search across lazy catalog tool names and descriptions, with optional server filter and limit
  - `describe`: return richer details for one or more matching tools/servers, including server, real MCP tool name, runtime tool id, description, and whether already loaded
  - `load`: mark one or more lazy tools as activated for the current session and return confirmation/instructions that the newly loaded real tool should be called next

  Recommended input schema:
  - `action: "search" | "describe" | "load"`
  - `query?: string` where required for `search`, trimmed, non-empty, and capped to a small maximum length
  - `server?: string` limited to configured server ids, validated before lookup with an allowlist regex and max length, and only allowed as a filter, not as a wildcard pattern
  - `tools?: { server: string, name: string }[]` where both `server` and `name` are required for `load`, both validated before lookup with allowlist regex + max lengths, validated against the catalog, deduplicated, and capped per request
  - `limit?: number` clamped to a small safe max

  Recommended behavior:
  - Search ranking should prioritize exact tool-name matches, then substring/description matches, with server-name matching only as a secondary aid.
  - Apply per-session rate limiting to `search` and `load`; when exceeded, return a compact retry-later error without mutating activation state.
  - Catalog/search/describe responses must be filtered by the requesting session’s effective permission rules before matching results are returned.
  - `search` must reject empty queries after trimming and return bounded results.
  - `describe` should support either validated `{ server, name }` selectors or a bounded query-based lookup, returning compact metadata only.
  - `load` must validate targets against the catalog, be idempotent, and return a stable success result when the tool is already loaded.
  - Unknown or unavailable tools should produce safe user-facing errors without mutating activation state.
  - The response should be compact and model-friendly; do not stream full schemas unless `describe` is asked for, and even then omit sensitive connection data.
  - The tool must never call `client.callTool()`.
    **Files**: `packages/opencode/src/session/prompt.ts`, `packages/opencode/test/session/prompt-effect.test.ts`
    **Acceptance**: Prompt-effect tests in `packages/opencode/test/session/prompt-effect.test.ts` show `mcp` validates `query/server/name` before lookup, enforces caps and per-session rate limits, filters catalog results by permission, loads idempotently, rejects bad selectors safely, and only changes exposure state rather than executing MCP operations.

- [ ] 7. Inject a compact lazy-tool prompt index
     **What**: Add a generated system-prompt section that advertises lazy MCP availability without dumping every schema. It should summarize lazy servers/tools compactly, for example by grouping by server and listing a few representative tool names plus a count, and explicitly instruct the model to use the `mcp` meta-tool to search/describe/load. Keep this section concise to avoid prompt bloat, cap the number of tools/names shown per server, and redact/omit any metadata beyond short descriptions.
     **Files**: `packages/opencode/src/session/prompt.ts`, `packages/opencode/test/session/prompt-effect.test.ts`
     **Acceptance**: Sessions with lazy MCP servers include a short bounded system reminder about lazy MCP discovery; sessions without lazy servers do not get extra noise; prompt text does not leak schemas, env, headers, or auth metadata.

- [ ] 8. Keep loaded-tool execution on the existing MCP wrapper path
     **What**: Update the tool assembly in `SessionPrompt` so it asks `mcp.tools({ includeLazy: activated })`, then wraps the returned MCP tools exactly as today. Do not add any special execution branch for lazy tools; once loaded, they should be indistinguishable from eager tools at execution time. Make permission inheritance explicit in code/comments/tests: catalog lookup and `load` only affect exposure, while actual execution still flows through the existing wrapper and `ctx.ask(...)` permission path.
     **Files**: `packages/opencode/src/session/prompt.ts`, `packages/opencode/test/session/prompt-effect.test.ts`
     **Acceptance**: Tests prove loaded lazy MCP tools still trigger the same permission checks, plugin hooks, truncation, and attachment/resource handling as eager tools through the existing wrapper code.

- [ ] 9. Confirm v1 prompt/resource behavior and isolate command impact
     **What**: Review `packages/opencode/src/command/index.ts` against the final MCP mode plumbing. Preferred v1 outcome: no lazy-specific changes because prompts/resources remain eager and out of scope. If lazy mode accidentally changes command visibility, add a minimal fix and test only for that regression.
     **Files**: `packages/opencode/src/command/index.ts`
     **Acceptance**: `/command` behavior for MCP prompts is either unchanged by design or covered by a narrowly scoped follow-up fix documented in code/tests.

- [ ] 10. Address remote/OAuth lazy-server edge cases at the design boundary
      **What**: Plan for remote/OAuth servers explicitly: in v1, `mode: "lazy"` affects tool exposure only, not auth/connection semantics. A lazy remote server that requires OAuth may still report `needs_auth` at startup and therefore may have no catalog entries until authentication succeeds. Document that the meta-tool should surface this as unavailable/unauthed rather than attempting auth itself. Also define mid-session behavior after lazy activation: if a loaded remote tool hits expired tokens or auth loss later, execution should fail through the existing MCP wrapper/status path, no silent auto-renew should be added in v1, and the user should be directed back to the existing auth routes/flows. Verify that existing connect/auth routes remain the recovery path.
      **Files**: `packages/opencode/src/mcp/index.ts`, `packages/opencode/src/server/routes/mcp.ts`, `packages/opencode/test/mcp/oauth-auto-connect.test.ts`, `packages/opencode/test/mcp/oauth-browser.test.ts`
      **Acceptance**: The plan and tests make clear that lazy mode does not silently skip or auto-complete OAuth, auth-required servers do not expose phantom tools, expired/invalid tokens after activation fail safely without auto-renew in v1, and existing auth routes remain the supported path.

- [ ] 11. Add regression coverage for edge cases
      **What**: Add tests for mixed eager/lazy/disabled configs, repeated loads, missing tool selections, search by description, search caps/limit clamping, per-session rate limiting for `search`/`load`, catalog redaction, permission-filtered catalog results, warning behavior for conflicting `mode` + `enabled`, selector regex/max-length validation before lookup, tool-id sanitization/collision expectations, expired-token behavior after lazy activation, and cleanup on cancellation/disposal. Use the existing test files that already exercise these layers rather than creating vague new placeholders.
      **Files**: `packages/opencode/test/config/config.test.ts`, `packages/opencode/test/mcp/lifecycle.test.ts`, `packages/opencode/test/mcp/oauth-auto-connect.test.ts`, `packages/opencode/test/mcp/oauth-browser.test.ts`, `packages/opencode/test/session/prompt-effect.test.ts`, `packages/opencode/test/session/session.test.ts`
      **Acceptance**: Targeted tests fail before the feature and pass after it, with no need for repo-root test runs.

- [ ] 12. Verify package-local quality gates
      **What**: Run package-local typecheck and targeted tests from `packages/opencode`, using the concrete files already identified in this plan.
      **Acceptance**: `bun typecheck` and the targeted `bun test` commands below pass from `packages/opencode`.

## Verification

- [ ] All tests pass
- [ ] No regressions
- [ ] `packages/opencode` typechecks with `bun typecheck`
- [ ] `cd packages/opencode && bun test test/config/config.test.ts`
- [ ] `cd packages/opencode && bun test test/mcp/lifecycle.test.ts`
- [ ] `cd packages/opencode && bun test test/mcp/oauth-auto-connect.test.ts`
- [ ] `cd packages/opencode && bun test test/mcp/oauth-browser.test.ts`
- [ ] `cd packages/opencode && bun test test/session/prompt-effect.test.ts`
- [ ] `cd packages/opencode && bun test test/session/session.test.ts`
- [ ] Mixed `eager` + `lazy` + `disabled` MCP configs behave as designed
- [ ] Conflicting `mode` + `enabled` preserves `mode` and logs a warning
- [ ] Lazy exposure is explicit v1 behavior; lazy servers may still connect at startup
- [ ] Meta-tool `search` and `load` enforce per-session rate limits
- [ ] Catalog/search responses are permission-filtered before results are returned
- [ ] `server` and `name` selectors are validated with allowlist regex and max lengths before lookup
- [ ] Startup connection fan-out remains bounded and does not introduce retry storms
- [ ] Lazy-loaded tools execute through the existing `session/prompt.ts` MCP wrapper path
- [ ] The `mcp` meta-tool never directly executes MCP tools
- [ ] Catalog/search results stay bounded and redact sensitive metadata
- [ ] Expired OAuth tokens after lazy activation fail safely and use existing auth flows for recovery
- [ ] Post-implementation security review is completed and recorded
