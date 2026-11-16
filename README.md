# @aalzubidy/pi-signal

Fork of `pi-signal` (base `github.com/aalzubidy/pi-signal`) with a **per-message backend readiness gate**: before every message is forwarded to the agent, this extension ensures the configured backend is reachable. If it isn't, an optional wake-up command runs and the check retries — all of it opt-in, fail-closed.

## Installation

```bash
pi install /home/jan/projs/pi-signal
```

See `~/.pi/agent/settings.json` after install — the fork must appear in the package list for the extension to load from `/etc/systemd/system/signal-pi.service` (or whichever unit runs `pi --mode rpc`).

## Optional configuration

All of the following env vars are **optional**. When unset, the readiness gate is skipped entirely and the fork behaves like upstream `pi-signal`.

| Env var | Default | Meaning |
|---------|---------|---------|
| `PI_SIGNAL_ACCOUNT` | — | Signal account to listen on (upstream) |
| `PI_SIGNAL_DAEMON_URL` | `http://127.0.0.1:8080` | signal-cli daemon RPC/SSE URL (upstream) |
| `PI_SIGNAL_PRIMARY` | `false` | Only the primary instance processes Signal messages (upstream) |
| `PI_SIGNAL_WAKEUP_CMD` | *(unset)* | Shell command to run when the backend is unreachable — typically a Wake-on-LAN + poll script for the box running the backend. When unset, the gate is skipped entirely |
| `PI_SIGNAL_READINESS_CMD` | `curl -s --max-time 5 -o /dev/null -w '%{http_code}' http://cyberkleiber.fritz.box:8081/` | Health-check command returning an HTTP status code (e.g. `200`) for the backend. Replaces hard-coding the target so any backend works |
| `PI_SIGNAL_WAKEUP_DELAY` | `30` | Seconds to wait after running `PI_SIGNAL_WAKEUP_CMD` before re-checking the backend |
| `PI_SIGNAL_MAX_BACKEND_RETRIES` | `5` | Number of retries to reach the backend before proceeding anyway (i.e. forwarding the message regardless) |

### Example: systemd unit env vars

```ini
[Service]
Environment=HOME=/home/jan
Environment=PI_SIGNAL_ACCOUNT=+4917620128275
Environment=PI_SIGNAL_WAKEUP_CMD=/home/jan/.local/bin/wakeup.cyberkleiber && sleep 10
Environment=PI_SIGNAL_READINESS_CMD=curl -s --max-time 5 -o /dev/null -w '%{http_code}' http://cyberkleiber.fritz.box:8081/
```

## Behavior when the backend is down

With `PI_SIGNAL_WAKEUP_CMD` set:

1. Readiness check (`PI_SIGNAL_READINESS_CMD`) fails → log and run `PI_SIGNAL_WAKEUP_CMD` once per retry, with a 120s timeout.
2. Wait `PI_SIGNAL_WAKEUP_DELAY` seconds.
3. Re-check the backend. Repeat up to `PI_SIGNAL_MAX_BACKEND_RETRIES` times.
4. If still unreachable: log a warning and **proceed with sending** the message to the agent anyway (fail-open). No message is ever dropped — only logged.

This behavior matches the I/O & Edge-Case matrix in the spec: "Machine cannot be reached → proceed with send regardless, log warning only".
