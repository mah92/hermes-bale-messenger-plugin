---
name: hermes-bale-bot-skill
description: "Use when adding Bale (بله) support to Hermes Gateway. Connect AI agent to Persian messenger via free Bot API."
version: 1.1.0
author: Ali Sani
license: MIT
metadata:
  hermes:
    tags: [bale, persian, messenger, platform, adapter, bot]
    related_skills: [hermes-platform-adapter, bale-hermes-connection]
---

# Bale Platform Adapter for Hermes

Adds Bale (بله) messenger support to Hermes Gateway as a platform plugin.
Your AI agent can send/receive messages, voice notes, images, and documents
through Bale via the Bot API.

## Quick Install

```bash
cd ~/.hermes/plugins/platforms/
git clone git@github.com:mah92/hermes-bale-messenger-plugin.git bale
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
4. **Voice messages arrive empty?** Transcription needs an STT provider. Install `bale-messenger-bot-stt-skill` for Persian STT, or set `stt.provider` in `~/.hermes/config.yaml`.
5. **Cache after edits:** Always `find ~/.hermes/plugins/platforms/bale -name __pycache__ -exec rm -rf {} +` after editing adapter files.
