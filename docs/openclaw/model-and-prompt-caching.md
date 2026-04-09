# OpenClaw Model Configuration & Prompt Caching

> Last updated: 2026-04-08

## Overview

OpenClaw supports multiple model providers via its plugin architecture. This article documents the configuration for running OpenClaw agents through **copilot-gateway** (a Deno-based Anthropic-to-Copilot API proxy), including critical settings for prompt caching to work correctly.

## Architecture

```
Discord / WeChat
      |
   OpenClaw (gateway)
      |
   copilot-gateway (copilot.xiaoyinggee.com)
      |
   Anthropic API
```

## Provider Configuration

In `~/.openclaw/openclaw.json`, define a custom provider under `models.providers`:

```json
{
  "models": {
    "providers": {
      "claude-proxy": {
        "baseUrl": "https://copilot.xiaoyinggee.com",
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

### Critical: API Format

**Must use `anthropic-messages`**. This is required for prompt caching to work. Other formats (e.g., `openai-completions`) will not trigger Anthropic's cache control headers.

### Critical: Model Name Format

**Must use dot notation** for model version numbers:

| Format | Example | Result |
|--------|---------|--------|
| Dot (correct) | `claude-sonnet-4.6` | Works |
| Dash (wrong) | `claude-sonnet-4-6` | `400 model_not_supported` |

The copilot-gateway proxy expects dot-format model names. Using dash format (e.g., `claude-sonnet-4-6`) causes the Copilot API to return:

```json
{
  "error": {
    "message": "The requested model is not supported.",
    "code": "model_not_supported",
    "type": "invalid_request_error"
  }
}
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
| Main / lead agent | opus | sonnet | Best quality for primary interactions |
| Other agents | sonnet | haiku | Balance quality and cost |
| Sub-agents | sonnet | - | Default for spawned sub-agents |

## Prompt Caching

### How It Works

Prompt caching is handled **server-side** by copilot-gateway, which transparently passes Anthropic's cache control headers. OpenClaw does not need any client-side configuration for caching.

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
| `model_not_supported` error | Wrong model name format | Use dot notation (`4.6` not `4-6`) |
| Cache miss after long pause | Cache TTL expired (~5min) | Normal behavior, next call repopulates |
| All usage fields zero | API call failed | Check error logs in `gateway.err.log` |
