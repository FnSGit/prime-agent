# Prime Agent 调研报告：skills 系统、工具面、compaction / goal / autonomous（自改进长会话）

> 只读调研，证据全部来自 `/Users/fengshuai/IDE/ai-agent/prime-agent` 当前 `main` 代码。所有引用为 `文件:行号` + 关键片段。

---

## 0. 一句话总览

Prime Agent 的"能力面"分三层：**工具**（`bash`/`edit`/`ipython` 三个 Rust 原生 tool + 内核内程序化工具）、**skill**（markdown 指令 或 Python 内核包，Python skill 通过 `uv pip install --editable` 装进内核 venv 并在 bootstrap 时代码预导入）、**长会话自改进装置**（compaction 压缩 + harness digest 记忆 + compact 触发的 auto-refine + goal/autonomous 循环驱动）。三者在 kernel bootstrap 处汇合。

---

## 1. Skills 系统

### 1.1 机制流程

```
会话启动
 └─ resources::load_resources()               resources/mod.rs:127
     └─ skills::load_skills(LoadSkillsOptions) skills/loader.rs:70
         ├─ (默认目录) user: <agent_dir>/skills、project: <cwd>/.prime/agent/skills   loader.rs:126-127
         └─ (显式路径) CLI/设置里的 skill_paths 逐个解析                     loader.rs:158-223
             └─ 目录 → skills::load_skills_from_dir()          discovery.rs:230
             └─ 单文件 .md → skills::load_skill_from_file()     discovery.rs:68
                 ├─ frontmatter::parse_frontmatter()           frontmatter.rs:36
                 ├─ validate_name / validate_description        mod.rs:158/190
                 └─ detect_python_skill()（仅 SKILL.md）        discovery.rs:35
                     └─ kind = Python|Python, python=SkillPythonMetadata
 → 会话持有 Vec<Skill>
   ├─ 系统提示：format_skills_for_prompt()   mod.rs:209  → <available_skills> 清单
   ├─ 用户输入：expand_skill_command()       mod.rs:262  → <skill name=... > 块
   └─ 内核启动：kernel_python_skills()       runtime_wiring.rs:192 → bootstrap 预导入
```

### 1.2 关键数据结构

**`Skill`（skills/mod.rs:90-99）**

```rust
pub struct Skill {
    pub name: String,
    pub description: String,
    pub file_path: PathBuf,
    pub base_dir: PathBuf,
    pub source_info: SourceInfo,
    pub disable_model_invocation: bool,
    pub kind: SkillKind,          // Markdown | Python
    pub python: Option<SkillPythonMetadata>,
}
```

**`SkillPythonMetadata`（mod.rs:81-85）** — 决定 Python skill 如何被装进内核：

```rust
pub struct SkillPythonMetadata {
    pub import_name: String,     // 目录名里 '-' → '_'
    pub package_path: PathBuf,   // skill 目录（含 pyproject.toml）
    pub pyproject_path: PathBuf,
}
```

**`PythonSkillRuntimeInfo`（mod.rs:129-134）** — 从 `Skill` 抽出、只保留内核需要的四元组；`get_python_skill_runtime_info()`（mod.rs:142）过滤 `is_python()`，**`expect("python skill has metadata")` 不会 fire**，因为 `kind==Python` 只在 metadata 解析成功时才置位（discovery.rs:141-145）。

### 1.3 markdown skill 与 python skill 的区别

| 维度 | Markdown | Python |
| --- | --- | --- |
| 判定 | 默认即 markdown（discovery.rs:141-145） | skill 目录存在 `pyproject.toml` **且** `src/<import_name>/__init__.py` 存在（discovery.rs:40-59） |
| 作用面 | 只进 `<available_skills>` 清单，模型需用 `ipython` 读文件后自行遵循 | 额外被 `uv pip install --editable` 装进内核 venv，bootstrap 时 `importlib.import_module` 预导入，直接 `await goal.get()` 调用 |
| 文件名要求 | 任意小写 `.md`（`skill_markdown_name`，mod.rs:57）；只有 `SKILL.md` 才可能升级为 Python（discovery.rs:121） | 仅 `SKILL.md` |
| 提示里标记 | `<type>markdown</type>`（mod.rs:228） | `<type>python</type>` + `<python_import>` （mod.rs:229-234） |

**Python skill 检测（discovery.rs:35-65）**

```rust
fn detect_python_skill(skill_dir, name, diagnostics) -> Option<SkillPythonMetadata> {
    let pyproject_path = skill_dir.join("pyproject.toml");
    if !pyproject_path.is_file() { return None; }
    let import_name = python_import_name_for_skill(name);      // '-'→'_'
    if !is_valid_python_import_name(&import_name) { …warn…; return None; }
    let package_init_path = skill_dir.join("src").join(&import_name).join("__init__.py");
    if !package_init_path.is_file() { …warn…; return None; }
    Some(SkillPythonMetadata { import_name, package_path: skill_dir.to_path_buf(), pyproject_path })
}
```

注意：目录结构约定是 `skills/<name>/{SKILL.md, pyproject.toml, src/<name>/__init__.py}` — 仓库自带示例 `skills/goal/`、`skills/compact/`、`skills/refine/` 全部是这个形状。

### 1.4 SKILL.md 的解析（frontmatter）

`skills/frontmatter.rs:12-52`：手写的极简 YAML frontmatter 切片（不依赖 yaml crate 的 frontmatter 库），行为对齐 TS 侧 `slice(4, idx)` 语义：

```rust
let Some(end_index) = normalized[3..].find("\n---") else { return (None, normalized); };
let Some(yaml) = normalized.get(4..end_index + 3) else { return (None, normalized); };
if yaml.is_empty() { return (None, normalized); }   // `---\n---` 不是 frontmatter
let body = normalized[end_index + 3 + 4..].trim().to_string();
```

- 换行/BOM 归一化：`normalize_newlines`（frontmatter.rs:5-10）去 BOM、CRLF→LF。
- YAML 解析失败**静默降级**为无 frontmatter（frontmatter.rs:43），不会丢弃 skill 主体。
- 消费字段：`name`（空则回退父目录名，discovery.rs:104-108）、`description`（空则**整条 skill 被丢弃**，discovery.rs:117-119）、`disable-model-invocation`（discovery.rs:137-140）。

**校验规则（mod.rs:158-201）**：name 必须等于父目录名、≤64 字符、只含 `a-z0-9-`、不能首尾连字符、不能 `--`；description 必填、≤1024。违规只产生 `ResourceDiagnostic::Warning`，**不阻止加载**（诊断与加载分离）。

**发现规则（discovery.rs:225-236 注释 + 278-298）**：
1. 目录里若有 `SKILL.md` → 它就是该目录的 skill，**不再向下递归**；
2. 否则根层直接的 `.md` 子文件算 skill（`include_root_files`，仅根目录为 true）；
3. 否则递归子目录继续找 `SKILL.md`；跳过 `.` 开头和 `node_modules`（discovery.rs:311-313）。
4. 递归共享一个 gitignore 匹配器（`.gitignore`/`.ignore`/`.fdignore`，`DiscoveryState`，discovery.rs:153-191），子目录继承父规则、同级也共享子目录发现的规则（discovery.rs:331-339）。

**冲突与顺序（loader.rs:70-124）**：`Vec<Skill>` + `name_winner` map 模拟 TS `Map` 的"插入序 + 先到先得"。`add_skill` 闭包：真实路径去重（canonicalize）→ 同名则记 `ResourceDiagnostic::Collision` 并丢弃后来者 → 否则登记。另外**Python import 名撞车只报 warning**（loader.rs:107-119），因为两个不同 skill 可能都想占用 `websearch` 这个名字。

### 1.5 Python skill 如何被预导入 Python 内核

四步链：

**① skill 列表 → KernelPythonSkill**（`session_engine/runtime_wiring.rs:191-201`）

```rust
pub fn kernel_python_skills(skills: &[Skill]) -> Vec<KernelPythonSkill> {
    get_python_skill_runtime_info(skills)
        .into_iter()
        .map(|info| KernelPythonSkill { name: info.name, import_name: info.import_name,
                                         package_path: info.package_path,
                                         pyproject_path: info.pyproject_path })
        .collect()
}
```

**② 归一化 + 写 bootstrap 版本文件**（`kernel/bootstrap/venv/skills.rs:121-148`）：按 `importName\0packagePath` 去重、递归解析**兄弟目录里的本地依赖**（`resolve_sibling_python_skill_dependency`，skills.rs:150-179：读 sibling `pyproject.toml` 的 `[project] name`，与依赖名匹配则一并安装）、按 `package_path, import_name` 稳定排序。记录带 `pyproject_hash`（sha256，skills.rs:15-20）。

**③ uv 装进内核 venv**（`kernel/bootstrap/venv.rs:166-252` `sync_python_skills`）：与版本文件对比，只装"缺失或 pyproject 哈希变化"的；**一次 `uv pip install --editable A B C` 批量装**，失败则退回逐个安装并逐个 warn 继续：

```rust
let mut install_args = vec!["pip","install","--python",python_str.clone()];
for skill in &missing { install_args.push("--editable"); install_args.push(skill.package_path.clone()); }
if run_async(uv, &install_args).await.is_ok() { …记录… } else { /* 逐个，失败仅 options.report(warn) */ }
```

kernel venv 本身由 `bootstrap_venv`（venv.rs:105-165）用 `uv python install` + `uv venv --seed` + `uv pip install <prime-agent-runtime>` 建在 `~/.prime/agent/kernel-venv`（`venv/layout.rs`）。

**④ bootstrap 代码预导入**（`kernel/bootstrap/runtime_code.rs:78-175`，在 `kernel/provisioner.rs:1038` 生成、`:1071` 交给 `ReplKernelManager`）：

```python
class _PrimeAgentCallableSkillModule(types.ModuleType):
    async def __call__(self, *args, **kwargs):
        result = self.run(*args, **kwargs)
        if inspect.isawaitable(result): return await result
        return result

def _prime_agent_wrap_skill_module(module):
    run = getattr(module, "run", None)
    if not callable(run): return module      # ← 没有 run 就不包装，保持普通模块
    …
    wrapped.__signature__ = inspect.signature(run); wrapped.__doc__ = run.__doc__
    sys.modules[module.__name__] = wrapped
    return wrapped

for _prime_agent_skill_name in ["compact","goal","refine",…]:
    try:
        globals()[name] = _prime_agent_wrap_skill_module(importlib.import_module(name))
    except Exception as e:
        _PRIME_AGENT_SKILL_IMPORT_ERRORS[name] = str(e) or type(e).__name__
        globals()[name] = _PrimeAgentUnavailableSkill(name, …)   # 占位符，调用时抛明确错
```

设计要点：
- 模块若有 `run` → 包成可 `await skill(...)` 的可调用模块（并把 `run` 的签名/doc 提到模块级，模型 `help(skill)` 能看到）；**没有 `run` 就保持原模块**，模型用 `await skill.fn()`。仓库的 goal/compact/refine 都属后者（`skills/goal/src/goal/__init__.py` 导出 `get/create/complete` 三个 async 函数，走 `rlm.host_request`）。
- 导入顺序 = 会话 skill 发现顺序去重（runtime_code.rs:80-88，对齐 TS `[...new Set(...)]`）。
- 导入失败不是静默：打印标记行 `__PRIME_AGENT_PYTHON_SKILL_IMPORT_ERRORS__` + JSON（runtime_code.rs:9-10、164-169），宿主 `parse_unavailable_python_skills()`（runtime_code.rs:19-31）解析后经 `on_unavailable_skills` 回调（provisioner.rs:1188-1201）变成下一轮可见的提示行（`session_engine/engine.rs:377-391` → `skills_unavailable_notice::notice_message`）。**让模型在第一次调用前就知道这个 skill 不可用**，而不是等到调用时看占位符报错。
- venv 未装 `prime-agent-runtime` 时，`rlm`/`bash` 被替换成会抛可执行指引的 `_PrimeAgentMissingRlm`（runtime_code.rs:42-73）。

**⑤ 内核侧 `bash()` 与工具的关系**：bootstrap 把 `rlm`、`bash`、`rlm.mcp` 绑进内核全局（runtime_code.rs:36-41），所以模型在 `ipython` 里写 `await bash('ls')` 即可执行 shell —— 这与模型层的 `bash` tool 是**两条独立通道**（`tools/bash.rs` 是 Rust 侧 spawn）。内核 bash 命令会随 cell 结果回传 `KernelBashCommands` 供 UI 展示（`tools/ipython.rs:59-62`、`kernel/shared.rs:173`）。

---

## 2. 工具面（tools/）与 skill 的关系

### 2.1 三个内建 tool

- `bash`：`create_bash_tool_definition`（tools/bash.rs:492-548），参数 `command`/`timeout`/`allowDestructiveGit`；后者专门用于丢弃型 git（有破坏性命令守卫 `tools/bash_guard.rs:521`，无法安全解析目标仓库时给 `RELOCATION_REFUSAL` 文案，bash.rs:551）。
- `edit`：`create_edit_tool_definition`（tools/edit.rs:307-325），`prepare_edit_arguments` 预处理（把相对路径等参数归一化）。
- `ipython`：`create_ipython_tool_definition`（tools/ipython.rs:490-533），**`execution_mode: Some(ExecutionMode::Sequential)`** —— 内核单线程，不能并发。

三个都登记在回放白名单里：`replay_built_in_tool_name`（tools/tool_definition.rs:108-114）只认 `"bash"|"edit"|"ipython"`。

### 2.2 ipython 工具的注册位置

`session_engine/engine.rs:403-415`：只有当外部没有传 `ipython` tool 时，引擎用 `runtime_wiring::ipython_tool_options(provisioner, …)` 补一个，绑到本会话的 `KernelProvisioner`。之后有 `prewarm` 分支（engine.rs:421-425）：深度 0 且配置开启、或会话有快照时，提前起内核 + 后台 settle MCP。

### 2.3 tool 与 skill 的关系（三条）

1. **skill 是"指令层"，tool 是"执行层"**：markdown skill 只出现在 `<available_skills>` 清单里（`prompts/system_prompt.rs:171-183`，且必须 `has_file_access`（有 ipython 或 bash）且有可见 skill 才注入），提示词明确说"用 ipython 读 skill 文件"（skills/mod.rs:219-221）。
2. **Python skill 是"工具层的可编程扩展"**：它不是 ToolDefinition，不出现在模型的 tool 列表里，而是内核全局名字空间里的模块；模型在 `ipython` 里直接 `await skill.fn()`。清单里用 `<python_import>` 告诉模型这个名字（skills/mod.rs:229-234）。
3. **skill 展开走提交路径，不走 tool 路径**：`/skill:<name>` 在 `admission.rs:86-93` 展开成 `<skill name=… location=…>` 文本块混入用户消息；渲染侧用 `pa_types::skill_blocks::parse_skill_block` 反解（`skill_blocks.rs:26-59`）。展开顺序：**先 skill 展开，再 prompt template 展开**（admission.rs:24-31，TS `_finishSubmissionNormalization` 顺序）；`skill_use_count` 遥测在 admission 处计（admission.rs:100-108，已预展开的块靠 `parse_skill_block` 反查名字补计）。

**`<skill>` 块格式**（skills/mod.rs:276-288）：

```rust
let block = format!(
    "<skill name=\"{}\" location=\"{}\">\nReferences are relative to {}.\n\n{}\n</skill>",
    skill.name, skill.file_path.display(), skill.base_dir.display(), body);
```

`skill_blocks::parse_skill_block` 是**锚定匹配**（整段文本必须"就是一个块"+ 可选 `\n\n` 后的用户文本），且 body 用"第一个 tail 仍能满足匹配的 `\n</skill>`"策略（skill_blocks.rs:32-51 + 测试 `the_body_keeps_close_tags_the_tail_cannot_absorb`）。两边的 opener 字节一致（`BLOCK_OPEN`，skill_blocks.rs:19）。

---

## 3. Compaction 机制（compact_session.rs）

### 3.1 三段式：prepare → summarize → commit

`session_engine/compact_session.rs` 把 `/compact` 拆成三个函数，注释写明**持锁边界**：

- `prepare_attempt`（compact_session.rs:147-221，**在会话锁内**）：读 `retained_entries`、记 `prefix_leaf`、算切点 `prepare_compaction(entries, settings.keep_recent_tokens)`（compact_session/prepare.rs:61）；开启语义边 ledger `semantic_compaction`；切分 `history`（自上次 compaction 保留边界起）与 `turn_prefix_messages`（split turn 的前缀）；算 `tokens_before`；算 `details`（读/改文件清单）。
- `summarize_attempt`（compact_session.rs:223-441，**不持锁**）：split turn 时**两个 summarizer 调用并发**（`tokio::join!` history_call + turn_prefix_call）；turn-prefix 那路**不流式**，等 join 后再把剩余部分 flush 到 live sink，保证"live 块收敛到与提交完全一致的 summary"（compact_session.rs:344-354 注释）；可路由到 `auxiliaryModel`（带 context-window 适配检查，失败回退会话模型）。
- `commit_attempt`（compact_session.rs:453-484，**在会话锁内**）：`compaction_prefix_intact(prefix_leaf)` 结构校验（返回 `Ok(false)` 让调用方**重新 prepare**，不会提交过期切点）；先落 ledger 再 append；`append_compaction` 持久化。

关键不变式（compact_session.rs:460-476 注释）：compaction 行的 parent 是**当前 leaf**，因此 `first_kept_entry` 之后到 compaction 行之间的"中途尾巴"仍然留在保留区。

### 3.2 自动压缩与 skill 的关系

- 阈值判定：`AgentSession::auto_compaction_due`（`compaction_arms.rs:26-47`）用 live loop 消息 vs 模型 context window 比阈值，且"上次 compaction 之前的 usage 不再触发"。
- 触发后 arm auto-refine：`pa-daemon/src/auto_compaction.rs:154` `mark_compact_auto_refine_pending()`。
- **压缩后给模型补一条"内核还在"的隐藏行**：kernel 跨 compaction 存活，命名空间不变，所以追加 `ipython_state` 行告诉模型哪些名字还有效、prune 掉了什么（`ipython_state.rs:1-21`；`CompactionKernelProbe` trait，ipython_state.rs:26-39；`IPYTHON_STATE_CUSTOM_TYPE`）。该 notice 5s 超时降级（`KERNEL_STATE_LISTING_TIMEOUT_MS`），不会把 compaction 卡住。
- **harness digest 记忆随 compaction 行落盘**：`CompactOptions.harness_digest`（compact_session.rs:46-48）在 commit 时渲染，作为 `harnessDigest` 快照存进 compaction durable row，并带 `state_fingerprint`（compact_session.rs:420-432）。即"自改进记忆"是**压进压缩条目的**，压缩后仍可被下一轮读回。

### 3.3 Auto-refine（自改进执行器）

`session_engine/auto_refine_trigger.rs`：compact 成功后 arm，边界处消费。状态机（auto_refine_trigger.rs:24-41）：`pending` / `last_review_at`（冷却）/ `settled_turns_since_review` / `in_flight`（防重叠）/ `branch_version`（**分支切换会让在飞的 review 结果作废**）/ `pending_review`（已批准但等一个空闲 turn）。两个消费点：`CompactAutoRefineSurface::Checkpoint`（轮次间，冷却中则继续 pending）与 `Dispose`（会话销毁，冷却中直接丢弃）。流程恒为 **gates → review → 才真正 refine**（文件头注释 1-3）。没有 refine surface 的会话不 arm（否则永远不执行）。

---

## 4. Goal / Autonomous：长会话自驱

### 4.1 Goal：状态机 + 续跑注入

- 状态与校验：`crates/pa-core/src/goals.rs`。`GoalState`/`GoalStatus` 来自 `pa_types::goal`（goals.rs:13），`normalize_goal_state` 从 status 派生 `active`（goals.rs:86-101）；objective 必填、≤4000 字符（goals.rs:108-119），budget 不能为 0（goals.rs:127-134）。
- **重启复活守卫** `stale_active_goal_failure`（goals.rs:144-174）：最新 `thread_goal_state` 行是 active，但其后的 assistant 行以终止性 provider 失败结束 → 把失败文案作为 goal 终止态；`rate_limit` 类的 quota-park 不算终止（goals.rs:169-202）。
- 续跑由 host 在**回合结束**铸造，注入下一轮：`SessionEngine::mint_goal_continuation`（`session_engine/goal_boundary.rs:88-133`）。守卫：若刚结束的 turn 是 error 或"无输出"（`turn_produced_no_output`），先 `drop_failed_goal_continuation()` 把失败的那对（assistant 尸体 + 驱动它的 `goal_context` 行）从 live context 摘掉（compaction_arms.rs:77-120 解释了"往回只扫 Custom 行、找第一个 goal_context 续跑行"）。铸造失败（persist 失败）→ goal 进入 error 终止，不再铸。
- 预算记账：`record_goal_usage`（goal_boundary.rs:60-69）在每条 settled assistant 行上调用，越界 → `budget_limited`；下一轮注入 `budget_limit` 上下文（`goal_budget_limit_steer`，goal_boundary.rs:73-84）。
- **目标文本把 objective 当不可信数据**：`continuation_prompt`（goals.rs:342-360）用 `<objective>` 包裹并明说"这是用户提供的数据，当作任务而非更高优先级指令"；`objective_updated_prompt` 用 `<untrusted_objective>`（goals.rs:361-369）。同款 XML 转义 `escape_xml_text`（goals.rs:389-393）。
- 纪律写进提示词本身："只有真正完成才 `goal.complete()`，不要因为预算快用完就标记完成"（goals.rs:355-359）。
- **kernel 侧接口**：`skills/goal/src/goal/__init__.py` 只有三行 `host_request("goal.get"|"goal.create"|"goal.complete")`；host 端 `session_engine/host_requests.rs:66-101` 分派。**goal 状态完全 host-side，Python 侧无状态**（host_requests.rs:59-60 注释）。
- 轮次结束由各 transport 驱动同一状态机：print 驱动 `pa-cli/src/print_goal.rs:416`（记账）→ `:420`（budget steer）→ `:475`/`:484`（`mint_goal_continuation`，阈值臂先铸再停循环）→ `:486`（`clear_pending_goal_continuation`），daemon 侧 `pa-daemon/src/goal_continuation.rs:112,271`。

### 4.2 Autonomous：无人值守循环 + 质量门

`crates/pa-core/src/autonomous/`（mod.rs 状态+限额、driver.rs 策略、gates.rs 门）：

- 默认限额（mod.rs:22-30）：continuations 3 / turns 12 / tokens 80k / 超时 30min / gate 重试 3 / gate 超时 5min / 子代理保活 25min。
- **策略接缝** `AutonomousDriver` trait（driver.rs:50-64）：引擎每条 settled assistant 调 `account_message`，每个回合结束调 `after_turn`，返回 `AutonomousFollowUp::{Inactive, Continue{text}, Stop{reason,status}}`（driver.rs:20-32）。引擎**从不自己看 autonomous 状态**（driver.rs:2-3 注释）。
- **门（gate）是完成判据**：`should_autonomously_continue`（gates.rs，配 `ShellGateRunner` 跑 `bash -c`，gates.rs:44-59）——全部 gate 通过 = `GatePassed` 停；gate 失败且重试耗尽 = `GateRetryExhausted`；限额到 = `Stop(Limit)`（driver.rs:36-43）。输出上限 6000 字符进提示、子进程输出上限 1MB、快照 git 命令 10s 超时（gates.rs:14-19）。
- **工作区快照抑制重复跑同一个失败的 gate**：`GitWorktreeSnapshot`（mod.rs:155）+ `capture_snapshot`（gates.rs:37-39，默认 `None` = 总是重跑）。工作区没变就不浪费一轮。
- 续跑文案明确禁止"自己收尾"（mod.rs:20）："不要自己结束会话；配置了门时由 verifier/evaluator 判定完成"。
- `/autonomous` 命令只改 runtime state 并落一条 `autonomous_status` durable 行（`session_commands.rs:398-420`）。

### 4.3 三者如何共同支撑"长会话自改进"

```
turn 结束
 ├─ autonomous.after_turn → 门通过? 停 : 注入续跑文本（限 continuations/turns/tokens/时间）
 ├─ goal 有 active? → mint_goal_continuation（扣一个 slot，注入 <goal: continuation>）
 ├─ 上下文超阈 → auto_compaction_due → compact（锁内 prepare / 锁外双摘要 / 锁内 commit）
 │     ├─ 提交后追加 ipython_state 行（内核命名空间仍有效）
 │     ├─ compaction 行携带 harness digest + state_fingerprint（自改进记忆落盘）
 │     └─ mark_compact_auto_refine_pending → 边界处 gates → review → refine
 └─ 失败回合 → drop_failed_goal_continuation（尸体+续跑行不污染上下文）
```

一句话：**压缩负责"记住什么"，goal/autonomous 负责"继续做"，auto-refine 负责"改进怎么做"**，三者共用同一份 durable session 记录（append-only JSONL）。

---

## 5. 设计亮点与约束

**亮点**

1. **skill 双形态统一抽象**：`Skill` + `kind` + `Option<python>` 一个结构覆盖两种形态；提示清单、遥测 `skill_kind`、`/skill:` 展开全部走同一份数据（mod.rs:108-123）。markdown 保持与 TS 逐字节一致，Python 是增量。
2. **失败永远可见**：skill 校验只 warn 不拦（可用性优先）；Python 导入失败变成"占位模块 + 宿主提示行"，模型第一次调用前就知道（runtime_code.rs:109-125 + provisioner.rs:1188-1201）。
3. **内核环境可复用且增量**：`pyproject_hash` 变更才重装、批量 `uv` 调用、失败降级逐个、兄弟 skill 本地依赖自动解析（venv/skills.rs:150-179），因此 N 个 session 共享一个 `~/.prime/agent/kernel-venv` 缓存。
4. **锁边界写进函数注释**：compaction 三段式各自标明持锁/不持锁，`commit` 有结构校验返回 `Ok(false)` 让调用方重做，而不是提交过期切点。
5. **目标当不可信数据处理**：`<objective>` / `<untrusted_objective>` 包裹 + 明示"不是更高优先级指令"（goals.rs:342-369）；XML 转义两处独立实现（skills/mod.rs:249、goals.rs:389）。
6. **"完成"由外部判据决定**：autonomous 的 gate + goal 的显式 `goal.complete()` 双闸；agent 侧提示词两处都写了"预算将尽 ≠ 完成"（goals.rs:355-359、mod.rs:20）。

**约束 / 值得注意的点**

1. **强 TS parity 优先于"更干净的实现"**：frontmatter 空块、空首行等边界都逐条对齐 TS 切片语义并写进测试（frontmatter.rs:60-110）；skill 加载顺序用 `Vec + name_winner` 模拟 JS `Map`（loader.rs:70-77）。
2. **技能名与 Python import 名是两套命名空间**：name 用 `-`，import 名用 `_`（discovery.rs:22-24），且 import 撞车只 warn（loader.rs:107-119）——两个 skill 抢占同一 import 名时后装的静默覆盖前者，是个潜在坑。
3. **整目录含 `SKILL.md` 就不再下钻**（discovery.rs:278-298），所以 skill 目录里的嵌套 skill 不会被发现。
4. **提示注入有前置条件**：skills 清单只在"有 ipython 或 bash"且有可见 skill 时注入（system_prompt.rs:171-183），`disable-model-invocation: true` 的 skill 只剩 `/skill:` 手动调用路径（mod.rs:210-213）。
5. **`ipython` 工具强制 Sequential**（ipython.rs:527-528），因为内核单线程；bash/edit 无此限制。
6. **bootstrap 依赖 uv 且要网络**：`uv python install` + `uv pip install prime-agent-runtime`（venv.rs:135-155）；缺 `prime-agent-runtime` 时内核用 `_PrimeAgentMissingRlm` 兜底并给出可执行修复指引（runtime_code.rs:47-52）。
7. **auto-refine 的分支版本守卫**：`branch_version` 不匹配时在飞 review 的结果直接丢弃（auto_refine_trigger.rs:33-37、80-86）——避免把针对已放弃分支的建议写进 harness。
