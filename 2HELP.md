# OpenClaw 使用说明书

- **产品：** OpenClaw（命令 `openclaw`）
- **对应版本：** 2026.9.5
- **官方文档：** https://docs.openclaw.ai
- **配套：** [1DESIGN.md](1DESIGN.md) 设计、[3SOLUTION.md](3SOLUTION.md) 优化方案、[4SOURCE.md](4SOURCE.md) 源码清单

本手册面向安装、配置与日常操作。默认把 Gateway 当作本机控制面；把通道当作你已经在用的聊天软件。

---

## 1. 整体功能概述

OpenClaw 在你自己的电脑上跑一个 **Gateway**。助手通过 Discord、Telegram、Slack、WhatsApp、iMessage 等通道，以及浏览器 Control UI、TUI、macOS/iOS/Android/Linux 伴侣与你对话。状态、记忆和凭据在本机 SQLite 与工作区文件里。模型（Claude、GPT、本地 Ollama 等）和 agent harness（内置 OpenClaw、Codex、Claude CLI 等）可替换。

你能做的事：

- 在已有聊天软件里让助手做事（查资料、跑命令、管日历草稿、写代码、控浏览器）
- 用 Control UI / TUI 直接聊
- 用技能、插件、MCP、Cron、心跳做自动化
- 用配对与审批控制谁能私聊、哪条命令能在主机执行
- 用节点把手机相机、屏幕、位置接到同一 Gateway

**不是：** 基金会托管的付费云；默认也不会把对话内容发回项目方。提示会发到你配置的模型提供商和聊天平台。

---

## 2. 安装

### 2.1 推荐安装脚本

macOS / Linux / WSL2：

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

Windows PowerShell：

```powershell
iwr -useb https://openclaw.ai/install.ps1 | iex
```

脚本会在需要时配置受支持的 Node.js（**Node 24.16+ 或 26.1+，推荐 26**）。

### 2.2 已有 Node 时用 npm

```bash
npm install -g openclaw@latest --allow-scripts=openclaw
```

npm 11.15 及更早不要加 `--allow-scripts=openclaw`。仓库开发请用 `pnpm install`，不要在仓库根用 `npm install`。

### 2.3 其他路径

Docker、Nix、从源码构建见 https://docs.openclaw.ai/install 。源码开发：

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
pnpm install
pnpm build
pnpm ui:build
pnpm openclaw --help
```

不要用 `node --import tsx src/index.ts` 跑 CLI；请用 `pnpm openclaw` 或 `pnpm dev`。

---

## 3. 快速开始

1. 安装脚本通常会打开引导。若用包管理器安装，执行：

```bash
openclaw onboard --install-daemon
```

2. 完成模型凭据、工作区、Gateway、（可选）通道。

3. 检查：

```bash
openclaw gateway status
openclaw dashboard
```

4. 在 Control UI 发一条消息，确认助手回复。

5. 连接通道后，用 `openclaw pairing approve <channel> <code>` 批准未知私聊发送者。

---

## 4. 各功能详细使用说明

### 4.1 引导与配置向导

| 命令 | 作用 |
|------|------|
| `openclaw onboard` | 完整引导：认证、模型、Gateway、工作区、通道、技能 |
| `openclaw onboard --install-daemon` | 同时安装系统服务 |
| `openclaw setup` | 引导未完成时的对话式设置 |
| `openclaw configure` | 交互改凭据、通道、Gateway、agent 默认 |
| `openclaw config` | 无交互 get/set/patch/unset/file/schema/validate |
| `openclaw doctor` | 健康检查 |
| `openclaw doctor --fix` | 迁移旧配置/状态并尽量修复 |

示例：

```bash
openclaw config get agents.defaults.workspace
openclaw config set agents.defaults.heartbeat.every "2h"
openclaw config unset plugins.entries.brave.config.webSearch.apiKey
openclaw config schema
```

配置文件：`~/.openclaw/openclaw.json`（JSON5，可注释）。也可用 `OPENCLAW_CONFIG_PATH` 指向真实文件，**不要用符号链接当写入目标**（原子 rename 会替换链接对象）。

### 4.2 Gateway 与服务

```bash
openclaw gateway              # 前台，日志 stdout
openclaw gateway status
openclaw daemon status        # launchd/systemd/schtasks
openclaw health --json
openclaw logs --follow
openclaw dashboard            # 打开 Control UI（带当前 token）
```

默认绑定 `127.0.0.1:18789`。远程请用 Tailscale、VPN 或 SSH 隧道，不要把未加固的 Gateway 暴露到公网。

```bash
ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
```

### 4.3 与助手对话

- **Control UI：** `openclaw dashboard` 或浏览器 `http://127.0.0.1:18789`
- **TUI：** `openclaw tui`；恢复最近会话 `openclaw resume`；本地别名 `openclaw chat`
- **一次性回合：** `openclaw agent --message "…"`（`--local` 走嵌入式，不经已运行 Gateway）
- **通道：** 在已连接的 Telegram/Slack 等里直接发消息

常用斜杠命令（聊天内）：

- `/compact [焦点说明]` 强制压缩上下文
- `/queue steer|followup|collect|interrupt` 队列模式
- `/steer <消息>` 注入当前回合
- `/model` 切换模型
- `/stop` 停止当前 run

### 4.4 会话

```bash
openclaw sessions
openclaw sessions --help
```

Direct 默认进入共享 `main`；群默认隔离。多 agent 用 `openclaw agents` 管理工作区与路由。Incognito 会话不写入常驻花名册/Activity recap。

### 4.5 通道

```bash
openclaw channels add
openclaw channels list
openclaw pairing approve telegram <code>
openclaw directory
```

核心自带 A2A、Reef、Telegram、WebChat。其他官方通道：

```bash
openclaw plugins install @openclaw/discord
openclaw plugins install @openclaw/whatsapp
# 或在 onboard / channels add 时按需安装
```

**安全：** 把入站消息当不可信输入。DM 默认配对。群里通常要 mention 才回复。

### 4.6 模型

```bash
openclaw models
openclaw models status
```

配置 `agents.defaults.model.primary`（如 `anthropic/claude-sonnet-4-6`）。凭据和默认模型是两回事：设了 `ANTHROPIC_API_KEY` 只解决认证。失败时按 `fallbacks` 与提供商内 auth profile 轮换。

- `utilityModel`：短内部任务（标题、Activity recap）
- `decisionModel`：插件决策，不进入聊天选择器
- `imageModel` / `pdfModel` / `mediaModels`：多模态与生成

本地模型：Ollama、vLLM、SGLang、llama.cpp、LM Studio，或任意 OpenAI/Anthropic 兼容端点。

### 4.7 工具、技能、插件

```bash
openclaw skills
openclaw plugins
openclaw plugins install @openclaw/<id>
openclaw mcp
```

技能是工作区或 ClawHub 上的流程文档+脚本。插件分 **code plugin**（运行时钩子、通道、提供商）和 **bundle plugin**（技能、MCP、配置包）。记忆插件同时只能启用一个。

主机 `exec` 默认在主会话上跑，**接其他用户或公网前先读沙箱与安全文档**。危险命令走审批：`openclaw approvals`。

### 4.8 自动化

```bash
openclaw cron
openclaw hooks
openclaw webhooks
openclaw tasks
```

Cron 经 Gateway 调度。心跳按 `agents.defaults.heartbeat.every`。内部 hooks 用工作区 `HOOK.md`（如 `command:new`）。HTTP webhook 是“外部触发工作”，不是 agent 循环订阅。

### 4.9 记忆

工作区文件：`AGENTS.md`（指令，仅人写）、`USER.md`/`MEMORY.md`（策核心）、`memory/YYYY-MM-DD.md`（日记）。Agent 会搜索情节记忆；Dreaming 在后台巩固。

```bash
# 记忆 CLI 由 memory-core 插件注册
openclaw memory status
openclaw memory search "项目约定"
```

不要指望心跳/cron 会话自动晋升长期记忆。

### 4.10 节点与伴侣

```bash
openclaw qr                 # 移动端配对码
openclaw nodes
openclaw devices
openclaw connect            # 本机作为 node 接入
openclaw node               # headless node host
```

配对后可用相机、屏幕、位置、语音。macOS 菜单栏应用提供 Canvas 小部件。

### 4.11 浏览器、沙箱、工作树

```bash
openclaw sandbox
openclaw worktrees
openclaw browser --help     # 浏览器插件 CLI
```

沙箱是 Docker 隔离的 agent 执行。Worktree 给会话隔离 git 工作区。

### 4.12 安全、密钥、审计

```bash
openclaw security
openclaw secrets
openclaw audit
openclaw approvals
```

密钥进 secret store，不要把真 token 写进文档或 git。`audit` 只看元数据账本，不含 prompt 正文。

### 4.13 备份、数据库、更新

```bash
openclaw backup create
openclaw database preflight
openclaw update
openclaw status
openclaw status --all       # 可分享的脱敏报告
openclaw status --deep
```

升级前备份。Schema 只向前，**不能把新库拿去给更旧的 OpenClaw 用**。

### 4.14 卸载与重置

```bash
openclaw reset              # 清配置/状态，保留 CLI
openclaw uninstall          # 卸服务与本地数据
```

### 4.15 开发者命令

```bash
pnpm test
pnpm check:changed
pnpm build && pnpm ui:build
openclaw docs               # 搜在线文档
```

---

## 5. 配置说明

### 5.1 文件位置

| 路径 | 内容 |
|------|------|
| `~/.openclaw/openclaw.json` | 主配置 |
| `~/.openclaw/state/openclaw.sqlite` | 全局状态 |
| `~/.openclaw/agents/<id>/agent/openclaw-agent.sqlite` | 每 agent 会话/记忆 |
| `~/.openclaw/agents/<id>/sessions/cold/` | 冷转录 `*.jsonl.zst` |
| `~/.openclaw/workspace` | 默认工作区（可改） |
| 环境变量 / `.env` | 密钥覆盖；服务环境必须能看到 |

### 5.2 最小配置

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
  channels: { telegram: { allowFrom: ["123456789"] } },
}
```

把 `telegram` 换成你的通道；`allowFrom` 按通道身份格式填写。

### 5.3 常见键

| 键 | 含义 |
|----|------|
| `agents.defaults.model.primary` | 主模型 `provider/model` |
| `agents.defaults.model.fallbacks` | 回退列表 |
| `agents.defaults.workspace` | 工作区 |
| `agents.defaults.timeoutSeconds` | 回合预算，默认 172800（48h），`0` 不限制执行（提供商活性仍适用） |
| `agents.defaults.maxConcurrent` | 全局并发 |
| `agents.defaults.compaction.mode` | `safeguard`（默认）或 `default` |
| `agents.defaults.compaction.enabled` | `false` 关闭主动压缩（溢出与 `/compact` 仍在） |
| `agents.defaults.heartbeat.every` | 心跳间隔 |
| `gateway.port` / `gateway.bind` | 监听 |
| `gateway.auth.mode` | token / password / trusted-proxy / none |
| `messages.queue.mode` | steer/followup/collect/interrupt |
| `update.checkOnStart` | `false` 关闭启动时版本检查与匿名统计相关上报 |
| `plugins.entries.<id>` | 插件启用与配置 |

完整字段以 `openclaw config schema` 和 Control UI Config 页为准。**未知键会导致 Gateway 拒绝启动。** 旧键由 `doctor --fix` 迁移，不要依赖静默别名。

### 5.4 热重载

Gateway 监视配置文件。`$include`、Nix 管理的配置、以及由**更新的 OpenClaw 写出**的配置不会自动迁移。

### 5.5 环境变量

提供商密钥常用 `ANTHROPIC_API_KEY`、`OPENAI_API_KEY` 等。systemd/launchd 用户服务读不到你交互 shell 的 `.env` 时，把环境写进服务单元或 secret store。`openclaw secrets` 是运行时控制面。

---

## 6. 故障排查 60 秒

```bash
openclaw status
openclaw status --all
openclaw gateway status
openclaw status --deep
openclaw logs --follow
openclaw doctor
openclaw health --verbose
```

Gateway RPC 挂了时看文件日志（日期与 profile 名会变）：

```bash
tail -f "/tmp/openclaw/openclaw-$(date +%F).log"
```

---

## 7. 常见问题 100 问

### 安装与首次运行

**Q1. OpenClaw 是什么？**  
跑在你设备上的开源个人/团队 AI 助手，经 Gateway 接入聊天软件、UI 与节点。

**Q2. 是不是 OpenAI 的产品？**  
不是。由 OpenClaw Foundation 托管。OpenAI 是捐赠方之一，不是所有者。

**Q3. 有没有付费档或官方托管？**  
没有付费档、托管服务或项目代币。

**Q4. 默认会上传我的聊天吗？**  
不会上传给基金会。对话会去你配置的模型与通道。可选匿名功能统计默认关；每日版本检查可关。

**Q5. 支持哪些操作系统？**  
macOS、Linux、Windows（含 WSL2）。伴侣应用覆盖 macOS、iOS、Android、Linux，另有 Windows Hub。

**Q6. Node 版本不够怎么办？**  
使用官方安装脚本，或安装 Node 24.16+/26.1+。不要用不受支持的 Node 硬跑。

**Q7. `npm install -g openclaw` 失败？**  
检查 npm 版本与 `--allow-scripts=openclaw`；公司镜像可能剥生命周期脚本。看安装指南的 lifecycle 合同。

**Q8. 仓库里 `npm install` 报错？**  
根仓库是 pnpm workspace，请用 `pnpm install`。

**Q9. onboard 卡在模型步骤？**  
先保证密钥在当前进程可见：`openclaw models status`。API key 与 OAuth（如 Codex）路径不同。

**Q10. 刚装完先做什么？**  
`openclaw gateway status` → `openclaw dashboard` → 发一条消息 → 再加通道并配对。

**Q11. 安装 daemon 失败（Linux）？**  
用户 systemd 可能需要 lingering：`loginctl enable-linger $USER`。Doctor 会提示。

**Q12. Windows 上 Gateway 起不来？**  
看任务计划/`openclaw daemon status`、端口占用、Windows Defender、Node 架构（x64 vs arm）。

**Q13. Docker 里功能变少？**  
容器默认更受限。要完整 exec/浏览器需按沙箱/Docker 文档加 cap 与卷。

**Q14. Nix 管理配置会被 doctor 改掉吗？**  
自动迁移会跳过 Nix 管理配置；请在配置源里改。

**Q15. 能不能多开几个 Gateway？**  
同一状态目录只允许一个写者（state lease）。要用多实例请分开状态目录/profile。

### 配置

**Q16. 配置文件在哪？**  
默认 `~/.openclaw/openclaw.json`。

**Q17. 改完要重启吗？**  
多数键热重载；鉴权/绑定/插件加载类可能要重启 Gateway。

**Q18. 未知键导致无法启动？**  
严格 schema。用 `openclaw doctor --fix` 或对照 `config schema` 删未知键。

**Q19. `config.apply` 把配置弄坏了？**  
看 `.bak` 环；`doctor --fix`；不要把损坏文件直接覆盖唯一备份。

**Q20. 如何设工作区？**  
`agents.defaults.workspace`，或 `openclaw config set agents.defaults.workspace "~/path/to/skills"`。

**Q21. SOUL.md 太大？**  
策核心有注入预算。超长会截断或挤掉其他上下文；把细节放到 `memory/` 日记并靠搜索召回。

**Q22. AGENTS.md 应该放哪？**  
Agent 工作区根，与 `SOUL.md`/`USER.md`/`MEMORY.md` 一起。

**Q23. 环境变量不生效？**  
确认 Gateway **服务**环境，而不是只在当前终端 export。用 `status --all` 看脱敏后的探测。

**Q24. `.env` 会被加载吗？**  
部分 CLI 路径会；daemon 不一定。关键密钥放 secret store 或服务 Environment。

**Q25. 如何关更新检查？**  
`update.checkOnStart: false`。

**Q26. JSON5 注释合法吗？**  
合法。也允许尾逗号。

**Q27. 配置可以用 `$include` 吗？**  
可以，但自动迁移不处理 include/多文件所有权冲突，需手改 owner 文件。

**Q28. Control UI 和 CLI 改的是同一份配置吗？**  
是，都写 canonical 文件（经 Gateway）。

**Q29. 高级设置找不到？**  
Settings 里展开 Advanced；搜索会打开对应高级分组。

**Q30. 如何限制允许的模型？**  
`agents.defaults.modelPolicy.allow`，支持 `provider/*` 通配。空或省略表示不限制。

### Gateway、端口、远程

**Q31. 端口 18789 被占用？**  
`openclaw gateway status` 看谁在听；改 `gateway.port` 或停掉旧进程。

**Q32. 提示 already running？**  
已有 Gateway 占用状态目录。用 `gateway status` 区分“服务活着”和“RPC 可达”。

**Q33. 远程 VPS 怎么连本机 UI？**  
SSH 隧道或 Tailscale；保持鉴权与配对。不要 `auth.mode: none` 对公网。

**Q34. Tailscale 还要配对吗？**  
要。非 loopback（含 tailnet）需要显式配对。

**Q35. trusted-proxy 模式何时用？**  
仅当前面有你控制的身份代理，且 Gateway 只信该代理头。

**Q36. Health 一直失败？**  
`openclaw health --verbose` 看探测 URL 与配置路径是否指向另一 profile。

**Q37. 日志在哪？**  
`openclaw logs`；文件日志在 `/tmp/openclaw/` 一类路径（随 OS/profile 变）。服务日志在 journalctl/launchd。

**Q38. RPC 通但通道不通？**  
通道是 Gateway 内连接。看通道插件是否启用、凭据、`channels` 状态。

**Q39. 多机共享一个 Gateway？**  
可以：团队部署共享 Gateway。配置与 ACL 决定谁能看见哪些会话。

**Q40. 如何确认用的是哪份配置？**  
`gateway status` 与 `health --verbose` 打印配置路径。

### 模型与认证

**Q41. 设了 ANTHROPIC_API_KEY 但模型不对？**  
那只启用认证。改 `agents.defaults.model.primary`。

**Q42. `No credentials found for profile "anthropic:default"`？**  
Gateway 在 SQLite auth store 里找不到该档案。重新 onboard 或写入 auth profile。

**Q43. Codex / ChatGPT 订阅怎么用？**  
走 OpenAI OAuth 档案 + `openai/*` 模型 ref；runtime 可能选 Codex app-server。`openai/*` 前缀本身不会单单选中 Codex。

**Q44. 想用 Claude CLI 而不是 API？**  
模型 ref 仍是 `anthropic/...`，在模型策略上设 `agentRuntime.id: "claude-cli"`。

**Q45. GitHub Copilot 会自动当 runtime 吗？**  
不会。`github-copilot/*` 的外部 Copilot runtime 必须显式选择。

**Q46. 模型 failover 不切换？**  
看 fallbacks、allow 列表、冷却、以及是“模型失败”还是“认证失败”（后者先转 profile）。

**Q47. utility 模型产生了额外账单？**  
会。标题/recap 是另一次短调用。可设空字符串关掉 utility 路由（标题生成仍可能走主模型）。

**Q48. 本地 Ollama 连不上？**  
确认 Ollama 监听地址对 Gateway 可达，插件已装，模型已 pull。

**Q49. 自定义兼容端点？**  
用提供商插件的 base URL 配置；不要把任意 HTTP 明文端点标成官方 OpenAI。

**Q50. `/model` 只改当前会话？**  
默认会话范围；`modelSelectionScope` 可改。显式 scope 以命令为准。

### 会话、队列、停止

**Q51. 如何开新会话？**  
Control UI New Session，或通道里新 thread/群（群默认隔离）。

**Q52. 如何重置上下文但留记忆文件？**  
会话 reset/rollover 开新 `session_window`；工作区文件仍在。

**Q53. 任务停不下来？**  
聊天 `/stop`；仍跑则 `openclaw sessions` 找 run，或重启 Gateway（会中断执行）。审批中的 exec 要在 approvals 里处理。

**Q54. steer 和 followup 区别？**  
steer 注入当前 run；followup 等当前结束再开一轮；collect 合并多条；interrupt 中止再跑最新。

**Q55. 为什么回复像被切成几段？**  
`partial`/`block` 流式 + steer 会在边界 finalize 预览。属预期。

**Q56. 群里不回？**  
需要 mention 或策略允许的静默规则；检查 allowlist 与 bot 权限。

**Q57. 几个聊天共用一个大脑？**  
Direct 常进 `main`。要隔离就用独立 agent 或非 main 会话。

**Q58. 上下文满了？**  
自动压缩；或 `/compact`。重要事实应写入 MEMORY.md。

**Q59. compaction 失败后历史丢了吗？**  
Safeguard 失败不写摘要、保留原历史。磁盘全文仍在。

**Q60. `agent.wait` 返回 timeout 是不是 run 死了？**  
不是。只是等待超时。用同一 `runId` 再等。

### 通道与配对

**Q61. 未知用户私聊？**  
默认配对。`openclaw pairing approve <channel> <code>`。

**Q62. WhatsApp 要扫码？**  
Baileys 会话由**这一台** Gateway 独占。不要两台机器同时开同一 WhatsApp。

**Q63. Telegram 在测试服？**  
维护者级 E2E 用 Telegram Test Server；普通用户用正式 Bot API token。

**Q64. Discord 语音？**  
默认用 bundled `libopus-wasm`，不必编原生 opus。

**Q65. iMessage 只能 macOS？**  
iMessage 插件依赖 Apple 主机能力；Linux Gateway 不能直接充当 iMessage 端点。

**Q66. 通道插件装了但没出现？**  
重启 Gateway；检查 `plugins.entries` 启用与 manifest。发现是轻路径，执行才加载重运行时。

**Q67. 如何查聊天 ID？**  
`openclaw directory`。

**Q68. 媒体发出去了但聊天里没有？**  
通道媒体限制、沙箱路径、或技能写了文件但没走 message 工具。看 FAQ media 页与日志。

**Q69. 多账户？**  
通道账户配置 + routing bindings。写操作必须打到绑定目标，失败不会偷换账户。

**Q70. 外部 WeChat 插件？**  
仓库外维护。按该插件自己的安装说明，仍走 Plugin SDK。

### 记忆、沙箱、技能

**Q71. 记忆为什么忘事？**  
情节日记不自动进上下文。要写入 MEMORY.md 或等 Dreaming。心跳内容不会晋升。

**Q72. 语义搜索要不要 OpenAI key？**  
取决于记忆插件与 embedding 提供商，不是硬编码必须 OpenAI。

**Q73. 记忆永久保存吗？**  
文件在直到你删。索引可重建。`memory forget` 按会话墓碑排除。

**Q74. 如何自定义技能又不弄脏 git？**  
技能放工作区或 ClawHub，不要改捆绑 `skills/` 除非你在开发本仓库。

**Q75. 自定义技能目录？**  
可以，见技能文档的 extra dirs 配置。

**Q76. Linux 能跑 macOS 专用技能吗？**  
不能调用 Apple 专有能力；用节点把 mac 接过来，或换跨平台技能。

**Q77. Cron 不触发？**  
Gateway 必须在跑；检查 cron 作业启用、时区、isolated agent 超时。

**Q78. Cron 触发了但通道没消息？**  
投递策略、通道断开、或 isolated run 切了模型/重试。看 cron receipts。

**Q79. 如何把主机目录挂进沙箱？**  
沙箱 bind 配置；只挂需要的路径。

**Q80. 主会话 exec 太危险？**  
给陌生人/群启用 Docker sandbox；保留审批。

### 安全与权限

**Q81. 可以把 Gateway 放到公网 IP 吗？**  
强烈不建议无隧道、无鉴权、无配对。先读 security 与 exposure runbook。

**Q82. `auth.mode: none`？**  
仅私有 ingress。公网等于敞开控制面。

**Q83. 插件风险怎么评估？**  
代码插件进进程；bundle 更窄。优先 ClawHub 有 provenance 的包。`openclaw security` 扫常见脚枪。

**Q84. 审批点了允许但没执行？**  
live authority 可能已过期或 run 已结束。看 approvals 状态与日志。

**Q85. 审计日志含对话正文吗？**  
不含。只有元数据与结果码。

**Q86. 如何轮换 Gateway token？**  
改 `gateway.auth` 并重启；更新所有客户端。Control UI dashboard 命令会带当前 token。

**Q87. 团队共享会不会看见我的本地文件？**  
共享的是该 Gateway 的工作区与会话，不是你另一台笔记本的家目录，除非你把节点/exec 指过去。

**Q88. Secret 误提交了？**  
按 SECURITY.md 报告；轮换密钥；不要只靠 git rewrite。

### 更新、备份、数据

**Q89. 更新会不会弄坏数据库？**  
更新前备份。新 schema 不能给旧二进制用。`database preflight` 可先检查。

**Q90. 如何备份？**  
`openclaw backup create`；含 SQLite 与配置。冷转录文件也要纳入。

**Q91. 降级到旧版本？**  
官方不支持带着新 schema 降级。只能从备份恢复到旧库。

**Q92. `update` 显示 abandoned？**  
账本认为驱动进程已死。看 `update status`；可 `update repair` 后重试。

**Q93. 更新时 Gateway 会停多久？**  
账本记录 downtime。目标是尽量让旧 Gateway 跑到切换完成。

**Q94. 数据都在本地吗？**  
控制面与 agent DB、工作区在本地。模型提供商与聊天平台仍会收到你发出的内容。

**Q95. 如何迁到新机器？**  
备份、在新机恢复、重做通道登录（WhatsApp 等设备绑定）、重配对节点。

### 界面与开发

**Q96. 改了 ui/src 但页面没变？**  
Gateway 提供预构建 `dist/control-ui`。需 `pnpm ui:build`。

**Q97. TUI 和 Control UI 状态不一致？**  
都应以 Gateway 为准。强制刷新；检查是否连了不同 Gateway。

**Q98. 如何搜索文档？**  
`openclaw docs` 或 https://docs.openclaw.ai 。

**Q99. 想加新通道？**  
写插件走 `openclaw/plugin-sdk`，不要改 `src/channels` 私有文件。先看是否已有官方插件。

**Q100. 还是搞不定？**  
`openclaw status --all` 贴到 Discord（https://discord.gg/clawd）或 GitHub issue chooser。漏洞走 SECURITY.md，不要公开贴密钥。

---

## 8. 日常建议

1. 先本地 loopback 跑通，再开通道。
2. 给 DM 配对，给群最小权限。
3. 重要长期记忆写进 `MEMORY.md`/`USER.md`，不要只靠聊天历史。
4. 升级前 `backup create`。
5. 把 `openclaw doctor` 当作常规体检，而不是只在灾难时才跑。
