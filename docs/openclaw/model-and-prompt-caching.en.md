# OpenClaw Model Configuration & Prompt Caching

> Last updated: 2026-04-08

## Overview

OpenClaw supports multiple model providers via its plugin architecture. This article documents the configuration for running OpenClaw agents through **[Copilot Gateway](../copilot-gateway/introduction.md)** (a Deno-based GitHub Copilot API proxy), including critical settings for prompt caching to work correctly.

## Architecture

```
Discord / WeChat
      |
   OpenClaw (gateway)
      |
   Copilot Gateway (self-hosted)
      |
   GitHub Copilot API → Anthropic API
```

## Provider Configuration

In `~/.openclaw/openclaw.json`, define a custom provider under `models.providers`:

```json
{
  "models": {
    "providers": {
      "claude-proxy": {
        "baseUrl": "https://<your-gateway-domain>",
        "apiKey": "<your-api-key>",
        "api": "anthropic-messages",
        "headers": {
          "User-Agent": "copilot-proxy/1.0"
        },
        "models": [
          { "id": "claude-opus-4.6", "name": "Claude Opus 4.6" },
          { "id": "claude-sonnet-4.6", "name": "Claude Sonnet 4.6" },
          { "id": "claude-haiku-4.5", "name": "Claude Haiku 4.5" }
        ]
      }
    }
  }
}
```

### Critical: Prompt Caching Requirements

Prompt caching requires **all** of the following conditions to be met:

!!! warning "Condition 1: Must use `anthropic-messages` API format"
    Other formats (e.g., `openai-completions`) will not trigger Anthropic's cache control headers. See the [Copilot Gateway introduction](../copilot-gateway/introduction.md) for supported API formats.

!!! warning "Condition 2: Must use Claude models"
    Prompt caching is an Anthropic feature and only works with Claude models (e.g., `claude-opus-4.6`, `claude-sonnet-4.6`, `claude-haiku-4.5`). Other models (e.g., GPT series) will not benefit from caching even with the `anthropic-messages` format.

### Critical: Claude Model Name Format

!!! danger "Must use dot notation"
    When using Claude models through Copilot Gateway, version numbers must use **dots** (`.`), not **dashes** (`-`). This is a known issue discovered during actual Claude model usage.

| Format | Example | Result |
|--------|---------|--------|
| Dot (correct) | `claude-sonnet-4.6` | Works |
| Dash (wrong) | `claude-sonnet-4-6` | `400 model_not_supported` |

[Copilot Gateway](../copilot-gateway/introduction.md) expects dot-format model names. Using dash format (e.g., `claude-sonnet-4-6`) causes the Copilot API to return:

```
HTTP 400: api_error: Copilot API error: 400 {"error":{"message":"The requested model is not supported.","code":"model_not_supported","param":"model","type":"invalid_request_error"}}
```

## Agent Configuration

Each agent is defined in `agents.list` with a primary model and optional fallbacks:

```json
{
  "agents": {
    "list": [
      {
        "id": "main",
        "model": {
          "primary": "claude-proxy/claude-opus-4.6",
          "fallbacks": ["claude-proxy/claude-sonnet-4.6"]
        }
      },
      {
        "id": "biyao",
        "model": {
          "primary": "claude-proxy/claude-sonnet-4.6",
          "fallbacks": ["claude-proxy/claude-haiku-4.5"]
        }
      }
    ]
  }
}
```

### Recommended Model Assignment Strategy

| Role | Primary | Fallback | Rationale |
|------|---------|----------|-----------|
| Main / lead agent | Opus | Sonnet | Best quality for primary interactions |
| Other agents | Sonnet | Haiku | Balance quality and cost |
| Sub-agents | Sonnet | — | Default for spawned sub-agents |

## Prompt Caching

### How It Works

Prompt caching is handled **server-side** by [Copilot Gateway](../copilot-gateway/introduction.md), which transparently passes Anthropic's cache control headers. OpenClaw does not need any client-side configuration for caching.

On each API call, Anthropic caches the system prompt and conversation history. Subsequent calls within the cache TTL (~5 minutes) reuse the cached tokens, dramatically reducing input token costs.

### Verification

Check the session JSONL files at `~/.openclaw/agents/main/sessions/`:

```bash
tail -10 ~/.openclaw/agents/main/sessions/<session-id>.jsonl | grep -o '"usage":{[^}]*}'
```

Look for the `cacheRead` and `cacheWrite` fields:

```json
// First call: writes to cache
{"input": 3, "output": 86, "cacheRead": 0, "cacheWrite": 65353}

// Second call: reads from cache
{"input": 3, "output": 43, "cacheRead": 65353, "cacheWrite": 471}
```

- **`cacheWrite > 0, cacheRead = 0`**: First call, cache is being populated
- **`cacheRead > 0`**: Cache hit, tokens are being reused
- **Both zero**: Caching is not working (check API format)

### Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `cacheRead` always 0 | Wrong API format | Set `"api": "anthropic-messages"` |
| `cacheRead` always 0 | Using non-Claude models | Switch to Claude model family |
| `model_not_supported` error | Wrong Claude model name format | Use dot notation (`4.6` not `4-6`) |
| Cache miss after long pause | Cache TTL expired (~5min) | Normal behavior, next call repopulates |
| All usage fields zero | API call failed | Check error logs in `gateway.err.log` |
