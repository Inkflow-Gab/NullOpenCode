# OpenCode v2.2.5 source map and free-message limit trace

## Snapshot

The `opencode/` submodule is pinned to upstream tag `v2.2.5`, commit `9bbb8ce18c80f099e0e448775f59d965ead40a21`. This matches the version reported by the Ubuntu proot installation. Initialize it when cloning this repository with:

```bash
git clone --recurse-submodules <repository-url>
# or, after cloning:
git submodule update --init --recursive
```

The upstream source is kept as a submodule rather than copied into this repository: the v2.2.5 tree contains about 9,248 tracked files and 141 MiB of blob content. The submodule makes the complete checkout available at `opencode/` while preserving upstream history and avoiding a large duplicate import.

## Repository structure

| Path | Responsibility |
| --- | --- |
| `opencode/packages/cli` | CLI entry point, commands, TUI startup, managed and standalone server lifecycle. |
| `opencode/packages/core` | Sessions, model/provider setup, tools, permissions, persistence, and session error conversion. |
| `opencode/packages/server` | HTTP/WebSocket API and the local background service. |
| `opencode/packages/client` | Typed API clients, service discovery/launch, and browser/client state synchronization. |
| `opencode/packages/ai` | Provider request execution, response parsing, HTTP error classification, and retry primitives. |
| `opencode/packages/tui` | Terminal interface, session lifecycle, dialogs, and command affordances. |
| `opencode/packages/session-ui` | Shared transcript/timeline projection and error-row rendering. |
| `opencode/packages/app` | Web application UI and its usage-limit dialogs. |
| `opencode/packages/schema` and `protocol` | Shared API/event/error schemas and endpoint definitions. |

Normal CLI startup is routed through `packages/cli/src/index.ts`. The default session handler resolves the local managed service through `ServerConnection.resolve`; standalone mode instead starts a one-session server process. The server, core, client, and UI packages are separate so the same typed session/error model can drive terminal and web clients.

## The quoted “today's free messages” notice

The exact text is rendered in `opencode/packages/tui/src/routes/session/index.tsx` by `FreeLimitNotice`. The line is a presentation of a typed failed model request; it is not the component enforcing the allowance.

### Error path

1. **Free-model availability is configured locally.** In `packages/core/src/plugin/provider/opencode.ts`, an unconnected OpenCode provider is enabled with a public placeholder key unless `OPENCODE_DISABLE_FREE_MODELS` is set. In that unauthenticated state, models with positive input cost are disabled. This controls which catalog entries the client offers; it is not a local message counter.
2. **The inference request is sent to OpenCode's hosted service.** A refusal returns as an HTTP error. The open-source client does not include the hosted service's account-level counter or the algorithm that decides the daily allowance.
3. **HTTP metadata is translated into a typed error.** `packages/ai/src/route/executor.ts` reads `Retry-After-Ms` or `Retry-After` (seconds or an HTTP date) and passes the result as `retryAfterMs`. `packages/ai/src/provider-error.ts` recognizes `FreeUsageLimitError`/`freeusagelimiterror` as quota evidence. Quota classification is checked before the generic HTTP 429 rate-limit branch, so this specific refusal is not mistaken for a transient throttle.
4. **The session layer gives the failure a stable UI type.** `packages/core/src/session/to-session-error.ts` maps a quota error whose response body contains `FreeUsageLimitError` to `provider.free-limit`, preserving HTTP status, response body, and retry delay. The message becomes “Free usage limit reached. Connect a provider or subscribe to OpenCode Go to keep going, or switch models.”
5. **Retries stop for this case.** The provider-error test asserts `SessionRunnerRetry.isRetryable(free) === false`; the client does not repeatedly resend a request after this account cap.
6. **The timeline derives the displayed reset.** `packages/session-ui/src/timeline/projection.ts` computes `resetAt = message completion/creation time + retryAfterMs`, but accepts only finite delays from zero through seven days. The TUI renders a local time when the reset falls today, otherwise a localized date and time. If the response has no valid retry delay, it displays the limit text without a reset time.
7. **The composer and upsell respond to the event.** The TUI restores the stopped prompt if the composer is empty, then offers the Go dialog at most once per 24 hours (or suppresses it if “don't show again” was saved). Dismissing it leaves the inline notice. The inline `/connect` and `/models` hints are commands, not a client-side unlock or reset mechanism.

Relevant implementation and tests:

- `packages/ai/test/provider-error.test.ts`: a `429` with `FreeUsageLimitError` and `Retry-After: 3600` becomes a non-retryable free-limit session error with `retryAfterMs: 3600000`.
- `packages/core/test/session-error.test.ts`: verifies the provider error is converted to `provider.free-limit`.
- `packages/session-ui/src/timeline/rows-current.test.ts`: verifies the projected reset timestamp and rejects unreasonable reset delays.
- `packages/tui/test/app-lifecycle.test.tsx`: verifies the exact notice, prompt restoration, modal cadence, dismissal, and Go connection flow.

## What the local source does not reveal

The client source does **not** define the number of free messages, how requests are counted, which account fields are consulted, or how the hosted service chooses the reset time. It receives a typed quota refusal and optional retry metadata from the remote inference service. The displayed “today”/reset time is therefore based on server-supplied retry metadata, not a locally calculated midnight quota window.

Do not conflate this with two other limit types in the source:

- `BudgetLimitError` becomes a workspace/model budget limit, with reset data supplied by OpenCode Console; these budgets can pause paid models while free models remain available.
- `GoUsageLimitError` becomes `provider.go-limit`, for an OpenCode Go subscription allowance. Go usage meters are separately fetched by `packages/core/src/usage-limits.ts` from the Console/inference API.

## Mobile proot relevance

The reported environment is AArch64 Ubuntu with glibc 2.43. The v2.2.5 lockfile includes the Linux ARM64 GNU PTY package. The most relevant device-specific behavior is therefore process/PTY startup under proot, rather than the free-message policy, which is decided remotely. The source has both a shared-service path and a `--standalone` path that can help isolate local service/PTY behavior.

## Verification

This is a source trace; the monorepo test suite was not run. The cited tests describe the intended error/UI behavior, but do not exercise the production hosted quota service.
