# 22 · Dashboard 与可观测性

## 这一章讲什么

这一章回答"`skillclaw dashboard sync` / `serve` 怎么把本地 + 共享的 skill / session / validation 投到 SQLite 投影，以及 FastAPI 后端和静态前端怎么配合"。

## 它在整个系统的哪个位置

`DashboardService`（`dashboard_server.py`）+ `DashboardStore`（`dashboard_store.py`）+ `build_dashboard_snapshot`（`dashboard_ingest.py`）。三个文件总计 ~2700 行，是客户端第二大子系统（仅次于 api_server）。

## 设计目的

让"我现在库里有哪些 skill、本地跟共享是否一致、哪些 session 触发了 skill 演化、validation 还在等多少"这些信息**可视化**。

**dashboard 的读写边界**:

- **视图层(GET 类端点)只读**——`/api/v1/overview` / `/api/v1/skills` / `/api/v1/sessions` / `/api/v1/validation/jobs` / `/api/v1/evolve/status` 全部从 SQLite 读,**不**写。
- **操作层(POST 类端点)**有 **5 个写操作**,每个**显式**走原有 `SkillHub` / `ValidationStore` 路径,**不**直接改 SQLite 之外的真实数据:
  1. `POST /api/v1/skills/{id}/activate` → `activate_skill_version` → 通过 `hub._bucket` 把历史版本回滚到 `skills/<name>/`(实际**写**到对象存储的 `skills/<name>/`,**写**到本地 `skills_dir`)
  2. `POST /api/v1/ops/export-sessions` → `export_local_sessions` → 通过 `hub._bucket` 把本地 session 上传到对象存储的 `sessions/<sid>.json`
  3. `POST /api/v1/validation/jobs/{job_id}/review` → `submit_validation_review` → 通过 `ValidationStore.save_result` 写 `validation_results/<job_id>/<user_alias>.json`(可能触发 inline finalize)
  4. `POST /api/v1/ops/pull` → `pull_skills` → **写**到本地 `skills_dir`(覆盖或追加;走 `SkillHub.pull_skills` 的 mirror / append 路径)
  5. `POST /api/v1/ops/push` → `push_skills` → **写**到共享存储的 `skills/<name>/` + 更新 `manifest.jsonl`(走 `SkillHub.push_skills` 路径)
- **这 5 个写操作都通过原有 `SkillHub` / `ValidationStore` 接口**——dashboard **不**直接调用对象存储 SDK,**不**绕过 push/pull/sync 路径;**它只是给这些操作加了一个 web 入口**。

**结论**:dashboard 是"**通过原有接口的写操作 + 自己的视图层**"两层。读和写都走 `SkillHub` / `ValidationStore` 的同一份代码,不绕路;读端不写,写端必须走原接口——**两层都通过原代码路径**。

---

## 1. 命令与数据流

```
skillclaw dashboard sync
  → DashboardService(cfg).sync()
    → build_dashboard_snapshot(cfg)   ← 拉所有源数据
    → DashboardStore.replace_snapshot(snapshot)   ← 全量替换到 SQLite

skillclaw dashboard serve [--host ...] [--port ...] [--no-sync-on-start]
  → DashboardService(cfg)（构造）
  → 启动时若 sync_on_start=True：service.sync()
  → uvicorn.run(create_dashboard_app(cfg), host, port)
    → FastAPI lifespan: store.initialize() + 可选 service.sync()
    → /api/v1/* 路由全部查 store
```

`dashboard.db` 是 SQLite 文件，默认 `~/.skillclaw/dashboard.db`。

## 2. 数据源（`build_dashboard_snapshot` 读哪儿）

`dashboard_ingest.py` 的 1082 行的 `build_dashboard_snapshot(config) -> dict`：

| 段 | 来源 |
|---|---|
| `meta` | hardcoded（snapshot build time、config 来源） |
| `skills` (local) | `_load_local_skills(cfg, warnings)`：扫 `cfg.skills_dir` 下 SKILL.md（`SkillManager`-style parser） |
| `skills` (remote) | `_load_shared_skills(cfg, warnings)`：从 `SkillHub.from_config` 拉 manifest，按 `name` join |
| `sessions` (local) | `_load_local_sessions(cfg, warnings)`：从 record 文件 + 共享存储的 sessions 拉 |
| `sessions` (remote) | `_load_state_sessions(cfg, warnings)`：从共享存储的 sessions/ 拉（evolve server 上传过的） |
| `validation_jobs` | `ValidationStore.list_jobs()`（如果 `cfg.sharing_enabled`） |
| `candidates` | 从 `candidate_skills/` 拉所有 `SKILL.md` |
| `evolve_status` | HTTP GET `{cfg.dashboard_evolve_server_url}/status`（如果配了） |
| `warnings` | 累积各类收集错误 |

**`_load_local_skills`** 不复用 `SkillManager`——是独立的小 parser（`_parse_skill_document`），只读 frontmatter + content，**不**算 fingerprint / stats。

**`_load_shared_skills`** 拉 manifest + 候选 SKILL.md（`_parse_skill_document`） → `merge_local_sessions` 把 local 和 remote 的 skills 按 name 合并，每条 skill record 含：

```json
{
  "name": "debug-systematically",
  "skill_id": "a1b2c3d4e5f6",
  "version": 3,
  "current_version": 3,
  "description": "...",
  "category": "coding",
  "source": "local",                      # 或 "shared" 或 "both"
  "has_local": true,
  "has_remote": true,
  "local_path": "/Users/alice/.skillclaw/skills/...",
  "remote_record": {<manifest entry>},
  "uploaded_at": "...",
  "uploaded_by": "...",
  "content_sha": "...",
  "content": "<SKILL.md text>"
}
```

**`_load_record_sessions`** 从 `cfg.record_dir/conversations.jsonl`（"本次进程内"） + 共享存储 `sessions/*.json`（历史）合并。

### 2.1 边角 case

- **共享存储不可达** → 把 warning 累积到 `warnings: []`，本地数据继续展示
- **`prm_scores.jsonl` 读不到** → 静默忽略（PRM 数据不影响 skill / session 列表）
- **`record_dir` 不存在** → 跳过本地 session
- **`validation_jobs` 拉失败** → 跳过 validation 段

## 3. `DashboardStore`（SQLite 投影）

`dashboard_store.py` 定义 `DashboardStore(db_path)` + `_connect()` 用 `sqlite3.connect(path, timeout=30)` + `row_factory=sqlite3.Row` + WAL 模式。

### 3.1 Schema

```sql
PRAGMA journal_mode=WAL;

CREATE TABLE meta (key TEXT PRIMARY KEY, value TEXT NOT NULL);

CREATE TABLE skills (
  skill_id TEXT PRIMARY KEY,
  name TEXT NOT NULL UNIQUE,
  description TEXT NOT NULL DEFAULT '',
  category TEXT NOT NULL DEFAULT 'general',
  source TEXT NOT NULL DEFAULT 'local',    -- local / shared / both
  has_local INTEGER NOT NULL DEFAULT 0,
  has_remote INTEGER NOT NULL DEFAULT 0,
  local_path TEXT NOT NULL DEFAULT '',
  uploaded_at TEXT NOT NULL DEFAULT '',
  uploaded_by TEXT NOT NULL DEFAULT '',
  updated_at TEXT NOT NULL DEFAULT '',
  current_version INTEGER NOT NULL DEFAULT 0,
  current_sha TEXT NOT NULL DEFAULT '',
  local_inject_count INTEGER NOT NULL DEFAULT 0,
  observed_injection_count INTEGER NOT NULL DEFAULT 0,
  read_count INTEGER NOT NULL DEFAULT 0,
  modified_count INTEGER NOT NULL DEFAULT 0,
  session_count INTEGER NOT NULL DEFAULT 0,
  effectiveness REAL NOT NULL DEFAULT 0.0,
  positive_count INTEGER NOT NULL DEFAULT 0,
  negative_count INTEGER NOT NULL DEFAULT 0,
  neutral_count INTEGER NOT NULL DEFAULT 0,
  content TEXT NOT NULL DEFAULT '',
  raw_json TEXT NOT NULL DEFAULT '{}'
);
CREATE INDEX idx_skills_name ON skills(name);
CREATE INDEX idx_skills_category ON skills(category);
CREATE INDEX idx_skills_source ON skills(source);
CREATE INDEX idx_skills_sessions ON skills(session_count DESC, observed_injection_count DESC);

CREATE TABLE skill_versions (
  skill_id TEXT NOT NULL,
  version INTEGER NOT NULL,
  content_sha TEXT NOT NULL DEFAULT '',
  action TEXT NOT NULL DEFAULT '',
  timestamp TEXT NOT NULL DEFAULT '',
  raw_json TEXT NOT NULL DEFAULT '{}',
  PRIMARY KEY (skill_id, version)
);
CREATE INDEX idx_skill_versions_timestamp ON skill_versions(skill_id, timestamp DESC);

CREATE TABLE sessions (
  session_id TEXT PRIMARY KEY,
  timestamp TEXT NOT NULL DEFAULT '',
  user_alias TEXT NOT NULL DEFAULT '',
  num_turns INTEGER NOT NULL DEFAULT 0,
  ...
);
```

（`sessions` / `validation_jobs` / `validation_results` 表结构类似，省略。完整版见 `dashboard_store.py` `initialize()`。）

### 3.2 `replace_snapshot(snapshot)`

**场景**:`skillclaw dashboard sync` 命令调 `replace_snapshot(snapshot)`——把"snapshot dict"全量替换进 SQLite 投影。这是 dashboard 的"刷新点",sync 之后 dashboard serve 读到的就是新数据。

**4 步全量替换**:

```python
def replace_snapshot(self, snapshot):
    # 段 1:开数据库连接(WAL 模式)
    with self._connect() as conn:
        # 段 2:全表清(DELETE FROM skills / sessions / validation_*)
        for table in ["skills", "sessions", "validation_jobs", "validation_results", "validation_decisions"]:
            conn.execute(f"DELETE FROM {table}")

        # 段 3:逐表 INSERT 每条记录
        skills_count = self._insert_skills(conn, snapshot.get("skills", []))
        sessions_count = self._insert_sessions(conn, snapshot.get("sessions", []))
        # ... 其它表类似

        # 段 4:UPDATE meta SET last_sync = now()
        conn.execute("UPDATE meta SET last_sync = ?", (iso_now(),))
        conn.commit()

    return {
        "skills": skills_count,
        "sessions": sessions_count,
        "validation_jobs": ...,
        "validation_results": ...,
    }
```

**WAL 模式**允许 sync 时 dashboard serve 还在读——读到的要么是旧 snapshot、要么是新 snapshot,**不会半中间状态**(WAL 模式 commit 前的修改对外不可见)。

**为什么是全量替换而不是增量**:

- 增量写容易出"漏 delete / 漏 update" bug——snapshot 源是 local JSON + 共享 storage,可能因为时钟漂移 / 客户端 race 出现"前次同步后又被删了"的 skill
- 全量替换以 source 为准,简单且**强一致**
- 缺点是 sync 慢(O(N) 写 SQLite)——但 dashboard 用只读视图,数据量小(几百个 skill + 几千个 session),同步成本低

**关键边界**:

- **sync 时 dashboard serve 在读**:WAL 模式让读端看到旧 snapshot(commit 前),不阻塞
- **snapshot 缺某表**(某 list 为空):该表全清(段 2 DELETE 后段 3 不 INSERT 任何行)
- **`last_sync` 时间戳**:给 dashboard 端展示"数据新鲜度"
- **sync 失败** (写盘失败):抛错,旧 snapshot 保留(rollback 到上一 commit)

**走完之后**:

- SQLite 里 4 张表完全反映新 snapshot
- `meta.last_sync` 更新到 sync 完成时间
- dashboard serve 端下次查询看到新数据(WAL commit 之后)

### 3.3 `get_overview()`

返回 dashboard 顶部的"概览"卡片数据：

```json
{
  "totals": {"skills": 42, "sessions": 17, "validation_jobs": 3, "candidates": 1},
  "by_source": {"local": 12, "shared": 8, "both": 22},
  "last_sync_at": "2026-04-20T15:00:00Z",
  "sync_warnings": ["..."]
}
```

### 3.4 `list_skills(search, category, source, limit)`

`list_skills(search="", category="", source="", limit=500) -> list[dict]` 走"动态拼 SQL + 排序 limit"两步:第一步 `sql = "SELECT * FROM skills WHERE 1=1"` 起点,`args = []` 累加参数;然后按条件追加——`if search` 追加 `AND (name LIKE ? OR description LIKE ?)` + 2 个 `f"%{search}%"` 参数;`if category` 追加 `AND category = ?` + category;`if source` 追加 `AND source = ?` + source;最后 `ORDER BY session_count DESC, observed_injection_count DESC LIMIT ?` + `args.append(limit)`。
    return [dict(row) for row in conn.execute(sql, args)]
```

排序：先按 session_count（多少 session 用过）降序，再按 observed_injection_count 降序——**最常用 → 最不常用**。

## 4. `DashboardService` 的写操作

`DashboardService` 是个**薄壳**——只把 DashboardStore 的数据 + SkillHub 集成起来，提供：

### 4.1 `sync()`

```python
def sync(self) -> dict:
    snapshot = build_dashboard_snapshot(self.config)
    summary = self.store.replace_snapshot(snapshot)
    return {"summary": summary, "overview": self.store.get_overview()}
```

### 4.2 `pull_skills(skill_names=None)`

```python
def pull_skills(self, *, skill_names=None) -> dict:
    hub = _require_sharing_hub(self.config)
    if skill_names:
        result = hub.pull_skills(self.config.skills_dir, mirror=False, include_names=skill_names)
    else:
        result = hub.pull_skills(self.config.skills_dir)
    sync_result = self.sync()
    return {"operation": "pull", "target": _sharing_target(self.config), ...}
```

**`mirror=False`**：只拉指定 / 全部 skill，不删本地的其它 skill（这是 dashboard 的"我**只**想看 / 拉某些 skill"场景）。

### 4.3 `push_skills(no_filter=False)`

```python
def push_skills(self, *, no_filter=False) -> dict:
    hub = _require_sharing_hub(self.config)
    result = hub.push_skills(self.config.skills_dir, skill_filter=_build_skill_filter(self.config, no_filter=no_filter))
    sync_result = self.sync()
    return {"operation": "push", ...}
```

### 4.4 `export_local_sessions(session_ids=None)`

```python
# 伪代码骨架（见 dashboard_server.py:DashboardServer.export_local_sessions）:
def export_local_sessions(self, *, session_ids=None) -> dict:
    # 1. 取 sharing_hub + dashboard snapshot
    # 2. 过滤 session_ids
    # 3. 与 OSS 已存在 sessions 对比 (try_get): 一致 → skipped / 否则 put_object
    return {"exported", "skipped", "errors": ...}
```

**手动从 dashboard 触发 session 上传**——绕过 session 自动关闭时的 `_upload_session_data` 路径。

### 4.5 `activate_skill_version(skill_id, *, target)`

```python
def activate_skill_version(self, skill_id, *, target) -> dict:
    """Rollback a skill to a specific version (skill_id, target=version_str)."""
    skill = self.store.get_skill(skill_id)
    version_record = self.store.get_skill_version(skill_id, target)
    bundle_files = fetch_version_bundle(hub._bucket, hub._prefix(), skill_name, version, version_record)
    write_skill_bundle(skill_root, bundle_files, clean=True)
```

**这是 dashboard 的"危险操作"**——把一个 skill 回滚到历史版本。`POST /api/v1/skills/{skill_id}/activate` 路由，body `{"target": "3"}` 触发。这条操作与 [41 章 §9.2 "dashboard 的 `activate_skill_version` 是反向操作（v<N> → current）"](../04-端到端/41-部署形态.md) 是同一条操作的两面——22 章讲"怎么调"、41 章讲"它对 manifest / registry 的影响和回滚约束"。

### 4.6 `submit_validation_review(job_id, accepted, score, notes, auto_finalize)`

```python
async def submit_validation_review(self, job_id, *, accepted, score, notes, auto_finalize=True):
    user_alias = self.config.sharing_user_alias or os.environ.get("USER", "anonymous")
    result = {"validator_mode": "manual", "decision": "accept"/"reject",
              "accepted": ..., "score": ..., "threshold": self.config.validation_min_mean_score,
              "reason": notes, "checks": {}}
    self._validation_store.save_result(job_id, user_alias, result)
    if auto_finalize: return await self._finalize_validation_job_now(job_id)
```

**让人手审 validation job**（跳过 replay）——`POST /api/v1/validation/jobs/{job_id}/review` body `{"accepted": true, "score": 0.9, "notes": "..."}`。

`_finalize_validation_job_now` 嵌一段 inline 的 `EvolveServer._finalize_validation_jobs` 逻辑——立即跑一次 finalize（不等下个 cycle）。

## 5. FastAPI 端点

`create_dashboard_app(config)` 暴露的 HTTP 端点：

| 路径 | 方法 | 用途 |
|---|---|---|
| `/` | GET | 静态 `index.html` |
| `/assets/*` | GET | 静态 css/js |
| `/api/v1/health` | GET | `{status: "ok", db_path, meta}` |
| `/api/v1/overview` | GET | 总览数据（`store.get_overview()`） |
| `/api/v1/skills?search=&category=&source=&limit=` | GET | 列表（store 查） |
| `/api/v1/skills/{skill_id}` | GET | 单条 skill 详情 |
| `/api/v1/skills/{skill_id}/activate` | POST | 回滚 skill 版本 |
| `/api/v1/sessions?skill_id=&search=&limit=` | GET | session 列表 |
| `/api/v1/sessions/{session_id}` | GET | session 详情 |
| `/api/v1/validation/jobs?status=&limit=` | GET | validation job 列表 |
| `/api/v1/evolve/status` | GET | 调 evolve server `/status`（如果配了 URL） |
| `/api/v1/sync` | POST | 立即 sync 一次 |
| `/api/v1/ops/pull` | POST | pull skills（body 可选 `{"skill_names": ["a", "b"]}`） |
| `/api/v1/ops/push` | POST | push skills（body 可选 `{"no_filter": true}`） |
| `/api/v1/ops/sync` | POST | pull+push |
| `/api/v1/ops/export-sessions` | POST | 导出本地 session 到共享存储 |
| `/api/v1/ops/trigger-evolve` | POST | 触发 evolve server `/trigger` |
| `/api/v1/validation/jobs/{job_id}/review` | POST | 手动 review |

### 5.1 lifespan

```python
@asynccontextmanager
async def lifespan(app):
    service.store.initialize()
    if config.dashboard_sync_on_start:
        try: service.sync()
        except: logger.exception("[Dashboard] initial sync failed")
    app.state.dashboard_service = service
    yield
```

**`store.initialize()`** 是幂等的（`CREATE TABLE IF NOT EXISTS`），重复启 dashboard 不会破坏 SQLite。

**`sync_on_start=False`**：用户在 dashboard 命令里加 `--no-sync-on-start` 时跳过——`dashboard.db` 还是上次 sync 的内容。

### 5.2 错误处理

`HTTPException(status_code=400, detail=...)` 用于：
- `ValueError` 从 service 抛出（"sharing not enabled" / "missing config" 等）
- `accept_skill` body 缺 `target` / 错的 skill_id

## 6. 静态前端（`dashboard_assets/`）

`skillclaw/dashboard_assets/{index.html, styles.css, app.js}`（**纯 vanilla JS**——不引 React/Vue）。

`app.js` 调 `/api/v1/*` 端点，用 fetch 拉数据 + 渲染 DOM。功能：
- 顶部 overview 卡片（totals / by_source / last_sync / warnings）
- skill 列表（搜索 / 分类 / 来源过滤）
- skill 详情（name / description / category / 各版本时间线 / activate 按钮）
- session 列表 + 详情（turn-by-turn 时间线）
- validation job 列表（pending / published / rejected 颜色区分）
- "sync now" 按钮（调 `POST /api/v1/sync`）
- "pull / push / sync" 按钮（调 `POST /api/v1/ops/*`）

**HTML 模板**只放空 div + id，JS 全部动态渲染。

## 7. `serve_dashboard(config)`

```python
def serve_dashboard(config):
    app = create_dashboard_app(config)
    uvicorn.run(app, host=str(config.dashboard_host or "127.0.0.1"),
                port=int(config.dashboard_port or 3788), log_level="info")
```

**`dashboard_host=127.0.0.1`**（默认）——dashboard 默认**只本机能看**。如果想远端访问，**用户必须显式**改 `dashboard.host`（CLI `skillclaw config dashboard.host 0.0.0.0`）。

**`dashboard_port=3788`** 是默认端口（区别于 skillclaw proxy 的 30000、evolve server 的 8787）。

## 8. 完整的数据流示例

> 某用户跑了一周想看 dashboard。

1. `skillclaw dashboard sync`：build snapshot → 写 `~/.skillclaw/dashboard.db`
2. `skillclaw dashboard serve --host 0.0.0.0 --port 3788`：起 FastAPI
3. 用户浏览器打开 `http://host:3788/`
4. 前端 `app.js` 拉 `/api/v1/overview` → 渲染顶部卡片
5. 用户点 "Skills" tab → 拉 `/api/v1/skills?source=both&limit=100` → 列表
6. 用户点某个 skill → 拉 `/api/v1/skills/{id}` → 详情 + 版本时间线
7. 用户点 "Activate v2" → `POST /api/v1/skills/{id}/activate` body `{"target": "2"}` → 服务端回滚(见 §4.5)
8. **回滚本身**走 `SkillHub` + `fetch_version_bundle` + `write_skill_bundle(clean=True)`,**不会**自动触发 `service.sync()`——`dashboard.db` 还显示旧 v3;`POST /api/v1/sync` 重新 build 后才会刷新。**用户要手动点 dashboard 的 "Sync now" 按钮**(`POST /api/v1/sync`)。
9. 浏览器重新拉 → 看到 v2 是 current_version

## 9. 待确认 / 已知限制

- **Dashboard 启动时 sync 的错误吞掉**：`try/except/logger.exception` 不 re-raise——dashboard 启动后**仍**会 serve，但数据是过期的。**待确认**要不要给个 `GET /api/v1/sync_status` 让前端感知。
- **`activate_skill_version` 没有 confirm**：一次 POST 就回滚，没法 undo。**待确认**要不要加 `dry_run=true` 模式预览。
- **`export_local_sessions` 写入的 user_alias** 写死 `"local"`（dashboard 标注）或用 `cfg.sharing_user_alias`——但 `_local_sessions_from_snapshot` 不会覆盖 record 文件里已有的 `user_alias`。**待确认**是否一致。
- **`replace_snapshot` 是 `DELETE + INSERT`**：SQLite WAL 模式下**快**，但**不**是真正的 upsert（如果有外键 / trigger 会触发）。**待确认**未来要不要加 `INSERT OR REPLACE` 优化。
- **`_load_local_skills` 独立 parser** 和 `SkillManager` 行为可能漂移——比如 `_extra_frontmatter` 解析差异、category 优先级。**待确认**是否要抽个共用 frontmatter parser。
- **Dashboard 的 SQLite 没设 `PRAGMA foreign_keys=ON`**——所有表都是独立 PK，没外键约束，**不会**触发；**待确认**未来加关系表时是否需要。
- **`build_dashboard_snapshot` 跑得慢**（拉 1000+ skills × 多个 source）：一次可能要 10-30s。**待确认**要不要后台线程预热。

---

→ **下一篇**：[30-Evolve服务端与引擎选择](../03-Evolve服务/30-Evolve服务端与引擎选择.md) — `python -m evolve_server` 的入口和配置
