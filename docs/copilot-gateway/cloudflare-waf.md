# Cloudflare WAF 配置

> 最后更新：2026-04-08

## 问题背景

当 Copilot Gateway 部署在 Cloudflare Workers 上时，Cloudflare 的 **Super Bot Fight Mode** 会拦截来自 Anthropic SDK 的请求，返回 `403 Your request was blocked.`。

根本原因是 Anthropic SDK（如 `@anthropic-ai/sdk`）发送请求时默认携带 `User-Agent: Anthropic/JS x.x.x`，被 Cloudflare 的 Bot 检测识别为自动化流量并拦截。

## 影响范围

所有通过 Anthropic Messages API 格式 (`/v1/messages`) 访问 Gateway 的客户端都可能受影响，包括：

- **OpenClaw** — 使用 `anthropic-messages` API 格式时
- **Claude Code** — 直接连接时
- 其他使用 Anthropic SDK 的客户端

## 排查过程

1. 使用 curl 对比测试不同 `User-Agent`，确认 `Anthropic/JS` UA 触发 403：

    ```bash
    # 200 OK
    curl -s -o /dev/null -w "%{http_code}" \
      -X POST "https://copilot.xiaoyinggee.com/v1/messages" \
      -H "User-Agent: curl/8.0" ...

    # 403 Blocked
    curl -s -o /dev/null -w "%{http_code}" \
      -X POST "https://copilot.xiaoyinggee.com/v1/messages" \
      -H "User-Agent: Anthropic/JS 0.30.0" ...
    ```

2. 检查 Gateway 源码确认：**服务端没有任何 User-Agent 校验逻辑**，403 来自 Cloudflare 层。

## 解决方案

在 Cloudflare Dashboard 添加 WAF Custom Rule，对匹配的请求跳过 Bot 检测。

### 配置步骤

1. 登录 [Cloudflare Dashboard](https://dash.cloudflare.com)
2. 选择域名 → **Security** → **WAF** → **Custom rules**
3. 点击 **Create rule**
4. 配置如下：

    | 字段 | 值 |
    |------|----|
    | **Rule name** | `Allow Anthropic/Copilot SDK` |
    | **Expression** | 见下方 |
    | **Action** | Skip |

    **Expression（表达式）：**

    ```
    http.host eq "copilot.xiaoyinggee.com" and (http.user_agent contains "Anthropic" or http.user_agent contains "Copilot")
    ```

    **跳过的 WAF 组件（仅勾选以下两项）：**

    - [x] 所有超级自动程序攻击模式规则（Super Bot Fight Mode）
    - [x] 用户代理阻止（User Agent Blocking）

    !!! warning "安全提示"
        不要勾选其他跳过选项（如托管规则、速率限制等），保持最小跳过范围。表达式中限定了 `http.host`，确保规则只对 Gateway 子域生效。

5. 点击 **Deploy**

### 临时 Workaround

如果无法修改 Cloudflare 配置，可以在客户端覆盖 User-Agent。例如在 OpenClaw 的 provider 配置中：

```json
{
  "headers": {
    "User-Agent": "copilot-proxy/1.0"
  }
}
```

这会覆盖 Anthropic SDK 的默认 UA，绕过 Bot 检测。但推荐从 Cloudflare 侧解决。
