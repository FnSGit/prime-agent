# Prime Agent self-improvement：refinement 管线调研报告

> 只读调研（未修改任何仓库文件）。所有结论附 `文件:行号` 证据。

---

## 1. 机制流程（端到端）

`refine.run()` 只做**调度**，真正的 refinement 在**回合边界**（turn end）同步执行。链路共 3 段：

### 1.1 REPL 侧：请求入队（Python kernel）
`refine.run(instructions, global_)` 是 kernel 内的一个瘦包装，仅调用通用 host 桥：

- `skills/refine/src/refine/__init__.py:50` — `return await host_request("refine.run", payload)`
- 参数类型检查：`__init__.py:39-44`（`instructions` 必须 str/None；`global_` 必须 bool）

### 1.2 Host 侧：调度为 PendingRefine（Rust）
`refine.run` 由 `TurnBoundaryRequests` 注册的 host handler 处理：

- `crates/pa-core/src/session_engine/turn_boundary.rs:320-369` — 注册 `"refine.run"`：
  - 必须**正在流式**（`state.is_streaming`，:345-350），否则返回 `no_active_turn`（"refine can only be requested while a turn is running"）。
  - 合并进 pending：`instructions` 新值优先、缺省保留旧值；`global` 同理（:351-363）。
  - 立即回 `{"scheduled": true, note: "Refinement runs when the current turn ends..."}`（:364-367）。
- `refine.status`：`:301-318` 返回 `{"pending", "in_flight"}`；因 Rust 侧是回合间同步消费，`in_flight` 恒为 `false`（TS 的后台规划路径未移植，注释 :312-314）。

### 1.3 回合边界消费：真正执行 refinement
- `turn_boundary.rs:411-434` — `consume_pending_refinement()`：`take_refine()` 取出 pending（无论成败都取，避免静默重跑），组装 `RefineOptions{global, instructions, rollback_id:None}`，以 `RefinementSource::SelfRefine` 调 `session.refine(...)`。
- 顺序：先 compaction 后 refinement（`consume_turn_boundary_requests`，:439-457；daemon 侧 `pa-daemon/src/agent_engine/turn/boundary.rs:104`）。
- CLI/headless 也各自消费：`crates/pa-cli/src/print_boundary.rs:172`。

### 1.4 落地执行（core refinement）
`AgentSession::refine` → `refine_with_refiner` → `execute_refinement_with_rows`：

- `crates/pa-core/src/session_engine/compaction_arms.rs:382-464` — `refine()` 用 `default_refiner_call(api_key)` 包一层真正模型 seam；`refine_with_refiner` 持 `compaction_flight` 锁串行化所有 live-context 重放（:415）。
- `crates/pa-core/src/session_engine/refine.rs:301-445` — `execute_refinement_with_rows()` 全流程（下面详解）。

#### execute_refinement_with_rows 关键步骤（refine.rs）
1. **前置校验**：local refinement 且会话无 dir → 报错（:329-336）。
2. **规划态**：读 global state；local 时 merge global+local 作为 planning state（:338-344）。
3. **历史**：global 历史 + 会话内审计历史合并（:345-346）。
4. **baseline 快照**（LLM 调用前捕获）：用于并发检测（:347-359）。
5. **规划** `plan_refinement(...)`（LLM 或 rollback）：(refine.rs:361-369 → executor.rs:194-268)。
6. **剥展示前缀** `strip_display_prefixes`：去掉 `local:`/`global:`（refine.rs:370, 224-237）。
7. **重读**：因 LLM 可能耗时数秒，重新载入 harness 文件（executor.rs:187-189 注释；refine.rs:372-376）。
8. **factory 门在 apply 时刻重读**（:386-394），off 时候拒 create/update。
9. **加锁落盘** `update_harness_state`（LockDir + 原子写，spawn_blocking）：(refine.rs:395-407)。
10. **结果回填 + 历史 + 审计**：`:408-443`：
    - `result.harness_state_path = ...`
    - global scope 时 `append_global_refinement` 追加 JSONL（:409-411）。
    - 会话内 append 三类 custom 行：审计 `prime-agent.refinement`、outcome（给 TUI，display=true）、notice（给模型，display=false，仅当有编辑落地）（:415-443）。
11. outcome/notice 行按 id 推到 live context（compaction_arms.rs:454-462）。

---

## 2. 关键数据结构

- **HarnessState**（mod.rs:84-92）：`schema` + `entries: BTreeMap<Kind, BTreeMap<id, HarnessEntry>>` + `refinements: Vec<HarnessRefinementEvent>`。落盘为 `harness_state.json`（:131-134）。
- **HarnessEntry**（mod.rs:49-69）：id/kind/title/content/path/scope/reference/arguments/metadata/source/created_at/updated_at/version。
- **RefinementResult**（mod.rs:326-339）：id/summary/rationale/expected_outcome/applied_edits/harness_state_path/rollback_of/scope。
- **AppliedRefinementEdit**（mod.rs:344-371）：action/kind/id + 计划字段 + before/after 快照 + applied/error/reason。
- **RefinementEdit**（planner.rs:21-42）：模型返回的未信任编辑（各字段 Option）。

---

## 3. 调用链（file:line）

REPL → 落盘主线：
```
skills/refine/src/refine/__init__.py:50        refine.run → host_request("refine.run")
pa-core/.../turn_boundary.rs:320-369           "refine.run" handler：必须 streaming，合并入 PendingRefine
pa-core/.../turn_boundary.rs:411-434           回合边界 consume_pending_refinement → session.refine
pa-core/.../compaction_arms.rs:382-398         AgentSession::refine → default_refiner_call
pa-core/.../refine.rs:301-445                  execute_refinement_with_rows（规划→重读→门→加锁落盘→审计/历史/行）
  executor.rs:194-268                            plan_refinement：拼 prompt + 调模型 + parse_proposal
    planner.rs:166-172                             parse_proposal（extract_json_object → normalize）
    planner.rs:250-256 / 578-632                   refinement_request：按 context 截断对话、钳 max_tokens
  executor.rs:289-312                            apply_refinement_plan
    planner.rs:310-501                            apply_refinement_proposal：逐条校验+落 entries+记 event
mod.rs:279-318                                  save_harness_state（原子写+fsync）/ update_harness_state（LockDir 加锁 RMW）
mod.rs:429-444                                  append_global_refinement（global JSONL 历史）
refine.rs:415-443                               追加审计/ outcome/notice 三类 custom 行
```

RefinementEvent 记录字段（planner.rs:483-490）：
`id, trigger(=summary), changes(成功编辑 "action kind:id" 列表), evidence(=rationale), outcome(=expected_outcome), created_at`。

---

## 4. 自动触发条件（auto-refine）

触发链路文件：`auto_refine_trigger.rs`、`compact_autorefine.rs`。

### 4.1 触发器
**只在「compaction 成功后」触发**（compact arm）：
- `auto_refine_trigger.rs:50-58` — `mark_compact_auto_refine_pending()`：成功压缩后 arm；`auto_refine_allowed()` 为假直接不 arm（:51-53）。
- daemon 侧在 turn settle 后台任务服务：`pa-daemon/src/compact_autorefine.rs:24-32`（arm）、`:47-97`（消费）；turn settle 点 `pa-daemon/src/agent_engine/turn/run_loop.rs:64` / `worker/turn.rs:1115`。

### 4.2 门控（按序，auto_refine_trigger.rs:132-178）
1. 无 pending 且无 retained review → 返回（:138-140）。
2. **表面门**：`auto_refine_allowed()`（= rlm_depth==0 且有 local harness dir，engine.rs:331）+ `gates.enabled` + `gates.compact`；任一关则清 pending 与 retained review（:143-147）。
3. **冷却** `cooldown_ms`（默认 20 分钟）：冷却内跳过（Checkpoint 保留 pending，Dispose 丢弃）（:148-158）。
4. **in_flight 去重**：重入则 re-arm 不叠加（:161-163）。
5. 已 retained 的 approved review 直接跑（不再评审）（:166-172）。

### 4.3 评审门（LLM review）
- 无 retained 时调 `review_compact_auto_refine`（refine.rs:476-526 → executor.rs:348-387 review_auto_refine）。
- review 返回 `shouldRefine`；若 agent 正在流式（审阅期间 turn 又起）→ `Deferred` 保留审阅等下个边界（auto_refine_trigger.rs:204-215）。
- **分支护栏**：branch_version 变了（用户切分支/丢弃）则丢弃，不落盘（auto_refine_trigger.rs:180-188, 219-231）。
- approved → `run_approved_refine`（refine.rs:564-584），以 `RefinementSource::Auto` 跑 refinement。

### 4.4 gates 默认值（refine.rs:38-64）
`enabled=true, turn_interval=25, compact=true, cooldown_ms=20*60*1000`；turn_interval clamp ≥1。

> **重要发现（约束）**：`turn_interval` 在 Rust 中被解析/钳制但**没有任何消费点用来 arm 触发器**——全仓仅在 `AutoRefineGates` 字段、默认值、clamp、settings 定义处出现。也就是说当前 Rust 版**只有 compact 臂自动触发**，「每 N 回合触发」的 interval 臂尚未接线（TS 侧有）。

---

## 5. 用哪个模型做 refinement 建议

**refinement（plan 与 review）直接用会话模型（session model），不走 auxiliaryModel。**

- compact-trigger review 传入的是会话活模型：`pa-daemon/src/compact_autorefine.rs:67-73`（`self.session_model()`，注释"follows the session's provider like the compaction that armed it (R8)"）。
- `plan_refinement`/`review_auto_refine` 的 `model` 参数一路来自该 session model（executor.rs:194-260, 348-385）。

对比：**auxiliaryModel 只服务 compaction / branch summary 的摘要 pass，不用于 refine**：
- `pa-core/src/session_engine/auxiliary_model.rs:1-4`（文件头明确"compaction and branch summaries"）、`:53-120` `resolve_auxiliary_model`。
- 全仓 refine 路径**无** `resolve_auxiliary_model` / `auxiliary` 引用（grep refine.rs 与 refinement/ 返回 NONE）。

输出预算（planner.rs:14-17）：
- refine 建议：`REFINEMENT_MAX_OUTPUT_TOKENS = 32_000`
- auto review：`AUTO_REFINE_REVIEW_MAX_OUTPUT_TOKENS = 4_096`
- 上下文超限：`refinement_request`（planner.rs:578-632）按「保留尾部、二分截断最老对话、钳 max_tokens」。

---

## 6. suggested edits 如何被应用 / 校验 / 回滚

### 6.1 解析与规范化
- `extract_json_object`（planner.rs:101-128）：直出 / ```fenced / 大括号切片；截断 JSON 单独报 `TRUNCATED_JSON_ERROR`（:19, 55-93）。
- `normalize_refinement_proposal`（:132-159）：容忍缺字段，非法字段保留到 apply 期校验。

### 6.2 校验（planner.rs:193-285 validate_edit）
- action/kind 必填；非 create 必须 id；非 delete 必须 title+content（:216-221）。
- `base_system_prompt` 禁改（:210-215）。
- skill 必须带 python reference（`type==python` + import + callable/call_pattern）与 arguments（:222-256）。
- factory：`arguments` 里 dag 与 machine 不可同时给、必须给其一（:257-283）。
- **factory 门**：非 delete 且 `factory.enabled` 关 → 拒（exact message），门在 apply 时重读（executor.rs:282-295；planner.rs:380-389）。

### 6.3 逐条应用（planner.rs:310-501 apply_refinement_proposal）
- create 已有同名 / update 不存在 / delete 不存在 → 记 error 行，不中断整体（:390-417）。
- **并发检测**：baseline 存在且该 entry 自 baseline 起被改过（非本次 proposal 改的）→ 拒 `"entry changed during refinement planning"`（:357-372）。
- 成功编辑构造新 `HarnessEntry`：版本 +1、updated_at 刷新、source="refine"、未给字段从 before 继承（:418-462）。
- 每条记 before/after 快照（用于回滚），最后 push 一个 `HarnessRefinementEvent`（:471-490）。

### 6.4 落盘与并发安全（mod.rs）
- `save_harness_state`：原子写 + fsync，0o600（mod.rs:274-292）。
- `update_harness_state`：LockDir `{file}.lock` 包住 reload→改→save，10s 重试（mod.rs:294-318）；并发写全部落盘有测试（mod.rs:697-722）。

### 6.5 回滚
- `rollback_proposal`（planner.rs:503-547）：按 applied_edits 逆序，`before` 存在→还原（update/create），只有 after（新建）→删除。`rollback_of` 记录被回滚的 id；scope 由 `infer_refinement_result_scope` 推断（mod.rs:402-421）。
- 入口：`rollback_id`（`/refine rollback <id>`，executor.rs:203-217）。注意：**回滚是「再提案一次」而非事务 undo**，仍走完整校验+落盘。

### 6.6 校验失败是否会回滚整批？
不会。逐条独立：非法条目只在自己那行记 error，其它正常落地（这是设计意图，见 :342-352 等分支）。真正的「整批原子」体现在单次文件写（原子 rename）。

---

## 7. 设计亮点 / 约束

**亮点**
1. **「只调度不打断」**：refine.run 只 arm，LLM 往返在回合边界同步跑，不打断进行中的 turn（turn_boundary.rs:1-5）。
2. **planning/apply 解耦 + 二次重读**：LLM 可能耗时数秒，apply 前重读文件 + baseline 并发检测，防覆盖其他写者（refine.rs:347-372）。
3. **评审门（LLM review）先行**：自动触发不是无脑跑，而是先让一个便宜的 review 判定 `shouldRefine`，且要求「宁缺勿滥」（planner.rs:12, executor.rs:343-386）。
4. **分支护栏**：branch_version 变化即丢弃审阅/结果，防对被放弃分支落盘（auto_refine_trigger.rs 多处）。
5. **门控分层 + 冷却 + in_flight 去重**：防频繁触发与叠加（auto_refine_trigger.rs:141-178）。
6. **factory opt-in 在 apply 时刻判定**而非规划快照，`/factory` 中途翻转即刻生效（refine.rs:377-394 + 测试 :847+）。
7. **before/after 快照 → 可回滚**：AppliedRefinementEdit 保留完整快照支持事后 rollback。
8. **锁+原子写**：LockDir RMW + fsync 原子 rename（mod.rs:274-318）。
9. **prompt 拼装防爆窗**：二分截断最老对话 + 钳输出预算（planner.rs:578-632）。

**约束 / 缺口**
1. **`turn_interval` 未接线**：解析但无消费点，Rust 当前只有 compact 臂自动触发（见 §4.4）。
2. **refinement 不走 auxiliaryModel**：固定用会话模型（§5）。
3. **refine.status 的 in_flight 恒 false**：TS 后台规划路径未移植（turn_boundary.rs:312-314）。
4. **回滚是「再提案」而非事务 undo**：可能与后续编辑叠加时按 baseline 冲突拒绝。
5. **local refinement 需持久化 session dir**，否则报错（refine.rs:329-336）。
6. 单条编辑失败不阻断整批（有意为之），故「部分落地」是常态。

---

## 附：RefinementEvent 字段（planner.rs:483-490）
`id`(refine_<ts>) · `trigger`(=summary) · `changes`(成功编辑列表) · `evidence`(=rationale) · `outcome`(=expected_outcome) · `created_at`。
