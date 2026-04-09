# Web 搜索与网页浏览方案

> 最后更新：2026-04-08

## 问题背景

通过 Copilot Gateway 使用 Claude Code 时，内置的 `WebSearch` 和 `WebFetch` 工具会被 **Copilot API 服务端禁用**，返回以下错误：

```
WebSearch: "The use of the web search tool is not supported."
WebFetch:  "Unable to verify if domain xxx is safe to fetch."
```

这是因为这两个工具由 Anthropic 服务端执行（请求从 Anthropic 的服务器发出），而 Copilot API 不提供相应的后端能力。`curl` 等本地命令不受影响，因为它们走本地网络。

## 解决方案：Playwright MCP + CLI

使用微软官方的 [Playwright MCP](https://github.com/microsoft/playwright-mcp) 和 [Playwright CLI](https://github.com/microsoft/playwright-cli)，在本地运行浏览器，完全绕过 Copilot API 的限制。

两种模式同时安装，由 AI 根据场景自动选择：

```
简单读网页 → Playwright CLI（省 token）
复杂多步交互 → Playwright MCP（持久状态）
```

### 两种模式对比

| 维度 | MCP 模式 | CLI 模式 |
|------|---------|---------|
| **调用方式** | MCP tools（`browser_navigate` 等） | Bash 调用 `playwright-cli` |
| **Token 消耗** | 较高（30+ tool schema 常驻上下文） | 低（只用 Bash，snapshot 存文件） |
| **浏览器状态** | MCP server 维持持久连接 | 通过 session 机制保持（`-s=`） |
| **适合场景** | 登录、填表、多页面跳转 | 打开 URL 读内容、搜索 |

!!! tip "Prompt Caching 与 Token 开销"
    MCP 的 tool schema（约 2-3k tokens）在每次 API 请求中都会发送。如果你的 Gateway 实现了 **Prompt Caching**，这部分会被缓存（首次全价，后续命中缓存），开销可以忽略。如果没有 caching，每次对话都会增加固定开销。

## 安装

### 1. Playwright CLI

```bash
npm install -g @playwright/cli@latest

# 验证
playwright-cli --help
```

### 2. Playwright MCP

在 `~/.claude/.mcp.json` 中添加：

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest", "--headless"]
    }
  }
}
```

添加后需要**重启 Claude Code** 才能加载 MCP server。

!!! note "headed 模式"
    去掉 `--headless` 参数可以弹出浏览器窗口，方便调试和观察 AI 的操作过程。

## 使用示例

### CLI 模式：读取网页内容

```bash
# 打开页面
playwright-cli open https://example.com --headless

# 获取页面结构化内容（accessibility tree）
playwright-cli snapshot --depth=4

# 完成后关闭
playwright-cli close
```

snapshot 会输出页面的无障碍树结构，AI 可以直接理解内容，不需要 vision 模型。

### CLI 模式：搜索信息

```bash
# 打开搜索引擎
playwright-cli open https://www.google.com --headless

# 填入搜索词并提交
playwright-cli fill 'textarea[name="q"]' "Playwright MCP tutorial"
playwright-cli press Enter

# 获取搜索结果
playwright-cli snapshot --depth=4

# 点击某个结果
playwright-cli click <ref>

# 读取目标页面
playwright-cli snapshot --depth=4

# 关闭
playwright-cli close
```

### CLI 模式：多 session 管理

```bash
# 创建命名 session
playwright-cli -s=research open https://docs.anthropic.com --headless

# 另一个 session
playwright-cli -s=github open https://github.com --headless

# 列出所有 session
playwright-cli list

# 关闭所有
playwright-cli close-all
```

### MCP 模式：需要登录的场景

当需要保持登录态、处理 Cookie、多步表单交互时，使用 MCP 工具：

1. `browser_navigate` — 打开目标 URL
2. `browser_snapshot` — 获取页面内容
3. `browser_fill_form` — 填写表单
4. `browser_click` — 点击按钮
5. `browser_evaluate` — 执行 JavaScript

MCP 模式的浏览器实例在整个对话期间保持活跃，登录状态不会丢失。

## CLAUDE.md 配置

在 `~/.claude/CLAUDE.md` 中添加选择规则，让 AI 自动判断使用哪种模式：

```markdown
# Playwright (Web browsing)

两种模式可用，根据场景选择：

- **CLI 模式**（默认）：通过 Bash 调用 `playwright-cli`。适合简单的网页读取。
- **MCP 模式**：通过 MCP tools。适合需要持久浏览器状态的多步交互。

选择规则：
- 只读一个页面内容 → CLI（省 token）
- 需要登录/填表/多页面跳转/保持 session → MCP
- 搜索信息 → CLI 打开搜索引擎 → snapshot 结果
```

## 工作原理

Playwright 不是通过截图让 AI "看" 页面，而是读取浏览器的 **accessibility tree**（无障碍树），将 DOM 转成结构化文本。

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Claude Code │────▶│  Playwright  │────▶│   Browser    │
│  (AI Agent)  │◀────│  MCP / CLI   │◀────│  (Chromium)  │
└──────────────┘     └──────────────┘     └──────────────┘
      结构化文本        accessibility tree      网页 DOM
```

优势：

- 不需要 vision 模型，纯文本交互
- 速度快，token 消耗远低于截图方案
- 操作精确——通过元素引用（ref）定位，不是像素坐标
- 浏览器跑在本地，不受 API 提供商的网络限制

## 局限性

- **JavaScript 渲染**：需要等待 SPA 页面加载完成，某些动态内容可能需要额外等待
- **验证码**：无法自动处理 CAPTCHA，遇到时需要切换到 headed 模式手动处理
- **Token 消耗**：复杂页面的 snapshot 可能很大（5k-15k tokens），使用 `--depth` 参数控制深度
- **并发限制**：CLI 模式下多个 session 共享资源，建议不超过 3 个并发 session

## 相关文章

- [Copilot Gateway 介绍](introduction.md) — Gateway 的基本架构与部署
- [Cloudflare WAF 配置](cloudflare-waf.md) — 解决 Bot 检测拦截问题
