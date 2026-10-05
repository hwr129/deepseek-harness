# Agent Note: session-telemetry-otel drops its baked-in endpoint fallback

Status: implemented

English | [中文](2026-08-22-telemetry-endpoint-requires-explicit-url.zh.md)

## Problem

A security review of the shipped defaults ([2026-07-31 default mount](../feature/2026-07-31-web-telemetry-default-mount.md)) found that `DSH_TELEMETRY_MODE` and `DSH_TELEMETRY_OTLP_URL` were not independent opt-ins: the shared base bundle's `exporter.url` fell back to a fixed DeepSeek-operated collector (`https://harness-telemetry.deepseeksvc.com/v1/logs`) whenever the override was unset. That note's own Consequences section already discloses that an enabled export carries the raw captured copy — full user/assistant message text, tool arguments and results, the system prompt, and the local `session.cwd` path, per its no-redaction-rule ruling. Setting `DSH_TELEMETRY_MODE=FULL` alone was therefore enough to send that raw copy to a specific external destination the operator never separately chose. `DSH_TELEMETRY_MODE` is itself a trusted, operator-set environment variable — nothing here was attacker-reachable — but coupling "authorize an upload" to "and send it here by default" is an unnecessary risk for a path whose whole purpose is keeping real session content off the wire unless a destination was deliberately chosen.

## Decision

The shared base bundle (`packages/bundle/base/cordis.patch.yml`) no longer supplies a fallback for `exporter.url`; it now reads `DSH_TELEMETRY_OTLP_URL` verbatim, which resolves to `undefined` when unset. `@deepseek-ai/dsh-session-telemetry-otel`'s own existing load-time check (`session-telemetry-otel/src/index.ts`, the branch after the `DISABLED`-mode early return) already throws `exporter.url is required (the full OTLP logs endpoint)` when the resolved mode is `FULL` or `FEEDBACK_ONLY` and no URL is present — this Note removes the composition-layer default that kept that check from ever firing. No package code changed; the fix is entirely in the shared bundle's config values.

`DISABLED` (the shipped default per the [default-off Note](../feature/2026-08-10-telemetry-default-off.md)) is unaffected: the constructor returns before the URL is read at all in that branch, so a fresh profile still makes no telemetry network request and needs no URL configured. Enabling upload now requires setting both `DSH_TELEMETRY_MODE` and `DSH_TELEMETRY_OTLP_URL` in the same launch environment — two independent, both-required decisions instead of one flag defaulting the other.

This partially supersedes the [default-mount Note](../feature/2026-07-31-web-telemetry-default-mount.md)'s Endpoint ruling only; that Note's mode default-off, hard opt-out variable, cadence, exit-drain bounds, compression, and CI isolation rulings are unchanged and remain the current authority for those facts, updated in place to cross-link here.

## Alternatives considered

**Warn instead of failing when the URL is missing.** Rejected: a log line is easy to miss in a headless or batch deployment, and the entire point is that raw session content must not leave the machine toward a destination the operator did not separately choose — a warning still lets the upload happen.

**Add a redaction rule instead of touching the endpoint default.** Rejected as a different, larger problem: redaction would shrink what an implicit destination receives, but does not address that the destination itself was never a decision the operator made when they only set the mode. The [default-mount Note](../feature/2026-07-31-web-telemetry-default-mount.md) already tracks the missing redaction rule as separate, still-open deployment work.

**Validate the URL against an allowlist instead of only requiring it be present.** Rejected as redundant: the existing load-time check already requires an operator-supplied http(s) URL (schemastery `Config`, validated in the constructor); an allowlist would add a second configuration surface for a value the operator already typed themselves.

## Consequences

- `DSH_TELEMETRY_MODE=FULL` or `FEEDBACK_ONLY` with no `DSH_TELEMETRY_OTLP_URL` now fails loud at plugin construction instead of silently uploading to the DeepSeek-operated collector. Any deployment that previously relied on the implicit default — including DeepSeek's own, per the default-mount Note's "internal collector" framing — must now also set `DSH_TELEMETRY_OTLP_URL` explicitly wherever it sets `DSH_TELEMETRY_MODE`. This is a breaking change for exactly that one implicit-default case.
- No behavior changes for the shipped default (`DISABLED`) or for any deployment that was already setting both variables together.
- The base bundle test (`packages/bundle/base/tests/base.spec.ts`) pins the `mode` expression only, not `exporter.url`, and needed no change; no test in the repository set `DSH_TELEMETRY_MODE=FULL` against the real base bundle composition, confirming no test relied on the removed fallback.
