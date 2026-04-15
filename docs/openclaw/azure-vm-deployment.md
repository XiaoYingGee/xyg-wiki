# Azure VM 部署 OpenClaw 独立实例

> 本文档记录了在 Azure VM 上部署独立 OpenClaw 实例（接入 Discord Bot）的完整过程，包含常见坑点和排查步骤，供后续参考。

---

## 环境信息

| 项目 | 值 |
|------|-----|
| 主机 | Azure VM（Linux/Ubuntu） |
| SSH | `ssh <YOUR_VM_ALIAS>`（via Cloudflare Access） |
| Node.js | v24.x（nvm 管理） |
| npm | 11.x |
| OpenClaw | 2026.4.x |
| 服务管理 | systemd (user session) |
| 监听端口 | 18789（默认） |

---

## 前置条件

- VM 已通过 SSH 可访问
- Node.js 已通过 nvm 安装（`source ~/.nvm/nvm.sh` 可用）
- Discord Bot 已在 [Developer Portal](https://discord.com/developers/applications) 创建，并已邀请加入目标服务器
- Discord Bot 已开启以下 Intents（Developer Portal → Bot → Privileged Gateway Intents）：
  - ✅ **Message Content Intent**
  - ✅ **Server Members Intent**（可选，用于用户列表）

---

## 安装步骤

### 1. 检查 Node.js 环境

```bash
source ~/.nvm/nvm.sh
node --version   # 需要 v20+
npm --version
```

> ⚠️ nvm 管理的 Node 不在默认 PATH 里，每次 SSH 都需要 `source ~/.nvm/nvm.sh`，或者加到 `~/.bashrc` 里。

### 2. 安装 OpenClaw

```bash
source ~/.nvm/nvm.sh
npm install -g openclaw
openclaw --version  # 验证安装
```

### 3. 初始化配置文件

创建 `~/.openclaw/openclaw.json`：

```json
{
  "meta": {
    "lastTouchedVersion": "2026.4.x"
  },
  "models": {
    "providers": {
      "copilot-gateway": {
        "baseUrl": "https://<YOUR_COPILOT_GATEWAY_URL>",
        "apiKey": "<YOUR_COPILOT_GATEWAY_API_KEY>",
        "api": "anthropic-messages",
        "models": [
          { "id": "claude-sonnet-4.6", "name": "Claude Sonnet 4.6" },
          { "id": "claude-haiku-4.5", "name": "Claude Haiku 4.5" }
        ]
      }
    }
  },
  "gateway": {
    "mode": "local",
    "bind": "lan"
  },
  "channels": {
    "discord": {
      "enabled": true,
      "token": "<YOUR_DISCORD_BOT_TOKEN>",
      "dmPolicy": "pairing",
      "guilds": {
        "<YOUR_GUILD_ID>": {
          "requireMention": false,
          "users": ["<AUTHORIZED_USER_DISCORD_ID>"]
        }
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "copilot-gateway/claude-sonnet-4.6"
      },
      "timeoutSeconds": 600
    },
    "list": [
      {
        "id": "main",
        "workspace": "/home/<YOUR_VM_USER>/.openclaw/workspace"
      }
    ]
  },
  "bindings": [
    {
      "agentId": "main",
      "match": {
        "channel": "discord",
        "accountId": "default"
      }
    }
  ]
}
```

**关键字段说明：**

| 字段 | 说明 |
|------|------|
| `models.providers.copilot-gateway.apiKey` | LLM API Key，由管理员分配 |
| `channels.discord.token` | Discord Bot Token，在 Developer Portal 生成 |
| `channels.discord.guilds.<GUILD_ID>` | Bot 所在服务器的 Guild ID（右键服务器 → 复制ID） |
| `channels.discord.guilds.<GUILD_ID>.users` | 允许与 Bot 交互的 Discord 用户 ID 列表 |
| `agents.list[0].workspace` | Agent 工作目录路径，需要提前创建 |

```bash
mkdir -p ~/.openclaw/workspace
```

### 4. 安装并启动 systemd 服务

```bash
source ~/.nvm/nvm.sh
openclaw gateway install   # 注册 systemd 服务，自动生成 gateway token
systemctl --user start openclaw-gateway.service
systemctl --user enable openclaw-gateway.service  # 开机自启
```

### 5. 验证运行状态

```bash
source ~/.nvm/nvm.sh
openclaw gateway status
# 期望输出包含：
# Runtime: running ...
# RPC probe: ok
```

---

## 验证 Discord 连接

查看日志确认 Bot 登录和 Guild 解析状态：

```bash
tail -50 /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log | grep -i discord
```

**正常日志应包含：**
```
[INFO] [default] starting provider (@<BotName>)
[INFO] discord channels resolved: <GUILD_ID> (guild:<ServerName>)
[INFO] discord channel users resolved: <USER_ID>
[INFO] discord client initialized as <BOT_ID> (<BotName>); awaiting gateway readiness
```

**如果看到 `unresolved`：**
```
[INFO] discord channels unresolved: guild:<GUILD_ID>
```
→ Bot 还未加入该服务器，需要用邀请链接将 Bot 添加进去。

---

## 常见坑点与排查

### ❌ 坑1：Guild ID 配错了

**症状**：日志显示 `discord channels unresolved`，Bot 在线但不响应

**原因**：配置的 Guild ID 和 Bot 实际加入的服务器不匹配

**排查**：
1. 在 Discord 里右键目标服务器图标 → 复制服务器 ID
2. 对比配置文件里的 `channels.discord.guilds` 的 key
3. 注意：**Bot 需要单独被邀请加入每个服务器**，不能跨服务器使用

**修复**：
```bash
# 编辑配置
vim ~/.openclaw/openclaw.json
# 修改 guilds 下的 key 为正确的 Guild ID

# 重启服务（支持热重载，也可以直接等）
systemctl --user restart openclaw-gateway.service
```

---

### ❌ 坑2：authorizedSenders 未配置

**症状**：Bot 在线，Guild 正确，但不响应特定用户的消息

**原因**：`users` 列表为空或未包含该用户的 Discord ID

**排查**：
1. 在 Discord 开启开发者模式（设置 → 高级 → 开发者模式）
2. 右键用户头像 → 复制用户 ID
3. 检查配置文件里 `channels.discord.guilds.<GUILD_ID>.users` 是否包含该 ID

**修复**：在 `users` 数组里添加用户 ID，重启服务。

---

### ❌ 坑3：Discord Bot Intents 未开启

**症状**：Bot 能登录但收不到消息内容（消息为空）

**原因**：Discord Developer Portal 里 Privileged Gateway Intents 未开启

**修复**：
1. 前往 [Discord Developer Portal](https://discord.com/developers/applications)
2. 选择 Bot → Bot 设置页
3. 开启 **Message Content Intent**（必须）
4. 保存后重启 OpenClaw 服务

---

### ❌ 坑4：nvm Node 路径问题

**症状**：systemd 服务启动报错找不到 node

**原因**：systemd 服务环境变量里没有 nvm 的 PATH

**修复**：
```bash
source ~/.nvm/nvm.sh
openclaw gateway install   # 重新安装服务，会自动写入当前 node 路径
systemctl --user daemon-reload
systemctl --user restart openclaw-gateway.service
```

> 长期建议：安装系统级 Node（`sudo apt install nodejs`），避免依赖 nvm 路径。

---

### ❌ 坑5：`plugins.entries.discord.token` 报错

**症状**：`openclaw gateway status` 报 `Invalid config: Unrecognized key "token"`

**原因**：Discord Token **不是**放在 `plugins.entries.discord` 里，正确位置是 `channels.discord.token`

**正确配置路径**：
```json
{
  "channels": {
    "discord": {
      "token": "<YOUR_DISCORD_BOT_TOKEN>"   ✅
    }
  }
}
```
❌ 错误（会报 config invalid）：
```json
{
  "plugins": {
    "entries": {
      "discord": {
        "token": "<...>"   ❌ 不是这里！
      }
    }
  }
}
```

---

## systemd 服务管理命令速查

```bash
# 查看状态
source ~/.nvm/nvm.sh && openclaw gateway status

# 启动
systemctl --user start openclaw-gateway.service

# 停止
systemctl --user stop openclaw-gateway.service

# 重启
systemctl --user restart openclaw-gateway.service

# 开机自启
systemctl --user enable openclaw-gateway.service

# 查看 systemd 日志
journalctl --user -u openclaw-gateway.service -f

# 查看 OpenClaw 日志
tail -f /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log
```

---

## 生成 Discord Bot 邀请链接

将 `<BOT_APP_ID>` 替换为 Bot 的 Application ID：

```
https://discord.com/api/oauth2/authorize?client_id=<BOT_APP_ID>&permissions=412317273088&scope=bot%20applications.commands
```

所需权限（`412317273088`）包含：
- 读取消息
- 发送消息
- 发送消息到 Thread
- 读取消息历史
- 使用斜杠命令

---

## 后续配置（待完成）

| 项目 | 说明 | 状态 |
|------|------|------|
| M2 模式配置 | 由管理员指定后添加 | ⏳ 待配置 |
| Agent 性格/SOUL.md | 配置 Agent 工作空间内容 | ⏳ 待配置 |
| 多 Agent 支持 | 根据需求添加更多 agents.list 条目 | ⏳ 可选 |

---

*部署时间：2026-04-15*  
*操作人：碧瑶*  
*任务来源：小白调度*
