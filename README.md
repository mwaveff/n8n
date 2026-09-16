# Telegram AI Content Bot (n8n)

A self-hosted Telegram bot that turns a topic into a ready-to-publish channel
post — powered by [n8n](https://n8n.io), an LLM, and live web search. Built to
run **for free** on an always-on Linux VM (e.g. Oracle Cloud Free Tier).

You send the bot a topic, it finds a fresh news article about it, writes a clean
post in your language, attaches the article's image (or generates one), and sends
it to you for approval with inline buttons. One tap publishes it to your channel.

---

## Features

- **On-demand posts** — message the bot a topic; it searches recent news
  (Tavily), writes a post from the real article, and pulls the article's image.
- **Human moderation** — every draft arrives with `Publish / Edit / Reject`
  inline buttons; nothing goes live without a tap.
- **Voice & media input** — send a voice note (auto-transcribed), a captioned
  photo, or a video link, and the bot builds a post around it.
- **Scheduled auto-posting** — pulls from RSS sources + web search every few
  hours and queues drafts for review.
- **Weekly digest** — a roundup of the week's top items every Friday.
- **Error alerts** — failures are reported straight to Telegram.
- **Access control** — only whitelisted Telegram user IDs can drive the bot.

## Architecture

```
Telegram ── webhook ──▶ n8n ──▶ Tavily (news search)
                          │      LLM (post generation)
                          │      article image / image generation
                          ▼
                    You (moderation) ──tap──▶ Channel
```

Everything runs in Docker behind Caddy (automatic HTTPS via Let's Encrypt).

## Workflows

| File | What it does |
|------|--------------|
| `bot-hub.json` | Main bot: input, classification, news search, generation, moderation, publishing |
| `autoposting.json` | Scheduled drafts from RSS + web search |
| `weekly-digest.json` | Friday weekly roundup |
| `error-notifications.json` | Sends workflow errors to Telegram |

## Tech stack

n8n · Docker · Caddy · Telegram Bot API · Tavily · an OpenAI-compatible LLM
(e.g. Groq `openai/gpt-oss-120b`) · optional OpenAI (voice/covers).

## Quick start (free hosting)

Full step-by-step guide: **`server/oracle-deploy.md`** (Oracle Cloud Free Tier
+ DuckDNS, 100% free). In short:

1. Create an always-free Linux VM and point a domain (or DuckDNS subdomain) at it.
2. Deploy with the provided `server/docker-compose.yml` + `server/Caddyfile`
   (or paste `server/oracle-cloud-init.yaml` at VM creation to auto-install).
3. Open your domain, create the n8n owner account.
4. Import the four workflow files.
5. Add credentials (below) and hit **Publish** on each workflow.

## Configuration

Create these credentials in n8n and set the placeholders in the workflows:

- **Telegram** — bot token from [@BotFather](https://t.me/BotFather).
- **LLM** — an OpenAI-type credential (e.g. Groq: base URL
  `https://api.groq.com/openai/v1`, model `openai/gpt-oss-120b`).
- **Tavily** — API key from [tavily.com](https://tavily.com) for news search.
- *(optional)* **OpenAI** — for voice transcription and cover generation.

Then replace the placeholders left in the workflow files:

- `YOUR_TELEGRAM_ID` — your numeric Telegram user ID (whitelist / where drafts go).
- `@your_channel` — the channel the bot publishes to (bot must be an admin).
- `tvly-YOUR_TAVILY_KEY` — only if a key is kept inline instead of a credential.

## Security

This repository is a sanitized template — it contains **no** real tokens, keys,
personal IDs, or channel names. Never commit `.env`, SSH keys, n8n's encryption
key, or database backups (all covered by `.gitignore`). Rotate any key that was
ever exposed.

## License

MIT — free to use, modify and distribute. **Attribution required:** you must keep the copyright notice and the `LICENSE` file (crediting the author) in all copies or substantial portions. No warranty.
