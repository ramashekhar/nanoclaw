# Feature: Gmail Email Alerts

Send an alert email to a Gmail address when the agent detects specific user intent in a conversation.

## What it does

When a user message matches defined criteria (e.g. trying to access confidential Workday data), the agent silently sends an alert email to a configured Gmail address before responding normally. Every alert is logged to `alerts.log`.

## Architecture

```
groups/telegram_main/features/gmail-alerts/
├── instructions.md   ← agent reads this at session start (defines criteria + how to send)
├── send-alert.sh     ← bash script that sends email via Gmail SMTP
├── email-config.sh   ← Gmail credentials (chmod 600)
└── alerts.log        ← log of all sent alerts
```

The agent reads `instructions.md` automatically at the start of each session (via the feature loader in `CLAUDE.md`). When criteria match, it runs `send-alert.sh` which loads credentials from `email-config.sh` and sends via Gmail SMTP port 587.

## Why credentials go in the group folder (not .env)

NanoClaw deliberately blocks `.env` inside containers — it mounts `/dev/null` over it as a security measure so the agent never sees API keys or secrets. The group folder (`groups/telegram_main/`) is mounted at `/workspace/group/` and is accessible to the agent.

## Prerequisites

- Gmail account with **2-Step Verification** enabled
- A **Gmail App Password**:
  1. Google Account → **Security** → **App passwords**
  2. Create one named "NanoClaw"
  3. Copy the 16-character password

## Setup

### 1. Store credentials

Create `groups/telegram_main/features/gmail-alerts/email-config.sh`:
```bash
GMAIL_USER=your@gmail.com
GMAIL_APP_PASSWORD=yourapppasswordhere
```

Restrict permissions:
```bash
chmod 600 groups/telegram_main/features/gmail-alerts/email-config.sh
```

### 2. Verify port 587 is open

```bash
timeout 5 bash -c 'echo >/dev/tcp/smtp.gmail.com/587' && echo "OPEN" || echo "BLOCKED"
```

> Port 465 (SMTPS) is blocked on most VPS providers. Port 587 (STARTTLS) is used instead.

### 3. Test the script

```bash
bash groups/telegram_main/features/gmail-alerts/send-alert.sh "Test Alert" "Test from NanoClaw."
```

Expected output:
```
Alert email sent to your@gmail.com
```

Check the log:
```bash
cat groups/telegram_main/features/gmail-alerts/alerts.log
```

### 4. Define alert criteria

Edit `groups/telegram_main/features/gmail-alerts/instructions.md` to define when the agent should send an alert. The file is self-contained — add or remove criteria without touching `CLAUDE.md`.

Example criteria (current setup):
- User asks how to view an employee's SSN in Workday
- User asks about Workday menus to access confidential employee records
- User asks about exporting sensitive employee data from Workday

### 5. Restart the service

```bash
systemctl --user restart nanoclaw        # Linux
launchctl kickstart -k gui/$(id -u)/com.nanoclaw  # macOS
```

## Adding a new feature (pattern)

Follow the same structure for any new feature:

```
groups/telegram_main/features/<feature-name>/
├── instructions.md   ← agent instructions (auto-loaded)
├── <scripts>         ← any scripts the agent needs to run
└── <output files>    ← logs, data produced by the feature
```

The `CLAUDE.md` feature loader picks up `instructions.md` from any subfolder under `features/` automatically — no changes to `CLAUDE.md` needed.

## Monitoring

```bash
# Live alerts
tail -f groups/telegram_main/features/gmail-alerts/alerts.log

# All alerts
cat groups/telegram_main/features/gmail-alerts/alerts.log
```

## Limitations

- Alert triggering is **LLM-judged** — WorkdayChat decides if criteria match. It can miss edge cases or have false positives.
- For deterministic alerting (every message, no AI judgment), implement it in `src/index.ts` at the orchestrator level instead.
- For full Gmail integration (send + receive + threading), use the `/add-gmail` skill (requires GCP OAuth setup).
