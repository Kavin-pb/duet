# Phase 1 Spec — Core 1:1 Messaging

**Status:** ready-for-agent (pending issue-tracker setup)
**Project:** WhatsApp-like messaging app (learning build)
**Roadmap position:** P1 of 5 (P1 core 1:1 → P2 groups → P3 media → P4 differentiators → P5 scale + E2EE)

---

## Problem Statement

People who want to message each other today must hand over their phone number, trust a closed platform, and accept whatever delivery guarantees the platform feels like giving. For this learning build, the concrete Phase 1 problem is simpler and more fundamental: **two people should be able to find each other by a username and exchange text messages in real time, with the same reliability guarantees they instinctively expect from WhatsApp** — messages never lost, never duplicated, correctly ordered, with visible proof of what happened to each message (sent / delivered / read), full history on any login, and a live sense of whether the other person is around. Everything must keep working on a phone with a flaky connection: sends queue offline, and reconnecting silently repairs the conversation.

## Solution

A mobile app (React Native + Expo) backed by a Python/FastAPI server where a user can:

1. **Register with just a username and password** — no phone number, no email.
2. **Find anyone by username** and start a 1:1 conversation.
3. **Send text messages** that appear instantly on their own screen (optimistic UI) and arrive on the recipient's device in real time over a single persistent WebSocket connection.
4. **See the state of every message** — pending, sent (server accepted), delivered (recipient's device got it), read (recipient opened the chat).
5. **See presence** — "online" or "last seen at …" — for their contacts.
6. **Scroll full history**, paginated, surviving reinstalls and new logins because the server is the source of truth.
7. **Lose connectivity without losing anything** — offline sends queue locally, and on reconnect the app catches up on everything it missed, automatically.

## User Stories

### Registration & authentication

1. As a new user, I want to register with only a username and password, so that I can use the app without exposing my phone number or email.
2. As a new user, I want immediate feedback if my chosen username is taken, so that I can pick another without re-submitting the whole form.
3. As a new user, I want username format rules (3–20 chars, letters/numbers/underscore) validated inline, so that I learn the rules before submitting.
4. As a new user, I want a minimum password strength enforced, so that my account is not trivially hijackable.
5. As a registered user, I want to log in with my username and password, so that I can access my account.
6. As a registered user, I want to stay logged in across app restarts, so that I don't type my password every time.
7. As a logged-in user, I want my session renewed transparently when it expires, so that I'm never abruptly logged out mid-conversation.
8. As a logged-in user, I want to log out, so that my tokens are revoked and my local data is wiped from the device.
9. As a security-conscious user, I want reuse of an old refresh token to kill all my sessions, so that a stolen token can't be exploited twice.
10. As a registered user, I want to set a display name, so that contacts see something nicer than my raw username.

### Discovery & conversations

11. As a user, I want to search for people by username, so that I can start a conversation without knowing their phone number.
12. As a user, I want to start a 1:1 conversation from a search result, so that I can message them immediately.
13. As a user, I want my conversation list ordered by most recent activity, so that my active chats are always on top.
14. As a user, I want each conversation row to show the last message preview and its timestamp, so that I can triage without opening it.
15. As a user, I want an unread count badge per conversation, so that I know what needs attention.
16. As a user, I want opening a conversation to load the most recent messages instantly, so that I'm never staring at a spinner for content I've seen before.
17. As a user, I want older messages to load as I scroll up, so that history is there without a heavy initial load.
18. As a user, I want timestamps in my local timezone with day separators, so that conversations are easy to follow.

### Messaging

19. As a sender, I want my message to appear on my screen the instant I hit send (with a pending indicator), so that the app feels as fast as WhatsApp.
20. As a recipient, I want messages to arrive in real time while I'm in the conversation, so that chat feels live.
21. As a recipient, I want messages arriving while I'm on the conversation list to update the preview and unread badge, so that nothing waits for a manual refresh.
22. As a user, I want messages to always render in the order the server accepted them, so that conversation logic never appears scrambled even on flaky networks.
23. As a sender, I want messages composed while offline to queue and send automatically when connectivity returns, so that I never retype anything.
24. As a sender, I want a clear failed state with tap-to-retry if a message ultimately can't be sent, so that failure is visible and recoverable.
25. As a sender, I want retries to never produce duplicate messages, so that flakiness doesn't spam my contact.
26. As a user, I want empty/whitespace-only messages rejected and a maximum length enforced, so that the data stays sane.

### Receipts

27. As a sender, I want a single tick when the server has accepted my message, so that I know it's safe even before the recipient sees it.
28. As a sender, I want a double tick when the recipient's device has received my message, so that I know it reached them.
29. As a sender, I want the double tick to turn blue (or themed equivalent) when the recipient opens the conversation, so that I know they've seen it.
30. As a sender, I want "delivered" to remain pending while the recipient is offline and complete when they reconnect, so that the receipt always means something true.
31. As a recipient, I want opening a conversation to mark everything visible as read in one batch, so that read state doesn't chatter per message.

### Presence

32. As a user, I want to see "online" under a contact's name while they're connected, so that I know it's a good moment to chat.
33. As a user, I want to see "last seen at …" when they're offline, so that I can gauge when they might reply.
34. As a user, I want presence to not flicker on brief network blips, so that the indicator is trustworthy.

### Sync & resilience

35. As a user on a fresh login or reinstall, I want my full conversation list and history available from the server, so that my account lives in the cloud, not on one device.
36. As a user, I want reconnecting after being offline to automatically fetch everything I missed, so that catch-up needs no manual action.
37. As a user, I want the app to open instantly showing my last-known state from the local cache, so that the app never feels empty while it syncs.

### Errors & edges

38. As a user, I want a clear, non-blocking indicator when the server is unreachable, so that I understand why messages are pending.
39. As a user, I want an expiring access token to be refreshed silently, so that my WebSocket and API calls keep working without interruption.
40. As a user, I want late or duplicated receipt updates to be harmless, so that message states never regress (e.g. read never flips back to delivered).

## Implementation Decisions

### Architecture shape

- **Monorepo** with two modules: `server` (Python 3.12 + FastAPI) and `mobile` (React Native + Expo + TypeScript). Shared protocol semantics live as Pydantic schemas server-side and a hand-mirrored TypeScript types module client-side (codegen is a P5 optimization, deliberately not now).
- **Single WebSocket connection per device** carries all real-time traffic: messages, receipts, presence, sync. REST carries everything else: auth, discovery, history fetches, conversation management.
- **Server is the source of truth**; the device's local SQLite database is a read-through cache plus an outbox for optimistic sends.
- **At-least-once delivery with client ACKs and idempotency keys** — the WhatsApp model. Every message carries a client-generated `client_msg_id`; the server dedupes on it, so retries are safe.
- **Server-assigned ordering**: each message gets a per-conversation monotonic `seq` number assigned transactionally at insert. Clients render by `seq`, never by client clocks. All timestamps are server UTC.

### Server modules (new)

- **Auth module** — register, login, refresh, logout. Argon2id password hashing. HS256 JWT access tokens (15 min TTL) + rotating refresh tokens (30 day TTL, stored hashed). Rotation detects reuse: presenting a previously-rotated refresh token revokes the entire token family.
- **Users module** — username search (substring, case-insensitive, ranked by prefix match), profile read/update (display name).
- **Conversations module** — create-or-get direct conversation (the pair of members is unique per direct conversation, so reopening a contact reuses history), list endpoint returning each conversation with its last message, unread count, and the other member's profile.
- **Messages module** — send (validates membership, assigns `seq` transactionally, persists, hands to the gateway), cursor-paginated history (`before_seq`, page size 50), batch mark-read.
- **Realtime gateway module** — the WebSocket endpoint and connection registry (in-memory map: user → connections). Handles envelope routing, ACK processing, receipt fan-out, heartbeat (25 s ping, 60 s dead-connection reaper). Single-process in P1; the registry is deliberately isolated behind an interface so Redis pub/sub replaces it in P5 without touching call sites.
- **Presence module** — online set maintained by the gateway; offline applied after a 10 s disconnect grace period (anti-flicker); `last_seen_at` persisted on going offline; presence broadcasts go only to users who share a conversation with the subject.
- **Sync module** — catch-up: given the client's last-known `seq` per conversation, return everything newer, plus any receipt updates newer than a client-supplied timestamp.

### WebSocket authentication

- Expo/native WebSockets can send headers but the web preview cannot, so the handshake uses a **ticket flow**: the client calls a REST endpoint with its access token to mint a single-use, 30-second-TTL ticket, then passes it as a query parameter on the WS connect. The gateway validates and burns the ticket. This also keeps long-lived connections authenticated without re-handshaking.

### Protocol contract (decision-encoding shape, from protocol sketch)

```jsonc
// Every frame, both directions:
{
  "type": "message.send | message.new | message.ack | receipt.delivered
         | receipt.read | presence.update | sync.catchup | error",
  "id": "client-generated uuid, echoed in responses for correlation",
  "ts": "server-assigned, present on all server-originated frames",
  "payload": { }
}

// message.send payload:
{ "conversation_id": "…", "client_msg_id": "uuid", "body": "…" }
// server replies message.ack { client_msg_id, message_id, seq, created_at }
// recipient receives message.new { message with seq }
```

- Client ACKs every `message.new`; the server marks it delivered. Opening a conversation sends one batched read receipt (`up_to_seq`), which the server fans out to the sender.
- Errors are typed frames, never silent disconnects (`error` with a machine-readable `code`).

### Message status state machine (decision-encoding, client-side)

```
pending ──ack──▶ sent ──delivered──▶ delivered ──read──▶ read
   │                                                    ▲
   └──send fails──▶ failed ──retry──▶ pending           │
                 (transitions only move forward; a stale
                  receipt can never regress a state)
```

### Schema changes (all new; Postgres 16)

- **users** — `id` (uuid pk), `username` (citext, unique), `password_hash`, `display_name`, `created_at`, `last_seen_at`
- **refresh_tokens** — `id`, `user_id` (fk), `token_hash` (unique), `family_id`, `expires_at`, `rotated_at`, `revoked_at`
- **conversations** — `id`, `type` (`direct` only in P1), `next_seq` (bigint, the transactional ordering counter), `created_at`
- **conversation_members** — `conversation_id`, `user_id`, `last_read_seq`, `joined_at`; pk `(conversation_id, user_id)`; for `direct`, a canonical ordered member-pair key enforces uniqueness
- **messages** — `id` (uuid pk), `conversation_id` (fk), `sender_id` (fk), `client_msg_id` (unique with `sender_id` — the idempotency key), `seq` (bigint), `body` (text, ≤ 4000 chars), `created_at`, `delivered_at`, `read_at`
- Key indexes: `messages(conversation_id, seq desc)`, `conversation_members(user_id)`, plus the uniques above. Migrations via Alembic.

### Client modules (new)

- **API client** — REST wrapper that attaches the access token and performs a single transparent refresh-and-retry on 401.
- **WS client** — ticket fetch, connect, exponential-backoff reconnect with jitter, heartbeat response, frame dispatch.
- **Local store** — `expo-sqlite` mirror of conversations and messages, plus an `outbox` table for pending sends.
- **Sync engine** — the heart of the client: drains the outbox on (re)connect, requests catch-up per conversation, applies receipts with forward-only state transitions, dedupes by `client_msg_id` and `message_id`.
- **State** — a lightweight store (zustand) over the local DB; UI renders from store, store hydrates from SQLite, SQLite feeds from WS.
- **UI** — auth screens, conversation list, chat screen (bubble, composer, tick indicators, day separators), presence in headers. NativeWind for styling, dark mode from day one, skeleton loading on first open, optimistic send wired through the outbox.

### Deployment / dev environment

- Docker Compose locally: Postgres 16 + backend with hot-reload. Expo dev client on a physical phone against the LAN IP. No cloud deploy in P1.

## Testing Decisions

**What makes a good test here:** tests assert only *externally visible behavior* — "a message sent by A appears at B with a delivered receipt flowing back" — and never mock internal modules of the thing under test. Infrastructure the project owns (Postgres) is real in tests; infrastructure it doesn't (none in P1) doesn't exist yet.

**Seams (two, the minimum possible given two runtimes):**

1. **Server public API seam** — the REST + WebSocket surface, exercised by a real HTTP client and a real WS client against a real ephemeral Postgres. Every server behavior (auth, rotation/reuse-revocation, send → ack → delivered → read flows, offline-then-catch-up, ordering under concurrent sends, idempotent resend, presence grace period) is tested only through this seam.
2. **Client sync-engine seam** — the sync engine's transport boundary. The engine is driven by an in-memory fake transport (scripted frames: acks, duplicates, gaps, out-of-order receipts) and asserts on the resulting local-store state and outbox behavior. This is where offline queue, reconnect catch-up, and forward-only receipts are proven.

UI components get a small number of smoke tests (render + optimistic send interaction); full e2e device testing is manual in P1 and formalized later.

**Prior art:** none — greenfield repo. These two seams and the "public behavior only" rule become the convention P2–P5 must follow. Tooling: pytest + pytest-asyncio (server), jest/vitest + Testing Library (client).

## Out of Scope

- Group chats (P2), media/files of any kind including avatars (P3)
- The P4 differentiators: threaded replies, full-text message search, and advanced username discoverability settings (P1 ships only the basic username identity needed to find people)
- Typing indicators (cheap P2 add, deliberately excluded to keep the P1 protocol minimal)
- Message editing and deletion, multi-device, push notifications (WS dies in background; reconnect-on-foreground covers P1)
- E2EE, email/password recovery, rate limiting beyond basic validation
- Redis, multi-instance fan-out, load testing, k8s (all P5)

## Further Notes

- **Known sharp edge accepted in P1:** backgrounding the phone kills the WebSocket, so "delivered" pauses while backgrounded. This is honest behavior (WhatsApp solves it with push, P3+ territory) and the receipt semantics stay truthful.
- **Forward compatibility:** messages are opaque payloads to the server and the gateway registry sits behind an interface — the two cheap decisions that let E2EE and Redis fan-out land in P5 without rewrites.
- **Milestone breakdown at 6–8 hrs/week (≈4 weeks):**
  1. **Week 1** — Monorepo, Docker Compose, FastAPI skeleton, Alembic, auth module end-to-end (register/login/refresh/logout) with seam-1 tests.
  2. **Week 2** — Gateway + ticket handshake + registry + heartbeat; mobile auth screens, API client with silent refresh, WS client with reconnect; first frame round-trip from phone.
  3. **Week 3** — Messages end-to-end: send → seq → ack → delivered → read, optimistic outbox, chat screen with bubbles and ticks.
  4. **Week 4** — History pagination, conversation list with previews/unread, batched read receipts, presence with grace period, catch-up sync, hardening + test pass.
