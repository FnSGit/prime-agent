# Prime Agent RLM 子代理与多代理协作调研报告

仓库：`/Users/fengshuai/IDE/ai-agent/prime-agent`（Rust，pa-core / pa-daemon + 内嵌 Python runtime `prime-agent-runtime`）
只读调研，未修改任何仓库文件。所有结论后附 `文件:行号` 证据。

---

## 0. 一句话结论

- `rlm.spawn` **只返回"准入门票"**（`RLMSpawnHandle`），不返回任何答案；子代理的任务 prompt 在一个**分离的 tokio 任务**里、且**等父会话当前回合结束后**才投递（`host.rs:155-258`）。因此答案没有返回通道，只能经 `agent_message.send` / 子会话终态通知回到父上下文（`lifecycle.rs:591-606`、`worker/input.rs:282-288`）。
- `rlm.collect` 是**快照 + 共享预算的有界等待**，超时返回当前快照而不是报错（`host.rs:463-537`、`__init__.py:399-433`）。

---

## 1. 机制流程（内核 → 宿主 → daemon）

### 1.1 三层结构

```
ipython 工具（Rust 侧调用 kernel）
   ↓ 执行 Python 单元：await rlm.spawn(...)
prime-agent-runtime/src/rlm/__init__.py   ← Python 门面（严格 snake_case 解析）
   ↓ host_request("rlm.run", {...})  —— repl.py 走 stdio 帧 {event:"host_request", id, data}
pa-core/src/session_engine/rlm_host.rs   ← wire 校验 + handler 注册（pa-core 决定形状）
   ↓ trait RlmSubagentHost（pa-core 定义契约，daemon 提供实现）
pa-daemon/src/rlm_children/host.rs       ← SupervisorChildSessions：真正的 spawn/roster/collect
   ↓ DaemonCommand::{Create, Prompt, GetState, WaitForIdle, Kill, FollowUp}
daemon supervisor → 每个子会话一个 worker 进程
```

Python 侧的请求/回包信封（`prime-agent-runtime/src/rlm/__init__.py:135-160`）：

```python
def _parse_host_reply(request_type, reply):
    status = reply.get("status")
    if status == "ok":    return reply["result"]
    if status == "error": raise RuntimeError(str(reply.get("error") ...))
```

内核里 `rlm` 名字的来源：`rlm = _prime_agent_rlm_module.rlm`，即 `rlm/__init__.py` 末尾导出的 `_RLMFactoryNamespace`（`crates/pa-core/src/tools/rlm_bootstrap.rs:10-18`；同名常量也存在于 `crates/pa-core/src/kernel/bootstrap/runtime_code.rs:36-41`，两处是 TS `buildRlmBootstrapCode` 的 Rust 镜像）。runtime 缺失时注入一个"每次调用都抛错"的桩（`rlm_bootstrap.rs:16-47`），所以错误信息是产品文案而不是 ImportError 泄漏。

### 1.2 spawn 的完整时序

1. Python：`await rlm.spawn(prompt, name=..., model=..., thinking=...)` → `host_request("rlm.run", {prompt, kwargs})`（`__init__.py:170-193`）。
2. pa-core `register_run`：校验 kwargs（只允许 `name/model/thinking`，`rlm_host.rs:489-511`）、取语义锚点 `spawned_by_request_id`（`rlm_host.rs:463-474`）、调用 `bridge.host.spawn(request)`、注册 usage 归因（`rlm_host.rs:475-481`）。
3. daemon `SupervisorChildSessions::spawn`（`rlm_children/host.rs:55-266`）：
   - **深度闸门**：`if identity.rlm_depth >= identity.rlm_max_depth { bail!("RLM recursion depth limit reached ...") }`（`host.rs:59-65`）。默认上限 `DEFAULT_RLM_MAX_DEPTH = 2`（`rlm_children.rs:33`），可运行时改（`rlm_children.rs:686-692`、`rlm_surface.rs:69-95`）。
   - 生成 `child_id = sub-<uuid8>`（`host.rs:66`），未给 name 时由 prompt slug + id 生成默认名（`kernel/rlm_runtime.rs:108-139`）。
   - **同辈名预留**：在第一次 await 之前 `reserve_spawn_name`，冲突直接失败（`host.rs:73-85`，RAII guard `SpawnNameReservationGuard`），随后 `assert_name_available`（`rlm_children.rs:876`）。
   - **模型白名单**：`resolve_child_model_allowlisted`（`host.rs:18-52`）→ `resolve_child_model`（`rlm_child_model.rs:44-54`）先按"父模型相等 → 精确 selector → 唯一短名"解析，再 `model_allowlist::assert_allowed`。**失败大声报错，绝不回退到父模型**，并发 `model refused` 遥测（`host.rs:42-48`；白名单 fail-closed：`model_allowlist.rs:12-60`）。
   - **thinking 校验**：`assert_thinking_supported`（`rlm_child_model.rs:113-142`）；未指定时继承父的 thinking（`host.rs:97`）。
   - **创建子会话**：`create_child` 通过 `DaemonCommand::Create`（`lifecycle.rs:58-147`），配置里带 `rlmDepth`/`rlmMaxDepth`/`provider`/`model`/`parentSessionPath`/`spawnedByRequestId`。
   - **登记 ChildRecord 并返回 handle**（`host.rs:122-152`、`259-264`）。
4. **任务 prompt 分离投递**（关键设计）：

```rust
// host.rs:203-206
let turn_generation = *this.turn_done.subscribe().borrow();
tokio::spawn(async move {
    watcher_this.wait_turn_done(turn_generation).await;   // 等父回合结束
    ...
    watcher_this.prompt_child(&child_active_session_id, &kickoff_content, Some(&kickoff_row)).await
    ...
    watcher_this.watch_child_settle(&watcher_record).await;  // 持续监视直到 settle
});
```

kickoff 内容形如 `"[task from parent]\n\n{prompt}"`，作为一条 `agent_message` custom row 落进子会话（`host.rs:165-196`）。投递失败时**用子会话 JSONL 是否已有该 `spawn:<child_id>` 行来仲裁**是否重试一次（`host.rs:225-239` + `host.rs:652-668`）。

### 1.3 create_session

只允许 depth-0：`if identity.rlm_depth != 0 { bail!("rlm.create_session is available only from a depth-0 session") }`（`host.rs:275-277`）。它不走 children registry，而是 `launch_child` 在共享 sessions 目录里创建一个**常驻 depth-0 会话**（`host.rs:268-330`），并校验返回摘要的深度必须是 0（`host.rs:313-315`）。

### 1.4 子代理的监管：settle watcher

`watch_child_settle`（`lifecycle.rs:428-540`）是每个子代理的后台监管任务：

- 分片 idle 等待 `wait_for_child(..., WATCH_WAIT_SLICE_MS)` → `refresh_record`（`lifecycle.rs:398-424`）；`child_busy` 把 `isStreaming | hasRunningSubagents | queuedCount>0` 视为忙（`lifecycle.rs:341-357`），因此**孙子没停，父 child 也算 running**。
- settle 后有 `WATCH_SETTLE_GRACE_MS` 的**稳定期二次确认**，防止"刚投递就空闲"的误判（`lifecycle.rs:450-462`）。
- 不可达轮询累计到 `WATCH_MAX_UNREACHABLE_POLLS` 判 `error`（`lifecycle.rs:506-537`）。
- settle 顺序被严格固定：先 `record_child_return`（写语义边账本）、再 `emit_child_update`、最后才投递终态通知（`lifecycle.rs:467-499`）。

### 1.5 消息路由（agent_message / agent_observe）

- `agent_message.send` 的 handler 在 pa-core（`agent_messaging.rs:385-527`）：只支持 `target:"all"` 广播或 `receiver_role` + `receiver_name` 定向；**位置参数式 target 被拒绝**（`agent_messaging.rs:415-425`）；父消息不允许带 `receiver_name`（`agent_messaging.rs:459-463`）；角色+名字必须在 family 名册里**恰好命中一个**，0 个报 `No {role} matches ...`，多个报 ambiguous（`agent_messaging.rs:487-515`）。
- family 名册由 daemon 侧 `LinkAgentMessageController::family()` 从 supervisor 推送的 peer roster + 本会话 children registry **join** 得出，顺序为 parent → siblings（按名）→ children（按名）（`agent_messaging/message.rs:67-200`）。关系**只由 durable 父子边判定**，不靠 runtimeKind（`message.rs:123-157`；worker 侧同样 `worker/input.rs:296-305`）。
- 投递路径：**先 worker 间直连票据**，拿不到票据才回落 supervisor `send_message`（`message.rs:240-318`），两者都不重试。
- `agent_observe` 走 supervisor `list {all:true}`（含被动账本子会话）后派生"核心家庭"（`observe.rs:41-64`、`150-170`），`recent_messages` 走 `get_messages` 截尾（`observe.rs:100-144`）。

---

## 2. 关键数据结构

| 结构 | 位置 | 作用 |
|---|---|---|
| `RLMSpawnHandle{rlm_child_id,name,session_dir,model}` | `__init__.py:27-33` / `rlm_host.rs:37-44` | spawn 返回值；**不含答案、不含句柄 future** |
| `RLMChildResult` | `__init__.py:83-97` / `rlm_host.rs:104-125` | collect 的每行：`status ∈ {queued,running,done,error,cancelled}` + `settled` + `answer_preview` + `error` |
| `RLMSubagent` | `__init__.py:58-75` | list_subagents 的行（比 collect 多 activity/progress_note/label） |
| `ChildRecord` | `rlm_children.rs:128-180` | 监管真源：`settled_status`、`settled`、`answer_preview`、`replied_since_task`、`notice_delivered`、`prompt_admitted` |
| `RlmHostBridge` | `rlm_host.rs:324-334` | 会话级共享：model registry、`RlmProgressNotes`、`Arc<dyn RlmSubagentHost>`、usage 归因 |
| `AgentFamilyMember` | `agent_messaging.rs:102-116` | 可寻址成员 + aliases（rlm_child_id / 持久 session id） |

`RlmSubagentHost` 是 pa-core/pa-daemon 的**唯一跨界契约**（`rlm_host.rs:162-175`）；无 daemon 的会话用 `NoRlmChildren`：spawn/create_session 直接报错、roster 为空、collect 每个 target 都 miss（`rlm_host.rs:194-255`）——即"绝不凭空造子代理"。

---

## 3. 三个问题的直接答案

### 3.1 spawn 返回什么

只返回 `RLMSpawnHandle`：子 id、名字、会话目录、解析后的完整模型 selector。返回时机是**登记完成**（`host.rs:122-152`），而任务的 prompt 还没投递（投递在分离任务里等父回合边界）。所以 handle 是"后续 selector + fan-in 落盘路径"，不是"任务句柄"。

### 3.2 为什么答案只能经 agent_message 回来

三条硬证据：

1. **没有返回通道**。`spawn` 在准入完成后立刻返回 handle，同一步里 `tokio::spawn` 出的任务负责 prompt + `watch_child_settle`（`host.rs:204-264`）。宿主不存在任何"等待子答案再回 spawn"的 future；`RLMSpawnHandle` 也不含通道字段。
2. **主动回复会"豁免"终态通知**。子会话在 settle 时若 `replied_since_task == false`，父会话收到一条 `[child-exited: no-reply child:<name>] (+ Last assistant text: …)` 的 follow-up 通知（`lifecycle.rs:591-606`；文案在 `pa-core/src/session_engine/rlm_notices.rs:61-77`）。而 `replied_since_task` 正是 worker 收到该子代理发来的消息时置位的（`worker/input.rs:280-288` → `session_engine_impl.rs:893-901` → `rlm_children.rs:605-616`）。即：**子代理主动 `agent_message.send` 是它"交作业"的正式途径**。
3. **进度旁路不是答案**。`rlm.progress_note` 只写入 pa-core 会话内的 `RlmProgressNotes`（10 秒节流、512 UTF-16 单元，`rlm_host.rs:280-305`、`409-443`），不 steer 父会话、不要求回复。

补充：终态通知本身也是走 `DaemonCommand::FollowUp` 投递给**父会话**的普通消息行（`lifecycle.rs:675-708`），所以从模型视角看，"结果"始终表现为父上下文里的一条消息。

### 3.3 collect 的语义

Python 侧文档与实现（`__init__.py:399-433`）：

```python
timeout_ms bounds the wait for the selected children to settle:
0 returns a non-blocking snapshot immediately; a positive value blocks
only this kernel call until the runs settle or the timeout elapses —
a timeout returns current snapshots, never an error, and the parent
session is never steered.
```

daemon 侧实现（`host.rs:463-537`）：

- `targets` 为空 → 选中全部未删除的直接子代理；否则逐个 `resolve_record`（接受 `rlm_child_id` / `active_session_id` / `session_id` / `session_name`，`rlm_children.rs:223-233`）。
- **共享预算**：`let deadline = Instant::now() + Duration::from_millis(timeout_ms);`，逐个 child 在 `deadline` 剩余时间里 `wait_for_child`（`host.rs:505-526`）——注意是**一个总预算**，不是每 child 一个 timeout。
- 已 settle 的 child 不等待，但若它又变忙（`child_busy`）会重挂 usage 观察（`host.rs:509-517`）。
- 每次 collect 前 `refresh_record`：child 空闲则落 `done` 并**一次性抓取最后助手文本**作为 `answer_preview`（`lifecycle.rs:410-423`，`child_answer` 走 `GetLastAssistantText`，`lifecycle.rs:360-372`）。
- 删除过的 child 保留 tombstone，collect 立刻回一个 `cancelled + settled:true` 的信封（`host.rs:476-501`、`rlm_children.rs:753-766`）。
- `wait_for_child` 用 `WaitForIdle { wait_for_rlm_quiescence: true }`，**超时不算错误**（`lifecycle.rs:374-394`）。

一句话：`collect` = **非阻塞快照 / 可选有界等待 + 永不因超时抛错**；它给的是状态与 answer preview，**不是子代理主动汇报的正文**。

---

## 4. 调用链速查（file:line）

- kernel 绑定 `rlm`：`crates/pa-core/src/tools/rlm_bootstrap.rs:10-18`；镜像常量 `crates/pa-core/src/kernel/bootstrap/runtime_code.rs:36-41`
- Python 门面：`prime-agent-runtime/src/rlm/__init__.py:170-193`（spawn）、`:208-232`（create_session）、`:330-336`（list）、`:399-433`（collect）、`:439-465`（progress_note）、`:468-487`（delete）
- host_request 帧：`prime-agent-runtime/src/rlm/repl.py:173-187`
- handler 注册总表：`crates/pa-core/src/session_engine/rlm_host.rs:356-365`
- spawn handler：`rlm_host.rs:445-485`；kwargs 校验 `:489-511`
- 契约 trait：`rlm_host.rs:162-175`；无子代理宿主 `rlm_host.rs:194-255`
- bridge 组装（daemon 把 children registry 作为 `subagent_host` 传入）：`crates/pa-core/src/session_engine/runtime_wiring.rs:171-179`、`crates/pa-daemon/src/agent_engine/lifecycle.rs:960-962`
- spawn 实现：`crates/pa-daemon/src/rlm_children/host.rs:55-266`（深度 `:59-65`，名预留 `:73-85`，模型白名单 `:89-95`，分离投递 `:203-258`，返回 handle `:259-264`）
- create_session：`host.rs:268-330`
- collect：`host.rs:463-537`
- delete：`host.rs:347-461`
- 监管：`crates/pa-daemon/src/rlm_children/lifecycle.rs:428-540`（watch）、`:398-424`（refresh）、`:591-606`（no-reply 通知）、`:675-708`（通知投递）
- 子会话创建：`lifecycle.rs:58-147`；空闲判定 `:341-357`；答案抓取 `:360-372`；有界等待 `:374-394`
- 名册与投递：`crates/pa-daemon/src/agent_messaging/message.rs:67-200`（family）、`:240-296`（直连）、`:299-318`（回落）
- family 角色判定（durable 边）：`message.rs:123-163`；worker 侧 `crates/pa-daemon/src/worker/input.rs:280-305`
- observe：`crates/pa-daemon/src/agent_messaging/observe.rs:41-64`、`:82-98`、`:150-170`
- 深度默认/调整：`crates/pa-daemon/src/rlm_children.rs:33`、`:686-692`；`crates/pa-daemon/src/rlm_surface.rs:69-95`
- 白名单 fail-closed：`crates/pa-daemon/src/model_allowlist.rs:12-60`
- 子模型解析/思考档校验：`crates/pa-daemon/src/rlm_child_model.rs:44-54`、`:113-142`
- 名字/thinking 归一化：`crates/pa-core/src/kernel/rlm_runtime.rs:48-104`；默认名 `:108-139`

---

## 5. 设计亮点与约束

**亮点**

1. **准入门票与任务执行解耦**：spawn 只等"登记"，任务 prompt 在父回合边界之后才发（`host.rs:203-206`）。避免父子互相阻塞，也避免父模型在自己的回合中途被子代理输出打断。
2. **幂等投递仲裁**：prompt 路由在 worker 替换场景下结果模糊，用子会话 JSONL 里 `details.id == spawn:<child_id>` 的行作为唯一仲裁依据（`host.rs:645-668`）——落盘即事实。
3. **监管者只做快照读取**：`list_subagents` 明确"不能排在长 supervisor 请求后面"（`host.rs:338-343`），worker 刷新由 settle watcher 独占。
4. **白名单 fail-closed + 显式拒绝**：settings 读不出来 = 策略未知 = 全部拒绝，而不是当作无限制（`model_allowlist.rs:12-23`）。
5. **深拷贝的幂等状态机**：`notice_delivered` / `settled` / `prompt_admitted` / `answer_captured` 四个标志把"终态通知恰好一次""settle 后才可被 quiescence 判定""未投递 prompt 的 child 不得 settle""答案只抓一次"这四条不变量编码进记录里（`rlm_children.rs:139-155`）。
6. **可寻址性来自 durable id**：`durable_child_selector`（`rlm_children.rs:120-126`）+ aliases，worker 被 passivation 后父仍能寻址（`message.rs:175-191`）。
7. **上下文边界严格**：`collect` 超时不 steer 父会话（`__init__.py:410-415`）；`progress_note` 不 steer；terminal notice 用 `FollowUp` 排队而非打断。

**约束**

- 递归默认上限 2（`rlm_children.rs:33`），超限直接失败而非静默降级（`host.rs:59-65`）。
- `create_session` 只有 depth-0 可用（`host.rs:275-277`）。
- spawn 的 kwargs 白名单只有 `name/model/thinking`，多余键按排序后的键名报错（`rlm_host.rs:492`、`705-720`）。
- 子代理名必须同辈唯一，且预留在第一次 await 之前完成，否则并行同名 spawn 会双双注册（`host.rs:70-85`）。
- `answer_preview` 是**抓取式快照**（最后助手文本，上限 `ANSWER_PREVIEW_MAX_CHARS = 160`，`rlm_child_model.rs:13`），不是子代理的正式汇报；想要完整结果必须让子代理主动 `agent_message.send`。

**观察到的不一致（值得跟进）**

- `RlmSubagentEntry.progress_note` 与 `replied_since_task` 在 daemon 的 `entry()` / `collect_result()` 里**恒为 `None`**（`rlm_children.rs:728-729`、`746-747`），而 `RlmProgressNotes::latest_note`（`rlm_host.rs:302-304`）除测试外**没有任何调用方**。也就是说"父在 roster 快照里看到子代理 progress note"这条链路目前只有数据结构、没有接线。

---

## 6. 复现路径建议

- 语义对照基线：TS 检出的 `~/prime-agent`（只读），对应符号 `buildRlmBootstrapCode`、`_startRlmChildRun`、`spawnMessage`、`selectAgentFamily`、`createRlmChildTerminalNotice`。
- 相关测试（可作 verifier）：`crates/pa-daemon/src/rlm_children/spawn_name_reservation_tests.rs`、`watch_tests.rs`、`crates/pa-daemon/src/agent_messaging/controller_tests.rs`。
