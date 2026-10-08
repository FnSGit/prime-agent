# Continual Harness（持续记忆层）调研报告

仓库：/Users/fengshuai/IDE/ai-agent/prime-agent（commit 967eb13fd）
范围：`crates/pa-core/src/refinement/`（状态模型+落盘+refine 执行）、`crates/pa-core/src/session_engine/harness_digest*.rs`（digest 渲染与注入）、`crates/pa-core/src/session_engine/compact_session.rs`（与压缩的关系）、`prime-agent-runtime/src/rlm/harness.py` + `__init__.py`（Python 内核侧 CRUD）。

> 说明：任务里提到的 `pa-types` 中并没有 HarnessEntry/HarnessState——它定义在 `pa-core`（`crates/pa-core/src/refinement/mod.rs`）；`pa-types` 只有压缩行携带的 `harness_digest`/`harness_state_fingerprint` 字段（`crates/pa-types/src/session.rs:128,204`）。

---

## 一、机制流程（先看整体）

两半结构：

1. **存储半（状态）**：harness 条目以 JSON 文件落盘，两个作用域——global（`<agent_dir>/harness/harness_state.json`，跨会话）与 local（`<session artifact dir>/harness/harness_state.json`，会话内）。Rust 与 Python 两套读写同一份文件（Rust 渲染 digest 读；Python 内核写入 `rlm.harness.*`；host `/refine` 也写）。
2. **呈现半（digest）**：会话在「冷边界」（session 启动 / resume / 压缩提交）把 global+local 合并、排序、渲染成一段 `[harness-digest]` 用户消息注入对话；正常轮次不重复注入，靠 fingerprint 判断是否变陈旧。

写入链路：agent 在 REPL 调 `rlm.harness.create_memory(...)`（Python）→ 原子写 `harness_state.json`；或调 `refine.run()` → host 端 LLM 规划一批编辑 → Rust 侧应用 → 同样落盘 + 追加 `refinement_history.jsonl`。

---

## 二、关键数据结构

### 1. 条目（两侧字段一一对应）

`crates/pa-core/src/refinement/mod.rs:41-66`
```rust
pub enum HarnessScope { Local, Global }

pub struct HarnessEntry {
    pub id: String,
    pub kind: RefinementKind,   // prompt|memory|skill|subagent|factory
    pub title: String,
    pub content: String,
    pub path: String,           // 分类：memory=general, prompt=policy …
    pub scope: Option<HarnessScope>,
    pub reference: serde_json::Map<String, Value>,  // 仅 skill：Python 调用契约
    pub arguments: serde_json::Map<String, Value>,
    pub metadata: serde_json::Map<String, Value>,
    pub source: String,         // "agent" | "refine"
    pub created_at: String,     // serde 保持 snake_case
    pub updated_at: String,
    pub version: u64,           // 每次 update +1
}
```
kind 常量：`crates/pa-core/src/refinement/mod.rs:9`
```rust
pub const REFINEMENT_KINDS: [&str; 5] = ["prompt", "memory", "skill", "subagent", "factory"];
```

### 2. 状态文件

`crates/pa-core/src/refinement/mod.rs:85-92`
```rust
pub struct HarnessState {
    pub schema: u64,                       // 当前恒为 1
    // 嵌套 BTreeMap：有序，写盘 key 顺序稳定，避免每次运行 churn 文件
    pub entries: BTreeMap<RefinementKind, BTreeMap<String, HarnessEntry>>,
    pub refinements: Vec<HarnessRefinementEvent>,
}
```
`HarnessRefinementEvent { id, trigger, changes, evidence, outcome, created_at }`：`mod.rs:74-83`。

Python 侧 dataclass 同构：`prime-agent-runtime/src/rlm/harness.py:184-211`（`HarnessEntry` / `RefinementEvent`）。

### 3. 落盘位置与格式

`crates/pa-core/src/refinement/mod.rs:15-17,122-131`
```rust
pub const HARNESS_STATE_DIR_NAME: &str = "harness";
pub const REFINEMENT_HISTORY_FILE_NAME: &str = "refinement_history.jsonl";

pub fn get_global_harness_state_dir(agent_dir: &Path) -> PathBuf { agent_dir.join(HARNESS_STATE_DIR_NAME) }
pub fn get_local_harness_state_dir(session_artifact_dir: Option<&Path>) -> Option<PathBuf> { …dir.join(HARNESS_STATE_DIR_NAME) }
pub fn get_harness_state_path(harness_state_dir: &Path) -> PathBuf { harness_state_dir.join("harness_state.json") }
```

- 格式：`{"schema":1,"entries":{"prompt":{id:entry},…},"refinements":[…]}`，Rust 侧 `serde_json::to_string_pretty` + 换行（`mod.rs:192-207`），Python 侧 `json.dump(indent=2, ensure_ascii=False)`（`harness.py:610-624`）。
- 实际路径示例（本机）：global `~/.prime/agent/harness/harness_state.json`；local `<conversation log>/../../session-artifacts/<session id>/harness/harness_state.json`（推导：`harness_digest.rs:407-419`）。
- 原子写：Rust 走 `settings::storage::atomic_write_with(fsync: true)`（`mod.rs:198-206`）；Python 走 tmp + `os.replace` + 目录 fsync（macOS 上 `F_FULLFSYNC`，`harness.py:626-657`）。新文件 0600。
- 并发：Rust `update_harness_state` 用 `{file}.lock` 目录锁（10s 超时、50 次重试，`mod.rs:210-231`）；Python `_locked()` 同样用 `harness_state.json.lock` 目录 + `owner` 文件 + 死进程回收（`harness.py:473-522`），并用 mtime 检测 host 侧 `/refine` 的外部写入（`_sync_from_disk`，`harness.py:435-445`）。
- 读损坏不抛：`load_harness_state` 解析失败降级为空状态（`mod.rs:172-180`），Python 侧同样当空（`harness.py:529-535`）——理由是 prompt 每轮都要构建，不能因为一个坏文件崩内核。

---

## 三、Python 内核侧 `rlm.harness.*` 如何暴露

`crates/pa-core/src/tools/rlm_bootstrap.rs:70-75` 只负责把 `rlm` 模块绑进内核命名空间（缺包时替一个抛 RuntimeError 的 stub）——**harness 本身不是 bootstrap 注入的代码**，它是 `rlm` 模块的一个属性：

`prime-agent-runtime/src/rlm/__init__.py:621-623`
```python
class _RLMNamespace:
    harness = _harness_state
    get_harness_state = staticmethod(get_harness_state)
```
`_harness_state` 是惰性代理（`__init__.py:510-556`）：
```python
class _HarnessProxy:
    def _resolve(self) -> HarnessState:
        try:    return get_harness_state()
        except RuntimeError as exc:   # 无会话持久化目录（--no-session）
            …HarnessState(in_memory=True, local_write_error=…)   # 读空、写报错
```
要点：**每次访问都按当前环境重新解析**（会话 env 可能晚于 import 应用）；解析永不抛异常，否则内核命名空间里一次异常就会带走内核。

路径解析：`harness.py:166-181`
```python
def _state_file(state_dir=None, *, global_=False) -> Path:
    root = _env_dir("RLM_GLOBAL_HARNESS_STATE_DIR") if global_ else _env_dir("RLM_HARNESS_STATE_DIR")
    if root is None and not global_ and (session_dir := _env_dir("RLM_SESSION_DIR")):
        root = Path(session_dir) / _DEFAULT_HARNESS_DIR_NAME
    …
    return _agent_dir() / _DEFAULT_HARNESS_DIR_NAME / _DEFAULT_FILE_NAME
```
即优先级：显式 `state_dir` > `RLM_HARNESS_STATE_DIR` / `RLM_GLOBAL_HARNESS_STATE_DIR` > `RLM_SESSION_DIR` >（仅 global）agent dir。

CRUD（全在 `HarnessState` 上，`harness.py`）：`create/update/upsert/get/delete/list`（`:660-915`）+ 每 kind 包装 `create_memory/update_memory/delete_memory`（`:914-941`）、`create_prompt_note/…`（`:943-968`）、`create_skill/…`（`:971-1029`）、`create_subagent/…`（`:1032-1058`）、`create_factory/…`（`:1061-1137`）、`record_refinement`（`:1141-1172`）、`plan_refinement`（`:1174-1189`）、`overview`（`:1191-1244`）、`search`（`:1246-1311`，tf-idf 排序）、`snapshot`（`:1313-1325`）、`get_harness_state`（缓存单例，`:1328-1352`）。

`global_=True` 的语义是**转发到 global 存储**而非写副本（`harness.py:602-608` 的 `_global_target`）；id 可带 `[local:id]/[global:id]` 前缀直接路由（`_strip_scope_prefix`，`harness.py:148-157`）。所有写先过 `_validate_entry_shape`（`:311-366`），例如 skill 必须带 `reference.type=="python"` + import + callable（`:217-236`）；factory 的 `arguments` 做 deepcopy 后校验，防止调用方后续改内存里的 spec（`harness.py:756-758`）。

宿主探针在启动时逐个校验这批方法存在：`crates/pa-core/src/kernel/bootstrap/venv/probe.rs:28`（`RUNTIME_READY_CHECK`，包含 `_harness_methods = ['create_memory', …, 'record_refinement']` 与 `assert 'scope' in HarnessEntry.__dataclass_fields__`）。

`refine.run()` 同理：Python skill `skills/refine/src/refine/__init__.py:26-51` 只是 `host_request("refine.run", …)` 的薄封装，真正执行在 Rust（`crates/pa-core/src/session_engine/refine.rs` → `crates/pa-core/src/refinement/{planner,executor}.rs`）。

---

## 四、[harness-digest] 如何渲染并注入

### 渲染（`crates/pa-core/src/session_engine/harness_digest.rs`）

`harness_digest.rs:82-110`
```rust
fn render_digest_with_fingerprint(context:&HarnessDigestContext, query_terms:HarnessQueryTerms)->HarnessDigestRender{
    let global = load_harness_state(&context.global_dir, HarnessScope::Global);
    let local  = context.local_dir.as_ref().map(|d| load_harness_state(d, HarnessScope::Local));
    let merged = merge_harness_states(&global, local.as_ref());
    let digest = format_harness_state_for_prompt(&merged, &HarnessStatePromptOptions{ …query_terms:Some(query_terms), ..});
    HarnessDigestRender{ digest, state_fingerprint: harness_digest_fingerprint(&merged, render_flags) }
}
```

- 正文格式在 `crates/pa-core/src/refinement/ranking.rs:195-370`：标题 `# Continual Harness State` + 若干固定指引段 + 每个 kind 一节 + 尾部 `recent refinements:`。条目行 `ranking.rs:312-322`
```rust
lines.push(format!("- [{}:{}] {} ({}, v{}){}{}: {}",
    scope, entry.id, entry.title, entry.path, entry.version,
    reference_text, arguments_text, compact_harness_text(&entry.content, max_content_length)));
```
  默认上限：每 kind 3 条、内容 140 字符、refinements 10 条（`refinement/mod.rs:19-22`），溢出显示 `+N more … entries`。
- 排序/相关性：`digest_query_terms`（`harness_digest.rs:43-63`）取 goal objective（权重 3.0）+ **最近 4 条** user/assistant 文本（权重 2.0→1.5→1.0→1.0），最多 48 个词；每 kind 内按 `score_harness_entry_for_query`（tf-idf，`ranking.rs:144-175`）降序，溢出时提示 “ranked by relevance … see harness.search”（`ranking.rs:275-282`）。
- 指纹 `harness_digest_fingerprint`（`ranking.rs:408-…`）只覆盖**真正影响渲染的字段**（scope/kind/id/title/path/version/content，skill 额外含 reference/arguments），**不含 query_terms**——注释明说「the digest stays frozen per delivery」，这样冷边界可以用指纹比较而不是文本比较（`ranking.rs:377-378, 403-406`）。

### 注入时机（冷边界）

构造会话时无条件走一次：`crates/pa-core/src/session_engine/mod.rs:253`
```rust
this.ensure_harness_digest_context().await?;
```
`harness_digest.rs:445-458`
```rust
pub(crate) async fn ensure_harness_digest_context(&self) -> anyhow::Result<()> {
    let state = self.agent.state().await;
    let empty = state.messages.is_empty();
    if empty { self.digest_pending.store(true, SeqCst); }   // 空上下文：推迟到第一轮
    else { self.append_stale_harness_digest().await?; }     // resume：直接追加
```
- **推迟的首轮 digest**：`pending_digest_prompt_row`（`harness_digest.rs:460-473`）被两条准入路径消费，作为本轮的 prompt 行排在用户消息之前：
  `crates/pa-core/src/session_engine/admission.rs:140-145`
```rust
let mut prompt_messages = Vec::new();
if let Some(digest_row) = self.pending_digest_prompt_row().await? { prompt_messages.push(digest_row); }
prompt_messages.extend(self.take_next_turn_rows().await);
prompt_messages.push(user_prompt_message(&normalized, &images));
```
（另一处是注入式消息 `admission.rs:43-45`。）
- **陈旧判定**：只有 fingerprint 不匹配才投递（`harness_digest.rs:481-490`）：
```rust
let fresh_matches = match latest { Some(l)=>match l.state_fingerprint.as_deref(){
    Some(fp)=>fp==fresh.state_fingerprint, None=>l.digest==fresh.digest }, None=>false };
(!fresh_matches).then_some(fresh)
```
  fingerprint 缺失（上下文重建把 custom 行转成普通 user 行）时，从 typed session 视图回捞 fingerprint（`harness_digest.rs:182-215`, `harness_digest_inputs` 附近）。
- **替换而非追加**：`append_stale_harness_digest`（`harness_digest.rs:511-540`）先过滤掉所有旧 digest 行 `is_digest_row`，再把摘要行里被新 digest 覆盖的块剥掉 `strip_compaction_digest_block`，最后 `set_messages` + `persist_digest`（写盘、display=false）。字节级匹配保证「用户在对话里引用 digest 原文」不会被误删（注释见 `harness_digest.rs:219-226`）。

### 方向（direction）逻辑

`crates/pa-core/src/session_engine/harness_digest/direction.rs` 整个文件是 `#[cfg(test)] mod direction;`（声明见 `harness_digest.rs:22`），它是**差分测试**：对照 TS 的 `slice(-4).reverse()`，验证窗口是**最新** 4 条而不是最旧 4 条。

`direction.rs:137-153`（base 侧 = 旧的 `truncate(4)` 取最旧）
```python
def base_selection_terms(texts: list[str]) -> HarnessQueryTerms:
    base: list[str] = texts[:]; base.truncate(4); base.reverse()
    return digest_query_terms(None, &base)
```
`direction.rs:157-176` 断言新实现给 `foxtrot=2.0, echo=1.5, delta=1.0, charlie=1.0`，且 `alpha/bravo`（最旧两条）不出现。对应实现侧是 `recent_message_texts_newest_first`（`harness_digest.rs:541-576`，注释明确 “`truncate(4)` would keep the chronological head and rank the wrong end”）。

### digest 内容随能力开关变化

`crates/pa-core/src/session_engine/engine.rs:490-499`
```rust
let digest_context = super::harness_digest::HarnessDigestContext {
    global_dir: crate::refinement::get_global_harness_state_dir(&config.agent_dir),
    local_dir: local_harness_dir,
    include_ipython: active_tool_names.iter().any(|n| n=="ipython"),
    include_shell_examples: active_tool_names.iter().any(|n| n=="bash"),
    include_refine: resources.skills.iter().any(|s| !s.disable_model_invocation && s.name==REFINE_SKILL_NAME),
};
```
即：没有 `ipython` 工具就渲染 shell 版 call contract，都没有就渲染「条目仅作路由提示」版（`ranking.rs:225-231`）。local dir 来源：session artifact dir 或 conversation log 推导（`engine.rs:316-328`）。

---

## 五、与 compaction 的关系

1. **摘要行携带 digest**：压缩提交时渲染一次 digest+指纹存进 `CompactionEntry`（`crates/pa-core/src/session_engine/compact_session.rs:418-439`）
```rust
let (harness_digest, harness_state_fingerprint) = options.harness_digest.as_ref()
    .map(|inputs| { let r = HarnessDigestInputs::render_with_fingerprint(inputs);
                    (Some(r.digest), Some(r.state_fingerprint)) }).unwrap_or_default();
let entry = compaction_entry_for(&result, &attempt.details, options.custom_instructions, harness_digest, harness_state_fingerprint);
```
   类型定义在 `crates/pa-types/src/session.rs:128-129, 204-205`。捕获时机在每次 attempt 内部（`compaction_arms.rs:259-265`，注释：冲突重试要重新取，否则会按被废弃分支的词排序）。
2. **重建上下文时重新拼**：压缩摘要转成 user 消息时，digest 块拼在 `[compaction-summary]` 之前（`crates/pa-core/src/session_engine/messages.rs:352-368`）
```rust
let digest_block = summary.harness_digest.as_deref()
    .map(|d| format!("{HARNESS_DIGEST_PREFIX}{d}{HARNESS_DIGEST_SUFFIX}\n\n")).unwrap_or_default();
… format!("{digest_block}{COMPACTION_SUMMARY_PREFIX}{}{COMPACTION_SUMMARY_SUFFIX}", summary.summary)
```
   常量：`messages.rs:12,16-17`（`[harness-digest]\n\nThe persistent memories produced across this session so far:\n\n<harness_state>\n` … `\n</harness_state>`）——正是本会话顶部那段。
3. **摘要器输入里 digest 行被剔除**：`compact_session.rs:69-72`，`if payload.custom_type == "harness_digest" { return None; }`——记忆不参与「总结对话」，而是由压缩这条独立通道重新注入。
4. **压缩后仍存活的东西**：harness 条目（磁盘文件，天然存活）+ 每次压缩刷新的 digest 快照（`CompactionSummary.harness_digest`）+ refinement history（`refinement_history.jsonl` / session 内）。**不存活**的：旧的 digest 消息行（被 `append_stale_harness_digest` 过滤/剥离），旧 refinement 记录（文件里只增不减，但 digest 只渲染最近 10 条，`ranking.rs:365-368`）。

### 完整调用链（file:line）

```
写入：REPL rlm.harness.create_memory(...)                    harness.py:914
      → _HarnessProxy._resolve → get_harness_state()        __init__.py:525, harness.py:1328
      → HarnessState.create → _locked() → _upsert → save()  harness.py:817/702/610
      → os.replace 原子落盘                                  harness.py:648
写入(refine)：await refine.run()                            skills/refine/src/refine/__init__.py:26
      → host_request("refine.run")                          同上 :51
      → session_engine/refine.rs execute_refinement_with_rows  refine.rs:301
      → refinement/executor.rs plan_refinement / apply_refinement_plan  executor.rs:194,289
      → planner.rs apply_refinement_proposal（version+1）    planner.rs:461
      → update_harness_state（锁 + 原子写）                  refinement/mod.rs:210
      → append_global_refinement（refinement_history.jsonl） mod.rs:326
读取+渲染：engine.rs:490 建 HarnessDigestContext
      → mod.rs:253 ensure_harness_digest_context             harness_digest.rs:445
        ├ 空上下文 → digest_pending → 首轮 pending_digest_prompt_row  harness_digest.rs:460 / admission.rs:141
        └ 非空     → append_stale_harness_digest（指纹比对）    harness_digest.rs:511
      → render_digest_with_fingerprint                       harness_digest.rs:82
        → load_harness_state + merge_harness_states           refinement/mod.rs:170,217
        → format_harness_state_for_prompt + ranking/score     ranking.rs:195,144
        → harness_digest_fingerprint                          ranking.rs:408
压缩：compaction_arms.rs:259 harness_digest_inputs
      → compact_session.rs:420 render_with_fingerprint → compaction_entry_for   compact_session.rs:433
      → messages.rs:352 digest 块 + [compaction-summary] 重建 user 行
```

---

## 六、设计亮点与约束

**亮点**

1. **状态极小 + 每次写入全量校验**：5 个 kind、扁平记录、原子写锁。没有数据库，文件即真相，双语言读写同一 schema。
2. **指纹而非文本比较**判断 digest 陈旧（`ranking.rs:403-406`），避免相关性排序造成的假变更；指纹刻意排除 query_terms，且只覆盖渲染实际读到的字段。
3. **冷边界投递**：正常轮次零开销（只有 resume/压缩后才有一次状态读 + 一次比较）；首轮 digest 走「延迟到第一个 prompt」，不阻塞会话创建。
4. **旧 digest 精确替换**：按 frame 字节匹配区分「引擎的 digest 行」与「用户引用了 digest 原文的消息」，摘要在前缀位置（`strip_compaction_digest_block`）。
5. **fail-safe 读**：损坏文件降级为空而不是抛错（prompt 每轮构建）；本地存储未配置时读空视图但写报错并提示用 `global_=True`（`__init__.py:530-538`）。
6. **作用域是存储位置而非副本**：`global_=True` 直接转发到另一个 `HarnessState`，避免双写不一致。
7. **skill 的 reference/arguments 强校验 + deepcopy**：防止外部改内存里的 factory spec 绕过写时校验（`harness.py:755-758`）。

**约束/坑**

- **单进程假设 + 目录锁**：跨进程靠 `harness_state.json.lock` 目录 + `owner(pid+uuid)` 文件、死进程 10s 后回收；Rust 与 Python 两侧实现独立（Rust `LockDir` / Python `_locked`），任一侧崩溃会短暂阻塞另一侧。
- **全量重写**：每次 upsert 重写整个 JSON；条目多时是 O(n) 写。
- **digest 是有损的**：每 kind 3 条 / 140 字符（可在 `HarnessStatePromptOptions` 覆盖，`ranking.rs:199-207`），细节要靠 `rlm.harness.search`。
- **相关性排序依赖窗口内容**：只看最近 4 条文本 + goal，冷启动（空上下文）时无 query terms，退化为 `path/title/id` 字典序（`ranking.rs:251-262`）。
- **作用域标签只在渲染期确定**：`HarnessEntry.scope` 是 `Option`，`merge_harness_states` 按来源文件打标（`refinement/mod.rs:217-260`）；local id 与 global id 冲突时前缀成 `local:{id}`（`mod.rs:250-258`）。
- **auto-refine 只对 root 会话开放**：`engine.rs:331` 要求 `rlm_depth == 0` 且会话目录存在（子代理不能 refine）。
- **factory 有独立 opt-in 门**：`refinement/mod.rs:139-160` 的 `factory_enabled()` 与 Python `require_factory_enabled` 共用同一条消息文本，任何写路径都做 spec 校验（`_validate_factory_arguments`）。
