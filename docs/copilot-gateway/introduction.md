# Copilot Gateway 介绍

> 最后更新：2026-04-08

## 概述

[Copilot Gateway](https://github.com/menci/copilot-gateway) 是一个轻量级的 API 代理，部署在 Cloudflare Workers 或 Deno 上，将你的 GitHub Copilot 订阅暴露为标准的 **Anthropic Messages API** 和 **OpenAI Responses API** 端点。这使得 [Claude Code](https://docs.anthropic.com/en/docs/claude-code)、[Codex CLI](https://github.com/openai/codex) 等编程 Agent 可以直接通过 Copilot 来调用模型。

## 工作原理

Copilot Gateway 在客户端和 GitHub Copilot API 之间实时转换 API 格式：

- **Claude Code** 使用 Anthropic Messages API → Gateway 转换为 Copilot 支持的格式
- **Codex CLI** 使用 OpenAI Responses API → Gateway 转换或直接透传
- **任何 OpenAI 兼容客户端** 可以使用 Chat Completions 端点

Gateway 会自动检测每个模型支持的端点（原生 Messages、Responses 或 Chat Completions），选择最佳的转换路径。

## 架构

```
Claude Code / Codex CLI / OpenClaw / 其他客户端
        │
        ▼
  Copilot Gateway (Hono 框架)
  ├── POST /v1/messages          ← Anthropic Messages API
  ├── POST /v1/responses         ← OpenAI Responses API
  ├── POST /v1/chat/completions  ← OpenAI Chat Completions
  ├── POST /v1/embeddings        ← Embeddings 透传
  └── GET  /v1/models            ← 模型列表
        │
        ▼ (按模型自动选择转换路径)
  GitHub Copilot API
```

## 支持的 API 格式

| 端点 | 格式 | 典型客户端 |
|------|------|-----------|
| `/v1/messages` | Anthropic Messages API | Claude Code, [OpenClaw](../openclaw/model-and-prompt-caching.md) |
| `/v1/responses` | OpenAI Responses API | Codex CLI |
| `/v1/chat/completions` | OpenAI Chat Completions | 通用 OpenAI 兼容客户端 |
| `/v1/embeddings` | Embeddings | 向量化工具 |
| `/v1/models` | 模型列表 | 所有客户端 |

!!! tip "OpenClaw 用户请注意"
    如果你通过 Copilot Gateway 接入 OpenClaw，**必须使用 `anthropic-messages` API 格式**才能让 Prompt Caching 生效。详见 [OpenClaw 模型配置与 Prompt Caching](../openclaw/model-and-prompt-caching.md)。

## 部署方式

### 方式一：Deno Deploy

```bash
git clone https://github.com/menci/copilot-gateway.git
cd copilot-gateway

# 设置管理密钥
export ADMIN_KEY=your-secret-admin-key

# 本地开发
deno task dev

# 部署到生产环境（需要 Deno >= 2.4）
deno deploy --prod
```

### 方式二：Cloudflare Workers

```bash
pnpm install

# 创建 D1 数据库
wrangler d1 create copilot-db

# 更新 wrangler.jsonc 中的 account_id 和 database_id
wrangler d1 migrations apply copilot-db

# 设置管理密钥
wrangler secret put ADMIN_KEY

# 部署
wrangler deploy
```

## 初始配置

1. 打开部署后的 URL，使用 `ADMIN_KEY` 登录管理面板
2. 在 **Upstream** 标签页通过 Device OAuth 流程关联你的 GitHub 账号（需要有 Copilot 订阅）
3. 在 **API Keys** 标签页创建 API 密钥
4. 使用生成的配置片段接入你的客户端

## 技术特点

- **TypeScript** 编写，基于 **Hono** 框架
- 95% 代码平台无关（Hono + Web APIs）
- 存储层抽象：Deno Deploy 使用 `DenoKvRepo`，Cloudflare Workers 使用 `D1Repo`
- 支持多 Key 轮转和使用统计
- 自动检测模型能力并缓存探测结果
- MIT 许可证

## 相关文章

- [OpenClaw 模型配置与 Prompt Caching](../openclaw/model-and-prompt-caching.md) — 如何在 OpenClaw 中配置 Copilot Gateway 作为模型提供商
