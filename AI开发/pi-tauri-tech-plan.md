# 技术方案：基于 Tauri + pi 的通用 AI 工作台（Mac）

## 1. 关键决策与依据

| 维度 | 决策 | 依据 |
|------|------|------|
| 产品形态 | 通用 AI 工作台 | 面向开发者与高级用户 |
| pi 复用层 | `pi-agent-core` + `pi-ai` | agent loop + LLM 统一 API 复用 |
| 工具实现位置 | **Tauri Rust 主进程**（不放在 sidecar） | 唯一副作用入口，权限模型清晰 |
| Sidecar 运行时 | **Bun 单二进制**（`bun build --compile`） | arm64-only，体积最优 |
| 分发 | 官网 DMG + Developer ID 签名/公证 | 无沙箱约束 |
| LLM 接入 | **纯 BYOK** | 无后端 |
| Skill 系统 | 自研，参考 OpenCode skills 模式 | 用户需求 |
| MCP 支持 | MCP servers 由 **Bun sidecar** 启动管理 | MCP SDK 是 Node 生态 |
| 数据存储 | **SQLite**（Tauri 侧） + 文件 | 会话/消息/审批规则 |

> ⚠️ 因为工具实现全部走 Rust，**sidecar 里的工具是"声明 + 转发"薄壳**，副作用只在 Rust 主进程发生。这是整个架构最关键的设计选择。

---

## 2. 进程模型与架构图

```
┌──────────────────────────────────────────────────────────────┐
│  React UI (WebView, WebKit)                                  │
│  ├─ 对话流 / Markdown / 代码块 / Diff 预览                   │
│  ├─ 工具调用审批弹窗（read/write/shell 等风险分级）          │
│  ├─ 多会话侧边栏 + 项目工作区（文件树、上下文选择）          │
│  ├─ Skills 管理 / MCP 配置 / Settings（API Key 走 Keychain） │
│  └─ 菜单栏常驻 + 全局快捷键唤起                              │
└────────────────┬─────────────────────────────────────────────┘
                 │ Tauri invoke() / event listen
┌────────────────▼─────────────────────────────────────────────┐
│  Tauri Rust 主进程（信任根 / 唯一副作用入口）                │
│  ├─ Sidecar 进程管理（spawn/kill/health/restart）            │
│  ├─ stdio JSON-RPC 桥（与 Bun sidecar 双向通信）             │
│  ├─ 工具实现：fs / shell / grep / glob / edit / web_fetch…   │
│  ├─ 权限审批引擎 + 持久化白名单（按项目/工具/参数模式）     │
│  ├─ Skill 加载（扫描目录、注入到 sidecar 启动参数）          │
│  ├─ SQLite：sessions / messages / projects / approvals       │
│  ├─ 文件监视（项目工作区）+ 系统通知                         │
│  ├─ 全局快捷键、菜单栏、托盘、自动更新                       │
│  └─ Keychain 凭据管理（API Key 加密存储）                    │
└────────────────┬─────────────────────────────────────────────┘
                 │ JSON-RPC over stdio (line-delimited JSON)
┌────────────────▼─────────────────────────────────────────────┐
│  Bun Sidecar（单二进制，约 50-60MB）                         │
│  ├─ pi-agent-core：agent loop、tool calling、状态机          │
│  ├─ pi-ai：统一 LLM API（OpenAI/Anthropic/Google…）          │
│  ├─ 工具声明层：定义 schema，执行时 RPC 转发到 Rust          │
│  ├─ Skill 运行时：注入 system prompt / 注册 skill 提供的工具 │
│  ├─ MCP 客户端：spawn 子进程 MCP servers，桥接为工具         │
│  └─ 流式输出：LLM token 流 → 逐 token emit 到 Rust          │
└──────────────────────────────────────────────────────────────┘
```

---

## 3. 技术栈

### 3.1 前端

| 类别       | 选型                                                            |
| -------- | ------------------------------------------------------------- |
| 框架       | React 18 + TypeScript + Vite                                  |
| UI       | Tailwind CSS + **shadcn/ui** + Radix                          |
| 状态       | **Zustand**（UI 状态）+ **TanStack Query**（异步）                    |
| 路由       | TanStack Router                                               |
| Markdown | `react-markdown` + `remark-gfm` + `rehype-highlight`（或 Shiki） |
| 代码/Diff  | **Monaco Editor**（diff 模式）或 CodeMirror 6                      |
| 布局       | `react-resizable-panels`（侧边栏/工作区）                             |
| 表单       | React Hook Form + Zod                                         |
| 包管理      | pnpm                                                          |
| 图标       | lucide-react                                                  |

### 3.2 Tauri 2.0 + 必装插件

| 插件 | 用途 |
|------|------|
| `tauri-plugin-sql` | SQLite |
| `tauri-plugin-store` | 轻量 KV 配置 |
| `tauri-plugin-dialog` | 文件/目录选择 |
| `tauri-plugin-fs` | 文件访问（项目工作区） |
| `tauri-plugin-shell` | 命令执行（与 sidecar 解耦） |
| `tauri-plugin-global-shortcut` | 全局快捷键 |
| `tauri-plugin-clipboard-manager` | 剪贴板 |
| `tauri-plugin-notification` | 系统通知 |
| `tauri-plugin-updater` | 自动更新 |
| `tauri-plugin-os` / `tauri-plugin-process` / `tauri-plugin-log` | 系统级辅助 |
| `tauri-plugin-stronghold` 或自封装 Keychain | API Key 加密存储 |

### 3.3 Bun Sidecar

- **Bun runtime** + `bun build --compile --target=bun-darwin-arm64`
- `@earendil-works/pi-agent-core` + `@earendil-works/pi-ai`
- `@modelcontextprotocol/sdk`
- 自研 IPC 入口 + 工具声明层 + skill 加载器

---

## 4. 核心模块设计

### 4.1 IPC 协议（JSON-RPC over stdio，line-delimited JSON）

**Rust → Sidecar（请求/命令）**

```jsonc
{ "method": "session.start",        "id": 1, "params": { "sessionId": "...", "projectId": "...", "model": "...", "skills": [...], "systemPrompt": "..." } }
{ "method": "session.cancel",       "id": 2, "params": { "sessionId": "..." } }
{ "method": "user.message",         "id": 3, "params": { "sessionId": "...", "text": "...", "attachments": [...] } }
{ "method": "tool.approve",         "id": 4, "params": { "toolCallId": "...", "approved": true,  "allowAlways": false } }
{ "method": "mcp.server.start",     "id": 5, "params": { "id": "...", "cmd": "...", "args": [...], "env": {...} } }
{ "method": "shutdown",             "id": 6 }
```

**Sidecar → Rust（事件流，无 id）**

```jsonc
{ "event": "message.delta",   "params": { "sessionId": "...", "text": "..." } }
{ "event": "message.end",     "params": { "sessionId": "...", "usage": {...} } }
{ "event": "tool.request",    "params": { "toolCallId": "...", "name": "write_file", "args": {...}, "risk": "high", "preview": "..." } }
{ "event": "tool.result",     "params": { "toolCallId": "...", "ok": true, "output": "..." } }
{ "event": "error",           "params": { "sessionId": "...", "code": "...", "message": "..." } }
{ "event": "mcp.log",         "params": { "serverId": "...", "level": "info", "msg": "..." } }
```

> 设计要点：**审批流是异步的** — sidecar 在发出 `tool.request` 后阻塞等待 `tool.approve`，期间 LLM 流暂停。Rust 持有审批状态机，UI 监听事件渲染弹窗。

### 4.2 工具系统（声明在 sidecar，实现在 Rust）

**Sidecar 侧（声明薄壳）**

```ts
// 只负责 schema 定义 + 转发到 Rust 执行
defineTool({
  name: "write_file",
  description: "...",
  schema: { path: "string", content: "string" },
  risk: "high",
  execute: async (args, ctx) =>
    ctx.rpc.call("tool.execute", { name: "write_file", args })
});
```

**Rust 侧（真正执行 + 审批）**

```rust
// 收到 sidecar 的 tool.execute 请求 → 查权限规则 → 必要时弹 UI 审批 → 执行 → 返回结果
// 白名单：{ project: "/foo", tool: "write_file", pathGlob: "/foo/src/**", decision: "allow"|"deny"|"ask" }
```

**MVP 工具集**

| 工具 | 风险 | 说明 |
|------|------|------|
| `read_file` | 低 | 默认允许（项目目录内） |
| `list_files` / `glob` | 低 | 默认允许 |
| `grep` / `search` | 低 | 默认允许 |
| `write_file` | 高 | 必须审批，可选 diff 预览 |
| `edit_file` | 高 | 必须审批 + diff |
| `run_shell` | 高 | 必须审批，按命令模式白名单 |
| `web_fetch` | 中 | 默认允许或首次询问 |

### 4.3 Skill 系统（参考 OpenCode skills 模式）

**Skill 目录约定**

```
~/.config/<app>/skills/<skill-name>/SKILL.md      # 用户全局
<project>/.<app>/skills/<skill-name>/SKILL.md     # 项目级
<app-bundle>/skills/<skill-name>/SKILL.md         # 内置
```

**SKILL.md 结构（YAML front matter + Markdown）**

```yaml
---
name: check
description: Reviews code diffs and PRs
triggers: [code_review, pr_triage]
provides_tools: []           # 可选：skill 暴露的工具（脚本）
injects_prompt: true         # 是否把正文注入 system prompt
mcp:                         # 可选：skill 启动的 MCP server
  cmd: npx
  args: [-y, some-mcp-server]
---
（正文会拼到 system prompt 末尾）
```

**加载流程**

1. Rust 启动时扫描三个 skill 目录 → 解析 front matter → 写 SQLite
2. 启动 sidecar 时传入"已激活 skills 列表"
3. Sidecar 把每个 skill 的 prompt 正文拼到 system prompt；MCP server 由 sidecar spawn 并桥接为工具
4. UI 的 Skills 页面：开关、查看源码、安装（从 URL/zip）

### 4.4 MCP 集成

- MCP server 配置存在 SQLite（用户在 Settings 配）
- 由 **sidecar spawn 子进程**（`@modelcontextprotocol/sdk`）
- MCP 工具被桥接为 pi-agent-core 工具，统一走审批（默认中风险）
- 崩溃自动重启（指数退避），超过阈值标记为 unhealthy 在 UI 提醒

### 4.5 数据持久化

**SQLite Schema（核心表）**

```sql
projects(id, name, root_path, created_at, settings_json)
sessions(id, project_id, title, model, created_at, updated_at)
messages(id, session_id, role, content_json, parent_id, created_at)
tool_calls(id, session_id, tool_name, args_json, result_json, status, created_at)
approvals(id, project_id, tool_name, pattern, decision, created_at)  -- 白名单
skills(id, source, name, path, enabled, frontmatter_json)
mcp_servers(id, name, cmd, args_json, env_json, enabled)
settings(key, value_json)
```

**凭据**：API Key 走 **macOS Keychain**（`tauri-plugin-stronghold` 或直接调 `security` 命令封装），**不落盘明文**。

### 4.6 流式输出

```
LLM token → pi-ai stream → pi-agent-core → sidecar stdout (message.delta 事件)
  → Rust 接收 → Tauri event emit → React 订阅 → 追加到当前消息 DOM
```

- React 侧用 `useEvent` + 防抖（每 ~30ms 批量 flush）避免高频 re-render
- Markdown 在流式过程中增量解析（保留未闭合 code block 状态）

### 4.7 菜单栏 + 全局快捷键

- **菜单栏模式**：`tauri::tray` 创建 NSStatusItem，点击展开轻量窗口（独立 WebView 实例或 popover）
- **全局快捷键**：`tauri-plugin-global-shortcut` 注册 ⌘+Shift+Space（可配置）唤起主窗口
- 主窗口关闭默认隐藏到菜单栏而非退出（可配置）

---

## 5. 推荐目录结构

```
app/
├── src-tauri/                  # Rust 主进程
│   ├── src/
│   │   ├── main.rs
│   │   ├── sidecar/            # sidecar 进程管理 + IPC
│   │   ├── tools/              # 工具实现（fs/shell/search/edit/web）
│   │   ├── permission/         # 审批引擎 + 规则匹配
│   │   ├── project/            # 工作区/文件树/文件监视
│   │   ├── skill/              # skill 扫描/解析
│   │   ├── storage/            # SQLite + Keychain 封装
│   │   ├── tray.rs             # 菜单栏/全局快捷键
│   │   └── commands/           # 暴露给前端的 invoke 命令
│   ├── resources/sidecar/      # 编译产物（bun 二进制）
│   ├── tauri.conf.json
│   └── Cargo.toml
│
├── src/                        # React 前端
│   ├── routes/                 # TanStack Router（chat/projects/settings/skills）
│   ├── components/
│   │   ├── chat/               # 消息流/流式渲染/Markdown
│   │   ├── approval/           # 工具审批弹窗 + diff 预览
│   │   ├── workspace/          # 文件树/上下文选择
│   │   ├── sidebar/            # 会话列表
│   │   └── ui/                 # shadcn 组件
│   ├── stores/                 # Zustand
│   ├── hooks/                  # useIPCEvent / useSession / useStream
│   ├── lib/                    # ipc.ts / events.ts / markdown.ts
│   └── types/                  # 共享类型（与 Rust/sidecar 对齐）
│
├── sidecar/                    # Bun sidecar 源码
│   ├── src/
│   │   ├── index.ts            # stdio JSON-RPC 入口
│   │   ├── agent.ts            # 包装 pi-agent-core
│   │   ├── tools/              # 工具声明薄壳
│   │   ├── skills/             # skill 加载 + prompt 拼装
│   │   ├── mcp/                # MCP 客户端
│   │   └── providers/          # pi-ai 适配（按用户 BYOK 配置）
│   ├── package.json
│   └── tsconfig.json
│
├── packages/shared/            # 共享类型（IPC 协议、工具 schema）
├── scripts/
│   ├── build-sidecar.sh        # bun build --compile
│   └── dev.ts                  # 开发模式（hot 重载 sidecar）
└── package.json
```

---

## 6. 打包与签名

| 步骤 | 工具 |
|------|------|
| 编译 sidecar | `bun build sidecar/src/index.ts --compile --target=bun-darwin-arm64 --outfile src-tauri/resources/sidecar/app-sidecar` |
| Tauri 打包 | `tauri build` → 产出 `.dmg` + `.app` |
| 签名 | `codesign --deep --options=runtime --sign "Developer ID Application: ..."` |
| 公证 | `xcrun notarytool submit ...` |
| 装订 | `xcrun stapler staple ...` |
| 自动更新 | `tauri-plugin-updater` + 静态 manifest（GitHub Releases / Cloudflare R2） |

**预计体积**：DMG ~25-35MB（压缩后），安装后 ~60-70MB。

---

## 7. 开发阶段规划

### 阶段 0：原型验证（1-2 周）

- Tauri + React + shadcn 空壳跑起来
- Bun sidecar spawn 跑通 stdio 双向 JSON-RPC
- 跑通最小闭环：用户发消息 → sidecar 调 OpenAI → 流式回传到 UI
- **目标**：验证 sidecar 方案可行、pi-agent-core 在 Bun 下正常工作

### 阶段 1：MVP（1-2 个月）

- 流式对话 + Markdown 渲染 + 代码高亮
- BYOK 配置（OpenAI/Anthropic，Keychain 存储）
- 基础工具集（read/write/shell/grep）+ 审批 UI
- 多会话管理 + SQLite 持久化
- 项目工作区 v1（绑定目录、文件树、简单上下文选择）
- 菜单栏 + 全局快捷键

### 阶段 2：完整功能（2-3 个月）

- 工具集完善（edit + diff 预览、web_fetch、glob）
- Skill 系统 v1（扫描、prompt 注入、UI 管理）
- MCP 服务器集成（配置、自动重启、日志）
- 文件监视与项目工作区 v2

### 阶段 3：打磨与扩展（持续）

- 自动更新、崩溃恢复
- 多 LLM provider 全覆盖（pi-ai 支持的所有）
- 高级 skill（脚本工具、远程 skill 安装）
- 主题、国际化、可访问性
- 性能优化（流式渲染、大文件、长会话）

---

## 8. 关键风险与对策

| 风险 | 影响 | 对策 |
|------|------|------|
| `pi-agent-core` 可能与 CLI/TUI 紧耦合 | 高 | 阶段 0 先做 PoC，必要时 fork 一个精简版 |
| Bun compile 下 pi 依赖兼容性问题 | 中 | 阶段 0 验证；fallback 到 `pkg` 打 Node |
| MCP server 不稳定 | 中 | 进程监护 + 自动重启 + UI 健康指示 |
| 审批 UX 打断流 | 中 | 引入"风险等级 + 项目白名单 + 持久化决策"，减少重复询问 |
| Gatekeeper 签名/公证流程踩坑 | 中 | 阶段 1 末就走通完整签名链路，避免后期返工 |
| pi 版本升级跟不上 | 低 | 锁定版本，跟随上游节奏升级 |

---

## 9. 待确认的开放项

1. **产品名/品牌**：会影响 bundle id、目录命名、UI 文案
2. **MVP 工具集范围**：上面列的够用吗？是否需要 `web_search`、`image_gen` 之类
3. **模型默认推荐**：BYOK 时默认推荐哪个 provider/model 作为初始
4. **是否要云端同步**：会话历史是否需要 iCloud/自建同步（MVP 建议先不做）
5. **多窗口**：是否支持多个独立会话窗口（影响窗口管理设计）
