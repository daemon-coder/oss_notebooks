# 16 · PRM(Process Reward Model)与反馈回路

> **PRM = Process Reward Model**(过程奖励模型)——用外部 LLM 评单条 turn 的回答质量,返回 +1 / 0 / -1。**注**:SkillClaw 的 PRM **不**参与训练,只产本地统计信号。

## 这一章讲什么

这一章回答"`PRMScorer` 怎么给一条 turn 打分、分数怎么沿 asyncio callback 链路写回 `SkillManager._stats`、Bedrock provider 走哪条不同分支、score 最后怎么影响团队 skill 推送"。

## 它在整个系统的哪个位置

`SkillClawAPIServer._fire_prm_scoring` → `PRMScorer.evaluate`(并发 `prm_m` 票) → `asyncio.Task` 完成 → `add_done_callback` 触发 `_on_prm_done` → `_apply_prm_result` → `SkillManager.record_feedback` → `_stats[name].effectiveness`。链路是异步的,用 `asyncio.Task.add_done_callback` 注册结果处理(PRM 不阻塞主请求流程)。

## 设计目的

让 SkillClaw 知道"哪些 skill 真的有用、哪些是注水的"——但**不**靠"人类标"或"另一个机器学习模型",而是**让一个外部 LLM 评**。PRM 输出不是"训练数据",是"本地 skill 库的统计信号 + 团队 push 时的过滤阈值"。

---

## 术语速查(读本章前 1 分钟过一遍)

> 顺序按"首次出现章位"排;每条一行,需要时回查。术语详尽定义见各章链接,本表只标用途和类型。

- **`PRMScorer`** —— `skillclaw/prm_scorer.py` 的类;对外只暴露 `async evaluate(response, instruction) -> {"score": float, "votes": list, "eval_text": str}`。score ∈ {-1.0, 0.0, +1.0}。
- **`prm_m`** —— `PRMScorer.prm_m`(`prm_scorer.py:149`),int,默认 3。**每次 evaluate 并发发的票数**;最终 score 是这 `prm_m` 票的**多数票**。推荐奇数 `prm_m ∈ {1, 3, 5}`(见 §3)。
- **`_majority_vote(scores)`** —— `prm_scorer.py:99` 的纯函数。输入 `list[Optional[int]]`,返回 float;**平票或全 None → 0.0**;否则返回 top 票的值(1.0 / 0.0 / -1.0)。**不**是均值,不是中位。
- **`_READ_TOOL_NAMES`** —— `api_server.py:119` 的常量 `{"read", "file_read", "read_file", "readfile"}`。**反向**用于 `_extract_read_skills_from_tool_calls` 识别"模型实际读了哪个 skill"(详 15 章 §6)。与 prompt 里提示模型用的 `read_tool_name` 是**两条独立机制**(15 章 §6.1)。
- **`_HERMES_SKILL_READ_TOOL_NAMES`** —— `api_server.py:120` 的常量 `{"skill_view"}`。Hermes 适配器专用,与上一条配合(详 15 章 §6)。
- **`next_state`** —— `dict` 形如 `{"role": "user", "content": ...}`,表示"下一条 user message"。**PRM 必须等到 `next_state` 进来才评上一条回答**——需要"对照下一条 user 提问"判断回答有没有用。`None` 表示"这是 session 最后一条,不需要评"。14 章 §3.1 解释 `_flush_pending_record(session_id, next_state)` 时如何传。
- **`drain`** —— `_close_session` 末尾的"等 PRM 跑完"动作,具体是 `await asyncio.wait_for(asyncio.gather(*active_prm_tasks, return_exceptions=True), timeout=_SHUTDOWN_DRAIN_TIMEOUT_SECONDS)`,**默认 15 秒**(`api_server.py:2081-2086`)。超时后未完成的 PRM task 走 `prm_result = None` 的 fallback finalize。14 章 §3.2 已铺。
- **`_closing_sessions`** —— `api_server.py:1464` 的 `set[str]`;`_close_session` 开头 `add`,`finally` 块 `discard`(14 章 §3 末段)。用于 PRM callback 写回 turn record 前 check,避免"session 已关还在写"。
- **`_pending_turn_data`** —— `api_server.py:1458` 的 `dict[str, dict[int, dict]]`;外层 key 是 `session_id`,内层 key 是 `turn_num`(1-based),value 是 `{response_text, prompt_text, injected_skills, read_skills, has_next_state, prm_result?}`。**每条 turn 的"待 finalize 数据"**。
- **`_prm_tasks`** —— `api_server.py:1459` 的 `dict[str, dict[int, asyncio.Task]]`;外层 session、内层 turn(1-based),value 是 PRM 的 `asyncio.Task`。`_close_session` 时遍历 drain。
- **`prm_scores.jsonl`** —— PRM 评完一条 turn 时追加一行的本地记录文件,路径 `~/.skillclaw/sessions/<sid>/prm_scores.jsonl`(实际由 `record_dir` + `prm_scores.jsonl` 决定,`api_server.py:1498`)。**每行** JSON 形如 `{"session_id": ..., "turn": N, "score": float, "votes": [...]}`,**不带** eval_text(`api_server.py:2181-2199`)。**只用于审计 / 离线分析**,**不**被 evolve server 消费(20 章 §共享存储与同步)。
- **`manifest.jsonl`** —— 团队 push 时的 skill 推送记录(20 章 §共享存储),每行一个 skill 条目,带 `effectiveness` / `positive_count` / `negative_count` / `neutral_count` / `inject_count` / `uploaded_by` / `last_injected_at` 字段。**仅记录**,**不**被 evolve server 消费。
- **`injected_skills` / `read_skills`** —— `_apply_prm_result` 把 score 派给这两组 skill 各自调 `SkillManager.record_feedback`。
  - `injected_skills: list[str]`(skill 名字;15 章 §5)
  - `read_skills: list[dict]`,每项 `{"skill_name": str, "path": str, ...}`(15 章 §6)
  - 15 章 §7.1 已铺。
- **`_apply_prm_result` / `_on_prm_done` / `_on_prm_done_record_only`** —— PRM task done callback 链路中的三个关键方法(`api_server.py:2244 / 2268 / 2281`)。
  - `_on_prm_done`:正常路径,score 写回 + 调 `record_feedback` + 检查 `readiness` 触发 finalize。
  - `_on_prm_done_record_only`:session 关闭路径,只写 `prm_scores.jsonl` 和 turn record,**不**调 `record_feedback`(避免在关闭过程中再次改 `_stats`)。
- **`session_upload_interval`** —— `SkillClawConfig.sharing_session_upload_interval`(11 章 §配置系统,单位:user turn 数,默认 `0` = 关闭)。`> 0` 时**每 N 个 user turn 一次**触发 `_maybe_upload_session_snapshot`(周期化路径 B,详 §4 块引用)。
- **`_trigger_evolve`** —— `SkillClawAPIServer._trigger_evolve(self) -> None`(`api_server.py:3121`),**无入参**——`evolve_server_url` 从 `self.config.evolve_server_url` 读;向 evolve server 发 `POST {evolve_server_url}/trigger`(httpx 异步 client,3 次重试,timeout 300s)。
- **`inject_count`** —— `SkillManager._stats[name]["inject_count"]`(`skill_manager.py:251`),在 `record_injection(skill_names)` 时 `+= 1`(15 章 §7)。**和 PRM 无关**——`inject_count` 由"注入"触发,不是由"评分"触发。`effectiveness = positive_count / inject_count`,分母是"被注入次数",分子是"被评 +1 的次数"(详 §7)。
- **`use_prm`** —— `SkillClawConfig.use_prm: bool`(11 章 §配置系统,默认 `True`)。`False` 时 `_maybe_finalize_ready_turns` 整段 `if` 短路,每个 turn 立即 finalize,不等 PRM(详 §4.4)。

---

## 1. PRM 的本质

`PRMScorer`(`prm_scorer.py`)是一个**异步 OpenAI 客户端包装**。它**不**调 SkillClaw 上游的 LLM——它有自己的 LLM 客户端,可以是 OpenAI / Together / Fireworks / 自建 vLLM / 任何 `/v1/chat/completions` 兼容端点,或 AWS Bedrock(走包装的 `BedrockChatClient`,详 §5.2)。

打分流程:

```
PRMScorer.evaluate(response, instruction)
  → 并行发 prm_m 票(默认 3 票),每票都是同一条 prompt 给上游 LLM
  → 每票解析 "Score: 1" / "Score: -1" / "Score: 0"
     (fallback: 兼容历史 \boxed{N} 格式——见 §2 末尾)
  → _majority_vote(scores): 平票或全 None → 0.0;否则返 top 票值
  → 返回 {score: float, votes: [int|"fail", ...], eval_text: str}
```

**为什么用多票**?单次 LLM 打分有**随机噪声**——模型可能因一两句话的语气、措辞、标点给 +1 或 -1,与真实质量无关;在 evaluation 任务上这是已知现象(RM 论文常用 "self-consistency" 缓解)。多票多数票能显著降低方差。

## 2. PRM 打分 prompt

`_build_prm_judge_prompt`(`prm_scorer.py:52`)构造两段消息:

```python
# prm_scorer.py:_build_prm_judge_prompt(关键 7 行;真源码 13 行字符串字面量,详 prm_scorer.py:52)
system = (
    "You are a quality reviewer for conversational responses. ... "      # 角色
    "Do NOT compare against any follow-up turn. ... "                      # 关键约束 1
    "Use +1 when the response clearly follows and substantially completes the instruction. "
    "Use -1 when the response is off-task, wrong, or fails to complete core requirements. "
    "Use 0 when completion is ambiguous or evidence is insufficient. "
    "End your reply with exactly one of: Score: 1 / Score: -1 / Score: 0" # 关键约束 2
)
user = f"Instruction:\n{clean_instruction}\n\nResponse:\n{clean_response}\n\n..."
```

**关键约束**(写进 system prompt 防 LLM 跑偏):

- "Do NOT compare against any follow-up turn"——只看 response 对 instruction 的完成度,不看后续
- "End your reply with exactly one of: Score: 1 / Score: -1 / Score: 0"——强制结构化输出

**`\boxed{N}` 旧格式说明**:`_parse_prm_score`(`prm_scorer.py:77`)主路径解析 `Score: N`,**fallback 路径**解析 `\boxed{N}`。后者是历史版本 / DeepSeek-Math 等数学论文里用过的输出格式——某些自托管 LLM 训练数据里仍偏好这种格式,留作 fallback 兼容(不抛错)。**当前推荐**用 `Score: N` 走主路径,`\boxed{N}` 仅作兜底。

### 2.1 `_sanitize_text`:在送 LLM 前清洗

`_sanitize_text`(`prm_scorer.py:40`)在拼 prompt 前清洗,防上游 content filter 误触:

```python
text = _re.sub(r"<tool_call>.*?</tool_call>", "[tool_call block]", text, flags=_re.DOTALL)
text = _re.sub(r"<[a-zA-Z_][^>]{0,80}>", "[tag]", text)
text = _re.sub(r"</[a-zA-Z_][^>]{0,80}>", "[/tag]", text)
```

**为什么第一条 regex 专门匹配 `<tool_call>` 块**?OpenClaw / Hermes 适配器把工具调用序列化为 `<tool_call>...</tool_call>` 块——这种"看起来像 prompt injection"的内容会让 Azure / 部分 vLLM 部署拒收请求。`[tool_call block]` 是一个对 LLM 友好的中性占位,让 PRM 看到"这里有工具调用发生过"但不会被 provider 拒。

**剩下两条** regex 把任何 `<tag>` / `</tag>` 替成 `[tag]` / `[/tag]`——同样防 content filter 误判。**不是** 替换所有尖括号(只替换"以字母 / 下划线开头的 tag"——数值、路径等不会误伤)。

> **注**:`_sanitize_text` 是按**半角**尖括号 + 半角字母正则的;早期草稿里误写为全角 `\uff08` / `\uff09`,**真源码里没有全角字符**,照源码写即可。

## 3. 多票多数票

```python
results = await asyncio.gather(*[self._query_once(msgs, i) for i in range(self.prm_m)])
scores = [r[0] for r in results]    # 每票分数 (int | None)
final = _majority_vote(scores)
```

`_majority_vote`(`prm_scorer.py:99`):

```python
def _majority_vote(scores):
    valid = [s for s in scores if s is not None]
    if not valid: return 0.0
    counter = collections.Counter(valid)
    top = counter.most_common(1)[0]
    if list(counter.values()).count(top[1]) > 1:  # 平票(如 [1, -1] 各一)
        return 0.0
    return float(top[0])
```

`prm_m=3` 常见结果:

| votes | final | 说明 |
|---|---|---|
| `[1, 1, 1]` | 1.0 | 一致 +1 |
| `[1, 1, -1]` | 1.0 | 2 vs 1,多数 +1 |
| `[1, -1, -1]` | -1.0 | 1 vs 2,多数 -1 |
| `[1, -1, None]` | 0.0 | top=1 出 1 次、top=-1 出 1 次 → 平票 |
| `[None, None, None]` | 0.0 | 全失败 → 0.0 |

**奇数 vs 偶数**:`_majority_vote` 用 `most_common(1)[0]` + `count(top[1]) > 1` 判定"是否有唯一 top"——**对奇数和偶数都成立**。所以:

- `prm_m=1`:不做投票,直接用单票结果(仍解析 `Score:` 行)。
- `prm_m=2`:永远平票(除非一票 None)→ 永远 0.0。**不推荐**——等于"永远无效"。
- `prm_m=3`:**推荐**。3-0 / 2-1 都能判出唯一 top。
- `prm_m=4`:`[1,1,1,-1]` → 1.0(top=1 出 3 次);`[1,1,-1,-1]` → 0.0(2-2 平票,top 出现 2 次算"top 不唯一")。
- `prm_m=5`:`[1,1,1,-1,-1]` → 1.0(3 vs 2);`[1,1,-1,-1,None]` → 0.0(2-2 平票)。

**结论**:`prm_m` 推荐奇数 `∈ {1, 3, 5}`。偶数会偶发"平票 → 0.0",但 0.0 跟"评估不出"是**同一个返回值**,reader 区分不开(详 §10 第 1 条)。

**`prm_m=0` 静默 0.0**(已知设计权衡,详 §10 第 1 条):`asyncio.gather(*[])` 跑空,`_majority_vote([])` 返 0.0,所有 turn 的 `prm_score` 都是 0.0,effectiveness 全 0%——**生产静默故障**。**当前实现不**加 `assert prm_m >= 1`(`prm_m` 是普通 int,不是 enum),`__init__` 不会报错;生产如要避免,在 `setup_wizard` / `config.yaml` 校验层加 `prm_m >= 1`(详 11 章 §配置系统 字段校验)。

## 4. 触发与回调链路

> **两条独立触发路径**——它们在 `SkillClawAPIServer` 里**都存在**,但**互不依赖**:
>
> - **路径 A(PRM callback)**:`_fire_prm_scoring` → asyncio task → `add_done_callback` → `_on_prm_done` → `_apply_prm_result` → 写 turn_record + 调 `record_feedback`(本节主体)。
> - **路径 B(周期化 evolve 触发)**:`sharing_session_upload_interval > 0` 时,`_maybe_upload_session_snapshot` 在**每 N 个 user turn** 触发 `_upload_session_snapshot_and_trigger` → `_upload_session_data` + `_trigger_evolve`(详 14 章 §3 与 20 章)。
>
> 这两条线**没有"互相等待"关系**:PRM 不必先完成才能 trigger evolve,evolve trigger 也不依赖 PRM 结果(但 session 上传时,已完成的 PRM 分数会一起过去;未完成的 turn 的 `prm_score` 字段是 `None`)。

### 4.1 触发条件(路径 A)

PRM **不是**每个 turn 都打——它要求"下一条 user message 进来"才能评上一条回答("这个回答对下一个问题有没有用")。

```python
# api_server.py:_fire_prm_scoring(关键 8 行,真源码 25 行详 api_server.py:2219)
def _fire_prm_scoring(self, session_id, turn_num, response_text, instruction_text,
                      next_state, finalize_ready_turns=True):
    if not self.prm_scorer or not next_state: return       # 关键守卫:无 prm_scorer / 无 next_state → 不评
    task = asyncio.create_task(self.prm_scorer.evaluate(...))   # 异步启 PRM task
    task.add_done_callback(self._task_done_cb)            # 通用 done 钩子
    cb = self._on_prm_done if finalize_ready_turns else self._on_prm_done_record_only
    task.add_done_callback(lambda _t, c=cb, sid=session_id, tn=turn_num: c(sid, tn, _t))
    self._prm_tasks.setdefault(session_id, {})[turn_num] = task
```

**`finalize_ready_turns` 行为说明**(代码块**下方**,非源码注释):

- `True`(默认):PRM 完成后 `_on_prm_done` 会**接着**调 `_maybe_finalize_ready_turns(session_id)`——检查该 session 所有 pending turn,看是否能往下走 finalize 步骤(详 §4.4)。**正常路径**。
- `False`:PRM 完成后调 `_on_prm_done_record_only`,**只**写 `prm_scores.jsonl` 和 turn record,**不**调 `record_feedback`,**不**调 `_maybe_finalize_ready_turns`。**仅** session 关闭流程使用(`_close_session` 末尾给"还没评的 pending turn 强制起 PRM"时)——见 14 章 §3.2。

**`turn_num` 是 1-based**:`_next_user_turn_num(session_id)`(`api_server.py:3095`)每次 `+= 1`,从 1 开始;`_user_turn_counts[session_id]` 1-based,只在 user turn boundary 时自增(14 章 §5)。`turns[turn_num - 1]` 是 0-based list 索引。

**`next_state` 必传**:`_flush_pending_record(session_id, next_state)`(14 章 §3.1)在每条 user turn 末尾调;`next_state = None` 表示"这是 session 最后一条,不再触发新 PRM,只 flush 当前 record"。

### 4.2 `_on_prm_done`:PRM 完成 callback

```python
# api_server.py:_on_prm_done(关键 9 行,真源码 12 行详 api_server.py:2268)
def _on_prm_done(self, session_id, turn_num, task):
    if task.cancelled(): return
    try: prm_result = task.result()
    except Exception: return                 # PRM 异常: prm_score 留 None(详 §10.1 第 7 条)
    self._apply_prm_result(session_id, turn_num, prm_result)
    if session_id in self._closing_sessions: return    # session 关闭过程中不触发 finalize
    self._maybe_finalize_ready_turns(session_id)
```

**`_on_prm_done_record_only`** 与之类似但**不**调 `_maybe_finalize_ready_turns`(session 关闭时用,避免在关闭过程中再触发新 task)。

### 4.3 `_apply_prm_result`:把分数写回 SkillManager(真源码 20 行,`api_server.py:2244`)

```python
def _apply_prm_result(self, session_id, turn_num, prm_result):
    score = (prm_result or {}).get("score", 0.0)                    # 评失败默认 0.0
    idx, turns = turn_num - 1, self._session_turns.get(session_id, [])
    if 0 <= idx < len(turns):                                       # 两边都回灌:注入组 + 读取组
        turn = turns[idx]; turn["prm_score"] = score                # 关键:写 turn record
        sm = self.skill_manager.record_feedback
        sm(turn.get("injected_skills", []), score)                  # 注入组
        read = [r["skill_name"] for r in turn.get("read_skills", []) if isinstance(r, dict)]
        if read: sm([n for n in read if n], score)                  # 读取组
    pending = self._pending_turn_data.get(session_id, {}).get(turn_num)
    if isinstance(pending, dict): pending["prm_result"] = prm_result
```

**两边都回灌 feedback**(`record_feedback` 详 15 章 §7):

- `injected_skills`:这一轮被注入到 prompt 的 skill(**所有**注入的——即使模型没读)。
- `read_skills`:这一轮**实际被读**的 skill(命中 `_READ_TOOL_NAMES` / `_HERMES_SKILL_READ_TOOL_NAMES`)。

> **设计哲学(本节定论,行内交叉见 §10 第 4 条)**:`injected_skills` 和 `read_skills` 是**两类独立事件**,被分别记分才能正确推 effectiveness。
> - 注入但没读 → "agent 看了但认为不重要"的**弱信号**(可能反映 `description` 写得不够清晰,值得后续 OPTIMIZE_DESC)
> - 读了但 PRM 评 -1 → "用了但失败"的**强信号**
>
> **重写时按"两边都回灌"实现**。如未来发现 `injected_skills` 拖累 effectiveness(极端 case:某 skill 频繁被注入但很少被读、被 PRM 评 -1),再**收敛**到 `read_skills` only——这条迭代路径明确,**不**是设计哲学本身待核。

**`turns[idx]["prm_score"] = score`**:把分数写进 session 完整 record——**这一步非常关键**,session 上传到 `~/.skillclaw/sessions/<sid>.json` 时,evolve server 在 LLM 摘要时看到这个分数。

### 4.4 `_maybe_finalize_ready_turns`(真源码 28 行,`api_server.py:3267`)

```python
def _maybe_finalize_ready_turns(self, session_id):
    prm_tasks = self._prm_tasks.setdefault(session_id, {})
    pending = self._pending_turn_data.get(session_id, {})
    for turn_num in sorted(pending.keys()):
        prm_task = prm_tasks.get(turn_num)
        if self.config.use_prm and self.prm_scorer:
            if prm_task is None or not prm_task.done(): continue  # 等 next_state / PRM 还在跑
        turn_data = pending.pop(turn_num)              # ready → pop 出 pending
        prm_result = turn_data.pop("prm_result", None) or _safe_task_result(prm_task)
        self._safe_create_task(self._finalize_turn_feedback(turn_num, turn_data, session_id, prm_result))
```

**"ready" 的定义**:

- `use_prm=False`:整段 `if` 短路,**每个 turn 立刻 ready**(PRM 不参与)。
- `use_prm=True, prm_scorer=None`:同(没 PRMScorer 实例化 → 当作 `use_prm=False`)。
- `use_prm=True, prm_scorer is not None`:必须 `prm_task` 已 done 且 `prm_result` 拿到。

`_finalize_turn_feedback`(`api_server.py:3297`)写 `prm_scores.jsonl`(`_append_prm_record`,每行 `{session_id, turn, score, votes}`)并 `logger.info` 一行"finalized turn"。

---

## 5. PRM 客户端构造

PRM 用哪条 LLM 客户端由 `prm.provider` 决定(`SkillClawConfig.prm_provider`,11 章 §配置系统)。两个合法值:

- `"openai"`(默认):用 `openai.OpenAI` SDK 调 OpenAI 兼容 `/v1/chat/completions` 端点。
- `"bedrock"`:用 `BedrockChatClient` 调 AWS Bedrock Converse 接口。

### 5.1 OpenAI 客户端路径

`PRMScorer.__init__`(`prm_scorer.py:144`)只在 `llm_client is None` 时构造 OpenAI 客户端,否则直接复用注入的 `llm_client`(Bedrock 路径用):

```python
# 构造(有 llm_client 注入 → 不构造 OpenAI client)
if llm_client is not None:
    self._client = llm_client
else:
    try:
        from openai import OpenAI
    except ImportError as e:
        raise ImportError("PRMScorer requires the 'openai' package. ...") from e
    client_kwargs = {"api_key": api_key, "base_url": prm_url.rstrip("/")}
    self._client = OpenAI(**client_kwargs)
```

**`api_key=None` 或 `api_key=""` 的处理**:`api_key=api_key or "..."` 这种 fallback 写法是**不存在**的——`api_key=""` 时 `OpenAI(api_key="", base_url=...)` 会被 SDK 接受(不抛错),但**实际请求时会因 401 失败**。如果用户跑本地 vLLM / 不需要 auth 的端点,直接 `api_key=""` 即可;否则**必须**传真实 key。生产里走 wizard 步骤 14(11 章 §配置系统 表格)用 `getpass` 收集,**不会**留空。

`_query_once`(`prm_scorer.py:209`)用 `asyncio.to_thread` 把同步 SDK 包成异步(事件循环不阻塞):

```python
completion = await asyncio.to_thread(
    self._client.chat.completions.create,
    model=self.prm_model,
    messages=messages,
    temperature=self.temperature,
    max_completion_tokens=self.max_new_tokens,
)
content = completion.choices[0].message.content or ""
return _parse_prm_score(content), content
```

### 5.2 Bedrock 路径

`SkillClawLauncher._run` 检测 `prm_provider == "bedrock"` 且 `prm_model` 非空时,构造 `BedrockChatClient`:

```python
from .bedrock_client import BedrockChatClient
prm_client = BedrockChatClient(model_id=prm_model, region=cfg.bedrock_region)
prm_scorer = PRMScorer(llm_client=prm_client, ...)
```

**`openai` 包的依赖行为**:`PRMScorer.__init__` 收到 `llm_client != None` 时**直接走第一分支**,**不**执行 `from openai import OpenAI` 这条 import(`if/else` 互斥)。但 `import openai` 这条语句本身在**模块顶层没**写——`try/except ImportError` 在 `__init__` 函数体内,所以**仅当走到"无 `llm_client` 且无 openai"分支时才报错**。**这意味着**:用 Bedrock 路径(`llm_client=prm_client` 注入)时,**`openai` 包可以完全没装**——Bedrock 部署不依赖 `openai`。

> **注**:`openai` 仍是 SkillClaw 其它模块的依赖(例如 `validate_skill_evaluator` 在 21 章),纯 Bedrock 部署的兼容性看其它模块的 import 策略,不是本章范围。

### 5.3 `BedrockChatClient`

`bedrock_client.py`(202 行)是**基于 boto3 SDK** 的 AWS Bedrock Runtime Converse 客户端,提供 OpenAI Chat 兼容接口(`chat.completions.create`)——给 `PRMScorer` 和 `api_server._forward_to_llm_bedrock` 用:

- 构造 `boto3.client("bedrock-runtime", region_name=region)`,**直接走 boto3 SDK** 调 Converse API。
- 提供 `_ChatCompletions` / `_Chat` 包装,签名仿 OpenAI:`chat.completions.create(model=, messages=, temperature=, max_completion_tokens=, ...) -> response`。
- 内部把 OpenAI Chat 格式 `{"messages": [...]}` 转 Bedrock Converse 格式 `{"messages": [...], "system": [...]}`;`tool_calls` 用 Bedrock 的 `toolConfig` 字段;`streaming` 也支持(虽然 PRM 不用 stream,但 `api_server._forward_to_llm_bedrock` 用)。

**为什么需要这层包装**:boto3 Converse 接口和 OpenAI Chat 格式有差异(消息结构、system 字段位置、tool 字段名),让 `PRMScorer` 和 `_forward_to_llm_bedrock` 不用为 Bedrock 写两套调用代码。

**认证**:不需要 API key——`BedrockChatClient` 走 IAM role credentials(boto3 默认从 instance profile / 环境变量 / `~/.aws/credentials` 读)。

## 6. PRM 客户端与 LLM 客户端的"区分"

**这是 SkillClaw 的一个关键设计选择**:PRM 和 LLM 转发**可以**用**不同的端点**。通常:

- LLM 转发:用"便宜快"的模型(kimi-k2.5 / gpt-4o-mini / haiku)
- PRM:用"强但慢"的模型(gpt-5.2 / claude-sonnet-4-6 / doubao-pro)

`SetupWizard` 支持独立配置(11 章 §配置系统 步骤 12-14):

```yaml
llm:
  model_id: kimi-k2.5         # 转发的 LLM
prm:
  provider: openai            # 或 bedrock
  url: https://api.openai.com/v1
  model: gpt-5.2              # PRM 用 gpt-5.2
  api_key: ...
```

`to_skillclaw_config`(11 章 §配置系统)把 `prm.url` / `prm.model` / `prm.api_key` 解析为 `SkillClawConfig.prm_url` / `prm_model` / `prm_api_key`,**没填时**回退到 LLM 配置(顺序:`prm.url or llm.api_base`,`prm.model or llm.model_id or "gpt-5.2"`,`prm.api_key or llm.api_key`)——保证最少配置能跑。

## 7. 反馈到 SkillManager 的全链路

```
PRM 评完
  ↓
_on_prm_done callback (或 _on_prm_done_record_only for session 关闭)
  ↓
_apply_prm_result(session_id, turn_num, prm_result)
  ↓
score = prm_result["score"]   # ∈ {-1.0, 0.0, 1.0}
  ↓
turns[turn_num-1]["prm_score"] = score    # 写 session turn record
  ↓
self.skill_manager.record_feedback(injected_skills, score)
self.skill_manager.record_feedback(read_skills, score)
  ↓
# SkillManager.record_feedback / record_injection(15 章 §7):
SkillManager._stats[name].positive_count += 1   (score > 0)
SkillManager._stats[name].negative_count += 1   (score < 0)
SkillManager._stats[name].neutral_count  += 1   (score == 0)
SkillManager._stats[name].effectiveness = positive_count / inject_count
  ↓
_maybe_flush_stats ← 累计 10 次 mutation 写一次 skill_stats.json
```

**`inject_count` 由谁更新?(R3 评审点名)**: `inject_count` **不**是 PRM 路径更新的——它在 `record_injection(skill_names)`(`skill_manager.py:244`)里 `+= 1`,**触发点是"skill 被注入到 prompt"**(由 `_inject_skills` 调,15 章 §5)。**PRM 路径只更新 `positive_count` / `negative_count` / `neutral_count`,**不**动 `inject_count`**。所以:

- `inject_count` 增 → "这个 skill 被用了一次"
- `positive_count` 增 → "这个 skill 评 +1 了一次"
- `effectiveness = positive_count / inject_count` ∈ [0, 1]

**举例**:某 skill 注入 10 次、PRM 评 6 次 +1 / 2 次 -1 / 2 次 0:`inject_count=10`, `positive_count=6`, `effectiveness=0.6`。

**`session_upload_interval > 0` 周期化触发**时也会**主动**调 `_trigger_evolve` → `POST {evolve_server_url}/trigger` → evolve server 跑一次 cycle(14 章 §3.4-3.5、20 章 §共享存储与同步)。**这条与路径 A 是独立的**——`_trigger_evolve` 不等 PRM,直接发;如果 PRM 还没评完,`prm_score` 还没写进 turn record,evolve server 看到的 session 里那条 turn 的 `prm_score` 是 `None`,后续 PRM 评完**也**只影响本地 effectiveness,**不会**回填到已上传的 session。

## 8. PRM 与 skill 演化的"闭环"

`skill_stats.json` 在 push 时被读(`cli.py:skills_push`):

```python
stats_path = os.path.join(cfg.skills_dir, "skill_stats.json")
if os.path.exists(stats_path):
    stats = json.load(open(stats_path))
    skill_filter = {
        "stats": stats,
        "min_injections": cfg.sharing_push_min_injections,        # 默认 5
        "min_effectiveness": cfg.sharing_push_min_effectiveness,  # 默认 0.3
    }
result = hub.push_skills(cfg.skills_dir, skill_filter=skill_filter)
```

`SkillHub.push_skills` 内部按 filter 过滤:`inject_count < min_injections` 或 `effectiveness < min_effectiveness` 的 skill **不上传**(保留本地)。**这是 SkillClaw "skill 演化反馈"的客户端实现**——PRM 数据影响推送给团队的 skill 选择。

**演化端怎么用 effectiveness**:`evolve_server/pipeline/execution.py` 构造 LLM prompt 时**不直接看 effectiveness 数值**——但 `_finalize_turn_feedback` 把 `prm_score` 写进 session turn record(§4.3),evolve server 看到的 session 里包含每条 turn 的 PRM 分(在 `_trajectory` / `_summary` 之后 LLM 摘要时会作为上下文的一部分),**由 LLM 决定如何权衡**(`effectiveness` 的消费方是"LLM 摘要",不是固定规则)。这条"LLM 自由解读"的链路在 31 章 §3 Stage 2 Summarize 与 §5 Stage 4 Aggregate 详述。

`manifest.jsonl` 每行是一个 skill 推送记录,带 `effectiveness` / `positive_count` / `negative_count` / `neutral_count` / `inject_count` / `uploaded_by`(=`sharing_user_alias or $USER`)字段——**仅记录,不被 evolve server 消费**(20 章 §共享存储与同步 详)。

## 9. PRM 行为调参

| 行为 | 字段 | 默认 | wizard 步骤 | 调参提示 |
|---|---|---|---|---|
| 启用 | `use_prm` / `prm.enabled` | `True` | 11 | 关掉后 PRM 不跑,turn record 的 `prm_score` 字段是 `None` |
| Provider | `prm_provider` / `prm.provider` | `"openai"` | 14.5 | `"bedrock"` 时用 `BedrockChatClient` |
| 票数 | `prm_m` / `prm.m` | `3` | — (wizard 不问) | 1=无投票;5=慢但稳;**推荐奇数**;0=静默 0.0(详 §10 第 1 条) |
| Temperature | `prm_temperature` / `prm.temperature` | `0.6` | — (wizard 不问) | 评估任务通常 0.0~0.2 更稳(详 §10 第 3 条) |
| Max tokens | `prm_max_new_tokens` / `prm.max_new_tokens` | `1024` | — (wizard 不问) | 实际只需 50-200;1024 留余量(详 §10 第 2 条) |
| URL | `prm_url` / `prm.url` | **未设置时**回退 `llm.api_base` | 12 | 配置项缺失 / 值是空字符串 → 走回退;**"未设置"指 YAML 里这个 key 不存在** |
| Model | `prm_model` / `prm.model` | **未设置时**回退 `llm.model_id or "gpt-5.2"` | 13 | 同上,key 不存在 → 回退 |
| API key | `prm_api_key` / `prm.api_key` | **未设置时**回退 `llm.api_key` | 14 (getpass) | 同上 |
| 上传间隔 | `sharing_session_upload_interval` / `sharing.session_upload_interval` | `0`(关闭) | 19(若 sharing 启用) | 单位:user turn 数;`> 0` 时每 N 个 user turn 触发一次 evolve cycle |
| Push 阈值 | `sharing_push_min_injections` / `sharing.push_min_injections` | `5` | — (wizard 不问) | 推送给团队的 skill 最少注入次数 |
| Push 阈值 | `sharing_push_min_effectiveness` / `sharing.push_min_effectiveness` | `0.3` | — (wizard 不问) | 推送给团队的 skill 最低 effectiveness |
| User alias | `sharing_user_alias` / `sharing.user_alias` | **未设置时**回退 `$USER` 环境变量 | 18 | 写入 `manifest.jsonl` 的 `uploaded_by` 字段 |

> **"未设置" / "(空)" 的精确含义**(R3 评审点名):YAML 里这个 key **不存在**(`config_store._normalize_*` 走 `dict.get("key", default)` 拿到 default)。**不**是值是空字符串 `""`,也**不**是值是 `null` / `~`——后者会被 `int("")` / `float("")` 报 `ValueError`,由 `ConfigStore.load()` 在加载时报错。换言之,本表"未设置"列等价于"YAML 缺这行";如果你看到配置 dump 里这行值是 `""` 或 `null`,**那是另一个 bug,详 11 章 §配置系统 已知问题**。

---

## 10. 已知设计 / 真正未决

> 本节**只**留"writer 也不能下笔 / 必须等 openbook 验证"的事项标 `[待核-R4]`;**已知 bug / 已知设计权衡**直接给结论,不带标签。

### 10.1 已知设计权衡(已下结论,无标签)

1. **`prm_m=0` 静默 0.0**:当前实现**不**加 `assert prm_m >= 1`(`prm_m` 是普通 int,不是 enum,加 assert 反而破坏用户配置加载);`_maybe_finalize_ready_turns` 在 `prm_m=0` 时 `asyncio.gather(*[])` 返空、`_majority_vote([])` 返 0.0,所有 turn 的 `prm_score` 都是 0.0。**生产静默故障的兜底放在配置层**:`setup_wizard` 步骤 11 / `config.yaml` 校验加 `prm_m >= 1`,**不**放在 `PRMScorer.__init__`(详 11 章 §配置系统 字段校验)。**重写时按这个分层实现**——`PRMScorer` 不报,配置层报。

2. **`prm_max_new_tokens=1024` 偏大**:PRM 输出"thought + `Score: 1`"在 50-200 token 之内,1024 浪费约 5x token 成本。**当前值是有意的"容错"**——某些 LLM(尤其 Claude)会先写 thought 再给分,1024 留余量避免截断。**优化路径**:不改默认,让用户在 `config.yaml` 调到 200;不要直接砍到 200(会有 LLM 跑截断)。

3. **`prm_temperature=0.6` 偏随机**:评估任务通常 0.0~0.2 更稳。0.6 让"同一条 response 不同票数"出现概率上升——这恰好是 `prm_m=3` 多数票想压的方差源。**当前值是默认遗留**(`OpenAI` 推荐 temperature 0.7);`prm_temperature=0.0` 在多数 provider 下能让"同票"概率上升,削弱多票的价值。**结论**:`prm_temperature=0.2` 是更稳的默认,改这个值要 PR(详 11 章 §配置系统 默认值变更流程)。

4. **`injected_skills` + `read_skills` 两边都回灌**(设计哲学,§4.3 已铺):**当前定论是"两边都回灌"**——`injected_skills` 是"注入了",`read_skills` 是"读了",两类独立事件分别计分。**这条不在"待核"列表**——R3 §10 第 4 条原标"待确认是否只对 `read_skills` 回灌更合理",**已被 §4.3 设计哲学段定论取代**。如未来发现 `injected_skills` 拖累 effectiveness(极端 case),收敛到 `read_skills` only 是**迭代路径**,不是设计本身待核。

5. **`_sanitize_text` 把 `<file>` 替为 `[tag]`**:OpenClaw / Hermes 适配器可能 emit 各种 `<file>` / `<output>` XML 块,统一替成 `[tag]` / `[/tag]` 防止 LLM provider content filter 误判。**当前是"宁可错替也不放过"的策略**——`[tag]` 是个无害占位,影响 PRM 看 response 全文时的可读性,**不**影响评分决策。**不**打算精细化"只替 OpenClaw 私有 tag"(投入产出比低)。

6. **Bedrock 路径 + 没装 `openai` 包的兼容性**:`PRMScorer.__init__` 走 `llm_client != None` 分支时不 `import openai`,所以**纯 Bedrock 部署 + `pip install skillclaw[bedrock]`**(不装 `openai` 依赖)能跑。但 `validate_skill_evaluator`(21 章)等其它模块可能顶层 import `openai`——纯 Bedrock 部署需保证**所有走到的路径**都不顶层 import `openai`。**当前** SkillClaw 的 `pyproject.toml` 把 `openai` 列为**核心依赖**,**不**走可选依赖——这条"纯 Bedrock 不装 openai"是**理论可行**,**不是承诺**;真要走需要单独 build。

7. **PRM 异常时 `prm_score` 留 `None`**:`_on_prm_done` 的 `except Exception: return` 让 PRM 任务失败时 turn record 的 `prm_score` 保持原值(初始 `None`)。**当前是有意设计**——evolve server 看到 `prm_score=None` 视作"无评分",不参与 LLM 摘要的 PRM 维度;**不**记 -1(避免把"评失败"和"评 -1"混淆)。session 上传后该 turn 仍正常出现在 trajectory,**只**是 LLM 摘要时少一个 PRM 信号。**重写时按"None = 评失败 / -1 = 评不好"实现**。

### 10.2 真正未决(`[待核-R4]`)

> 这些是"writer 写不动、需要等 openbook 验证或下一次实测"的事项,**不**是设计本身有问题。

- **`prm_m=4` 时 3-1 vs 2-2 的具体 LLM 行为**:§3 表推了 `[1,1,1,-1]→1.0` / `[1,1,-1,-1]→0.0`,但生产里 LLM 经常给"半 +1 半 0"(`[1,1,0,0]`)——`_majority_vote` 把这个算成 1.0(top=1 出 2 次、top=0 出 2 次 → 平票 → 0.0)。**待核-R4**:真实流量里 `[1,1,0,0]` / `[1,0,0,-1]` 之类"非典型平票"的分布,以及是否值得给 `_majority_vote` 加"top 票 vs 次票差 ≥ 2 才算多数"的更强条件。
- **Bedrock `Converse` 流对 `system` 多模态(图片)的兼容性**:`BedrockChatClient` 内部把 OpenAI 格式 `system: "..."` 转 Bedrock `system: [{"text": "..."}]`,**但** Claude 3.5 / Claude 4 在 Bedrock 上 `system` 块支持图片多模态,`BedrockChatClient` 当前**不**识别 `system` 里的 image_url——PRM 不会触发多模态(`_query_once` 只用 text),**但** `api_server._forward_to_llm_bedrock` 转发的请求里如果 `system` 块有图片,**会被丢掉**。**待核-R4**:此条不影响 PRM,影响 LLM 转发——属于 §5.3 范围外。
- **`sharing_user_alias` 在 `setup_wizard` 未运行时的 fallback 顺序**:`config_store._normalize_*` 在 wizard 之前**不**回退到 `$USER`——回退逻辑在 `_load_or_init` 的最后一步(11 章 §配置系统 §3)。**待核-R4**:`pip install skillclaw && skillclaw serve`(没跑过 wizard)是否能让 `manifest.jsonl` 的 `uploaded_by` 正确写到 `$USER`,还是会被 loader 当作空字符串。

---

← **上一篇**：[15 · 技能库与注入](15-技能库与注入.md) — 本章直接消费 `injected_skills` / `read_skills` / `record_feedback` 的输出

→ **下一篇**：[17 · 适配器矩阵](17-适配器矩阵.md) — SkillClaw 怎么把 12 种 CLI agent 的本地配置改写、备份、回滚
