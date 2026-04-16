# Multi-Agent Discord 通信配置指南

本文记录了五人 Agent 团队在 OpenClaw + Discord 环境下的通信配置方案，包括每项配置的原因和踩过的坑。

## 团队结构

| Agent | 职责 |
|-------|------|
| 小白 | 主调度官，任务拆解与调度 |
| 雪琪 | PM + 知识整理 |
| 碧瑶 | 后端开发 |
| 灵儿 | 前端开发 |
| 瓶儿 | QA 测试 |

---

## 当前配置方案

### 全局配置

```yaml
agentToAgent:
  enabled: false          # 关闭 sessions_send A2A 通道

sessions:
  visibility: "tree"      # 默认值，不开放跨 session 可见性

ackReactionScope: off     # 关闭 ack reaction，减少噪音
```

**原因：**
- `agentToAgent.enabled: false`：sessions_send 会产生严重的 context 污染问题（见踩坑记录），已回退到 Discord @ 模式
- `sessions.visibility: "tree"`：避免 agent 间 session 互相可见导致意外行为

---

### 各 Agent Discord 配置

**小白（主调度官）：**
```yaml
discord:
  allowBots: "mentions"       # 响应 bot 的 @ 消息
  groupPolicy: "allowlist"
  dmPolicy: "pairing"
  mentionPatterns: []         # 禁用名字/emoji 自动 mention 匹配
  guilds:
    <guild_id>:
      requireMention: false   # 小白无需被 @ 也可响应（主调度，需监听全局）
      users:
        - <主人 ID>
        - <全部 5 个 agent ID>
```

**其他四人（雪琪/碧瑶/灵儿/瓶儿）：**
```yaml
discord:
  allowBots: "mentions"       # 只响应 bot 的 @ 消息
  groupPolicy: "allowlist"
  dmPolicy: "pairing"
  mentionPatterns: []         # 禁用名字/emoji 自动 mention 匹配
  guilds:
    <guild_id>:
      requireMention: true    # 必须被 @ 才响应
      users:
        - <主人 ID>
        - <小白 bot ID>       # 只允许主人和小白触发
```

> **安全提示：** 配置中的 `<guild_id>`、`<主人 ID>`、`<agent ID>` 等请替换为实际值，不要将含真实 ID 的配置提交到公开仓库。

---

### `mentionPatterns: []` 的作用

OpenClaw 默认会将 `identity.name` 和 `identity.emoji` 自动生成为 mention 触发正则。这意味着：

- 消息中包含 `🌸碧瑶` → OpenClaw 识别为 mention → 触发碧瑶响应
- 消息中只有 emoji `🌸` → 也会触发碧瑶（emoji 单独匹配）

设置 `mentionPatterns: []` 后：

- 关闭所有自动名字/emoji 匹配
- 只有 Discord 原生 `<@bot_id>` 格式的 @ 才能触发
- emoji 和文字名字单独出现不再触发

---

### 派任务的正确方式

✅ **正确：Discord 原生 @**
```
@碧瑶 请帮我实现用户登录 API
```

❌ **错误：emoji + 名字（会误触发）**
```
🌸碧瑶 请帮我实现用户登录 API
```

❌ **错误：mentionPatterns 未清空时的任何名字提及**

---

## 踩坑记录

### 坑 1：A2A sessions_send 导致 context 污染（Context Rot）

**问题：**
使用 `agentToAgent.enabled: true` + `sessions_send` 进行 agent 间通信时：

1. A 向 B 发送任务 → B 收到并处理
2. B 处理完毕发送 announce → announce 消息路由回 A 的 session
3. A 错误地将 announce 消息当作用户消息处理，产生多余回复
4. 多轮后 context 中充斥大量内部消息，导致 SOUL 规则（称呼、角色）被"稀释"遗忘

**根因：** sessions_send 的 announce 步骤会向发起方 session 注入系统消息，长期运行后造成 context 污染。

**解决：** 关闭 `agentToAgent.enabled`，回退到 Discord @ 模式。

**参考：** OpenClaw GitHub issues #62872, #64917, #53145 均描述了类似问题。

---

### 坑 2：emoji mention 误触发

**问题：**
小白在 Discord 发送包含其他 agent emoji+名字（如 `🌸碧瑶`）的消息时，碧瑶会被误触发响应，即使消息没有 Discord 原生 @。

**根因：** OpenClaw 的 `deriveMentionPatterns` 函数自动将 `identity.emoji` 和 `identity.name` 加入 mention 匹配正则。白名单（users）检查通过后，名字/emoji 匹配即视为 mention。

**排查过程：**
1. 发现纯文字名字（不带 emoji）不触发 → 猜测 emoji 是关键
2. 发现单独 emoji `🌸` 也触发 → 确认 emoji 独立匹配
3. 阅读源码确认 `deriveMentionPatterns` 逻辑
4. 设置 `mentionPatterns: []` → 测试通过

**解决：** 为所有 agent 设置 `mentionPatterns: []`，关闭自动名字/emoji 匹配。

---

## 通信流程图

```
主人
  │
  ├─ @ 小白（或直接说话）
  │
小白（主调度）
  │
  ├─ Discord 原生 @ 雪琪 → 雪琪响应
  ├─ Discord 原生 @ 碧瑶 → 碧瑶响应
  ├─ Discord 原生 @ 灵儿 → 灵儿响应
  └─ Discord 原生 @ 瓶儿 → 瓶儿响应
```

---

## 相关文档

- [Multi-Agent 通信模式选型](./multi-agent-communication.md)
- [OpenClaw A2A sessions_send 踩坑记录](../../knowledge/pitfalls/openclaw-a2a-context-pollution.md)（内部知识库）
