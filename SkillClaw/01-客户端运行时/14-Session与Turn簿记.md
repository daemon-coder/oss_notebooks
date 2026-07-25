# 14 · Session 与 Turn 簿记

## 这一章讲什么

这一章回答"一次完整会话从开到关，SkillClaw 内部存了哪些状态、按什么规则关、用什么 cadence 上传到共享存储"。覆盖 TUI 边界检测、idle sweeper、`_close_session` 的完整流程、PRM 与 turn 的联动。

## 它在整个系统的哪个位置

`_handle_request` 之后台：`SkillClawAPIServer` 持有的 `self._*` 状态机 + `api_server.py` 的 `_close_session` / `_session_idle_sweeper_loop` / `_drain_active_sessions` / `_maybe_finalize_ready_turns`。

## 设计目的

把"一堆 stateless HTTP 请求"重新拼成"一次连续对话"——但**不**用一个长生命周期的 WebSocket 或数据库连接。SkillClaw 的所有 session 状态都在**单个进程的内存 + record JSONL 文件**里，session 关闭时一次性上传到共享存储。

---

## 1. Session 来源的四种分类

`_handle_request` 收到的 session_id 有四种来源：

| 来源 | 解析位置 | session_id 形态 | 备注 |
|---|---|---|---|
| `X-Session-Id` header | `chat_completions` / `responses` / `anthropic_messages` | 客户端自定义 | OpenClaw / Hermes / 显式传 |
| `body.session_id` | 同上 | 同上 | 兜底（当 header 缺失） |
| `x-claude-code-session-id` header | `anthropic_messages` | UUID-like | Claude Code 客户端 |
| TUI 启发式 | `_resolve_tui_session` | `tui-<model>-<8hex>` | 通用 OpenAI 客户端（QwenPaw / IronClaw / …） |

### 1.1 `_resolve_tui_session(model, msg_count)`

TUI 类客户端（不发 `X-Session-Id`）的伪 session 检测：

```python
# 伪代码骨架（见 api_server.py:_resolve_tui_session）:
async def _resolve_tui_session(self, model, msg_count):
    tui_key = f"tui-{model}"
    meta = self._tui_session_meta.get(tui_key)
    # 第一次见 / 跨对话边界 → 分配新 session
    if meta is None or is_tui_boundary(meta, msg_count):
        await self._close_session(meta["session_id"], reason="tui_boundary")
        return allocate_new_session(tui_key, msg_count)
    # 否则复用 + 更新 msg_count / last_request_time
    return meta["session_id"]
```

**关键不变量**：
- **同一 `tui-<model>` 永远只对应一个活跃 session**——切新时**显式 close 老 session**
- 消息数下降 / 300s 不活动任一条件触发就切
- 切新 session 时**继承** `tui_key`（用 model 名），保证两个不同 model 互不干扰

`_tui_inactivity_timeout = 300s`（硬编码在 `__init__`，不来自 config——**待确认**是否应该 expose 为 config 字段）。

## 2. Session 关闭的四个触发源

```python
# 在不同地方都会调
await self._close_session(session_id, reason=...)
```

| reason | 触发位置 | 备注 |
|---|---|---|
| `explicit`（默认） | `body.session_done=True` 或 `X-Session-Done: true` | 客户端显式关 |
| `idle_timeout` | `_session_idle_sweeper_loop` 每 15s 扫一次 | 距上次活动 ≥ 180s |
| `tui_boundary` | `_resolve_tui_session` 检测到消息数下降 / 300s 不活动 | 老 TUI session 被替换 |
| `server_shutdown` | `_shutdown_cleanup` 在 FastAPI lifespan 退出时 | 整进程关 |

### 2.1 `_collect_idle_session_ids`

```python
def _collect_idle_session_ids(self, now=None) -> list[str]:
    if self._session_idle_close_seconds <= 0:  # 关掉 idle 关闭
        return []
    if now is None: now = time.time()
    threshold = float(self._session_idle_close_seconds)  # 默认 180
    return sorted(
        sid for sid, ts in self._session_last_active.items()
        if sid and sid not in self._closing_sessions and (now - float(ts)) >= threshold
    )
```

`_session_idle_close_seconds` 可以从 config 读 `session_idle_close_seconds`（**config 字段没在 `SkillClawConfig` 暴露**——**待确认**是否遗漏），否则用模块级默认 180s。

`_session_idle_sweeper_loop` 每 `_session_sweep_interval_seconds`（默认 15s）扫一次：

```python
while True:
    await asyncio.sleep(self._session_sweep_interval_seconds)
    for sid in self._collect_idle_session_ids():
        await self._close_session(sid, reason="idle_timeout")
```

**关掉 idle 关闭**：`config.session_idle_close_seconds <= 0` 时整个 sweeper 不启（`_start_session_idle_sweeper` 直接 return）。

## 3. `_close_session` 全流程

```python
# 伪代码骨架（见 api_server.py:_close_session）:
async def _close_session(session_id, reason="explicit"):
    防重入 + 7 步收尾:
      1. _flush_pending_record → conversations.jsonl 最后一行
      2. 触发 pending turn 的 PRM（record-only 路径）
      3. drain PRM (最多 15s)
      4. finalize turn → prm_scores.jsonl + record_feedback
      5. SkillManager._save_stats() 强制刷盘
      6. _upload_session_data + _pull_skills_from_cloud
      7. 清 7 个 session_id 状态表
```

### 3.1 `_flush_pending_record(session_id, next_state)`

把当前 turn 的 record 缓冲（`_pending_records[session_id]`）写一行到 `conversations.jsonl`。`next_state=None` 标志"这是最后一次 flush"——**不**触发新 PRM（只把当前 turn 的 record 落盘）。

### 3.2 PRM 同步 drain

session 关闭时还要等正在跑的 PRM task 跑完（最多 15s）——这样能让最后一轮 PRM 分数回灌 `SkillManager._stats`。**超时 15s 后强制 finalize**（用 `prm_result = None` 走 fallback 分支）。

### 3.3 SkillManager 写盘

```python
if self.skill_manager: self.skill_manager._save_stats()
```

强制把 `_stats` 写回 `skill_stats.json`（不管 `_maybe_flush_stats` 的 10-mutation 阈值）。

### 3.4 Session 上传

```python
async def _upload_session_data(self, session_id, turns) -> bool:
    hub = SkillHub.object_storage_from_config(self.config)
    if hub is None: return False
    payload = {"session_id": session_id, "timestamp": "<UTC ISO-8601>",
               "user_alias": "...", "num_turns": len(turns), "turns": turns}
    hub._bucket.put_object(f"{hub._prefix()}sessions/{session_id}.json",
                           json.dumps(payload, ensure_ascii=False).encode("utf-8"))
    return True
```

- **走 `SkillHub.object_storage_from_config`**——这是"非 Nacos" 的对象存储后端。Nacos 不存 session 资产。
- key 格式：`{group_id}/sessions/{session_id}.json`
- 失败不重试（**待确认**——这一行 `return False` 后上层 caller 不看返回值）

### 3.5 拉新 skill

```python
self._safe_create_task(self._pull_skills_from_cloud(skip_names=modified_skill_names))
```

session 关闭时**也**触发一次 pull——这样"用户改了一个 skill" → "下次 session 关闭" → "本地立刻拉到新版"。`skip_names` 跳过本 session 内被模型**修改**的 skill，避免覆盖用户刚改的本地版。

### 3.6 状态清理顺序

```python
self._session_last_active.pop(...)   # 最后才清 idle tracker
for meta in _tui_session_meta:
    if meta["session_id"] == session_id:
        _tui_session_meta.pop(key)   # TUI 元数据也清
```

`_closing_sessions.discard(session_id)` 在 `finally` 块，保证重入保护的恢复。

## 4. Record 与 PRM 的双文件

### 4.1 `conversations.jsonl`

每行一个 turn 记录，由 `_flush_pending_record` 写入：

```python
rec = {
    "session_id": ..., "turn": ..., "timestamp": ...,
    "messages": messages, "response_text": ...,
    "instruction_text": ..., "prompt_text": ..., "tool_calls": ...,
    "next_state": ...,   # 下一条 user message（PRM 评这一条）
}
```

`next_state` 字段是 PRM 触发器的关键——**PRM 必须等下一条 user 进来才评**（需要"对照这一条回答对 user 的下一条问题"才能判断"回答是不是真的有帮助"）。见 `_fire_prm_scoring` 详情：

```python
def _fire_prm_scoring(self, session_id, turn_num, response_text, instruction_text, next_state,
                      finalize_ready_turns=True):
    if not self.prm_scorer or not next_state: return    # 没 next_state → 不评
    task = asyncio.create_task(self.prm_scorer.evaluate(...))
    task.add_done_callback(self._on_prm_done 回调)     # record-only 或 finalize 两条路径
    self._prm_tasks[session_id][turn_num] = task
    pending_turn["has_next_state"] = True
```

### 4.2 `prm_scores.jsonl`

```python
{"session_id": "...", "turn": N, "score": -1.0, "votes": [1, -1, -1]}
```

每个 turn 一行。**纯 append**——重跑 evolve server 不会读这个文件（它读 `sessions/{session_id}.json`），但用户做 local debugging 可以 grep。

### 4.3 启动清空

```python
if config.record_enabled:
    os.makedirs(config.record_dir, exist_ok=True)
    self._record_file = f"{record_dir}/conversations.jsonl"
    self._prm_record_file = f"{record_dir}/prm_scores.jsonl"
    with open(self._record_file, "w"): pass   # 截断
    with open(self._prm_record_file, "w"): pass
```

**这两个文件在每次 SkillClawAPIServer 启动时被截断**——**record 是"本次进程内"的，不跨重启累积**。如果想保留历史记录做后处理，应当在进程**未启动**时把 record 文件归档。

`record_enabled=False` 时 `_record_file = ""`，所有 `_append_*` 函数 `if not self._record_file: return`——`purge_record_files()` 是显式清理入口。

## 5. Turn 簿记的关键不变量

- **`_turn_counts[session_id]`**：1-based，**每个 main turn 自增一次**（side turn 不计）
- **`_user_turn_counts[session_id]`**：只在 `_is_user_turn_boundary(raw_turn_kind) == True` 时自增
- **`_pending_turn_data[session_id][turn_num]`**：每个 turn 维护 `{prompt_text, response_text, has_next_state}`——prompt_text 来自 `_extract_last_user_instruction(messages)`（只取最后一条 user）
- **`_pending_records[session_id]`**：单条缓冲（同一 session 同一时刻只缓冲最新 turn），等 next_state 进来就 flush 到 conversations.jsonl

## 6. 周期化 session 上传（`_maybe_upload_session_snapshot`）

不同于"session 关闭时上传"，**SkillClaw 也支持"每 N 个 user_turn 上传一次"**（用于长会话中途同步）：

```python
def _maybe_upload_session_snapshot(self, session_id, user_turn_num):
    interval = max(0, int(getattr(self.config, "sharing_session_upload_interval", 0) or 0))
    if not self.config.sharing_enabled or interval <= 0:
        return
    if user_turn_num <= 0 or user_turn_num % interval != 0:
        return
    turns = copy.deepcopy(self._session_turns.get(session_id, []))
    if not turns: return
    self._safe_create_task(self._upload_session_snapshot_and_trigger(session_id, turns))
```

`sharing_session_upload_interval=0`（默认）= 关闭。设成 5 就是每 5 个 user turn 上传一次。

`_upload_session_snapshot_and_trigger` 在上传完后调 `_trigger_evolve`——**主动**通知 evolve server 跑一次 cycle，而不是等 10 分钟周期。

```python
async def _trigger_evolve(self):
    url = str(getattr(self.config, "evolve_server_url", "")).strip().rstrip("/")
    if not url: return
    for attempt in range(3):                  # 失败指数退避: 1s, 2s
        try:
            resp = await httpx.AsyncClient(timeout=300).post(f"{url}/trigger")
            if resp.json().get("uploaded_skills", 0) > 0:
                await self._pull_skills_from_cloud()
            return
        except Exception: await asyncio.sleep(1.0 * (attempt + 1))
```

**3 次重试**，间隔 1s/2s。`evolve_server_url` 默认空——只有用户配了才生效。

## 7. 主动关闭（`_drain_active_sessions`）

`lifespan` 退出时调：

```python
async def _drain_active_sessions(self, reason: str):
    active_ids = self._collect_active_session_ids()
    if not active_ids: return
    for sid in active_ids:
        await self._close_session(sid, reason=reason)
```

`_collect_active_session_ids` 把所有"还在活动"的 session id 收齐（来自 `_session_last_active` / `_pending_records` / `_session_turns` / `_pending_turn_data` / `_turn_counts` / `_session_scored_turns` / `_prm_tasks` 这 7 个状态表）。

## 8. `IdleStateProvider` 协议（给 ValidationWorker 用）

```python
class IdleStateProvider(Protocol):
    def active_session_count(self) -> int: ...
    def last_request_age_seconds(self) -> Optional[float]: ...
    def is_idle_for_validation(self, idle_after_seconds: int) -> bool: ...
```

`SkillClawAPIServer` 实现这三个：
- `active_session_count()`：`len(self._collect_active_session_ids())`
- `last_request_age_seconds()`：`time.time() - self._last_request_at`
- `is_idle_for_validation(N)`：active_count == 0 **且** age ≥ N

`SkillClawLauncher` 把 server 注入 `ValidationWorker(idle_provider=server)`——server 自己也用 `_mark_request_activity()` 维持 `_last_request_at`。

## 9. 关闭时序示例

> 一个用户跑了 1 小时、产生 50 个 turn、PRM 在 5 个 turn 还在跑。

1. 用户按 Ctrl-C → SIGINT → Launcher handler → `launcher.stop()`
2. Launcher 调 `server.stop()` → `uvicorn.Server.should_exit = True`
3. uvicorn 走完手头请求 → 进 `lifespan` finally → `_shutdown_cleanup`
4. `_skill_reload_task.cancel()` / `_session_sweeper_task.cancel()`：关两个常驻 task
5. `_drain_active_sessions("server_shutdown")`：50 个 session 顺序 `_close_session`
6. 每个 session：
   - `_flush_pending_record(sid, None)`：最后一行落 conversations.jsonl
   - 触发剩余 5 个 turn 的 PRM（`finalize_ready_turns=False` 走 record-only 路径）
   - `asyncio.wait_for(gather, timeout=15s)`：等 5 个 PRM
   - 15s 超时后强制 finalize：写入 `prm_scores.jsonl` 和 `SkillManager._stats`
   - `_upload_session_data`：上传 `{group_id}/sessions/{sid}.json`
   - `_pull_skills_from_cloud(skip_names=modified_skills)`：拉新 skill
7. `_await_background_tasks(15s)`：等上传 task 跑完
8. uvicorn 进程退出

**潜在问题**：50 个 session 顺序关、每个最多等 15s——极端情况下整个进程要 750s 才能退。**待确认**应该并行 close 而非顺序 close。生产部署里建议给 systemd / launchd `KillSignal=SIGKILL` after `TimeoutStopSec=60`。

## 10. 待确认 / 已知限制

- **`_session_idle_close_seconds` 暴露**：在 `__init__` 里 `getattr(config, "session_idle_close_seconds", _SESSION_IDLE_CLOSE_SECONDS)`——但 `SkillClawConfig` 字段没定义它。**待确认**这字段应该是 config 还是硬编码。
- **`session_done` 校验**：`_resolve_session_done` 是 `bool(...)`，但 `X-Session-Done: "false"` 字符串会被 truthy 处理成 True。**待确认**这是 bug 还是按 "agent 显式说 done 就是 done" 设计的。
- **TUI inactivity timeout 也是 300s 硬编码**：`_INACTIVITY_TIMEOUT = 300` 写在 `__init__` 顶部注释里。**待确认**为何不 expose。
- **`_truncate_messages` 跟 session 簿记解耦**：丢老 message 不影响 `_session_turns`——被截断丢的 message **仍在** session 完整记录里，只是不再喂给 LLM。**这是有意的**（agent 视角下"对话是连续的"），但意味着 session 越来越大。**待确认**要不要给 session 长度也加一个硬上限。
- **`is_idle_for_validation` 与 session_idle_close 的关系**：session 关闭阈值（180s）小于 validation idle 阈值（300s）——保证 session 关完之后再过 120s 才开始 validation。**待确认**这两个值是不是应该用同一个 config 字段。

---

→ **下一篇**：[15-技能库与注入](15-技能库与注入.md) — SkillManager 怎么从 SKILL.md 列表里挑出本轮要注入的 skill
