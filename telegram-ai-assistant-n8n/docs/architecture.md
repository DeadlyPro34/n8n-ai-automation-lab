# Architecture

This document describes how the Telegram AI Voice & Text Assistant workflow is structured in n8n.

## Overview

The workflow is triggered by incoming Telegram messages and branches into two paths depending on whether the message is text or voice. Both paths converge on a single Groq-backed LLM chain, and the result is sent back to the user as a Telegram reply.

## Flow diagram

```
                    Telegram message
                          |
                    Telegram trigger
                          |
                    If (has voice?)
                    /              \
                 yes                no
                  |                  |
             Get voice file     Edit fields
                  |             (extract text)
          Set audio metadata         |
             (Code node)             |
                  |                  |
            Groq Whisper             |
          (speech to text)           |
                  \                 /
                   \               /
                    Groq LLM chain
                   (openai/gpt-oss-20b)
                          |
                    Telegram reply
```

## Nodes

| Node | Type | Role |
|---|---|---|
| Telegram Trigger | `telegramTrigger` | Entry point; fires on incoming Telegram messages |
| If | `if` | Branches on whether `message.voice` is present |
| Get a file | `telegram` (file resource) | Downloads the voice note from Telegram by `file_id` |
| Code in JavaScript | `code` | Tags the downloaded binary with filename/extension/MIME type so Groq accepts it |
| HTTP Request | `httpRequest` | Calls Groq's Whisper transcription endpoint directly (`whisper-large-v3-turbo`) |
| Edit Fields | `set` | Extracts `message.text` for the text-only path |
| Groq Chat Model | `lmChatGroq` | LLM backend (`openai/gpt-oss-20b`) feeding the LLM chain |
| Basic LLM Chain | `chainLlm` | Generates the reply from either transcribed or typed text |
| Send a text message | `telegram` | Sends the final AI response back to the user |

## Design notes

- **Single convergence point** — regardless of input type (text or voice), both branches normalize down to plain text before reaching the LLM chain. This keeps the prompt logic in one place instead of duplicating it per path.
- **Transcription is a raw HTTP call, not a native node** — Groq's Whisper endpoint is called via a generic HTTP Request node with header auth, rather than a dedicated n8n integration node, since none exists for this specific endpoint.
- **Stateless** — the workflow does not persist conversation history between messages; each message is answered independently.
- **Credentials are never embedded** — Telegram, Groq, and Header Auth credentials are referenced by n8n's credential store, not hardcoded in the workflow JSON. See the main [README](README.md#setup) for setup steps.

## Possible extensions

- Add a memory/state node (e.g. keyed by Telegram chat ID) to support multi-turn conversations.
- Add error-handling branches around the Whisper HTTP call and LLM chain for graceful failure messages.
- Add tool-calling nodes (web search, calculator) ahead of the LLM chain.
