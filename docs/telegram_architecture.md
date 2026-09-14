# Telegram Architecture

How the Telegram app communicates with NanoClaw end-to-end.

---

## Message In (Telegram → NanoClaw)

```
Your Telegram app
    │  (HTTPS long-poll)
    ▼
Telegram Bot API servers
    │  grammY library pushes update
    ▼
TelegramChannel.connect()  [src/channels/telegram.ts]
  bot.on('message:text', ...)
    │  normalizes content, builds NewMessage object
    │  checks: is this a registered group? (tg:8727658058)
    │  checks: sender allowlist (only your ID allowed)
    ▼
opts.onMessage(chatJid, msg)
    │  defined in index.ts as channelOpts.onMessage
    ▼
storeMessage(msg)  [src/db.ts]
    │  written to SQLite
    ▼
startMessageLoop()  [src/index.ts]
  polls DB every POLL_INTERVAL
    │  sees new message for tg:8727658058
    │  checks: does it need a trigger? (main group → no trigger needed)
    │  formats messages into prompt
    ▼
queue.enqueueMessageCheck(chatJid)
    │  GroupQueue serializes per-group processing
    ▼
processGroupMessages(chatJid)
    │  getMessagesSince(lastAgentTimestamp)
    │  formatMessages() → plain text prompt
    ▼
runContainerAgent()  [src/container-runner.ts]
    Docker container running Claude Agent SDK
```

---

## Agent Processing (Container)

```
Docker container
  - mounts: group folder (rw), /ideas (rw), rest read-only
  - reads: groups/telegram_main/CLAUDE.md (identity + features)
  - Claude SDK runs with your prompt
  - API key injected at request time by OneCLI
  - streams results back via stdout
```

---

## Response Out (NanoClaw → Telegram)

```
runContainerAgent streams ContainerOutput
    │  each result.result → strip <internal> blocks
    ▼
channel.sendMessage(chatJid, text)
    │  TelegramChannel.sendMessage()
    │  strips "tg:" prefix → numeric chat ID
    │  splits if > 4096 chars
    │  tries Markdown parse_mode, falls back to plain text
    ▼
Telegram Bot API
    │
    ▼
Your Telegram app
```

---

## Key Design Points

| Thing | Detail |
|---|---|
| **Transport** | grammY long-polling (not webhooks) — bot pulls updates from Telegram |
| **Storage** | Every message goes into SQLite before the agent sees it |
| **Trigger** | Main group (`tg:8727658058`) needs no trigger word — everything goes to the agent |
| **Non-main groups** | Would need `@WorkdayChat` trigger in the message |
| **Concurrency** | GroupQueue ensures one container per group at a time; new messages pipe into the running container |
| **Typing indicator** | `sendChatAction('typing')` fires while container is running |
