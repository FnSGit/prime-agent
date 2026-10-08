# Prime Agent 的 self-improving coding/research：机制、代码证据与实践方案

> 综合报告。四份子系统调研在本目录（各带完整 file:line 证据）：[refinement-pipeline.md](refinement-pipeline.md) · [continual-harness-digest.md](continual-harness-digest.md) · [rlm-multi-agent.md](rlm-multi-agent.md) · [skills-compaction-goal.md](skills-compaction-goal.md)。
> 仓库：/Users/fengshuai/IDE/ai-agent/prime-agent（Rust，main @ 967eb13fd）。

---

## 0. 结论先行

Prime Agent 的 self-improving **不是权重训练**，而是"运行时自我改造"：

1. **可编辑的持久状态层**（Continual Harness）：prompt/memory/skill/subagent/factory 五类条目，落盘为 JSON，独立于 token 历史，跨 compaction 与跨会话存活。
2. **修改该状态的自动化管线**（Refinement）：会话轨迹 →（LLM 审核 → LLM 提案 → 校验 → 原子落盘 → 审计）→ 状态更新，下轮对话即生效。
3. **让长任务不散架的上下文工程**：持久 Python 内核（RLM）+ compaction + harness-digest 定向注入 + goal/autonomous 续跑。
4. **多代理组织**（rlm.spawn / agent_message / collect）：把"做研究/写代码"分解为可复用的子代理规格，沉淀即能力。

代码里的自我定位（REFINEMENT_SYSTEM_PROMPT, planner.rs:10）：
"This is similar in spirit to context compaction, but instead of summarizing the conversation you emit precise Create, Update, or Delete edits to reusable state."

---

## 1. 机制拆解与代码证据

### 1.1 数据层：Continual Harness

- HarnessEntry（refinement/mod.rs:51-69）：id / kind / title / content / path / scope / reference / arguments / metadata / source("agent"|"refine") / created_at / updated_at / **version(每次 update +1)**。
- 五类 kind（mod.rs:10）：prompt（行为策略补充）、memory（事实/教训/偏好）、skill（Python 可调用技能）、subagent（委派规格）、factory（子代理状态机工作流，独立 opt-in 门）。
- HarnessState（mod.rs:85-92）：BTreeMap 嵌套保证写盘 key 顺序稳定；附 refinements 事件表。
- 落盘（mod.rs:122-134）：global `<agent_dir>/harness/harness_state.json`；local `<session-artifacts>/<id>/harness/harness_state.json`；另有 refinement_history.jsonl 审计。
- 并发安全：`update_harness_state` 用 `{file}.lock` 目录锁 + reload→改→原子写+fsync（mod.rs:210-318）；Python 内核侧同构实现（harness.py:626-657, os.replace + F_FULLFSYNC）；文件损坏降级为空、不崩 prompt 构建（mod.rs:172-180）。
- 两侧（Rust 宿主与 Python 内核）读写**同一份文件、同一 schema**——这是"记忆即数据"而不是"记忆即提示词"的关键。

### 1.2 修改层：Refinement 管线（三条入口，同一执行器）

**入口 A：agent 主动调用** `await refine.run(instructions=None, global_=False)`
- Python skill 只是薄封装：skills/refine/src/refine/__init__.py:50 → host_request("refine.run")。
- host 只调度不打断：turn_boundary.rs:320-369 要求当前 turn 正在流式，合并 pending，立即返回 scheduled=true；**回合结束的边界**才执行（consume_pending_refinement, turn_boundary.rs:411-434，RefinementSource::SelfRefine）。

**入口 B：用户命令** `/refine [--global] [instructions]`、`/refine rollback <id>`（pa-daemon/session_custom.rs:291，RefinementSource::User）。

**入口 C：自动触发（compact 臂）**
- compaction 成功 → mark_compact_auto_refine_pending（auto_refine_trigger.rs:50-58，仅 depth-0 会话 arm，wiring.rs:105-108）。
- 下一个 settled turn 边界后台消费（pa-daemon/worker/turn.rs:1115）：门序 = 表面门(auto_refine_allowed) → gates.enabled → gates.compact → 冷却(默认 20min) → in_flight 去重（auto_refine_trigger.rs:132-178）。
- 通过后先跑 **LLM 审核门**（AUTO_REFINE_REVIEW_SYSTEM_PROMPT, planner.rs:12）：输出 JSON `{shouldRefine, rationale, instructions}`，"宁缺勿滥"；agent 又在流式则 Deferred 留待下个边界；分支切换（branch_version 变）直接丢弃，绝不给废弃分支落盘（auto_refine_trigger.rs:180-231）。

**执行器** execute_refinement_with_rows（refine.rs:301-445）：
1. 组装 prompt：当前轨迹 + harness 总览（每 kind 40 条/240 字符, executor.rs:41-76）+ 全部 refinement 历史；超窗时二分截断最老对话（planner.rs:578-632）。
2. plan_refinement（executor.rs:194-268）调 LLM（REFINEMENT_SYSTEM_PROMPT）输出严格 JSON：`{summary, rationale, expectedOutcome, edits[{action,kind,id,title,content,path,reference,arguments,metadata,reason}]}`。
3. **LLM 规划期间重读磁盘**（refine.rs:372-376）+ baseline 并发检测（planner.rs:357-372，"entry changed during refinement planning" 拒绝）。
4. apply_refinement_proposal 逐条校验（planner.rs:193-285）：base 系统提示禁改；skill 必须带 python reference+arguments；factory 受 apply 时刻重读的 `factory.enabled` 门控；单条失败只记 error 不阻断整批。
5. 落盘：锁+原子写（mod.rs:294-318）；global 追加 refinement_history.jsonl（mod.rs:429-444）；会话内追加三类行——审计(prime-agent.refinement)、outcome(给 TUI)、notice(给模型，display=false)（refine.rs:415-443）。
6. 每条成功编辑保留 **before/after 快照** → 支持事后 `/refine rollback`（逆序重放提案，planner.rs:503-547；注意是"再提案一次"而非事务 undo）。
7. 审计事件 RefinementEvent{id=refine_<ts>, trigger=summary, changes, evidence=rationale, outcome=expectedOutcome, created_at}（planner.rs:483-490）。

**模型选择（关键）**：refine 的 plan 与 review 都直接用**会话模型**（compact_autorefine.rs:67-73 注释 R8）；auxiliaryModel 只服务 compaction/branch 摘要（auxiliary_model.rs:1-4）。输出预算 refine 32k / review 4k（planner.rs:14-17）。

**gates 配置**（settings.autoRefine, refine.rs:38-64）：enabled=true / turnInterval=25 / compact=true / cooldownMs=20min。

### 1.3 呈现层：[harness-digest] 定向注入

- 注入时机 = **冷边界**：会话启动（空上下文则推迟到首轮 prompt 行，admission.rs:141）、resume（指纹陈旧则追加）、compaction 提交（digest+指纹存进 CompactionEntry，重建时拼在 [compaction-summary] 前，messages.rs:352-368）。正常轮次零开销。
- 陈旧判定用 **state_fingerprint** 而非文本，指纹刻意排除 query_terms（ranking.rs:403-406）；旧 digest 行按字节精确替换，用户引用 digest 原文的消息不被误删（harness_digest.rs:511-540）。
- 渲染：merge global+local → **tf-idf 相关性排序**：query terms = goal objective(权重3.0) + 最近 4 条对话(2.0/1.5/1.0/1.0)，CJK 按二元组切词（harness_digest.rs:39-63, ranking.rs:17-60,144-175）→ 每 kind 只取前 3 条、每条 140 字符、最近 10 条 refinement（mod.rs:17-19 DEFAULT_OVERVIEW_*）。溢出提示用 `rlm.harness.search`。
- digest **不进摘要器输入**（compact_session.rs:69-72）——记忆有独立通道，不参与"总结对话"。

### 1.4 执行层：RLM 内核 / skills / 多代理

**Python skill（能力进化的主载体）**
- 判定：SKILL.md + pyproject.toml + src/<import>/__init__.py（discovery.rs:35-65）。
- 安装：`uv pip install --editable` 批量装进共享 kernel-venv，仅 pyproject 哈希变化的才重装（venv.rs:166-252, venv/skills.rs:15-20）。
- 预导入：bootstrap 生成 importlib 代码（runtime_code.rs:78-175）——模块有 `run` 则包装成 `await skill(...)`（签名/doc 上提到模块级）；导入失败生成明确抛错的占位对象 + 宿主解析标记行，**下一轮模型就能看到"该 skill 不可用"提示**（provisioner.rs:1188-1201）。
- markdown skill 只进 <available_skills> 清单；`/skill:<name>` 展开成 <skill> 块走提交路径（admission.rs:86-93）。

**多代理（研究/编码的组织方式）**
- rlm.spawn 只返回**准入门票** RLMSpawnHandle{rlm_child_id,name,session_dir,model}（host.rs:259-264）；任务 prompt 由分离 tokio 任务在**父回合边界后**才投递（host.rs:203-206）。没有答案返回通道——这是设计而非缺陷：结果只能经 agent_message（正式汇报）或子会话终态通知回父上下文；子代理 settle 且从未回复时父收到 `[child-exited: no-reply]`（lifecycle.rs:591-606，replied_since_task 由 worker 收到子消息时置位，worker/input.rs:280-288）。
- rlm.collect = 快照 + **共享预算**有界等待；超时返回当前快照、从不报错、从不 steer 父会话（host.rs:463-537）；answer_preview 只是最后助手文本前 160 字，不是正式汇报。
- 深度闸门默认 2（rlm_children.rs:33）；模型白名单 **fail-closed**，解析失败大声拒绝不回退（model_allowlist.rs:12-60）；create_session 仅 depth-0（host.rs:275-277）。
- agent_message 角色定位：关系只由 durable 父子边判定（message.rs:123-157）；role+name 必须在 family 名册恰好命中一个。

**长会话支撑**
- compaction 三段式：prepare(锁内)→summarize(锁外，split-turn 双摘要并发，可走 auxiliaryModel)→commit(锁内，prefix intact 校验失败返回 Ok(false) 让调用方重做)（compact_session.rs:147/223/453）；提交后追加 ipython_state 行告知"内核命名空间仍有效"。
- goal：状态全在 host（Python 侧三行 host_request）；回合结束 mint_goal_continuation 注入续跑；失败回合 drop_failed_goal_continuation 摘掉尸体；objective 按**不可信数据**包裹 <objective>（goals.rs:342-369）。
- autonomous：AutonomousDriver trait 接缝 + shell 质量门（gates）作为完成判据 + 工作区快照抑制重复跑；限额 continuations 3/turns 12/tokens 80k/30min（autonomous/mod.rs:22-30）。

---

## 2. 案例分析

### 案例 A：本机 global harness 的真实沉淀（自我改进的第一手证据）

`~/.prime/agent/harness/harness_state.json`（本次实测）：
- 5 条 global 条目，全部 source="agent"（直接 CRUD，未经 refine 管线），refinement events = 0：
  1. shell 命令一律走 ipython+subprocess（prompt，v1）——源于 bash 工具间歇性 "Tool bash not found" 故障；
  2. pi 扩展 prime agent 化三条路结论（memory，v1）；
  3. git-safe：丢弃型 git 的备份守卫（memory，**v3**——三次证据迭代）——源于 llm-proxy 未提交改动被误杀事故；
  4. watchdog-diff Python skill（memory，v1）；
  5. prime-agent 仓库 git 布局（memory，v2）。
- 含义：self-improving 有**两条写路径**——agent 直接 create/update（轻、无审计事件、即时）与 refine.run 管线（LLM 提案 + summary/rationale/outcome 审计 + 可回滚）。事故→教训→条目→后续所有会话行为改变，闭环成立（git-safe v1→v3 的版本演进就是证据）。

### 案例 B：一次 auto-refine 的完整生命周期（代码走读）

用户会话中 compaction 触发 → arm（auto_refine_trigger.rs:50）→ 25 分钟冷却已过 + 非流式边界 → review 模型调用（4k 预算，输出 {"shouldRefine": true, "rationale": "...", "instructions": "..."}）→ run_approved_refine 以 RefinementSource::Auto 执行 → plan（32k 预算，轨迹+总览+历史 → JSON edits）→ 逐条校验/版本+1/before-after 快照 → 原子落盘 + JSONL 审计 → notice 行进入 live context → **下一轮模型直接看到更新的记忆**。中途任一环节（冷却/忙碌/分支切换/agent 转流式）都会安全推迟或丢弃，不干扰用户。

### 案例 C：本任务自身——多代理研究的实战演示

- 主会话用 4 个并行 sonnet 子代理（refine/harness/rlm/skills）分头深挖，报告落盘 fan-in（即本目录四份子系统报告），主会话亲自核查最核心证据（REFINEMENT_SYSTEM_PROMPT、auto_refine_trigger 状态机、runtime_code bootstrap、auxiliary_model 边界）后再综合——**子代理产出上下文，主会话持有判断**。
- 两个真实的失败-恢复案例：① spawn 被同步 helper 包装未 await → 静默未创建（RuntimeWarning 是唯一线索）；② refine 子代理第一回合结束却未写文件未汇报 → 主会话 steer 补救（明确列出两项交付物）→ 补齐成功。这两个教训已沉淀为 global memory「子代理交付纪律」——本会话在研究 self-improving 的同时完成了一次 self-improvement。

### 案例 D：技能化——能力面的扩展方式

watchdog-diff（pi TS 扩展 → prime agent python skill）、git-safe（TS 扩展等价移植为独立命令）、llm-proxy-models（模型选型 skill）：三者都遵循"重复出现的需求 → 固化为 skill/命令 → 系统提示的 <available_skills> 自动发现 → bootstrap 预导入 → 所有后续会话可直接 await 调用"。skill 不是配置，是**装进内核 venv 的真包**（uv pip install --editable + 哈希增量重装）。

---

## 3. 正确使用方案与实践

### 3.1 沉淀决策树（什么时候写什么）

```
观察到模式/教训
├─ 重复出现的委派角色      → subagent spec（rlm.harness.create_subagent）或封装 spawn 模板
├─ 重复出现的操作流程      → skill（Python 可调用，走 skill-creator）
├─ 稳定事实/偏好/环境结论   → memory
├─ 窄的行为策略修正        → prompt note
└─ 一次性任务状态/临时上下文 → local memory（会话结束即弃）
```

**scope 选择**：默认 local（当前会话）；global 只放"跨会话稳定的教训/偏好/可复用工件"，项目级教训要写进标题/内容并注明项目名。

**写路径选择**：
- 明确知道要写什么 → `rlm.harness.create_memory/update_memory` 直接写（快、可控）；
- 轨迹里有可提炼但说不清的内容 / 需要审计与回滚 → `await refine.run(instructions=...)`；
- 系统检测到重复失败时会建议 refine——保持条目 lean，删掉过时条目（update/delete 同样是一等公民）。

### 3.2 子代理调度纪律（事故沉淀）

1. `await rlm.spawn(...)` 链路上每一环都要 await；同步 helper 包装 = 静默不 spawn。
2. spawn prompt 必须写明完成定义：**写文件（fan-in 落盘）+ agent_message.send 回报**，"写完才算完成"；不是每条消息都需要回复，但需要结论的任务必须显式要求。
3. 子代理 settle 但未交付 → 先发 steer 消息（receiver_role="child"）补交付，不要直接重开（省额度、保留已做的工作）。
4. 只读任务共享 cwd；写任务必须 worktree/独立 cwd；动手前查杀同项目残留 worker。
5. fan-in 用文件（/tmp/工作目录落盘）+ collect 快照轮询；collect 超时是快照不是错误，别把它当同步调用。
6. 模型分档：scout=haiku / researcher=sonnet / planner,reviewer=opus / 顾问=fable；执行类子代理**必须显式传 model**（无全局默认时继承父会话 opus+max 极浪费）。
7. 保留 handle；内核重启/压缩后用 rlm.list_subagents() 恢复。

### 3.3 长会话与压缩配合

- 大数据（报告/数据集/代码中间产物）落盘，REPL 只留变量引用——compaction 会清掉 >16MiB 的变量，磁盘文件不受影响。
- 依赖 harness-digest 自动注入记忆，不要手动在对话里复述记忆全文；细节用 `rlm.harness.search` 按需查。
- 需要"做到完成为止"的任务用 goal.create（用户显式要求才建）+ 完成判据靠 goal.complete() 的纪律；无人值守用 /autonomous（质量门决定完成，agent 无权自判）。
- 心跳（rlm_heartbeat）用于定时自唤醒的巡检/跟进任务。

### 3.4 配置要点（settings.json）

- `autoRefine: {enabled, turnInterval, compact, cooldownMs}`：auto-refine 仅 depth-0 且有本地 harness 目录的会话生效；想更频繁沉淀可调低 cooldownMs/turnInterval，但 review 门仍会"宁缺勿滥"。
- `auxiliaryModel`：只影响 compaction/branch 摘要，**不影响 refine**（refine 跟会话模型）；想省 refine 开销请换会话模型或降低触发频率。
- `factory.enabled`（默认关）：允许 refinement/内核写 factory 状态机；apply 时刻实时重读，中途翻转立即生效。
- `allowedModels`：fail-closed 白名单，spawn 不在名单直接失败（不静默回退）。
- `rlmMaxDepth`：递归上限（默认 2）。

### 3.5 反模式清单

- 把 harness 当聊天记录囤积（digest 每 kind 只取 3 条/140 字符，条目过多会互相挤掉——lean 是硬要求）。
- 用 refine 写"本任务进行到第 3 步"这类易腐内容到 global（应 local）。
- 给 skill 起与已装 skill 相同的 python import 名（撞车只 warn，后者静默覆盖前者）。
- 在 skill 目录里嵌套 skill（含 SKILL.md 的目录不再下钻）。
- 把 collect 当"等答案"的同步阻塞（它永不报错、超时返回快照，答案永远在 agent_message 里）。
- 在 prompt/记忆里复述系统提示已承诺的内容（基座提示不可变，prompt note 只放增量策略）。

### 3.6 已知缺口（Rust 版现状，使用时绕开）

1. `autoRefine.turnInterval` 被解析但**无消费点**——当前只有 compaction 臂会自动触发 refine（TS 有 interval 臂）。
2. `refine.status()` 的 in_flight 恒为 false（TS 后台规划未移植）。
3. 子代理 roster 里 progress_note/replied_since_task 字段未接线（rlm_children.rs:728-729 恒 None）——别依赖 collect/list 读子代理进度，用 steer 消息问。
4. refine 的 plan/review 走会话模型，长轨迹 refine 开销可观；cooldown 是主要节流手段。
5. 回滚是"再提案"不是事务 undo，与后续编辑可能冲突拒绝。

---

## 4. 附：证据文件

- [refinement-pipeline.md](refinement-pipeline.md) —— refinement 管线全链路
- [continual-harness-digest.md](continual-harness-digest.md) —— 存储与 digest 注入
- [rlm-multi-agent.md](rlm-multi-agent.md) —— spawn/collect/消息路由
- [skills-compaction-goal.md](skills-compaction-goal.md) —— skills/工具面/compaction/goal/autonomous
