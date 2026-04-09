# OpenClaw 模型配置与 Prompt Caching

> 最后更新：2026-04-08

## 概述

OpenClaw 通过插件架构支持多种模型提供商。本文记录了通过 **[Copilot Gateway](../copilot-gateway/introduction.md)**（基于 Deno 的 GitHub Copilot API 代理）运行 OpenClaw Agent 的配置方法，包括让 Prompt Caching 正确生效的关键设置。

## 架构

```
Discord / 微信
      |
   OpenClaw（网关）
      |
   Copilot Gateway (自部署)
      |
   GitHub Copilot API → Anthropic API
```

## Provider 配置

在 `~/.openclaw/openclaw.json` 中，于 `models.providers` 下定义自定义 provider：

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

### 关键点：API 格式

!!! warning "必须使用 `anthropic-messages`"
    这是 Prompt Caching 生效的前提条件。使用其他格式（如 `openai-completions`）不会触发 Anthropic 的缓存控制头。关于 [Copilot Gateway](../copilot-gateway/introduction.md) 支持的 API 格式，请参阅其介绍文档。

### 关键点：模型名称格式

!!! danger "必须使用点号格式"
    模型版本号必须使用**点号**（`.`），不能使用**短横线**（`-`）。

| 格式 | 示例 | 结果 |
|------|------|------|
| 点号（正确） | `claude-sonnet-4.6` | 正常工作 |
| 短横线（错误） | `claude-sonnet-4-6` | `400 model_not_supported` |

[Copilot Gateway](../copilot-gateway/introduction.md) 要求点号格式的模型名。使用短横线格式（如 `claude-sonnet-4-6`）会导致 Copilot API 返回：

```json
{
  "error": {
    "message": "The requested model is not supported.",
    "code": "model_not_supported",
    "type": "invalid_request_error"
  }
}
```

## Agent 配置

每个 Agent 在 `agents.list` 中定义，包含主模型和可选的备用模型：

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

### 推荐的模型分配策略

| 角色 | 主模型 | 备用模型 | 理由 |
|------|--------|----------|------|
| 主 Agent / 领队 | Opus | Sonnet | 主要交互需要最佳质量 |
| 其他 Agent | Sonnet | Haiku | 平衡质量与成本 |
| 子 Agent | Sonnet | — | 生成的子 Agent 使用默认模型 |

## Prompt Caching

### 工作原理

Prompt Caching 由 [Copilot Gateway](../copilot-gateway/introduction.md) **服务端**处理，它会透明地传递 Anthropic 的缓存控制头。OpenClaw 客户端**无需额外配置**。

每次 API 调用时，Anthropic 会缓存系统提示词和对话历史。在缓存 TTL（约 5 分钟）内的后续调用会复用缓存的 token，大幅降低输入 token 成本。

### 验证方法

检查 `~/.openclaw/agents/main/sessions/` 下的会话 JSONL 文件：

```bash
tail -10 ~/.openclaw/agents/main/sessions/<session-id>.jsonl | grep -o '"usage":{[^}]*}'
```

查看 `cacheRead` 和 `cacheWrite` 字段：

```json
// 第一次调用：写入缓存
{"input": 3, "output": 86, "cacheRead": 0, "cacheWrite": 65353}

// 第二次调用：从缓存读取
{"input": 3, "output": 43, "cacheRead": 65353, "cacheWrite": 471}
```

- **`cacheWrite > 0, cacheRead = 0`**：首次调用，正在填充缓存
- **`cacheRead > 0`**：缓存命中，token 被复用
- **两者都为零**：缓存未生效（检查 API 格式）

### 故障排查

| 现象 | 原因 | 解决方法 |
|------|------|----------|
| `cacheRead` 始终为 0 | API 格式错误 | 设置 `"api": "anthropic-messages"` |
| `model_not_supported` 错误 | 模型名称格式错误 | 使用点号格式（`4.6` 而非 `4-6`） |
| 长时间暂停后缓存未命中 | 缓存 TTL 过期（约 5 分钟） | 正常现象，下次调用会重新填充 |
| 所有 usage 字段为零 | API 调用失败 | 检查 `gateway.err.log` 错误日志 |
