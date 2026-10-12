---
name: hermes-bale-messenger
description: "Use when adding Bale (بله) support to Hermes Gateway. Connect AI agent to Persian messenger via free Bot API."
version: 1.2.0
author: علی محمودی
license: MIT
metadata:
  hermes:
    tags: [bale, persian, messenger, platform, adapter, bot]
    related_skills: [hermes-bale-stt]
---

# Bale Platform Adapter for Hermes

Adds Bale (بله) messenger support to Hermes Gateway as a platform plugin.
Your AI agent can send/receive messages, voice notes, images, and documents
through Bale via the Bot API.

## Quick Install

```bash
cd ~/.hermes/plugins/platforms/
git clone https://github.com/mah92/hermes-bale-messenger-plugin.git bale
hermes plugins enable hermes-bale-messenger
```

Then add your bot token to `~/.hermes/.env`:

```env
BALE_BOT_TOKEN=1234567890:ABCdefGHIjklMNOpqrsTUVwxyz
BALE_ALLOWED_CHATS=123456789,987654321
```

Restart the gateway and you're done.

## Files

| File | Purpose |
|------|---------|
| `plugin.yaml` | Platform manifest — env vars, metadata |
| `__init__.py` | Package entry — re-exports `register()` |
| `adapter.py` | Full `BasePlatformAdapter` implementation |

## Features

- Text messages (send & receive)
- Voice messages — send via TTS, receive voice/audio as downloadable files
- Images — send by URL (with download fallback) or local file upload
- Documents — upload and send
- Typing indicators (`sendChatAction`)
- Group chat support with optional @mention gate
- User/chat allowlisting
- Cron delivery support

## Configuration

All via `~/.hermes/.env`:

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `BALE_BOT_TOKEN` | ✅ | — | Bot token from @BotFather |
| `BALE_ALLOWED_CHATS` | — | all | Comma-separated chat IDs |
| `BALE_ALLOWED_USERS` | — | all | Comma-separated user IDs |
| `BALE_ALLOW_ALL_USERS` | — | false | Set "true" for open access |
| `BALE_HOME_CHANNEL` | — | — | Default chat for cron delivery |
| `BALE_REQUIRE_MENTION` | — | false | Require @mention in groups |
| `BALE_MAX_VOICE_DURATION` | — | 30 | Max voice/audio duration (seconds) for STT. 0 = no limit |

## How It Works

The adapter uses Bale's Bot API (Telegram-compatible) with long-polling.
No gRPC, no user account — just HTTP calls to `tapi.bale.ai`.

```
Bale Server ←→ HTTP Long Poll ←→ Hermes Gateway ←→ AI Agent
```

## Common Pitfalls

1. **Bot blocked by user:** User must `/start` the bot before it can DM them.
2. **Bot-to-bot blocked:** Bale (like Telegram) blocks bots from seeing each other's messages, even in groups. Use a user account bridge for bot-to-bot.
3. **Group privacy:** The bot must be an admin to see all group messages, otherwise it only sees `/command` and replies.
4. **Voice messages arrive empty?** Transcription needs an STT provider. Install `hermes-bale-stt-skill` for Persian STT, or set `stt.provider` in `~/.hermes/config.yaml`.
5. **Cache after edits:** Always `find ~/.hermes/plugins/platforms/bale -name __pycache__ -exec rm -rf {} +` after editing adapter files.
6. **Every incoming file saved as `.ogg`?** `_download_media()` used to take the local suffix from the `file_path` returned by `getFile` with an `or ".ogg"` fallback. Bale returns an **extensionless** `file_path` (a long id, e.g. `123456:-4565197760554590464:1:<hash>`), so PDFs, DOCX and JPEGs all landed on disk as `.ogg` and were then treated as voice notes (transcribed, and rendered twice — once as document, once as audio). The suffix must come from the MESSAGE metadata: `file_name` → `mime_type` → remote path → an explicit `default_ext` (`.ogg` only for real voice). Note Bale reports `mime_type` as the literal junk string `?/?` for documents, so `file_name` must be checked FIRST. A new media branch that forgets to pass `file_name`/`mime_type` silently regresses. The INFO log `[bale] Downloaded media name=... mime=... remote=... -> path` makes receipts self-diagnosing.
7. **Wrong bot token / user number pegs a CPU core (fixed 2026-10-12):** Bale answers a bad token, bad offset or server fault with a *fast* JSON error — HTTP 401/404/500 (or HTTP 200 with `ok: false`) in ~0.15s — **not** a network failure. The old `_api_get`/`_api_post` returned that body as a normal result and `_get_updates()` read it as "no updates", so `_poll_loop` re-polled with **no delay**: measured 5,441 requests/s and 100% CPU (16,324 hits in 3s). Two things fix it: (a) `_api_get`/`_api_post`/`_api_post_multipart` now raise `BaleApiError` on `status >= 400` or `ok: false`, so the poll loop's `except` branch sleeps `POLL_INTERVAL`; (b) `connect()` refuses to start when `getMe` yields no bot id, so a wrong token never even begins polling; plus a floor guard in `_poll_loop` (a sub-second round trip with no updates yields before re-polling). Never let a helper hand an error body back as a successful empty result on a long-poll path.
