# Copilot Gateway Introduction

> Last updated: 2026-04-08

## Overview

[Copilot Gateway](https://github.com/menci/copilot-gateway) is a lightweight API proxy deployed on Cloudflare Workers or Deno that exposes your GitHub Copilot subscription as standard **Anthropic Messages API** and **OpenAI Responses API** endpoints — letting you use [Claude Code](https://docs.anthropic.com/en/docs/claude-code), [Codex CLI](https://github.com/openai/codex), and other coding agents through Copilot.

## How It Works

Copilot Gateway translates between API formats on the fly:

- **Claude Code** talks Anthropic Messages API → Gateway translates to whatever Copilot supports
- **Codex CLI** talks OpenAI Responses API → Gateway translates or passes through accordingly
- **Any OpenAI-compatible client** can use the Chat Completions endpoint

The gateway auto-detects each model's supported endpoints (native Messages, Responses, or Chat Completions) and picks the best translation path.

## Architecture

```
Claude Code / Codex CLI / OpenClaw / any client
        │
        ▼
  Copilot Gateway (Hono)
  ├── POST /v1/messages          ← Anthropic Messages API
  ├── POST /v1/responses         ← OpenAI Responses API
  ├── POST /v1/chat/completions  ← OpenAI Chat Completions
  ├── POST /v1/embeddings        ← Embeddings passthrough
  └── GET  /v1/models            ← Model listing
        │
        ▼ (auto-selects translation path per model)
  GitHub Copilot API
```

## Supported API Formats

| Endpoint | Format | Typical Clients |
|----------|--------|-----------------|
| `/v1/messages` | Anthropic Messages API | Claude Code, [OpenClaw](../openclaw/model-and-prompt-caching.md) |
| `/v1/responses` | OpenAI Responses API | Codex CLI |
| `/v1/chat/completions` | OpenAI Chat Completions | Generic OpenAI-compatible clients |
| `/v1/embeddings` | Embeddings | Vectorization tools |
| `/v1/models` | Model listing | All clients |

!!! tip "For OpenClaw Users"
    If you're connecting OpenClaw through Copilot Gateway, you **must use the `anthropic-messages` API format** for prompt caching to work. See [OpenClaw Model Configuration & Prompt Caching](../openclaw/model-and-prompt-caching.md) for details.

## Deployment

### Option 1: Deno Deploy

```bash
git clone https://github.com/menci/copilot-gateway.git
cd copilot-gateway

# Set admin key
export ADMIN_KEY=your-secret-admin-key

# Local development
deno task dev

# Deploy to production (requires Deno >= 2.4)
deno deploy --prod
```

### Option 2: Cloudflare Workers

```bash
pnpm install

# Create D1 database
wrangler d1 create copilot-db

# Update wrangler.jsonc with your account_id and database_id
wrangler d1 migrations apply copilot-db

# Set admin key
wrangler secret put ADMIN_KEY

# Deploy
wrangler deploy
```

## Initial Setup

1. Open the deployed URL in a browser, log in with your `ADMIN_KEY`
2. Go to the **Upstream** tab and connect your GitHub account (needs active Copilot subscription) via Device OAuth
3. Go to the **API Keys** tab and create an API key
4. Use the generated configuration snippets for your client

## Technical Highlights

- Written in **TypeScript**, built on the **Hono** framework
- 95% platform-agnostic code (Hono + Web APIs)
- Storage abstraction: `DenoKvRepo` for Deno Deploy, `D1Repo` for Cloudflare Workers
- Multi-key rotation with usage statistics
- Automatic model capability detection with cached probes
- MIT License

## Related Articles

- [OpenClaw Model Configuration & Prompt Caching](../openclaw/model-and-prompt-caching.md) — How to configure Copilot Gateway as a model provider in OpenClaw
