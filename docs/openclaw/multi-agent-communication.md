# Multi-Agent 通信模式选型：Discord @ vs sessions_send

> **结论先行：** 在 OpenClaw 中运行多 agent 团队时，推荐使用 Discord @ 模式（`allowBots: "mentions"`）进行 bot 间通信，而非 `sessions_send` 内部通道。后者会导致 context 污染，使 agent 逐渐忘记角色设定。

## 背景

OpenClaw 支持两种 bot 间通信方式：

1. **Discord @ 模式**：bot 在 Discord 中 @ 另一个 bot，触发对方 LLM
2. **sessions_send 模式**：通过内部 A2A 通道发送消息，不经过 Discord

我们在 5 agent 团队（1 调度 + 4 执行者）中实测了两种方案。

## sessions_send 模式

### 配置

```json
{
  "channels.discord.accounts.*.allowBots": false,
  "tools.agentToAgent.enabled": true,
  "tools.agentToAgent.allow": ["main", "biyao", "xueqi", "jinpinger", "tianlinger"],
  "tools.sessions.visibility": "all"
}
```

### 工作流

1. 调度 agent 用 `sessions_send` 发内部消息给目标 agent
2. 目标 agent 处理后用 `message` tool 发结果到 Discord thread
3. Discord 中可见各自头像/身份

### 优点

- 速度更快（不经 Discord→LLM 链路）
- 不会触发 bot 间 Discord 消息循环
- Discord thread 中保持各自身份

### 致命问题：Context 污染 → SOUL 遗忘

`sessions_send` 创建的 session 是 webchat 通道，context 会不断累积：

| 消息类型 | 来源 | 有用？ |
|---------|------|-------|
| 调度者的任务指令 | sessions_send | ✅ |
| agent 的回复 | LLM | ✅ |
| `Agent-to-agent announce step` | 系统自动 | ❌ |
| `ANNOUNCE_SKIP` | agent 响应 | ❌ |
| `delivery-mirror` 回显 | 系统自动 | ❌ |

**每次 sessions_send 至少产生 2-3 条无用消息**。实测中，5 次任务后 session 累积 55+ 条消息，其中大量是系统噪音。

随着 context 增长，模型对 system prompt（含 SOUL.md / AGENTS.md）的注意力被稀释，agent 开始忘记：
- 角色称呼（「傻瓜」变成「你」）
- 自称（「瑶儿」消失）
- 性格特征

## Discord @ 模式

### 配置

```json
{
  "channels.discord.accounts.*.allowBots": "mentions",
  "tools.agentToAgent.enabled": false
}
```

### 为什么不会有 Context 污染

- Session 绑定 discord 通道，每次新消息都带完整 system prompt
- Context pruning（`keepLastAssistants: 15`）裁剪后剩余的都是正常对话
- 没有 announce step、delivery-mirror 等系统噪音
- SOUL.md / AGENTS.md 影响力稳定

### 注意事项

- 需要 `maxPingPongTurns`（默认 5）防止 bot 间无限循环
- bot 需要互相在 `guilds.*.users` 白名单中
- 串行派任务（OpenClaw 多 bot 启动 bug，一次只能保证一个 bot 在线）

## 总结

| 维度 | Discord @ | sessions_send |
|------|-----------|---------------|
| 速度 | 较慢 | ⚡ 快 |
| SOUL 稳定性 | ✅ 稳定 | ❌ 逐渐遗忘 |
| Context 干净度 | ✅ 干净 | ❌ 系统噪音多 |
| Discord 可见性 | ✅ 各自身份 | ✅ 各自身份 |
| 循环风险 | ⚠️ 需 maxPingPongTurns | ✅ 无 |

**推荐：使用 Discord @ 模式**，直到 OpenClaw 解决以下问题：
- sessions_send 的 announce step 可关闭
- 支持一次性 session（不复用 context）
- context 自动过滤系统噪音

## 参考

- [Anthropic: Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — Context Rot 概念，context 越长注意力越稀释
- [Intrinsic Memory Agents (arxiv 2508.08997)](https://arxiv.org/html/2508.08997v1) — 结构化 agent 记忆模板，角色对齐，内生更新
- [Google A2A Protocol](https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/) — 任务级 session 隔离
- 相关 pitfall：[A2A sessions_send 通信导致 SOUL 遗忘](/knowledge/memory/pitfalls/openclaw-a2a-context-pollution.md)
