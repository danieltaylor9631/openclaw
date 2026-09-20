# OpenClaw 源代码说明

- **分析提交基线：** 2026.9.5（`package.json`）
- **统计方法：** 遍历仓库，排除 `node_modules`、`dist`、`.git`；文本行按换行计数
- **配套：** [1DESIGN.md](1DESIGN.md)、[2HELP.md](2HELP.md)、[3SOLUTION.md](3SOLUTION.md)

---

## 1. 源代码整体介绍

### 1.1 产品与仓库

OpenClaw 是 TypeScript 为主的 pnpm monorepo：CLI + Gateway 守护进程 + Control UI + 捆绑插件 + 原生伴侣应用 + Rust 节点宿主。入口二进制 `openclaw.mjs`；发布包名 `openclaw`。

### 1.2 语言

| 语言 | 文件数 | 行数 | 主要用途 |
|------|--------|------|----------|
| TypeScript | 38,351 | 11,253,239 | 核心、插件、UI、测试 |
| Swift | 1,325 | 439,526 | macOS / iOS / 共享 Apple 层 |
| Kotlin | 567 | 250,604 | Android |
| JavaScript | 567 | 148,923 | 启动包装、少量运行时 `.mjs` |
| Markdown | 1,916 | 426,721 | 文档、技能、AGENTS |
| YAML | 775 | 146,219 | CI、配置 |
| JSON | 791 | 169,415 | schema、清单、locale |
| CSS | 151 | 81,646 | Control UI |
| Shell | 260 | 63,947 | 安装、CI、脚本 |
| Rust | 54 | 47,611 | `crates/`、Linux 伴侣部分 |
| Python | 51 | 23,564 | 辅助脚本 |
| Go | 30 | 10,983 | 少量工具 |
| XML | 124 | 51,314 | Android/Apple 资源 |
| SQL | 3 | 3,679 | 权威 schema（另有大量 TS 内嵌 SQL） |
| 其他 | — | — | PowerShell、C/ObjC、HTML、TOML、Plist |

**合计（含 Markdown/JSON/YAML）：** 45,023 文件，13,124,181 行。  
**代码合计（排除 Markdown/JSON/YAML/TOML/Plist）：** 41,374 文件，12,328,331 行。  
**测试文件（`*.test.*` 等）：** 16,924 个。

### 1.3 开发工具与运行时

| 类别 | 选择 |
|------|------|
| 包管理 | pnpm workspace（`nodeLinker: isolated`） |
| 语言运行时 | Node `>=24.16.0 <25 \|\| >=26.1.0`（推荐 26）；Bun ≥1.4.0 可选 |
| 模块 | TypeScript ESM，`module: NodeNext`，`strict`，`verbatimModuleSyntax` |
| 构建 | tsdown（核心）、Vite（`ui/`）、tsgo 类型检查 |
| 格式 / Lint | oxfmt、oxlint、stylelint（UI） |
| 测试 | Vitest 5（threads，`isolate: false`）、Playwright（UI E2E） |
| 数据库 | SQLite + Kysely |
| 原生 | Xcode/Swift、Gradle/Kotlin、Cargo/Rust |
| CI | GitHub Actions + Blacksmith；见 `.github/workflows` |
| 编辑器 | `.vscode/`、Cursor/Claude 技能在 `.agents/skills`、`.claude/skills` |

根脚本入口：`package.json` 的 `scripts`（`pnpm openclaw`、`pnpm test`、`pnpm build`、`pnpm check:changed` 等）。不要用 `node --import tsx src/index.ts`。

### 1.4 顶层目录行数

| 目录 | 文件 | 行 | 说明 |
|------|------|-----|------|
| `src/` | 20,938 | 6,238,629 | 核心运行时与测试 |
| `extensions/` | 10,686 | 2,796,474 | 捆绑插件 |
| `ui/` | 4,183 | 1,238,641 | Control UI |
| `apps/` | 2,217 | 874,270 | 原生应用 |
| `test/` | 1,679 | 653,722 | 跨切面测试与 helpers |
| `scripts/` | 1,443 | 438,897 | 构建、CI、docs、发布 |
| `docs/` | 1,355 | 316,544 | 产品文档 |
| `packages/` | 1,212 | 235,388 | 可复用包 |
| `.github/` | 200 | 74,792 | Actions |
| `CHANGELOG/` | 138 | 86,113 | 发布说明镜像 |
| `qa/` | 579 | 47,218 | QA 场景与成熟度 |
| `.agents/` | 223 | 43,992 | Agent 技能 |
| `crates/` | 22 | 15,678 | Rust |
| `skills/` | 73 | 7,844 | 捆绑技能 |
| 根文件 | 38 | 46,107 | `openclaw.mjs`、`node-*.mjs` 等 |

---

## 2. 根目录源文件

| 文件 | 目录 | 主要功能 |
|------|------|----------|
| `openclaw.mjs` | `/` | CLI bin：运行时守卫后转交 `dist/entry.js` |
| `node-version.mjs` | `/` | Node 版本探测与支持矩阵 |
| `node-sqlite.mjs` | `/` | SQLite 能力与 WAL 安全版本 |
| `node-runtime-update.mjs` | `/` | 运行时自更新辅助 |
| `node-runtime-recovery.mjs` | `/` | 运行时恢复 |
| `node-host-launcher.mjs` | `/` | 节点宿主启动 |
| `package.json` | `/` | 包清单、scripts、schemaVersions |
| `pnpm-workspace.yaml` | `/` | workspace 成员 |
| `pnpm-lock.yaml` | `/` | 锁 |
| `tsconfig.json` | `/` | TS 工程与 path 映射 |
| `tsdown.config.ts` | `/` | 核心打包 |
| `vitest.config.ts` | `/` | 测试根配置 |
| `Dockerfile` / `docker-compose.yml` | `/` | 容器 |
| `VISION.md` / `AGENTS.md` / `CONTRIBUTING.md` / `SECURITY.md` | `/` | 产品与贡献 |
| `taxonomy.yaml` | `/` | 成熟度分类 |

对应 `.d.mts` 为 JS 运行时模块的类型声明。

---

## 3. `src/` 模块清单（文件数 / 行数）

统计含该子树测试。功能简介针对生产代码。

| 目录 | 文件 | 行 | 主要功能 |
|------|------|-----|----------|
| `src/agents/` | 4,047 | 1,351,031 | Agent 组装、embedded runner、工具、权限、压缩 |
| `src/gateway/` | 3,341 | 1,190,554 | WS/HTTP Gateway、RPC、会话投影、worker 环境 |
| `src/infra/` | 2,034 | 569,645 | 运行时守卫、锁、更新账本、出站、迁移 |
| `src/plugins/` | 1,306 | 376,600 | 插件发现/加载/注册表 |
| `src/commands/` | 1,265 | 423,289 | CLI 命令实现（status/doctor/agent/…） |
| `src/cli/` | 1,075 | 324,497 | Commander 注册、lazy CLI、completion |
| `src/config/` | 1,059 | 268,187 | 配置 schema、校验、热重载、uiHints |
| `src/auto-reply/` | 902 | 311,824 | 通道自动回复、队列、调度 |
| `src/plugin-sdk/` | 718 | 110,223 | 对外 SDK 子路径 |
| `src/channels/` | 512 | 100,731 | 核心通道契约与适配（非插件作者 import） |
| `src/cron/` | 473 | 131,984 | 定时任务调度 |
| `src/state/` | 366 | 96,664 | SQLite schema SQL/TS、打开与校验 |
| `src/skills/` | 268 | 68,712 | 技能发现、workshop、注入 |
| `src/shared/` | 225 | 22,720 | 跨模块纯辅助 |
| `src/daemon/` | 207 | 59,372 | launchd/systemd/schtasks |
| `src/secrets/` | 201 | 52,108 | 密钥存储与扫描 |
| `src/tasks/` | 193 | 53,027 | 后台任务 / TaskFlow |
| `src/process/` | 175 | 38,418 | 子进程、取消、stdin |
| `src/node-host/` | 164 | 51,170 | 无头节点宿主客户端 |
| `src/acp/` | 147 | 38,808 | ACP 桥 |
| `src/tui/` | 142 | 55,523 | 终端 UI |
| `src/logging/` | 129 | 37,879 | 日志与诊断 |
| `src/system-agent/` | 128 | 41,602 | 系统助手 |
| `src/claws/` | 124 | 36,808 | 实验性 Claws |
| `src/media/` | 103 | 23,903 | 入站媒体 |
| `src/flows/` | 98 | 34,034 | 引导/doctor 流程 |
| `src/media-understanding/` | 94 | 27,810 | 多模态理解 |
| `src/sessions/` | 94 | 21,916 | 会话键、覆盖、分类 |
| `src/security/` | 92 | 20,953 | 安全审计 |
| `src/hooks/` | 88 | 18,266 | 内部 HOOK.md |
| `src/talk/` | 85 | 20,318 | 语音 Talk |
| `src/meeting-bot/` | 84 | 21,233 | 会议机器人 |
| `src/worker/` | 79 | 18,405 | 云 worker 运行时 |
| `src/wizard/` | 65 | 28,349 | 交互向导 |
| `src/tts/` | 58 | 13,203 | TTS 注册 |
| `src/transcripts/` | 53 | 11,101 | 会议转录存储 |
| `src/audit/` | 52 | 15,013 | 审计账本读写 |
| `src/utils/` | 52 | 6,700 | 通用工具 |
| `src/llm/` | 50 | 8,043 | 提供商载荷兼容 |
| `src/` 根 | 47 | 8,904 | `entry.ts`、`index.ts`、`runtime.ts` |
| `src/plugin-state/` | 38 | 9,897 | 插件 KV/blob SQLite |
| `src/status/` | 32 | 7,823 | status 投影 |
| `src/context-engine/` | 27 | 7,135 | 上下文引擎 |
| `src/mcp/` | 24 | 5,853 | MCP 服务与桥 |
| `src/proxy-capture/` | 24 | 8,030 | 调试代理抓包 |
| `src/canvas/` | 21 | 4,335 | 托管 widget |
| `src/fleet/` | 21 | 8,699 | 实验租户 cell |
| `src/boards/` | 21 | 5,115 | 会话看板 |
| `src/snapshot/` | 20 | 7,485 | git/快照备份 |
| `src/routing/` | 20 | 5,348 | 通道账户路由 |
| `src/pairing/` | 18 | 3,558 | 设备/通道配对存储 |
| `src/trajectory/` | 17 | 8,106 | 运行轨迹 |
| `src/model-catalog/` | 17 | 4,097 | 模型目录 |
| `src/image-generation/` | 15 | 3,228 | 图像生成运行时 |
| `src/video-generation/` | 15 | 5,097 | 视频生成 |
| `src/memory-host-sdk/` | 13 | 1,785 | 记忆宿主 SDK 面 |
| `src/projects/` | 13 | 3,237 | 项目克隆登记 |
| 其余小组件 | — | — | `web-search`、`web-fetch`、`music-generation`、`link-understanding`、`chat`、`types`、`bootstrap`、`memory`、`compat` 等 |

### 3.1 `src/` 根生产文件

| 文件名 | 所在目录 | 主要功能 |
|--------|----------|----------|
| `entry.ts` | `src/` | 进程入口、主循环转交 |
| `entry.respawn.ts` | `src/` | 崩溃 respawn |
| `entry.compile-cache.ts` | `src/` | 编译缓存 |
| `entry.esm-resolve-fast-path.ts` | `src/` | ESM 解析快路径 |
| `entry.version-fast-path.ts` | `src/` | `--version` 快路径 |
| `index.ts` | `src/` | 库入口 |
| `library.ts` | `src/` | 可嵌入库表面 |
| `runtime.ts` | `src/` | RuntimeEnv |
| `logger.ts` / `logging.ts` | `src/` | 日志引导 |
| `globals.ts` / `global-state.ts` | `src/` | 进程全局 |
| `version.ts` | `src/` | 版本字符串 |
| `utils.ts` | `src/` | 入口级工具 |
| `polls.ts` / `poll-params.ts` | `src/` | 投票参数 |
| `param-key.ts` | `src/` | 参数键 |
| `docker-healthcheck.ts` | `src/` | 镜像健康检查 |
| `browser-lifecycle-cleanup.ts` | `src/` | 浏览器生命周期清理 |

### 3.2 Agent 工具生产文件（`src/agents/tools/*-tool.ts`）

| 文件名 | 目录 | 主要功能 |
|--------|------|----------|
| `sessions-tool.ts` | `src/agents/tools/` | 会话工具 |
| `sessions-list-tool.ts` | 同上 | 列表 |
| `sessions-history-tool.ts` | 同上 | 历史 |
| `sessions-search-tool.ts` | 同上 | 搜索 |
| `sessions-spawn-tool.ts` | 同上 | 派生 |
| `sessions-send-tool.ts` | 同上 | 发送 |
| `sessions-yield-tool.ts` | 同上 | 让出 |
| `session-status-tool.ts` | 同上 | 状态 |
| `message-tool.ts` 及 `message-tool-*.ts` | 同上 | 通道消息动作 |
| `cron-tool.ts` | 同上 | Cron |
| `nodes-tool.ts` | 同上 | 节点命令 |
| `computer-tool.ts` | 同上 | 电脑使用 |
| `screen-tool.ts` | 同上 | 屏幕 |
| `pdf-tool.ts` | 同上 | PDF |
| `image-tool.ts` / `image-generate-tool.ts` | 同上 | 图像 |
| `video-generate-tool.ts` / `music-generate-tool.ts` | 同上 | 视频/音乐 |
| `tts-tool.ts` | 同上 | TTS |
| `plugins-tool.ts` | 同上 | 插件 |
| `secrets-tool.ts` | 同上 | 密钥 |
| `gateway-tool.ts` | 同上 | 网关 |
| `dashboard-tool.ts` | 同上 | 控制面 |
| `portal-tool.ts` | 同上 | Portal |
| `progress-card-tool.ts` | 同上 | 进度卡 |
| `ask-user-tool.ts` | 同上 | 提问 |
| `subagents-tool.ts` / `agents-list-tool.ts` / `agents-wait-tool.ts` | 同上 | 子代理 |
| `openclaw-delegate-tool.ts` | 同上 | 委派 |
| `system-agent-tool.ts` | 同上 | 系统助手 |
| `transcripts-tool.ts` | 同上 | 转录 |
| `theme-tool.ts` | 同上 | 主题 |
| `github-publish-tool.ts` / `github-identity-status-tool.ts` | 同上 | GitHub |
| `skill-workshop-tool.ts` | 同上 | 技能工坊 |
| `mobile-ui-tool.ts` | 同上 | 移动 UI |
| `heartbeat-response-tool.ts` | 同上 | 心跳 |
| `structured-output-tool.ts` | 同上 | 结构化输出 |
| `terminal-tool.ts` | 同上 | 终端/exec |
| `goal-tools.ts` / `conversation-tools.ts` | 同上 | 目标与对话 |

`src/agents/embedded-agent-runner/` 核心生产文件包括：`run.ts`、`runs.ts`、`compact` 相关、`system-prompt.ts`、`tool-result-truncation.ts`、`transcript-rewrite.ts`、`stream-resolution.ts`、`thinking.ts`、`sandbox-skills.ts` 等（该目录约 291 个文件，含测试）。

---

## 4. `packages/` 清单

可复用 TypeScript 包（生产+测试文件计数）：

| 包目录 | 代码文件约数 | 主要功能 |
|--------|----------------|----------|
| `packages/ai` | 326 | 模型传输、流、校验 |
| `packages/gateway-protocol` | 250 | TypeBox 协议 schema |
| `packages/memory-host-sdk` | 128 | 记忆宿主契约 |
| `packages/gateway-client` | 61 | WS 客户端 |
| `packages/plugin-sdk` | 60 | 发布用 SDK 构建 |
| `packages/markdown-core` | 56 | Markdown |
| `packages/normalization-core` | 55 | 规范化/expect |
| `packages/agent-core` | 50 | Agent 核心原语 |
| `packages/terminal-core` | 32 | 终端抽象 |
| `packages/acp-core` | 22 | ACP |
| `packages/sdk` | 20 | 嵌入 SDK 聚合 |
| `packages/media-core` | 19 | 媒体 |
| `packages/model-catalog-core` | 19 | 模型目录 |
| `packages/llm-core` | 13 | LLM 原语 |
| `packages/media-understanding-common` | 13 | 理解共用 |
| `packages/tool-call-repair` | 13 | 工具调用修复 |
| `packages/session-url-contract` | 12 | 会话 URL |
| `packages/net-policy` | 11 | 网络策略 |
| `packages/media-generation-core` | 7 | 生成共用 |
| `packages/mermaid-renderer` | 5 | Mermaid |
| `packages/plugin-package-contract` | 3 | 插件包合同 |
| `packages/retry` | 2 | 重试 |
| `packages/workboard-contract` | 2 | Workboard 契约 |

---

## 5. `extensions/` 捆绑插件清单

约 **168** 个插件包（含测试夹具与文档）。每个插件目录通常含 `package.json`、`openclaw.plugin.json`、`src/`、`index.ts`。**禁止** import `src/**` 核心内部。

### 5.1 通道类

| 目录 | 主要功能 |
|------|----------|
| `telegram` | Telegram Bot |
| `whatsapp` | WhatsApp / Baileys |
| `discord` | Discord |
| `slack` | Slack |
| `signal` | Signal |
| `imessage` | iMessage（Apple） |
| `matrix` | Matrix |
| `msteams` | Microsoft Teams |
| `googlechat` | Google Chat |
| `feishu` | 飞书 |
| `line` | LINE |
| `irc` | IRC |
| `mattermost` | Mattermost |
| `nextcloud-talk` | Nextcloud Talk |
| `nostr` | Nostr |
| `sms` | SMS |
| `synology-chat` | Synology Chat |
| `tlon` | Tlon |
| `twitch` | Twitch |
| `voice-call` | 语音通话通道 |
| `zalo` / `zalouser` | Zalo |
| `a2a` | Agent-to-agent |
| `reef` | Reef 通道 |
| `imap` | 邮件 |
| `qa-channel` | QA 通道 |

### 5.2 模型提供商类

`anthropic`、`anthropic-vertex`、`openai`、`google`、`amazon-bedrock`、`amazon-bedrock-mantle`、`azure-speech`（语音）、`groq`、`mistral`、`cohere`、`deepseek`、`xai`、`together`、`fireworks`、`deepinfra`、`cerebras`、`openrouter`、`ollama`、`vllm`、`sglang`、`lmstudio`、`llama-cpp`、`litellm`、`huggingface`、`github-copilot`、`copilot`、`copilot-proxy`、`minimax`、`moonshot`、`kimi-coding`、`qwen`、`qianfan`、`alibaba`、`byteplus`、`volcengine`、`tencent`、`stepfun`、`zai`、`venice`、`synthetic`、`novita`、`nvidia`、`meta`、`microsoft`、`microsoft-foundry`、`cloudflare-ai-gateway`、`vercel-ai-gateway`、`baseten`、`chutes`、`featherless`、`arcee`、`longcat`、`kilocode`、`opencode`、`opencode-go` 等。

### 5.3 记忆 / 浏览器 / 媒体 / 基础设施

| 目录 | 主要功能 |
|------|----------|
| `memory-core` | 默认记忆引擎 |
| `memory-lancedb` | LanceDB 向量 |
| `memory-wiki` | Wiki 记忆 |
| `active-memory` | 活动记忆 |
| `browser` | CDP 浏览器 + CLI |
| `crabbox` | 沙箱/远程执行适配 |
| `cua-computer` | 电脑使用 |
| `image-generation-core` | 图像生成核心 |
| `comfy` / `fal` / `runway` / `pixverse` | 生成后端 |
| `elevenlabs` / `fish-audio-speech` / `azure-speech` / `tts-local-cli` / `senseaudio` / `inworld` / `apple-fm` | 语音 |
| `deepgram` | 转写 |
| `document-extract` / `web-readability` | 文档/网页抽取 |
| `brave` / `duckduckgo` / `exa` / `firecrawl` / `tavily` / `searxng` / `perplexity` | 搜索 |
| `acpx` | ACPX 运行时 |
| `codex` | Codex harness 插件 |
| `github` | GitHub 集成 |
| `onepassword` / `vault` | 密钥后端 |
| `diagnostics-otel` / `diagnostics-prometheus` | 诊断导出 |
| `webhooks` | Webhook |
| `bonjour` / `device-pair` | 发现与配对 |
| `linux-node` | Linux 节点 |
| `canvas` / `workboard` | 画布与看板 |
| `lobster` | Lobster 工作流 |
| `diffs` / `diffs-language-pack` | Diff 技能 |
| `migrate-claude` / `migrate-hermes` | 迁移 |
| `session-share` / `visitor-access` / `team-reports` | 协作 |
| `qa-lab` | QA Lab 插件 |
| `oc-path` | 路径 CLI |
| `policy` / `typesafe` / `tokenjuice` / `clawrouter` | 策略与路由辅助 |

每个插件的源文件位于 `extensions/<id>/src/`，典型文件：`channel.ts`、`gateway.ts`、`index.ts`、`config-schema.ts`、`doctor.ts`、`runtime.ts`、`openclaw.plugin.json`。

---

## 6. Control UI（`ui/`）

| 路径 | 约文件数 | 主要功能 |
|------|----------|----------|
| `ui/src/pages/` | 1,640 | 路由页面（chat、settings、sessions…） |
| `ui/src/e2e/` | 648 | Playwright E2E |
| `ui/src/components/` | 645 | Lit 组件 |
| `ui/src/lib/` | 485 | Gateway 客户端缓存、会话列表 |
| `ui/src/app/` | 277 | 应用壳 |
| `ui/src/test-helpers/` | 141 | 测试 |
| `ui/src/styles/` | 116 | CSS token |
| `ui/src/i18n/` | 77 | 英文源 + locale 适配 |
| `ui/src/plugins/` | 23 | UI 插件点 |
| `ui/src/main.ts` | 1 | 启动 |
| `ui/src/app-routes.ts` | 1 | 路由表 |

技术：Lit、Vite、自定义属性 token、与 Gateway **同版本**分发。

---

## 7. 原生应用与 crate

| 路径 | 代码文件 | 语言 | 主要功能 |
|------|----------|------|----------|
| `apps/macos/` | 626 | Swift | 菜单栏伴侣、Canvas、节点能力 |
| `apps/android/` | 572 | Kotlin | Android 节点 |
| `apps/shared/` | 382 | Swift | Apple 共享协议模型 |
| `apps/ios/` | 288 | Swift | iOS 节点 |
| `apps/linux/` | 59 | Rust/JS | Linux 伴侣 |
| `apps/swabble/` | 27 | Swift | 辅助 app |
| `apps/macos-mlx-tts/` | 4 | Swift | MLX 本地 TTS |
| `crates/openclaw-node-host/` | 15 | Rust | 节点宿主 |
| `crates/openclaw-gateway-client/` | 3 | Rust | Gateway 客户端 |

生成的 Swift 模型来自 Gateway JSON Schema（见协议文档）。

---

## 8. 脚本、测试、技能、文档

| 路径 | 作用 |
|------|------|
| `scripts/` | 构建、docs 链接、plugin inventory、vitest 包装、发布、i18n |
| `test/` | 跨包集成、vitest 分片配置、helpers（`test/helpers/AGENTS.md`） |
| `skills/` | 捆绑技能（约 52 个技能目录） |
| `docs/` | 用户与参考文档（Mintlify 发布） |
| `qa/` | QA 场景、成熟度分数 |
| `config/` | oxlint/stylelint 等 |
| `deploy/` | 部署清单 |
| `examples/` | 示例 |
| `security/` | 安全辅助 |
| `.github/workflows/` | CI |
| `.agents/skills/` | 维护者/代理技能（autoreview 等） |

---

## 9. 权威数据文件

| 文件 | 目录 | 功能 |
|------|------|------|
| `openclaw-state-schema.sql` | `src/state/` | 全局库 130 表 |
| `openclaw-agent-schema.sql` | `src/state/` | Agent 库 42 表 |
| `core-command-descriptors.ts` | `src/cli/program/` | 核心 CLI 名 |
| `subcli-descriptors.ts` | `src/cli/program/` | 子 CLI 名 |
| `server-methods-list.ts` | `src/gateway/` | 广告 RPC 与事件 |
| `plugin-sdk-entrypoints.json` | `scripts/lib/` | SDK 子路径清单 |

---

## 10. 测试源码位置

测试与生产代码相邻（`*.test.ts`），另有：

- `src/**/*.e2e.test.ts`
- `ui/src/**/*.e2e.test.ts`
- `test/vitest/*.config.ts` 分片
- `extensions/*/src/**/*.test.ts`

约 16,924 个测试文件；套件含义见 `1DESIGN.md` 第 7 节与 `docs/help/testing/suites.md`。

---

## 11. 阅读顺序建议

1. `VISION.md`、根 `AGENTS.md`
2. `src/entry.ts` → `src/cli/run-main.ts` → `src/gateway/` 启动
3. `src/agents/embedded-agent-runner/run.ts` 回合循环
4. `src/state/*.sql` 数据
5. `src/plugin-sdk/` + `extensions/AGENTS.md` 扩展边界
6. `ui/src/main.ts` 与 `ui/AGENTS.md`

---

## 12. 统计局限

- 未计入 `node_modules`、构建产物、Git 对象。
- SQL 行数偏低：多数 DDL 内嵌于 TypeScript。
- 单目录下还有大量 `*.test.ts`；上表 `src/<module>` 文件数包含测试。
- 完整 45,023 个路径的逐文件 dump 体积过大，不适合放进单一 PR 文本；本清单覆盖**全部顶层与全部 `src/` 子系统、全部 package、全部捆绑插件、全部原生 app**，并列出核心入口与全部 `*-tool.ts`。需要机器可读全量列表时，可在干净 checkout 对上述扩展名再跑一遍 `find`。
