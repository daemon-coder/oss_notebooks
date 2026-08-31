# 30 · Evolve 服务端与引擎选择

## 这一章讲什么

这一章回答"`python -m evolve_server` 的入口是什么、`EvolveServerConfig` 怎么填、env vs skillclaw-config 两种来源、workflow / agent 两种引擎怎么选"。

## 它在整个系统的哪个位置

`evolve_server/__main__.py` 是入口，调用 `build_parser()` 解析 argv，构造 `EvolveServerConfig`，按 `--engine` 选 `EvolveServer`（workflow）或 `AgentEvolveServer`（agent）。`evolve_server.core.config.EvolveServerConfig` 是唯一的配置 dataclass。

## 设计目的

让服务端**能复用客户端配置**——`from_skillclaw_config(skillclaw_config)` 把 `SkillClawConfig` 的 sharing + LLM 字段映射过来；同时允许**环境变量覆盖**任何字段（方便 systemd / k8s 部署）。两种引擎共用 `EvolveEngineMixin`，差异只在"session 怎么变成 skill"。

---

## 1. 入口与命令

### 1.1 `python -m evolve_server`

```python
# 伪代码骨架（见 evolve_server/__main__.py:main）:
def main():
    args = build_parser().parse_args()
    config = _build_config_from_args(args)
    server = _build_server(config, mock=args.mock, mock_root=args.mock_root)
    # 一次性: --once 或 --mock → run_once() 后退出
    # 长跑: --port → run_periodic + FastAPI 并行；否则只 run_periodic
```

**4 种运行模式**：
- `--once`：跑一次 `run_once` + 打印 JSON summary
- `--mock`：用本地 `evolve_server/mock/` 当 bucket（默认 `evolve_server/mock`），一次跑完
- `--port N`：周期跑 + 起 FastAPI `/trigger` / `/status` / `/health` 端点
- 都不传：周期跑（默认 600s 一次，无 HTTP）

### 1.2 入口点（pyproject.toml）

```toml
[project.scripts]
skillclaw = "skillclaw.cli:skillclaw"
skillclaw-evolve-server = "evolve_server.__main__:main"
```

`skillclaw-evolve-server` 命令在 pip install 后可用。

### 1.3 CLI 选项（节选）

```
--engine {workflow,agent}      # 默认 workflow
--once                         # 跑一次就退
--mock [--mock-root PATH]      # 本地 mock 模式
--port N                       # 启 HTTP 端点
--interval N                   # 周期秒数（默认从 config）
--publish-mode {direct,validated}
--nacos-publish-mode {draft,review,direct}
--use-skillclaw-config         # 从 ~/.skillclaw/config.yaml 读 sharing + LLM
--storage-backend {local,oss,s3}
--local-root PATH              # 启用 local 存储
--model MODEL                  # 覆盖 LLM model
--llm-api-type {openai-completions,anthropic-messages,openai-responses,google-generative-ai,ollama}
--skill-verifier / --no-skill-verifier
--skill-verifier-min-score 0.75
--validation-required-results N
--validation-required-approvals N
--validation-min-mean-score 0.75
--validation-max-rejections N
--openclaw-bin / --openclaw-home   # agent engine 用
--fresh / --no-fresh            # agent engine 用
--agent-timeout 600
--workspace-root / --agents-md  # agent engine 用
--mock / --mock-root            # 本地存储
```

`_build_config_from_args` 把所有 CLI flag 应用到 config（CLI 优先于 env / skillclaw-config）。

## 2. `EvolveServerConfig` 完整字段

```python
# 伪代码骨架（见 evolve_server/core/config.py:EvolveServerConfig）:
@dataclass
class EvolveServerConfig:
    engine / storage_backend,endpoint,bucket,keys / group_id / local_root
    llm_api_key,base_url,model,max_tokens,temperature,api_type
    use_session_judge / use_skill_verifier / skill_verifier_min_score
    publish_mode: "direct" / "validated"
    validation_required_results / required_approvals / min_mean_score / max_rejections
    skill_storage_backend / nacos_* (Nacos 启用时) / interval_seconds / http_port
```

`__post_init__` 归一化：
- `engine` lowercase + 兜底 `"workflow"`
- `skill_verifier_min_score` 截到 [0, 1]
- `publish_mode` 必须是 `"direct"` / `"validated"`，否则归一为 `"direct"`
- `nacos_publish_mode` 必须在 `{draft, review, direct}`，否则 `"review"`
- `skill_reload_mode` 必须在 `{off, poll, callback}`，否则 `"poll"`
- `validation_*` 各项最低值：results / approvals / max_rejections ≥ 1；mean_score 截到 [0, 1]
- `engine="agent"` 时如果没设 `llm_model`，用 `_DEFAULT_AGENT_EVOLVE_MODEL = "gpt-5.4"`
- `openclaw_home` 空时用 `_PACKAGE_DIR / ".openclaw_home"`
- `workspace_root` 空时用 `_PACKAGE_DIR / "agent_workspace"`

## 3. 两种来源：env vs skillclaw-config

### 3.1 `from_env()`

读 `os.environ`（`_load_dotenv()` 在模块导入时把 `evolve_server/.env` 灌进去）：

| 字段 | env key | 默认 |
|---|---|---|
| `engine` | `EVOLVE_ENGINE` | `workflow` |
| `storage_backend` | `EVOLVE_STORAGE_BACKEND`（自动从 `EVOLVE_OSS_*` 或 `local_root` 推断） | `""` |
| `storage_endpoint` | `EVOLVE_STORAGE_ENDPOINT` / `EVOLVE_OSS_ENDPOINT` | `""` |
| `storage_bucket` | `EVOLVE_STORAGE_BUCKET` / `EVOLVE_OSS_BUCKET` | `""` |
| `storage_access_key_id` | `EVOLVE_STORAGE_ACCESS_KEY_ID` / `EVOLVE_OSS_KEY_ID` | `""` |
| `storage_secret_access_key` | `EVOLVE_STORAGE_SECRET_ACCESS_KEY` / `EVOLVE_OSS_KEY_SECRET` | `""` |
| `storage_region` | `EVOLVE_STORAGE_REGION` | `""` |
| `storage_session_token` | `EVOLVE_STORAGE_SESSION_TOKEN` | `""` |
| `local_root` | `EVOLVE_STORAGE_LOCAL_ROOT` / `EVOLVE_LOCAL_ROOT` | `""` |
| `group_id` | `EVOLVE_GROUP_ID` | `default` |
| `llm_api_key` | `OPENAI_API_KEY` | `""` |
| `llm_base_url` | `OPENAI_BASE_URL` | `https://api.openai.com/v1` |
| `llm_model` | `EVOLVE_MODEL` | `gpt-4o` |
| `llm_api_type` | `EVOLVE_LLM_API_TYPE` | `openai-completions` |
| `llm_max_tokens` | `EVOLVE_LLM_MAX_TOKENS` | `100000` |
| `llm_temperature` | `EVOLVE_LLM_TEMPERATURE` | `0.4` |
| `evolve_strategy` | `EVOLVE_STRATEGY` | `dynamic_edit_conservative` |
| `use_success_feedback` | `EVOLVE_USE_SUCCESS_FEEDBACK`（truthy ≠ {0,false,no}） | True |
| `evolve_batch_size` | `EVOLVE_BATCH_SIZE` | `20` |
| `reject_rewrite` | `EVOLVE_REJECT_REWRITE` | False |
| `use_session_judge` | `EVOLVE_USE_SESSION_JUDGE` | True |
| `use_skill_verifier` | `EVOLVE_USE_SKILL_VERIFIER` | False |
| `skill_verifier_min_score` | `EVOLVE_SKILL_VERIFIER_MIN_SCORE` | `0.75` |
| `publish_mode` | `EVOLVE_PUBLISH_MODE` | `direct` |
| `validation_required_results` | `EVOLVE_VALIDATION_REQUIRED_RESULTS` | `1` |
| `validation_required_approvals` | `EVOLVE_VALIDATION_REQUIRED_APPROVALS` | `1` |
| `validation_min_mean_score` | `EVOLVE_VALIDATION_MIN_MEAN_SCORE` | `0.75` |
| `validation_max_rejections` | `EVOLVE_VALIDATION_MAX_REJECTIONS` | `1` |
| `skill_storage_backend` | `EVOLVE_SKILL_STORAGE_BACKEND` | `""` |
| `nacos_server` | `EVOLVE_NACOS_SERVER` | `""` |
| `nacos_namespace_id` | `EVOLVE_NACOS_NAMESPACE_ID` | `public` |
| `nacos_access_token` | `EVOLVE_NACOS_ACCESS_TOKEN` | `""` |
| `nacos_username` | `EVOLVE_NACOS_USERNAME` | `""` |
| `nacos_password` | `EVOLVE_NACOS_PASSWORD` | `""` |
| `nacos_label` | `EVOLVE_NACOS_LABEL` | `latest` |
| `nacos_publish_mode` | `EVOLVE_NACOS_PUBLISH_MODE` | `review` |
| `skill_reload_mode` | `EVOLVE_SKILL_RELOAD_MODE` | `poll` |
| `proxy_reload_url` | `EVOLVE_PROXY_RELOAD_URL` | `""` |
| `proxy_reload_api_key` | `EVOLVE_PROXY_RELOAD_API_KEY` | `""` |
| `interval_seconds` | `EVOLVE_INTERVAL` | `600` |
| `http_port` | `EVOLVE_PORT` | `8787` |
| `history_path` | `EVOLVE_HISTORY_LOG` | `evolve_history.jsonl` |
| `processed_log_path` | `EVOLVE_PROCESSED_LOG` | `evolve_processed.json` |
| `openclaw_bin` | `AGENT_EVOLVE_OPENCLAW_BIN` | `openclaw` |
| `openclaw_home` | `AGENT_EVOLVE_OPENCLAW_HOME` | `""` |
| `fresh` | `AGENT_EVOLVE_FRESH` | `True` |
| `agent_timeout` | `AGENT_EVOLVE_TIMEOUT` | `600` |
| `workspace_root` | `AGENT_EVOLVE_WORKSPACE_ROOT` | `""` |
| `agents_md_path` | `AGENT_EVOLVE_AGENTS_MD` | `""` |

`engine="agent"` 时，`llm_api_key` / `llm_base_url` / `llm_model` / `llm_api_type` 优先用 `AGENT_EVOLVE_*` env（fallback `EVOLVE_*` / `OPENAI_*`）。

### 3.2 `from_skillclaw_config(skillclaw_config)`

**场景**:用户已经在跑 SkillClawAPIServer(`~/.skillclaw/config.yaml` 配好了 sharing + LLM 字段),想顺便在同一台机器起 evolve server——**不**想再配一遍 evolve 的 LLM 字段。`from_skillclaw_config` 把 SkillClaw 的 config 字段映射到 `EvolveServerConfig`,**复用**已有配置。

**`_build_config_from_args` 4 步映射规则**:

```python
def _build_config_from_args(args, skillclaw_config=None) -> EvolveServerConfig:
    # 段 1:读 sharing_* + llm_* + engine=agent
    #   engine="agent" 时,AGENT_EVOLVE_* env 优先于 EVOLVE_*
    sharing_args = read_sharing_args(args, skillclaw_config)
    llm_args = read_llm_args(args, skillclaw_config, engine=args.engine)

    # 段 2:storage_endpoint 禁用 nacos
    #   Nacos 不是对象存储接口,放 storage_endpoint 会冲突
    if sharing_args.sharing_backend == "nacos" and not sharing_args.session_backend:
        sharing_args.storage_endpoint = ""

    # 段 3:storage_backend 隐式推断
    #   顺序: session_backend > local_root > sharing_backend > endpoint 域名
    storage_backend = infer_storage_backend(sharing_args)

    # 段 4:LLM fallback 链
    #   key / base_url / model: llm_* → prm_* → env
    llm_key, llm_base, llm_model = resolve_llm_fallback(llm_args)

    return EvolveServerConfig(...)
```

**关键设计**(`storage_backend` 推断规则):

| sharing_backend | session_backend | 推断结果 |
|---|---|---|
| `nacos` | 没设 | `""`(空 string) → 强制 server 走 local/OSS/S3 之一 |
| `nacos` | `local` | `local` |
| `oss` | 没设 | `oss` |
| 任意 | 设了 | 用 `session_backend` |

**LLM fallback 链**(避免重复配):

`llm_key` / `llm_base_url` / `llm_model` 三个字段按 `llm_*` → `prm_*` → `env` 顺序回退——**优先用 llm_***(SkillClaw 主 LLM),**没有**才用 `prm_*`(PRM 用的 LLM),**还没有**才用 `env`。

**关键边界**:

- **SkillClaw config 没配 LLM 字段** (罕见):fallback 到 env vars
- **`engine="agent"` + `AGENT_EVOLVE_LLM_API_KEY` 设了**:env 覆盖 skillclaw_config
- **`sharing_backend="nacos"` + `session_backend="local"`** (混合部署):走 `from_skillclaw_config` 自动推断
- **storage_endpoint 是 Nacos URL** (误配):`_build_config_from_args` 会覆盖为空 string

## 4. 引擎选择

### 4.1 `workflow`(默认)

`EvolveServer` 继承 `EvolveEngineMixin`,`__init__` 走"config + 5 个子对象初始化":

```python
class EvolveServer(EvolveEngineMixin):
    def __init__(self, config: EvolveServerConfig, mock: bool = False, mock_root: str | None = None):
        self.config = config
        self._bucket = self._build_bucket(config, mock, mock_root)  # mock/OSS/S3/local
        self._prefix = f"{config.group_id}/"
        self._llm = AsyncLLMClient(...)  # LLM 客户端
        self._validation_store = ValidationStore(...)  # 验证存储
        self._id_registry = SkillIDRegistry()  # ID registry
        # 非 Nacos 模式启动时 load 已有 registry
```

`run_once` 11 步详见 [31 · Workflow 引擎详解](31-Workflow引擎详解.md) §1。

### 4.2 `agent`

`AgentEvolveServer` 继承 `EvolveEngineMixin`,`__init__` 走"workspace + runner + session_id 3 步":

```python
class AgentEvolveServer(EvolveEngineMixin):
    def __init__(self, config, mock=False, mock_root=None):
        # 段 1:构造工作区
        self._workspace = AgentWorkspace(config.workspace_root)

        # 段 2:构造 OpenClaw runner
        self._runner = OpenClawRunner(
            openclaw_bin=config.openclaw_bin,
            openclaw_home=config.openclaw_home,
            fresh=config.fresh,
            timeout=config.timeout,
            llm_api_key=..., llm_base_url=..., llm_model=..., llm_api_type=...,
        )

        # 段 3:算跨轮记忆用的稳定 session_id
        self._agent_session_id = f"evolve-{config.group_id}"
        # 同一 group 跨多次 cycle 用相同 session_id(配合 --no-fresh 模式读 MEMORY.md)
```

`run_once()`：
1. `_drain_sessions()`
2. `_summarize_sessions(sessions)`（重走 summarizer pipeline）
3. `if self.config.fresh: self._workspace.reset()`
4. 拉 manifest + 现有 skill bundles
5. `_workspace.prepare(sessions, existing_skills, manifest, agents_md, registry_info)`：把所有数据写到 `workspace/`
6. `_workspace.snapshot_skills()`：拿所有 skill 的 tree_sha 字典（"before"）
7. `await asyncio.to_thread(self._runner.run, workspace_path, message, session_id)`：调 OpenClaw 子进程
8. `_workspace.collect_changes(before_snapshot)`：diff 哪些 skill 改了
9. 逐个上传 changed skill
10. `_id_registry.save_to_oss(...)` + `delete_session_keys(...)` + `_workspace.cleanup_sessions()`

详见 [32 · Agent 引擎详解](32-Agent引擎详解.md)。

### 4.3 怎么选

| 场景 | 推荐 | 理由 |
|---|---|---|
| 团队 / 中心化 server | `workflow` | 行为可预测；LLM 调用次数固定；好审计 |
| 实验 / 单用户 / 想让 LLM 自由发挥 | `agent` | 给 LLM 整片 workspace + 完整 SKILL.md；让 LLM 自己改 |
| 调试 / CI 验证 pipeline | `workflow` + `--once` | 一次跑完、JSON summary |
| 跑通 baseline | `workflow` | 默认；最少依赖（不需要 OpenClaw） |
| 已有 OpenClaw 工具链 | `agent` | 复用 OpenClaw 的 LLM 调度、文件操作 |

**注意**：`agent` 引擎需要 `openclaw` 在 PATH（`openclaw_bin`），且 LLM 能调通 OpenClaw 的 chat 接口。

## 5. `EvolveEngineMixin`:两种引擎的共用基类

`engines/common.py`:`EvolveEngineMixin` 提供 5 个核心方法(2 个引擎共用):

| 方法 | 功能 | 用在哪 |
|---|---|---|
| `_build_bucket(config, mock, mock_root)` | 选 `LocalBucket(mock)` / `build_object_store(oss/s3/local)` | 构造 storage 抽象层 |
| `_uses_local_storage()` | 判定走同步还是异步 | `backend=="local" or self._mock or (LocalBucket + local_root)` |
| `_call_storage(func, *args)` | 调 storage——local 同步 / remote 走 `asyncio.to_thread` | 所有 storage IO 入口 |
| `_append_history(summary)` | 写 `evolve_history.jsonl`(失败只 logger.warning 不 raise) | Stage 8 finalize |
| `_drain_sessions()` | 列 sessions(`list_session_keys` + `read_json_object` → `(sessions, keys)`) | Stage 1 |

**辅助方法**:`_load_remote_skills`(读 manifest)/ `_sanitise_name`(规范化 skill name)/ `_call_storage_async` 包装等。

**`_call_storage` 的 local/remote 分支**(最常用方法,每段 storage IO 都走它):

```python
async def _call_storage(self, func, *args):
    if self._uses_local_storage():
        return func(*args)  # local 同步
    return await asyncio.to_thread(func, *args)  # remote 异步
```

- **local store** (filesystem) 走同步——filesystem 调够快,放线程池反而引入 context switch 开销
- **remote store** (OSS / S3) 走 `asyncio.to_thread`——boto3 / oss SDK 是阻塞,不放线程池会卡死 event loop

**`AsyncLLMClient` 自己也用 `asyncio.to_thread`**——所以 OSS 读和 LLM 调都是非阻塞的。event loop 同时跑 cycle / HTTP / session 簿记都不卡。

## 6. 完整启动示例

```bash
# 1. 用 skillclaw-config（团队共享 + evolve 也在同台机器）
skillclaw-evolve-server --use-skillclaw-config --interval 300 --port 8787

# 2. 纯 env 模式（独立机器，跑 OSS）
export EVOLVE_ENGINE=workflow
export EVOLVE_STORAGE_BACKEND=oss
export EVOLVE_STORAGE_ENDPOINT=https://oss-cn-hangzhou.aliyuncs.com
export EVOLVE_STORAGE_BUCKET=my-bucket
export EVOLVE_STORAGE_ACCESS_KEY_ID=...
export EVOLVE_STORAGE_SECRET_ACCESS_KEY=...
export EVOLVE_GROUP_ID=my-team
export OPENAI_API_KEY=sk-...
export EVOLVE_MODEL=kimi-k2.5
export EVOLVE_INTERVAL=300
export EVOLVE_PUBLISH_MODE=validated
export EVOLVE_VALIDATION_MIN_MEAN_SCORE=0.8
skillclaw-evolve-server --port 8787

# 3. 跑一次（CI / 调试）
skillclaw-evolve-server --use-skillclaw-config --once

# 4. mock 模式（完全本地、无外部依赖）
skillclaw-evolve-server --mock --mock-root /tmp/skillclaw-mock --once
```

## 7. 待确认 / 已知限制

- **`engine="agent"` 必须装 `openclaw`**：CLI 不会校验——`_build_server` 直接构造 `AgentEvolveServer`（不装也能构造），只有 `runner.run` 时 `subprocess.run(["openclaw", ...])` 才会因为 `FileNotFoundError` 失败。**待确认**是否要给 server startup 加预校验。
- **`from_skillclaw_config` 的 Nacos 检测**：`sharing_backend == "nacos" and not session_backend` → 强制 `storage_endpoint=""`——这会触发**server 端 storage_backend 推断失败**（如果没有 `local_root`）。`__main__.main` 会 `SystemExit(1)`。**待确认**是不是该自动 fallback 到 mock / local。
- **`evolve_history.jsonl` 无界增长**：`_append_history` 只 append，不 rotate。生产 1 年会变 GB 级。**待确认**要不要加 rotate。
- **`processed_log_path` 字段没被读**：`evolve_processed.json` 在 dataclass 里但没代码消费——`workflow` / `agent` 都用 `delete_session_keys` 立即删已处理 session。**待确认**这是 dead code 还是 future use。
- **`_load_remote_skills` 在 workflow 里被 Nacos 覆盖**：`EvolveServer._load_remote_skills` 优先 Nacos，否则 super。`AgentEvolveServer` **没**实现这个 override——agent 引擎下 Nacos 不工作。**待确认**是不是 bug。
- **`_notify_proxy_reload` 的 URL 校验**：`mode != "callback" or not url` 时直接 return。**生产**应该测一下 callback 端的可达性——失败只 log warning。

---

→ **下一篇**：[31-Workflow引擎详解](31-Workflow引擎详解.md) — 6 段流水线每一步干什么
