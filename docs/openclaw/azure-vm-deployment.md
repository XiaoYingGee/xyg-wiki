# Azure VM 部署 OpenClaw 独立实例

> 状态：基础安装完成，Discord Bot Token 和 LLM API Key 待填入

## 环境信息

| 项目 | 值 |
|------|-----|
| 主机 | Azure VM |
| 用户 | `<YOUR_VM_USER>` |
| SSH | `ssh <YOUR_VM_USER>@<YOUR_VM_HOST>`（via Cloudflare Access） |
| Node.js | v24.x（nvm 管理） |
| npm | 11.x |
| OpenClaw | 2026.4.x |
| 服务管理 | systemd (user session) |
| 监听端口 | 18789（默认） |
| Dashboard | `http://<VM_LAN_IP>:18789/` |

---

## 安装步骤

### 1. 检查 Node.js 环境

```bash
source ~/.nvm/nvm.sh
node --version
npm --version
```

> 注意：Node 通过 nvm 管理，需要先 `source ~/.nvm/nvm.sh` 才能使用。

### 2. 安装 OpenClaw

```bash
source ~/.nvm/nvm.sh
npm install -g openclaw
openclaw --version  # 验证安装
```

### 3. 初始化配置

创建 `~/.openclaw/openclaw.json`（替换 `<...>` 占位符为实际值）：

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
  "agents": {
    "defaults": {
      "model": {
        "primary": "copilot-gateway/claude-sonnet-4.6"
      },
      "timeoutSeconds": 600
    }
  }
}
```

### 4. 安装并启动 systemd 服务

```bash
source ~/.nvm/nvm.sh
openclaw gateway install           # 注册 systemd 服务，自动生成 gateway token
systemctl --user start openclaw-gateway.service
systemctl --user enable openclaw-gateway.service  # 开机自启（可选）
```

### 5. 验证运行状态

```bash
source ~/.nvm/nvm.sh
openclaw gateway status
# 期望输出：Runtime: running ... RPC probe: ok
```

---

## 待配置项

| 项目 | 说明 | 填入位置 |
|------|------|----------|
| Discord Bot Token | 新建 Bot 的 Token | `plugins.entries.discord.token` |
| Copilot Gateway API Key | 由管理员分配 | `models.providers.copilot-gateway.apiKey` |
| M2 模式配置 | 后续指定 | `gateway.m2` 或相关字段 |

---

## Discord Bot 配置（待完成）

1. 前往 [Discord Developer Portal](https://discord.com/developers/applications)
2. 创建新 Application
3. 进入 Bot 页面，生成 Token
4. 将 Token 填入配置文件的 `plugins.entries.discord` 段
5. 重启服务：`systemctl --user restart openclaw-gateway.service`

---

## 已知注意事项

1. **nvm PATH 警告**：systemd 服务配置中 Node 路径来自 nvm，升级 nvm 后可能失效。建议后续安装系统级 Node（`sudo apt install nodejs`）并更新服务配置。
2. **M2 模式**：尚未配置，等指定后再添加。
3. **与其他实例完全独立**：各 OpenClaw 实例无关联，独立运行。

---

## 常用命令速查

```bash
# 查看状态
source ~/.nvm/nvm.sh && openclaw gateway status

# 重启服务
systemctl --user restart openclaw-gateway.service

# 查看日志
tail -f /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log

# 停止服务
systemctl --user stop openclaw-gateway.service
```

---

*部署时间：2026-04-15*  
*操作人：碧瑶*  
*任务来源：小白调度*
