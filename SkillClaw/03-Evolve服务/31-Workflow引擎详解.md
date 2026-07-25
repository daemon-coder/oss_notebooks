# 31 · Workflow 引擎详解

## 这一章讲什么

这一章回答"workflow 引擎的 6 段流水线每一步干什么、conflict 怎么 merge、verified mode 怎么排队、verifier 怎么打分"。

## 它在整个系统的哪个位置

`evolve_server/engines/workflow.py`（1063 行），`evolve_server/pipeline/{summarizer, aggregation, execution, session_judge, skill_verifier}.py`（共 ~1675 行）。`EvolveServer` 用 `EvolveEngineMixin` + workflow-specific 方法实现 6 段流水线。

## 设计目的

把"session 变 skill"拆成**可独立调 LLM 的 6 段**，让每段都有清晰的 system prompt / 输入 / 输出，运维可以单独调 temperature、verifier 阈值、merge 策略。

---

## 1. `run_once` 顶层流程

```python
# 伪代码骨架（详见 workflow.py:EvolveServer.run_once）:
async def run_once(self) -> dict:
    sessions, session_keys = await self._drain_sessions()
    if not sessions:
        return await self._finalize_validation_jobs_empty_queue()    # 仍跑 validation
    # 6 段: summarize → judge → aggregate → evolve(+create) → finalize
    # 收尾: registry save / session delete / history / notify_proxy_reload
    return summary
```

**6 段（如果 sessions 存在）**：
1. **Drain**：从 storage 拉 `sessions/*.json`
2. **Summarize**：LLM 给每个 session 加 `_trajectory` + `_summary` + 提取 metadata
3. **Session Judge**（optional）：LLM 给整个 session 打 0~1 分（加权 4 维）
4. **Aggregate**：按 `_skills_referenced` 分组
5. **Evolve/Create**：每个 group 调 LLM 决定 improve / optimize_description / create / skip
6. **Upload**：confict 检测 → merge（如有）→ write skill + versions + manifest

**额外 1 段（无论 sessions 是否有）**：
7. **Finalize validation**：扫所有 `validation_jobs` 还没 finalize 的，结果数够了就 publish

## 2. Stage 1 · Drain（`_drain_sessions`）

基类 `EvolveEngineMixin._drain_sessions`：

```python
keys = await self._call_storage(list_session_keys, self._bucket, self._prefix)
sessions, consumed_keys = [], []
for key in keys:
    session = await self._call_storage(read_json_object, self._bucket, key)
    if session: sessions.append(session); consumed_keys.append(key)
return sessions, consumed_keys
```

**`list_session_keys(bucket, prefix)`** 列 `{prefix}sessions/*.json`。

`_call_storage` 关键：**local store 同步调用**（filesystem 够快），**remote store 走 `asyncio.to_thread`**（OSS/S3 阻塞）。`if not sessions: logger.info("queue empty")`——空 queue 走 fast path（直接进 stage 7 finalize）。

## 3. Stage 2 · Summarize（`summarizer.py`）

### 3.1 双重表征

对每个 session 加 3 个字段：

| 字段 | 产生者 | 用途 |
|---|---|---|
| `_trajectory` | 程序构造（**无 LLM**） | 损失极小的 step-by-step 路径（tool_call 顺序、prompt / response 摘录、PRM 分、tool_errors） |
| `_summary` | LLM（system prompt 控制） | 因果链 + 洞察（"agent 策略、关键转折点、skill 有效性、结果评估"） |
| `_skills_referenced`, `_avg_prm`, `_has_tool_errors` | 程序提取 | 后续 aggregate / judge / evolve 用 |

### 3.2 轨迹构造（`build_session_trajectory`）

```python
def _format_tool_calls(turn):
    # 每个 tool_call 拼成 "    name(args) → ✓ cmd=... | content..." 或 "→ ✗ [err_type] msg"
    # 限制：args ≤ 400 字符，results ≤ 400 字符，errors ≤ 300 字符，最多 8 个 tool/step

def build_session_trajectory(session):
    # 1. flat trajectory（多 rollout 串成时间线）
    #    或 rollout trajectory（每个 rollout 单独一段）
    # 2. 拼 first_prompt + turns 列表 → 多行文本
```

`_PROMPT_MAX = 400` / `_RESPONSE_MAX = 400` / `_TOOL_ARG_MAX = 400` / `_TOOL_RESULT_MAX = 400` / `_TOOL_ERR_MAX = 300` / `_MAX_TOOLS_PER_STEP = 8`：每个值的字符上限（防 LLM context 爆炸）。

`_build_session_payload(session)`：从 turn 提取 skill 引用、PRM 分、tool errors、PRM votes 拼成 `_summary` LLM 看的"压缩证据"。

### 3.3 LLM 调用（`summarize_session`）

```python
async def summarize_session(llm, session) -> str:
    payload = _build_session_payload(session)
    raw = await llm.chat([
        {"role": "system", "content": SUMMARIZER_SYSTEM},  # 8-15 句分析 prompt
        {"role": "user", "content": json.dumps(payload, ensure_ascii=False, indent=2)},
    ], max_tokens=4096, temperature=0.3)
    return raw
```

**`summarize_sessions_parallel`** 用 `asyncio.gather` 并行多个 session 的 LLM 调用（concurrency 由 `llm` 内部 + event loop 调度）。

`_extract_session_metadata(session)`：从 turn 列表提取 `_avg_prm` / `_has_tool_errors` / `_skills_referenced` / `_assistant_tools` / `_tool_error_count` / `_user_turn_count` / `_first_user_turn` / `_last_user_turn` / `_trajectory`——**注意**：`_summary` 不在这里生成，由 LLM 写。

`_SUMMARIZER_DEBUG_DIR`（set_summarizer_debug_dir）：如果设了 `config.debug_dump_dir`，会把 system / user / raw 输出 dump 到 `<debug_dump_dir>/<session_id>_*.txt`。

## 4. Stage 3 · Session Judge（`session_judge.py`）

### 4.1 是否需要

`use_session_judge=True`（默认）才跑；`False` 时直接跳过。

### 4.2 输入

每个 session 的 `_trajectory` + `_summary` + 提取的 source artifacts + 提取的 output artifacts + 轻量 metadata。

### 4.3 Prompt

```
You are a session-level evaluator for SkillClaw trajectories.

Score on 0.0-1.0:
  task_completion: 0.55
  response_quality: 0.30
  efficiency: 0.05
  tool_usage: 0.10

Use 0.5 for mixed/uncertain, 1.0 for clearly excellent, 0.0 for clearly failed.
Prefer trajectory as ground truth; summary is supporting analysis.
Do not assume benchmark labels exist.
```

返回 JSON：
```json
{
  "task_completion": 0.85,
  "response_quality": 0.7,
  "efficiency": 0.6,
  "tool_usage": 0.75,
  "overall_score": 0.76,
  "rationale": "..."
}
```

### 4.4 跳过条件（`_should_skip_judging`）

- 已有 `aggregate.rollout_count > 0` 且 `mean_score` 存在（benchmark 已经给过分）
- 已有 `_judge_scores.overall_score`（已评过）
- 已有 `aggregate.success_count` / `fail_count`（明确结果）

这些情况直接用现成分。

### 4.5 `_apply_judge_scores(session, scores)`

把 LLM 返回的分写进 `session["_judge_scores"]` = `{task_completion, response_quality, efficiency, tool_usage, overall_score, rationale}`。

### 4.6 `judge_sessions_parallel(llm, sessions)`:`asyncio.gather` 并行多个 LLM 调用。

> **关于"重试"**:本章早期版本说"`judge_sessions_parallel` 用正则兜底提取,**3 次重试**"——这是**错的**。`judge_sessions_parallel`(`session_judge.py:397+`)和它调用的 `judge_session`(`session_judge.py:362+`)都是**单次 LLM 调用**:`judge_session` 调 LLM 一次,失败/解析失败都 `return None`(`session_judge.py:377` / `385-391`),**没有**外层 `for attempt in range(3)` 之类的重试结构。读者按"3 次重试"实现,会多走 2 倍 LLM 流量并改变统计语义——按"单次调用;失败/解析失败返回 None,session 不打分,后续靠 PRM 列表回退"实现。

### 4.7 `_apply_judge_scores` 把 PRM 历史回写

`_apply_judge_scores(session, scores)`(`session_judge.py:319-335`)**还会**回写 PRM 历史的 2 个字段:

```python
previous_prm_scores = list(session.get("_prm_scores") or [])   # 由 _extract_session_metadata 从 turn-level prm_score 收集
if previous_prm_scores:
    judge_scores["original_prm_scores"] = previous_prm_scores
    judge_scores["previous_last_prm_score"] = previous_prm_scores[-1]
session["_judge_scores"] = judge_scores
```

**`original_prm_scores`**:judge 评之前所有 turn 的 PRM 列表(原值不动,**不**是 judge 之后的);**只有 `previous_prm_scores` 非空时才会写入**——空 PRM 列表(`[]`)是 falsy,字段不出现。
**`previous_last_prm_score`**:PRM 列表最后一项,同上——只在非空时写入。

> **本章早期版本的"Step 3 输出"示例把 `original_prm_scores` 写成 `[]`、`previous_last_prm_score` 漏掉**——这是**错的**。`_extract_session_metadata`(`summarizer.py:349-382`)把所有非 None 的 turn-level `prm_score` 收进 `_prm_scores`;本例 turn 2 的 `prm_score=0.2` 让 `_prm_scores=[0.2]`,进 judge 时 `previous_prm_scores=[0.2]`(truthy),`session_judge.py:328-329` 会执行 `judge_scores["original_prm_scores"] = [0.2]`,`session_judge.py:330-331` 还会设置 `judge_scores["previous_last_prm_score"] = 0.2`。读者照错版示例写代码会出现"PRM 列表消失"的 bug。

## 5. Stage 4 · Aggregate（`aggregation.py`）

```python
def aggregate_sessions_by_skill(sessions) -> dict[str, list[dict]]:
    groups = defaultdict(list)
    for session in sessions:
        skills = session.get("_skills_referenced") or set()
        if not skills:
            groups[NO_SKILL_KEY].append(session)
        else:
            for skill_name in skills:
                groups[skill_name].append(session)
    return dict(groups)
```

**规则**（来自 docstring + 代码）：
- 如果 session 引用了 skill X → 进 X 的 group
- 引用了 A+B → 同时进 A 和 B 的 group
- 没引用任何 skill → 进 `__no_skill__` group（`NO_SKILL_KEY` 常量）

`NO_SKILL_KEY = "__no_skill__"`：特殊 key，给 Stage 6 的"创建新 skill"逻辑用。

## 6. Stage 5 · Evolve/Create（`execution.py`）

### 6.1 `_evolve_skill_group(skill_name, sessions, existing_skill_names)`

```python
# 伪代码骨架（见 workflow.py:EvolveServer._evolve_skill_group）:
async def _evolve_skill_group(self, skill_name, sessions, existing_skill_names):
    current_skill = await self._fetch_or_init_current_skill(skill_name)
    result = await evolve_skill_from_sessions(self._llm, ..., current_skill, existing_skill_names)
    if not result or result.action == SKIP: return None
    # OPTIMIZE_DESC 保留 body；其他动作 inherit frontmatter
    return await self._materialize_skill(result.skill, result.action, ..., current_skill=current_skill)
```

**4 个动作**(`DecisionAction` 常量,字面值见 `evolve_server/core/constants.py:35-41`):
- `CREATE = "create_skill"`:从 no-skill bucket 造新 skill
- `IMPROVE = "improve_skill"`:老 skill 改写
- `OPTIMIZE_DESC = "optimize_description"`:只改 description,body 继承老的
- `SKIP = "skip"`:LLM 决定不动

> **"默认" 的精确说法**:`_parse_evolve_result`(`execution.py:447`)的 action 默认值是 `DecisionAction.SKIP`——LLM 不返回 action 字段时,默认**跳过**该 skill(`action = result.get("action", DecisionAction.SKIP)`)。`workflow.py:852` 的 `action_type = result.get("action", DecisionAction.IMPROVE)` 几乎不会触发,因为 `_parse_evolve_result` 已经把"无 action"转换为 SKIP 走早返回路径。读者按"IMPROVE 当默认"实现,会让"LLM 偶尔不返回 action"的 session 反而被强制 IMPROVE——按"SKIP 是 default,IMPROVE 是想得到的非 skip 路径"实现。
>
> **`CREATE` 路径的兜底降级**:`_parse_evolve_result`(`execution.py:456-465`)有更具体的兜底:LLM 偶发返回 `create_skill` 但 `name == skill_name`(等于改名没改)时,会被**静默改写**为 `IMPROVE`——避免同名 create 改写历史。读者按"CREATE 走 create 路径"重写,会跟代码不一致——会出"应该 improve 的 skill 被错误地走 create 路径并改了 name 失败"。
>
> **`_materialize_skill` 强制 name 行为**:`workflow.py:760-764` 在 `_materialize_skill` 里:**`action_type == DecisionAction.IMPROVE` 时强制 overwrite skill name 为 `current_skill['name']`**——LLM 想 rename 也会被覆盖;`CREATE` 路径才走 `_sanitise_name`。

**`_inherit_current_skill`**:老 skill 的 body / category / extra_frontmatter 灌到新 skill(按 overwrite_body 决定 body)。

### 6.2 `evolve_skill_from_sessions(llm, skill_name, sessions, current_skill, existing_skill_names)`

```python
async def evolve_skill_from_sessions(llm, skill_name, sessions, current_skill, existing_skill_names):
    system = _EVOLVE_FROM_SESSIONS_SYSTEM.replace("{skill_name}", skill_name)
    skill_section = _build_skill_block(current_skill) if current_skill else ""
    evidence = _build_session_evidence(sessions)              # 最多 30 sessions
    user_msg = f"{skill_section}## Session evidence ({len(sessions)} sessions)\n\n{evidence}\n\n## Existing skill names\n\n{', '.join(existing_skill_names) or '(none)'}\n"
    raw = await llm.chat(messages, max_tokens=8192, temperature=0.4)
    return _parse_evolve_result(raw, skill_name)
```

**`_build_session_evidence(sessions, max_sessions=30)`** 把 sessions 列表转成"### Session {id} ({prm}, {aggregate}, {errors}, {skills})\n**Trajectory**:\n...\n**Analysis**:\n..." 文本块。超过 30 个 session 加 "... and X more sessions"。

**`_parse_evolve_result(raw, skill_name)`** 解析 LLM 输出：
- 期望 JSON：`{"action": "improve_skill" | "optimize_description" | "create_skill" | "skip", "skill": {name, description, category, content}, "rationale": "..."}`
- 用正则剥 ```json fences；用 `find("{") + rfind("}")` 抽 JSON 块
- 校验：action 是 SKIP 时只要 rationale；其它要有 `skill` 字段（dict）
- `create_skill` 时必须 name
- `improve_skill` 时如果没 name，**用 `skill_name` 当 name**（不让 LLM 改老 skill 的 name）

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

```python
# 伪代码骨架（见 workflow.py:EvolveServer._materialize_skill）:
async def _materialize_skill(self, evolved_skill, action_type, ...):
    evolved_skill["name"] = self._sanitise_name(evolved_skill["name"])
    # 1. _run_skill_verifier（use_skill_verifier 启用时）— 不通过则 verification_rejected
    # 2. 分配 skill_id
    # 3. publish_mode=validated → _queue_validation_job（返回 queued_for_validation）
    # 4. 否则 → _resolve_and_upload
    return {"action": ..., "uploaded": ..., "verification": ...}
```

**5 个出口**：
1. `verification_rejected`：verifier 拒（verifier enabled + accept=False）
2. `queued_for_validation`：publish_mode=validated，写 validation job
3. `uploaded`：直接上传成功
4. `*_pending_review`：Nacos `review` 模式（已 upload，等人工 review）
5. `*_pending_publish`：Nacos `direct` 模式但 `_wait_nacos_publish` 30s 超时（uploaded but latest label 未改）
6. `*_draft`：Nacos `draft` 模式（只上传、不进审核流）

### 7.2 `_run_skill_verifier`（`skill_verifier.py`）

```python
async def _run_skill_verifier(self, skill, action_type, sessions, *, current_skill=None):
    if not self.config.use_skill_verifier:
        return {"enabled": False}
    return await verify_skill_candidate(self._llm, skill, sessions, action_type, current_skill=current_skill, min_score=self.config.skill_verifier_min_score)
```

> **设计目标(必读)**:`verify_skill_candidate` 用**同一个 LLM**(跟生成 skill 的 evolve LLM 共享 `AsyncLLMClient`,见 [30 · Evolve 服务端与引擎选择](../03-Evolve服务/30-Evolve服务端与引擎选择.md) §X)做"价格低 + 一致性高 + 部署简化"的**快速预筛**——**不是**独立审计;要独立审计可加 rule-based 闸门或换 LLM。verifier LLM 跟 generator LLM 同一份 = "自己审自己",无法识别同一 LLM 的系统性偏见。

`verify_skill_candidate` 的 LLM prompt("final publication gate"):
- 必须 grounded in evidence
- 不丢 useful environment-specific facts
- specific and reusable(不是泛泛 agent advice)
- 评价 4 个维度:grounded_in_evidence / preserves_existing_value / specificity_and_reusability / safe_to_publish
- 0.75 分以下 reject(默认)

**`verify_skill_candidate` 内部**(`skill_verifier.py:222-231`):
- 构造 payload(candidate_skill + current_skill + session_evidence + acceptance_threshold)
- 调 LLM(max_tokens=2000, temperature=0.1)
- `_extract_json_object` 剥 ```json fences + 抽 JSON
- 解析 4 个 check 维度的 0-1 分(赋给 `checks` 字典)
- LLM 返回 `score` 优先用 LLM 给的;没给就用 4 个 check 的均分(`_compute_score`)
- **拒绝条件(精确)**——只有**总 `score < min_score`** 时拒绝;`decision_raw` 不在 `{"accept", "reject"}` 时兜底用 `score >= min_score`:
  ```python
  if decision_raw == "accept":
      accepted = True
  elif score is not None and score < float(min_score):
      accepted = False                                # 唯一硬性拒绝
  elif decision_raw not in {"accept", "reject"}:
      accepted = score is not None and score >= float(min_score)
  ```

> **关于"4 checks 任一 < 0.5 拒绝"**:本章早期版本说 verifier 拒绝条件是 "`score < min_score` **或者 4 个 checks 有任一 < 0.5 时** 拒绝"——这是**错的**。`verify_skill_candidate` 整段代码**没有任何**对 `checks` 内单个维度 < 0.5 的拒绝逻辑,`checks` 字段只是辅助输出(展示给运维看),**不**参与 gating。读者按"per-check 守门"实现,会出现"LLM 总分 0.8 但 `grounded_in_evidence=0.3` 仍被放行"——按"当前只按 `score < min_score` 拒绝"实现。`per-check 阈值` 是已知未实现项,见 §11 `[待核]`。

### 7.3 `_resolve_and_upload`（conflict → merge → upload）

```python
# 伪代码骨架（见 workflow.py:EvolveServer._resolve_and_upload）:
async def _resolve_and_upload(self, skill, action_type):
    # 1. _detect_conflict: 同名但 sha 不同 → merge
    # 2. 无冲突: _upload_skill(skill, action_type)
    # 3. 冲突: execute_merge(LLM) → 成功用 merge 结果；失败用 incoming
    return self._upload_status_to_action(action_type, upload_status)
```

**`_detect_conflict`**(`workflow.py:338-352`)是 if/else 分支——**OSS 路径**用 `id_registry` 存的内容 SHA;**Nacos 路径**用 `_fetch_skill` 拉当前 SKILL.md 重算 SHA:

```python
if self._sharing_skill_client:                          # Nacos 路径
    existing = await self._fetch_skill(skill_name)
    existing_sha = sha256(existing.encode()) if existing else ""
else:                                                   # OSS 路径
    existing_sha = self._id_registry.get_content_sha(skill_name) or ""
incoming_sha = sha256(build_skill_md(incoming_skill).encode())
return existing_sha and existing_sha != incoming_sha
```

**为什么 Nacos 路径不同?**Nacos 模式没有 `evolve_skill_registry.json`(`SkillIDRegistry` 不持久化),所以 SHA 是从远端 SKILL.md 实时算的——这意味着 Nacos 模式下 conflict detection 每次都走一次网络,会比 OSS 模式慢。读者按 OSS 写法去重写 Nacos 分支会卡住(找不到 `id_registry`)。

**`execute_merge(llm, existing_skill, incoming_skill)`**：把两个 SKILL.md 喂给 LLM，让 LLM 合并（用 `_MERGE_SKILL_SYSTEM` prompt）。返回的 merged skill 再走 `_upload_skill`（`action="merge"`）。

### 7.4 `_upload_skill`(Nacos / OSS / S3 三路径)

```python
# 伪代码骨架（详见 workflow.py:EvolveServer._upload_skill）:
def _upload_skill(self, skill, action) -> str:
    # Nacos 路径: working version 检测 → 上传 zip → submit/publish
    # OSS/S3/Local 路径:
    #   1. _id_registry.get_or_create(name) → skill_id
    #   2. build_skill_md(skill) → put_object SKILL.md
    #   3. record_update(name, content_sha, action, bundle_record)
    #   4. save_version_bundle (files + record.json)
    #   5. update manifest + save_manifest
    return "uploaded"
```

Nacos 路径详细（`upload_skill` 流程）：
1. 读 remote record + detail
2. `_nacos_working_version(record, detail)`：找 working version（有但没 published）
3. 如果有 working version：拉下来 `_bundle_matches_remote` 比——相同就 skip；不同就用 working version 号覆盖
4. 否则 `_next_version(record, detail)` 算新版本号
5. 打包 zip + upload
6. `nacos_publish_mode in {"review", "direct"}` → `submit(name, version)` 进审核
7. `nacos_publish_mode == "direct"` → `_wait_nacos_publish(name, version, timeout=30.0)` 轮询等 `labels.latest == version`
8. 返回 `uploaded` / `uploaded_draft` / `uploaded_pending_review` / `uploaded_pending_publish` / `skipped_existing_<status>`

### 7.5 `_queue_validation_job`（validated 模式）

```python
def _queue_validation_job(self, skill, action_type, sessions, rationale, source, *, current_skill=None):
    job_id = self._validation_store.make_job_id(skill["name"])  # YYYYMMDDHHMMSS-slug-8hex
    job = {"job_id": job_id, "status": "pending_validation",
           "candidate_skill_name": ..., "candidate_skill": skill, "current_skill": current_skill,
           "proposed_action": action_type, "source": source, "rationale": rationale,
           "session_ids": ..., "session_evidence": ..., "replay_cases": ...,
           "min_results": ..., "min_approvals": ..., "min_score": ..., "max_rejections": ...}
    self._validation_store.save_job(job)
    return {"action": "queued_for_validation", "validation_job_id": job_id, ...}
```

**`_build_replay_cases(sessions)`**：从每个 session 抽 `(session_id, turn_num, instruction, baseline_response, candidate_response)`——给 client validation worker 跑（详见 [21 · 后台验证工作流](21-后台验证工作流.md)）。

## 8. Stage 7 · Finalize Validation(`publish_mode=validated` 模式下每轮必跑)

> **关于"必跑"的精确说法**:本章早期版本标题写"`_finalize_validation_jobs`(每轮必跑)"——这在 `direct` 模式(默认)下其实是空转,会误导读者以为这条收尾在所有模式下都生效。**实际**:
>
> - `workflow.py:577-580` 函数首行就是 `if self.config.publish_mode != "validated": return [], summary`——**`direct` 模式下函数立刻返回空**,所有"扫 pending job"逻辑都不会跑。
> - **`validated` 模式**下,在 `run_once` 末尾**不管有没有 session 都跑**(即使 drain 出来 0 条 session 也会扫 pending job)——这是为了"不漏 finalize"。

`self._finalize_validation_jobs` 详见 [21 · 后台验证工作流](21-后台验证工作流.md) §4(判定伪代码 + 4 阈值)。

## 9. 周期 / 启动 / 停止

```python
async def run_periodic(self):
    while self._running:
        try: await self.run_once()
        except Exception: logger.error(...)
        await asyncio.sleep(self.config.interval_seconds)

def create_http_app(self):     # /trigger /status /health
    app = FastAPI(title="SkillClaw Evolve Server")
    @app.post("/trigger") @app.get("/status") @app.get("/health")
    return app
```

`run_periodic` 跑在 `asyncio.run(server.run_periodic())`；HTTP 模式把 `run_periodic` 和 `uvicorn.Server.serve()` 用 `asyncio.gather` 并行。

`/trigger` 立刻跑一次 cycle——客户端可以通过这个"长会话中途触发"演化（与 `dashboard.ops.trigger-evolve` 对接）。

`/status` 报告当前 `pending_sessions`（list_session_keys 数量）+ `registered_skills`（SkillIDRegistry 长度）。

`/health` 永远 200。

## 10. 数据流示例：一次有冲突的演化

```
1. drain: 3 个 session 进 queue
2. summarize: 每个 session 加 _trajectory / _summary / _skills_referenced
3. judge: 3 个都已有 aggregate score → 跳过
4. aggregate: skill_groups = {"debug-systematically": [3 sessions]}, no_skill = []
5. _evolve_skill_group("debug-systematically", sessions, [...])
   - _fetch_skill("debug-systematically") → 老 SKILL.md
   - evolve_skill_from_sessions() → LLM 决定 "improve_skill"，返回 {name, description, category, content, rationale}
   - _materialize_skill(...)
     - verifier: enabled=False 跳过
     - publish_mode=direct 跳过 queue
     - _resolve_and_upload(skill, "improve_skill")
       - _detect_conflict: sha256 不一样 → True
       - execute_merge(existing, incoming) → LLM 合并两版 → merged_skill
       - _upload_skill(merged_skill, "merge")
         - object_key = "default/skills/debug-systematically/SKILL.md"
         - put_object(...)
         - SkillIDRegistry.record_update(name, sha, "merge", bundle_record)
         - save_version_bundle(v<N>/{SKILL.md, bundle.json})
         - manifest[name] = {...}
         - save_manifest(...)
       - return ("merge", True)
   - record = {"action": "merge", "skill_name": "...", "version": 4, ...}
6. (no_skill = []) → 跳过
7. (publish_mode=direct) → 跳过 finalize
8. _id_registry.save_to_oss(...) → registry 持久化
9. delete_session_keys(...) → 3 个 session 删
10. _append_history(summary) → evolve_history.jsonl
11. uploaded_skills > 0 → _notify_proxy_reload (callback 模式)
```

返回的 summary：

```json
{
  "timestamp": "2026-04-20T15:00:00Z",
  "elapsed_seconds": 12.4,
  "sessions": 3,
  "skill_groups": 1,
  "no_skill_sessions": 0,
  "actions": 1,
  "skills_evolved": 1,
  "uploaded_skills": 1,
  "candidates_queued": 0,
  "published_after_validation": 0,
  "evolutions": [{"action": "merge", "skill_name": "debug-systematically", "version": 4, ...}],
  "session_judge": {"enabled": true, "judged_sessions": 0, "scored_sessions": 0, ...},
  "skill_verifier": {"enabled": false, "verified": 0, "accepted": 0, "rejected": 0, "min_score": 0.75},
  "validation_publish": {"enabled": false, "jobs_scanned": 0, "pending": 0, ...},
  "had_processing_error": false
}
```

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
