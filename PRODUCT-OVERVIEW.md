# OpenClaw Product Overview

A guide for understanding how OpenClaw works — written for product thinkers, not just engineers.

---

## What Is OpenClaw?

OpenClaw is a **self-hosted AI assistant gateway** that runs on your own hardware (in our case, a Mac Mini). It connects to any messaging platform you use — iMessage, Slack, WhatsApp, Telegram, Discord, and 25+ others — and routes your conversations through an LLM of your choice.

Think of it as your own private ChatGPT that lives on your machine, talks to you on whatever platform you already use, and can take actions on your behalf.

**Key value props:**
- **You own the data** — conversations never leave your machine unless you choose to send them to an LLM provider
- **Platform agnostic** — same AI assistant across all your messaging apps
- **Extensible** — custom tools, skills, hooks, and plugins let you teach it new tricks
- **Always on** — runs as a background service, responds 24/7

---

## Architecture at a Glance

```
┌─────────────────────────────────────────────────────────┐
│                     YOUR MAC MINI                       │
│                                                         │
│  ┌─────────────┐    ┌──────────────┐    ┌───────────┐  │
│  │   Channels   │───▶│   Gateway    │───▶│   Agent   │  │
│  │  (iMessage,  │    │  (WebSocket  │    │  Runtime  │  │
│  │   Slack,     │◀───│   server)    │◀───│  (LLM +   │  │
│  │   Discord…)  │    │              │    │   tools)  │  │
│  └─────────────┘    └──────┬───────┘    └───────────┘  │
│                            │                            │
│              ┌─────────────┼─────────────┐              │
│              │             │             │              │
│         ┌────▼───┐   ┌────▼───┐   ┌─────▼────┐        │
│         │ Memory │   │  Cron  │   │  Hooks   │        │
│         │(SQLite │   │(sched- │   │(side     │        │
│         │ + FTS) │   │ uled   │   │ effects) │        │
│         └────────┘   │ jobs)  │   └──────────┘        │
│                      └────────┘                        │
└─────────────────────────────────────────────────────────┘
                            │
                     ┌──────▼──────┐
                     │  LLM APIs   │
                     │ (Gemini,    │
                     │  OpenAI,    │
                     │  Anthropic) │
                     └─────────────┘
```

---

## Core Components

### 1. Gateway (the brain)

The gateway is a Node.js WebSocket server that coordinates everything. It starts at boot, listens on a local port (18789), and never stops.

**What it does:**
- Receives messages from all connected channels
- Routes them to the right agent/session
- Manages authentication and access control
- Serves as the single control plane for the entire system

**Config:** `~/.openclaw/openclaw.json`

### 2. Channels (the ears and mouth)

Channels are messaging platform integrations. Each channel has a "monitor" that watches for incoming messages and a "sender" that delivers replies.

**Currently connected:**
- **iMessage** (primary) — via `imsg` CLI tool on macOS
- **Slack** — via Bolt SDK bot

**Available but not connected:**
- WhatsApp (web bridge), Telegram, Discord, Signal, Google Chat, Microsoft Teams, IRC, Matrix, LINE, and 15+ more

**How channels work:**
1. Channel monitor detects a new message (webhook or polling)
2. Forwards it to the gateway with sender identity, channel name, and content
3. Gateway processes it through the agent
4. Reply is delivered back through the same channel
5. Formatting is adapted per-channel (e.g., markdown stripped for iMessage)

### 3. Agent Runtime (the thinker)

When a message arrives, the agent runtime:
1. Loads the conversation session (past messages)
2. Assembles context from memory + session history
3. Calls the LLM with the full context
4. If the LLM requests tool use, executes the tool and feeds results back
5. Repeats steps 3-4 until the LLM produces a final reply
6. Sends the reply for delivery

**Model selection:**
- Primary: `google/gemini-3.1-flash-lite-preview` (fast, cheap)
- Fallback 1: `openai/gpt-4.1-mini`
- Fallback 2: `anthropic/claude-sonnet-4-6`
- Automatic failover if a provider is rate-limited or down

**Key design choice:** The agent model is intentionally lightweight. Complex work is done *outside* the agent (in scripts, webhook handlers, CLI tools), and the agent is only called for simple tasks like "classify this email" or "summarize this text." This keeps costs low and responses fast.

### 4. Memory System (the recall)

OpenClaw remembers conversations across sessions using:
- **Session store** — full message history per conversation, stored as JSON
- **Semantic search** — embeddings-based retrieval (Gemini embeddings + SQLite FTS) to find relevant past context
- **Compaction** — when a session gets too large, older messages are summarized to fit within the LLM's context window

**Auto-reset:** Sessions are automatically compacted at 300KB after a quiet period, and hard-reset at 800KB to prevent token bloat.

### 5. Hooks (the reflexes)

Hooks are event-driven side effects — they fire when something happens but don't block the main message flow.

**Active hooks:**
- **imessage-ack** — instantly sends a 👍 reaction when a message arrives, then sends "still working..." updates every 8 seconds while the agent thinks
- **imessage-delivery-check** — verifies that outbound messages were actually delivered (checks the Messages.app database 5 seconds after sending)
- **Model failover detection** — scans logs for `[FAILOVER-RAW]` and notifies if the agent switched models due to an error

**How hooks work:**
1. An event occurs (message received, message sent, etc.)
2. Hook handler runs asynchronously
3. Handler can read context, perform side effects, or even patch the message before the agent sees it
4. Failures are logged but never block message delivery

### 6. Cron Jobs (the scheduler)

Scheduled tasks that run at specific times, each spinning up an isolated agent session.

**Active jobs:**
| Job | Schedule | What it does |
|-----|----------|-------------|
| Morning Briefing | 8:00 AM ET daily | Calendar, weather, business news → iMessage + Slack |

**Disabled jobs (available to re-enable):**
| Job | Schedule | What it does |
|-----|----------|-------------|
| Evening Summary | 9:00 PM ET daily | News, orders, tasks recap |
| Email → Calendar | Every 5 min | Scan sent emails for meeting times |

---

## Custom Infrastructure (Your Additions)

Beyond the core OpenClaw platform, this installation has significant custom infrastructure:

### Webhook Receiver (`webhook-receiver.js`)

A standalone Node.js server (port 18795) that handles email triage — the most complex custom component.

**Flow:**
```
Gmail (new email arrives)
    ↓
Google Apps Script (runs every 1 min, detects new emails)
    ↓
Webhook POST to Mac Mini via Tailscale Funnel
    ↓
Webhook Receiver classifies the email
    ├── Direct routing (Amazon, VRBO, travel, billing → known handlers)
    └── AI classification (everything else → Gemini API call)
    ↓
Actions based on priority:
    ├── CRITICAL/HIGH → Star blue + Google Task + iMessage alert
    ├── NORMAL → Star yellow + Google Task
    ├── LOW → Leave unread, no action
    ├── NEWSLETTER → Archive
    ├── SPAM → Move to "Suspected as spam" label
    └── IGNORE → Skip entirely
```

**Special handlers:**
- **Amazon orders** — detects order/ship/deliver status, alerts for Miami-area deliveries
- **VRBO/Airbnb bookings** — extracts guest name, party size, dates, property name
- **Travel reservations** — forwards to TripIt
- **Sent email analysis** — detects proposed meeting times, creates follow-up tasks

### Watchdog (`watchdog.sh`)

System health monitor that runs every 5 minutes and auto-recovers from failures.

**Monitors:**
- Network connectivity (ping, Ethernet/Wi-Fi failover)
- Internet Sharing daemon
- Gateway process health
- Messages.app (required for iMessage)
- Tailscale connectivity
- Disk space, CPU load, memory, swap, thermal throttling
- Dist patch integrity (re-applies if npm update stripped them)

**Alert chain:** iMessage first → Slack fallback → both for critical issues

### Custom CLI Tools (`bin/`)

| Tool | Purpose |
|------|---------|
| `notes` | Read/write/search Apple Notes |
| `findme` | iCloud Find My locations (people + devices) |
| `gog` | Gmail, Calendar, Drive operations |
| `imsg` | iMessage send/read |
| `sonos` | Sonos speaker control |
| `spam-filter` | Email blocklist/allowlist management |

These tools extend what the agent can do — they're registered in the agent's tool allowlist so it can invoke them during conversations.

---

## The Fork: What Changed

The upstream OpenClaw npm package ships as minified JavaScript bundles. We've forked the source repository to maintain patches properly in TypeScript.

### Patches Applied (branch: `moshik/patches`)

| # | File | What | Why |
|---|------|------|-----|
| 1 | `deliver.ts` | Strip markdown for iMessage | iMessage renders markdown as literal `**text**` — looks ugly |
| 2 | `markdown-to-line.ts` | Strip `[[annotation]]` tags | Internal tags leak into user-visible messages |
| 3 | `failover-matches.ts` | Add billing error patterns | Anthropic billing errors weren't detected, causing failed retries |
| 4 | `errors.ts` | Reorder billing vs rate-limit checks + add logging | Billing errors were misclassified as rate limits, hiding the real issue |
| 5 | `lifecycle.ts` | Add `[FAILOVER-RAW]` logging | Raw provider errors were masked — impossible to debug failover |
| 6 | `agent-runner-payloads.ts` | Fix duplicate message dedup | Same reply sent twice when inbound phone ≠ iMessage target |
| 7 | `model-catalog.ts` | Register Gemini 3.1 Flash Lite | New model not in upstream registry yet |
| 8 | `send.ts` | Disable reply tag prepending | `imsg` v0.5.0 doesn't parse `[[reply_to:...]]` — tags leak verbatim |

### Managing the Fork Going Forward

**To sync with upstream:**
```bash
cd /tmp/openclaw-fork  # or wherever you clone it
git fetch upstream
git merge upstream/main
# Resolve conflicts in patched files if any
# Build and test
```

**To build from source:**
```bash
pnpm install
pnpm build  # produces dist/ with patches baked in
```

**To install your fork instead of npm:**
```bash
# Link globally from your built fork
npm link /path/to/openclaw-fork
# Or publish to a private registry
```

---

## How to Think About Improvements

### What's working well
- **Event-driven email triage** — emails are classified and triaged within seconds of arrival
- **iMessage as primary channel** — natural, low-friction interaction
- **Auto-recovery** — watchdog catches and fixes most failures automatically
- **Model failover** — transparent switching when a provider is down

### Areas to explore
- **More channels** — WhatsApp and Telegram are configured but not active
- **Richer tools** — the agent can call CLI tools but could do more (home automation, financial data, etc.)
- **Proactive behaviors** — beyond cron jobs, the system could react to more events (calendar changes, location triggers, etc.)
- **Voice** — OpenClaw supports voice-call extensions
- **Multi-agent routing** — different agents for different tasks (coding agent, research agent, personal agent)
- **Session management** — currently one main session; could have separate sessions per topic/project
- **Plugin ecosystem** — OpenClaw has a plugin SDK and marketplace (ClawHub) that's mostly untapped

### Architectural constraints to be aware of
- **Agent context is ephemeral** — the Gemini Flash Lite model has no persistent state between runs. Everything must be fed in via context assembly. Don't rely on the agent "remembering" — use memory search or explicit context.
- **Dist patches are fragile** — upstream updates can break patches. The fork solves this by patching source, but you'll need to resolve merge conflicts when syncing.
- **macOS dependency** — iMessage integration requires macOS + Messages.app. The gateway itself is cross-platform, but this installation is Mac-specific.
- **Single machine** — everything runs on one Mac Mini. No redundancy. If it's offline, the assistant is offline.
