# OpenClaw 后续优化方案（面向 2026 Agent 趋势）

- **基线：** 当前仓库版本 2026.9.5，state schema 17，agent schema 21
- **立场：** OpenClaw 已从 2025 年的“能在通道里跑工具的个人助手”长成 **可信 Gateway + 不可信执行 + 确定性策略** 的控制面。本方案不另起炉灶，只在现有所有者上加能力或收口重复实现。
- **配套：** [1DESIGN.md](1DESIGN.md)、[2HELP.md](2HELP.md)、[4SOURCE.md](4SOURCE.md)

下文把 2026 年 Agent 产品的主流变化映射到 OpenClaw 已有缝（harness、记忆出处、队列 steer、Plugin SDK、session 投影、live authority），并给出可落地的工作包。

---

## 1. 2026 年 Agent 趋势与 OpenClaw 现状对照

| 2026 趋势 | 行业表现 | OpenClaw 已有 | 缺口 |
|-----------|----------|---------------|------|
| **Harness 分化** | 模型提供商与执行环分离（Codex app-server、Claude Code、Copilot CLI、ACP） | `agentRuntime.id`、embedded / CLI backend / ACP 三族 | 用户仍混淆 provider/model/runtime；harness 能力矩阵不完整 |
| **长程记忆要可审计** | 写路径比检索更重要；防毒化 | 出处列、Dreaming、tombstone、单记忆插件槽 | 巩固质量评测弱；跨会话“用户模型”未产品化 |
| **可中断的长任务** | steer、队列、子代理、handoff | session lane、steer/collect、sessions_yield、TaskFlow | 长任务可见性与跨通道收据不统一 |
| **Computer-use / 浏览器** | 桌面与浏览器成为一等工具 | computer/browser/sandbox/nodes | 观察环、权限分级、回放证据偏碎片 |
| **MCP 实用化** | MCP 当工具总线，而不是第二个 agent | MCP server + 运行时集成 | 与技能/插件/ClawHub 的发现体验仍并行 |
| **团队 Agent** | 具名委派、多人会话、归属 | delegate 架构、session sharing、Git co-author | 组织 IdP、发送“on behalf of”的通道覆盖不足 |
| **本地与混合推理** | Ollama/vLLM + 云 failover | 大量提供商插件 | 路由策略仍偏静态；设备端小模型未打通节点 |
| **实时语音** | 双向语音、会议机器人 | Talk/WebRTC、meeting transcripts、TTS、voicewake | 端到端延迟与打断策略未成为默认 UX |
| **评测即产品** | 个人助手基准、轨迹、回归 | trajectory 表、QA lab、claw-score | 缺少发布门上的长程任务黄金集 |
| **上下文工程** | 压缩质量、工具结果预算、context engine | safeguard compaction、tool truncation、context-engine 目录 | engine 仍偏实验；提示稳定前缀未充分产品化 |
| **Skills 生态** | 可安装、可签名、可撤销的技能包 | ClawHub、skill library SQL、workshop | 技能运行沙箱与权限声明对用户不够一目了然 |
| **证据与问责** | receipt、审计、live authority | audit ledger、execution identity、writer claim | 用户可读的“这次为什么能执行”仍偏运维 |

原则：**优先扩展现有 owner**；新能力先插件；重复需求才升格为 SDK 契约；核心提示/工具每加一项都有上下文税。

---

## 2. 方案 A — Agent Runtime 产品化（最高杠杆）

### 2.1 问题

2026 年用户买的是“执行环”，不是模型名。OpenClaw 已经分开了 provider / model / runtime，但 Control UI、`/model`、status 仍容易显示成一个下拉框。

### 2.2 目标

一次选择拆成三行，且 **runtime 选择不暗示计费**：

1. 提供商与认证档案（Anthropic API vs Claude CLI vs Codex OAuth）
2. 模型 ref
3. Harness（`openclaw` / `codex` / `claude-cli` / `copilot` / `acp`）

### 2.3 工作

- 在 Control UI 与 `openclaw models status` 复用同一投影：认证、计费、runtime、通道互不影响。
- 为每个捆绑 harness 维护**能力矩阵**（steer、并行工具、电脑使用、MCP、压缩所有权），挂在 plugin manifest，不进核心提示。
- Doctor：继续把遗留 whole-agent runtime pin 迁到 model-scoped policy（已有方向，做到默认路径零遗留）。
- 文档与 onboard：用“你要哪一种执行环”代替“选一个模型供应商”。

### 2.4 验收

同一 `openai/gpt-…` ref 在 API key 与 Codex 订阅下展示不同 runtime/计费；切换 harness 不改会话 key 合同。

---

## 3. 方案 B — 长程记忆从“文件堆”变成“可证明的用户模型”

### 3.1 问题

2026 年长程 agent 失败主要在**写入**：心跳噪声、召回回环、未标注的网页毒化。OpenClaw 的出处模型是对的，但操作者几乎看不到。

### 3.2 目标

- Control UI 增加 Memory inspector：按 origin class 过滤、看 Dreaming 报告、手动晋升/降级。
- 把 `USER.md` 当用户模型的唯一策核心；Dreaming 只提议 diff，应用走与 skill workshop 类似的提案表（已有 `skill_workshop_*` 可作模式，不要新 JSON 仓）。
- Standing intents 与 cron 在 UI 上同列表，避免“前瞻记忆”和“定时任务”两套心智。

### 3.3 工作

- 记忆插件槽保持单一激活；把 inspector 放在 host SDK 契约，而不是每个记忆引擎各做一套 UI。
- 发布一小套长程回归（个人助手基准包文档已存在：`docs/concepts/personal-agent-benchmark-pack.md`）作为 `pnpm test` 之外的夜间 lane，不放进默认 PR 税。
- `openclaw memory forget` 与会话删除在 UI 暴露，并显示墓碑覆盖范围（已有 provenance 文档边界）。

### 3.4 验收

未信任网页内容不能出现在 `MEMORY.md` 除非操作者确认；关闭记忆引擎不得阻塞回复（已是设计原则，加 UI 失败可见）。

---

## 4. 方案 C — 长任务操作系统（steer / 子代理 / 收据）

### 4.1 问题

2026 的工作型 agent 是小时级任务：浏览器、仓库、会议。OpenClaw 有队列与 yield，但用户看见的是聊天气泡，不是任务。

### 4.2 目标

统一 **Task 作为一等 UX**：`task_runs` / Control UI Activity / 通道进度卡 / `sourceReplyDelivered` 收据走同一生命周期。

### 4.3 工作

- 把 progress card、Activity recap、cron receipt 投影到同一任务对象（只读投影，写入仍归现有表）。
- Steer 在所有 harness 上对齐语义：文档已区分 Codex `turn/steer` 与内置工具边界；做一张运行时合同测试，而不是各插件一份故事。
- 子代理完成通知：强制走 announce 目标解析（已有 `sessions-announce-target.ts`），回归“发到错误频道”类缺陷。
- 可选：对超长 run 提供“公园式”挂起（保留 writer claim 与 placement，不占用模型流）——必须用现有 session window reason，不新造会话存储。

### 4.4 验收

从 Discord 发起的长任务，在 Control UI 能看到同一 `runId`、能否 steer、是否已投递到源会话。

---

## 5. 方案 D — Computer-use 与沙箱分级

### 5.1 问题

趋势是“agent 真的用电脑”。OpenClaw 工具面已经很大；风险是**默认过强**或**沙箱过弱**。

### 5.2 目标

三级执行轮廓，全部落在现有 sandbox/approvals owner：

| 轮廓 | 默认 | 用途 |
|------|------|------|
| Chat-only | 无 exec | 只读通道 |
| Sandboxed | Docker | 群、访客、未知 DM |
| Privileged | 主机 + 审批 | 操作者主会话 |

### 5.3 工作

- onboard 按威胁模型选轮廓，而不是事后读安全文档。
- Computer-use 观察环（截图、DOM、辅助功能树）统一经 `computer` 工具与 node caps；回放证据进 audit **元数据**（路径、哈希、结果码），不把像素拷进 audit 表。
- Crabbox 继续作为云沙箱提供商的家；OpenClaw 只保留编排与策略。

### 5.4 验收

新装默认不能让未知 DM 在主机 `exec`；操作者一键提升有明确警告。

---

## 6. 方案 E — 上下文工程默认化

### 6.1 问题

2026 年“提示工程”让位给 **context engineering**：稳定前缀、工具结果预算、压缩质量、前缀缓存。

### 6.2 目标

- 保持 transcript 字节稳定（已是核心原则）；把 context-engine 从实验收口到单一 owner（`src/context-engine/`），或删除遗留 `legacy.ts` 双路径。
- Safeguard 压缩的质量审计对操作者可见（失败原因，而不是静默 skip）。
- 工具结果截断策略按模型家族共享 helper（Plugin SDK 已强调 family helper），避免每提供商一份 lambda。

### 6.3 工作

- 测量热路径：系统提示+工具 schema 的前缀稳定性（便于提供商 prefix cache）。
- 为 CJK 压缩估计加黄金集（算法已考虑 CJK）。
- 明确 `NO_REPLY` 与“必需回复 finalization”的产品文案，减少“以为静音授权”的误解。

---

## 7. 方案 F — 语音与会议作为一等入口

### 7.1 问题

Talk、meeting-bot、实时转写、TTS 已经在源码里，但安装心智仍是“先打字”。

### 7.2 目标

一条 onboard 路径：麦克风 → 打断 → 转录写入 meeting 表 → 可选总结进情节记忆（untrusted，除非操作者在场确认）。

### 7.3 工作

- 统一打断：steer 队列 + Talk 模式事件共用“用户开口即注入”。
- 会议 occupancy 重开规则（已有 `sessionIdOrigin`）在 UI 上可解释。
- 节点侧语音走现有 caps，不把音频二进制写入 state DB。

---

## 8. 方案 G — 发现面收口（MCP / Skills / Plugins / ClawHub）

### 8.1 问题

2026 年 MCP 爆发导致每个产品都出现四套“加能力”入口。VISION 已写明不复制路径。

### 8.2 目标

用户只看见 **一个安装动作**，内部路由到 bundle plugin / code plugin / MCP server / skill。

### 8.3 工作

- Control UI “Extensions” 按能力搜索，而不是按实现类型。
- Manifest-first：安装前不执行插件代码（已是边界）。
- 安装策略 `security.installPolicy` 覆盖 CLI 与 UI，不靠 `before_install` 钩子当安全边界。

---

## 9. 方案 H — 团队与委派

### 9.1 问题

个人助手和团队 Gateway 已是产品目标；delegate 文档存在，通道“on behalf of”覆盖不齐。

### 9.2 工作

- 会话 sharing 与 profile catalog（投影层已有）做成默认团队 UX：谁能看、谁被归因。
- 委派发送走通道账户身份，永不冒充人类（保持现有原则）。
- 站立指令与审批在组织策略里可导出，但不把 IdP 做进核心——用插件。

---

## 10. 方案 I — 成本、路由与本地混合

### 10.1 工作

- 决策/效用/主模型三路已经分开；补**任务级路由策略**：检索用本地 embedding，编码用云，闲聊用小模型——策略数据放配置 schema，不放新存储。
- 节点设备上的小模型（如 macOS MLX TTS 已有 `apps/macos-mlx-tts`）推广为可选 local runtime 插件。
- usage 投影（已有 usage RPC）给操作者看 per-run 成本，而不是只有提供商控制台。

---

## 11. 方案 J — 评测、轨迹与发布门

### 11.1 工作

- 把 `trajectory_runtime_events` 接到可选导出（操作者同意），用于本地回归，不上报基金会。
- 个人助手基准包：安装、配对、压缩、记忆晋升、steer 各一条黄金路径，进夜间 QA，不进默认 `pnpm test`。
- 对 harness × 提供商做“发布单元格”（VISION 的 updates 证明模式的推广）：每个捆绑 harness 至少一条 live 或录制契约。

---

## 12. 明确不做什么

1. **不在核心再堆一套编排器。** 已有 agent loop、TaskFlow、cron。
2. **不新建 JSON/JSONL 状态仓。** 状态进 SQLite。
3. **不把云沙箱提供商做成 OpenClaw 插件。** 归 Crabbox。
4. **不为每个提供商复制 stream wrapper。** 升格 family helper。
5. **不把 MCP 做成第二条技能市场。** ClawHub + 插件合同。
6. **不加默认遥测。** 诊断 opt-in；审计保持无正文。
7. **不在 UI 做 Gateway 决策。** 渲染器只缓存。

---

## 13. 建议实施顺序（按依赖，非日历）

1. **Runtime 三行选择 + Doctor 清遗留 pin**（用户每天碰到，改动面在 UI/status/config 投影）。
2. **执行轮廓 onboard**（安全默认，少代码多产品）。
3. **任务对象只读投影**（表已在，做 UI/RPC 聚合）。
4. **Memory inspector + 晋升提案**（复用 workshop 模式）。
5. **Harness 能力矩阵与 steer 合同测试**。
6. **Extensions 统一发现**。
7. **语音 onboard 路径**。
8. **夜间长程基准**。
9. **委派通道覆盖**（按需求最高的邮件/日历插件）。
10. **Context-engine 单所有者收口**。

每一步保持：一个 owner、调用方一次切完、证明走真实入口、UI 可见变化附带截图门（若改 Control UI）。

---

## 14. 风险与缓解

| 风险 | 缓解 |
|------|------|
| 核心提示变长 | 能力放插件与按需工具面；`toolsAllow` 已支持收窄 |
| Schema 膨胀 | 优先 JSON 列内加法与无 bump 索引修复（已有先例） |
| Harness 行为分叉 | 合同测试打在 SDK，不打在每个插件的故事测试 |
| 记忆误伤 | 晋升需操作者；失败不阻塞回复 |
| 更新不能破旧安装 | 候选修复只依赖已发布 driver 标记；账本 abandoned 逻辑保持保守 |

---

## 15. 成功标准

操作者可以在不读架构文档的情况下回答四件事：

1. 这一回合用的是哪套 **执行环**、谁在计费。
2. 助手“记住”的某句话来自 **owner / 网页 / 群成员** 中的哪一类。
3. 一个跑了 40 分钟的任务现在能否 **steer**、有没有 **投递收据**。
4. 未知私聊能否在本机执行命令（答案必须是 **不能**，除非显式提升轮廓）。

达到这四点，OpenClaw 就从 2025 年的多通道 chatbot，对齐 2026 年“可问责的计算机代理”主线，同时不破坏现有 Gateway 合同。
