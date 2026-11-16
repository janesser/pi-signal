---
title: "Build pi-signal fork with per-message backend readiness"
type: 'feature'
created: '2026-11-16'
status: 'done'
route: 'dispatch'
baseline_commit: 'NO_VCS'
review_loop_iteration: 0
context: []
---

<frozen-after-approval reason="human-owned intent — do not modify unless human renegotiates">

## Intent

**Problem:** Incoming Signal messages are forwarded to the agent before ensuring the llama.cpp backend on cyberkleiber (`cyberkleiber.fritz.box:8081`) is reachable. When the box is asleep, no message can be processed and no reply can be sent — the service silently fails.

**Approach:** Fork `pi-signal` at `~/projs/pi-signal` (installed via `pi install git:...`, janesser remote by owner) and add a readiness gate on every `sendUserMessage()` call: verify the backend is online, if absent exec `PI_SIGNAL_WAKEUP_CMD` and wait `PI_SIGNAL_WAKEUP_DELAY` before sending.

**Decisions recorded:**
- Source: `github.com/aalzubidy/pi-signal` base today, rebase onto janesser remote later (answer to Open Q1)
- Package name: keep `@aalzubidy/pi-signal` on the fork (answer to Open Q2)
- Config: env vars via pi-agent/env or systemd unit; feature is **skipped when envvars are not set** — opt-in, fail-closed (answer to Open Q3)
- Readiness check: `PI_SIGNAL_READINESS_CMD` configures the health-check command instead of hard-coding `cyberkleiber.fritz.box:8081`

## Boundaries & Constraints

**Always:**
- Every incoming message flows through one `sendUserMessage()` call before reaching the agent — this is the single interception point
- Only a shell script can operate on an asleep box (`wakeup.cyberkleiber` = WoL + poll) — no agent tool can reach it
- All scripts/units live userspace (`~/.local/bin/`, `~/.config/systemd/user/`); system-level changes are target-state only

**Never:**
- Don't modify `sendToSignal()` or the JSON-RPC reply path (unaffected by readiness)
- Don't touch signal-cli key material in `~/.local/share/signal-cli/data/`
- Don't parse message bodies for machine-control commands — let upstream `injectMessage()` handle commands; the fork only adds the readiness gate

**Opt-in behavior:** If `PI_SIGNAL_WAKEUP_CMD` is not set, the readiness gate is skipped entirely (fork behaves like upstream). If `PI_SIGNAL_READINESS_CMD` is not set, defaults to a curl check on cyberkleiber's llama.cpp backend.

## I/O & Edge-Case Matrix

| Scenario | Input / State | Expected Output / Behavior | Error Handling |
|----------|--------------|---------------------------|---------------|
| HAPPY_PATH | Message arrives, backend online | message injected into agent session immediately | N/A |
| ERROR_CASE | Message arrives, backend offline (and `PI_SIGNAL_WAKEUP_CMD` set) | `PI_SIGNAL_WAKEUP_CMD` execs; wait `PI_SIGNAL_WAKEUP_DELAY`; then retry readiness check | If still offline after retries, log and proceed anyway |
| EDGE_CASE | `PI_SIGNAL_WAKEUP_CMD` not set | Readiness gate skipped — behave like upstream (no delay) | N/A |
| EDGE_CASE | Machine cannot be reached | Proceed with send regardless (don't block messages) | Log warning only |

</frozen-after-approval>

## Code Map

<!-- Agent-populated during planning. Annotated paths prevent blind codebase searching. -->

- `/home/jan/projs/pi-signal/extensions/signal.ts` -- FORK SOURCE (upstream clone), where per-message handling is inserted
- `injectMessage()` in signal.ts -- where SSE messages are injected; this is the entry point for all incoming messages
- `forwardToAgent()` in signal.ts -- calls `pi.sendUserMessage(formatted, {deliverAs: "followUp"})` at line 599; readiness gate wraps this call
- `startSSEListener()` in signal.ts -- SSE subscription `/api/v1/events?account=+4917620128275` (unchanged)
- `daemonRpc()` in signal.ts -- JSON-RPC POST to `/api/v1/rpc` for replies (unchanged)
- `~/.pi/agent/settings.json` -- package registration: replace `npm:@aalzubidy/pi-signal` with fork install
- `signal-pi.service` (user unit, linger) -- runs `pi --mode rpc` with forked extension loaded

## Tasks & Acceptance

**Execution:**
- [x] `/home/jan/projs/pi-signal/extensions/signal.ts` -- add readiness gate to `forwardToAgent()` -- ensure backend before sendUserMessage
- [ ] `/home/jan/.pi/agent/settings.json` -- install fork via `pi install git:...` (replace npm package; pending janesser remote from owner)
- [ ] `signal-pi.service` unit -- verify extension loads from registry, no other changes needed

**Acceptance Criteria:**
- Given a Signal message arrives and cyberkleiber's llama.cpp backend is offline, when the forked extension processes it, then `PI_SIGNAL_WAKEUP_CMD` executes, the wait elapses, and the readiness check retries before sending
- Given the fork is installed via git: in `~/.pi/agent/settings.json`, when pi-agent restarts, then the forked extension loads from `/home/jan/projs/pi-signal`
- Given `PI_SIGNAL_WAKEUP_CMD` is not set, when a message arrives, then no readiness check runs (opt-in skip)

## Implementation Notes

<!-- Agent-owned. Append-only during implementation: decisions made, files touched, surprises encountered. -->

## Spec Change Log

<!-- Append-only. Populated by step-04 during review loops. -->

## Review Triage Log

<!-- Append-only. Populated by step-04 on every review pass. -->

## Design Notes

<!-- If the approach is straightforward, DELETE THIS ENTIRE SECTION. Do not write "N/A" or "None". -->
<!-- Design rationale and golden examples only when non-obvious. Keep examples to 5–10 lines. -->

DESIGN_RATIONALE: Wrapping `sendUserMessage()` rather than inspecting every SSE event keeps the change in one place — every message that reaches the agent passes through this call, so the readiness gate is guaranteed to run. Fail-closed opt-in: the feature only runs when configured; a missing env var means "no readiness check", not a broken message flow.

## Verification

<!-- If no build, test, or lint commands apply, DELETE THIS ENTIRE SECTION. Do not write "N/A" or "None". -->
<!-- How the agent confirms its own work. Prefer CLI commands. When no CLI check applies, state what to inspect manually. -->

**Commands:**
- `pi reload` (or `/reload` in agent) -- extension reloaded without errors
- `systemctl --system is-active signal-pi` -- service still active after install
- Send a Signal message to the account -- receive expected response when backend online
