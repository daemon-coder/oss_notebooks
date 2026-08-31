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

> 顺序按"首次出现章位"排;每条一行,需要时回查。**通用术语**(PRM / Effectiveness / Manifest / 共享存储等)的详尽定义见 [02 章 核心概念词典](../00-总览/02-核心概念词典.md),本表只标本章新引入的内部数据 / 函数 / 文件。

- **`PRMScorer`** —— `skillclaw/prm_scorer.py` 的类;对外只暴露 `async evaluate(response, instruction) -> {"score": float, "votes": list, "eval_text": str}`。score ∈ {-1.0, 0.0, +1.0}。
- **`prm_m`** —— `PRMScorer.prm_m`(`prm_scorer.py:149`),int,默认 3。**每次 evaluate 并发发的票数**;最终 score 是这 `prm_m` 票的**多数票**。推荐奇数 `prm_m ∈ {1, 3, 5}`(见 §3)。(PRM 概念详见 [02 章 §10](../00-总览/02-核心概念词典.md))
- **`_majority_vote(scores)`** —— `prm_scorer.py:99` 的纯函数。输入 `list[Optional[int]]`,返回 float;**平票或全 None → 0.0**;否则返回 top 票的值(1.0 / 0.0 / -1.0)。**不**是均值,不是中位。
- **`_READ_TOOL_NAMES`** —— `api_server.py:119` 的常量 `{"read", "file_read", "read_file", "readfile"}`。**反向**用于 `_extract_read_skills_from_tool_calls` 识别"模型实际读了哪个 skill"(详 15 章 §6)。与 prompt 里提示模型用的 `read_tool_name` 是**两条独立机制**(15 章 §6.1)。
- **`_HERMES_SKILL_READ_TOOL_NAMES`** —— `api_server.py:120` 的常量 `{"skill_view"}`。Hermes 适配器专用,与上一条配合(详 15 章 §6)。
- **`next_state`** —— `dict` 形如 `{"role": "user", "content": ...}`,表示"下一条 user message"。**PRM 必须等到 `next_state` 进来才评上一条回答**——需要"对照下一条 user 提问"判断回答有没有用。`None` 表示"这是 session 最后一条,不需要评"。14 章 §3.1 解释 `_flush_pending_record(session_id, next_state)` 时如何传。
- **`drain`** —— `_close_session` 末尾的"等 PRM 跑完"动作,具体是 `await asyncio.wait_for(asyncio.gather(*active_prm_tasks, return_exceptions=True), timeout=_SHUTDOWN_DRAIN_TIMEOUT_SECONDS)`,**默认 15 秒**(`api_server.py:2081-2086`)。超时后未完成的 PRM task 走 `prm_result = None` 的 fallback finalize。14 章 §3.2 已铺。
- **`_closing_sessions`** —— `api_server.py:1464` 的 `set[str]`;`_close_session` 开头 `add`,`finally` 块 `discard`(14 章 §3 末段)。用于 PRM callback 写回 turn record 前 check,避免"session 已关还在写"。
- **`_pending_turn_data`** —— `api_server.py:1458` 的 `dict[str, dict[int, dict]]`;外层 key 是 `session_id`,内层 key 是 `turn_num`(1-based),value 是 `{response_text, prompt_text, injected_skills, read_skills, has_next_state, prm_result?}`。**每条 turn 的"待 finalize 数据"**。
- **`_prm_tasks`** —— `api_server.py:1459` 的 `dict[str, dict[int, asyncio.Task]]`;外层 session、内层 turn(1-based),value 是 PRM 的 `asyncio.Task`。`_close_session` 时遍历 drain。
- **`prm_scores.jsonl`** —— PRM 评完一条 turn 时追加一行的本地记录文件,路径 `~/.skillclaw/sessions/<sid>/prm_scores.jsonl`(实际由 `record_dir` + `prm_scores.jsonl` 决定,`api_server.py:1498`)。**每行** JSON 形如 `{"session_id": ..., "turn": N, "score": float, "votes": [...]}`,**不带** eval_text(`api_server.py:2181-2199`)。**只用于审计 / 离线分析**,**不**被 evolve server 消费(20 章 §共享存储与同步)。(Record 文件概念详见 [02 章 §25](../00-总览/02-核心概念词典.md))
- **`manifest.jsonl`** —— 团队 push 时的 skill 推送记录(20 章 §共享存储),每行一个 skill 条目,带 `effectiveness` / `positive_count` / `negative_count` / `neutral_count` / `inject_count` / `uploaded_by` / `last_injected_at` 字段。**仅记录**,**不**被 evolve server 消费。(Manifest 概念详见 [02 章 §19 / §20](../00-总览/02-核心概念词典.md))
- **`injected_skills` / `read_skills`** —— `_apply_prm_result` 把 score 派给这两组 skill 各自调 `SkillManager.record_feedback`。
  - `injected_skills: list[str]`(skill 名字;15 章 §5)
  - `read_skills: list[dict]`,每项 `{"skill_name": str, "path": str, ...}`(15 章 §6)
  - 15 章 §7.1 已铺。
- **`_apply_prm_result` / `_on_prm_done` / `_on_prm_done_record_only`** —— PRM task done callback 链路中的三个关键方法(`api_server.py:2244 / 2268 / 2281`)。
  - `_on_prm_done`:正常路径,score 写回 + 调 `record_feedback` + 检查 `readiness` 触发 finalize。
  - `_on_prm_done_record_only`:session 关闭路径,只写 `prm_scores.jsonl` 和 turn record,**不**调 `record_feedback`(避免在关闭过程中再次改 `_stats`)。
- **`session_upload_interval`** —— `SkillClawConfig.sharing_session_upload_interval`(11 章 §配置系统,单位:user turn 数,默认 `0` = 关闭)。`> 0` 时**每 N 个 user turn 一次**触发 `_maybe_upload_session_snapshot`(周期化路径 B,详 §4 块引用)。
- **`_trigger_evolve`** —— `SkillClawAPIServer._trigger_evolve(self) -> None`(`api_server.py:3121`),**无入参**——`evolve_server_url` 从 `self.config.evolve_server_url` 读;向 evolve server 发 `POST {evolve_server_url}/trigger`(httpx 异步 client,3 次重试,timeout 300s)。
- **`inject_count`** —— `SkillManager._stats[name]["inject_count"]`(`skill_manager.py:251`),在 `record_injection(skill_names)` 时 `+= 1`(15 章 §7)。**和 PRM 无关**——`inject_count` 由"注入"触发,不是由"评分"触发。`effectiveness = positive_count / inject_count`,分母是"被注入次数",分子是"被评 +1 的次数"(详 §7)。(Effectiveness 概念详见 [02 章 §11](../00-总览/02-核心概念词典.md))
- **`use_prm`** —— `SkillClawConfig.use_prm: bool`(11 章 §配置系统,默认 `True`)。`False` 时 `_maybe_finalize_ready_turns` 整段 `if` 短路,每个 turn 立即 finalize,不等 PRM(详 §4.4)。

---

## 1. PRM 的本质

`PRMScorer`(`prm_scorer.py`)对外**异步**暴露 `async evaluate(response, instruction)`,内部用 `asyncio.to_thread` 把 sync SDK 调成非阻塞(见 §5.1)。`openai.OpenAI` 是 sync 客户端,**不是**异步客户端——这种"async 外壳 + sync 内核"的模式让 PRM 不阻塞主请求流的事件循环。

它**不**调 SkillClaw 上游的 LLM——它有自己的 LLM 客户端,可以是 OpenAI / Together / Fireworks / 自建 vLLM / 任何 `/v1/chat/completions` 兼容端点,或 AWS Bedrock(走包装的 `BedrockChatClient`,详 §5.2)。

打分流程:PRM 在 `evaluate(response, instruction)` 入口拿用户指令和模型回答,先并行发 `prm_m` 票(默认 3 票,每票都是同一条 prompt 给上游 LLM),每票解析出 `Score: 1` / `Score: -1` / `Score: 0` 三种之一(fallback 兼容历史 `\boxed{N}` 格式,详 §2 末尾),把这 `prm_m` 个分数喂给 `_majority_vote` ——平票或全 None 时返 0.0,否则返 top 票值,最终返回 `{score: float, votes: [int|"fail", ...], eval_text: str}`。

**为什么用多票**?单次 LLM 打分有**随机噪声**——模型可能因一两句话的语气、措辞、标点给 +1 或 -1,与真实质量无关;在 evaluation 任务上这是已知现象(RM 论文常用 "self-consistency" 缓解)。多票多数票能显著降低方差。

## 2. PRM 打分 prompt

`_build_prm_judge_prompt` 在 PRM 评分模块里负责把 instruction + response 拼成两段消息:一条 system(角色 + 关键约束 + 评分规则) + 一条 user(instruction 段 + response 段)。完整 13 行字符串字面量,真实入口在 prm_scorer.py 评分模块。

**system 段的关键约束**有两道——都写进 system prompt 防 LLM 跑偏:

- "Do NOT compare against any follow-up turn"——只看 response 对 instruction 的完成度,不看后续
- "End your reply with exactly one of: Score: 1 / Score: -1 / Score: 0"——强制结构化输出

**评分规则**(同样在 system 段里):

- **+1** —— response 明显遵循 instruction 并基本完成
- **-1** —— response 跑题、错误或核心需求没完成
- **0** —— 完成度模糊或证据不足

**user 段** 直接拼 `Instruction:\n{clean_instruction}\n\nResponse:\n{clean_response}\n\n...` 三段文本,最后一段是给 LLM 的"按上面规则给分,结尾输出 Score: N"指令。

**`\boxed{N}` 旧格式说明**:`_parse_prm_score`(在 PRM 评分模块)主路径解析 `Score: N`,**fallback 路径**解析 `\boxed{N}`。后者是历史版本 / DeepSeek-Math 等数学论文里用过的输出格式——某些自托管 LLM 训练数据里仍偏好这种格式,留作 fallback 兼容(不抛错)。**当前推荐**用 `Score: N` 走主路径,`\boxed{N}` 仅作兜底。

### 2.1 `_sanitize_text`:在送 LLM 前清洗

`_sanitize_text`(在 PRM 评分模块的常量区)走三条 regex 串行清洗输入文本,防上游 content filter 误触:

1. **第一条 regex 专门匹配 `<tool_call>...</tool_call>` 整段块**——用 DOTALL 模式跨行匹配,替成 `[tool_call block]`。
2. **第二条 regex 匹配任何"以字母 / 下划线开头、长度 ≤80"的尖括号 tag**——替成 `[tag]`。
3. **第三条 regex 匹配对应的闭合 tag**——替成 `[/tag]`。

**为什么第一条 regex 专门匹配 `<tool_call>` 块**?OpenClaw / Hermes 适配器把工具调用序列化为 `<tool_call>...</tool_call>` 块——这种"看起来像 prompt injection"的内容会让 Azure / 部分 vLLM 部署拒收请求。`[tool_call block]` 是一个对 LLM 友好的中性占位,让 PRM 看到"这里有工具调用发生过"但不会被 provider 拒。

**剩下两条** regex 把任何 `<tag>` / `</tag>` 替成 `[tag]` / `[/tag]`——同样防 content filter 误判。**不是** 替换所有尖括号(只替换"以字母 / 下划线开头的 tag"——数值、路径等不会误伤)。

> **注**:`_sanitize_text` 是按**半角**尖括号 + 半角字母正则的;早期草稿里误写为全角 `\uff08` / `\uff09`,**真源码里没有全角字符**,照源码写即可。

## 3. 多票多数票

PRM 多票并行走的流程是:在 `evaluate()` 里用 `asyncio.gather` 并行起 `prm_m` 个 `_query_once` 调用(每个调用一次上游 LLM),然后从每个结果里抽第一项(每票的 int 分数或 None),把这 `prm_m` 个分数喂给 `_majority_vote` 拿最终 score。

`_majority_vote`(PRM 评分模块的纯函数)的决策流程分四步:

1. **过滤 None**——只保留非空票;全 None 直接返 0.0
2. **计票**——用 `collections.Counter` 统计每种分值出现次数
3. **找 top**——`most_common(1)[0]` 拿出现次数最多的分值(以及它的票数)
4. **判平票**——如果 top 票数出现 > 1 次(即有 ≥ 2 种分值都达到 top 票数),返 0.0;否则把 top 分值转 float 返回

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

**场景**:Turn N 进来时,SkillClawAPIServer 需要决定"要不要为 turn N-1 触发 PRM 评分"。如果上一轮回答有用,评 +1;没用评 -1;平庸评 0。

**为什么是"下一条 user message 进来"才评**:PRM 的语义是"评**上一个回答对下一个问题有没有用**"——turn N 还没进来时评 turn N-1 没意义(没"下一个问题")。这种"延后一格"的设计让 PRM 能用 turn N 的 user message 作为评分上下文。

**`_fire_prm_scoring` 走 5 步**(为什么是 5 步而不是 1-2 步):

1. **守卫**——`prm_scorer` 为空或 `next_state` 为空就早返回(不评)
   - 为什么:配置层可能没开 PRM(`use_prm=False` 时 `prm_scorer=None`);`next_state=None` 表示这是 session 末尾(详见 `_close_session` §2.5),不评
2. **后台 task 化**——用 `asyncio.create_task` 把 PRM 评分布到后台 task
   - 为什么:PRM 调第三方 LLM 阻塞,如果同步等,Latin agent 主循环会卡住(影响下一条 user message 的处理)
3. **注册 done 钩子**——`task.add_done_callback(_log_prm_task_exception)` 记录异常
   - 为什么:task 抛异常时,event loop 会"未观察的 task exception"——加一个钩子避免污染日志
4. **选 callback**——根据 `finalize_ready_turns` 选 `_on_prm_done` / `_on_prm_done_record_only`
   - 为什么:正常路径走 `_on_prm_done`(会触发 finalize),session 关闭路径走 `_on_prm_done_record_only`(不触发新 finalize,避免 race condition)
5. **挂到 `_prm_tasks[session_id][turn_num]`**——`asyncio.create_task` 之后立刻存
   - 为什么:`_prm_tasks` 是嵌套 dict,外层 session、内层 turn;`_close_session` 时用这个 dict 来 drain 所有 PRM task

**`finalize_ready_turns` 行为说明**:

- `True`(默认):PRM 完成后 `_on_prm_done` 会**接着**调 `_maybe_finalize_ready_turns(session_id)`——检查该 session 所有 pending turn,看是否能往下走 finalize 步骤(详 §4.4)。**正常路径**。
- `False`:PRM 完成后调 `_on_prm_done_record_only`,**只**写 `prm_scores.jsonl` 和 turn record,**不**调 `record_feedback`,**不**调 `_maybe_finalize_ready_turns`。**仅** session 关闭流程使用(`_close_session` 末尾给"还没评的 pending turn 强制起 PRM"时)——见 14 章 §3.2。

**`turn_num` 是 1-based**:`_next_user_turn_num(session_id)` 每次 `+= 1`,从 1 开始;`_user_turn_counts[session_id]` 1-based,只在 user turn boundary 时自增(14 章 §5)。`turns[turn_num - 1]` 是 0-based list 索引。

**`next_state` 必传**:`_flush_pending_record(session_id, next_state)`(14 章 §3.1)在每条 user turn 末尾调;`next_state = None` 表示"这是 session 最后一条,不再触发新 PRM,只 flush 当前 record"。

**关键边界**:

- **`prm_scorer=None`**:守卫早返,这一步不评(避免后续 NPE)
- **`next_state=None`**:守卫早返(避免给"没有下一个问题"的 turn 评)
- **`asyncio.create_task` 在 shutdown 期间**:task 创建后立即被 event loop 取消,`_on_prm_done` 不会跑——需要在 `_close_session` 里显式 drain(详见 §2.5 Step 3)
- **`finalize_ready_turns=False`**:session 关闭时用,避免在关流程中再触发 `_maybe_finalize_ready_turns`(可能引起 race condition)
- **task 内部异常**:第三方 LLM 调失败 / 解析失败,异常被 `add_done_callback` 钩子捕到,score 走 None 兜底(详见 §4.2)

**走完之后**:

- 内存里 `_prm_tasks[session_id][turn_num]` 多了一条 task 引用
- 第三方 LLM 异步评分(0.5-2s 完成,取决于上游)
- task done 后触发 callback,score 写进 turn record + SkillManager + pending dict
- `_maybe_finalize_ready_turns` 在下一个 turn 进来时检查这个 turn 能否 finalize

**`finalize_ready_turns` 行为说明**:

- `True`(默认):PRM 完成后 `_on_prm_done` 会**接着**调 `_maybe_finalize_ready_turns(session_id)`——检查该 session 所有 pending turn,看是否能往下走 finalize 步骤(详 §4.4)。**正常路径**。
- `False`:PRM 完成后调 `_on_prm_done_record_only`,**只**写 `prm_scores.jsonl` 和 turn record,**不**调 `record_feedback`,**不**调 `_maybe_finalize_ready_turns`。**仅** session 关闭流程使用(`_close_session` 末尾给"还没评的 pending turn 强制起 PRM"时)——见 14 章 §3.2。

**`turn_num` 是 1-based**:`_next_user_turn_num(session_id)` 每次 `+= 1`,从 1 开始;`_user_turn_counts[session_id]` 1-based,只在 user turn boundary 时自增(14 章 §5)。`turns[turn_num - 1]` 是 0-based list 索引。

**`next_state` 必传**:`_flush_pending_record(session_id, next_state)`(14 章 §3.1)在每条 user turn 末尾调;`next_state = None` 表示"这是 session 最后一条,不再触发新 PRM,只 flush 当前 record"。

### 4.2 `_on_prm_done`:PRM 完成 callback

**场景**:`_fire_prm_scoring` 在 §4.1 用 `asyncio.create_task` 创建的 PRM 评分 task 这一刻 done 了——`add_done_callback` 注册的 callback 被调用,要把"评分结果"落进系统。

**为什么需要 4 步而不是直接调 `_apply_prm_result`**:

task done 时有 3 种情况:
1. **正常 done**——task 跑完,`task.result()` 拿到 PRM 结果
2. **被 cancel**——比如 `_close_session` 把所有 PRM task 取消(task 还在 in-flight)
3. **抛异常**——第三方 LLM 调失败 / 解析失败 / 网络异常

如果直接调 `_apply_prm_result`,**case 2 拿到 `CancelledError`,case 3 拿到其他异常**——这俩 case 应该静默处理(cancel 是不需要处理的,异常是已经在 `_fire_prm_scoring` 钩子里记录过的),但如果不做 try/except 会污染 event loop 日志。

**怎么走**(4 步,顺序固定):

1. **Cancel 守卫**——`if task.cancelled(): return`
   - 为什么:被 cancel 的 task 不处理——这是与 `_close_session` 协调的语义
2. **try `task.result()`**——`except Exception: return` 静默 return
   - 为什么:异常情况下 turn record 的 `prm_score` 留 None(已经是默认值,不需要额外处理)
3. **调 `_apply_prm_result`**——把 score 写回 turn record + 调 `record_feedback` + 挂 pending dict(详见 §4.3)
4. **Finalize 协调**——如果 session 正在关闭(`_closing_sessions` set)就早返,否则调 `_maybe_finalize_ready_turns` 检查该 session 是否有 turn 进入 ready 状态、可以走 finalize

**为什么第 4 步要"正在关闭就早返"**:`_maybe_finalize_ready_turns` 会触发新的 `_finalize_turn_feedback` task——session 关闭流程中**不应**再触发新 task(避免 race condition)。如果 session 正在关,PRM 落进 turn record + SkillManager 即可,finalize 由 `_close_session` 收尾流程统一处理。

**`_on_prm_done_record_only` 变体**:

```python
# _on_prm_done (默认)
def _on_prm_done(self, task, session_id, turn_num):
    if task.cancelled(): return
    try: prm_result = task.result()
    except Exception: return
    self._apply_prm_result(session_id, turn_num, prm_result)  # 含 record_feedback
    if session_id not in self._closing_sessions:
        self._maybe_finalize_ready_turns(session_id)  # 触发 finalize

# _on_prm_done_record_only (session 关闭时用)
def _on_prm_done_record_only(self, task, session_id, turn_num):
    if task.cancelled(): return
    try: prm_result = task.result()
    except Exception: return
    self._apply_prm_result(session_id, turn_num, prm_result)  # 含 record_feedback
    # 没有 _maybe_finalize_ready_turns ——关闭流程统一 finalize
```

差异:仅最后一步"是否调 `_maybe_finalize_ready_turns`"——`_on_prm_done` 调,`_on_prm_done_record_only` 不调。两者 `_apply_prm_result` 都跑(score 写进 turn record + SkillManager 收分)。

**关键边界**:

- **`task.cancelled()`**:直接返,不会 try `task.result()`(避免 `CancelledError` 污染)
- **`task.result()` 抛异常**:静默 return——异常已在 `_fire_prm_scoring` Step 3 钩子里 logger 记录
- **session 正在关闭** (`_closing_sessions` 包含 sid):早返,**不**调 `_maybe_finalize_ready_turns`
- **多个 PRM task 并发 done**:每个 task 独立调 callback,`_apply_prm_result` 内部用 `self._pending_turn_data[session_id][turn_num]` 互不干扰

**走完之后**:

- turn record 多了一个 `prm_score` 字段
- SkillManager 收到两边都回灌的 feedback
- `_pending_turn_data[session_id][turn_num]` 多了 prm_result dict
- 下个 turn 进来时 `_maybe_finalize_ready_turns` 会检查这个 turn 能否 finalize(若 session 没在关)
- 如果是 session 关闭时,`_close_session` Step 4 会统一 finalize

### 4.3 `_apply_prm_result`:把 PRM 分数落进三个独立的地方

**场景**:第三方 LLM 评完一个 turn,异步 task 拿到 `score ∈ {-1, 0, +1}` + 投票明细。现在需要把这个分数**落进三个独立的地方**——而且这三个落点的**顺序不能换**。

**为什么三个地方**:SkillClaw 把"score"和"skill 效果"分开存,每个地方服务于不同消费方:

| 落点 | 数据结构 | 消费方 | 用途 |
|---|---|---|---|
| **turn record** | `session.turns[idx]["prm_score"]` | evolve server 拉 session 后做摘要时看到 | 给 LLM 摘要的"上下文"——"这个 session 在第 N turn 评了 -1 分" |
| **SkillManager** | `SkillManager._stats[skill_name]` 的 `positive_count` / `inject_count` | 下次 `SkillClawAPIServer` 注入 skill 时读 | 决定 `effectiveness = positive_count / inject_count`——低于阈值就不推送 |
| **pending dict** | `_pending_turn_data[session_id][turn_num]` | `_maybe_finalize_ready_turns` 流程 | 决定这个 turn 能否走 finalize——finalize 要拿完整 prm_result(不止 score,还有 votes / reasoning)写 `prm_scores.jsonl` |

**为什么顺序不能换**:

- turn record 必须在 SkillManager **之前**:如果先调 SkillManager 写失败(比如 OOM),turn record 也没了——evolve server 拉 session 时看不到这个分,该 turn 就被当成"未评",浪费了一次评估信号。
- SkillManager 必须在 pending dict **之前**:如果先挂 pending dict,finalize 流程会看到 score 但看不到 SkillManager 已经收分(实际是已收但记录在别处),可能重复触发 finalize 导致双写。

**SkillManager 的特殊处理——两边都回灌**:

`record_feedback` 不是被调一次,而是被调**两次**——因为 SkillClaw 把"评分"分到两类 skill:

- **注入组**(`injected_skills`)——这一轮被塞进 system prompt 的**所有** skill,**即使模型根本没读**
- **读取组**(`read_skills`)——这一轮模型**实际** `read` 了哪个 skill(从 `_READ_TOOL_NAMES` / `_HERMES_SKILL_READ_TOOL_NAMES` 命中)

为什么"注入但没读"也要记分:这是"agent 看了 description 但认为不重要"的**弱信号**——可能反映 skill 的 `description` 写得不够清晰,值得后续 `OPTIMIZE_DESC` 优化。`read_skills` 才是"用了且有结果"的**强信号**。两类被分别记分才能正确推 effectiveness。

**怎么走**(六步,顺序固定):

1. 从 `prm_result` 拿 `score` 字段,缺省 0.0
2. `turn_num` 减 1 转 0-based idx
3. 边界 check:idx 落在 session turns list 范围内
4. 写 turn record:`turns[idx]["prm_score"] = score` ——这一步**关键**:session 上传到 `~/.skillclaw/sessions/<sid>.json` 时,evolve server 在 LLM 摘要时看到的就是这个字段
5. 调 `SkillManager.record_feedback` **两边都回灌**:先注入组(`injected_skills`),再读取组(`read_skills`)
6. 把 `prm_result` 挂到 `_pending_turn_data[session_id][turn_num]` ——让 `_maybe_finalize_ready_turns` 能拿到最终结果

**数据流向**(用一次完整 turn 走完看):

```
LLM 异步 task 返回 {score: -1, votes: [-1, -1, +1]}

  ↓ 第 4 步:写 turn record
turns[idx]["prm_score"] = -1   # session 完整证据

  ↓ 第 5 步:SkillManager 两边都回灌
SkillManager._stats["coding-review-checklist"]["inject_count"] += 1
SkillManager._stats["coding-review-checklist"]["positive_count"] += 0
SkillManager._stats["react-hooks-patterns"]["inject_count"] += 1
SkillManager._stats["react-hooks-patterns"]["positive_count"] += 0

  ↓ 第 6 步:挂到 pending dict
_pending_turn_data[session_id][3] = {score: -1, votes: [-1, -1, +1]}

  # 走完之后:
  # - session 完整 record 多了一个字段 `prm_score`
  # - SkillManager 知道"这俩 skill 这次被注入后没人用,效果 -1"
  # - finalize 流程看到 score=-1 + votes(2:1 多数票 -1),准备写 prm_scores.jsonl
```

**为什么需要 `_pending_turn_data` 而不是直接拿 task.result()**:

`_maybe_finalize_ready_turns` 是按 session × turn 维度判断"这个 turn 能不能 finalize"的——它需要拿 PRM 完整结果(不止 score,还有 votes / reasoning / provider 元数据)来写 `prm_scores.jsonl`(`_append_prm_record` 每行 `{session_id, turn, score, votes}`)。把 prm_result 挂到 pending dict 是为了**避免 finalize 时再异步等 PRM task 完**——task 完时已经写入,finalize 直接读。

> **设计哲学(本节定论)**: `injected_skills` 和 `read_skills` 是**两类独立事件**,被分别记分才能正确推 effectiveness。
> - 注入但没读 → "agent 看了但认为不重要"的**弱信号**(可能反映 `description` 写得不够清晰,值得后续 OPTIMIZE_DESC)
> - 读了但 PRM 评 -1 → "用了但失败"的**强信号**

**`turns[idx]["prm_score"] = score`**:把分数写进 session 完整 record——**这一步非常关键**,session 上传到 `~/.skillclaw/sessions/<sid>.json` 时,evolve server 在 LLM 摘要时看到这个分数。

### 4.4 `_maybe_finalize_ready_turns`

**场景**:SkillClawAPIServer 跑完 `_on_prm_done` 后,需要看"该 session 是否有 turn 进入 ready 状态、可以走 finalize"。这是把"PRM 完成"和"PRM 落盘"解耦的中间步骤——`_on_prm_done` 是"完成 callback",`_maybe_finalize_ready_turns` 是"批量检查并触发出 finalize"。

**为什么需要这一步而不是 `_on_prm_done` 直接 finalize**:

- `_on_prm_done` 一次只处理一个 turn 的 PRM 完成事件
- 但**一个 session 可能有多个 turn 在并发跑 PRM**(Turn 2 进来时,Turn 1 的 PRM 还在跑;Turn 3 进来时,Turn 1 / 2 的 PRM 可能都还没完)
- finalize 应该**批量**进行——所有 ready 的 turn 一次性 finalize,避免"每个 PRM 完都触发一次 finalize"的冗余

`_maybe_finalize_ready_turns` 就是把"一个一个 finalize"改成"批量 finalize"的中间步骤。

**怎么走**(3 步,顺序固定):

1. **遍历 session 的所有 pending turn**——从 `_pending_turn_data[session_id]` dict
2. **判定 ready**——3 种情况:
   - `use_prm=False`:整段 `if` 短路,**每个 turn 立刻 ready**(PRM 不参与)
   - `use_prm=True, prm_scorer=None`:同(`prm_scorer` 没实例化 → 当作 `use_prm=False`)
   - `use_prm=True, prm_scorer is not None`:必须 `prm_task` 已 done 且 `prm_result` 拿到
3. **ready 的 turn 触发 finalize**——把 turn_data 从 pending dict 里 pop 出来、拿最终 prm_result(优先取 turn_data 里的,否则从 task 取)、用 `_safe_create_task` 启一个 `_finalize_turn_feedback` 异步 finalize 流程

**"ready" 三种情况的判据**:

```
use_prm=False ────────────────→ ready=True  (PRM 不参与,每 turn 立刻 finalize)
                   │
use_prm=True, prm_scorer=None ──→ ready=True  (没 PRMScorer 实例,等同 use_prm=False)
                   │
use_prm=True, prm_scorer≠None ──→ ready=(prm_task.done() AND prm_result 拿到)
                                     (必须等异步 PRM 完成)
```

**为什么 `_finalize_turn_feedback` 用 `_safe_create_task`**:

`_safe_create_task` 是 SkillClaw 的"安全 create_task"包装——它会检查 event loop 是否在关闭(进程 shutdown 中),如果关闭则不创建 task,避免"`asyncio.create_task() got Future <...> attached to a different loop`" 异常。finalize 是收尾动作,shutdown 时不 finalize 也行(数据已经落进 turn record + SkillManager)。

**`_finalize_turn_feedback` 写什么**:

```python
async def _finalize_turn_feedback(self, session_id, turn_num, prm_result):
    _append_prm_record(  # 写 ~/.skillclaw/prm_scores.jsonl
        session_id=session_id,
        turn=turn_num,
        score=prm_result.get("score", 0.0),
        votes=prm_result.get("votes", []),
        # 一行 JSON: {"session_id": ..., "turn": ..., "score": ..., "votes": [...]}
    )
    logger.info(f"finalized turn {session_id}/{turn_num}")
```

每行写一条 `{session_id, turn, score, votes}`——给 SkillClaw 自己审计用(evolve server **不**读这个文件,只读 session.json 里的 `prm_score`)。

**关键边界**:

- **`_pending_turn_data[sid]` 为空**:遍历空,直接 return(无 turn 可 finalize)
- **PRM task done 但 `prm_result=None`**:task done 但回调里异常被吞了,这种情况 prm_result 是 None——`ready=False`,跳过(避免把空结果当 finalize 触发)
- **session 在 `_closing_sessions`**:这个调用来自 `_on_prm_done` 时已被早返;但如果来自 idle sweeper / 显式 trigger,需要额外判定是否在关
- **多个 turn 同时 ready**:批量 pop 多个,每个都 `_safe_create_task` 启一个 finalize task(并发 finalize,无锁)
- **`_safe_create_task` 拒绝** (event loop 关闭):静默 return,本 turn 的 finalize 不跑(数据已落 turn record,不会丢)

**走完之后**:

- `_pending_turn_data[session_id]` 该 turn 的 entry 被 pop
- 一组 `_finalize_turn_feedback` task 启起来(每个 ready turn 一个)
- 这些 task 各自跑 `_append_prm_record` 写 `prm_scores.jsonl`
- SkillClawAPIServer 主循环不阻塞(fire-and-forget)

---

## 5. PRM 客户端构造

PRM 用哪条 LLM 客户端由 `prm.provider` 决定(`SkillClawConfig.prm_provider`,11 章 §配置系统)。两个合法值:

- `"openai"`(默认):用 `openai.OpenAI` SDK 调 OpenAI 兼容 `/v1/chat/completions` 端点。
- `"bedrock"`:用 `BedrockChatClient` 调 AWS Bedrock Converse 接口。

### 5.1 OpenAI 客户端路径

**场景**:`PRMScorer` 需要一个 LLM 客户端来调第三方"评 PRM"的模型。SkillClaw 允许两种来源:
- `prm_provider=openai`(默认)——用 `openai.OpenAI` SDK 调 OpenAI 兼容 `/v1/chat/completions` 端点
- `prm_provider=bedrock`——用 `BedrockChatClient` 调 AWS Bedrock Converse 接口

这两种 LLM 客户端在 PRM 评分里**接口完全一致**——都是 OpenAI Chat 格式(`chat.completions.create(...)`),这样上层 PRM 评分逻辑不需要为不同 provider 写两套。

**`PRMScorer.__init__` 走二选一分支**:

```
if llm_client is not None:
    # Bedrock 路径:llm_client 由 SkillClawLauncher._run 注入
    self._client = llm_client
else:
    # OpenAI 路径:自建 client
    try:
        from openai import OpenAI
    except ImportError as e:
        raise ImportError("需要 openai 包: pip install openai") from e
    client_kwargs = {
        "api_key": api_key,
        "base_url": prm_url.rstrip("/"),  # 去掉末尾 / 防止 404
    }
    self._client = OpenAI(**client_kwargs)
```

**为什么是二选一不是 if/elif/else**:

- 注入的 `llm_client` 已经是"准备好"的 client,直接用
- 没注入就要自建——`from openai import OpenAI` 写在 `__init__` 函数体内,**只在没走注入分支时才执行 import**

这意味着:用 Bedrock 路径时,`openai` 包**可以完全没装**——`__init__` 不会触发 import,不会 ImportError。

**`api_key=None` 或 `api_key=""` 的处理**:

SkillClaw **不**做 `api_key = api_key or "dummy"` 这种 fallback——

- `api_key=""` + `base_url=https://local-vllm:8000`:SDK 接受(不抛错),实际请求时本地 vLLM 不要求 auth,正常工作
- `api_key=""` + `base_url=https://api.openai.com`:SDK 接受,实际请求时**401 Unauthorized**——因为 OpenAI 强制要求 API key

**生产部署**:wizard 步骤 14(11 章 §配置系统)用 `getpass` 强制收集 API key,**不会**留空——避免上面 401 错误。

**`_query_once` 走两段调用**:

```python
async def _query_once(self, prompt: str) -> tuple[int, str]:
    # 段 1:用 asyncio.to_thread 把同步 OpenAI SDK 包成异步
    completion = await asyncio.to_thread(
        self._client.chat.completions.create,
        model=self._model,
        messages=[{"role": "system", "content": PRM_SYSTEM_PROMPT},
                  {"role": "user", "content": prompt}],
        temperature=self._temperature,
        max_completion_tokens=self._max_new_tokens,
    )
    # 段 2:解析响应
    text = (completion.choices[0].message.content or "").strip()
    score = self._parse_prm_score(text)
    return score, text
```

**为什么用 `asyncio.to_thread` 而不是直接 await**:

`openai.OpenAI` SDK 是**同步阻塞**的——`client.chat.completions.create()` 调 HTTP 请求是阻塞 IO。如果直接 await 这个同步函数,会**阻塞 event loop**——其它并发请求(包括 SkillClawAPIServer 主循环)都排队等,导致整条 daemon 卡住。

`asyncio.to_thread` 把同步函数丢进默认 executor(线程池),在后台线程跑阻塞 IO,event loop 继续处理其它任务。SDK 调完后再 await 拿到结果。

**为什么 model / temperature / max_completion_tokens 是构造时确定**:

这三个参数在 `__init__` 时从 `SkillClawConfig` 读,后续 PRM 评分时**不变**。每次 `_query_once` 都用相同参数调同一个模型——保证多次评分的"评分标准"一致(`prm_m=3` 票都来自同一个模型 + 同一 prompt + 同一 temperature)。

**关键边界**:

- **`llm_client` 已注入**:跳过 OpenAI import,直接用注入的 client
- **`from openai import OpenAI` ImportError**:抛带提示语的 `ImportError`("需要 openai 包: pip install openai")——让用户能看懂报错
- **`api_key=""` + OpenAI API**:SDK 接受,请求 401 失败——`prm_scorer` task 抛异常,`_on_prm_done` 钩子记录,score 走 None
- **`prm_url` 末尾 `/`**:`rstrip("/")` 去掉——`https://api.openai.com/v1/` 和 `https://api.openai.com/v1` 是同一个 URL,但 `openai.OpenAI` SDK 内部拼路径时会出 bug
- **`max_completion_tokens=1024` 偏大** (§10.1 设计权衡第 2 条):PRM 评分实际只需要 ~50 tokens,1024 浪费 5x——但 LLM 偶尔 verbose 时截断风险要权衡
- **`temperature=0.6` 偏随机** (§10.1 设计权衡第 3 条):与"多票压方差"设计哲学矛盾(0.6 推高方差,多票想压低方差)——是否改 0.2 留作 §10 待确认

**走完之后**:

- `self._client` 是 OpenAI sync client(Bedrock 路径则是 `BedrockChatClient` 实例,接口相同)
- 后续 `_query_once` 调 `self._client.chat.completions.create(...)` 走 PRM 评分
- PRM 评分在主循环外(异步)跑,event loop 不阻塞

### 5.2 Bedrock 路径

**场景**:用户用 AWS Bedrock 上的 Claude / Llama 模型评 PRM——不想用 OpenAI API(数据合规 / 已有 AWS 账号 / 已有 IAM 权限)。

**Bedrock 路径的设计哲学**:SkillClaw 的 PRM 评分**不直接**调 boto3 Converse API——而是**包一层 OpenAI Chat 兼容接口**给 PRMScorer 用。这样 PRM 评分代码不需要为 Bedrock 写两套。

**怎么走**(`SkillClawLauncher._run` 检测到 `prm_provider == "bedrock"` 时的流程):

```python
# Step 1: 拉 Bedrock 客户端类(lazy import, 避免纯 openai 部署也 import boto3)
from .bedrock_client import BedrockChatClient

# Step 2: 用 model_id + region 实例化
bedrock_client = BedrockChatClient(
    model_id=cfg.prm_model,        # 例如 "anthropic.claude-3-haiku-20240307-v1:0"
    region_name=cfg.bedrock_region # 例如 "us-east-1"
)

# Step 3: 注入给 PRMScorer
PRMScorer(
    llm_client=bedrock_client,  # 走 §5.1 的"有 llm_client 注入"分支
    api_key=None,               # 注入路径不需要 api_key
    prm_url="",                 # 注入路径不需要 base_url
    # ... 其它 PRM 配置
)
```

**`openai` 包的依赖行为**(为什么 Bedrock 部署不需要装 openai):

`PRMScorer.__init__` 收到 `llm_client != None` 时**直接走第一分支**(用注入的 client),**不**执行 `from openai import OpenAI` 这条 import。`if/else` 互斥:

```
PRMScorer.__init__(llm_client=bedrock_client, ...)
  │
  ├─ if llm_client is not None: self._client = llm_client  # 走这里
  │     # 不 import openai
  │
  └─ else: ... (try import openai, raise if missing)
```

但**仅 Bedrock PRM 路径不需要 openai**——`openai` 仍是 SkillClaw 其它模块的依赖(例如 `validate_skill_evaluator` 在 21 章的 `_call_validator_llm` 调 OpenAI SDK 评 skill 验证)。

**纯 Bedrock 部署的兼容性**:

- **PRM 评分**:`PRMScorer(llm_client=bedrock_client)` 走注入分支,不 import openai ✓
- **主 LLM 转发**:`api_server._forward_to_llm_bedrock` 也走 Bedrock 客户端,不 import openai ✓
- **`validate_skill_evaluator`** (21 章):调 OpenAI SDK 评 skill——纯 Bedrock 部署需要确认这个模块的 import 策略(详见 21 章)
- **其它模块**:看具体 import 策略,本章范围外

**关键边界**:

- **`prm_provider="bedrock"` 但 `prm_model=""`**:不走 Bedrock 路径,fallback 到 openai 路径(`api_key` 必须配)
- **`prm_provider="bedrock"` 但没装 boto3**:`from .bedrock_client import BedrockChatClient` 抛 ImportError("需要 boto3 包")
- **AWS IAM 权限不够**:`boto3.client("bedrock-runtime", region_name=region)` 创建 client 不报错,实际调 Converse API 时报 `AccessDeniedException`
- **Bedrock 模型 ID 拼错**:调 Converse API 时报 `ValidationException`——`prm_model` 字段在 wizard 里校验(详见 11 章)

**走完之后**:

- `self._client` 是 `BedrockChatClient` 实例(OpenAI Chat 兼容接口)
- 后续 PRM 评分走 `self._client.chat.completions.create(...)`——**和 OpenAI 路径完全相同的代码**
- `openai` 包**不**被 import——纯 Bedrock 部署可裁剪 openai 依赖(仅 PRM 评分这块)

### 5.3 `BedrockChatClient`

**场景**:`PRMScorer` 和 `api_server._forward_to_llm_bedrock` 都需要调 AWS Bedrock 上的模型——但 boto3 Converse API 跟 OpenAI Chat API 格式不兼容。`BedrockChatClient` 是 SkillClaw 自己写的"翻译层",把 OpenAI Chat 格式调成 boto3 Converse 调。

**`bedrock_client.py` 的整体结构**(202 行):

```python
class BedrockChatClient:
    def __init__(self, model_id, region_name):
        self._client = boto3.client("bedrock-runtime", region_name=region_name)
        self._model_id = model_id
        # chat.completions.create(...) 入口
        self.chat = self._ChatCompletions(self)
    
    def _invoke(self, body: dict) -> dict:
        # 直接走 boto3 Converse API
        return self._client.converse(modelId=self._model_id, **body)
    
    class _ChatCompletions:
        def __init__(self, parent): self._parent = parent
        def create(self, model, messages, temperature, max_completion_tokens, ...):
            body = self._convert_openai_to_bedrock(messages, temperature, max_completion_tokens)
            response = self._parent._invoke(body)
            return self._convert_bedrock_to_openai(response)
    
    @staticmethod
    def _convert_openai_to_bedrock(messages, temperature, max_tokens):
        # OpenAI Chat: [{"role": "system", ...}, {"role": "user", ...}]
        # Bedrock Converse: {"system": [...], "messages": [{"role": "user", ...}]}
        # 拆 system message 出来放顶层
        system = [m["content"] for m in messages if m["role"] == "system"]
        convo = [m for m in messages if m["role"] != "system"]
        return {"system": system, "messages": convo, "inferenceConfig": {"temperature": ..., "maxTokens": ...}}
    
    @staticmethod
    def _convert_bedrock_to_openai(response):
        # Bedrock Converse response: {"output": {"message": {"content": [{"text": "..."}]}}}
        # OpenAI Chat response: {"choices": [{"message": {"content": "..."}}]}
        text = response["output"]["message"]["content"][0]["text"]
        return {"choices": [{"message": {"content": text}, "finish_reason": "stop"}], "usage": {...}}
```

**为什么需要这层包装**:

boto3 Converse API 跟 OpenAI Chat API 有 4 处格式差异:
1. **System message 位置** — OpenAI Chat 放 messages[0],Bedrock Converse 放顶层 `system` 字段
2. **Tool calls 字段** — OpenAI 用 `tools` / `tool_choice`,Bedrock 用 `toolConfig`(`tools` 字段名相同但格式不同)
3. **max_tokens 字段名** — OpenAI 用 `max_completion_tokens`,Bedrock 用 `maxTokens` (在 `inferenceConfig` 里)
4. **Response 结构** — OpenAI 返 `choices[0].message.content`,Bedrock 返 `output.message.content[0].text`

如果没有这层包装,`PRMScorer` 和 `_forward_to_llm_bedrock` 都要写 2 套调用代码——一份给 OpenAI 路径,一份给 Bedrock 路径。`BedrockChatClient` 包装后,**上层只看到 OpenAI Chat 接口**,底层自动翻译。

**关键 API 行为**(`chat.completions.create(...)` 签名):

| 参数 | OpenAI Chat 格式 | 翻译到 Bedrock Converse |
|---|---|---|
| `model` | 模型名 | `modelId` 顶层 |
| `messages` | list of `{role, content}` | 拆出 system,剩下当 `messages` |
| `temperature` | 顶层 | `inferenceConfig.temperature` |
| `max_completion_tokens` | 顶层 | `inferenceConfig.maxTokens` |
| `tools` | list of `{type, function}` | `toolConfig.tools` |
| `stream` | bool | `BedrockChatClient` 也支持(给 `_forward_to_llm_bedrock` 用) |

**为什么 streaming 也支持**:

`api_server._forward_to_llm_bedrock` 是 SkillClawAPIServer 转发主 LLM 请求到 Bedrock 的路径——主 LLM 转发需要 streaming(SSE 给 agent 实时看 token 出来),不能像 PRM 评分那样一次性等。`BedrockChatClient` 的 `_ChatCompletions.create` 看到 `stream=True` 时走 boto3 `converse_stream` API,把 streaming event 转成 OpenAI SSE 格式。

**认证**:

`BedrockChatClient` **不**用 API key——走 AWS IAM role credentials。boto3 默认从以下来源按顺序找 credentials:

1. `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` 环境变量
2. `~/.aws/credentials` 文件(`[default]` profile)
3. EC2 instance profile / ECS task role / EKS service account(IAM role)

**生产部署建议**:用 IAM role(不要在 `~/.aws/credentials` 写死 key)——既安全又自动 rotate。

**关键边界**:

- **`boto3` 没装**:`from .bedrock_client import BedrockChatClient` 抛 ImportError
- **AWS credentials 找不到**:`boto3.client("bedrock-runtime", ...)` 构造时不报错(只是 client 没绑 credentials),实际调 Converse API 时报 `NoCredentialsError`
- **`model_id` 拼错**:调 Converse API 时报 `ValidationException: model identifier is invalid`
- **AWS region 没开通 Bedrock**:调 Converse API 时报 `AccessDeniedException: User is not authorized to perform bedrock:InvokeModel`
- **streaming 与 non-streaming 协议不一致**:`BedrockChatClient` 内部对 `stream=True/False` 走不同 boto3 API(converse vs converse_stream),返回的格式转换也不同
- **System message 多条**:OpenAI Chat 允许多个 system message 散布,Bedrock Converse 的 `system` 字段是 list——`BedrockChatClient` 把所有 system role 的 message 都收进 `system` list

**走完之后**:

- 上层(`PRMScorer` / `_forward_to_llm_bedrock`)只看到 OpenAI Chat 兼容接口
- 底层走 boto3 Converse API
- API 路径不需写 2 套——`BedrockChatClient` 翻译层全包
- AWS IAM 鉴权由 boto3 内部处理,业务代码不接触 credentials

---

## 本章 4 段核心讲解段(§4.1-§4.4)+ 3 段构造段(§5.1-§5.3)整体流程

把 4 段 PRM 数据流串起来看(从评分触发到落盘):

```
Turn N 进来
  ↓
§4.1 _fire_prm_scoring (turn N-1 的 PRM 触发)
  - 5 步:守卫 → create_task → done 钩子 → 选 callback → 挂 _prm_tasks
  ↓
异步 task 跑 _query_once (§5.1 OpenAI 或 §5.2 Bedrock 路径)
  ↓ (task done)
§4.2 _on_prm_done (PRM 完成 callback)
  - 4 步:cancel 守卫 → try result → _apply_prm_result → finalize 协调
  ↓
§4.3 _apply_prm_result (score 落进三个地方)
  - 6 步:拿 score → idx → 边界 → 写 turn record → SkillManager 两边都回灌 → 挂 pending dict
  ↓
§4.4 _maybe_finalize_ready_turns (批量检查 + 触发 finalize)
  - 3 步:遍历 pending → 判定 ready → 启 _finalize_turn_feedback task
  ↓
_finalize_turn_feedback (写 prm_scores.jsonl)
  - fire-and-forget,不阻塞主循环
```

## 6. PRM 客户端与 LLM 客户端的"区分"

**这是 SkillClaw 的一个关键设计选择**:PRM 和 LLM 转发**可以**用**不同的端点**。通常:

- LLM 转发:用"便宜快"的模型(kimi-k2.5 / gpt-4o-mini / haiku)
- PRM:用"强但慢"的模型(gpt-5.2 / claude-sonnet-4-6 / doubao-pro)

`SetupWizard` 支持独立配置(11 章 §配置系统 步骤 12-14)——`llm.model_id` 决定转发的 LLM,`prm` 段独立决定 provider / url / model / api_key。`to_skillclaw_config`(11 章 §配置系统)把 `prm.url` / `prm.model` / `prm.api_key` 解析为 `SkillClawConfig.prm_url` / `prm_model` / `prm_api_key`,**没填时**回退到 LLM 配置(顺序:`prm.url or llm.api_base`,`prm.model or llm.model_id or "gpt-5.2"`,`prm.api_key or llm.api_key`)——保证最少配置能跑。

## 7. 反馈到 SkillManager 的全链路

PRM 评完到 skill stats 落盘的完整链:PRM 评完 → 触发 `_on_prm_done` callback(session 关闭时用 `_on_prm_done_record_only`,不调 `record_feedback`)→ 进入 `_apply_prm_result(session_id, turn_num, prm_result)` → 从 `prm_result` 取 `score`(∈ {-1.0, 0.0, 1.0}) → 把 `score` 写进 `turns[turn_num-1]["prm_score"]`(写 session turn record)→ 对注入组和读取组**两边都回灌**——`self.skill_manager.record_feedback(injected_skills, score)` + `self.skill_manager.record_feedback(read_skills, score)` → `SkillManager.record_feedback` 内部按 `score` 的正 / 零 / 负分别累加 `positive_count` / `neutral_count` / `negative_count`,并按 `positive_count / inject_count` 重算 `effectiveness` → `_maybe_flush_stats` 在累计 10 次 mutation 后批量写一次 `skill_stats.json`(减少 IO 频次)。

**`inject_count` 由谁更新**:`inject_count` **不**是 PRM 路径更新的——它在 `record_injection(skill_names)`(`skill_manager.py:244`)里 `+= 1`,**触发点是"skill 被注入到 prompt"**(由 `_inject_skills` 调,15 章 §5)。**PRM 路径只更新 `positive_count` / `negative_count` / `neutral_count`,**不**动 `inject_count`**。所以:

- `inject_count` 增 → "这个 skill 被用了一次"
- `positive_count` 增 → "这个 skill 评 +1 了一次"
- `effectiveness = positive_count / inject_count` ∈ [0, 1]

**举例**:某 skill 注入 10 次、PRM 评 6 次 +1 / 2 次 -1 / 2 次 0:`inject_count=10`, `positive_count=6`, `effectiveness=0.6`。

**`session_upload_interval > 0` 周期化触发**时也会**主动**调 `_trigger_evolve` → `POST {evolve_server_url}/trigger` → evolve server 跑一次 cycle(14 章 §3.4-3.5、20 章 §共享存储与同步)。**这条与路径 A 是独立的**——`_trigger_evolve` 不等 PRM,直接发;如果 PRM 还没评完,`prm_score` 还没写进 turn record,evolve server 看到的 session 里那条 turn 的 `prm_score` 是 `None`,后续 PRM 评完**也**只影响本地 effectiveness,**不会**回填到已上传的 session。

## 8. PRM 与 skill 演化的"闭环"

`skill_stats.json` 在 push 时被读(`cli.py` 里的 `skills_push` 子命令)——CLI 拼出本地 `stats_path`(在 `cfg.skills_dir` 下的 `skill_stats.json`),如果存在就 `json.load` 整个 stats 字典,然后构造 `skill_filter` 字典:把 stats 本身 + 推送阈值(`min_injections` 默认 5 / `min_effectiveness` 默认 0.3)三件套一起,作为 `skill_filter=` 参数传给 `hub.push_skills(skills_dir, skill_filter=...)`。

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

> **"未设置" / "(空)" 的精确含义**:YAML 里这个 key **不存在**(`config_store._normalize_*` 走 `dict.get("key", default)` 拿到 default)。**不**是值是空字符串 `""`,也**不**是值是 `null` / `~`——后者会被 `int("")` / `float("")` 报 `ValueError`,由 `ConfigStore.load()` 在加载时报错。换言之,本表"未设置"列等价于"YAML 缺这行";如果你看到配置 dump 里这行值是 `""` 或 `null`,**那是另一个 bug,详 11 章 §配置系统 已知问题**。

---

## 10. 待确认 / 已知限制

> 本节按"**已知设计权衡**(已下结论)+ **待对照验证**(边界未实测)"两类整理;**已知设计权衡**直接给结论,不带标签。

### 10.1 已知设计权衡

1. **`prm_m=0` 静默 0.0**:当前实现**不**加 `assert prm_m >= 1`(`prm_m` 是普通 int,不是 enum,加 assert 反而破坏用户配置加载);`_maybe_finalize_ready_turns` 在 `prm_m=0` 时 `asyncio.gather(*[])` 返空、`_majority_vote([])` 返 0.0,所有 turn 的 `prm_score` 都是 0.0。**生产静默故障的兜底放在配置层**:`setup_wizard` 步骤 11 / `config.yaml` 校验加 `prm_m >= 1`,**不**放在 `PRMScorer.__init__`(详 11 章 §配置系统 字段校验)。当前实现是:`PRMScorer` 不报,配置层报。

2. **`prm_max_new_tokens=1024` 偏大**:PRM 输出"thought + `Score: 1`"在 50-200 token 之内,1024 浪费约 5x token 成本。**当前值是有意的"容错"**——某些 LLM(尤其 Claude)会先写 thought 再给分,1024 留余量避免截断。**优化路径**:不改默认,让用户在 `config.yaml` 调到 200;不要直接砍到 200(会有 LLM 跑截断)。

3. **`prm_temperature=0.6` 偏随机**:评估任务通常 0.0~0.2 更稳。0.6 让"同一条 response 不同票数"出现概率上升——这恰好是 `prm_m=3` 多数票想压的方差源。**当前值是默认遗留**(`OpenAI` 推荐 temperature 0.7);`prm_temperature=0.0` 在多数 provider 下能让"同票"概率上升,削弱多票的价值。**结论**:`prm_temperature=0.2` 是更稳的默认,改这个值要 PR(详 11 章 §配置系统 默认值变更流程)。

4. **`injected_skills` + `read_skills` 两边都回灌**(设计哲学,§4.3 已铺):**当前定论是"两边都回灌"**——`injected_skills` 是"注入了",`read_skills` 是"读了",两类独立事件分别计分。如未来发现 `injected_skills` 拖累 effectiveness(极端 case:某 skill 频繁被注入但很少被读、被 PRM 评 -1),收敛到 `read_skills` only 是已知的迭代路径。

5. **`_sanitize_text` 把 `<file>` 替为 `[tag]`**:OpenClaw / Hermes 适配器可能 emit 各种 `<file>` / `<output>` XML 块,统一替成 `[tag]` / `[/tag]` 防止 LLM provider content filter 误判。**当前是"宁可错替也不放过"的策略**——`[tag]` 是个无害占位,影响 PRM 看 response 全文时的可读性,**不**影响评分决策。**不**打算精细化"只替 OpenClaw 私有 tag"(投入产出比低)。

6. **Bedrock 路径 + 没装 `openai` 包的兼容性**:`PRMScorer.__init__` 走 `llm_client != None` 分支时不 `import openai`,所以**纯 Bedrock 部署 + `pip install skillclaw[bedrock]`**(不装 `openai` 依赖)能跑。但 `validate_skill_evaluator`(21 章)等其它模块可能顶层 import `openai`——纯 Bedrock 部署需保证**所有走到的路径**都不顶层 import `openai`。**当前** SkillClaw 的 `pyproject.toml` 把 `openai` 列为**核心依赖**,**不**走可选依赖——这条"纯 Bedrock 不装 openai"是**理论可行**,**不是承诺**;真要走需要单独 build。

7. **PRM 异常时 `prm_score` 留 `None`**:`_on_prm_done` 的 `except Exception: return` 让 PRM 任务失败时 turn record 的 `prm_score` 保持原值(初始 `None`)。**当前是有意设计**——evolve server 看到 `prm_score=None` 视作"无评分",不参与 LLM 摘要的 PRM 维度;**不**记 -1(避免把"评失败"和"评 -1"混淆)。session 上传后该 turn 仍正常出现在 trajectory,**只**是 LLM 摘要时少一个 PRM 信号。当前实现是:`None` = 评失败,`-1` = 评不好,两者语义不混。

### 10.2 暂留（待对照验证）

> 这些是"边界未实测、需要部署环境或样本统计才能定论"的事项;**不**是设计本身有问题。

- **`prm_m=4` 时 3-1 vs 2-2 的具体 LLM 行为**:§3 表推了 `[1,1,1,-1]→1.0` / `[1,1,-1,-1]→0.0`,但生产里 LLM 经常给"半 +1 半 0"(`[1,1,0,0]`)——`_majority_vote` 把这个算成 0.0(top=1 出 2 次、top=0 出 2 次 → 平票)。需要统计真实流量里 `[1,1,0,0]` / `[1,0,0,-1]` 之类"非典型平票"的分布,以及是否值得给 `_majority_vote` 加"top 票 vs 次票差 ≥ 2 才算多数"的更强条件。
- **Bedrock `Converse` 流对 `system` 多模态(图片)的兼容性**:`BedrockChatClient` 内部把 OpenAI 格式 `system: "..."` 转 Bedrock `system: [{"text": "..."}]`,**但** Claude 3.5 / Claude 4 在 Bedrock 上 `system` 块支持图片多模态,`BedrockChatClient` 当前**不**识别 `system` 里的 image_url——PRM 不会触发多模态(`_query_once` 只用 text),**但** `api_server._forward_to_llm_bedrock` 转发的请求里如果 `system` 块有图片,**会被丢掉**。此条不影响 PRM,影响 LLM 转发——属于 §5.3 范围外。
- **`sharing_user_alias` 在 `setup_wizard` 未运行时的 fallback 顺序**:`config_store._normalize_*` 在 wizard 之前**不**回退到 `$USER`——回退逻辑在 `_load_or_init` 的最后一步(11 章 §配置系统 §3)。`pip install skillclaw && skillclaw serve`(没跑过 wizard)是否能让 `manifest.jsonl` 的 `uploaded_by` 正确写到 `$USER`,还是会被 loader 当作空字符串——需要部署环境验证。

---

→ **下一篇**：[17 · 适配器矩阵](17-适配器矩阵.md) — SkillClaw 怎么把 12 种 CLI agent 的本地配置改写、备份、回滚
