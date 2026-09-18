# Backlog

## Security

### ~~Close OneCLI gateway's public exposure properly~~ — CLOSED 2026-09-18
**Context:** `~/.onecli/docker-compose.yml` publishes `onecli-app-1` on
`10254`/`10255` (and postgres on `5432`) bound to `0.0.0.0`. First found
2026-09-14 being actively abused as an open relay by internet scanners
(gateway logs showed tunneled traffic to random hosts — mail servers, AWS
sign-in, a VPN API — for peers that have nothing to do with this setup).
That abuse exhausted the gateway's file descriptors and crashed it, which
silently broke the daily GitHub-trending email — twice: once around
2026-09-13, and again by 2026-09-14T22:32 UTC (crashed and stayed dead for
4 days before being noticed on 2026-09-18). Full incidents in
`docs/dev-log.md`.

Rebinding the app ports to `127.0.0.1` was tried on 2026-09-14 and reverted
— it also blocks NanoClaw's own containers, since they reach the host via
`host.docker.internal` (the Docker bridge IP), not true loopback.

**Resolved 2026-09-18** with `DOCKER-USER` iptables rules instead of a
port-bind change — ports stay on `0.0.0.0` (needed for container access)
but the host firewall now only accepts 10254/10255 from `127.0.0.1` and
`172.16.0.0/12` (covers Docker's default bridge `172.17.0.0/16` and the
`onecli_onecli` bridge `172.21.0.0/16`), dropping everything else:
```
sudo iptables -I DOCKER-USER -p tcp --dport 10254 -s 172.16.0.0/12 -j ACCEPT
sudo iptables -I DOCKER-USER -p tcp --dport 10254 -s 127.0.0.1 -j ACCEPT
sudo iptables -A DOCKER-USER -p tcp --dport 10254 -j DROP
# same for 10255
sudo netfilter-persistent save   # survive reboot
```
**Gotcha hit during setup:** `-I` (insert at head) run multiple times in
sequence pushes each new rule above the last, so the `DROP` rules ended up
evaluated *before* the `ACCEPT` rules — silently blocking all traffic,
including legitimate container access. Fixed by deleting the `DROP` rules
and re-adding them with `-A` (append) so they land after the `ACCEPT`
rules. Always verify with `iptables -L DOCKER-USER -n --line-numbers`
after writing multi-rule chains — order determines behavior, not intent.

Verified with a live retest of the GitHub-trending task after applying:
real email sent, 187s runtime, no error.

`installing iptables-persistent` removed `ufw` as an automatic dependency
conflict resolution — not a concern, since `ufw` was already confirmed not
to affect Docker-published ports (Docker writes its own iptables rules
that bypass the standard `ufw`/INPUT chain).

Postgres (`5432`) was left on `127.0.0.1`-only from the 2026-09-14 change —
nothing needs bridge access to it, so no iptables rule was needed there.

### Rotate OneCLI postgres credentials
**Context:** `POSTGRES_USER`/`POSTGRES_PASSWORD` default to `onecli`/`onecli`
in the compose file and were never overridden. The DB was reachable from the
public internet (see above) for an unknown period before this was caught.
**Fix:** set real credentials via `~/.onecli/.env`, recreate the postgres
container, confirm the app still connects.

## Reliability

### ~~Find and prevent the source of the duplicate nanoclaw process~~ — CLOSED 2026-09-18
**Context:** On 2026-09-14, a second `node dist/index.js` process was found
running outside systemd's tracking, racing the systemd-managed instance for
the same due tasks and Telegram messages, causing duplicate replies and a
stuck message queue. A pidfile-based single-instance lock was added
(`src/index.ts:acquireSingleInstanceLock`) to contain the damage, but the
actual source stayed unidentified.

**Resolved 2026-09-18:** root cause was a second, system-level
`nanoclaw.service` at `/etc/systemd/system/nanoclaw.service`
(`WantedBy=multi-user.target`, starts on boot), separate from the
documented user-level unit at `~/.config/systemd/user/nanoclaw.service`.
Both auto-started and had been racing each other since whenever the system
unit was installed. Disabled with `sudo systemctl disable --now nanoclaw`,
keeping only the user-level unit. The pidfile lock stays in place as a
safety net but is no longer needed to survive this specific conflict.

### Silent failures in scheduled-task precheck scripts
**Context:** The precheck-script mechanism (`runScript` in
`container/agent-runner/src/index.ts`) treats any script error — including
network failures — identically to "nothing to do," so a broken precheck
looks exactly like a legitimately quiet day. This is what let the
GitHub-trending break go unnoticed for two days with zero error signal
anywhere (task status stayed `success`, `task_run_logs.error` stayed NULL).
Only the ECGONREFUSED/Anthropic-API-timeout errors are currently surfaced,
because those come from the full-agent path, not the precheck path.
**Fix:** have `runScript` distinguish "script ran, decided not to wake
agent" from "script failed to run" (e.g. non-zero exit, thrown error,
malformed output) and log/surface the latter distinctly — at minimum a WARN
in the main log, ideally a `task_run_logs.error` entry even when
`wakeAgent` never got evaluated.

### Audit other scheduled tasks for the same gateway-allowlist trap
**Context:** Only the GitHub-trending task's precheck was checked in this
incident. The Anthropic-news task's precheck also fetches an external host
(`anthropic.com/news`) directly — it happened to keep working because that
host quality-of-service wasn't affected, but it's exposed to the same
allowlist gap if OneCLI's policy changes again. Any other scheduled task
with a precheck script hitting a non-`api.anthropic.com` host has the same
exposure.
**Fix:** grep `scheduled_tasks.script` for other outbound `fetch()` calls
and confirm each one still works now that precheck env is stripped of
gateway vars (this session's fix already covers all of them structurally,
since the strip is unconditional — but worth a one-time verification pass).

### Stale session resume can hang a container indefinitely
**Context:** Separately from the above, two containers were found hung on
2026-09-14, both resuming a session ID (`bac729d7-...`) that traced back to
a stale session from a June 2026 API-usage-limit incident. They sat at
"Session initialized" for 5+ minutes without producing output or erroring,
blocking the group's message queue until manually killed
(`docker kill`). Root cause of *why* a months-old session got resumed, or
why the resume hung instead of failing fast, was not investigated.
**Fix:** investigate the session-resume logic (`resumeAt: latest` in
agent-runner) for why it picked a stale session, and consider a timeout on
container runs so a hung resume can't block the queue indefinitely (the
existing `SCRIPT_TIMEOUT_MS` only bounds precheck scripts, not the main
agent query).

## Observability

### No alerting on gateway or process health
**Context:** `onecli-app-1` sat `unhealthy` for at least a day before
anyone noticed, and the duplicate-process bug ran undetected across
multiple restarts. Nothing pages or notifies when either happens — it was
only caught by manually chasing an unrelated user complaint ("didn't get my
GitHub email").
**Fix:** at minimum, a periodic check (cron or a NanoClaw scheduled task
itself) that alerts via Telegram if `docker inspect onecli-app-1` reports
unhealthy, or if more than one `dist/index.js` process is running.
