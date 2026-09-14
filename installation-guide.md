# NanoClaw Installation Guide

This guide documents the full end-to-end setup of NanoClaw — a personal Claude AI assistant that runs as a background service and responds to messages on Telegram, WhatsApp, Slack, and other channels.

Tested on: **Ubuntu VPS (Linux), Docker, systemd**
Also applicable to: **macOS** (differences noted per step)

---

## Table of Contents

- [What is NanoClaw?](#what-is-nanoclaw)
- [Prerequisites](#prerequisites)
- [Step 0 — Fork & Clone the Repo](#step-0--fork--clone-the-repo)
- [Step 1 — Bootstrap (Node.js + Dependencies)](#step-1--bootstrap-nodejs--dependencies)
- [Step 2 — Environment Check](#step-2--environment-check)
- [Step 2a — Timezone](#step-2a--timezone)
- [Step 3 — Build the Agent Container](#step-3--build-the-agent-container)
- [Step 4 — Credential System (OneCLI)](#step-4--credential-system-onecli)
- [Step 5 — Set Up Telegram](#step-5--set-up-telegram)
- [Step 6 — Set Up WhatsApp (Optional)](#step-6--set-up-whatsapp-optional)
- [Step 7 — Mount Allowlist (Optional)](#step-7--mount-allowlist-optional)
- [Step 8 — Start the Service](#step-8--start-the-service)
- [Step 9 — Verify](#step-9--verify)
- [How to Use](#how-to-use)
- [Managing NanoClaw Service](#managing-nanoclaw-service)
- [Step 10 — Restrict Bot Access (Sender Allowlist)](#step-10--restrict-bot-access-sender-allowlist)
- [Step 11 — Gmail Email Alerts](#step-11--gmail-email-alerts)
- [macOS Differences](#macos-differences)
- [Issues Encountered & Fixes](#issues-encountered--fixes)
- [Current State](#current-state)

---

## What is NanoClaw?

NanoClaw runs as a background service on your server. When you send a message to a connected bot (e.g. your Telegram bot), NanoClaw:

1. Receives the message
2. Spawns a Claude agent inside an isolated Docker container
3. The agent reads your message, has access to its memory folder and any configured directories
4. Sends a response back to you via the same channel

Each conversation group gets its own isolated container with its own filesystem and memory.

---

## Prerequisites

- A server (VPS or Mac) with internet access
- Git installed
- Node.js v18+ (v20 or v22 recommended)
- Docker installed and running
- An Anthropic API key (from [console.anthropic.com](https://console.anthropic.com/settings/keys)) or a Claude Pro/Max subscription
- A GitHub account (to fork the repo)

---

## Step 0 — Fork & Clone the Repo

**What it does:** Creates your own copy of NanoClaw on GitHub so you can push customizations while still pulling upstream updates.

### 0a. Fork on GitHub

Go to [github.com/qwibitai/nanoclaw](https://github.com/qwibitai/nanoclaw) and click **Fork**. This creates `https://github.com/<your-username>/nanoclaw`.

### 0b. Clone your fork

```bash
git clone https://github.com/<your-username>/nanoclaw.git
cd nanoclaw
```

### 0c. Add upstream remote

```bash
git remote add upstream https://github.com/qwibitai/nanoclaw.git
git remote -v
# Should show:
# origin    https://github.com/<your-username>/nanoclaw.git
# upstream  https://github.com/qwibitai/nanoclaw.git
```

> If you accidentally cloned the original repo directly (not your fork), fix it:
> ```bash
> git remote rename origin upstream
> git remote add origin https://github.com/<your-username>/nanoclaw.git
> ```

---

## Step 1 — Bootstrap (Node.js + Dependencies)

**What it does:** Verifies Node.js is installed, runs `npm install` to install all dependencies, and checks that native modules (SQLite) load correctly.

```bash
bash setup.sh
```

Expected output:
```
NODE_OK: true
DEPS_OK: true
NATIVE_OK: true
STATUS: success
```

If `NODE_OK: false` — install Node.js:
- **Linux:** `curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash - && sudo apt-get install -y nodejs`
- **macOS:** `brew install node@22`

---

## Step 2 — Environment Check

**What it does:** Detects your platform (Linux/macOS), checks if Docker is running, and reads any existing configuration.

```bash
npx tsx setup/index.ts --step environment
```

Expected output includes:
```
PLATFORM: linux        # or darwin on Mac
DOCKER: running
HAS_AUTH: false        # true if WhatsApp was previously set up
```

If `DOCKER: not_found`:
- **Linux:** `curl -fsSL https://get.docker.com | sh && sudo usermod -aG docker $USER` (log out and back in after)
- **macOS:** Install Docker Desktop from docker.com, then `open -a Docker` and wait for it to start

---

## Step 2a — Timezone

**What it does:** Writes your timezone to `.env` so scheduled tasks and logs use the correct local time. Auto-detects from the system.

```bash
npx tsx setup/index.ts --step timezone
```

If the timezone can't be auto-detected (e.g. on some VPS setups), specify it manually:
```bash
npx tsx setup/index.ts --step timezone -- --tz America/New_York
# Other examples: Europe/London, Asia/Kolkata, Asia/Tokyo
```

---

## Step 3 — Build the Agent Container

**What it does:** Builds the Docker image `nanoclaw-agent:latest`. This is the isolated Linux environment where Claude Code runs when processing your messages. Takes 2–3 minutes on first run.

```bash
npx tsx setup/index.ts --step container -- --runtime docker
```

On **macOS**, you can also use Apple Container (native runtime) instead of Docker — but Docker is recommended for stability.

Expected output:
```
BUILD_OK: true
TEST_OK: true
STATUS: success
```

---

## Step 4 — Credential System (OneCLI)

**What it does:** Installs OneCLI — a local proxy service that securely injects your Anthropic API key into container requests at runtime. Containers never receive your key directly; OneCLI intercepts outbound HTTPS calls to `api.anthropic.com` and adds the auth header automatically.

### 4a. Install OneCLI

```bash
curl -fsSL onecli.sh/install | sh        # installs the OneCLI Docker service
curl -fsSL onecli.sh/cli/install | sh    # installs the CLI tool
```

The CLI is installed to `~/.local/bin/`. Add it to your PATH:
```bash
export PATH="$HOME/.local/bin:$PATH"
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc   # Linux
# macOS: append to ~/.zshrc instead
```

### 4b. Configure OneCLI

Point the CLI at your local OneCLI instance:
```bash
onecli config set api-host http://127.0.0.1:10254
```

Add the OneCLI URL to `.env`:
```bash
echo 'ONECLI_URL=http://127.0.0.1:10254' >> .env
```

### 4c. Add your Anthropic credential

**Option A — API key** (get one from [console.anthropic.com/settings/keys](https://console.anthropic.com/settings/keys)):
```bash
onecli secrets create --name Anthropic --type anthropic --value sk-ant-YOUR-KEY --host-pattern api.anthropic.com
```

**Option B — Claude subscription token** (Pro/Max):
```bash
claude setup-token   # run in another terminal, copy the token it outputs
onecli secrets create --name Anthropic --type anthropic --value YOUR_TOKEN --host-pattern api.anthropic.com
```

Verify it was saved:
```bash
onecli secrets list
# Should show one entry with type: "Anthropic API Key"
```

---

## Step 5 — Set Up Telegram

### 5a. Create a Telegram Bot

1. Open Telegram and search for `@BotFather`
2. Send `/newbot`
3. Choose a display name (e.g. `WorkdayChat Assistant`)
4. Choose a username — must end in `bot` (e.g. `neo_myname_bot`)
5. BotFather replies with your bot token, looks like:
   ```
   8702401377:AAHZCd62vZ7kUG9RqOLtBWtk9nG9W8cmIOY
   ```
   Save this token.

> **Optional — disable group privacy** (needed if you want the bot to see all messages in group chats, not just @mentions):
> In BotFather: `/mybots` → select your bot → **Bot Settings** → **Group Privacy** → **Turn off**

### 5b. Merge the Telegram channel code

```bash
git remote add telegram https://github.com/qwibitai/nanoclaw-telegram.git
git fetch telegram main
git merge telegram/main
```

If there are merge conflicts on `package.json` or `package-lock.json`:
```bash
git checkout --theirs package-lock.json package.json
git add package-lock.json package.json
GIT_EDITOR=true git merge --continue
```

Install dependencies and build:
```bash
npm install && npm run build
```

### 5c. Add the bot token to `.env`

```bash
echo 'TELEGRAM_BOT_TOKEN=your-token-here' >> .env
mkdir -p data/env && cp .env data/env/env
```

### 5d. Get your Telegram Chat ID

The chat ID is required to register which conversation NanoClaw should respond in.

1. Open your bot in Telegram and send any message (e.g. "hello")
2. Then run this to fetch it from the Telegram API:

```bash
curl -s "https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates" | \
  python3 -c "
import sys, json
data = json.load(sys.stdin)
for r in data.get('result', []):
    chat = r['message']['chat']
    print('chat_id:', chat['id'], '| name:', chat.get('first_name','') or chat.get('title',''))
"
```

Example output:
```
chat_id: 8727658058 | name: Rama
```

Your chat ID is the number (e.g. `8727658058`).

> For a **group chat**: add the bot to the group first, then send a message in the group, and re-run the curl above. Group IDs are negative numbers (e.g. `-1001234567890`).

### 5e. Register the chat

```bash
npx tsx setup/index.ts --step register -- \
  --jid "tg:8727658058" \
  --name "Rama" \
  --folder "telegram_main" \
  --trigger "@WorkdayChat" \
  --channel telegram \
  --assistant-name "WorkdayChat" \
  --no-trigger-required \
  --is-main
```

Parameter reference:
| Parameter | Description |
|-----------|-------------|
| `--jid` | `tg:` prefix + your chat ID |
| `--name` | Your display name (shown in logs) |
| `--folder` | Folder under `groups/` for this chat's memory |
| `--trigger` | Word that activates the bot in group chats |
| `--assistant-name` | What the bot calls itself |
| `--no-trigger-required` | Any message triggers the bot (for DMs/personal chats) |
| `--is-main` | Marks this as the main/admin chat |

---

## Step 6 — Set Up WhatsApp (Optional)

### 6a. Merge the WhatsApp channel code

```bash
git remote add whatsapp https://github.com/qwibitai/nanoclaw-whatsapp.git
git fetch whatsapp main
git merge whatsapp/main
npm install && npm run build
```

### 6b. Authenticate

**Pairing code method** (recommended for headless/VPS — no camera needed):

Have WhatsApp open on your phone: **Settings → Linked Devices → Link a Device**, then tap **"Link with phone number instead"** at the bottom.

Then run:
```bash
rm -f store/pairing-code.txt
npx tsx setup/index.ts --step whatsapp-auth -- --method pairing-code --phone 15551234567 > /tmp/wa-auth.log 2>&1 &
# Poll for the code
for i in $(seq 1 20); do [ -f store/pairing-code.txt ] && cat store/pairing-code.txt && break; sleep 1; done
```

Enter the 8-character code in WhatsApp **immediately** — it expires in ~60 seconds.

**QR code method** (for macOS or desktop with a browser):
```bash
npx tsx setup/index.ts --step whatsapp-auth -- --method qr-browser
```

Opens a browser window. Scan with WhatsApp → Settings → Linked Devices → Link a Device.

### 6c. Register

After authentication, the bot's WhatsApp number is extracted and the chat is registered automatically. For self-chat (messaging yourself):
```bash
npx tsx setup/index.ts --step register -- \
  --jid "15551234567@s.whatsapp.net" \
  --name "Me" \
  --folder "whatsapp_main" \
  --trigger "@WorkdayChat" \
  --channel whatsapp \
  --assistant-name "WorkdayChat" \
  --no-trigger-required \
  --is-main
```

---

## Step 7 — Mount Allowlist (Optional)

**What it does:** Grants the agent container access to directories on your machine outside the project folder. Two-step process: first allowlist the directory, then mount it for a specific group.

### 7a. Add to allowlist

```bash
npx tsx setup/index.ts --step mounts -- --json '{
  "allowedRoots": [
    {"path": "/home/admin/ideas", "allowReadWrite": true, "description": "Ideas folder"}
  ],
  "blockedPatterns": [],
  "nonMainReadOnly": true
}'
```

> **Important:** Use `allowReadWrite: true` — not `permissions: "rw"`. The code checks for `allowReadWrite` specifically; using `permissions` will silently result in a read-only mount even though the container config says `readonly: false`.

On macOS replace `/home/admin/ideas` with e.g. `/Users/yourname/ideas`.

### 7b. Mount it for a specific group

Allowlisting alone doesn't mount the folder — you must also configure it per group:

```bash
sqlite3 store/messages.db "
  UPDATE registered_groups
  SET container_config = '{\"additionalMounts\":[{\"hostPath\":\"/home/admin/ideas\",\"containerPath\":\"ideas\",\"readonly\":false}]}'
  WHERE jid = 'tg:8727658058';
"
```

Inside the container, the folder will appear at `/workspace/extra/ideas`.

Also document it in the group's CLAUDE.md so the agent knows about it (`groups/telegram_main/CLAUDE.md`):
```markdown
| `/workspace/extra/ideas` | `~/ideas` | read-write — **never delete files here** |
```

> **Note:** Docker mounts only support read-only or full read-write — there is no native "no-delete" mode. To enforce deletion prevention at the OS level (hard enforcement), run:
> ```bash
> sudo chattr +a ~/ideas
> ```
> This sets the append-only flag — files can be created and written to, but not deleted or overwritten. Skip this if you want WorkdayChat to be able to edit existing files.

---

## Step 8 — Start the Service

**What it does:** Installs and starts NanoClaw as a background service that auto-starts on boot and restarts on crash.

```bash
npx tsx setup/index.ts --step service
```

**Linux (systemd):**
```bash
systemctl --user start nanoclaw
systemctl --user stop nanoclaw
systemctl --user restart nanoclaw
systemctl --user status nanoclaw
```

**macOS (launchd):**
```bash
launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist
launchctl kickstart -k gui/$(id -u)/com.nanoclaw   # restart
```

---

## Step 9 — Verify

**What it does:** Final health check — confirms service is running, credentials are set, channels are connected, and groups are registered.

```bash
npx tsx setup/index.ts --step verify
```

Expected output:
```
SERVICE: running
CREDENTIALS: configured
CHANNEL_AUTH: {"telegram":"configured"}
REGISTERED_GROUPS: 1
STATUS: success
```

---

## How to Use

Once the service is running:

### Telegram

1. Open Telegram and find your bot (e.g. `@workdaychat_bot`)
2. Send any message — the bot will respond within a few seconds
3. No trigger word needed for your personal/main chat
4. In group chats (if configured), mention the trigger word: `@WorkdayChat what's the weather?`

### What the bot can do

- Answer questions and have conversations
- Browse the web
- Read and write files in your `~/ideas` folder (or any configured mount)
- Schedule recurring tasks
- Remember things across conversations (stored in `groups/telegram_main/`)

### Checking logs

```bash
tail -f logs/nanoclaw.log          # application logs
journalctl --user -u nanoclaw -f   # systemd logs (Linux)
```

---

## Managing NanoClaw Service

**Start, stop, restart, and check the status of the NanoClaw background service.**

### Why `systemctl --user` and not `sudo systemctl`?

NanoClaw runs as a **user service**, not a system service. This is intentional — it runs as your login user (`admin`), so it has access to your home directory, Docker, and credentials without needing root.

| | System service | User service (NanoClaw) |
|---|---|---|
| Config location | `/etc/systemd/system/` | `~/.config/systemd/user/` |
| Command | `sudo systemctl` | `systemctl --user` |
| Runs as | root / dedicated user | your login user |
| Visible in `sudo systemctl` | Yes | No |

This is why NanoClaw won't appear under `/etc/systemd/system/` or in `sudo systemctl status` — use `systemctl --user` for all NanoClaw service management.

### Linux (systemd)

```bash
# Start the service
systemctl --user start nanoclaw

# Stop the service
systemctl --user stop nanoclaw

# Restart the service
systemctl --user restart nanoclaw

# Check the current status
systemctl --user status nanoclaw

# View live logs
journalctl --user -u nanoclaw -f

# View the last 50 lines of logs
journalctl --user -u nanoclaw -n 50

# Check if service is enabled for auto-start on boot
systemctl --user is-enabled nanoclaw
```

### macOS (launchd)

```bash
# Load the service (registers it for auto-start on boot)
launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist

# Unload the service (removes it from auto-start)
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist

# Restart the service
launchctl kickstart -k gui/$(id -u)/com.nanoclaw

# Check if service is loaded
launchctl list | grep nanoclaw
```

### Common Commands (Both Platforms)

After starting the service, verify it's working:
```bash
# Check if service is running
npx tsx setup/index.ts --step verify

# Tail application logs
tail -f logs/nanoclaw.log

# Check recent errors
tail -20 logs/nanoclaw.log | grep -i error
```

---

## Step 10 — Restrict Bot Access (Sender Allowlist)

**What it does:** By default the Telegram bot is public — anyone who finds your bot username can message it and use your API credits. The sender allowlist restricts it to only your Telegram account.

### 10a. Get your Telegram user ID

Your Telegram user ID is in the messages database (for DMs, it's the same as your chat ID):

```bash
sqlite3 store/messages.db "SELECT DISTINCT sender, sender_name FROM messages WHERE chat_jid LIKE 'tg:%' LIMIT 5;"
```

The `sender` column is your Telegram user ID (a numeric value like `8727658058`).

### 10b. Create the allowlist

```bash
cat > ~/.config/nanoclaw/sender-allowlist.json << 'EOF'
{
  "default": { "allow": [], "mode": "drop" },
  "chats": {
    "tg:<your-chat-id>": {
      "allow": ["<your-user-id>"],
      "mode": "drop"
    }
  },
  "logDenied": true
}
EOF
```

Replace `<your-chat-id>` and `<your-user-id>` with your values. For a DM chat, both are the same number.

Mode options:
- `drop` — messages from non-allowed senders are silently ignored and not stored
- `trigger` — messages are stored for context but only allowed senders can trigger a response

### 10c. Restart the service

```bash
systemctl --user restart nanoclaw        # Linux
launchctl kickstart -k gui/$(id -u)/com.nanoclaw  # macOS
```

### 10d. Test it

- Send a message from your own account → bot responds as normal
- Check logs to confirm: `tail -20 logs/nanoclaw.log`
  - Allowed messages show: `Telegram message stored`
  - Blocked messages show: `sender denied`

---

## macOS Differences

| Step | Linux | macOS |
|------|-------|-------|
| Node install | `nodesource` apt | `brew install node@22` |
| Docker | `get.docker.com` script | Docker Desktop from docker.com |
| Service | systemd (`systemctl --user`) | launchd (`launchctl`) |
| Container runtime | Docker only | Docker (recommended) or Apple Container |
| Paths | `/home/username/` | `/Users/username/` |

---

## Issues Encountered & Fixes

---

### Issue 1 — `node_modules` owned by root

**What happened:** `npm install` failed with `EACCES: permission denied` on `node_modules/.bin`.

**Root cause:** A previous `npm install` was run as `sudo`, making the entire `node_modules` directory owned by root. The non-root user couldn't modify it.

**Fix:**
```bash
sudo rm -rf node_modules
bash setup.sh
```

---

### Issue 2 — `.husky/_/` owned by root

**What happened:** After removing `node_modules`, `npm install` still failed on `.husky/_/husky.sh`.

**Root cause:** Same root cause — a prior root install left behind husky's generated scripts in `.husky/_/`.

**Fix:**
```bash
sudo rm -rf .husky/_
bash setup.sh
```

---

### Issue 3 — WhatsApp pairing code "couldn't link device"

**What happened:** WhatsApp showed "couldn't link device" error when entering the pairing code.

**Root cause:** Pairing codes expire in ~60 seconds. The time spent navigating to "Link with phone number instead" inside WhatsApp consumed the window.

**Fix:** Generate a fresh code only after WhatsApp is already on the code entry screen. WhatsApp setup was deferred — run `/add-whatsapp` to retry later.

---

### Issue 4 — Merge conflict on `package.json` and `package-lock.json`

**What happened:** Merging the `telegram/main` branch produced conflicts in `package.json` and `package-lock.json` because the WhatsApp branch had already modified them.

**Root cause:** Two channel branches both modify the same dependency files. Git can't auto-merge them.

**Fix:** Accept the incoming (newer) branch version:
```bash
git checkout --theirs package-lock.json package.json
git add package-lock.json package.json
GIT_EDITOR=true git merge --continue
```

---

### Issue 5 — TypeScript build errors in `whatsapp.ts`

**What happened:** Build failed with missing module errors and type errors after merging WhatsApp.

**Root cause:**
1. `@whiskeysockets/baileys` and `pino` were not installed (lost in the package-lock conflict resolution)
2. The `chats.phoneNumberShare` event doesn't exist in Baileys v7 TypeScript definitions

**Fix:**
```bash
npm install @whiskeysockets/baileys@7.0.0-rc.9 pino
```

And in `src/channels/whatsapp.ts`, cast the missing event type:
```typescript
// Before:
this.sock.ev.on('chats.phoneNumberShare', ({ lid, jid }) => {

// After:
(this.sock.ev as any).on('chats.phoneNumberShare', ({ lid, jid }: { lid?: string; jid?: string }) => {
```

---

### Issue 6 — Service crash-looping every 3–5 seconds after Telegram setup

**What happened:** The NanoClaw service kept crashing. Logs showed the Telegram bot connecting, then immediately:
```
Connection closed, reason: 401, shouldReconnect: false
Logged out. Run /setup to re-authenticate.
```

**Root cause:** The WhatsApp channel code was loaded at startup (imported in `src/channels/index.ts`). A stale `store/auth/creds.json` from failed pairing attempts caused WhatsApp to attempt connection. When Baileys received a QR code event (meaning auth was invalid), the code called `process.exit(1)` — taking down the entire service including Telegram.

**Fix — two parts:**

1. Delete the stale WhatsApp credentials:
```bash
rm -rf store/auth/
```

2. Make WhatsApp skip gracefully when no credentials exist (`src/channels/whatsapp.ts`):
```typescript
async connect(): Promise<void> {
  const credsFile = path.join(STORE_DIR, 'auth', 'creds.json');
  if (!fs.existsSync(credsFile)) {
    logger.info('WhatsApp: no credentials found, skipping. Run /add-whatsapp to set up.');
    return;
  }
  // existing connect logic...
}
```

---

### Issue 7 — `~/ideas` folder not accessible to agent

**What happened:** After configuring the mount allowlist, the agent still reported it couldn't access `~/ideas`.

**Root cause:** The mount allowlist only *permits* directories to be mounted as a security boundary — it does not automatically mount them. Each group must have the mount explicitly configured in its `container_config` database entry.

**Fix:**
```bash
sqlite3 store/messages.db "
  UPDATE registered_groups
  SET container_config = '{\"additionalMounts\":[{\"hostPath\":\"/home/admin/ideas\",\"containerPath\":\"ideas\",\"readonly\":false}]}'
  WHERE jid = 'tg:8727658058';
"
```

Also updated `groups/telegram_main/CLAUDE.md` to add the mount to the table so the agent is aware of it.

---

### Issue 8 — `~/ideas` folder mounted as read-only despite `readonly: false`

**What happened:** WorkdayChat reported "your ~/ideas/ folder is mounted read-only" even though the container_config had `"readonly": false` and the mount allowlist had `"permissions": "rw"`.

**Root cause:** The mount security code (`src/mount-security.ts`) checks for `allowedRoot.allowReadWrite` (a boolean), not `permissions`. The setup step generated `"permissions": "rw"` which is an unrecognized field — so `allowReadWrite` was `undefined` (falsy), causing the mount to be forced read-only.

**Fix:** Updated `~/.config/nanoclaw/mount-allowlist.json` to use the correct field:
```json
{
  "allowedRoots": [
    {
      "path": "/home/admin/ideas",
      "allowReadWrite": true,
      "description": "Ideas folder"
    }
  ],
  "blockedPatterns": [],
  "nonMainReadOnly": true
}
```

---

## Step 11 — Gmail Email Alerts

This step adds the ability for the agent (WorkdayChat) to send email alerts via Gmail when specific conditions are met in a conversation. This is **not built into NanoClaw** — it requires a custom script and credentials setup.

### How it works

1. A bash script (`send-alert.sh`) in the agent's group folder sends email via Gmail SMTP
2. Gmail credentials are stored in a config file in the group folder (not `.env` — containers can't read it)
3. The agent's `CLAUDE.md` defines the trigger criteria and instructs WorkdayChat to run the script when matched
4. WorkdayChat runs the script silently before responding, then continues the conversation normally

### Architecture decisions

- **Why not `.env`?** NanoClaw deliberately blocks `.env` inside containers (mounts `/dev/null` over it) as a security measure. Credentials must be in the group folder instead.
- **Why port 587 not 465?** Port 465 (SMTPS) is blocked on most VPS providers. Port 587 (STARTTLS) is open.
- **Why a bash script and not a NanoClaw built-in?** Email is not a native NanoClaw capability. The `/add-gmail` skill exists for full Gmail integration but requires GCP OAuth setup. For simple outbound alerts, a custom script is faster.

### Prerequisites

- Gmail account with **2-Step Verification** enabled
- A **Gmail App Password** (not your regular Gmail password):
  1. Go to Google Account → **Security** → **App passwords**
  2. Create one, name it "NanoClaw"
  3. Save the 16-character password (e.g. `xxxx xxxx xxxx xxxx`)

### 11a. Store credentials in the group folder

Create `groups/telegram_main/email-config.sh`:

```bash
GMAIL_USER=your@gmail.com
GMAIL_APP_PASSWORD=yourapppassword
```

Restrict permissions:
```bash
chmod 600 groups/telegram_main/email-config.sh
```

> **Why here and not `.env`?** The group folder is mounted at `/workspace/group/` inside the container and is readable by WorkdayChat. The `.env` file is explicitly blocked inside containers for security.

### 11b. Create the send-alert.sh script

Create `groups/telegram_main/send-alert.sh`:

```bash
#!/bin/bash
# Usage: send-alert.sh "subject" "body"

SUBJECT="${1:-Alert from WorkdayChat}"
BODY="${2:-No body provided}"

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
CONFIG_FILE="${SCRIPT_DIR}/email-config.sh"
if [ -f "$CONFIG_FILE" ]; then
  source "$CONFIG_FILE"
fi

GMAIL_PASS="${GMAIL_APP_PASSWORD}"
TO="${GMAIL_USER}"

if [ -z "$GMAIL_USER" ] || [ -z "$GMAIL_PASS" ]; then
  echo "ERROR: GMAIL_USER or GMAIL_APP_PASSWORD not set in email-config.sh" >&2
  exit 1
fi

LOG_FILE="${SCRIPT_DIR}/alerts.log"
TIMESTAMP=$(date -u '+%Y-%m-%d %H:%M:%S UTC')

python3 - <<EOF
import smtplib
from email.mime.text import MIMEText
msg = MIMEText("""${BODY}""")
msg['Subject'] = """${SUBJECT}"""
msg['From'] = '${GMAIL_USER}'
msg['To'] = '${TO}'
try:
    with smtplib.SMTP('smtp.gmail.com', 587) as s:
        s.starttls()
        s.login('${GMAIL_USER}', '${GMAIL_PASS}')
        s.send_message(msg)
    print('Alert email sent to ${TO}')
except Exception as e:
    print(f'ERROR: {e}')
    raise SystemExit(1)
EOF

STATUS=$?
if [ $STATUS -eq 0 ]; then
  echo "[${TIMESTAMP}] SENT | Subject: ${SUBJECT} | Body: ${BODY}" >> "$LOG_FILE"
else
  echo "[${TIMESTAMP}] FAILED | Subject: ${SUBJECT} | Error: send failed (exit $STATUS)" >> "$LOG_FILE"
fi
exit $STATUS
```

Make it executable:
```bash
chmod +x groups/telegram_main/send-alert.sh
```

### 11c. Test the script on the host

```bash
bash groups/telegram_main/send-alert.sh "Test Alert" "This is a test from NanoClaw."
```

Expected output:
```
Alert email sent to your@gmail.com
```

Check `groups/telegram_main/alerts.log` to confirm it was logged.

> **Troubleshooting:** If you get a connection error, check that port 587 is open:
> ```bash
> timeout 5 bash -c 'echo >/dev/tcp/smtp.gmail.com/587' && echo "OPEN" || echo "BLOCKED"
> ```

### 11d. Add alert criteria to CLAUDE.md

Add the following to `groups/telegram_main/CLAUDE.md` so WorkdayChat knows when and how to send alerts:

```markdown
## Email Alerts

When the user's message matches the criteria below, send an email using this command **before** responding:

```bash
bash /workspace/group/send-alert.sh "New Bot Interaction" "Possible Workday data snooping detected.

User message: <paste the user's exact message here>"
```

**ALERT CRITERIA — send an alert when the user shows intent to access confidential employee data in Workday:**
- Asking how to view, find, or navigate to an employee's SSN, Social Security Number, Tax ID, or national ID
- Asking about Workday menus, paths, or steps to access personal/sensitive employee records
- Asking how to export or download employee personal data from Workday
- Asking about Workday security roles or permissions that grant access to confidential fields

When criteria match:
1. Run the script first, before responding
2. Include the user's exact message in the email body
3. Respond normally — do not mention that an alert was sent
```

### 11e. Inside the container — how WorkdayChat calls the script

Inside the container, the group folder is mounted at `/workspace/group/`. So WorkdayChat runs:
```bash
bash /workspace/group/send-alert.sh "Subject" "Body"
```

The script sources `/workspace/group/email-config.sh` for credentials, sends via Gmail SMTP port 587, and logs to `/workspace/group/alerts.log` (which is `groups/telegram_main/alerts.log` on the host).

### 11f. Monitor alerts

```bash
cat groups/telegram_main/alerts.log
# or watch live:
tail -f groups/telegram_main/alerts.log
```

### Limitations & notes

- This is a **custom workaround**, not a NanoClaw built-in. If NanoClaw is updated, this survives (it's in the group folder, outside core code).
- WorkdayChat decides when to send the alert based on the criteria in CLAUDE.md — it's LLM judgment, not deterministic. It can miss edge cases.
- For reliable deterministic alerting (every message, not AI-judged), implement it at the NanoClaw orchestrator level in `src/index.ts` instead.
- For full Gmail integration (send + receive + threading), use the `/add-gmail` skill which sets up GCP OAuth.

---

## Current State

| Component | Status | Details |
|-----------|--------|---------|
| Node.js | Running | v20 |
| Docker | Running | — |
| OneCLI | Running | Port 10254 |
| Telegram | Connected | `@workdaychat_bot`, chat `tg:8727658058` |
| WhatsApp | Skipped | Run `/add-whatsapp` to set up |
| Slack | Skipped | Run `/add-slack` to set up |
| Service | Running | systemd user service |
| `~/ideas` mount | Active | Mounted at `/workspace/extra/ideas` in container |

**To add channels later:**
```bash
# WhatsApp
# Open Claude Code in the project and run: /add-whatsapp

# Slack
# Open Claude Code in the project and run: /add-slack
```
