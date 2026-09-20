# OpenClaw 设计说明书

- **产品：** OpenClaw（CLI 包名 `openclaw`）
- **分析版本：** `2026.9.5`（仓库 `main`，schema：state=17，agent=21）
- **许可证：** MIT
- **治理：** OpenClaw Foundation（独立 501(c)(3)）
- **分析范围：** 本仓库全部源码树（`src/`、`extensions/`、`packages/`、`ui/`、`apps/`、`crates/`、`scripts/`、`test/`、`skills/`）
- **配套文档：** [2HELP.md](2HELP.md) 使用说明、[3SOLUTION.md](3SOLUTION.md) 后续优化方案、[4SOURCE.md](4SOURCE.md) 源代码清单

本说明书按源码与权威 schema 编写。仓库约 **45,023** 个源码类文件、**13,124,181** 行；其中代码（不含 Markdown/JSON/YAML）约 **41,374** 文件、**12,328,331** 行。公开导出函数以万计，下文按**责任所有者（owner）**穷尽模块、数据模型、公开函数族、算法与测试套件；逐文件清单见 `4SOURCE.md`。

---

## 1. 设计目标与约束

### 1.1 产品目标

OpenClaw 是运行在操作者自有设备上的多通道 AI 助手：

1. **可信网关、不可信执行、确定性策略。** Gateway 拥有会话、工具、事件与通道连接；模型与 harness 是可替换插件。
2. **状态、记忆、凭据留在本机。** 默认不向基金会上报使用分析；每日版本检查可关闭。
3. **小核心、强插件。** 核心承担通用契约；通道、提供商、记忆、媒体、技能走插件。
4. **一个责任一个所有者。** 调用方消费 owner 的操作与已记录事实；适配器翻译契约；缓存必须有失效生命周期。
5. **更新必须尽量完成。** `openclaw update` 对每种安装 best-effort；Doctor 迁移旧配置；拒绝仅发生在有具体数据风险时。

### 1.2 非目标（当前）

- 新核心技能默认进仓库（应先发 ClawHub）
- 全量人工维护的多语言文档集
- 把云沙箱提供商做成 OpenClaw 插件（归 Crabbox）
- 对已支持通道再包一层无能力增量的 wrapper
- 与现有 MCP / ACPX / 插件 / ClawHub 路径重复的 MCP 工作

### 1.3 技术选型理由

编排系统（提示、工具、协议、集成）选择 TypeScript：可黑客、迭代快、类型严格（ESM、`verbatimModuleSyntax`、禁止 `@ts-nocheck`）。持久化选择 SQLite + Kysely；原生伴侣用 Swift / Kotlin；节点宿主用 Rust。

---

## 2. 整体架构

### 2.1 逻辑分层

```
┌──────────── 人机表面 ────────────┐
│ CLI / TUI / Control UI / 原生 App │
│ macOS · iOS · Android · Linux Hub │
└──────────────┬───────────────────┘
               │ WebSocket JSON-RPC + HTTP
┌──────────────▼───────────────────┐
│           Gateway 守护进程         │
│  握手 · 鉴权 · 配对 · 会话投影     │
│  队列 · 审批 · Cron · 插件注册表   │
└──────────────┬───────────────────┘
        ┌──────┼──────┬──────────┐
        ▼      ▼      ▼          ▼
   Agent Loop  Channels  Nodes   Workers
   (embedded /  Telegram  camera  cloud
    Codex/ACP)  Slack…    screen  sandbox
        │
        ▼
   Tools · Skills · Memory · MCP · Browser
        │
        ▼
   SQLite state DB + per-agent DB + workspace files
```

### 2.2 进程与角色

| 角色 | 入口 | 职责 |
|------|------|------|
| Gateway | `openclaw gateway` / daemon | 每主机一台；打开通道会话；暴露 WS API；投影会话行；调度 cron/heartbeat |
| Operator client | CLI、Control UI、macOS 菜单栏 | `role` 为操作者；订阅 `agent`/`chat`/`sessions.changed` |
| Node | iOS/Android/macOS/headless | `role: node`；设备配对；`camera.*` / `screen.record` / `location.get` |
| Worker | `openclaw worker` | 受限云执行；placement claim + 方言升级边界 |
| Embedded agent | Gateway 内或 `openclaw agent --local` | 内置 OpenClaw harness |
| CLI backend | Claude CLI 等 | 子进程执行，模型 ref 仍规范为 `provider/model` |
| ACP / ACPX | `openclaw acp` | 外部 agent 控制面桥 |

### 2.3 核心所有权切分

| 目录 | 所有者 | 不该做的事 |
|------|--------|------------|
| `src/gateway/` | 控制面、WS RPC、会话投影、投递 | 为静态描述去物化插件运行时 |
| `src/agents/` | Agent 组装、run authority、工具、embedded runner | 热路径反复加载通道插件 |
| `src/channels/` | 核心通道契约与适配 | 插件作者直接 import |
| `src/plugins/` | 发现、manifest、加载、注册表 | 把插件配置散读成核心所有 |
| `src/plugin-sdk/` | 对外 SDK 契约 | 把内部实现当公共 API |
| `extensions/` | 捆绑插件（与第三方同一边界） | import `src/**` 内部 |
| `src/state/` | SQLite schema 与迁移 | 新 JSON/JSONL 状态仓 |
| `src/infra/` | 运行时守卫、更新账本、锁、出站 | 在事务回调里 `await` |
| `ui/` | Control UI（Lit） | 在渲染层重做 Gateway 决策 |
| `apps/` | 原生伴侣 | 自管 Gateway 协议分叉 |

### 2.4 一次 Agent 回合（主路径）

1. 通道 ingress 或 RPC `agent` 入站。
2. 配对 / allowlist / mention 门控。
3. 解析 `sessionKey` / `sessionId`，写入会话元数据，立即返回 `{ runId, acceptedAt }`。
4. 消息队列模式（steer / followup / collect / interrupt）进入 session lane，再进全局 `main` lane。
5. 录取（admission）一次：解析模型、auth profile、工具面、sandbox、skills snapshot。重试复用同一权威，不另造权威。
6. `runEmbeddedAgent`：workspace、bootstrap、系统提示、writer claim。
7. 模型推理 + 工具循环；流式 `assistant` / `tool` / `lifecycle`。
8. 转录写入校验 `expectedWriterRunId`；被取代的 run 不能提交陈旧转录。
9. 回复整形（过滤 `NO_REPLY`、去重消息工具、必要回复的 tool-free finalization）。
10. 生命周期 `end|error`；`agent.wait` 等待该 `runId`。可选记忆 flush / 压缩在投递结算后后台进行。

### 2.5 Gateway 连接生命周期

1. 首帧必须是 `connect`（非 JSON / 非 connect 硬关闭）。
2. 设备身份 + `connect.challenge` 签名（v3 绑定 `platform` 与 `deviceFamily`）。
3. 鉴权：token/password、Tailscale、trusted-proxy、或私有 ingress 的 `none`。
4. 新设备需配对；loopback 可自动批准。
5. `hello-ok` 携带 presence、health、`features.methods`（发现列表，不是全量 dump）。
6. 之后：`{type:req,id,method,params}` → `{type:res,id,ok,payload|error}`；事件 `{type:event,event,payload,seq?}`。
7. 副作用方法需要幂等键；事件不重放，缺口时客户端刷新。

### 2.6 包与工作区

pnpm workspace：根包 `openclaw`、`ui`、`packages/*`、`extensions/*`、`examples/*`。构建：`tsdown`（核心）、Vite（Control UI）、Xcode/Gradle（原生）、Cargo（节点宿主）。格式化 `oxfmt`，静态分析 `oxlint`，类型检查 `tsgo`，测试 Vitest。

---

## 3. 数据模型

持久化只有两类 SQLite：**全局控制面** `~/.openclaw/state/openclaw.sqlite`（state schema 17）与 **每 agent 数据面** `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`（agent schema 21）。写事务同步：先完成异步规划，再重读权威行后写入；事务回调禁止 `Promise`/`await`。表均为 `STRICT`。

### 3.1 全局状态库（130 张表）

权威 SQL：`src/state/openclaw-state-schema.sql`。

#### 3.1.1 Schema 与租约

| 表 | 主键/关键列 | 用途 |
|----|-------------|------|
| `schema_meta` | `meta_key` | 角色、schema_version、app_version |
| `state_leases` | 租约身份 | 防多 Gateway 同目录并发写 |
| `agent_databases` | agent 路径登记 | 控制面知道每 agent DB 位置 |
| `agent_deletion_journal` | 删除流水 | 删除 agent 的可恢复账本 |
| `agent_provenance` | agent 来源 | 创建/导入出处 |
| `agent_database_leases` | agent DB 租约 | 数据面打开权 |

#### 3.1.2 配置与健康

| 表 | 用途 |
|----|------|
| `config_machine_state` | 机器级配置状态（含 mentions inbox 等无新表特性） |
| `config_revision_keys` | 配置修订身份 |
| `config_health_entries` | Doctor/健康条目 |
| `workspace_setup_state` | 工作区引导状态 |
| `workspace_path_aliases` | 工作区路径别名 |
| `workspace_generated_bootstrap_hashes` | 生成 bootstrap 文件哈希 |

#### 3.1.3 设备、节点、配对

| 表 | 用途 |
|----|------|
| `device_identities` | 设备身份 |
| `device_auth_tokens` | 设备令牌 |
| `device_pairing_pending` / `device_pairing_paired` | 设备配对队列与已配对 |
| `device_bootstrap_tokens` | 引导令牌 |
| `device_pair_setup_completions` | 配对 setup 完成 |
| `device_pairing_join_codes` | 加入码 |
| `gateway_origin_device_tokens` | 源 Gateway 设备令牌 |
| `channel_pairing_requests` / `channel_pairing_allow_entries` | 通道 DM 配对与允许名单 |
| `node_worker_launches` / `_containers` / `_cleanup` / `_turns` | 节点 worker 启动与回合 |
| `node_worker_prepared_workspaces` | 预热工作区 |
| `macos_port_guardian_records` | macOS 端口守护 |

#### 3.1.4 审批与执行身份

| 表 | 用途 |
|----|------|
| `exec_approvals_config` | exec 审批策略 |
| `operator_approvals` | 操作者审批 |
| `operator_approval_execution_identities` | 执行身份伴生表（仅 provenance，不授权） |
| `operator_approval_standing_grants` | 常设授权 |
| `execution_identity_contexts` | 执行上下文 |
| `execution_decision_facts` | 决策事实 |
| `execution_owner_lifecycle_bindings` | owner 生命周期绑定 |
| `plugin_binding_approvals` | 插件绑定审批 |

#### 3.1.5 审计与出站

| 表 | 用途 |
|----|------|
| `audit_events` | 元数据审计账本（无 prompt/工具参数） |
| `audit_identity_keys` | 审计身份键 |
| `outbound_message_execution_bindings` | 出站消息绑定执行身份 |
| `outbound_message_progress` | queued / platform-started 进度 |
| `outbound_media_provenance` | 出站媒体出处 |
| `delivery_queue_entries` | 投递队列 |
| `channel_ingress_events` | 入站事件去重/回放 |

#### 3.1.6 会话控制面投影辅助

| 表 | 用途 |
|----|------|
| `session_state_events` / `session_state_heads` / `session_watch_cursors` | 会话状态流 |
| `session_upstream_links` | 上游链接 |
| `session_groups` | 会话分组 |
| `current_conversation_bindings` | 当前会话绑定 |
| `user_preferences` | 用户偏好 |

#### 3.1.7 技能、插件、MCP

| 表 | 用途 |
|----|------|
| `skill_usage` | 技能使用计数 |
| `skill_library_entries` / `_revisions` / `_events` / `_uploads` | 档案技能库 |
| `skill_workshop_proposals` / `_collection_reviews` / `_proposal_rollbacks` / `_proposal_events` | Skill workshop |
| `skill_uploads` / `skill_upload_chunks` | 上传分块 |
| `plugin_state_entries` | 插件键值状态（含 TTL listing index） |
| `plugin_blob_entries` | 插件 blob |
| `official_external_plugin_catalog_snapshots` | 官方外部插件目录快照 |
| `clawhub_promotion_claims` | ClawHub 促销领取 |
| `mcp_oauth_stores` / `mcp_oauth_pending_authorizations` | MCP OAuth |

#### 3.1.8 Cron、任务、流

| 表 | 用途 |
|----|------|
| `cron_jobs` | 定时任务定义 |
| `cron_run_receipts` | 运行收据 |
| `cron_run_trigger_state_retirements` | 触发状态退役 |
| `cron_job_runtime_authorities` | 运行时权威 |
| `cron_job_scratch` | 暂存 |
| `task_runs` / `task_delivery_state` | 后台任务 |
| `subagent_runs` | 子代理运行 |
| `flow_runs` | TaskFlow / 工作流 |

#### 3.1.9 会议转录

| 表 | 用途 |
|----|------|
| `meeting_transcript_sessions` | 一次采集身份 |
| `meeting_transcript_utterances` | 按 sequence 追加的话语 |
| `meeting_transcript_summaries` | 每采集一条摘要 |
| `capture_sessions` / `capture_blobs` / `capture_events` | 通用采集 |

#### 3.1.10 更新、备份、迁移

| 表 | 用途 |
|----|------|
| `update_runs` | 更新账本（UUID `run_id`，JSON 列 16KiB 截断） |
| `gateway_restart_sentinel` / `_intent` / `_handoff` / `gateway_boot_lifecycle` | 重启交接 |
| `backup_runs` | 备份运行 |
| `migration_runs` / `migration_sources` | Doctor 文件→SQLite 收据 |
| `diagnostic_events` | 诊断事件 |

#### 3.1.11 Worker / 云 / GitHub / Fleet

| 表 | 用途 |
|----|------|
| `worker_environments` | worker 环境 |
| `worker_environment_ssh_fallback_ports` | SSH 回退端口 |
| `worker_environment_credentials` | 环境凭据 |
| `worker_session_placements` / `_placement_moves` | 放置与迁移 |
| `worker_turn_tool_authorities` / `worker_session_tool_operations` | 回合工具权威 |
| `worker_workspace_reconciliations` / `_pending_results` | 工作区对账 |
| `worker_transcript_commit_heads` / `_commits` | 转录提交 |
| `worker_inference_turns` | 推理回合 |
| `local_workspace_projections` | 本地工作区投影 |
| `session_repository_workspaces` | 会话仓库工作区 |
| `github_publication_requests` / `_personal_` / `_session_lifecycles` / `github_repository_publication_requests` | GitHub 发布 |
| `projects` / `worktrees` / `worktree_provisioned_file_chunks` / `worktree_templates` | 项目与 managed worktree |
| `fleet_cells` | 实验性租户 cell |
| `sandbox_registry_entries` | 沙箱登记 |

#### 3.1.12 ACP、推送、密钥、Claws

| 表 | 用途 |
|----|------|
| `acp_sessions` / `acp_replay_sessions` / `acp_replay_events` | ACP 会话与回放字节估计 |
| `web_push_subscriptions` / `web_push_approval_deliveries` | Web Push |
| `apns_registrations` / `apns_registration_tombstones` | APNs |
| `secret_store_entries` | 密钥存储 |
| `native_hook_relay_bridges` | 原生 hook 中继 |
| `managed_outgoing_image_records` | 托管出站图 |
| `claw_installs` / `claw_workspace_files` / `claw_package_refs` / `claw_cron_refs` / `claw_mcp_server_refs` | 实验性 Claws |

### 3.2 Agent 数据库（42 张表）

权威 SQL：`src/state/openclaw-agent-schema.sql`。教义：`session_nodes.entry_json` 是逻辑会话的权威记录；提升列只是索引投影。

#### 3.2.1 会话身份

| 表 | 关键列 | 用途 |
|----|--------|------|
| `schema_meta` | `meta_key` | agent schema 版本 |
| `session_nodes` | `session_key` PK；`entry_json`；`current_session_id`；status/owner/archive/pin | 逻辑会话 |
| `session_participants` | `(session_key, identity_namespace, actor_id)` | 参与者贡献计数 |
| `session_key_contract` | 单行 `id=1` | `main` 键契约 |
| `session_windows` | `session_id` PK | 转录世代（reset/rollover/fork/rewind/compaction） |
| `session_canonical_validation_pending` | `session_key` | 触发器维护的校验队列 |
| `session_members` | 会话成员 | 共享会话 |
| `session_suggestions` | 建议 | 任务/下一步建议 |
| `session_conversations` | 会话↔对话 | 绑定 |

触发器：插入/更新 `entry_json` 或身份列会把 `entry_valid` 置 0，并登记 canonical pending。窗口插入/更新/删除同样登记。

#### 3.2.2 对话与投递

| 表 | 用途 |
|----|------|
| `conversations` | 通道侧对话身份 |
| `conversation_deliveries` | 投递收据 |
| `message_tool_run_outcomes` | 消息工具运行结果 |
| `session_goal_operations` | 目标操作 |
| `session_progress_cards` | 进度卡 |
| `heartbeat_outcomes` | 心跳结果 |
| `session_pending_inputs` / `session_input_completions` | 挂起输入与完成 |

#### 3.2.3 转录 DAG

| 表 | 用途 |
|----|------|
| `transcript_events` | 转录事件（parentId 链/DAG，禁止无 parentId 的裸 JSONL 追加） |
| `transcript_event_identities` | 事件身份 |
| `session_transcript_archives` | 热归档 |
| `session_transcript_cold_archives` | 冷归档元数据；载荷在 `sessions/cold/<sha256>.jsonl.zst` |
| `transcript_rewrite_watermarks` | 重写水位 |
| `session_transcript_index_state` / `session_transcript_active_events` | 索引与活动事件 |
| `trajectory_runtime_events` | 运行轨迹 |
| `acp_parent_stream_events` | ACP 父流 |
| `context_engine_turn_outbox` | 上下文引擎 outbox |

#### 3.2.4 看板与缓存

| 表 | 用途 |
|----|------|
| `board_tabs` / `board_widgets` | 会话看板 |
| `cache_entries` | 通用缓存 |
| `auth_profile_store` / `auth_profile_state` | 每 agent 认证档案 |

#### 3.2.5 记忆索引

| 表 | 用途 |
|----|------|
| `memory_index_meta` | 索引元数据 |
| `memory_index_sources` | 源文件 |
| `memory_index_chunks` | 块 + 嵌入引用 |
| `memory_index_chunk_recall_metadata` | 召回元数据 |
| `memory_index_chunk_provenance` | 出处（owner/agent/untrusted/system） |
| `memory_entry_origins` | 条目来源会话 |
| `memory_session_tombstones` | forget 墓碑 |
| `memory_embedding_cache` | 嵌入缓存 |
| `memory_index_state` | 索引状态机 |
| `standing_intents` | 前瞻意图 |

### 3.3 工作区文件模型（非 SQL）

| 文件 | 层级 | 谁写 | 谁注入 |
|------|------|------|--------|
| `AGENTS.md` | 指令 | 仅人类 | 会话开始总是注入 |
| `SOUL.md` / `USER.md` / `MEMORY.md` | 策核心 | Dreaming 或用户请求 | 出处合格时按预算注入 |
| `memory/YYYY-MM-DD.md` | 情节 | Agent / flush | 不自动注入，可搜索 |
| 会话转录 | 情节 | Runner | 不注入全文 |
| `DREAMS.md` | 审阅 | Dreaming | 不注入 |
| `openclaw.json` | 配置 | 操作者 / Doctor | 运行时读取当前 schema |

Activity recap 存在 `session_nodes.entry_json.activitySummary`（可重建缓存，无 schema bump）。

### 3.4 配置数据模型（运行时）

根配置 `~/.openclaw/openclaw.json`（JSON5）。两桶：根兄弟是基础设施与跨 agent 默认；`agents.defaults` 是 agent 循环行为；`agents.entries.*` 可覆盖。未知键启动失败。`$schema` 是唯一根级例外。热重载由 Gateway 监视文件。插件配置在 `plugins.entries.<id>.config`，由插件 doctor 修复，核心不散读。

主要根键族：`agents`、`channels`、`gateway`、`session`、`messages`、`tools`、`skills`、`plugins`、`cron`、`hooks`、`secrets`、`sandbox`、`logging`、`update`、`models`、`ui`。

### 3.5 协议数据模型

TypeBox → JSON Schema → Swift 模型。帧：

- `connect` / `hello-ok`
- `req` / `res` / `event`
- 幂等键缓存
- Node：`role`、`caps`、`commands`、`permissions`

事件名见 `src/gateway/server-methods-list.ts` 的 `GATEWAY_EVENTS`（`agent`、`chat`、`sessions.changed`、`presence`、`cron`、`task`、审批与配对事件等）。

### 3.6 插件状态模型

插件用共享 `plugin_state_entries`（键、命名空间、TTL、配额）。通道私有状态不得另开 JSON sidecar。记忆插件槽同时只能激活一个。

---

## 4. 功能设计（全功能）

### 4.1 安装与引导

- 安装脚本（macOS/Linux/Windows）可配置 Node 24.16+ 或 26.1+。
- `openclaw onboard --install-daemon`：模型、工作区、Gateway、通道、技能。
- `openclaw setup`：不完整安装时聊天式引导。
- `openclaw configure`：交互向导。
- `openclaw doctor --fix`：配置/状态迁移、备份 `.bak` 环、健康检查。

### 4.2 Gateway 与守护

- 前台 `openclaw gateway`；`daemon` 管 launchd/systemd/schtasks。
- 默认 `127.0.0.1:18789`。
- HTTP：Control UI、`/__openclaw__/canvas/`、`/__openclaw__/a2ui/`、工具 invoke API、health。
- `openclaw gateway status` / `health` / `logs`。

### 4.3 会话与聊天

- Direct 默认折叠进共享 `main`；群默认隔离。
- 多 agent 路由、绑定、workspace 隔离。
- 流式：partial / block；typing indicators；progress drafts。
- `/compact`、`/queue`、`/steer`、`/model`、斜杠命令。
- 会话列表投影：脏行精确读；干净快照无 SQL；incognito 不进常驻花名册。

### 4.4 Agent 运行时

- 内置 `openclaw` harness。
- 插件 harness：`codex`、`copilot` 等。
- CLI backend：`claude-cli` 等（不是 embedded harness id）。
- ACP / ACPX 桥。
- 子代理、sessions spawn/send/yield、delegate（组织身份）。
- System agent（引导/修复助手）。
- 并行 specialist lanes、queue steering、subagent yield handoff。

### 4.5 工具面（核心 `src/agents/tools/*-tool.ts`）

| 工具 | 函数工厂 | 功能 |
|------|----------|------|
| `sessions` | `createSessionsTool` | 列/读/补丁/归档会话 |
| `sessions_list` | `createSessionsListTool` | 列表查询 |
| `sessions_history` | `createSessionsHistoryTool` | 历史 |
| `sessions_search` | `createSessionsSearchTool` | 搜索 |
| `sessions_spawn` | `createSessionsSpawnTool` | 派生子会话 |
| `sessions_send` | `createSessionsSendTool` | 向会话发送 |
| `sessions_yield` | `createSessionsYieldTool` | 让出 |
| `session_status` | `createSessionStatusTool` | 状态 |
| `message` | message-tool 族 | 通道消息动作 |
| `cron` | `createCronTool` | 自动化 |
| `nodes` | `createNodesTool` | 设备命令 |
| `computer` | `createComputerTool` | 计算机使用 |
| `browser` | 插件 | CDP 浏览器 |
| `exec` / `terminal` | `createTerminalTool` | 主机执行（审批） |
| `screen` | `createScreenTool` | 屏幕 |
| `pdf` | `createPdfTool` | PDF 理解 |
| `image` / `image_generate` | image tools | 图理解/生成 |
| `video_generate` / `music_generate` | media tools | 视频/音乐 |
| `tts` | `createTtsTool` | 语音 |
| `web_search` / `web_fetch` | 运行时 + 插件 | 搜索与抓取 |
| `memory` | 记忆插件 | 搜索/写入/forget |
| `plugins` | `createPluginsTool` | 插件管理 |
| `secrets` | `createSecretsTool` | 密钥 |
| `gateway` | `createGatewayTool` | 网关自省 |
| `dashboard` | `createDashboardTool` | 控制面 |
| `portal` | `createPortalTool` | Portal |
| `progress_card` | `createProgressCardTool` | 进度卡 |
| `ask_user` | `createAskUserTool` | 向用户提问 |
| `subagents` / `agents_list` / `agents_wait` | 子代理 | 列表与等待 |
| `openclaw` delegate | `createOpenClawDelegateToolsForRun` | 委派 |
| `system_agent` | `createSystemAgentTool` | 系统助手 |
| `transcripts` | `createTranscriptsTool` | 会议转录 |
| `theme` | `createThemeTool` | UI 主题 |
| `github_publish` / `github_identity_status` | GitHub | 发布与身份 |
| `skill_workshop` | workshop | 技能提案 |
| `mobile_ui` | `createMobileUiTool` | 移动 UI |
| `heartbeat_response` | heartbeat | 心跳响应 |
| `structured_output` | schema | 结构化输出 |
| `goal` | `goal-tools.ts` | 目标 |
| `conversation` | conversation-tools | 对话辅助 |

消息动作契约由核心拥有；通道只做账户/安全/会话/传输。

### 4.6 通道

核心安装：A2A、Reef、Telegram、WebChat。其余官方插件：Discord、Feishu、Google Chat、iMessage、IRC、LINE、Matrix、Mattermost、Microsoft Teams、Nextcloud Talk、Nostr、QQ Bot、Raft、Signal、Slack、SMS、Synology Chat、Tlon、Twitch、Voice Call、WhatsApp、Zalo、Zalo Personal 等。外部：WeChat、Yuanbao 等。群 mention 激活；DM 配对。

### 4.7 模型与提供商

`provider/model` 选择提供商+模型，不直接选择 runtime。选择顺序：primary → fallbacks → 提供商内 auth failover。`modelPolicy.allow`、utility/decision/image/pdf/media 模型。捆绑提供商插件覆盖 Anthropic、OpenAI、Google、Bedrock、Azure、Groq、Mistral、Ollama、vLLM、SGLang、LM Studio、OpenRouter 等。

### 4.8 记忆

层级 + 出处门控 + Dreaming 巩固 + 召回通道 + standing intents。写路径是安全边界。失败不阻塞回复。LanceDB / wiki / builtin 等记忆插件互斥激活。

### 4.9 压缩与上下文

Safeguard 模式默认。工具调用与 toolResult 成对切分。CJK 估算。溢出识别数十种提供商字符串。取消不回滚已完成压缩。

### 4.10 自动化

Cron、heartbeat、hooks（内部 `HOOK.md` + 插件 `api.on`）、webhooks、flows、standing orders、TaskFlow。

### 4.11 安全

配对、allowlist、sandbox（Docker）、exec 审批、secret store + egress proxy、`openclaw security` 审计、安装策略、live authority 在副作用前重验。

### 4.12 伴侣与节点

macOS 菜单栏、iOS/Android 节点、Linux hub、Windows Hub、语音、Canvas、相机、屏幕、位置、Talk/WebRTC。

### 4.13 控制 UI / TUI

Gateway 同版本分发 Control UI。TUI：`openclaw tui` / `resume` / `chat`。

### 4.14 插件与 ClawHub

Code plugin vs bundle plugin。Manifest-first。SDK 子路径。`openclaw plugins install @openclaw/<id>`。

### 4.15 更新、备份、数据库

`openclaw update` 账本、`backup`、`database preflight`、冷转录、schema 只向前。

### 4.16 MCP / ACP / Browser / 沙箱 / Fleet

MCP 服务器与运行时集成；ACP 桥；浏览器自动化；Docker 沙箱；实验性 fleet cells 与 Claws。

---

## 5. 核心函数（按所有者）

下列为各模块的**公开契约函数**。内部辅助以文件名在 `4SOURCE.md` 列出。命名惯例：`createXTool` 工厂、`registerX` CLI、`*Command` 命令、`scan*` 只读探测。

### 5.1 入口

| 函数/符号 | 文件 | 职责 |
|-----------|------|------|
| CLI bin | `openclaw.mjs` | 包装器：Node 版本、SQLite、生命周期脚本 |
| `runMain` | `src/cli/run-main.ts` | Commander 程序装配 |
| Gateway/agent 入口 | `src/entry.ts` | 进程入口、respawn、compile cache |
| `isSupportedOpenClawNodeVersion` | `node-version.mjs` / `src/infra/runtime-guard.ts` | Node 24.16+/26.1+ |

### 5.2 CLI 注册

`src/cli/program/core-command-descriptors.ts`：`setup`、`onboard`、`configure`、`config`、`backup`、`database`、`migrate`、`doctor`、`triage`、`dashboard`、`reset`、`uninstall`、`message`、`mcp`、`transcripts`、`agent`、`agents`、`status`、`health`、`audit`、`sessions`、`tasks`。

`subcli-descriptors.ts`：`acp`、`gateway`、`daemon`、`logs`、`system`、`models`、`promos`、`telemetry`、`infer`、`approvals`、`nodes`、`devices`、`users`、`node`、`connect`、`worker`、`sandbox`、`fleet`、`worktrees`、`tui`、`cron`、`dns`、`docs`、`proxy`、`hooks`、`webhooks`、`qr`、`pairing`、`plugins`、`channels`、`directory`、`security`、`secrets`、`skills`、`update`、`completion` 等。

关键实现：`getCoreCliCommandDescriptors`、`registerCoreCliByName`、`registerSubCliByName`、各 `src/commands/*Command`。

### 5.3 Gateway

| 函数 | 职责 |
|------|------|
| `listGatewayMethods` | 广告方法目录 |
| `listCoreAdvertisedGatewayMethodNames` | 核心方法策略 |
| 各 `src/gateway/server-methods/*.ts` 处理器 | RPC 族：sessions、agent、config、update、skills、tasks、users、worktrees、tts、terminal、usage、wizard… |
| `session-row-projection.ts` | 常驻会话行；`sessionChanges` 精确失效 |
| `session-row-projection-access.ts` | 请求上下文绑定 runtime owner |
| worker `placement-turn-claims.ts` | worker 回合活性 |
| `src/infra/agent-run-registry.ts` | run 活性 |

### 5.4 Agent loop

| 函数 | 文件 | 职责 |
|------|------|------|
| `runEmbeddedAgent` | `embedded-agent-runner/run.ts` | 主循环 |
| `subscribeEmbeddedAgentSession` | runner | 桥接流 |
| `waitForAgentRun` | gateway agent.wait | 等 lifecycle end/error |
| `rewriteTranscriptEntriesInSessionManager` | transcript-rewrite.ts | 带 writer fence 的重写 |
| `truncateOversizedToolResultsInSessionManager` | tool-result-truncation.ts | 工具结果裁剪 |
| `normalizeContextTokenBudget` | utils.ts | token 预算 |
| `flushPendingToolResultsAfterIdle` | wait-for-idle-before-flush.ts | 空闲后 flush |

Admission：一次录取上下文；`finally` 关闭；harness host capabilities 绑定工具/审批；await 后副作用前重验。

### 5.5 队列

Session lane `session:<key>` + 全局 `main`/`subagent`。默认 steer、500ms debounce、cap 20、drop summarize。`main` 并发 `max(8, CPU*4)`，subagent 默认 8。

### 5.6 插件加载

`src/plugins/`：发现、manifest 校验、activation planner、public-surface-loader（轻路径）、runtime 解析（重路径）。`extensions/*/openclaw.plugin.json` 是元数据源。

### 5.7 状态与迁移

`OPENCLAW_STATE_SCHEMA_SQL` / `OPENCLAW_AGENT_SCHEMA_SQL`；`src/infra/state-migrations*.ts`；`src/infra/update-run-ledger.ts`（更新账本、abandoned 判定：30 分钟 + 驱动进程已死）。

### 5.8 记忆

Memory-core 插件：`memory_index_*` 同步、KNN 搜索、dreaming、provenance 分类、`openclaw memory forget`。Host SDK：`src/memory-host-sdk/` 与 `packages/memory-host-sdk`。

---

## 6. 算法

### 6.1 会话 lane FIFO + 全局并发帽

按 session key 串行，避免转录/工具竞态；全局 cap 限制 LLM 并行。Steer 在运行中注入；Collect 合并安静窗口内消息；Interrupt 中止当前 run。

### 6.2 Writer claim fencing

录取后记录 `activeWriterRunId`。每次转录 append/rewrite 带 `expectedWriterRunId`，同步事务校验。被取代 run 无法提交。

### 6.3 会话行投影

每物理 store 启动时 hydrate 一次。脏键精确读 SQLite；干净物化快照零 SQL。归档行冷索引；incognito 只做瞬时精确读。事件携带 projection generation，替换身份则拒绝发布。

### 6.4 压缩切分

在 token 阈值或溢出错误时，把历史总结为 compact 条目，保留近期尾部。切点不得打断 tool/toolResult 对。Safeguard：摘要必须含必要标题与标识符，失败则不写转录。CJK 按字符加权估算。

### 6.5 模型选择与 failover

primary → fallbacks；提供商内旋转 auth profile 与冷却；overflow 字符串匹配；OpenAI Responses `end_turn: false` 则再请求一轮。Codex 仅在官方 HTTPS Responses 路由且无 authored override 时由 auto 选中。

### 6.6 记忆出处与巩固

出处闭集：owner / agent / untrusted / system。Cron/heartbeat/subagent 不可晋升。召回内容标记后不得再提取。Dreaming 后台巩固；回复路径有超时降级。

### 6.7 幂等与投递

副作用 RPC 要幂等键。出站绑定 `context_id+execution_id+runId`。`sourceReplyDelivered` 才算对源会话投递成功。超时不证明未执行，需按 owner 对账。

### 6.8 更新账本对账

`update_runs` 心跳 30s。自动 abandoned：>30 分钟无更新且本机所有 recorded driver PID 死亡或 startIdentity 变化。身份不可用则排除自动对账。2026.9.2 无 driver 的 requested 行 24h 后 `legacy-driver-expired`。

### 6.9 Control UI 花名册刷新

首事件 200ms debounce；1s 内合并；刷新后等待 `max(1s, min(15s, 3*duration))`。显式刷新绕过退避。

### 6.10 配对签名

challenge nonce；v3 绑定 platform/deviceFamily；元数据变化需重新配对。

### 6.11 Secret egress

明文 HTTP 拒绝；SSRF 护栏；密钥扫描走审计路径。

### 6.12 工具结果预算

按字符/token 截断工具结果；cache-TTL 修剪；图/二进制用标记而非像素进摘要。

### 6.13 ACP 回放字节

`estimated_bytes` = UTF-8 字节 + 每行 32。Doctor 重建总量；超限按既有驱逐顺序。

---

## 7. 测试设计

### 7.1 套件地图

权威：`docs/help/testing/suites.md`。Vitest `threads`、`isolate: false`。约 **16,924** 个 `*.test.ts` 文件。

| 套件 | 命令 | 覆盖 |
|------|------|------|
| 默认单元/集成 | `pnpm test` | 13 个 shard：core-unit-fast/src/security/ui/support、boundary、tooling、contracts、bundled、runtime、agentic、auto-reply、extensions |
| 最大并行 | `pnpm test:max` | 同范围更高 worker |
| 变更路由 | `pnpm test:changed` | 按 diff 廉价 lane |
| Gateway 稳定性 | `pnpm test:stability:gateway` | 真实 loopback Gateway + diagnostics.stability |
| E2E 聚合 | `pnpm test:e2e` | Gateway smoke + Control UI Playwright |
| Gateway E2E | `pnpm test:e2e:gateway` | `*.e2e.test.ts` |
| UI E2E | `pnpm test:ui:e2e` | mocked WS + 部分真 Gateway |
| 覆盖率 | `pnpm test:coverage` | 信息性 V8 |
| Docker live | `pnpm test:docker:live-*` | 需密钥，发布门 |
| QA lab | `pnpm qa:lab:up` | 场景 |
| 类型 | `pnpm check` / `check:test-types` | tsgo lanes |
| 链接/配置示例 | `pnpm docs:check-*` | 文档 |

### 7.2 测试用例分类（按缺陷原序）

每个 `*.test.ts` 用 `describe`/`it` 锁可观察契约。代表族：

**Agent / runner**

- `run.overflow-compaction.test.ts` / `.loop.test.ts`：溢出压缩并重试，不重放已完成工具。
- `compact.hooks.test.ts`：压缩 hooks 经真实 `run.ts`/`compact.ts`。
- `runs.lifecycle.test.ts`：start/finishing/end/error。
- `runs.steering.test.ts`：steer 注入边界。
- `run.session-permissions.test.ts`：会话权限。
- writer-claim 相关：被取代 run 不能 append。

**Gateway / 会话投影**

- `sessions-row-projection.benchmark.test.ts`：干净快照无 SQL。
- `sessions.dispatch*.test.ts`：授权、abandonment、device。
- `sessions.reclaim.test.ts`：回收。
- connect 握手：非 connect 首帧关闭。

**配置 / Doctor**

- schema 未知键拒绝。
- 遗留键迁移写 `.bak`。
- `$include` / Nix 不自动迁移。

**通道**

- 各 `extensions/<channel>/src/**/*.test.ts`：inbound/outbound/pairing/session-route/monitor replay。
- 合同测试：`channel-contract-testing`。

**记忆**

- provenance 分类、tombstone、向量重建、legacy sidecar 导入。

**更新账本**

- abandoned 需要死 PID；identity-unavailable 不自动对账；legacy 24h 过期。

**UI**

- 花名册 reconciler；i18n verify；Playwright 路由。

**安全**

- pairing、sandbox bind、secret redaction、SSRF。

### 7.3 测试文件规模（抽样）

| 区域 | 约文件数 | 典型用例主题 |
|------|----------|----------------|
| `src/agents/**/*.test.ts` | 数千 | 工具、runner、权限、压缩 |
| `src/gateway/**/*.test.ts` | 数千 | RPC、投影、update、skills |
| `src/commands/**/*.test.ts` | 数百 | CLI 输出、status JSON |
| `src/cli/**/*.test.ts` | 数百 | 命令注册、lazy load |
| `extensions/*` | 数千 | 每插件合同 + 行为 |
| `ui/src/**/*.test.ts` | 数百 | 组件、路由、E2E |
| `test/**` | 1679 源文件 | 跨切面 helpers 与集成 |
| `apps/**` | 原生测试 | Swift/Kotlin |

完整测试文件路径按目录计入 `4SOURCE.md` 统计；CI shard 清单在 `vitest.full-*.config.ts`。

### 7.4 测试原则

- 回归必须打在原始缺陷上。
- 禁止用重试、加长超时、放宽断言、扩大 mock 掩盖 flake；遇到 flake 要修根因。
- 解析/注册表测试用生成的小插件夹具，不用真实捆绑插件 API 证明路径回退。
- Embedded runner 的消息工具发现与压缩必须保留经 `run.ts` 的集成证明。

---

## 8. 安全与权威模型

1. 入站消息视为不可信。
2. 特权动作需要 **当前 owner 持有的 live authority**；token/签名/TTL/ID 匹配不够。
3. 审批身份行只记录 provenance。
4. Worker 需要精确 placement、environment、owner epoch、placement generation、turn claim。
5. 方言不兼容则拒绝并 reprovision，不降级 payload。
6. 工具可用性 ≠ 授权。
7. 最小有效限制：默认可工作；风险路径显式且操作者可控。

---

## 9. 构建、发布与兼容

- 版本日历：`YYYY.M.PATCH`。
- State/agent schema 只向前；旧构建拒绝新库。
- 配置无长期别名；破坏性变更必须带 Doctor 迁移。
- 插件 SDK 破坏走版本化类型与捆绑调用方一次性迁完。
- 更新：已安装 updater 先跑，不能给自己打补丁；候选修复依赖已发布 driver 的标记。

---

## 10. 文档与源码对照

| 主题 | 源码所有者 | 公开文档 |
|------|------------|----------|
| 架构 | `src/gateway`、`src/agents` | `docs/concepts/architecture.md`、`agent-loop.md` |
| Schema | `src/state/*.sql` | `docs/reference/database-schemas/` |
| 插件 | `src/plugins`、`extensions` | `docs/plugins/` |
| 测试 | `test/`、`*.test.ts` | `docs/help/testing/suites.md` |
| 配置 | `src/config` | `docs/gateway/configuration.md` |
| 协议 | `packages/gateway-protocol` | `docs/gateway/protocol/` |

本说明书与上述源码在 **2026.9.5 / state 17 / agent 21** 对齐。Schema 或 RPC 变更时应同步本文件对应表与函数族。
