# 31 · Workflow 引擎详解

## 这一章讲什么

这一章回答"workflow 引擎的 6 段流水线每一步干什么、conflict 怎么 merge、verified mode 怎么排队、verifier 怎么打分"。

## 它在整个系统的哪个位置

`evolve_server/engines/workflow.py`（1063 行），`evolve_server/pipeline/{summarizer, aggregation, execution, session_judge, skill_verifier}.py`（共 ~1675 行）。`EvolveServer` 用 `EvolveEngineMixin` + workflow-specific 方法实现 6 段流水线。

## 设计目的

把"session 变 skill"拆成**可独立调 LLM 的 6 段**，让每段都有清晰的 system prompt / 输入 / 输出，运维可以单独调 temperature、verifier 阈值、merge 策略。

---

## 1. `run_once` 顶层流程

**场景**:周期任务(`run_periodic`)或 HTTP `/evolve/run-once` 触发器调起 `run_once`——SkillClaw 的后台"演化循环"在这一刻跑一次。

**为什么这个函数是三段式而不是一段**:SkillClaw 把"演化"拆成**有数据依赖**的三个阶段:

1. **Drain**——把客户端上传的 session 拉进 evolve server 的内存
2. **6 段流水线**——把 session 炼成新 skill(LLM 重活)
3. **收尾**——把新 skill 写回共享存储,通知客户端

如果某一轮没有 session 进来,**第 2 段直接跳过**——但第 1 段和第 3 段**仍跑**(因为 validation job finalize 是收尾阶段的事,可能需要触发)。这就是为什么是"三段式"而不是"if/else 二选一"。

**整体流程图**:

```
run_once()
  │
  ├─→ 第 1 段:Drain
  │     `_drain_sessions()` 拉 sessions/*.json
  │     ├─ 空 sessions? → fast path(跳到第 3 段)
  │     └─ 有 sessions?  → 继续第 2 段
  │
  ├─→ 第 2 段:6 段流水线(sessions 非空时)
  │     1. Drain(同第 1 段,但 sessions 已在内存里)
  │     2. Summarize(LLM 摘要 + 提取 metadata)
  │     3. Session Judge(LLM 给整 session 打 0~1,可选)
  │     4. Aggregate(按 _skills_referenced 分组)
  │     5. Evolve/Create(LLM 决定 improve / optimize_desc / create / skip)
  │     6. Upload(conflict → merge → write skill + versions + manifest)
  │
  └─→ 第 3 段:收尾(无论 sessions 是否有)
        ├─ registry save(更新 evolve_skill_registry.json)
        ├─ session delete(本轮已炼的 session key 从 storage 删)
        ├─ history append(evolve_history.jsonl 加一行)
        └─ notify_proxy_reload(回调客户端拉新 skill)
        + validation finalize(扫 validation_jobs,结果数够了就 publish)

return summary_dict
```

**6 段的"为什么这么排"**:

| # | 段 | 依赖上一步什么 | 这一步产生什么 |
|---|---|---|---|
| 1 | Drain | (起点) | `sessions: list[dict]` + `consumed_keys` |
| 2 | Summarize | sessions | 每个 session 加 `_trajectory` + `_summary` + `_skills_referenced` |
| 3 | Session Judge | sessions + summary | 给每个 session 加 `_session_judge_score` (0~1) |
| 4 | Aggregate | sessions + _skills_referenced | 按 skill 分桶,每桶聚合 `(rollout_count, mean_score, stability)` |
| 5 | Evolve | buckets + 当前 SKILL.md | 每桶调 LLM,返回 `DecisionAction` + 新 SKILL.md |
| 6 | Upload | 新 SKILL.md + conflict 检测 | 写 `skills/<name>/versions/v<N>/` + 更新 `manifest.jsonl` |

**为什么 Stage 3 (Judge) 是 optional**:Session Judge 调 LLM 给"整个 session"打一个 0~1 分,作为信号而非门限——`EVOLVE_USE_SESSION_JUDGE=1` 默认开,但可以关(关掉后 Stage 4 / 5 改用 PRM 单 turn 多数票分)。

**为什么 Stage 7 (validation finalize) 在第 3 段而非第 2 段**:validation 是**异步跨周期**的——本轮 cycle 跑完时,validation_jobs 可能还在等其它客户端投票(配置 `validation.required_user_aliases`)。所以"扫未 finalize 的 job 继续等"必须在收尾段做,与 session 处理**解耦**。

**关键边界**:

- **空 sessions**:跳到第 3 段,但**仍跑 validation finalize**——不能跳过
- **LLM 调失败** (Stage 2 / 3 / 5):单个 session 失败,其它 session 继续;失败的进 `evolve_history.jsonl` 的 `error` 字段
- **Upload 失败** (Stage 6):本 session 失败,下一轮 cycle 重试(从 manifest 看到旧版本)
- **conflict 检测**:manifest 里已有同名 + 更高 `version` → 走 `execute_merge` fallback(详见 §6.2)

**走完之后系统状态**:

- `evolve_skill_registry.json` 更新了本轮所有新 version
- `manifest.jsonl` 多了 N 行(每行一条新 skill bundle_v1)
- `sessions/*.json` 本轮 consumed 的 key 被删(避免下一轮重复炼)
- `evolve_history.jsonl` 多了一行 cycle 记录
- 客户端在 30s 内 poll 到新 manifest,拉新 skill
- validation_jobs 还没 finalize 的继续等

## 2. Stage 1 · Drain（`_drain_sessions`）

**场景**:`run_once` 触发后,evolve server 要从共享存储把**所有未消费的 session** 拉进内存——这是 6 段流水线的"输入"。session 是 `conversations.jsonl` 整段上传的产物(由 SkillClawAPIServer 在 session 关闭时调用 `_upload_session_data` 写进 `{prefix}sessions/<session_id>.json`)。

**为什么这是"列 keys + 读每个"两步而不是"全列后批量读"**:

- **列 keys** 是一次轻量请求(只返路径列表,不含 session body)——可以快速知道"有多少 session 要处理"
- **逐个读 session JSON** 才是重活(每个 session 可能几 MB 完整 messages + tool_calls)——分次读避免单次 OOM

这两步**在概念上是分离的**,但实际在 `_drain_sessions` 里**串行执行**(一次 asyncio event loop 里跑,等所有 read 完才返)。

**怎么走**(基类 `EvolveEngineMixin._drain_sessions`):

1. **列 keys**:
   ```python
   keys = await self._call_storage(
       list_session_keys, self._bucket, self._prefix
   )
   # 列 {prefix}sessions/*.json 所有 key
   ```

2. **遍历 keys,逐个读 session**:
   ```python
   sessions, consumed_keys = [], []
   for key in keys:
       session = await self._call_storage(
           read_json_object, self._bucket, key
       )
       if session:  # 命中(非 None)
           sessions.append(session)
           consumed_keys.append(key)
   return sessions, consumed_keys
   ```

**`_call_storage` 的 local/remote 分支**(为什么要用这个包装函数):

```python
# LocalObjectStore 是同步的(filesystem 够快,直接调)
if isinstance(store, LocalObjectStore):
    return func(bucket, key)  # 同步调

# OSS / S3 是阻塞的,放线程池不阻塞 event loop
return await asyncio.to_thread(func, bucket, key)
```

`local` 走同步(因为 filesystem 调够快,放到线程池反而引入 context switch 开销);`remote` 走 `asyncio.to_thread`(因为 boto3 / oss SDK 是阻塞的,不放线程池会卡死整个 event loop,导致其它并发请求排队)。

**关键边界**:

- **空 sessions**:`if not sessions: logger.info("queue empty")`——返回 `([], [])`,`run_once` 检测到空走 fast path(直接进 stage 7 validation finalize,**跳过 stage 2-6**)
- **单个 session JSON 读失败**:`read_json_object` 返 None,该 key 不 append 到 `consumed_keys`——下次 cycle 还会重读(这可能是"暂时性失败",比如 OSS 暂时不可达)
- **Storage 调用超时**:`_call_storage` 本身有 timeout(配置在 `EvolveServerConfig.storage_timeout_seconds`),超时抛 `StorageTimeoutError`,`run_once` 整段失败,下个 cycle 重试

**走完之后**:

- 内存里 `sessions: list[dict]` 装着所有"未消费"的 session
- `consumed_keys: list[str]` 记着哪些 key 已经在处理(Stage 8 finalize 阶段会从 storage 删掉这些 key,避免下轮重复处理)
- 注意:`consumed_keys` 不在本段删除——Stage 8 收尾时再删。这样设计是**"消费但不立刻删"**——如果 Stage 2-6 任意一步失败,可以回滚(下次 cycle 重读)
- `empty_summary` 路径:空 sessions 时,Stage 1 返回后 `run_once` 走 fast path,`consumed_keys=[]`,Stage 8 也不删任何 key(因为根本没消耗)

## 3. Stage 2 · Summarize（`summarizer.py`）

### 3.1 双重表征

对每个 session 加 3 个字段:

| 字段 | 产生者 | 用途 |
|---|---|---|
| `_trajectory` | 程序构造(**无 LLM**) | 损失极小的 step-by-step 路径(tool_call 顺序、prompt / response 摘录、PRM 分、tool_errors) |
| `_summary` | LLM(system prompt 控制) | 因果链 + 洞察("agent 策略、关键转折点、skill 有效性、结果评估") |
| `_skills_referenced`, `_avg_prm`, `_has_tool_errors` | 程序提取 | 后续 aggregate / judge / evolve 用 |

### 3.2 轨迹构造（`build_session_trajectory`）

`_format_tool_calls(turn)` 把每个 tool_call 拼成 `"    name(args) → ✓ cmd=... | content..."` 或 `"→ ✗ [err_type] msg"`,并强制字符上限:args ≤ 400、results ≤ 400、errors ≤ 300、每步最多 8 个 tool。`build_session_trajectory(session)` 先判断是 flat trajectory(多 rollout 串成时间线)还是 rollout trajectory(每个 rollout 单独一段),然后把 `first_prompt` + `turns` 列表拼成多行文本。

`_PROMPT_MAX = 400` / `_RESPONSE_MAX = 400` / `_TOOL_ARG_MAX = 400` / `_TOOL_RESULT_MAX = 400` / `_TOOL_ERR_MAX = 300` / `_MAX_TOOLS_PER_STEP = 8`:每个值的字符上限(防 LLM context 爆炸)。

`_build_session_payload(session)`:从 turn 提取 skill 引用、PRM 分、tool errors、PRM votes 拼成 `_summary` LLM 看的"压缩证据"。

### 3.3 LLM 调用（`summarize_session`）

**场景**:Stage 2 拿到每个 session 的 `_trajectory`(由 §3.2 程序构造),但**只有程序构造的轨迹不够**——evolve server 还需要"LLM 视角的 session 摘要"(为什么 agent 这么做、关键转折点、skill 有效性)。这一步调 LLM 给每个 session 加 `_summary` 字段。

**为什么是"构 payload + 调 LLM"两步而不是"直接调 LLM"**:

LLM 不能直接看完整 session(messages + tool_calls 可能几 MB)——`build_session_payload` 把 session **压成 LLM 看得懂的形状**:提取 turn 里的 skill 引用、PRM 分、tool errors、PRM votes。这是"压缩证据 + LLM 摘要"的两段式,跟 §3.1 "双重表征"设计一致。

**怎么走**:

```python
async def summarize_session(llm, session):
    # 段 1:构 payload(从 turn 抽 skill 引用 / PRM / tool errors)
    payload = _build_session_payload(session)

    # 段 2:调 LLM
    response = await llm.chat(
        messages=[
            {"role": "system", "content": SUMMARIZER_SYSTEM_PROMPT},  # 8-15 句分析 prompt
            {"role": "user", "content": json.dumps(payload)},
        ],
        max_tokens=4096,
        temperature=0.3,
    )
    return response
```

`SUMMARIZER_SYSTEM_PROMPT` 是 8-15 句固定 prompt——告诉 LLM"你是 session 摘要器,按 4 维分析、给因果链、识别关键转折"。

**`summarize_sessions_parallel` 并行化**:

```python
async def summarize_sessions_parallel(llm, sessions):
    return await asyncio.gather(*[summarize_session(llm, s) for s in sessions])
```

多个 session 的 LLM 调用**并行**——`asyncio.gather` 把所有 call 排进 event loop,LLM 客户端的并发由 `llm` 内部 + event loop 调度。**这是 6 段流水线里最耗时的一段**(每 session 调一次 LLM,几十个 session 几十次调用)。

**`_extract_session_metadata`**(程序从 turn 列表提取的字段):

从 turn 列表提取 9 个程序字段 + `_trajectory`(§3.2 构造):

| 字段 | 含义 | 用途 |
|---|---|---|
| `_avg_prm` | session 内 PRM 多数票平均 | aggregator 排序 |
| `_has_tool_errors` | 是否有 tool error | signal 而非门限 |
| `_skills_referenced` | 调过的 skill 名字 list | aggregator 分桶的 key |
| `_assistant_tools` | assistant 用过的 tool 列表 | judge 评 tool_usage 维度 |
| `_tool_error_count` | 错误 tool 调用次数 | signal |
| `_user_turn_count` | user turn 数(只计 final) | metadata |
| `_first_user_turn` | 首条 user turn text | judge 引用 |
| `_last_user_turn` | 最后 user turn text | judge 引用 |
| `_trajectory` | §3.2 构造的程序轨迹 | 给 LLM 看 |

**注意**:`_summary` 不在 `_extract_session_metadata` 里生成——`_summary` 由 LLM 写(本段)。

**`_SUMMARIZER_DEBUG_DIR`**(debug 开关):

如果设了 `config.debug_dump_dir`,会把 system / user / raw 输出 dump 到 `<debug_dump_dir>/<session_id>_*.txt`——给调试用,生产不开启(避免每个 session 写文件)。

**关键边界**:

- **LLM 调失败**:单个 session 失败,其它 session 继续;失败的进 `evolve_history.jsonl` 的 `error` 字段
- **`_summary` 解析失败**:JSON 解析失败时 fallback 到默认空 summary(`{"causal_chain": "", "insights": []}`)
- **max_tokens=4096**:够 4 维 + rationale;若 LLM 返回截断,Stage 4 / 5 / 6 仍能跑(只是 _summary 字段不完整)
- **temperature=0.3**:比 PRM 的 0.6 低——摘要任务要稳定,不需随机
- **debug_dump_dir 设置时**:性能损耗(每个 session 写 3 个文件)— 生产不开启

**走完之后**:

- 每个 session 多了一个 `_summary` 字段(LLM 写的 4 维分析 + 因果链 + insights)
- `_extract_session_metadata` 提取的 9 个程序字段都在 session 顶层
- Stage 4 (Session Judge) 读 `_trajectory + _summary` 给整体 session 打 0~1 分
- Stage 5 (Evolver) 按 `_skills_referenced` 分桶后,每桶调 LLM 决定 improve / optimize_desc / create / skip

## 4. Stage 3 · Session Judge（`session_judge.py`）

### 4.1 是否需要

`use_session_judge=True`（默认）才跑；`False` 时直接跳过。

### 4.2 输入

每个 session 的 `_trajectory` + `_summary` + 提取的 source artifacts + 提取的 output artifacts + 轻量 metadata。

### 4.3 Prompt

Session Judge 给 LLM 的 system prompt 关键信息:角色是 session-level evaluator;按 4 维 0.0-1.0 打分——`task_completion: 0.55` / `response_quality: 0.30` / `efficiency: 0.05` / `tool_usage: 0.10`;给分规则——0.5 模糊、1.0 明确优秀、0.0 明确失败;trajectory 优先于 summary;**不**假设有 benchmark labels。

返回 JSON:6 个字段——`task_completion` / `response_quality` / `efficiency` / `tool_usage`(4 维 0-1 分) + `overall_score`(加权综合分) + `rationale`(字符串,LLM 给的判断理由)。例:某 session 4 维分别 0.85 / 0.7 / 0.6 / 0.75,综合分 0.76,`rationale` 一句话解释。

### 4.4 跳过条件（`_should_skip_judging`）

- 已有 `aggregate.rollout_count > 0` 且 `mean_score` 存在（benchmark 已经给过分）
- 已有 `_judge_scores.overall_score`（已评过）
- 已有 `aggregate.success_count` / `fail_count`（明确结果）

这些情况直接用现成分。

### 4.5 `_apply_judge_scores(session, scores)`

把 LLM 返回的分写进 `session["_judge_scores"]` = `{task_completion, response_quality, efficiency, tool_usage, overall_score, rationale}`。

### 4.6 `judge_sessions_parallel(llm, sessions)`:`asyncio.gather` 并行多个 LLM 调用。

**注意:常见误读是"`judge_sessions_parallel` 用正则兜底提取,带 3 次重试"——实际是单次 LLM 调用**。`judge_sessions_parallel`(`session_judge.py:397+`)和它调用的 `judge_session`(`session_judge.py:362+`)都是单次 LLM 调用:`judge_session` 调 LLM 一次,失败/解析失败都 `return None`(`session_judge.py:377` / `385-391`),**没有**外层 `for attempt in range(3)` 之类的重试结构。失败/解析失败时 session 不打分,后续靠 PRM 列表回退。

### 4.7 `_apply_judge_scores` 把 PRM 历史回写

`_apply_judge_scores(session, scores)`(`session_judge.py:319-335`)**还会**回写 PRM 历史的 2 个字段:先把 `session.get("_prm_scores")` 拷成 list(从 `_extract_session_metadata` 收集的 turn-level `prm_score`),如果 `previous_prm_scores` 非空就把 `original_prm_scores` 设为原 PRM 列表 + `previous_last_prm_score` 设为最后一项;最后把整个 `judge_scores` 写进 `session["_judge_scores"]`。

**`original_prm_scores`**:judge 评之前所有 turn 的 PRM 列表(原值不动,**不**是 judge 之后的);**只有 `previous_prm_scores` 非空时才会写入**——空 PRM 列表(`[]`)是 falsy,字段不出现。
**`previous_last_prm_score`**:PRM 列表最后一项,同上——只在非空时写入。

**注意:示例输出里 `original_prm_scores` / `previous_last_prm_score` 的可见性**:`_extract_session_metadata`(`summarizer.py:349-382`)把所有非 None 的 turn-level `prm_score` 收进 `_prm_scores`;若 PRM 列表非空,`session_judge.py:328-329` 会写入 `original_prm_scores`,`session_judge.py:330-331` 还会设置 `previous_last_prm_score`——这两个字段在 PRM 列表非空时**一定**出现,不会被吞掉。

## 5. Stage 4 · Aggregate（`aggregation.py`）

**场景**:Stage 3 给每个 session 加了 `_judge_scores`(整 session 0~1 分),现在需要把"每个 session"组织成"每个 skill 一组"——给 Stage 5 (Evolver) 用。Evolver 是按"skill 桶"做决策的,一个 bucket 决定这个 skill 整体怎么改。

**为什么按 `_skills_referenced` 分组而不是按 session 整体**:

Evolver 的设计是"针对每个 skill 看 N 个 session 一起决定怎么改"——不是针对每个 session 决定整个 skill 集怎么改。如果按 session 整体分组,Stage 5 要对每个 session 跑一次 LLM(成本高),且 LLM 看不到"这个 skill 在所有 session 里的累积"。

按 skill 分桶后,Stage 5 对每个 skill 跑一次 LLM(看到该 skill 相关的所有 session),决策更稳。

**怎么走**(`aggregate_sessions_by_skill(sessions)`):

```python
def aggregate_sessions_by_skill(sessions):
    groups = defaultdict(list)
    for session in sessions:
        referenced = session.get("_skills_referenced") or set()
        if not referenced:
            # 没引用任何 skill → 进 __no_skill__ 桶
            groups[NO_SKILL_KEY].append(session)
        else:
            # 引用了 A+B → 同时进 A 和 B 的桶
            for skill_name in referenced:
                groups[skill_name].append(session)
    return dict(groups)
```

**关键边界**:

- **session 引用 A+B** → 同时进 A 桶和 B 桶(同一 session 出现在多个桶里)
- **session 引用 0 个 skill** → 进 `__no_skill__` 桶(`NO_SKILL_KEY = "__no_skill__"` 常量)
- **`_skills_referenced` 是 None 或空 set** → 等同于没引用
- **多客户端改同一 skill** → 同一 skill 的桶里**可能**有不同来源的 session——Stage 6 上传时 conflict detection 兜底

**走完之后**:

- `groups: dict[skill_name, list[session]]`——给 Stage 5 (Evolver) 用
- `groups[NO_SKILL_KEY]` 里的 session 进 Stage 6.3 `create_skill_from_sessions` 路径(创建新 skill)
- 每个 skill_name 桶里的 session 进 Stage 6.1 `_evolve_skill_group` 路径(改写老 skill)

---

## 6. Stage 5 · Evolve/Create（`execution.py`）

### 6.1 `_evolve_skill_group(skill_name, sessions, existing_skill_names)`

**场景**:Aggregate 后,Stage 5 拿到 `(skill_name, sessions_for_this_skill, ...)` 桶——给这个 skill 一组 session,LLM 决定"怎么改这个 skill"。

**为什么是"取老 skill → 调 LLM evolve → materialize"三步而不是 1 步**:

- **取老 skill** 是前置依赖——LLM 改写时要知道"现在这个 skill 长啥样",不然改完跟现状不一致
- **调 LLM evolve** 是核心决策——LLM 看完老 skill + 一组 session,返回 `DecisionAction` + 新 SKILL.md
- **materialize** 是落盘——把 LLM 的决策写到存储,继承老 frontmatter(避免 LLM 改掉不该改的字段)

**怎么走**:

```python
async def _evolve_skill_group(self, skill_name, sessions, existing_skill_names):
    # 段 1:取老 skill
    current_skill = await self._fetch_or_init_current_skill(skill_name)
    # 没老 skill → init 一份空 SKILL.md

    # 段 2:调 LLM evolve
    result = await evolve_skill_from_sessions(
        self._llm, skill_name, sessions, current_skill, existing_skill_names
    )
    if result is None or result.action == DecisionAction.SKIP:
        return None  # LLM 决定不动 / 解析失败 → 不 materialize

    # 段 3:materialize 到存储
    return await self._materialize_skill(
        result.skill, result.action,
        ..., current_skill=current_skill
    )
    # OPTIMIZE_DESC 保留 body;其他动作 inherit 老 frontmatter
```

**`DecisionAction` 4 个动作**(`evolve_server/core/constants.py:35-41`):

| 字面值 | 含义 | 用途 |
|---|---|---|
| `CREATE = "create_skill"` | 从 no-skill bucket 造新 skill | Stage 6.3 路径 |
| `IMPROVE = "improve_skill"` | 老 skill 改写 | Stage 6.1 路径(本段) |
| `OPTIMIZE_DESC = "optimize_description"` | 只改 description,body 继承老的 | Stage 6.1 路径(本段) |
| `SKIP = "skip"` | LLM 决定不动 | 早返,不 materialize |

**默认行为**(三个"为什么这样定"):

- **SKIP 是 default**:`_parse_evolve_result` 的 action 默认值是 `DecisionAction.SKIP`——LLM 不返回 action 字段时,默认**跳过**该 skill。这避免"LLM 偶尔不返回 action"导致误改老 skill。
- **CREATE 路径的兜底降级**:`_parse_evolve_result`(`execution.py:456-465`)有兜底——LLM 偶发返回 `create_skill` 但 `name == skill_name`(等于改名没改)时,会被**静默改写**为 `IMPROVE`,避免同名 create 改写历史。
- **`_materialize_skill` 强制 name 行为**:`workflow.py:760-764` 在 `_materialize_skill` 里——`action_type == DecisionAction.IMPROVE` 时强制 overwrite skill name 为 `current_skill['name']`,LLM 想 rename 也会被覆盖;`CREATE` 路径才走 `_sanitise_name`(让 LLM 决定新 skill name)。

**`_inherit_current_skill`**:老 skill 的 body / category / extra_frontmatter 灌到新 skill——按 `overwrite_body` 决定 body 是否覆盖。

**关键边界**:

- **LLM 返回 SKIP**:早返 None,Stage 7 finalize 不写这条 skill
- **LLM 返回 OPTIMIZE_DESC**:`_materialize_skill` 保留老 body,只更新 description 字段
- **LLM 返回 IMPROVE**:`_materialize_skill` 强制 name 等于 current_skill.name(防止 LLM 改 name)
- **LLM 返回 CREATE 但 name 与 skill_name 相同**:兜底降级为 IMPROVE(避免同名 create 改写历史)
- **`current_skill` 拉取失败**(OSS 不可达):`_fetch_or_init_current_skill` 返回空 SKILL.md,LLM 当全新 skill 改(没老 skill 上下文)
- **`sessions` 空**:Aggregate 已经保证每个桶至少 1 session;空桶是 no_skill path,不走本段

**走完之后**:

- 老 skill 文件被新 SKILL.md 覆盖(在 `versions/v<N>/` 下)
- `evolve_skill_registry.json` 更新这条 skill 的 `version + content_sha`
- `manifest.jsonl` 多一行 `bundle_v1` 记录
- `evolve_history.jsonl` 多一行 cycle 记录

### 6.2 `evolve_skill_from_sessions(llm, skill_name, sessions, current_skill, existing_skill_names)`

**场景**:`_evolve_skill_group` Step 2 调 LLM 拿决策——这个函数是"调 LLM"的具体实现,把"老 skill + 一组 session"压成 LLM prompt,LLM 返回 `DecisionAction` + 新 SKILL.md。

**为什么是"构 prompt → 调 LLM → 解析"三步而不是 1 步**:

- **构 prompt** 是关键——LLM 看不到完整 session(几 MB),要把老 skill + session 摘要 + 现有 skill 名压成 LLM 看得懂的形状
- **调 LLM** 是异步阻塞(0.5-3s),要 `await` 不能同步
- **解析** 是反 LLM 输出"反规范化"——LLM 经常返回 markdown + JSON 混合体,要把 JSON 抽出来 + 校验 + 兜底

**怎么走**:

```python
async def evolve_skill_from_sessions(llm, skill_name, sessions, current_skill, existing_skill_names):
    # 段 1:构 prompt
    system = _EVOLVE_FROM_SESSIONS_SYSTEM.replace("{skill_name}", skill_name)
    user = "".join([
        _build_skill_block(current_skill),  # 老 skill 的 markdown 块,无则空串
        "## Session evidence (N sessions)\n" + _build_session_evidence(sessions),
        # 最多 30 sessions,超过加 "... and X more"
        "## Existing skill names\n" + (", ".join(existing_skill_names) or "(none)"),
    ])

    # 段 2:调 LLM
    raw = await llm.chat(
        messages=[{"role": "system", "content": system}, {"role": "user", "content": user}],
        max_tokens=8192,
        temperature=0.4,
    )

    # 段 3:解析
    return _parse_evolve_result(raw, skill_name)
```

**`_build_session_evidence(sessions, max_sessions=30)`** 把 sessions 列表转成"### Session {id} ({prm}, {aggregate}, {errors}, {skills})\n**Trajectory**:\n...\n**Analysis**:\n..." 文本块——**最多 30 个 session**,超过加 "... and X more sessions"避免 prompt 爆。

**`_parse_evolve_result(raw, skill_name)`** 解析 LLM 输出:

| 字段 | 类型 | 说明 |
|---|---|---|
| `action` | 字面值 | 4 个 DecisionAction 之一;无则默认 SKIP |
| `skill` | dict | 4 字段:`name` / `description` / `category` / `content` |
| `rationale` | str | LLM 给的判断理由 |

解析步骤:
- 正则剥 ```` ```json ```` fences
- `find("{") + rfind("}")` 抽 JSON 块(LLM 经常在 JSON 前后有 prose)
- 校验:action 是 SKIP 时只要 rationale;其它要有 `skill` 字段(dict)
- `create_skill` 时必须 name
- `improve_skill` 时如果没 name,**用 `skill_name` 当 name**(不让 LLM 改老 skill 的 name)

**关键边界**:

- **LLM 返回 prose 不含 JSON**:`find("{")` 返 -1 → 解析失败 → `return None` → Stage 6.1 早返
- **LLM 返回非 4 个合法 action**:`_parse_evolve_result` 兜底为 SKIP(避免 5 个 action 之外的乱码)
- **`_build_session_evidence` 超过 30 sessions**:加 "... and X more" 省略(避免 prompt 爆);LLM 不知道被省略的部分
- **`improve_skill` 无 name**:用 `skill_name` 兜底(防止 LLM 改老 skill name 引起 conflict)
- **CREATE + name 撞老 skill**:`_parse_evolve_result` 静默改写为 IMPROVE(兜底降级)
- **`max_tokens=8192` 偏大**:LLM 偶尔 verbose 时截断风险要权衡;实际生成 SKILL.md 通常 2-4K tokens

**走完之后**:

- 返回 `DecisionResult` 对象(包含 `action` + `skill` 字典 + `rationale`)
- Stage 6.1 Step 2 拿这个 result 进 Step 3 `_materialize_skill`
- 如果解析失败:`return None` → Stage 6.1 早返 → Stage 7 finalize 不写这条 skill

### 6.3 `create_skill_from_sessions(llm, sessions, existing_skill_names)`

同 `_evolve_skill_group` 但**没** `current_skill`——给 LLM 一堆 no-skill session + 现有 skill 名列表，让 LLM 决定：
- 是不是新 skill（还是已经存在的 skill 的 no-skill case）
- 如果是新的，name / description / content

### 6.4 `_handle_no_skill_sessions(sessions, existing_skill_names)`

调 `create_skill_from_sessions` → 拿 result → 调 `_materialize_skill(..., action_type=DecisionAction.CREATE, source="no_skill", current_skill=None)`。

## 7. Stage 6 · Upload（`_materialize_skill` + 冲突检测）

### §X. 后端选型:OSS vs Nacos(为 §7.2 / §7.3 / §7.4 铺垫)

SkillClaw 支持两种 skill 资产后端,workflow 引擎在同一 cycle 内**只走一条**:

- **OSS / S3 / Local 路径**:`sharing_skill_client` 为空,`storage_backend ∈ {local, s3, oss}`——单次 upload 是"写 SKILL.md + files → append manifest.jsonl + record_update registry"原子序列;走 `SkillIDRegistry` 维护 skill_id / version 演进。
- **Nacos 路径**:`sharing_skill_client` 非空(`nacos_skill_hub.NacosSkillClient` 实例)——多了"草稿 / 审核 / 标签"状态机:`publish_mode ∈ {draft, review, direct}`,label-based 版本路由(`labels: {name: version}`);**不**写 `manifest.jsonl` / `evolve_skill_registry.json`,这两份文件对 Nacos 后端透明。

`NacosSkillClient` **不**实现 `ObjectStore` 接口——它走 HTTP REST(Nacos 配置中心)而不是 OSS 兼容协议。这是为什么 `SkillHub` 要有专门的 `NacosSkillHub` 分支。完整对比见 [20 · 共享存储与同步 §2.2](../02-客户端共享层/20-共享存储与同步.md) / [33 · 共享层与存储适配 §X](../03-Evolve服务/33-共享层与存储适配.md)。

### 7.1 `_materialize_skill` 决策树

**场景**:Stage 5 拿到 LLM 的 `DecisionResult`(action + skill 字典),现在要把"决策"落进存储——不同 action 不同走法,不同 publish_mode 不同走法,verifier 启用/禁用又不同走法。`_materialize_skill` 是"决策 → 落盘"的中央调度。

**为什么是 4 步而不是 1-2 步**:

- 步骤 1 规范化名字(LLM 给出怪字符)
- 步骤 2 可选 verifier(默认关)
- 步骤 3 分配 skill_id(OSS 路径才需要,Nacos 路径跳过)
- 步骤 4 按 publish_mode 分流(`validated` 走 queue,其它走 upload)

**怎么走**:

```python
async def _materialize_skill(self, evolved_skill, action_type, ..., current_skill):
    # 段 1:名字规范化
    evolved_skill["name"] = self._sanitise_name(evolved_skill["name"])
    # 防 LLM 给出 "my-skill v2" / "react (advanced)" 这类怪字符

    # 段 2:可选 verifier
    if self.config.use_skill_verifier:
        verdict = await self._run_skill_verifier(...)
        if not verdict.get("accepted"):
            return {"action": "verification_rejected", "verification": verdict}

    # 段 3:分配 skill_id(OSS 路径)
    if not self._sharing_skill_client:  # OSS / S3 / Local 路径
        skill_id, version = self._id_registry.get_or_create(evolved_skill["name"])

    # 段 4:按 publish_mode 分流
    if self.config.publish_mode == "validated":
        return await self._queue_validation_job(evolved_skill, action_type, ...)
    else:
        return await self._resolve_and_upload(evolved_skill, action_type, ...)
```

**6 个出口**(返给 Stage 8 finalize 的状态字符串):

| 出口 | 触发条件 | 后续 |
|---|---|---|
| `verification_rejected` | verifier enabled + accept=False | 不写存储,记 history |
| `queued_for_validation` | publish_mode=validated | 写 validation job,等其它客户端投票 |
| `uploaded` | 直接上传成功 | 写存储 + manifest + history |
| `*_pending_review` | Nacos `review` 模式 | 已 upload,等人工 review |
| `*_pending_publish` | Nacos `direct` 模式 + `_wait_nacos_publish` 30s 超时 | uploaded 但 latest label 未改 |
| `*_draft` | Nacos `draft` 模式 | 只上传、不进审核流 |

### 7.2 `_run_skill_verifier`（`skill_verifier.py`）

**场景**:SkillClaw 默认 `EVOLVE_USE_SKILL_VERIFIER=0`(关 verifier),开启后给新生成的 skill 加一道"快速预筛"——避免 LLM 写出来的"看起来对但实际不能用"的 skill 被上传。

**设计目标(必读)**:`verify_skill_candidate` 用**同一个 LLM**(跟生成 skill 的 evolve LLM 共享 `AsyncLLMClient`)做"价格低 + 一致性高 + 部署简化"的**快速预筛**——**不是**独立审计;要独立审计可加 rule-based 闸门或换 LLM。verifier LLM 跟 generator LLM 同一份 = "自己审自己",无法识别同一 LLM 的系统性偏见。

**怎么走**(`_run_skill_verifier`):

```python
async def _run_skill_verifier(self, skill, sessions, action_type, current_skill):
    # 段 1:开关检查
    if not self.config.use_skill_verifier:
        return {"enabled": False}

    # 段 2:调底层 verifier
    return await verify_skill_candidate(
        self._llm, skill, sessions, action_type,
        current_skill=current_skill,
        min_score=self.config.skill_verifier_min_score,  # 默认 0.75
    )
```

**`verify_skill_candidate` 内部**:

1. 构造 payload(candidate_skill + current_skill + session_evidence + acceptance_threshold)
2. 调 LLM(`max_tokens=2000, temperature=0.1`)
3. `_extract_json_object` 剥 ```json fences + 抽 JSON
4. 解析 4 个 check 维度的 0-1 分(赋给 `checks` 字典)
5. LLM 返回 `score` 优先用 LLM 给的;没给就用 4 个 check 的均分(`_compute_score`)
6. 三段 if/elif 判定 accepted:
   - `decision_raw == "accept"` → `accepted = True`
   - 否则 `score < min_score` → `accepted = False`(唯一硬性拒绝)
   - 否则 `decision_raw not in {"accept", "reject"}` → 兜底 `accepted = score >= min_score`

**拒绝条件(精确)**:**只**按 `score < min_score` 拒绝。`checks` 字段(4 维 0-1 分)是辅助输出(展示给运维看),**不**参与 gating——这是当前实现,**`per-check` 阈值是已知未实现项**(见 §11)。

**LLM prompt 4 维评价**:
- `grounded_in_evidence` — 是否有 session 证据支持
- `preserves_existing_value` — 是否丢了老 skill 的有用 facts
- `specificity_and_reusability` — 是否具体可复用(不是泛泛 agent advice)
- `safe_to_publish` — 是否会破坏 agent 行为

**关键边界**:

- **`use_skill_verifier=False`**:段 1 早返,不调 LLM
- **`score < min_score`**:拒,返 `verification_rejected`
- **`decision_raw="reject"`** 但 `score >= min_score`:**接受**——LLM 显式说拒但分够,走分(`score < min_score` 是唯一硬性拒绝)
- **`decision_raw` 非法** (e.g. "maybe"):兜底按 `score >= min_score` 决定
- **`checks` 单维度 < 0.5`**:不拒(`per-check` 阈值未实现,见 §11)
- **`score=None`** (LLM 返回无 score 字段 + check 也不全):accepted = False(显式拒)

**走完之后**:

- verifier 接受 → 进 Step 3 分配 skill_id
- verifier 拒绝 → 返 `verification_rejected` 状态字符串,Stage 8 finalize 不写这条 skill
- `verification: verdict` 进 history

### 7.3 `_resolve_and_upload`（conflict → merge → upload）

**场景**:Stage 5 / 6.1 拿到新 SKILL.md,现在要上传到存储——但可能有 conflict(其它客户端/evolve server 同一 cycle 也改了同一个 skill)。`_resolve_and_upload` 是"冲突检测 → 解决冲突 → 上传"的中央调度。

**为什么需要这一步而不是直接上传**:

- 多客户端 / 多 evolve server 并发改同一 skill 是常见场景(尤其在团队部署)
- 直接上传会**覆盖**别人刚改的版本——丢数据
- 需要先检测冲突,有冲突就调 LLM 合并(merge),没冲突就直接上传

**怎么走**:

```python
async def _resolve_and_upload(self, skill, action_type, current_skill):
    # 段 1:检测冲突
    has_conflict = self._detect_conflict(skill, current_skill)

    # 段 2:没冲突时直接上传
    if not has_conflict:
        upload_status = await self._upload_skill(skill, action_type)

    # 段 3:有冲突时调 LLM 合并
    else:
        try:
            merged_skill = await execute_merge(self._llm, current_skill, skill)
            upload_status = await self._upload_skill(merged_skill, action="merge")
        except Exception:
            # 合并失败 fallback 用 incoming
            upload_status = await self._upload_skill(skill, action_type)

    # 段 4:把上传状态翻译成 materialize 出口
    return self._upload_status_to_action(action_type, upload_status)
```

**`_detect_conflict`**(OSS 路径 vs Nacos 路径):

| 后端 | SHA 来源 | 性能 |
|---|---|---|
| **OSS / S3 / Local** | `self._id_registry.get_content_sha(skill_name) or ""` | 快(本地) |
| **Nacos** | `_fetch_skill` 拉当前 SKILL.md + 重算 SHA | 慢(每次走网络) |

**为什么 Nacos 路径不同**:Nacos 模式没有 `evolve_skill_registry.json`(`SkillIDRegistry` 不持久化),所以 SHA 是从远端 SKILL.md 实时算的——Nacos 模式下 conflict detection 每次都走一次网络,会比 OSS 模式慢。

**判断逻辑**:
```python
existing_sha = ... # 上面的 if/else 拿
incoming_sha = sha256(build_skill_md(incoming_skill).encode())
return existing_sha and existing_sha != incoming_sha
# 空 SHA 视为"没冲突"——首次 push 的 case
```

**`execute_merge(llm, existing_skill, incoming_skill)`**:把两个 SKILL.md 喂给 LLM,让 LLM 合并(用 `_MERGE_SKILL_SYSTEM` prompt)。返回的 merged skill 再走 `_upload_skill`(`action="merge"`)。

**关键边界**:

- **空 SHA** (首次 push):`return False`(无冲突)
- **相同 SHA** (push 一模一样的版本):`return False`(无冲突)
- **不同 SHA** (并发改):`return True`(有冲突,走 merge)
- **merge 调 LLM 失败** (`execute_merge` 抛异常):fallback 用 incoming skill 上传(`action_type` 不变)——保证 cycle 不中断
- **Nacos `_fetch_skill` 超时**:抛异常,fallback 抛错,Stage 7 finalize 失败但 cycle 继续

**走完之后**:

- 状态字符串 `uploaded` / `*_pending_*` / `*_draft` 返给 Stage 8 finalize
- 写入 `manifest.jsonl` + `evolve_skill_registry.json`(OSS 路径)
- Nacos 路径不写这两份文件(只走 Nacos 状态机)

### 7.4 `_upload_skill`(Nacos / OSS / S3 三路径)

**场景**:`_resolve_and_upload` 决定要上传(无冲突或 merge 完),现在要把"SKILL.md + files"实际写进存储。SkillClaw 支持两种后端,走法不同。

**OSS / S3 / Local 路径 5 步**:

```python
async def _upload_skill(self, skill, action_type, files=None):
    # 段 1:拿 skill_id
    skill_id, version = self._id_registry.get_or_create(skill["name"])

    # 段 2:写 SKILL.md
    skill_md = build_skill_md(skill)  # 把字典序列化成 markdown
    await self._call_storage(put_object, self._bucket, f"{skill['name']}/SKILL.md", skill_md)

    # 段 3:写 registry
    content_sha = sha256(skill_md.encode())
    bundle_record = build_bundle_record(skill, content_sha, files)
    self._id_registry.record_update(skill["name"], content_sha, action_type, bundle_record)

    # 段 4:写 version dir
    await self._save_version_bundle(skill, version, files, bundle_record)

    # 段 5:更新 manifest
    manifest_entry = build_manifest_entry(skill, skill_id, version, content_sha, bundle_record)
    self._manifest[skill["name"]] = manifest_entry
    await self._save_manifest()  # 落盘

    return "uploaded"
```

**Nacos 路径 8 步**(更复杂,因状态机):

1. 读 remote record + detail(`_nacos_get_record_and_detail`)
2. `_nacos_working_version(record, detail)` 找 working version(有但没 published)
3. 如果有 working version:拉下来 `_bundle_matches_remote` 比——相同就 skip;不同就用 working version 号覆盖
4. 否则 `_next_version(record, detail)` 算新版本号
5. 打包 zip + upload
6. `nacos_publish_mode in {"review", "direct"}` → `submit(name, version)` 进审核
7. `nacos_publish_mode == "direct"` → `_wait_nacos_publish(name, version, timeout=30.0)` 轮询等 `labels.latest == version`
8. 返回 `uploaded` / `uploaded_draft` / `uploaded_pending_review` / `uploaded_pending_publish` / `skipped_existing_<status>`

**`_wait_nacos_publish` 30s 超时**:Nacos `direct` 模式下,upload 后要等 Nacos 服务端把 `labels.latest` 改到新 version——30s 超时是兜底(网络慢/服务端卡)。超时后返 `uploaded_pending_publish`(语义上算成功,只是 label 未即时改)。

**关键边界**:

- **OSS 路径**:`skill_id, version` 走 `SkillIDRegistry.get_or_create`——新 skill 返 `(new_id, 1)`,老 skill 返 `(existing_id, next_version)`
- **Nacos 路径**:不走 registry,version 完全由 Nacos 服务端分配
- **`content_sha` 计算**:对 `build_skill_md(skill).encode()` 算 sha256——任何字段改动都会改 SHA
- **Nacos `_wait_nacos_publish` 超时**:`uploaded_pending_publish` 状态字符串(语义成功,实际 label 延后改)
- **`_save_manifest` 失败** (OSS 路径):抛错,cycle 失败但 evolve_skill_registry.json 已更新——下轮 cycle reconcile

**走完之后**:

- 存储里有新 SKILL.md
- `evolve_skill_registry.json` 更新了(OSS 路径)
- `manifest.jsonl` 多了新行(OSS 路径)
- Nacos 状态机更新了(Nacos 路径)
- Stage 8 finalize 拿到 status 字符串写 `evolve_history.jsonl`

### 7.5 `_queue_validation_job`(validated 模式)

**场景**:publish_mode=validated 时,新生成的 skill **不直接上传**——而是写到 `validation_jobs/<job_id>.json`,等其它客户端投票。`_queue_validation_job` 是"建 job_id → 组装 job dict → 保存"的中央步骤。

**为什么是 3 步而不是 1 步**:

- job_id 要有可读时间戳(给运维查)+ 8hex 随机后缀(避免冲突)
- job dict 12 个字段分散在不同来源(skill / sessions / 4 个验证阈值)
- 保存要分两步:先在内存组装,再调 storage save

**怎么走**:

```python
async def _queue_validation_job(self, skill, action_type, sessions, current_skill, ...):
    # 段 1:建 job_id
    job_id = self._validation_store.make_job_id(skill["name"])
    # 格式: YYYYMMDDHHMMSS-slug-8hex

    # 段 2:组装 job dict(12 个字段)
    job = {
        "job_id": job_id,
        "status": "pending_validation",
        "candidate_skill_name": skill["name"],
        "candidate_skill": skill,
        "current_skill": current_skill,
        "proposed_action": action_type,
        "source": "evolve_server",
        "rationale": rationale,
        "session_ids": [s["session_id"] for s in sessions],
        "session_evidence": _build_session_evidence(sessions),
        "replay_cases": _build_replay_cases(sessions),
        "min_results": self.config.validation_min_results,  # 验证阈值 1
        "min_approvals": self.config.validation_min_approvals,  # 验证阈值 2
        "min_score": self.config.validation_min_score,  # 验证阈值 3
        "max_rejections": self.config.validation_max_rejections,  # 验证阈值 4
    }

    # 段 3:保存
    await self._validation_store.save_job(job)

    return {"action": "queued_for_validation", "validation_job_id": job_id, ...}
```

**`_build_replay_cases(sessions)`**:从每个 session 抽 `(session_id, turn_num, instruction, baseline_response, candidate_response)`——给 client validation worker 跑(详见 21 章)。

**关键边界**:

- **`make_job_id` 冲突** (同一毫秒 + 同一 skill):8hex 后缀保证低概率冲突
- **validation_store 不可达**:save_job 抛错,cycle 失败但 `evolve_history.jsonl` 记 error
- **replay_cases 过大** (session 太多):`_build_replay_cases` 不截断——可能 job 写不下,见 §11 待确认

**走完之后**:

- `validation_jobs/<job_id>.json` 写好
- 其它客户端的 validation worker 在 idle 时扫这个文件,跑 replay 验证
- 验证完成后 Stage 7 finalize (跨周期)再 publish
- Stage 8 finalize 拿到 `queued_for_validation` 状态字符串写 history

## 8. Stage 7 · Finalize Validation(`publish_mode=validated` 模式才跑)

**场景**:Stage 6 上传时,`publish_mode=validated` 的 skill 被写到 `validation_jobs/<job_id>.json` 等其它客户端投票——这个 job 可能在多个 cycle 里挂着,直到满足 4 个阈值(`min_results` / `min_approvals` / `min_score` / `max_rejections`)。Stage 7 是**每轮 cycle 末尾**扫所有 pending job,看哪些已满足阈值需要 publish。

**`workflow.py:577-580` 函数首行的关键 guard**:

```python
async def _finalize_validation_jobs(self, summary):
    if self.config.publish_mode != "validated":
        return [], summary  # direct 模式立刻返回空
    # ... 扫 pending job 逻辑
```

`direct` 模式下函数**立刻返回空**——所有"扫 pending job"逻辑都不会跑(空转)。`validated` 模式下,在 `run_once` 末尾**不管有没有 session 都跑**(即使 drain 0 条 session 也会扫 pending job)——这是为了"不漏 finalize"。

**为什么是"`direct` 模式空返"而不是"按需跑"**:

如果 `_finalize_validation_jobs` 不 guard,`direct` 模式下也会扫 storage 的 `validation_jobs/` 目录——但 `direct` 模式**根本不会写** validation jobs(Stage 6.1 直接上传)。所以扫了也白扫。guard 让代码意图更明确。

**为什么 `validated` 模式"不管有没有 session 都跑"**:

validation 是**跨周期**的——一个 validation job 可能挂 3-5 个 cycle 才有足够客户端投票。如果只在"有 session 的 cycle 跑",没 session 的 cycle 就漏扫,validation job 卡住。"每轮都跑"保证 finalize 及时触发。

`self._finalize_validation_jobs` 的具体判定逻辑(4 阈值)详见 21 章 §4。

**关键边界**:

- **`publish_mode=direct`**:立刻返空,不扫 validation_jobs 目录
- **`publish_mode=validated` + 没 session**:仍跑 finalize 阶段
- **`publish_mode=validated` + 有 session**:正常跑(包含 drain + 6 段 + finalize)
- **所有 validation job 都不满足阈值**:不 publish,但扫了不浪费(只是查表)
- **某个 validation job 满足阈值**:调用 `validate` 把 candidate skill 提到正式 skill(走 Stage 6.3/6.4 路径)

**走完之后**:

- 满足阈值的 validation job 被 publish
- `evolve_history.jsonl` 加一行 `validation_published` 记录
- `validation_jobs/<job_id>.json` 状态字段从 `pending_validation` 改 `published`

## 9. 周期 / 启动 / 停止

`run_periodic` 是一个 `while self._running` 循环:循环体走"跑一次 cycle → sleep 一轮间隔"两步。

```python
async def run_periodic(self):
    while self._running:
        try:
            await self.run_once()
        except Exception:
            logger.error("cycle failed", exc_info=True)
            # 失败不退出循环
        await asyncio.sleep(self.config.interval_seconds)  # 默认 600s
```

**为什么 `try/except` 在 while 里**:cycle 失败(LLM 不可达 / storage 不可达 / 任意异常)**不应该让 evolve server 退出**——下个 cycle 也许能成功。`logger.error` 记日志,继续。

**为什么 `asyncio.sleep(interval_seconds)` 在 cycle 完才睡**:

- 不是"先 sleep 再 cycle"(会延后第一次触发)
- 是"先 cycle 再 sleep"(cycle 完等下一轮)— `interval_seconds=600` 时,每 10 分钟一次

**HTTP 触发器**:`create_http_app` 起一个 FastAPI app(title `"SkillClaw Evolve Server"`),挂 3 个端点:

| 端点 | 方法 | 功能 |
|---|---|---|
| `/trigger` | POST | 立刻跑一次 cycle(不 sleep) |
| `/status` | GET | 报 `pending_sessions` (list_session_keys 数量) + `registered_skills` (SkillIDRegistry 长度) |
| `/health` | GET | 永远 200(给 liveness probe) |

`/trigger` 立刻跑一次 cycle——客户端可以通过这个"长会话中途触发"演化(与 `dashboard.ops.trigger-evolve` 对接)。

**运行模式**:

- **后台模式**:`run_periodic` 跑在 `asyncio.run(server.run_periodic())`
- **HTTP 模式**:把 `run_periodic` 和 `uvicorn.Server.serve()` 用 `asyncio.gather` 并行——同时跑周期和 HTTP 触发

**关键边界**:

- **`run_once` 跑超 interval_seconds** (cycle 太慢):下个 cycle 立刻跑(没有补 sleep)
- **进程被 SIGTERM 终止**:`while self._running` 循环收到 cancel,`run_once` 完成后退出
- **`/trigger` 并发调用**:两次 trigger 不会并发跑(内部有 lock 串行化)
- **`/trigger` 在 cycle 中触发**:等当前 cycle 完才跑下一个
- **`interval_seconds=0`**:禁用周期(只靠 `/trigger` 跑)

## 10. 数据流示例：一次有冲突的演化

**场景**:某次 cycle,evolve server 拉了 3 个 session(都引用了 `debug-systematically`),其它客户端的上一轮 cycle 刚改过这个 skill,导致本轮 `_detect_conflict` 返 `True`。走完整 11 步看具体怎么走。

**为什么这个例子重要**:这个 cycle 触发了 §7.1 → §7.3 → §7.4 整条"conflict → merge → upload"流水线——`evolve_history.jsonl` 里这条记录很有诊断价值(能看出"LLM merge 决定是什么"、"最终 version 是 4 还是 5")。

**整体 11 步流程图**:

```
run_once()
  │
  ├─ 1. Drain: 3 个 session 进 queue (sessions: list[dict])
  │
  ├─ 2. Summarize: 给每个 session 加 _trajectory / _summary / _skills_referenced
  │
  ├─ 3. Judge: 3 个 session 都有 aggregate.score 跳过 LLM judge
  │
  ├─ 4. Aggregate: skill_groups = {"debug-systematically": [3 sessions]}, no_skill = []
  │
  ├─ 5. Evolve: _evolve_skill_group("debug-systematically", sessions, [...])
  │     ├─ _fetch_skill 拿老 SKILL.md (sha = "abc123")
  │     ├─ evolve_skill_from_sessions 调 LLM
  │     │    └─ LLM 返 {action: "improve_skill", skill: {name, desc, category, content}, rationale}
  │     └─ _materialize_skill:
  │          ├─ verifier enabled=False 跳过
  │          ├─ publish_mode=direct 跳过 queue
  │          └─ _resolve_and_upload:
  │               ├─ _detect_conflict: 远端 sha != incoming sha → True
  │               ├─ execute_merge(LLM 合并两版) → merged_skill
  │               └─ _upload_skill(merged_skill, action="merge"):
  │                    ├─ skill_id, version = id_registry.get_or_create("debug-systematically")  # (id_42, 4)
  │                    ├─ put_object 写 SKILL.md
  │                    ├─ record_update 写 registry
  │                    ├─ save_version_bundle 写 v4/files/ + record.json
  │                    └─ update manifest + save_manifest
  │               → 返 ("merge", True), group_record = {action: "merge", skill_name: ..., version: 4, ...}
  │
  ├─ 6. no_skill = [] 跳过 create_skill 路径
  │
  ├─ 7. publish_mode=direct 跳过 finalize_validation
  │
  ├─ 8. _id_registry.save_to_oss(...) 持久化 registry
  │
  ├─ 9. delete_session_keys(...) 删 3 个 session(从 storage 删)
  │
  ├─ 10. _append_history(summary) 写 evolve_history.jsonl
  │
  └─ 11. uploaded_skills > 0 → 触发 _notify_proxy_reload(callback 模式)
       # 客户端 30s 内 poll 到新 manifest,拉新 skill
```

**每步的"为什么"和"输出"**:

| # | 步 | 输出 | 关键 |
|---|---|---|---|
| 1 | Drain | `sessions = [3 个 session dict]` | `_drain_sessions` 拉 3 个 key |
| 2 | Summarize | 每个 session 加 `_trajectory` + `_summary` + `_skills_referenced` | LLM 调 3 次(并行) |
| 3 | Judge | 3 个 session 跳过 LLM judge(已有 aggregate.score) | aggregate.score 来源是 benchmark/前次 cycle |
| 4 | Aggregate | `skill_groups = {"debug-systematically": [3 sessions]}` | 3 个 session 引用同一 skill |
| 5 | Evolve | 老 skill 被合并,version 升 4,上传新 SKILL.md | conflict 触发 merge |
| 6 | no_skill 路径 | 跳过 | 本例没 no_skill session |
| 7 | finalize_validation | 跳过 | publish_mode=direct |
| 8 | save registry | `evolve_skill_registry.json` 写盘 | OSS 路径必走 |
| 9 | delete sessions | `sessions/*.json` 删 3 个 key | 避免下轮重复炼 |
| 10 | append history | `evolve_history.jsonl` 加 1 行 | cycle 收尾 |
| 11 | notify proxy | 客户端 SkillClawAPIServer 收到 callback | 仅 callback 模式 |

**`summary` 字典结构**(返给 `/status` 端点 + 写 history):

| 类别 | 字段 | 含义 |
|---|---|---|
| **基础信息** | `timestamp` | ISO 时间 |
| | `elapsed_seconds` | cycle 耗时(秒) |
| | `had_processing_error` | bool |
| **会话统计** | `sessions` | 处理数(本例 3) |
| | `skill_groups` | group 数(本例 1) |
| | `no_skill_sessions` | 无 skill 引用数(本例 0) |
| **演化结果** | `actions` | 各类 action 计数 |
| | `skills_evolved` | 改写数 |
| | `uploaded_skills` | 上传数 |
| | `candidates_queued` | 排队等验证数 |
| | `published_after_validation` | 验证后发布数 |
| | `evolutions` | 每条 `{action, skill_name, version, ...}` |
| **子系统状态** | `session_judge` | `{enabled, judged_sessions, scored_sessions}` |
| | `skill_verifier` | `{enabled, verified, accepted, rejected, min_score}` |
| | `validation_publish` | `{enabled, jobs_scanned, pending}` |

**关键边界**:

- **`aggregate.score` 已存在**:Step 3 跳过 LLM judge——用现成分(避免重复调 LLM)
- **`_detect_conflict` True**:Step 5 走 merge 路径,不是直接 upload
- **merge LLM 失败**:fallback 用 incoming skill 上传(§7.3)
- **3 个 session 引用同一 skill**:Step 4 聚合到同一 group,Step 5 调 LLM 一次(不是 3 次)
- **`_notify_proxy_reload` 只在 callback 模式**:polling 模式不触发(callback 模式客户端已注册,polling 模式客户端不订阅)

## 11. 待确认 / 已知限制

- **`_run_session_judge` 用 `asyncio.gather` 全部并行**：10 个 session 就是 10 个并发 LLM 调用——可能触发 rate limit。**待确认**是否要加 semaphore。
- **`_build_session_evidence` 限制 30 sessions**——超过 30 个会丢尾部。**待确认**对"长期积累的场景"会不会丢。
- **`_detect_conflict` 用 sha256 比**：如果两个 client 同时上传不同内容 → race condition（last-write-wins）。**待确认**是否要用 ETag / If-Match。
- **`save_version_bundle` 删除 stale files**：但 `v<N>/files/` 之外的 `v<M>/files/`（M < N）**不**清——历史版本完整保留。**待确认**这是有意的（审计需要）还是疏忽。
- **`execute_merge` 失败 fallback to incoming**：万一 LLM 给出"无法 merge"，**不** reject candidate——直接把 incoming 上传。**待确认**该 reject 还是降级。
- **`_append_history` 单文件 append**：长时间运行后无 rotation。
- **`run_periodic` 用 `self._running` 标志位**：但 `asyncio.sleep` 不能被中断——Ctrl-C 收到后最坏要等 `interval_seconds`（默认 600s）才退出。生产应配 `TimeoutStopSec`。

---

→ **下一篇**：[32-Agent引擎详解](32-Agent引擎详解.md) — 把整片 workspace 丢给 OpenClaw 子进程的玩法
