# OmniScout — operations & troubleshooting

Read this file when the health check in `SKILL.md` indicates the daemon
is missing, not running, or you've hit an unexpected error. Every non-healthy
state has a routing entry below; don't guess fixes outside of these.

## Path convention

All state lives under `~/Library/Application Support/omniscout/daemon/` (macOS)
or `$XDG_DATA_HOME/omniscout/daemon/` (Linux):

| File | Purpose |
|---|---|
| `daemon.pid` | Current daemon PID (absent when not running) |
| `daemon.port` | Picked TCP port (defaults to 7720) |
| `daemon.log` | Current run's stdout+stderr |
| `daemon.prev.log` | Previous run's log (rotated on start) |

## Routing table

| Observed | Action |
|---|---|
| `command not found` for `omniscout` | Install: `pip install omniscout` |
| `{"running": false}` from `omniscout daemon status` | `omniscout daemon start` |
| Daemon starts but immediately exits | `omniscout daemon logs --prev -n 200` to inspect the previous run; common cause is a port conflict — try `omniscout daemon start --port 0` to auto-pick. |
| Action returns `error_kind: "backend_unavailable"` | Extension backend selected but extension isn't connected. Either install the extension (see `extension/README.md`) OR drop `args.backend = "extension"` so omniscout uses Playwright. |
| Action returns `error_kind: "no_such_ref"` | The ref expired or the page changed. Re-run `snapshot` and use the new refs. |
| Action returns `error_kind: "timeout"` | Page hasn't settled. Try `wait --idle` then retry; or bump `--timeout-ms`. |
| Action returns `error_kind: "unsupported"` | Backend doesn't support that verb. `pdf` and `upload` require Playwright; switch backend. |
| `start` fails with "address already in use" | Another process is on `:7720`. Run `omniscout daemon stop && omniscout daemon start`. If still failing: `lsof -i :7720` and kill the conflicting PID, or specify a new port with `--port`. |

## Daily operations

```bash
omniscout daemon status              # JSON health & active sessions
omniscout daemon start               # idempotent
omniscout daemon stop                # SIGTERM then SIGKILL after 5s
omniscout daemon restart             # stop + start
omniscout daemon logs -n 100         # tail recent logs
omniscout daemon logs -f             # follow live
omniscout daemon logs --prev -n 200  # the previous run's log (after a crash)
```

## `/status` JSON fields

- `running` (bool) — daemon listening on its port
- `port` (int)
- `version` (string)
- `protocol_version` (string) — `"1"` today; agents should refuse to talk to a mismatching major version
- `extension_connected` (bool)
- `extension_id` (string\|null)
- `uptime_seconds` (int)
- `sessions` (list of session names currently held)
- `backends` (list of available backend names)
- `pid` (int)

## Common failure shapes

| Symptom | Diagnosis | Fix |
|---|---|---|
| `omniscout browser snapshot` returns a huge tree on a complex page | Reduce noise by capturing only refs: `omniscout browser snapshot --refs-only`. |
| `click @eN` returns `no_such_ref` | The ref expired (TTL 120s) or you re-navigated. Re-`snapshot`. |
| `fill` writes to the wrong element on SPAs | The role-based locator is ambiguous. Use the snapshot's `value` field to verify which `@e` you want; or fall back to a more specific CSS selector. |
| Screenshot path lives in a temp dir you can't access | Pass `--out /path/you/control.png` explicitly. |
| Banner doesn't appear on extension-driven tabs | Reload the page once after enabling the extension. |

## Reset to a clean state

```bash
omniscout daemon stop
rm -rf "$(omniscout install --print-data-dir 2>/dev/null || echo ~/Library/Application\ Support/omniscout)/daemon"
omniscout daemon start
```

Profiles, cookies, and the cache survive this; only the daemon's own state
(pidfile, portfile, logs) is removed.
