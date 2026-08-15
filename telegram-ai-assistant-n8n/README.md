# Telegram AI Voice & Text Assistant

An AI-powered Telegram bot built with n8n that handles both text and voice messages, using Groq for fast LLM inference and Whisper for speech-to-text.

## Features

- Text-based AI conversations
- Voice message support (auto-transcribed before being answered)
- Automatic routing between text and voice paths
- Groq-hosted LLM (`openai/gpt-oss-20b`) for responses
- Groq-hosted Whisper (`whisper-large-v3-turbo`) for transcription
- Fully automated, event-driven workflow (Telegram webhook trigger)

## Architecture

A Telegram message triggers the workflow. An `If` node checks whether the incoming message contains a voice note:

- **Voice path** — the file is downloaded from Telegram, tagged with the correct filename/MIME type via a small Code node, then sent to Groq's Whisper endpoint for transcription.
- **Text path** — the message text is extracted directly via a Set node.

Both paths converge into a single LLM Chain node backed by Groq, and the response is sent back to the user via the Telegram API.

## Workflow

```
Telegram message
      |
Telegram trigger
      |
   If (voice?)
   /        \
 yes          no
  |            |
Get file    Edit fields
  |            |
Set audio       |
metadata        |
  |             |
Groq Whisper     |
(transcribe)     |
   \            /
    \          /
     Groq LLM chain
          |
    Telegram reply
```

## Tech stack

| Technology | Purpose |
|---|---|
| n8n | Workflow automation / orchestration |
| Telegram Bot API | Chat interface |
| Groq (`openai/gpt-oss-20b`) | LLM inference |
| Groq Whisper (`whisper-large-v3-turbo`) | Speech-to-text |
| n8n Code node (JavaScript) | Audio metadata tagging |

## Setup

1. Import `telegram-ai-assistant.json` into your n8n instance.
2. Create/select a **Telegram API** credential (Telegram Trigger, Get a file, Send a text message nodes) using your bot token from [@BotFather](https://t.me/BotFather).
3. Create a **Groq API** credential (Groq Chat Model node) with your Groq API key.
4. Create a **Header Auth** credential (HTTP Request node) that sends `Authorization: Bearer <your-groq-api-key>` — this is used for the raw Whisper transcription call.
5. Activate the workflow.
6. Message your bot on Telegram to test both text and voice paths.

## Security

No credentials, tokens, chat IDs, or webhook secrets are included in this repository. All sensitive values are referenced through n8n's credential system and must be configured locally after import — see [Setup](#setup).

## Future improvements

- Conversation memory across turns
- Tool calling (web search, calculators, etc.)
- Database-backed logging/analytics
- Error handling and failure notifications
- Response quality evaluation
