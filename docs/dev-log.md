# Dev Log

## 2026-09-14

### Daily GitHub-trending email went silent

The 09:00 cron task that emails a daily GitHub trending digest stopped sending
on 2026-09-13, with no visible error anywhere — the task record just showed
"success" with an empty result and a ~3.5s runtime instead of its usual
3-5 minutes.

Root cause: the task's precheck script (a lightweight Node script that runs
inside the agent container to decide whether to wake the full Claude agent)
fetches `github.com/trending` to check reachability. All outbound container
traffic is unconditionally routed through the OneCLI gateway for credential
injection, and the gateway only allowlists hosts it knows about (chiefly
`api.anthropic.com`). `github.com` isn't on that list, so the precheck's
fetch got rejected — quickly, silently, with no error surfaced, because the
script's own logic just treated a non-200 response as "nothing to do."

Fixed by stripping the gateway's proxy/CA env vars for the precheck
subprocess specifically (`container/agent-runner/src/index.ts`), since
prechecks don't need injected credentials and should reach the open internet
directly. The full agent invocation still goes through the gateway as
before.

### OneCLI gateway was a public open relay

While digging into why the gateway stopped working exactly on 2026-09-13,
found the actual trigger: `onecli-app-1`'s ports (10254, 10255) — and
postgres's 5432 — were published on `0.0.0.0` in
`~/.onecli/docker-compose.yml`, reachable from the open internet. Gateway
logs showed it tunneling traffic for random external IPs to hosts that have
nothing to do with this setup (mail servers, AWS sign-in, a VPN API) —
being abused as an open relay by internet scanners. That abuse exhausted the
gateway's file descriptors and crashed it on 2026-09-13 08:28 UTC, hours
before that morning's cron task ran, which is the real reason the email
silently stopped.

Rebinding those ports to `127.0.0.1` seemed like the fix but broke container
access entirely — containers reach host services via
`host.docker.internal` (the Docker bridge IP), not true loopback, so a
`127.0.0.1`-only bind blocks them exactly like it blocks the public
internet. Reverted back to `0.0.0.0` to restore service. The correct fix —
`DOCKER-USER` iptables rules restricting the port to the bridge subnet while
keeping the bind public — is written up but not yet applied; needs `sudo`
access this session didn't have.

### Duplicate process was racing the real one

Separately found a second `node dist/index.js` process running outside
systemd's tracking (parent PID 1, not spawned by any known cron/timer/pm2 —
source never identified), racing the real systemd-managed instance for the
same due tasks and Telegram messages. That's what caused the double replies
and stuck message queue on top of the gateway issue.

Added a pidfile-based single-instance lock (`src/index.ts`,
`acquireSingleInstanceLock`) so a second instance refuses to start and logs
why instead of silently racing. First version leaked the lock file on
`SIGTERM` because Node doesn't run `exit` handlers on a bare signal unless
you catch it explicitly — fixed by adding `SIGTERM`/`SIGINT` handlers that
call `process.exit()`.

### Follow-ups

- Apply `DOCKER-USER` iptables rules to close the gateway's public exposure
  without breaking container access.
- Rotate the postgres password — it ran on the default `onecli/onecli`
  creds while publicly exposed.
- Never fully identified what launched the rogue duplicate process; if it
  recurs, the pidfile lock will at least stop it from causing damage and
  will log the conflict.
