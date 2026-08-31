# 10 · CLI 与启动生命周期

## 术语速查(读本章前 1 分钟过一遍)

> 本章新引入的内部概念(`SetupWizard` / `daemon.start.lock` 等)只给最简释义;通用术语(PRM / Claw / Two Loops / Sharing Backend / Skill Backend)的权威定义见 [02 章 核心概念词典](../00-总览/02-核心概念词典.md)。

- **PRM (Process Reward Model)** —— 用第三方 LLM(OpenAI-compatible `/v1/chat/completions` 端点)对单条 turn 打 +1/-1/0,跑 `prm_m=3` 票取多数票。详见 16 章。(PRM 概念详见 [02 章 §10](../00-总览/02-核心概念词典.md))
- **Bedrock** —— AWS Bedrock 托管的 LLM 服务(Amazon 的模型市场)。当 `llm.provider == "bedrock"` 时,SkillClaw 用 `BedrockChatClient` + AWS region 调上游,不走 OpenAI 协议。
- **Claw** —— SkillClaw 对它所代理的本地 CLI agent 框架的统称(OpenClaw / Hermes / Claude Code / Codex / OpenCode / QwenPaw / IronClaw / PicoClaw / ZeroClaw / NanoClaw / NemoClaw / `none`)。(详见 [02 章 §5 / §6](../00-总览/02-核心概念词典.md))
- **Two Loops** —— 任务时 loop(`SkillClawAPIServer` 实时) + 演化时 loop(`evolve_server` 后台),通过共享存储通信。(详见 [02 章 §7](../00-总览/02-核心概念词典.md))
- **Sharing Backend** —— 客户端"通用对象存储"后端:`local` / `s3` / `oss` / `nacos`(nacos 仅承载 skill 资产)。(详见 [02 章 §12](../00-总览/02-核心概念词典.md))
- **Skill Backend** —— `sharing.skill_backend` 单字段覆盖,允许"skill 在 Nacos、session 在 OSS"的混合部署。(详见 [02 章 §13](../00-总览/02-核心概念词典.md))
- **SetupWizard** —— `setup_wizard.py:SetupWizard.run()` 交互式首次配置,**问 28 个问题**(详见 §2),用户全回车走默认也能跑通。

## 技术栈速查

本章涉及以下库/框架;首次出现时点一下,不展开:

- **Click** —— Python CLI 框架,类似 argparse 的升级版(`@click.group` / `@click.command`)。
- **FastAPI / uvicorn** —— ASGI web 框架 + 服务器(`SkillClawAPIServer` 在 `uvicorn.Server` 上跑)。
- **fcntl.flock / msvcrt.locking** —— POSIX / Windows 平台文件锁(用于 `daemon.start.lock`)。
- **subprocess.Popen** —— 后台 daemon 拉起机制。
- **asyncio** —— `SkillClawLauncher.start()` 跑在 `asyncio.run` 里,所有 I/O 协程化。

## 这一章讲什么

这一章回答"用户在终端敲 `skillclaw` 之后到底发生了什么"——从 Click 命令解析、`setup` 向导、`start` / `start --daemon` / `stop` / `status` / `config` / `doctor` / `restore` / `validation` / `dashboard` / `skills` 这一整套子命令的语义,到后台进程(daemon)的拉起、PID 锁、信号处理、launcher orchestrator。

## 它在整个系统的哪个位置

客户端运行时的"用户接口面"。所有"我敲 `skillclaw` 然后…"的逻辑都在这里。

## 设计目的

把 SkillClaw 的"操作面"和"运行面"分清楚:
- `setup` / `config` / `skills` / `doctor` / `restore` 是**一次性的、有终态的**操作
- `start` / `start --daemon` / `stop` / `status` 是**长期服务**操作
- `validation` / `dashboard` 是**可选后台 / 服务**操作

命令解析(Click)和运行时(`SkillClawLauncher` + FastAPI)的解耦让"前台跑 + 后台 daemon + systemd / launchd 接管"三种部署模式共存。

### 三种部署模式的选择

- **调试 / 临时**:`skillclaw start` 前台跑,日志直出 stdout,Ctrl-C 收。**注意**:前台模式的日志走 stdout,**不会**写 `~/.skillclaw/skillclaw.log`。
- **个人长期跑**:`skillclaw start --daemon`,日志追加进 `~/.skillclaw/skillclaw.log`,`stop` / `status` 控制。
- **生产**:`systemd` / `launchd` 拉 `skillclaw start --daemon`,让 daemon 在用户登出后仍能跑;`systemd` 负责 watchdog(OOM kill 后自动重启)。

---

## 1. CLI 入口与命令结构

### 1.1 入口

- 仓库入口点（`pyproject.toml`）：
  ```toml
  [project.scripts]
  skillclaw = "skillclaw.cli:skillclaw"
  skillclaw-evolve-server = "evolve_server.__main__:main"
  ```
- `skillclaw/__main__.py` 只有一行 `from .cli import skillclaw`（让 `python -m skillclaw` 也能用）
- `cli.py` 顶层 `click.group()` 函数名是 `skillclaw`（即 entry point 引用的对象）

### 1.2 命令树

```
skillclaw
├── setup                                 # 交互式首次配置
├── start    [--port N] [--daemon] [--log-file PATH]
├── stop
├── status
├── config   KEY_OR_ACTION [VALUE]
│            action: "show" 展示完整配置
│            KEY: 点号路径如 "proxy.port"，VALUE 省略时是读
├── doctor
│   ├── hermes
│   ├── codex
│   ├── claude
│   └── opencode
├── restore
│   ├── hermes   [--backup PATH]
│   ├── codex    [--backup PATH]
│   ├── claude   [--backup PATH]
│   └── opencode [--backup PATH]
├── validation
│   ├── status
│   └── run-once [--force]
├── dashboard
│   ├── sync    [--sharing-local-root ... --sharing-group-id ... ...]
│   └── serve   [--host ... --port ... --no-sync-on-start ...]
└── skills
    ├── push    [--no-filter]
    ├── publish  NAME VERSION [--no-update-latest]   # 仅 nacos skill backend
    ├── pull
    ├── sync
    └── list-remote
```

`evolve_server`（即 `python -m evolve_server`）是独立入口，详见 [30 · Evolve 服务端与引擎选择](03-Evolve服务/30-Evolve服务端与引擎选择.md)。

### 1.3 设计原则

- **每个子命令有 docstring**——Click 把它当 `--help` 输出
- **配置存在 `~/.skillclaw/config.yaml`**（YAML 格式）——所有子命令共享
- **子命令不直接 `print` 错误就退出**——用 `click.ClickException(...)` 抛，Click 统一打印为红色 + 非零退出码
- **跨子命令的同义逻辑尽量下沉到 helper**（如 `_sharing_backend` / `_sharing_target` / `_require_sharing` 都被 `skills` 子命令共享）

---

## 2. `setup`：交互式首次配置

入口在 `setup_wizard.py` 的 `SetupWizard.run()`。它按下面顺序向用户提问（每一步都打印提示语 + 接受回车用默认值）：

| 步骤 | 提问 | 默认值 | 写到 `config.yaml` 的字段 |
|---|---|---|---|
| 1 | CLI agent to configure | `openclaw` | `claw_type` |
| 2 | LLM provider | `custom` | `llm.provider`（内置 8 个预设：kimi / qwen / openai / minimax / novita / openrouter / bedrock / custom） |
| 3 | API base URL | 预设的 `preset["api_base"]` | `llm.api_base`（bedrock 留空） |
| 4 | Model ID | 预设的 `preset["model_id"]` | `llm.model_id` |
| 5 | API key | 上次配置的值 | `llm.api_key`（getpass 输入不回显） |
| 6 | OpenRouter 路由策略 | `fallback` | `openrouter.route`（4 选 1：fallback / price / throughput / latency） |
| 7 | OpenRouter fallback models | 空 | `openrouter.fallback_models`（逗号分隔） |
| 8 | OpenRouter data policy | `allow` | `openrouter.data_policy`（`allow` 时留空，`deny` 时写 `deny`） |
| 9 | Enable skill injection | `True` | `skills.enabled` |
| 10 | Skills directory | 跟 claw_type 适配的默认目录 | `skills.dir` |
| 11 | Enable PRM scoring | `True` | `prm.enabled` |
| 12 | PRM API URL | 同 LLM API URL | `prm.url` |
| 13 | PRM model ID | 同 LLM model ID | `prm.model` |
| 14 | PRM API key | 同 LLM API key | `prm.api_key` |
| 14.5 | PRM provider | 同 LLM provider | `prm.provider`(`openai` / `bedrock`;决定 PRMScorer 走 OpenAI 客户端还是 `BedrockChatClient`) |
| 15 | Enable shared skill storage | `False` | `sharing.enabled` |
| 16 | Storage backend | `s3` | `sharing.backend`（4 选 1：local / s3 / oss / nacos） |
| 17 | Group ID | `default` | `sharing.group_id` |
| 18 | User alias | 空 | `sharing.user_alias` |
| 19 | Auto-pull on startup | `False` | `sharing.auto_pull_on_start` |
| 20 | Local shared storage root | 空 | `sharing.local_root`（仅 local） |
| 21 | Storage endpoint | 空 | `sharing.endpoint` |
| 22 | Bucket name | 空 | `sharing.bucket` |
| 23 | Access key ID | 空 | `sharing.access_key_id` |
| 24 | Secret access key | 空 | `sharing.secret_access_key` |
| 25 | Region | 空（仅 s3） | `sharing.region` |
| 26 | Session token | 空（仅 s3） | `sharing.session_token` |
| 27 | Proxy model name exposed to agents | `skillclaw-model` | `proxy.served_model_name` |
| 28 | Proxy port | `30000` | `proxy.port` |

最后 `cs.save(data)` 写 YAML，并 `mkdir(parents=True, exist_ok=True)` 建好 skills_dir + 必要时 local_root。

`codex` 适配会额外提示 "After starting SkillClaw, run: `codex --profile skillclaw`"。

`SetupWizard` 用 `EOFError` / `KeyboardInterrupt` 兜底——Ctrl-C 不会崩，会回退到默认值。

---

## 3. `start [--port N] [--daemon] [--log-file PATH]`

按 `daemon` 是否指定走两条完全不同的路径。

### 3.0 启动 + 停止总时序图

```
  ┌─────────────┐                  ┌─────────────┐                  ┌─────────────┐
  │  用户终端    │                  │  CLI (click) │                  │  ConfigStore│
  └──────┬──────┘                  └──────┬──────┘                  └──────┬──────┘
         │ skillclaw start                │                                 │
         │ ──────────────────────────────►│                                 │
         │                                 │ 加载 ~/.skillclaw/config.yaml   │
         │                                 │ ────────────────────────────────►│
         │                                 │                                 │
         │                                 │ to_skillclaw_config() → cfg     │
         │                                 │ ◄────────────────────────────────│
         │                                 │                                 │
         │                                 │ (前台模式)                       │
         │                                 │ SkillClawLauncher(cs)            │
         │                                 │ asyncio.run(launcher.start())    │
         │                                 │                                 │
         │                                 │ (daemon 模式)                    │
         │                                 │ daemon_start_lock() — fcntl.flock│
         │                                 │ read_pid() → 检测到无运行实例    │
         │                                 │ _spawn_daemon_process()          │
         │                                 │ ├ Popen(start_new_session=True)  │
         │                                 │ │  / DETACHED_PROCESS +          │
         │                                 │ │  CREATE_NEW_PROCESS_GROUP      │
         │                                 │ │  env: SKILLCLAW_RUNTIME_KIND   │
         │                                 │ │       = "daemon"               │
         │                                 │ │  log_path via SKILLCLAW_RUNTIME│
         │                                 │ │  _LOG_PATH env                 │
         │                                 │ │  (子进程 launcher 启动时直接   │
         │                                 │ │  打开 log file,绕 ConfigStore)  │
         │                                 │ ├ _wait_for_daemon_ready()       │
         │                                 │ │   每 200ms 打 /healthz          │
         │                                 │ │   超时默认 15s                 │
         │                                 │ │   0=不超时,负数视为 15s        │
         │                                 │ │   超时后:(1) SIGTERM 整组     │
         │                                 │ │          (2) 等 grace          │
         │                                 │ │          (3) SIGKILL          │
         │                                 │ └ PID 写 ~/.skillclaw/skillclaw.pid│
         │                                 │   ← 父进程返回                   │
         │                                 │                                 │
         │  (前台模式进入 _run)            │                                 │
         │                                 │ _run(cfg)                       │
         │                                 │ ├ 构造 SkillManager(use_skills= │
         │                                 │ │  skills.enabled)              │
         │                                 │ ├ 构造 PRMScorer(prm_provider)  │
         │                                 │ │  bedrock→BedrockChatClient    │
         │                                 │ │  openai→自建 OpenAI 客户端    │
         │                                 │ │  缺 url+model→disable+warn    │
         │                                 │ ├ if sharing_enabled+auto_pull: │
         │                                 │ │  SkillHub.pull_skills(...)     │
         │                                 │ │  + skill_manager.reload()      │
         │                                 │ ├ SkillClawAPIServer.start()    │
         │                                 │ ├ wait_until_ready(30s)         │
         │                                 │ │  (30s 内未 ready→RuntimeError,│
         │                                 │ │   launcher 退出前清 PID)       │
         │                                 │ ├ claw_adapter.configure_claw() │
         │                                 │ │  1) 备份 ~/.skillclaw/backups/ │
         │                                 │ │     <claw>/<ISO8601>/         │
         │                                 │ │  2) 改写 agent 的本地 config  │
         │                                 │ │     (api_base + model)        │
         │                                 │ │  3) 记 ~/.skillclaw/state.json │
         │                                 │ ├ if validation_enabled:        │
         │                                 │ │  ValidationWorker.run()        │
         │                                 │ │  (同 event loop;stop 顺序:     │
         │                                 │ │   worker → server)             │
         │                                 │ └ await asyncio.sleep(1.0)       │
         │                                 │   (1.0s 是 stop 响应 vs CPU      │
         │                                 │    占用的折衷)                   │
         │  (Ctrl-C / SIGTERM)             │                                 │
         │ ───────────────────────────────►│                                 │
         │                                 │ _handler → launcher.stop()      │
         │                                 │ ├ _stop_event.set()              │
         │                                 │ ├ validation_worker.stop()       │
         │                                 │ ├ api_server.stop()              │
         │                                 │ │  (uvicorn should_exit=True;    │
         │                                 │ │   当前请求走 graceful drain    │
         │                                 │ │   ~30s uvicorn 默认)           │
         │                                 │ └ unlink PID 文件                │
         │  uvicorn should_exit=True       │                                 │
         │  → 在飞请求被 graceful drain     │                                 │
         │  → 完成后 server 退出           │                                 │
```

### 3.1 `start`(前台运行)

**场景**:用户在终端敲 `skillclaw start`(不带 `--daemon`)——server 在前台跑,Ctrl-C 触发优雅退出。

**为什么是"起 launcher + 跑 start + 接 Ctrl-C"3 步**:

- **段 1 构造 launcher**:`SkillClawLauncher(cs)` 接收 `ConfigStore`,内部组装 SkillClawAPIServer + validation worker + claw_adapter
- **段 2 跑 launcher**:launcher.start() 内部按"先 claw_adapter 改写 → 再起 SkillClawAPIServer → 再起 validation worker" 顺序跑
- **段 3 接 Ctrl-C**:KeyboardInterrupt 触发后,调 `launcher.stop()` 优雅退出(等价于 §3.5 `_shutdown_cleanup`)

**怎么走**:

```python
# 段 1:构造 launcher
launcher = SkillClawLauncher(cs)

# 段 2:跑 launcher 的 async 入口
try:
    asyncio.run(launcher.start())
    # launcher 内部:
    #   1. 跑 claw_adapter 改写 12 种 agent 配置
    #   2. 起 SkillClawAPIServer(uvicorn)
    #   3. 起 validation worker(后台 task)
except KeyboardInterrupt:
    # 段 3:Ctrl-C 触发
    launcher.stop()  # 调 _shutdown_cleanup
```

**`--port` 临时覆盖**:

如果传了 `--port`,临时把 `proxy.port` 写到 NamedTemporaryFile YAML、构造临时 `ConfigStore`、用完 unlink——**不污染**用户配置(用户 config.yaml 不动)。

**关键边界**:

- **Ctrl-C 一次**:KeyboardInterrupt 抛出,launcher.stop() 收尾(server 关闭 + session 排空)
- **Ctrl-C 多次**:`asyncio.run` 第二次 KeyboardInterrupt 抛 `RuntimeError`("Event loop stopped")——让用户知道"server 强退"
- **`asyncio.run` 创建新 event loop**:每次调 start 都新建 event loop(不共享全局 loop)
- **daemon 模式**:用 `--daemon` 走 §3.2 双 fork 路径,本节是前台路径
- **`launcher.stop()` 异常**:_shutdown_cleanup 内部 try/except,不让 shutdown 失败阻断进程退出

### 3.2 `start --daemon`（后台守护进程）

关键点：
1. `_ensure_daemon_not_running()`：读 PID 文件，如果进程还活着就报错
2. `_spawn_daemon_process()`：用 `subprocess.Popen` 启动一个新的 `python -m skillclaw start` 进程，参数 + 环境变量 `SKILLCLAW_RUNTIME_KIND=daemon` + `SKILLCLAW_RUNTIME_LOG_PATH=...`
3. `_wait_for_daemon_ready()`：每 200ms 打 `/healthz`，直到收到 `{"ok": true}` 或超时
4. 超时（默认 15s，可通过 `SKILLCLAW_DAEMON_READY_TIMEOUT_S` 改）就 SIGTERM / SIGKILL 整个进程组（**0 = 不超时**；**负数 = 视为 15s**；**正数 = 实际秒数**；建议 < 600）

**daemon 模式 env 变量**(子进程 launcher 读):
- `SKILLCLAW_RUNTIME_KIND=daemon` —— 子进程 launcher 读到后**跳过** `_spawn_daemon_process` 走前台 launcher 路径(避免无限 fork)。
- `SKILLCLAW_RUNTIME_LOG_PATH=<path>` —— 子进程 launcher 用此路径直接打开 log file,**绕 ConfigStore** 的 `proxy.log_file` 字段(因为 daemon 模式可能因 env 改变而希望独立控制 log 路径)。
- `SKILLCLAW_DAEMON_READY_TIMEOUT_S=<int>` —— `_wait_for_daemon_ready` 的超时(秒),默认 15。

`Popen` 在 Windows 用 `DETACHED_PROCESS | CREATE_NEW_PROCESS_GROUP`，在 POSIX 用 `start_new_session=True`——子进程独立进程组，不被父进程 SIGINT 影响。

日志路径默认 `~/.skillclaw/skillclaw.log`，父进程持有日志句柄，子进程写入。

### 3.3 `start` 后续（前台 / daemon 都走的）流程

`SkillClawLauncher.start()`（`launcher.py`）是 orchestrator：
1. `to_skillclaw_config()` 把 ConfigStore 转换成 `SkillClawConfig` dataclass
2. `_write_pid()` 写 `~/.skillclaw/skillclaw.pid`
3. `_setup_signal_handlers()` 注册 SIGTERM / SIGINT 调 `launcher.stop()`
4. `_run(cfg)`：
   - 构造 `SkillManager`（`use_skills=True` 时）→ 加载 `~/.skillclaw/skills/`（或 claw 适配的目录）
   - 构造 `PRMScorer`（`prm.enabled=True` 时），按 `prm.provider` 选 `BedrockChatClient` 或自建 OpenAI 客户端
   - **如果 `sharing_enabled=True && sharing_auto_pull_on_start=True`**：调 `SkillHub.pull_skills(cfg.skills_dir)` 拉最新 skill + 必要时 `skill_manager.reload()`
   - 构造 `SkillClawAPIServer(cfg, skill_manager=..., prm_scorer=...)` 并 `server.start()`
   - 等 `wait_until_ready(timeout_s=30.0)`（详情见 12 章）
   - **调 `claw_adapter.configure_claw(cfg)`**——这一步把 agent 的本地配置改成"指向 127.0.0.1:30000"
   - **如果 `validation_enabled=True`**：起 `ValidationWorker`，用 `asyncio.create_task(worker.run())` 在后台跑
   - 阻塞 `await asyncio.sleep(1.0)` 直到 stop event 触发
5. 退出时：停 validation worker → 停 server → unlink PID 文件

PRM provider 分支（`launcher._run`）：
- `prm_provider == "bedrock" && prm_model`：用 `BedrockChatClient(model_id=..., region=cfg.bedrock_region)`
- `prm_provider == "bedrock" && not prm_model`：警告 "PRM disabled"（因为 model 必填）
- `prm_url && prm_model`：自建 OpenAI 客户端
- 否则：警告并 disable

---

## 4. `stop` / `status`

### 4.1 `stop`

读 `~/.skillclaw/skillclaw.pid`，`os.kill(pid, SIGTERM)`。`ProcessLookupError` 时清理过期 PID 文件。其它异常回显到 stderr。

### 4.2 `status`

`runtime_state.process_alive(pid)` + `urllib.request.urlopen("http://127.0.0.1:PORT/healthz", timeout=2.0)`。
- 进程不在 → "not running"
- PID 在 / 进程不在 → "not running (stale PID file)" + 清 PID
- 进程在 + healthz 通 → "running (PID=N, proxy=:PORT)"
- 进程在 + healthz 不通 → "starting"

---

## 5. `config KEY [VALUE]`

`ConfigStore.set(key, value)` 用点号路径写入（如 `proxy.port 30001`），`set` 会用 `_coerce` 自动把字符串 `true`/`1` 转 bool / int / float。`config show` 走 `ConfigStore.describe()` 打印所有段（claw_type / llm / openrouter / proxy / skills / prm / sharing / evolve / validation / dashboard）的当前值。

---

## 6. `doctor` / `restore`（claw 集成诊断与回滚）

这两个 group 走 `claw_adapter.py`：
- `doctor hermes` / `doctor codex` / `doctor claude` / `doctor opencode`：调 `inspect_<claw>_config(cfg)` 返回一个 dict，CLI 用 `_echo_report` 打印有序字段（`status` / `integration_scope` / `config_path` / `config_exists` / `expected_model` / `configured_model` / `expected_base_url` / `configured_base_url` / `configured_provider` / `proxy_match` / `expected_skills_dir` / `skills_dir_exists` / `skills_dir_mode` / `legacy_skillclaw_skills_dir` / `legacy_skillclaw_skills_present` / `latest_backup` / `session_boundary_mode`），最后是 `issues` / `notes` / `next_steps` 列表。
- `restore <claw> [--backup PATH]`：调 `restore_<claw>_config(backup_path)`，从 `~/.skillclaw/backups/<claw>/` 里最新或指定的备份还原。

详见 [17 · 适配器矩阵](17-适配器矩阵.md)。

---

## 7. `validation` / `dashboard` / `skills`

### 7.1 `validation`

- `status`：调 `ValidationWorker.status_snapshot()` 一次性返回 `enabled` / `mode` / `sharing_enabled` / `group_id` / `user_alias` / `idle_after_seconds` / `poll_interval_seconds` / `max_jobs_per_day` / `jobs_completed_today` / `active_sessions` / `last_request_age_seconds` / `idle_now` / `open_jobs_for_me`
- `run-once [--force]`：调 `ValidationWorker.run_once(force=force)`，跑一轮 idle 验证

### 7.2 `dashboard`

- `sync`：`DashboardService(cfg).sync()` ——把本地 + 共享数据刷到 `~/.skillclaw/dashboard.db`
- `serve`：`serve_dashboard(cfg)` 起 FastAPI 在 `cfg.dashboard_host:cfg.dashboard_port`（默认 `127.0.0.1:3788`），可选 `--no-sync-on-start` 跳过启动同步

详见 [22 · Dashboard 与可观测性](02-客户端共享层/22-Dashboard与可观测性.md)。

### 7.3 `skills`

这一组子命令是"手动触发共享同步"：
- `push [--no-filter]`：默认带 effectiveness 过滤（`min_injections >= sharing_push_min_injections` 且 `eff >= sharing_push_min_effectiveness`），`--no-filter` 跳过
- `publish NAME VERSION [--no-update-latest]`：仅当 `sharing.skill_backend == "nacos"` 时可用
- `pull`：从共享存储镜像拉取到本地
- `sync`：pull 然后 push
- `list-remote`：列出共享存储里所有 skill

`push` / `pull` 都走 `SkillHub.from_config(cfg)` 路由（OSS / S3 / local / Nacos）。

`push` 命令会读 `~/.skillclaw/skills/skill_stats.json` 构造 `skill_filter`，`--no-filter` 时不过滤——`SkillHub.push_skills` 内部会按这个 filter 决定哪些 skill 上传。

详见 [20 · 共享存储与同步](02-客户端共享层/20-共享存储与同步.md)。

---

## 8. PID 锁与进程生命周期

`runtime_state.py` 的关键 API（被 CLI / launcher 共用）：

| 函数 | 行为 |
|---|---|
| `pid_file_path()` | 返回 `~/.skillclaw/skillclaw.pid` |
| `daemon_start_lock_path()` | 返回 `~/.skillclaw/daemon.start.lock`（防止两个 `start --daemon` 同时拉起） |
| `process_alive(pid)` | POSIX 用 `os.kill(pid, 0)`；Windows 用 `OpenProcess` + `GetExitCodeProcess` |
| `read_pid()` | 读 PID 文件，解析失败返回 `None` |
| `clear_pid()` / `clear_pid_if_matches(pid)` | 清 PID 文件 |
| `daemon_start_lock()` | context manager；用 `fcntl.flock`（POSIX）或 `msvcrt.locking`（Windows）做文件级互斥 |

`_spawn_daemon_process` 在 `daemon_start_lock` 上下文内做 `_ensure_daemon_not_running` + 拉起子进程——保证同一时间只能有一个 `start --daemon` 在跑（不论是手敲的还是 `SkillClaw` 脚本里循环触发的）。

POSIX `process_alive` 注意：当 PID 是僵尸时 `os.kill(pid, 0)` 返回成功但 `waitpid(WNOHANG)` 能拿到退出码——SkillClaw 的 `process_alive` 没处理这个，**待确认**：如果 daemon 因为 OOM 留下僵尸 PID 文件，下次 `start --daemon` 会被 `_ensure_daemon_not_running` 误判为在跑。实际场景里 daemon 是 uvicorn Server 主进程，不容易成僵尸，但要在生产里长期跑建议加 systemd / launchd 兜底。

---

## 9. 信号处理

`_setup_signal_handlers` 给 SIGTERM / SIGINT 注册同一个 handler：

```python
def _handler(signum, frame):
    logger.info("[Launcher] signal %s received — stopping …", signum)
    self.stop()
```

`stop()` 是同步方法，做四件事：
1. `self._stop_event.set()` —— 让主循环 `await asyncio.sleep(1.0)` 退出
2. `validation_worker.stop()`（如果存在）—— 置 `worker._stop_event`
3. `api_server.stop()`（如果存在）—— uvicorn `Server.should_exit = True`
4. `_PID_FILE.unlink(missing_ok=True)` —— 立刻清 PID 文件

FastAPI 那边还有一层（`lifespan` 退出时调 `_shutdown_cleanup`），详细见 [12 · 代理服务器总览](12-代理服务器总览.md)。

**异常分支**：`(OSError, ValueError)` 时静默忽略（`signal.signal` 在非主线程调用会抛这两个），保证在子线程里不会炸。

---

## 10. 与"安装"的边界

CLI 不参与安装。安装走 `scripts/install_skillclaw.sh`（macOS/Linux 一把梭；Windows 走 PowerShell 手动）：

```bash
python -m venv .venv
.venv/bin/pip install -e ".[evolve,sharing,server]"
.venv/bin/skillclaw setup   # 触发 SetupWizard
.venv/bin/skillclaw start --daemon
```

`install_skillclaw.sh` 的关键 flag：
- `--venv-dir PATH`：改 venv 位置（默认 `.venv`）
- `--extras LIST`：覆盖 `evolve,sharing,server`（可选 `all`）
- `--run-setup` / `--run-start`：装完直接跑 `setup` / `start`

`scripts/install_skillclaw_server.sh` 是**独立**的 venv（默认 `.venv-server`）——服务端通常和客户端分开部署。`.env` 文件从 `evolve_server/evolve_server.env.example` 拷过来手动改。

---

## 11. 待确认 / 已知限制

- **PID 文件跨平台**：`os.kill(pid, 0)` 在 Windows 不可用（虽然 `_process_alive_windows` 用了 `OpenProcess`，但 `os.kill` 在某些旧版 Python 仍会被调用）。生产建议在 Windows 用 `tasklist` 兜底——**待确认**当前 Windows 路径是否完整测试过。
- **`os.name == "nt"` 分支的 `creationflags` 行为**：当 Python 子进程不识别 `DETACHED_PROCESS` / `CREATE_NEW_PROCESS_GROUP` 常量（极旧 Windows）时会 fallback 到 `0`，等价于普通子进程——可能因为父进程退出而一起死掉。**待确认**在实际 Windows 10/11 上的行为。
- **`start --port` 的临时 YAML**：通过 `NamedTemporaryFile` 创建、用完 `unlink(missing_ok=True)`——理论上不应该泄漏，但如果 `os._exit` / `KeyboardInterrupt` 在 `yaml.dump` 之后、`unlink` 之前发生，会留下临时文件。生产里建议**别**频繁用 `--port`。

---

→ **下一篇**：[11 · 配置系统与字段归一化](11-配置系统.md) — `~/.skillclaw/config.yaml` 的每一个字段到底归谁管
