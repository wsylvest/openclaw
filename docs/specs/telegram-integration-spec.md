# Telegram Bot Integration: Implementation Spec & Prompt Plan

> A reusable specification for integrating a Telegram bot into any TypeScript/Node.js project.
> Derived from production patterns in the OpenClaw gateway.

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Implementation Phases](#implementation-phases)
4. [Phase 1: Bot Bootstrapping](#phase-1-bot-bootstrapping)
5. [Phase 2: Inbound Message Pipeline](#phase-2-inbound-message-pipeline)
6. [Phase 3: Outbound Sending](#phase-3-outbound-sending)
7. [Phase 4: Access Control](#phase-4-access-control)
8. [Phase 5: Connectivity (Polling & Webhook)](#phase-5-connectivity-polling--webhook)
9. [Phase 6: Error Handling & Resilience](#phase-6-error-handling--resilience)
10. [Phase 7: Advanced Features](#phase-7-advanced-features)
11. [Phase 8: Onboarding & Configuration](#phase-8-onboarding--configuration)
12. [Phase 9: Testing](#phase-9-testing)
13. [Configuration Schema Reference](#configuration-schema-reference)
14. [File Structure Reference](#file-structure-reference)
15. [Prompt Templates](#prompt-templates)

---

## Overview

### What This Spec Covers

A production-grade Telegram bot integration that supports:

- **Dual connectivity**: long-polling (default) and webhook modes
- **Multi-account**: multiple bot tokens bound to different internal agents/tenants
- **DM and group chat**: separate policies, mention gating, per-group overrides
- **Forum topics**: isolated sessions per topic thread
- **Media handling**: photos, documents, audio, voice, stickers, locations
- **Message buffering**: text fragment coalescing, media group assembly, inbound debouncing
- **Streaming responses**: real-time draft message updates during generation
- **Inline keyboards**: callback buttons with scoped visibility
- **Reactions**: receive and send emoji reactions
- **Rate limiting & retry**: exponential backoff, jitter, recoverable error classification
- **Onboarding wizard**: guided setup for token, access policy, and allowed users

### Tech Stack Assumptions

| Concern | Choice | Notes |
|---------|--------|-------|
| Language | TypeScript (ESM) | Strict typing, no `any` |
| Runtime | Node 22+ / Bun | Both supported |
| Telegram SDK | grammY | Type-safe, middleware-based |
| Polling runner | @grammyjs/runner | Per-chat sequencing |
| Rate limiting | @grammyjs/transformer-throttler | API call rate management |
| Testing | Vitest | Colocated `*.test.ts` |

---

## Architecture

### High-Level Flow

```
┌─────────────────────────────────────────────────────────┐
│                    Telegram Bot API                       │
└──────────────┬──────────────────────────┬───────────────┘
               │ inbound updates          │ outbound calls
               ▼                          ▲
┌──────────────────────────┐  ┌───────────────────────────┐
│  Connectivity Layer       │  │  Send Layer                │
│  (polling OR webhook)     │  │  (text, media, reactions)  │
│  monitor.ts / webhook.ts  │  │  send.ts                   │
└──────────┬───────────────┘  └───────────▲───────────────┘
           │                              │
           ▼                              │
┌──────────────────────────┐              │
│  Handler Layer            │              │
│  (buffering, debouncing)  │              │
│  bot-handlers.ts          │              │
└──────────┬───────────────┘              │
           │                              │
           ▼                              │
┌──────────────────────────┐              │
│  Context Builder          │              │
│  (normalize, enrich)      │              │
│  message-context.ts       │              │
└──────────┬───────────────┘              │
           │                              │
           ▼                              │
┌──────────────────────────┐              │
│  Access Control           │              │
│  (pairing, allowlists)    │              │
│  access.ts                │              │
└──────────┬───────────────┘              │
           │                              │
           ▼                              │
┌──────────────────────────────────────────┐
│  Application Core                         │
│  (routing, agent dispatch, session mgmt)  │
└──────────────────────────────────────────┘
```

### Key Design Principles

1. **Per-chat sequencing** — Process messages for a single chat serially to prevent race conditions. Use grammY runner's `sequentialize` middleware keyed on chat ID.
2. **Layered buffering** — Absorb Telegram's message fragmentation (albums, multi-part text) at the handler layer so downstream code receives complete payloads.
3. **Cascading configuration** — Base config merges with per-account, per-group, and per-topic overrides.
4. **Context-aware error handling** — Classify errors by context (polling vs. send vs. webhook) with different recovery strategies.
5. **Plugin isolation** — The Telegram module registers through a channel plugin interface; it does not import application core directly.

---

## Implementation Phases

| Phase | Scope | Priority |
|-------|-------|----------|
| 1 | Bot bootstrapping & token resolution | P0 |
| 2 | Inbound message pipeline | P0 |
| 3 | Outbound sending (text, media) | P0 |
| 4 | Access control (DM/group policies) | P0 |
| 5 | Connectivity (polling + webhook) | P0 |
| 6 | Error handling & resilience | P0 |
| 7 | Advanced features (streaming, stickers, buttons, reactions) | P1 |
| 8 | Onboarding & configuration wizard | P1 |
| 9 | Testing | P0 (parallel with each phase) |

---

## Phase 1: Bot Bootstrapping

### Goal

Create and configure a grammY `Bot` instance with middleware, token resolution, and multi-account support.

### Token Resolution

Implement a cascading token lookup:

```typescript
type TokenSource = "config" | "tokenFile" | "env" | "none";

type ResolvedAccount = {
  accountId: string;
  enabled: boolean;
  name?: string;
  token: string;
  tokenSource: TokenSource;
  config: TelegramAccountConfig;
};

// Priority: config.botToken > config.tokenFile > env TELEGRAM_BOT_TOKEN
function resolveAccount(cfg: Config, accountId?: string): ResolvedAccount;
```

**Rules:**
- Environment variable (`TELEGRAM_BOT_TOKEN`) only applies to the default account
- `tokenFile` reads a path (for secret managers / mounted volumes)
- Config value takes highest priority
- If no token resolves, mark account as disabled

### Bot Factory

```typescript
function createBot(token: string, options?: {
  clientOptions?: ApiClientOptions;  // proxy, timeout, custom fetch
}): Bot<BotContext> {
  const bot = new Bot<BotContext>(token, { client: clientOptions });
  bot.api.config.use(apiThrottler());  // rate limiting
  return bot;
}
```

### Middleware Stack (install order matters)

1. **Sequentialize** — `bot.use(sequentialize(getSequentialKey))`
2. **Throttler** — `bot.api.config.use(apiThrottler())`
3. **Update deduplication** — Drop updates with already-seen `update_id`
4. **Handlers** — Register message/callback/reaction handlers

### Sequential Key Function

```typescript
function getSequentialKey(ctx: Context): string {
  const chatId = ctx.chat?.id
    ?? ctx.callbackQuery?.message?.chat?.id
    ?? ctx.messageReaction?.chat?.id;
  if (!chatId) return "unknown";

  // Fast-track control commands (e.g., /status, /reset)
  if (isControlCommand(ctx)) return `telegram:${chatId}:control`;

  return `telegram:${chatId}`;
}
```

### Files to Create

```
src/telegram/
  bot.ts              — Bot factory, middleware, sequential key
  accounts.ts         — Token resolution, multi-account listing
  token.ts            — Token file reading, env lookup
  types.ts            — Shared types (BotContext, ResolvedAccount, etc.)
```

---

## Phase 2: Inbound Message Pipeline

### Goal

Receive Telegram updates, buffer fragmented input, build a normalized message context, and hand off to the application core.

### Handler Registration

```typescript
function registerHandlers(params: {
  bot: Bot<BotContext>;
  config: Config;
  accountId: string;
  processMessage: (ctx: NormalizedInbound) => Promise<void>;
}): void;
```

Register handlers for these update types:
- `message` — text, media, forwarded, location, contact, stickers
- `edited_message` — treat as new message with edit flag
- `callback_query` — inline button presses
- `message_reaction` — emoji reaction changes

### Three-Layer Buffering

#### Layer 1: Media Group Buffer

Telegram sends album items as separate updates sharing a `media_group_id`. Buffer them:

```typescript
type MediaGroupEntry = {
  messages: TelegramMessage[];
  timer: ReturnType<typeof setTimeout>;
};

const mediaGroupBuffer = new Map<string, MediaGroupEntry>();
const MEDIA_GROUP_TIMEOUT_MS = 500;
```

On each message with `media_group_id`:
1. Add to buffer map keyed by `media_group_id`
2. Reset the timer
3. On timeout, flush all buffered messages as one payload

#### Layer 2: Text Fragment Buffer

Users sometimes send a long message that Telegram splits, or type multiple short messages in quick succession:

```typescript
type TextFragmentEntry = {
  parts: string[];
  totalChars: number;
  timer: ReturnType<typeof setTimeout>;
};

const TEXT_FRAGMENT_LIMITS = {
  maxChars: 4000,
  maxGapMs: 1500,
  maxParts: 12,
  maxTotalChars: 50_000,
};
```

Flush when any limit is exceeded or the gap timer fires.

#### Layer 3: Inbound Debouncer

A final debounce layer that coalesces rapid-fire messages per chat:

```typescript
type DebouncerEntry = {
  chatId: number;
  messages: BufferedMessage[];
  timer: ReturnType<typeof setTimeout>;
};
```

Skip debounce if:
- Message contains media
- Message is a control command (`/status`, `/reset`, etc.)
- Debounce interval is 0 (disabled)

### Context Building

After buffering, build a normalized inbound context:

```typescript
type NormalizedInbound = {
  channelId: "telegram";
  accountId: string;
  chatId: number;
  chatType: "private" | "group" | "supergroup";
  threadId?: number;              // forum topic
  senderId: number;
  senderUsername?: string;
  senderDisplayName: string;
  text: string;                   // combined/buffered text
  media: MediaAttachment[];       // downloaded files
  replyToMessageId?: number;
  isEdit: boolean;
  isForwarded: boolean;
  forwardedFrom?: string;
  location?: { lat: number; lon: number };
  callbackData?: string;          // inline button press
  raw: TelegramMessage;           // preserve original
};
```

**Mention matching:** In groups with `requireMention: true`, check if the message mentions the bot by `@username`, display name, or configured mention patterns. Skip processing if no mention found.

### Files to Create

```
src/telegram/
  bot-handlers.ts       — Handler registration, buffering layers
  message-context.ts    — Context building, mention matching
  media.ts              — Media download (getFile API), MIME detection
  sticker.ts            — Sticker download, vision description cache
```

---

## Phase 3: Outbound Sending

### Goal

Send text, media, reactions, and interactive elements to Telegram chats.

### Primary Send Function

```typescript
type SendOptions = {
  token?: string;
  accountId?: string;
  chatId: number | string;
  text?: string;
  mediaUrl?: string;
  mediaType?: "photo" | "document" | "audio" | "voice" | "sticker" | "animation";
  replyToMessageId?: number;
  messageThreadId?: number;       // forum topic
  buttons?: InlineKeyboardButton[][];
  silent?: boolean;               // disable notification
  textMode?: "markdown" | "html";
  retry?: RetryConfig;
};

type SendResult = {
  messageId: number;
  chatId: number;
};

async function sendMessage(opts: SendOptions): Promise<SendResult>;
```

### Text Processing Pipeline

1. **Render Markdown to Telegram HTML** — Convert a safe subset (bold, italic, strikethrough, code, pre, links). Telegram's HTML parser is strict; unsupported tags cause `400 Bad Request`.
2. **Chunk by limit** — Default 4096 chars. Two modes:
   - `"length"`: split at char limit
   - `"newline"`: split at last newline before limit
3. **Send each chunk** — Sequential sends; first chunk gets reply threading and buttons.
4. **Parse error fallback** — If Telegram rejects HTML, retry as plain text (strip all tags).

```typescript
function renderToTelegramHtml(markdown: string): string;
function chunkText(text: string, limit: number, mode: "length" | "newline"): string[];
```

### Media Sending

| Telegram Method | When to Use |
|-----------------|-------------|
| `sendPhoto` | Image files (JPEG, PNG, GIF < 10MB) |
| `sendDocument` | Any file, or images > 10MB |
| `sendAudio` | Audio files (MP3, etc.) |
| `sendVoice` | OGG/Opus voice notes |
| `sendSticker` | WEBP/WEBM stickers by file_id |
| `sendAnimation` | GIF animations |

For media with text:
- Attach text as `caption` (max 1024 chars)
- If text exceeds caption limit, send media first, then text as separate message

### Caption Splitting

```typescript
function splitCaption(text: string, limit?: number): {
  caption: string;
  overflow: string | null;
};
```

### Reply Threading

Three modes controlled by config:

| Mode | Behavior |
|------|----------|
| `"off"` | Never set `reply_to_message_id` |
| `"first"` | Reply to the triggering message only (first chunk) |
| `"all"` | Every chunk replies to the triggering message |

Special tags in message text:
- `[[reply_to_current]]` — reply to the triggering message
- `[[reply_to:<id>]]` — reply to a specific message ID

### Forum Topic Handling

When sending to a forum-enabled group:
- Include `message_thread_id` for all non-General topics
- **General topic (id=1)**: omit `message_thread_id` from sends (Telegram rejects it), but include it for `sendChatAction` (typing indicator)

### Reactions

```typescript
async function setReaction(opts: {
  chatId: number;
  messageId: number;
  emoji: string;
  remove?: boolean;
  token?: string;
  accountId?: string;
}): Promise<void>;
```

Only works on bot-sent messages (Telegram API limitation for bots). Track sent message IDs in an in-memory cache to know which messages the bot can react to.

### Files to Create

```
src/telegram/
  send.ts               — Primary send function, text chunking, media dispatch
  format.ts             — Markdown-to-Telegram-HTML renderer
  caption.ts            — Caption splitting logic
  reactions.ts          — Reaction set/remove
  sent-message-cache.ts — Track bot-sent message IDs
```

---

## Phase 4: Access Control

### Goal

Gate inbound messages through configurable DM and group policies.

### DM Policy

```typescript
type DmPolicy = "pairing" | "allowlist" | "open" | "disabled";
```

| Policy | Behavior |
|--------|----------|
| `pairing` | Unknown users receive a time-limited pairing code. Owner approves via CLI. |
| `allowlist` | Only users in `allowFrom` (numeric IDs or `@username`) can DM. |
| `open` | All users can DM. Requires `allowFrom: ["*"]` as explicit opt-in. |
| `disabled` | Block all DMs silently. |

### Group Policy

```typescript
type GroupPolicy = "allowlist" | "open" | "disabled";
```

| Policy | Behavior |
|--------|----------|
| `allowlist` | Only users in `groupAllowFrom` or `allowFrom` can trigger the bot. |
| `open` | Any group member can trigger (mention gating still applies). |
| `disabled` | Ignore all group messages. |

### Allowlist Matching

```typescript
function isAllowed(senderId: number, senderUsername: string | undefined, allowList: Array<string | number>): boolean;
```

Match against:
- Numeric user ID (exact match)
- `@username` (case-insensitive)
- `"*"` wildcard (allow all)
- `tg:` prefixed entries (channel-specific)

### Pairing Flow

1. Unknown user sends DM
2. Bot generates a 6-character alphanumeric code, valid for 1 hour
3. Bot replies: "Send this code to the gateway owner for approval: `ABC123`"
4. Owner runs `cli pairing approve telegram ABC123`
5. Pairing store persists user ID to `allowFrom`
6. Bot confirms: "Pairing approved"

```typescript
type PairingEntry = {
  code: string;
  senderId: number;
  senderUsername?: string;
  createdAt: number;
  expiresAt: number;
};

// Persist to JSON file
const PAIRING_STORE_PATH = "~/.appdata/telegram/pairing-store.json";
```

### Per-Group Overrides

Each group can override the base config:

```typescript
type GroupConfig = {
  enabled?: boolean;
  requireMention?: boolean;     // default: true
  allowFrom?: Array<string | number>;
  systemPrompt?: string;
  skills?: string[];
  topics?: Record<string, TopicConfig>;
};
```

Resolution order: topic config > group config > wildcard group (`"*"`) > base channel config.

### Files to Create

```
src/telegram/
  access.ts          — DM/group policy enforcement, allowlist matching
  pairing-store.ts   — Pairing code generation, persistence, approval
```

---

## Phase 5: Connectivity (Polling & Webhook)

### Goal

Support both long-polling (simple, default) and webhook (low-latency, scalable) connectivity modes.

### Long-Polling (Default)

Use `@grammyjs/runner` for managed polling with per-chat concurrency:

```typescript
import { run } from "@grammyjs/runner";

function startPolling(bot: Bot, config: Config): RunnerHandle {
  return run(bot, {
    runner: {
      fetch: {
        timeout: 30,
        allowed_updates: ["message", "edited_message", "callback_query", "message_reaction"],
      },
      silent: true,
      maxRetryTime: 5 * 60 * 1000,   // 5 min max retry window
      retryInterval: "exponential",
    },
    sink: {
      concurrency: config.maxConcurrent ?? 10,
    },
  });
}
```

**Update offset persistence:**

```typescript
// Persist last processed update_id to resume after restart
type UpdateOffsetStore = {
  read(): Promise<number | null>;
  write(updateId: number): Promise<void>;
};
```

Store at `~/.appdata/telegram/update-offset-<accountId>.json`.

### Webhook Mode

```typescript
async function startWebhook(opts: {
  bot: Bot;
  publicUrl: string;
  path?: string;        // default: "/telegram-webhook"
  port?: number;        // default: 8787
  host?: string;        // default: "0.0.0.0"
  secret?: string;      // recommended: validate X-Telegram-Bot-Api-Secret-Token
}): Promise<{ server: http.Server; stop: () => Promise<void> }>;
```

Steps:
1. Create HTTP server
2. Register grammY `webhookCallback` handler at `opts.path`
3. Add health endpoint at `/healthz`
4. Call `bot.api.setWebhook(publicUrl, { secret_token, allowed_updates })`
5. Return server handle and stop function

**Secret token validation:** Telegram sends `X-Telegram-Bot-Api-Secret-Token` header. grammY's `webhookCallback` validates it automatically.

### Monitor Entrypoint

A single entrypoint that decides polling vs. webhook:

```typescript
async function monitor(opts: MonitorOptions): Promise<void> {
  const account = resolveAccount(opts.config, opts.accountId);
  if (!account.token) return; // disabled

  if (opts.config.telegram?.webhookUrl) {
    await startWebhook({ ... });
  } else {
    await startPolling({ ... });
  }
}
```

### Files to Create

```
src/telegram/
  monitor.ts              — Entrypoint, polling vs webhook decision
  webhook.ts              — Webhook HTTP server, setWebhook call
  update-offset-store.ts  — Persist/resume update offset
```

---

## Phase 6: Error Handling & Resilience

### Goal

Classify errors, implement retry policies, and ensure the bot recovers from transient failures.

### Error Classification

```typescript
type ErrorContext = "polling" | "send" | "webhook";

function isRecoverableNetworkError(err: unknown, context?: ErrorContext): boolean;
```

**Recoverable error codes:** `ECONNRESET`, `EPIPE`, `ETIMEDOUT`, `ENOTFOUND`, `ECONNREFUSED`, `UND_ERR_CONNECT_TIMEOUT`, `UND_ERR_SOCKET`, `UND_ERR_BODY_TIMEOUT`

**Recoverable error names:** `AbortError`, `TimeoutError`, `ConnectTimeoutError`, `BodyTimeoutError`

**Recoverable message patterns:** `"fetch failed"`, `"network error"`, `"socket hang up"`, `"getaddrinfo"`, `"EHOSTUNREACH"`

**Recursive traversal:** Walk `error.cause`, `error.reason`, and `error.errors[]` to find nested root causes.

### Retry Configuration

```typescript
type RetryConfig = {
  attempts: number;       // default: 3
  minDelayMs: number;     // default: 100
  maxDelayMs: number;     // default: 10_000
  jitter: number;         // default: 0.1 (10% randomness)
};
```

### Backoff Policy (polling recovery)

```typescript
const POLLING_BACKOFF = {
  initialMs: 2000,
  maxMs: 30_000,
  factor: 1.8,
  jitter: 0.25,
};
```

### Specific Error Handling

| Error | Context | Action |
|-------|---------|--------|
| `409 Conflict` (getUpdates) | Polling | Fatal — another instance is polling with same token. Log error, retry with increasing backoff. |
| `429 Too Many Requests` | Send | Respect `retry_after` from Telegram response. grammY throttler handles this automatically. |
| HTML parse error (`400`) | Send | Retry same message as plain text (strip HTML tags). |
| `VOICE_MESSAGES_FORBIDDEN` | Send | Skip voice, do not fallback to audio. |
| Network timeout | Polling | Automatic retry via grammY runner (up to 5 min). |
| Network timeout | Send | Retry per `RetryConfig`, then fail. |

### Custom Fetch Wrapper

Handle IPv4/IPv6 selection and proxy support:

```typescript
function createTelegramFetch(options?: {
  proxy?: string;
  autoSelectFamily?: boolean;
  timeoutMs?: number;
}): typeof fetch;
```

Node 22+ introduced Happy Eyeballs (tries IPv6 first with 300ms fallback to IPv4). Some environments need this disabled. Expose as config: `network.autoSelectFamily`.

### Files to Create

```
src/telegram/
  network-errors.ts   — Error classification, recursive traversal
  fetch.ts            — Custom fetch wrapper (proxy, IPv4/IPv6)
  network-config.ts   — Network settings resolution
  proxy.ts            — Proxy fetch builder (SOCKS/HTTP)
```

---

## Phase 7: Advanced Features

### 7a: Draft Streaming (Real-Time Response Preview)

Show partial responses as the agent generates them:

```typescript
type DraftStream = {
  update(text: string): void;
  flush(): Promise<void>;
  stop(): void;
};

function createDraftStream(params: {
  api: Bot["api"];
  chatId: number;
  draftId: number;            // message ID of the draft bubble
  maxChars?: number;          // default: 4096
  throttleMs?: number;        // default: 300
  messageThreadId?: number;
}): DraftStream;
```

**Flow:**
1. Send initial empty/placeholder message → get `draftId`
2. On each generation chunk, call `stream.update(partialText)`
3. Throttle API calls to avoid rate limits (min 300ms between edits)
4. Deduplicate: skip if text hasn't changed since last send
5. On completion, `stream.stop()` and send final message (replaces draft)

**Stream modes (configurable):**
- `"partial"` — Update on every chunk (live preview)
- `"block"` — Update in larger blocks (fewer API calls)
- `"off"` — Disable streaming, only send final message

**Constraints:**
- Only in private chats (group draft edits are noisy)
- Max 4096 chars per edit (Telegram limit)

### 7b: Inline Keyboards

```typescript
type InlineButton = {
  text: string;
  callback_data: string;   // max 64 chars
};

// Scope control for when buttons are shown
type ButtonScope = "off" | "dm" | "group" | "all" | "allowlist";
```

Callback handling: When a user presses a button, Telegram sends a `callback_query` update. Extract `callback_data` and process as a message with metadata indicating it was a button press.

### 7c: Sticker Cache with Vision

For receiving stickers with meaningful context:

1. Download static sticker (WEBP format) via `bot.api.getFile`
2. Send to vision model (OpenAI, Anthropic, etc.) for description
3. Cache description keyed by `file_unique_id`
4. On future receipt, look up cached description
5. Provide fuzzy search for agents to find and send stickers

```typescript
type StickerCacheEntry = {
  fileId: string;
  fileUniqueId: string;
  emoji?: string;
  setName?: string;
  description: string;
  cachedAt: number;
};
```

### 7d: Reactions

```typescript
type ReactionLevel = "off" | "ack" | "minimal" | "extensive";
type ReactionNotifications = "off" | "own" | "all";
```

- `"ack"`: Send eyes emoji while processing, remove after reply
- `"minimal"`: Agent can react sparingly
- `"extensive"`: Agent reacts when appropriate

Track bot-sent messages in an in-memory LRU cache so reactions to non-bot messages can be filtered.

### 7e: Native Commands

Auto-register slash commands with BotFather:

```typescript
const NATIVE_COMMANDS = [
  { command: "status", description: "Show bot status" },
  { command: "reset", description: "Reset conversation" },
  { command: "model", description: "Switch AI model" },
  { command: "help", description: "Show help" },
];

// Register on startup
await bot.api.setMyCommands(NATIVE_COMMANDS);
```

### 7f: Group Migration Handling

Telegram may migrate groups to supergroups, changing the chat ID:

```typescript
bot.on("message:migrate_to_chat_id", async (ctx) => {
  const oldId = ctx.chat.id;
  const newId = ctx.message.migrate_to_chat_id;
  await migrateGroupConfig(oldId, newId);
});
```

### Files to Create

```
src/telegram/
  draft-stream.ts         — Streaming draft updates
  inline-buttons.ts       — Button scope resolution
  sticker-cache.ts        — Vision-based sticker descriptions
  reactions.ts            — Reaction level gating
  native-commands.ts      — Slash command registration
  group-migration.ts      — Chat ID migration handling
```

---

## Phase 8: Onboarding & Configuration

### Goal

Guide users through bot setup with a wizard.

### Wizard Steps

1. **Token prompt** — Ask for BotFather token. Offer env var option for default account.
2. **Token verification** — Call `bot.api.getMe()` to validate token, display bot username.
3. **DM policy selection** — Choose from: pairing (recommended), allowlist, open, disabled.
4. **User allowlist** — If allowlist or open, prompt for user IDs or @usernames.
5. **Username resolution** — Resolve `@username` to numeric ID via `bot.api.getChat("@username")`.
6. **Save config** — Write to config file at appropriate nesting level.
7. **Privacy mode warning** — If groups are enabled, warn about BotFather `/setprivacy` setting.

### Helper Guidance Text

Provide inline help for common questions:
- "How do I get a bot token?" → Link to @BotFather, steps: `/newbot`, copy token
- "How do I find my user ID?" → Start the bot, check logs for `from.id`, or use @userinfobot
- "What is privacy mode?" → Explanation of BotFather `/setprivacy` and its effect on group messages

### Files to Create

```
src/telegram/
  onboarding.ts   — Wizard prompts, token verification, config writing
```

---

## Phase 9: Testing

### Strategy

| Level | Scope | Approach |
|-------|-------|----------|
| Unit | Token resolution, text chunking, HTML rendering, error classification | Pure function tests, no mocking |
| Integration | Handler buffering, context building, send pipeline | Mock grammY Bot API |
| E2E | Full polling → handle → reply cycle | Mock Telegram HTTP API |

### Key Test Scenarios

**Bot initialization:**
- Multi-account token resolution (config > env > file)
- Disabled account skipping
- Middleware installation order

**Inbound handling:**
- Text fragment buffering (combine 3 short messages)
- Media group assembly (album of 4 photos)
- Debounce timer reset
- Mention matching (exact, pattern, case-insensitive)
- Forum topic thread isolation
- Callback query deduplication

**Outbound sending:**
- Text chunking at limit boundary
- Newline-mode chunking
- HTML parse error fallback to plain text
- Caption splitting for media
- Reply threading modes (off, first, all)
- Forum General topic edge case (thread_id=1)
- Button attachment to first chunk only

**Access control:**
- DM policy enforcement (all four modes)
- Group policy enforcement (all three modes)
- Allowlist matching (numeric ID, @username, wildcard, tg: prefix)
- Pairing code generation, expiry, approval

**Error handling:**
- Recoverable error classification (all code/name/message patterns)
- Recursive cause traversal
- Retry with backoff
- 409 conflict detection
- 429 rate limit respect

**Streaming:**
- Throttle interval enforcement
- Deduplication (same text skipped)
- Max char truncation
- Stop signal handling

### Test File Naming

```
src/telegram/
  bot.test.ts                     — Bot factory, middleware
  accounts.test.ts                — Token resolution
  bot-handlers.test.ts            — Buffering, debouncing
  message-context.test.ts         — Context building, mentions
  send.test.ts                    — Text/media sending
  send.caption.test.ts            — Caption splitting
  send.threading.test.ts          — Reply threading
  format.test.ts                  — HTML rendering
  access.test.ts                  — Policy enforcement
  pairing-store.test.ts           — Pairing flow
  network-errors.test.ts          — Error classification
  draft-stream.test.ts            — Streaming throttle
  sticker-cache.test.ts           — Vision cache
  webhook.test.ts                 — Webhook server
```

---

## Configuration Schema Reference

```typescript
type TelegramConfig = TelegramAccountConfig & {
  accounts?: Record<string, TelegramAccountConfig>;
};

type TelegramAccountConfig = {
  // === Identity ===
  name?: string;
  enabled?: boolean;                  // default: true
  botToken?: string;
  tokenFile?: string;

  // === Access Control ===
  dmPolicy?: "pairing" | "allowlist" | "open" | "disabled";    // default: "pairing"
  groupPolicy?: "allowlist" | "open" | "disabled";             // default: "allowlist"
  allowFrom?: Array<string | number>;
  groupAllowFrom?: Array<string | number>;

  // === Messaging ===
  textChunkLimit?: number;            // default: 4096
  chunkMode?: "length" | "newline";   // default: "length"
  replyToMode?: "off" | "first" | "all";  // default: "first"

  // === Streaming ===
  streamMode?: "off" | "partial" | "block";  // default: "partial"
  draftThrottleMs?: number;           // default: 300

  // === Media ===
  mediaMaxMb?: number;                // default: 5

  // === Reactions ===
  reactionNotifications?: "off" | "own" | "all";      // default: "off"
  reactionLevel?: "off" | "ack" | "minimal" | "extensive";  // default: "ack"

  // === Networking ===
  proxy?: string;                     // SOCKS/HTTP proxy URL
  timeoutSeconds?: number;            // API call timeout
  network?: {
    autoSelectFamily?: boolean;       // Node 22+ Happy Eyeballs
  };
  retry?: {
    attempts?: number;                // default: 3
    minDelayMs?: number;              // default: 100
    maxDelayMs?: number;              // default: 10_000
    jitter?: number;                  // default: 0.1
  };

  // === Webhook ===
  webhookUrl?: string;
  webhookSecret?: string;
  webhookPath?: string;               // default: "/telegram-webhook"

  // === Groups ===
  groups?: Record<string, {
    enabled?: boolean;
    requireMention?: boolean;         // default: true
    allowFrom?: Array<string | number>;
    systemPrompt?: string;
    skills?: string[];
    topics?: Record<string, {
      enabled?: boolean;
      requireMention?: boolean;
      allowFrom?: Array<string | number>;
      systemPrompt?: string;
      skills?: string[];
    }>;
  }>;

  // === History ===
  historyLimit?: number;
  dmHistoryLimit?: number;

  // === Display ===
  linkPreview?: boolean;              // default: true
};
```

---

## File Structure Reference

```
src/telegram/
  # Core
  bot.ts                    — Bot factory, middleware stack, sequential key
  accounts.ts               — Multi-account resolution, config merging
  token.ts                  — Token file/env lookup
  types.ts                  — Shared TypeScript types

  # Inbound
  bot-handlers.ts           — Handler registration, 3-layer buffering
  message-context.ts        — Normalize inbound messages, mention matching
  media.ts                  — File download, MIME detection
  sticker-cache.ts          — Vision-based sticker descriptions

  # Outbound
  send.ts                   — Text/media sending, chunking, fallbacks
  format.ts                 — Markdown → Telegram HTML
  caption.ts                — Caption splitting
  reactions.ts              — Set/remove reactions
  sent-message-cache.ts     — Track bot-sent message IDs
  draft-stream.ts           — Streaming response drafts
  inline-buttons.ts         — Button scope resolution

  # Connectivity
  monitor.ts                — Entrypoint (polling vs webhook)
  webhook.ts                — HTTP server, setWebhook
  update-offset-store.ts    — Persist update offset

  # Resilience
  network-errors.ts         — Error classification
  fetch.ts                  — Custom fetch (proxy, IPv4/IPv6)
  network-config.ts         — Network settings
  proxy.ts                  — Proxy builder

  # Access Control
  access.ts                 — DM/group policy enforcement
  pairing-store.ts          — Pairing code persistence

  # Features
  native-commands.ts        — Slash command registration
  group-migration.ts        — Chat ID migration

  # Setup
  onboarding.ts             — Guided setup wizard

  # Tests (colocated)
  *.test.ts                 — One per source file
```

---

## Prompt Templates

These prompts can be used with an AI coding assistant to implement each phase.

### Prompt 1: Bootstrap

```
Implement a Telegram bot bootstrap module using grammY in TypeScript (ESM).

Requirements:
- Create a bot factory function that accepts a token and optional client options
  (proxy, timeout, custom fetch)
- Install grammY apiThrottler for rate limiting
- Implement per-chat sequentialization using @grammyjs/runner's sequentialize
  middleware, keyed on chat ID extracted from the update context
- Fast-track control commands (/status, /reset, etc.) with a separate sequence key
  so they bypass normal message queuing
- Implement multi-account token resolution with cascading priority:
  config.botToken > config.tokenFile (read from disk) > env TELEGRAM_BOT_TOKEN
- The env var fallback only applies to the default account
- Support listing all enabled accounts and resolving a single account by ID
- Add update deduplication middleware that drops updates with already-seen update_id
  (use a bounded Set or LRU)

Type exports needed: ResolvedAccount, TokenSource, BotContext (extending grammY Context).

File structure:
- src/telegram/bot.ts (factory, middleware)
- src/telegram/accounts.ts (token resolution)
- src/telegram/token.ts (file/env lookup)
- src/telegram/types.ts (shared types)
```

### Prompt 2: Inbound Pipeline

```
Implement the inbound message pipeline for a grammY Telegram bot in TypeScript.

Requirements:
- Register handlers for: message, edited_message, callback_query, message_reaction
- Implement 3-layer buffering:
  1. Media group buffer: collect updates sharing media_group_id, flush after 500ms
     timeout with no new items
  2. Text fragment buffer: combine consecutive short text messages from same chat,
     flush when: total chars > 4000, gap > 1500ms, parts > 12, or total > 50K chars
  3. Inbound debouncer: final coalescing per chat, skip if message has media or
     is a control command
- Build a normalized NormalizedInbound context from buffered messages including:
  chatId, chatType, threadId (forum topic), senderId, senderUsername,
  senderDisplayName, combined text, downloaded media array, replyToMessageId,
  isEdit, isForwarded, location, callbackData
- Implement mention matching for group messages: check @username, display name,
  and configurable mention patterns (regex). Skip processing if requireMention
  is true and no mention found.
- Download media files via bot.api.getFile, detect MIME type, save to temp dir.
- Handle stickers: download WEBP for static stickers, skip animated/video.

The registerHandlers function should accept a processMessage callback that receives
the normalized context.
```

### Prompt 3: Outbound Sending

```
Implement outbound message sending for a Telegram bot using grammY in TypeScript.

Requirements:
- Primary sendMessage function accepting: chatId, text, mediaUrl, mediaType,
  replyToMessageId, messageThreadId, buttons, silent flag, textMode, retry config
- Markdown to Telegram HTML renderer supporting: bold, italic, strikethrough,
  inline code, code blocks, links. Must produce valid Telegram HTML only.
- Text chunking with two modes:
  - "length": split at char limit (default 4096)
  - "newline": split at last newline before limit
- Parse error fallback: if Telegram returns 400 for HTML, retry as plain text
- Caption splitting: if text > 1024 chars for media, send media with truncated
  caption then overflow as separate text message
- Reply threading modes: "off" (never reply), "first" (reply on first chunk only),
  "all" (reply on every chunk)
- Forum topic handling: include message_thread_id for non-General topics;
  omit for General topic (id=1) on sends but include on typing actions
- Inline keyboard buttons attached to first chunk only
- Media type routing: sendPhoto for images, sendDocument for files, sendAudio
  for audio, sendVoice for voice notes, sendSticker for stickers
- Return { messageId, chatId } on success
```

### Prompt 4: Access Control

```
Implement access control for a Telegram bot in TypeScript.

Requirements:
- DM policies: "pairing", "allowlist", "open", "disabled"
  - pairing: generate 6-char alphanumeric code valid for 1 hour, persist to JSON
    file, owner approves via CLI command
  - allowlist: match sender against allowFrom list
  - open: allow all (requires explicit allowFrom: ["*"])
  - disabled: silently drop
- Group policies: "allowlist", "open", "disabled"
  - allowlist: check groupAllowFrom then allowFrom
  - open: allow any group member (mention gating still applies)
  - disabled: silently drop
- Allowlist matching: numeric user ID (exact), @username (case-insensitive),
  "*" wildcard, "tg:" prefixed entries
- Pairing store: JSON file at configurable path, auto-expire entries,
  approve/reject API
- Per-group config overrides: enabled, requireMention, allowFrom, systemPrompt,
  skills, per-topic overrides
- Config cascade: topic > group > wildcard group ("*") > base channel config
```

### Prompt 5: Connectivity

```
Implement dual connectivity (polling + webhook) for a grammY Telegram bot
in TypeScript.

Requirements:
- Long-polling mode (default):
  - Use @grammyjs/runner with per-chat concurrency
  - Configure: 30s fetch timeout, allowed_updates list, silent mode,
    5 min maxRetryTime, exponential retry interval
  - Persist update offset to JSON file for restart recovery
  - Concurrency cap from config (default 10)
- Webhook mode (opt-in):
  - HTTP server on configurable host:port (default 0.0.0.0:8787)
  - grammY webhookCallback at configurable path (default /telegram-webhook)
  - Secret token validation via X-Telegram-Bot-Api-Secret-Token header
  - Health endpoint at /healthz
  - Call bot.api.setWebhook with public URL, secret, and allowed_updates
  - Graceful shutdown: close server, stop bot
- Monitor entrypoint that resolves account, checks for webhookUrl in config,
  and starts the appropriate mode
- Handle 409 conflict (another polling instance) as fatal with backoff
```

### Prompt 6: Error Handling

```
Implement error handling and resilience for a Telegram bot in TypeScript.

Requirements:
- Error classification function: isRecoverableNetworkError(err, context?)
  - Context: "polling" | "send" | "webhook"
  - Recoverable codes: ECONNRESET, EPIPE, ETIMEDOUT, ENOTFOUND, ECONNREFUSED,
    UND_ERR_CONNECT_TIMEOUT, UND_ERR_SOCKET, UND_ERR_BODY_TIMEOUT
  - Recoverable names: AbortError, TimeoutError, ConnectTimeoutError
  - Message patterns: "fetch failed", "network error", "socket hang up",
    "getaddrinfo", "EHOSTUNREACH"
  - Recursive traversal: walk error.cause, error.reason, error.errors[]
- Retry utility with exponential backoff + jitter
  - Config: attempts (3), minDelayMs (100), maxDelayMs (10000), jitter (0.1)
- Polling backoff: initialMs 2000, maxMs 30000, factor 1.8, jitter 0.25
- Custom fetch wrapper supporting: SOCKS/HTTP proxy, IPv4/IPv6 selection
  (Node 22+ autoSelectFamily override), configurable timeout
- HTTP error logging with sensitive text redaction
```

### Prompt 7: Advanced Features

```
Implement advanced Telegram bot features in TypeScript using grammY.

Requirements:
1. Draft streaming: create a DraftStream object that throttles editMessageText
   calls (min 300ms), deduplicates unchanged text, caps at 4096 chars,
   supports stop/flush. Three modes: partial, block, off.

2. Inline keyboards: button scope control (off, dm, group, all, allowlist),
   callback_data max 64 chars, callback query handling that extracts data
   and processes as a message.

3. Sticker cache: download static WEBP stickers, send to vision API for
   description, cache by file_unique_id in JSON file, fuzzy search on
   description + emoji + set name.

4. Reactions: configurable levels (off, ack, minimal, extensive),
   notification modes (off, own, all), track bot-sent messages in LRU cache,
   ack reaction (eyes emoji) while processing with auto-remove after reply.

5. Native commands: register /status, /reset, /model, /help with BotFather
   via setMyCommands on startup. Custom command support from config.

6. Group migration: handle migrate_to_chat_id events, update group config
   keys from old to new chat ID.
```
