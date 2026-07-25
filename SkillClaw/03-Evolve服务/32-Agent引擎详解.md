# 32 · Agent 引擎详解

## 术语速查（读本章前 1 分钟过一遍）

- **OpenClaw** —— 一个**外部的 agent 运行时**（AMAP-ML 同团队开源，仓库 `https://github.com/AMAP-ML/openclaw`；**不是** SkillClaw 内部子模块；`--llm-api-type=anthropic-messages` 时调 `openclaw` 自带的 chat 客户端，workflow 引擎调 Anthropic 走 `bedrock_client`，agent 引擎和 workflow 引擎是两条独立路径）。SkillClaw **用 `subprocess.run` 调它的 binary**（`openclaw agent --session-id ... --message ... --json --local --timeout ...`），把"读 sessions + 改 SKILL.md"这件事外包给它。**这跟 17 章 12 种 CLI agent 适配是两类不同的事**——那些是 SkillClaw **替**它们配（`configure_openclaw` 等），这个是 SkillClaw **反过来调**它。
- **`workspace_root` vs `openclaw_home`（两个 HOME，正交两件事）**：
  ```
  workspace_root  = SkillClaw 喂给 OpenClaw 的"工作区"目录
                    由 AgentWorkspace 管;每 cycle prepare() 全量重写
                    默认 <evolve_server_pkg>/agent_workspace
                    (来自 AgentEvolveServer.__init__ 取 config.workspace_root;
                     30 章 env 表里 env key = AGENT_EVOLVE_WORKSPACE_ROOT,
                     空时 __post_init__ 用 _PACKAGE_DIR/"agent_workspace")
                    装什么:sessions/ + skills/ + manifest.json + skill_registry.json
                           + EVOLVE_AGENTS.md + AGENTS.md / SOUL.md / IDENTITY.md / USER.md
  openclaw_home   = OpenClaw 自己的 HOME,放 .openclaw/openclaw.json
                    + .openclaw/agents/main/sessions/sessions.json
                    + OpenClaw 自己的 MEMORY.md / memory/ (memory-core 管)
                    由 OpenClawRunner 管;--fresh=True 时 wipe, --no-fresh 时保留
                    默认 <evolve_server_pkg>/.openclaw_home
                    (env key = AGENT_EVOLVE_OPENCLAW_HOME,空时 _prepare_home 取
                     Path.cwd() / ".openclaw_home"——是 OpenClawRunner 构造时的 cwd,
                     不是 agent.py 启动时的 cwd;30 章用 _PACKAGE_DIR 是另一条路径,
                     见 [待核-R4])
  --fresh=True  → _prepare_home wipe openclaw_home(OpenClaw 自己的 MEMORY.md 等)
                → workspace_root 每次 prepare() 都重写,与 --fresh 无关
  --no-fresh    → openclaw_home 保留 → OpenClaw 能读上轮 MEMORY.md
  ```
  关键决策点:`--fresh` 只影响 `openclaw_home`,**不**影响 `workspace_root`——后者每次 `prepare()` 都重建。`--no-fresh` + 跨轮 `session_id` 是 OpenClaw 跨轮记忆的两个支撑。
- **EVOLVE_AGENTS.md** —— SkillClaw 喂给 OpenClaw 的**完整指令文件**(`evolve_server/engines/EVOLOLVE_AGENTS.md`→`EVOLVE_AGENTS.md`,407 行;`wc -l engines/EVOLVE_AGENTS.md` 可核对),告诉 OpenClaw "你的任务是改 SKILL.md;请按这个流程走"。**要改 agent 行为,改这份 markdown,不用动 Python**。§5 给出 7 步流程概览,完整 prompt 文本以原文件为准。
- **agent 引擎** —— SkillClaw 的 `evolve_server/engines/agent.py`(524 行)+ `agent_workspace.py`(310 行)+ `openclaw_runner.py`(209 行)+ `agents_md.py`(14 行)+ `EVOLVE_AGENTS.md`(407 行) 实现,负责"准备 workspace → 调 OpenClaw → 收集 diff → 上传"。
- **OpenClaw 的 4 个关键约定**(本章反复用到):
  1. `AGENTS.md` 协议 —— OpenClaw 启动时读 `<workspace>/AGENTS.md` 作为指令;
  2. `ensureAgentWorkspace()` —— OpenClaw 自己的初始化函数,创建 workspace 时用 `writeFileIfMissing(flag='wx')`(`wx` = write exclusive,文件存在则不覆盖);
  3. `openclaw.json` —— OpenClaw 的配置文件,SkillClaw 喂给它;
  4. `~/.openclaw/agents/main/sessions/sessions.json` —— OpenClaw 的 session 持久化路径(OpenClaw 自己管,SkillClaw **不**写)。
- **EVOLVE_AGENTS.md 的 4 条禁区**(对 agent 的硬约束,与上面 4 个关键约定**不是同一组**——4 个关键约定是 OpenClaw 自己的协议,4 条禁区是 SkillClaw 借 EVOLVE_AGENTS.md 立的"不要碰"清单):
  1. 所有 file 操作必须留在 workspace 内;
  2. 不修改 `sessions/` / `manifest.json` / `skill_registry.json`(只读);
  3. 只在 `skills/<name>/` 写变化;
  4. 每次改完必须 self-validate。
- **3 个清单文件**(agent 引擎涉及到的清单,关系见 §4.5):
  - `<workspace>/manifest.json` —— workspace 视图,OpenClaw 只读;
  - `<bucket>/<group_id>/manifest.jsonl` —— 共享存储清单,SkillClaw 写,客户端读;
  - `<bucket>/<group_id>/evolve_skill_registry.json` —— SkillIDRegistry 持久化(全 cluster 共享一份)。
- **可观测层 `evolve_history.jsonl`** —— 本地文件,每个 cycle 末尾 append 一行 summary,用于排障与生产监控(详见 §4.6)。路径由 `EVOLVE_HISTORY_LOG` env 控制,默认 `evolve_history.jsonl`(相对路径,相对 evolve server 启动时的 cwd)。

## 它在整个系统的哪个位置

`evolve_server/engines/agent.py`(524 行) + `agent_workspace.py`(310 行) + `openclaw_runner.py`(209 行) + `agents_md.py`(14 行) + `EVOLVE_AGENTS.md`(407 行)。

agent 引擎与 workflow 引擎共享基类 `EvolveEngineMixin`(5 个方法: `_build_bucket` / `_uses_local_storage` / `_call_storage` / `_append_history` / `_drain_sessions`)。`AgentEvolveServer` 在此基础上加 `AgentWorkspace` + `OpenClawRunner` 两块,`_id_registry = SkillIDRegistry()` 是引擎间共享的(但每个 engine instance 自己 load/save)。

## 设计目的

让"skill 演化"在 workflow 引擎的 6 段流水线之外再开一条路:**给 LLM 一片带 sessions 的工作区,让它自己读 session、自己改 SKILL.md**。agent 引擎适合"我想看 LLM 自由发挥会改出什么样的 skill"——比 workflow 更不可预测,但**对 LLM 编码能力的利用更充分**。和 workflow 引擎共享存储后端、SkillIDRegistry、可观测层;**只**在"决定怎么改 SKILL.md"这一段换路。

## 本章范围

**涵盖**:workspace 准备 / OpenClaw 子进程调度 / upload 协议 / EVOLVE_AGENTS.md 协议摘要 / 跨轮记忆机制 / 可观测层 / `_AnthropicMessagesLLMClient` 细节 / SkillIDRegistry 细节 / `_summarize_sessions` LLM 选型。
**不涵盖**:workflow 引擎的 6 段流水线(见 [31 · Workflow 引擎详解](31-Workflow引擎详解.md))/ 共享存储后端实现细节(见 [33 · 共享层与存储适配](33-共享层与存储适配.md))/ Nacos 适配器(见 [20 · 共享存储与同步](../02-客户端共享层/20-共享存储与同步.md))/`AsyncLLMClient` 内部(见 30 章)/ 配置全字段(见 [30 · Evolve 服务端与引擎选择](30-Evolve服务端与引擎选择.md))。

---

## 1. 顶层 10 步

```python
# 伪代码骨架（见 agent.py:AgentEvolveServer.run_once）:
async def run_once(self) -> dict:
    sessions, session_keys = await self._drain_sessions()
    if not sessions: return empty_summary
    # 10 步: drain → summarize → reset(可选) → fetch → prepare → snapshot
    #       → run(OpenClaw 子进程) → collect → upload → finalize(safe+delete+cleanup+history)
    return summary
```

**10 步**(与 [30 · 4.2 agent](30-Evolve服务端与引擎选择.md) 一致):

| # | 步 | 在源码里叫什么 | 在哪一节讲 |
|---|----|---------------|----------|
| 1 | Drain | `self._drain_sessions()` | 见 31 章 §2 |
| 2 | Summarize | `self._summarize_sessions(sessions)` | §6 |
| 3 | Reset(可选) | `if self.config.fresh: self._workspace.reset()` | §2 / §3.2 |
| 4 | Fetch all skills | `self._load_remote_skills()` + `self._fetch_all_skills(manifest)` | §2.2 |
| 5 | Prepare workspace | `self._workspace.prepare(...)` | §2.2 |
| 6 | Snapshot | `self._workspace.snapshot_skills()` | §2.3 |
| 7 | Run agent | `await asyncio.to_thread(self._runner.run, ...)` | §3.5 |
| 8 | Collect changes | `self._workspace.collect_changes(before_snapshot)` | §2.4 |
| 9 | Upload(逐个 change) | `self._upload_skill(...)` × N | §4 |
| 10 | Finalize(safe+delete+cleanup+history) | `save_to_oss` + `delete_session_keys` + `cleanup_sessions` + `_append_history` | §4.4 + §4.6 + §2.5 |

> **Step 7 / 8 / 9 / 10 是一段流水线,不是并行的**——OpenClaw 子进程跑完才能 collect,collect 完才能 upload,upload 完才能 finalize。Step 9 内部对每个 change 独立跑"upload → record_update"两步,中间失败跳到下一个 change(不打断 cycle),见 §4.4。

## 2. `AgentWorkspace`(`agent_workspace.py`,310 行)

对应 10 步里的 Step 3(可选 reset)/ Step 5(prepare)/ Step 6(snapshot)/ Step 8(collect)/ Step 10 部分(cleanup_sessions)。

### 2.1 工作区布局

```
<workspace_root>/
├── AGENTS.md                  # SkillClaw 写的,指向 EVOLVE_AGENTS.md
├── SOUL.md                    # 最小 bootstrap(避免 OpenClaw 浪费 tokens)
├── IDENTITY.md
├── USER.md
├── EVOLVE_AGENTS.md           # 完整演化指南(来自 engines/EVOLVE_AGENTS.md)
├── TOOLS.md                   # SkillClaw 不预写,留给 OpenClaw
├── MEMORY.md                  # SkillClaw 不预写,留给 OpenClaw(跨轮记忆的载体)
├── memory/                    # 同上
├── sessions/<session_id>.json # compact session 数据
├── skills/<name>/
│   ├── SKILL.md
│   ├── references/
│   ├── scripts/
│   ├── assets/
│   └── history/               # 仅 --no-fresh 模式积累
├── manifest.json              # 只读视图
└── skill_registry.json        # 只读视图
```

**关键设计**:`AGENTS.md` / `SOUL.md` / `IDENTITY.md` / `USER.md` 不是 OpenClaw 默认的——`ensureAgentWorkspace` 用 `writeFileIfMissing`(flag `'wx'`)创建 bootstrap 文件,**不覆盖已存在的**。SkillClaw 提前把这 4 个文件预写进去,OpenClaw 跳过;`TOOLS.md` / `MEMORY.md` / `memory/` **不**预写,留给 OpenClaw 自己管(尤其是 `MEMORY.md` 是 `--no-fresh` 跨轮记忆的载体——见术语速查的"两个 HOME"框)。

**frontmatter 解释**:OpenClaw 读 SKILL.md 时先解析 frontmatter(`---` 包裹的 YAML),拿 `name` / `description` / `metadata` 等元数据;`EVOLVE_AGENTS.md` 要求 frontmatter 必须有 `name` 和 `description`,`category` 可选。`parse_skill_content(name, raw_md)` 是 SkillClaw 的 frontmatter + body 解析函数(详见 33 章)。

**`clean=True` vs `clean=False`**:AgentWorkspace 写 `skills/<name>/` 时固定 `clean=True`(调用 `write_skill_bundle(skill_dir, content, clean=True)`),**没有** `clean=False` 路径——因为 `--no-fresh` 模式下需要保留 OpenClaw 在 `history/` 加的 evidence 文件,而 `clean=True` 删**整个** skill 目录再重建。SkillClaw 的取舍:让 `clean=True` 删掉 OpenClaw 之前加的辅助文件,只重新写 SkillClaw 知道的子集(`SKILL.md` + bundle 里有的),**但** `history/` 由 `clean=True` 也删——所以 `--no-fresh` 模式 history/ 实际**不**保留(重新 prepare 时被删),OpenClaw 跨轮记忆的真正载体是 `openclaw_home/MEMORY.md` 而不是 `workspace/skills/<name>/history/`。

### 2.2 `prepare(sessions, existing_skills, manifest, agents_md, skill_registry_info)`

```python
def prepare(self, sessions, existing_skills, manifest, agents_md, skill_registry_info=None):
    # 1. 清 sessions/,写新 session JSONs(compact: 9 字段,见下)
    # 2. 清多余的 skills/ 子目录;按 existing_skills 写 skills/(clean=True)
    # 3. 写 manifest.json(workspace 视图,只读)
    # 4. 写 skill_registry.json(workspace 视图,只读)
    # 5. 写 EVOLVE_AGENTS.md(完整 407 行)
    # 6. 预写 AGENTS.md / SOUL.md / IDENTITY.md / USER.md(被 OpenClaw 用 'wx' 跳过)
    # 7. 不预写 TOOLS.md / MEMORY.md / memory/(留给 OpenClaw 自己管)
```

**compact session 的 9 个字段**(写进 `sessions/<sid>.json`):

| 字段 | 含义 | 来源 |
|------|------|------|
| `session_id` | session UUID | session 顶层 |
| `task_id` | 任务的业务 ID(不是 session_id) | session.metadata.task_id(详见 14 章) |
| `num_turns` | session 内 turn 数 | session.turns 长度 |
| `aggregate` | session 维度聚合指标 dict | `_summarize_sessions` 阶段填:含 `avg_prm` / `skill_usage_count` / `tool_error_count` 等,agent 用来快速判断 session 整体健康度 |
| `_skills_referenced` | list[str],session 内引用的 skill name | session 累积(去重 + 排序) |
| `_avg_prm` | float,session 平均 PRM 分 | session 累积 |
| `_has_tool_errors` | bool,session 是否有 tool 错误 | session 累积 |
| `_trajectory` | list[dict],step-by-step LLM 调用记录 | summarizer 程序生成(无损) |
| `_summary` | str,LLM 生成的 session 摘要 | summarizer LLM 生成 |

### 2.3 `snapshot_skills()`

```python
def snapshot_skills(self) -> dict[str, str]:
    snapshot = {}
    for skill_dir in sorted(self.skills_dir.iterdir()):
        if not skill_dir.is_dir(): continue
        bundle = read_skill_bundle(skill_dir)
        if "SKILL.md" in bundle:
            snapshot[skill_dir.name] = bundle_tree_sha256(bundle)
    return snapshot
```

返回 `{skill_name: tree_sha256}`——给 Step 8 `collect_changes` 做 diff。**bundle_tree_sha256** 把 bundle 里所有文件按 path 排序后算 `sha256(join(content_bys))`,所以任意子文件变动都会让 hash 变。

### 2.4 `collect_changes(before_snapshot)`

```python
def collect_changes(self, before_snapshot):
    # after = 当前 workspace/skills 树
    # diff before vs after: sha 不同 → action=create(新增) / improve(修改)
    # 注: 删掉的 skill 会被 logger.warning 忽略(不支持 delete action)
    return [{"name", "action", "skill", "bundle_files", "tree_sha256"}, ...]
```

**`action`** 判定:before_snapshot 里**没有**这个 name → `create`;**有** + sha 变了 → `improve`。
**`deleted` 不算** change(agent 删了 skill 不上传)——这一条是 design choice:**不**实现 delete action,只 create / improve。
**conflict 不处理**:`collect_changes` 只看本地的 before/after,**不**调 `_detect_conflict` / `_resolve_and_upload`(那是 workflow 引擎的 31 章 §6 范围);agent 改了 skill X 但 server 端 sha 已被别人改过——**没有 merge**,agent 的版本直接覆盖(见 §8.2 错误路径)。

### 2.5 `cleanup_sessions()`

删 `sessions/` 整个目录(保留其它数据,**不** reset workspace)——下次 cycle 的 `prepare` 会重建。这是 Step 10 的 3 个 finalize 子动作之一,另外两个见 §4.4。

## 3. `OpenClawRunner`(`openclaw_runner.py`,209 行)

对应 10 步里的 Step 3(`_prepare_home`)/ Step 7(`run`)/ Step 4 间接(`_write_config` 把 workspace 路径告诉 OpenClaw)。

### 3.1 构造

```python
OpenClawRunner(
    openclaw_bin="openclaw",            # 默认值在 PATH;env key = AGENT_EVOLVE_OPENCLAW_BIN
    openclaw_home=Path(""),             # 默认 <cwd>/.openclaw_home(本 runner 构造时的 cwd)
    fresh=True,                         # env key = AGENT_EVOLVE_FRESH("1"/"true" 视为 True)
    timeout=600,                        # env key = AGENT_EVOLVE_TIMEOUT,默认 600s
    llm_api_key, llm_base_url, llm_model, llm_api_type,
)
```

**`_openclaw_dir` / `_config_path`**:`self._openclaw_dir = self.openclaw_home / ".openclaw"`,`self._config_path = self._openclaw_dir / "openclaw.json"`。`openclaw_home` 是 OpenClaw 自己的 HOME,`_openclaw_dir` 才是 OpenClaw 真正读 config / sessions 的子目录。

**`config.group_id` 来源**:`self._agent_session_id = f"evolve-{config.group_id}"`——`config.group_id` 来自 `EvolveServerConfig.group_id`,用户启动 evolve server 时通过 `EVOLVE_GROUP_ID` env 传入(默认 `"default"`,详见 30 章)。

### 3.2 `_prepare_home()`

```python
def _prepare_home(self):
    if self.fresh:
        if self.openclaw_home.exists():
            shutil.rmtree(self.openclaw_home, ignore_errors=True)
    self.openclaw_home.mkdir(parents=True, exist_ok=True)
    self._openclaw_dir.mkdir(parents=True, exist_ok=True)
```

**`fresh=True` 时彻底 wipe `<openclaw_home>`**——OpenClaw 自己的 memory / sessions / state 全部清零。
**`fresh=False` 时保留**——后续轮可以读上一轮 OpenClaw 的 memory(`MEMORY.md` / `memory/` / `agents/main/sessions/sessions.json`)。**`mkdir(parents=True, exist_ok=True)` 不删**——这是 `--no-fresh` 跨轮记忆的核心机制:**保留什么**取决于上轮 OpenClaw 自己写的内容(OpenClaw 内部的 memory-core 在管),SkillClaw 这边只做"如果存在就复用"。

### 3.3 `_write_config(workspace_path)`

写 `<openclaw_home>/.openclaw/openclaw.json`——SkillClaw **真正决策的字段**只有 6 个(用紧凑伪 JSON 表示):

```json
gateway.mode=local, gateway.bind=loopback
models.providers.evolve-llm={api:<llm_api_type>, baseUrl:<llm_base_url>,
                             apiKey:<llm_api_key>,
                             models:[{id:<llm_model>, name:<llm_model>}]}
agents.defaults.model.primary=evolve-llm/<llm_model>
agents.defaults.sandbox.mode=off
agents.defaults.workspace=<workspace_path absolute>
```

**6 个决策字段**:`gateway.mode=local` / `gateway.bind=loopback` / `models.providers.evolve-llm.{api,baseUrl,apiKey,models[0].id,name}` / `agents.defaults.model.primary` / `agents.defaults.sandbox.mode=off` / `agents.defaults.workspace`。

**其它字段都是 OpenClaw schema 默认值,SkillClaw 不动**(包括 `models[*].contextWindow=200000` / `maxTokens=16384` / `cost.{input,output,cacheRead,cacheWrite}=0` / `reasoning=false` / `input=["text"]`)。SkillClaw 在 `_write_config` 里**只**设上面 6 个决策字段,如果 `openclaw.json` 已有这些字段先 `setdefault`,不覆盖;之后用 `config.setdefault("gateway", {})` / `config["agents"] = ...` 写入 SkillClaw 决策值。

**`gateway.mode=local` + `bind=loopback`**:不让 OpenClaw 暴露端口(本地执行即可);`--local` CLI flag(见 §3.5)和 `gateway.mode=local` JSON 配置**等价**——CLI flag 优先级更高(后解析覆盖先解析),所以 `openclaw agent --local` 是 source of truth,JSON 是兜底。

**`agents.defaults.workspace`**:让 OpenClaw 把工作区切到 SkillClaw 的 workspace 路径(`workspace_path.resolve()` 绝对路径)。

**`sessions.json` bootstrap**:`agents/main/sessions/sessions.json` 如果不存在则写空对象 `{}`——OpenClaw 启动需要这个文件存在。

### 3.4 `_build_env()`

```python
def _build_env(self) -> dict[str, str]:
    env = dict(os.environ)
    env["HOME"] = str(self.openclaw_home)                    # 隔离:见下
    env["OPENCLAW_HOME"] = str(self.openclaw_home)
    env["OPENCLAW_CONFIG_PATH"] = str(self._config_path)
    env["HF_HUB_OFFLINE"] = "1"                              # 禁用 HF Hub 联网
    env["TRANSFORMERS_OFFLINE"] = "1"                        # 禁用 transformers 联网
    return env
```

**`HOME` 覆盖**:`env["HOME"] = str(self.openclaw_home)` —— **有意为之**,隔离 OpenClaw 子进程对用户 home 目录的访问(避免 OpenClaw 误读 `~/.aws/credentials` / `~/.bashrc` 等)。如果 OpenClaw 真的需要访问用户真实 home,**必须**用绝对路径硬编码而不是依赖 `$HOME`。

**`HF_HUB_OFFLINE=1` + `TRANSFORMERS_OFFLINE=1`**:禁用 Hugging Face Hub 和 transformers 库的联网检查——保证 OpenClaw 不会在子进程启动时去拉新模型(网络不可用时直接报错而不是 hang)。

**`OPENCLAW_HOME` / `OPENCLAW_CONFIG_PATH`**:OpenClaw 自己读这两个 env(可能跟 `HOME` 重复,但 OpenClaw 内部优先用这两个)——确保 OpenClaw 找到 SkillClaw 写的 `openclaw.json` 而不是用户 home 下的同名文件。

### 3.5 `run(workspace_path, message, session_id=None)`

```python
def run(
    self,
    workspace_path: Path,
    message: str,
    session_id: str | None = None,
) -> subprocess.CompletedProcess[str]:
    if session_id is None:
        session_id = f"evolve-{uuid.uuid4().hex[:12]}"

    self._prepare_home()
    self._write_config(workspace_path)

    cmd = [
        self.openclaw_bin, "agent",          # 子命令是 "agent" 不是 "runner"
        "--session-id", session_id,
        "--agent", "main",
        "--message", message,
        "--json",                              # OpenClaw 输出 JSON 格式
        "--local",                             # 本地模式(等价 gateway.mode=local,CLI 优先)
        "--timeout", str(self.timeout),
    ]
    env = self._build_env()

    result = subprocess.run(
        cmd,
        cwd=str(workspace_path),               # OpenClaw 在 workspace 目录跑
        env=env,
        capture_output=True,                   # 捕获 stdout+stderr
        text=True,                             # 解码为 str
        check=False,                           # 不抛异常
        timeout=self.timeout + 30,             # 30s grace
    )
    return result
```

**`session_id` 跨轮记忆**:`AgentEvolveServer._agent_session_id = f"evolve-{config.group_id}"`——同一 group 跨多次 cycle 用**相同**的 OpenClaw session_id(`run_once` 里 `session_id = self._agent_session_id if not self.config.fresh else None`:`fresh=True` 给 None 走 UUID,`fresh=False` 复用 stable id),OpenClaw 内部能维持 conversation history(结合 `--no-fresh` 模式从 `openclaw_home/MEMORY.md` 读上轮 context)。

**`subprocess.run` 与 `asyncio.to_thread` 的关系**:`OpenClawRunner.run` 是**同步** `def`(不是 `async def`);但**调用方** `AgentEvolveServer.run_once` 用 `await asyncio.to_thread(self._runner.run, ...)` 包装——所以 event loop **不**被同步阻塞。`subprocess.run` 本身带 `capture_output=True` + `text=True` 捕获 OpenClaw 输出,但**没有**把 stdout/stderr 持久化到 `evolve_history.jsonl` 或独立 log——只在 `agent_returncode != 0` 时 `logger.warning` 截前 500 字符 stderr(详见 §4.6)。**捕到了但没存**——这是设计选择 [待核-R4](是否要加 `agent_stderr` 字段)。

**`--local`**:OpenClaw 本地模式(不暴露 gateway / 不用远程 agent)。

**`timeout=self.timeout+30`**:subprocess 自己 30s grace(如果 OpenClaw 卡住,先 `TimeoutExpired` 30s 后杀)。**TimeoutExpired 处理**:`try/except` 捕到后返回 `subprocess.CompletedProcess(returncode=-1, stderr=f"TimeoutExpired: ...")`——cycle summary 里 `agent_returncode=-1`。

**`cwd=str(workspace_path)`**:OpenClaw 在 workspace 目录跑——保证所有相对路径(`skills/<name>/SKILL.md` 等)正确。

**`return type` 是 `subprocess.CompletedProcess[str]`**:`text=True` 让 stdout/stderr 是 str 而不是 bytes。`agent.py` 的 summary 只用 `result.returncode`,不用 `result.stdout` / `result.stderr`——这两个值是"看完即弃"的(只有非零 returncode 时截 stderr 写 logger.warning)。

## 4. Step 9 · Upload 与 Registry 更新(`_upload_skill` + SkillIDRegistry)

> **本节是 §1 顶层 10 步里的 Step 9 + Step 10(部分)+ 可观测层**。从原 §3(OpenClawRunner)抽出独立成章——Step 9 跟 OpenClaw 子进程无关(子进程只是改了 workspace 文件,真正上传是 SkillClaw 自己做)。

每个 change 独立跑 "上传 bundle + 写 manifest + 写 registry" 三步,中间失败跳到下一个 change(不打断 cycle);最后所有 change 跑完才 `save_manifest`(全量重写一次)。

### 4.1 `_upload_skill(skill, bundle_files, action)` 流程

对应 Step 9 的核心函数(`agent.py:_upload_skill`):

```
1. 拿 skill.name,空名 return
2. skill_id = self._id_registry.get_or_create(name)         # 分配/复用 ID
3. 若 bundle_files 没 SKILL.md,补一个(用 build_skill_md(skill))
4. 上传 SKILL.md       → <bucket>/<group_id>/skills/<name>/SKILL.md
5. 上传其它文件        → <bucket>/<group_id>/skills/<name>/files/<rel_path>
   + 删 stale files   → list_object_keys + diff keep_keys
6. 算 content_sha(tree_sha) + bundle_record(format/entrypoint/tree_sha/files)
7. version = self._id_registry.record_update(name, content_sha, action, bundle_record)
8. save_version_bundle(...)                                  # 写 v<N>/ 目录
9. manifest = self._load_remote_skills()                     # 重新读远程 manifest
10. manifest[name] = {name, skill_id, version, sha256, tree_sha256,
                       format, entrypoint, files, uploaded_by, uploaded_at,
                       description, category}
11. save_manifest(self._bucket, self._prefix, manifest)      # 全量重写 manifest.jsonl
```

### 4.2 `save_manifest` / 整文件覆盖

`<bucket>/<group_id>/manifest.jsonl` 的写盘策略(见 [33 · 共享层与存储适配](33-共享层与存储适配.md) §3):

- **写**:每行一条 skill 记录,`save_manifest` **全量重写** 整个文件(`"\n".join(lines) + "\n"`),不是行级 append。
- **读**:`load_manifest` 按行解析,`{skill_name: record}` 字典,后写覆盖前写(因为重名时 dict 赋值会覆盖)。
- **并发**:`bucket.put_object` 自身在 OSS/S3 后端是覆盖语义,**没有** atomic rename;并发场景(多 agent 引擎实例同时跑)可能读到半写状态。当前设计假设**单实例**——[待核-R4](多实例时是否需要 OSS 的 ETag / If-Match 串行化)。
- **Nacos 路径**:`save_manifest` **不**调,改走 `nacos_skill_client.upload_skill_zip` + `submit()` + 视 `nacos_publish_mode` 决定 `publish()`(见 20 章 Nacos 适配器)。

### 4.3 `SkillIDRegistry` 细节

**类位置**:`evolve_server/core/skill_registry.py`(`SkillIDRegistry`)。agent 引擎在 `__init__` 里 `self._id_registry = SkillIDRegistry()` + `self._id_registry.load_from_oss(self._bucket, self._prefix)`(从 `<bucket>/<group_id>/evolve_skill_registry.json` 读已有 mapping)。

**`skill_id` 分配规则**:
```python
def get_or_create(self, skill_name: str) -> str:
    entry = self._map.get(skill_name)
    if entry:
        return entry["skill_id"]
    sid = hashlib.sha256(skill_name.encode()).hexdigest()[:12]   # 12-char hex
    ...
    return sid
```
- **算法**:`sha256(name)[:12]`(前 12 个 hex 字符),**确定性**——同一个 name 永远映射到同一个 id;**全局唯一**(sha256 碰撞概率忽略不计);**可重算**(不需要集中协调)。
- **跨 group 共享**:**全 cluster 共享一份**——`evolve_skill_registry.json` 存在 `<bucket>/<group_id>/` 下,**每个 group 一份**,但**ID 分配函数对所有 group 一样**。所以**两个 group 各自有独立的 SkillIDRegistry 实例**(load 自己的 `evolve_skill_registry.json`),但同一 name 算出的 id 在两边是一样的。**"跨 group 共享 ID 空间"**指的是 id 算法一致,不是 registry 实例共享——[待核-R4](是否需要跨 group registry 合并)。

**何时分配 ID**:`get_or_create(name)` 在每次 `record_update` 之前调一次(确保 entry 存在)。**首次 `record_update` 之前** `get_or_create` 就会创建 entry(`version=0, content_sha=""`),所以**整个 ID 在第一次 upload 之前就已经确定**——`record_update` 只把 `version+1`。

**`record_update(name, content_sha, action, bundle_record)` 行为**:
- `version += 1`,`content_sha` 更新;
- `history` 数组追加 `{version, content_sha, timestamp, action}`(**最多 20 条**,超过截尾);
- 返回新 version 号。

**`save_to_oss` / `load_from_oss`**:整个 dict 一次性 `put_object` 写到 `<bucket>/<group_id>/evolve_skill_registry.json`;**失败只 logger.warning 不抛**——所以 registry 写挂了 cycle summary 仍正常,只是下次启动 registry 缺数据(会从 0 version 重新建 entry,version 号会乱——见 §8.2)。

### 4.4 失败重试 / 部分成功

- **`put_object` 失败**:`agent.py` 的 Step 9 循环里 `try/except Exception as e: logger.error(...)`——**不**重试,跳到下一个 change;`skills_evolved` 计数跳过这个失败的。**部分成功**:`save_manifest` 在所有 change 跑完才写(成功 + 失败的子集都反映进去——失败的 change 没进 manifest)。
- **`delete_session_keys` 失败**:`oss_helpers.py:delete_session_keys` 内部 `try/except` 单独处理每个 key,返回成功删除数。**不抛**——失败的 key 留在 storage,下次 drain 会再读进来(可能重复处理)。**audit**:`delete_session_keys` **不**写 `evolve_history.jsonl`(audit 留[待核-R4])。
- **`_notify_proxy_reload` 失败**:**共享工具**,workflow 引擎也调用(31 章 §10 步骤 11)。agent 引擎在 `create_http_app` / `run_periodic` 里**不直接调** `_notify_proxy_reload(详见 31 章 10 节)`——只有 `uploaded_skills > 0` 才由 `EvolveEngineMixin` 内的对应逻辑触发(agent 引擎目前**没**实现 `uploaded_skills > 0 → _notify_proxy_reload` 这一步,与 workflow 不同——见 §8.1 设计限制)。
- **`save_to_oss` 失败**(registry):只 logger.warning 不抛。**不**触发 `had_processing_error`(agent summary 没有这个字段)。

### 4.5 三个清单文件的关系(workspace `manifest.json` / storage `manifest.jsonl` / `evolve_skill_registry.json`)

| 文件 | 路径 | 谁写 | 谁读 | 写盘时机 | 格式 |
|------|------|------|------|----------|------|
| workspace `manifest.json` | `<workspace_root>/manifest.json` | `AgentWorkspace.prepare`(Step 5) | OpenClaw agent(只读) | 每个 cycle 覆盖 | 单个 JSON 对象 `{name: record}` |
| storage `manifest.jsonl` | `<bucket>/<group_id>/manifest.jsonl` | `_upload_skill` Step 11(`save_manifest`) | 客户端 `SkillHub.sync`(见 20 章) | 每个 change 上传后**全量重写** | 每行一个 JSON record(`\n` 分隔) |
| storage `evolve_skill_registry.json` | `<bucket>/<group_id>/evolve_skill_registry.json` | `SkillIDRegistry.save_to_oss`(Step 10) | `SkillIDRegistry.load_from_oss`(engine 启动) | Step 10 收尾一次 | 单个 JSON 对象 `{name: {skill_id, version, content_sha, history:[...20], ...}}` |

**关键差异**:
- `workspace manifest.json` 名字是 `.json`(不是 `.jsonl`),格式是**单个 JSON 对象**;`storage manifest.jsonl` 名字带 `.jsonl` 是历史习惯(命名早于实现,实际也是单 dict 写到单文件,客户端按行解析时只有一行)——见 [20 · 共享存储与同步](../02-客户端共享层/20-共享存储与同步.md) §3。
- workspace 这份是 SkillClaw 在 Step 5 喂给 OpenClaw 的"只读视图",OpenClaw 看完就丢;storage 那份是**真正的"当前 skill 集合"**——Step 9 上传时,先把 workspace manifest 拉到 storage,然后重写。
- `evolve_skill_registry.json` 是 **ID + version + history** 的元数据,**不**在 `manifest.jsonl` 里(manifest 只记"当前 skill 长什么样",registry 记"这个 skill 的版本演进")。客户端**不直接读** registry(对客户端透明)——20 章的"客户端只需要知道 `manifest.jsonl` + `skills/<name>/SKILL.md` 两个文件"就是这意思。

### 4.6 可观测层 `evolve_history.jsonl`

**路径**:`config.history_path`(env key `EVOLVE_HISTORY_LOG`,默认 `evolve_history.jsonl`,**相对路径,相对 evolve server 启动时的 cwd**)。改路径用 `EVOLVE_HISTORY_LOG=/var/log/skillclaw/history.jsonl`。

**写盘**:`EvolveEngineMixin._append_history(summary)`(见 30 章 §5)——open + write + close,**单行 JSON,无 rotation**。**IO 失败只 logger.warning 不抛**(与 31 章 共享)。

**每行 schema**(agent 引擎 summary,见 `agent.py:run_once` 末尾的 `summary = {...}`):

```json
{
  "timestamp": "2026-04-20T15:00:00.123456+00:00",   // ISO8601 UTC
  "elapsed_seconds": 247.3,                            // 单 cycle 耗时
  "sessions": 3,                                       // 本轮 drain 的 session 数
  "skills_evolved": 2,                                 // 本轮成功上传的 skill 数
  "agent_returncode": 0,                               // OpenClaw 子进程 exit code;TimeoutExpired=-1
  "evolutions": [                                      // 每个成功上传的 change 一条
    {
      "action": "create" | "improve",
      "skill_name": "<name>",
      "skill_id": "<12-char hex>",
      "version": 5,
      "source": "agent"
    }
  ]
}
```

**与 workflow summary 的差异**(见 31 章 §10):agent summary **没有** `had_processing_error` / `skill_groups` / `validation_publish` / `session_judge` / `skill_verifier` 字段——agent 引擎的 10 步里没有 session_judge / skill_verifier / validation publish,所以 summary 也不带这些。

**与 `evidence.md` 的区别**:`history/v<N>_evidence.md` 是 OpenClaw 写在 workspace 里(`<workspace>/skills/<name>/history/`),记录 self-validation 结果;`evolve_history.jsonl` 是 SkillClaw 写在 evolve server cwd 里,记录每个 cycle 的宏观 summary——**两个文件不同,关注点不同**。

**生产监控**:41 章部署形态建议"定期 `cat evolve_history.jsonl` + `curl /status`"——配合 filebeat 拉走 + 磁盘告警(因为无 rotation,1 年会变 GB 级 [待核-R4])。

## 5. `EVOLVE_AGENTS.md` 协议(agent 行为宪法,407 行)

`evolve_server/engines/EVOLVE_AGENTS.md`(407 行)——**agent 引擎的"宪法"**。OpenClaw agent 读 `AGENTS.md`(SkillClaw 写的)被指向这个文件,然后严格按它的流程工作。

> **完整 407 行请直接读 `evolve_server/engines/EVOLVE_AGENTS.md`**——`wc -l engines/EVOLVE_AGENTS.md` 可确认行数。本节**不**重抄全文,只列出最关键的 7 步流程 + 4 条禁区——重写 SkillClaw 的 agent 引擎时,先看本节定位**流程边界**,再看 `EVOLVE_AGENTS.md` 拿**完整 prompt 文本**。

### 5.0 agent 的 7 步主流程(从 `EVOLVE_AGENTS.md` 协议体中提取)

OpenClaw agent 接到 prompt 后按以下 7 步走(对应 10 步里的 Step 4 部分 + Step 7 内部):

```
Step 1 · Read & Understand Session Data
  读 sessions/<session_id>.json 的 _summary(快速概览)+ 必要时 _trajectory(step-by-step 细节)
  ↓
Step 2 · Analyze Patterns
  哪些 skill 被用了? 哪些有效? 哪些失败?
  哪些 skill 被创建了但没用?
  ↓
Step 3 · Decide Action
  对每个 skill / pattern 决策:
    SKIP / IMPROVE / OPTIMIZE_DESCRIPTION / CREATE
  ↓
Step 4 · Read Existing Skill State
  读 skills/<name>/SKILL.md(当前内容)
  + 读 skills/<name>/history/v<N>_evidence.md(自检证据,必有)
  + 可能读 history/v<N>.md(决策记录,可选)
  ↓
Step 5 · Edit Skill
  写 skills/<name>/SKILL.md
  (可改: body / description / frontmatter)
  (不可改: name —— 改名 = 创新 skill)
  ↓
Step 6 · Self-Validate
  检查 SKILL.md 是否符合 AgentSkills 协议
  (frontmatter / name / description / category)
  跑一次 LLM 验证(可选)评估改进质量
  失败 → revert 或继续改
  ↓
Step 7 · Record
  写 skills/<name>/history/v<N>_evidence.md
  (注:workspace prepare() 用 clean=True 重写,history/ 实际不保留;
   OpenClaw 跨轮记忆的真正载体是 openclaw_home/MEMORY.md,见术语速查"两个 HOME")
```

### 5.1 4 条禁区(对 agent 的硬约束)

1. **所有 file 操作必须留在 workspace 内**(OpenClaw sandbox 关闭、但**靠 EVOLVE_AGENTS.md 协议约束**)——禁止 `os.chdir("..")` / 写 `/tmp` / 读 `/etc/...`。
2. **不修改** `sessions/` / `manifest.json` / `skill_registry.json`(只读)——这三份是 SkillClaw 喂的输入,agent 改了就破坏 `collect_changes` 的 diff 机制(SkillClaw 在 §4.1 第 2 条约束里也明确点了**同一组文件**——和 EVOLVE_AGENTS.md 是同一约束的两个引用)。
3. **只在 `skills/<name>/` 写变化**——其它目录(workspace 根、AGENTS.md、SESSION.md)都是只读上下文。
4. **每次改完必须 self-validate**——失败就 revert 或继续改,不能留 known-failing skill 在 `skills/`;自检结果写到 `<skill>/history/v<N>_evidence.md`。

### 5.2 自检失败的处理

`EVOLVE_AGENTS.md` 写明:

> Before finalizing any changed skill, complete the self-validation required by EVOLVE_AGENTS.md; if validation fails, keep editing or revert the change rather than leaving a known-failing skill in `skills/`.
> Record self-validation results in the paired `history/v<N>_evidence.md` file.

**这是 OpenClaw 自己的 LLM 决定**——SkillClaw 不介入。失败的 skill 仍在 `skills/`(被 detect 为 changed),但 `history/v<N>_evidence.md` 会记录失败原因。**revert 机制**:`EVOLVE_AGENTS.md` 假设 OpenClaw 的 `edit` 工具记录了 edit history(`<skill>/.edits/`,OpenClaw 内部机制,SkillClaw 不知道细节),agent 调 `edit_revert` 回退到上一次成功状态;**不**依赖 git。

## 6. `_summarize_sessions`(Step 2,agent 引擎专属入口)

`agent.py:AgentEvolveServer._summarize_sessions(sessions)`(180 行起)——给 session 加 `_trajectory` / `_summary` / `aggregate` 等 compact 字段,然后写到 workspace 的 `sessions/<sid>.json`。

### 6.1 LLM adapter 选型(由 `llm_api_type` 决定)

| `llm_api_type` 值 | 调谁 | 说明 |
|-------------------|------|------|
| `""`(空) / `openai-completions` / `openai-responses` / `ollama` | `evolve_server.core.llm_client.AsyncLLMClient` | OpenAI 兼容,evolve_server 自家 |
| `anthropic-messages` | `agent.py:_AnthropicMessagesLLMClient` | **agent 引擎专属**,直连 Anthropic `/v1/messages` |
| `google-generative-ai` / 其它 / 未知 | 退化路径 | **不**调 LLM,只填 `_trajectory` + 空 `_summary`;agent 只看 trajectory,不看 summary |

完整 `llm_api_type` 取值表见 [30 · Evolve 服务端与引擎选择](30-Evolve服务端与引擎选择.md) §3.1(`EVOLVE_LLM_API_TYPE` env 表)。

**workflow 引擎调 Anthropic 走 `bedrock_client`(`skillclaw/bedrock_client.py`);agent 引擎调 Anthropic 走 `_AnthropicMessagesLLMClient` 直连 `/v1/messages`**——两条独立路径,**不**共用一个 client。

### 6.2 `_AnthropicMessagesLLMClient` 三个细节(agent 引擎专属)

**文件**:`_AnthropicMessagesLLMClient` 单独定义在 `agent.py` 内,**agent 引擎专属**(与 workflow 引擎无关)——workflow 引擎调 Anthropic 走 `bedrock_client`(详见 30 章 + 13 章)。

**协议**:`POST <llm_base_url>/v1/messages`(Anthropic Messages API,**不**走 boto3 Bedrock)。`llm_base_url` 默认 `https://api.openai.com/v1` 但 `_messages_url()` 会智能补 `/v1/messages` 后缀。

**`system` 字段处理**(代码骨架):
```python
system_parts: list[str] = []
body_messages: list[dict[str, str]] = []
for message in messages:
    role = str(message.get("role") or "")
    content = str(message.get("content") or "")
    if role == "system":
        if content: system_parts.append(content)
        continue
    if role in {"user", "assistant"}:
        body_messages.append({"role": role, "content": content})

request_body = {
    "model": self.model,
    "messages": body_messages or [{"role": "user", "content": ""}],
    "max_tokens": kwargs.pop("max_tokens", self.max_tokens),
    "temperature": kwargs.pop("temperature", self.temperature),
}
if system_parts:
    request_body["system"] = "\n\n".join(system_parts)   # system 提到顶层
```
- `messages` 里 `role='system'` 的项被**提取合并**到 `body['system']` 字符串(`"\n\n"` join);
- `role in {'user', 'assistant'}` 的项保留在 `body['messages']`;
- 其它 role 丢弃;
- 如果 `body_messages` 空(全是 system),fallback `[{role:'user', content:''}]`(Anthropic API 要求 messages 非空)。

**重试与 backoff**(代码骨架):
```python
max_retries = 6
async with httpx.AsyncClient(timeout=httpx.Timeout(600.0, connect=30.0)) as client:
    for attempt in range(max_retries):           # attempt ∈ {0,1,2,3,4,5}
        try:
            resp = await client.post(self._messages_url(), json=request_body, headers=headers)
            resp.raise_for_status()
            payload = resp.json()
            # 从 payload.content 提取所有 type=='text' 的 text 字段拼接返回
            return "".join(parts)
        except Exception:
            if attempt < max_retries - 1:
                wait = min(2**attempt + random.uniform(0, 1), 30)   # 限到 30s
                await asyncio.sleep(wait)
                continue
            raise
```
- **共 6 次**(含首试),`attempt ∈ {0, 1, 2, 3, 4, 5}`;
- **最大等待**:`min(2**attempt + random.uniform(0, 1), 30)`——`2**5 = 32`,被 30s 截掉,实际最大等 30s;
- 失败抛异常(`raise` 最后一轮的 exception),由 `summarize_sessions_parallel` 处理。

**不支持**:`vision` / `tool_use` / `cache_control` / `thinking`——只用 `messages: [{role, content}]` + `system` 字符串。**待核-R4**(是否需要 vision 给 agent 看图)。

**默认参数**:`model="gpt-5.4"`(注意 `_DEFAULT_AGENT_EVOLVE_MODEL` 是 `"gpt-5.4"`,不是标准 `"gpt-4o"`)、`max_tokens=100000`、`temperature=0.4`、`base_url="https://api.anthropic.com"`。

## 7. 完整流程示例

### 7.1 启动命令

```bash
# 启动(agent 模式,跨轮记忆)
skillclaw-evolve-server --engine agent --no-fresh --interval 600 --port 8787
#                            ^^^^^^^^^^^^^^ ^^^^^^^^^  fresh 默认 True
```

### 7.2 第一次 cycle(`fresh=True` 模式)

把 §1 顶层 10 步映射到实际子操作:

| §1 步 | 实际子操作 | 见 |
|------|------------|---|
| 1 Drain | `list_session_keys` 拉 3 个 session → `read_json_object` 读 JSON | 31 章 §2 |
| 2 Summarize | `_summarize_sessions` 调 LLM(假设 `llm_api_type="openai-completions"`,走 `AsyncLLMClient`)给每个 session 加 `_trajectory` / `_summary` | §6 |
| 3 Reset | `if fresh: self._workspace.reset()` → `shutil.rmtree(workspace_root)` + mkdir | §2 / §3.2 |
| 4 Fetch | `_load_remote_skills()` 读 `<bucket>/<group_id>/manifest.jsonl`;`_fetch_all_skills` 拉 5 个 skill 的 bundle bytes | §4.5 |
| 5 Prepare | `AgentWorkspace.prepare()`:写 3 个 session JSONs + 5 个 skills + manifest.json + skill_registry.json + EVOLVE_AGENTS.md + 4 个 bootstrap | §2.2 |
| 6 Snapshot | `snapshot_skills()` 算 5 个 tree_sha | §2.3 |
| 7 Run | `await asyncio.to_thread(self._runner.run, workspace, message, session_id=None)`(`fresh=True` 走 UUID);`OpenClawRunner._prepare_home` wipe openclaw_home;`subprocess.run(['openclaw', 'agent', '--session-id', 'evolve-a1b2c3d4e5f6', ...])` 跑 10 分钟;returncode=0 | §3.5 |
| 8 Collect | `collect_changes(before)` → 7 个 after_sha,2 个 diff(1 create + 1 improve) | §2.4 |
| 9 Upload | 对每个 change 调 `_upload_skill`:写 SKILL.md + files + 删 stale + record_update + save_version_bundle + 拉新 manifest + save_manifest 全量重写 | §4.1 |
| 10 Finalize | `save_to_oss` 写 registry;`delete_session_keys` 删 3 个 session key;`cleanup_sessions` 删 workspace sessions/;`_append_history` 写 1 行到 `evolve_history.jsonl` | §4.3 + §4.4 + §2.5 + §4.6 |

### 7.3 第二次 cycle(`--no-fresh` 模式)

`session_id = self._agent_session_id`(`f"evolve-{config.group_id}"`,稳定不变):

1. drain 1 个 session
2. summarize(同上)
3. **不** reset openclaw_home(`fresh=False`,`_prepare_home` 走 `mkdir(exist_ok=True)` 保留上轮 `MEMORY.md` / `memory/` / `agents/main/sessions/sessions.json`);`workspace_root` 每次 prepare 都重写(与 `--fresh` 无关)
4. fetch manifest(5 旧 + 1 上轮新 = 6 个 skill)
5. **不**保留 `skills/<name>/history/`(因 prepare 用 `clean=True`,history/ 被删;跨轮记忆靠 `openclaw_home/MEMORY.md` 不靠 workspace history/)
6. 写 1 个新 session
7. snapshot
8. 启 OpenClaw(**同一个** session_id=`evolve-my-group`)→ OpenClaw 记得上一轮 context(从 `openclaw_home/MEMORY.md` + `agents/main/sessions/sessions.json` 读)
9. 跑完

### 7.4 经验值

- 10 条 session + 3 个 skill 的典型 group:一次 run 大约 3-8 分钟(`agent_returncode=0`,`elapsed_seconds=180-480`)
- 50 条 session + 10 个 skill:接近 `agent_timeout` 上限(默认 600s)— `TimeoutExpired` 概率升高,见 §8.1
- 实际部署推荐 `interval_seconds=300` (5 分钟)或 `600` (10 分钟),见 41 章 agent 引擎部署段

## 8. 已知限制 / 错误路径 / 待核(分 3 类)

> 上一版 §7 把 3 类内容混在一起。R4 拆成 3 个独立子节——**已知设计限制**(SkillClaw 当前实现就这样的,不是 bug) / **错误路径与边界**(生产 incident 时会发生的事) / **待核/待对照官验证项**(还没验证的设计选择)。

### 8.1 已知设计限制

- **`openclaw` 必须装且能调 LLM**:`AgentEvolveServer` 构造时**不**校验 OpenClaw 是否能跑通——subprocess 启动失败才报错(`FileNotFoundError: 'openclaw'`,整个 cycle 抛异常,`run_periodic` 抓 Exception 走下一轮)。**这是设计选择**——减少启动时间 / openclaw 升级可能 break CLI flag,出错让 subprocess 失败自然浮上来。
- **`_AnthropicMessagesLLMClient` 不支持 vision / tool_use / cache_control / thinking**:只用 `messages: [{role, content}]` + `system` 字符串。**待确认**对 vision 任务(agent 看图片)够不够用。
- **`AnthropicClient` 不持久化连接**:每次 `_summarize_sessions` 都新建 `httpx.AsyncClient`(在 `async with` 里),不连接池——性能调优点。
- **`--no-fresh` + OpenClaw memory 累积**:OpenClaw 的 `openclaw_home/MEMORY.md` 会**无界**增长(OpenClaw 内部 memory-core 在管,SkillClaw 不介入)。**待确认** OpenClaw 自己有没有 rotation。
- **`_id_registry` 跨 group 共享 ID 空间**:**id 算法一致**(`sha256(name)[:12]`),**registry 实例独立**(每 group 自己的 `evolve_skill_registry.json`)。同一 name 在两个 group 算出同一 id,但 registry 不合并——**待核-R4** 是否需要跨 group registry 合并。
- **`_notify_proxy_reload` agent 引擎不调**:workflow 引擎 Step 10.5 会 `if uploaded_skills > 0: _notify_proxy_reload()`(31 章 §10 步 11);agent 引擎**没**实现这一步——**待核-R4** 是设计遗漏还是有意。
- **agent 引擎不实现 Nacos 路径**:`_load_remote_skills` 在 workflow 里有 Nacos override,`AgentEvolveServer` **没** override——`skill_storage_backend="nacos"` 走 agent 会失败(底层用 default `_load_remote_skills` 拉 `manifest.jsonl`,Nacos 后端没有这个文件)。30 章 §7 第 5 条已标待确认。

### 8.2 错误路径与边界

- **agent 改完但 LLM `_summarize_sessions` 调用失败**(网络挂 / LLM 5xx):`summarize_sessions_parallel` 内部会失败,session 写不进 workspace → `prepare` 时 compact 字段空(`_summary=""` / `_trajectory=""`)→ agent 看不到 summary。**部分成功**:`agent.py` 的 try/except 包了 Step 5 之后的所有步骤,但 Step 2 `_summarize_sessions` 失败会让 `agent.py:run_once` 抛异常,Step 3-10 **不**跑。
- **agent 改了 skill X,server 端 sha 已被别人改过**(并发场景):`collect_changes` **不**调 `_detect_conflict`,Step 9 直接覆盖——**last-write-wins**,agent 改的可能丢掉别人的改动。
- **未知 `llm_api_type` fallback**:退化路径只填 `_trajectory`(程序生成的),`_summary=""`——agent 看不到 LLM 摘要,只能看 trajectory。
- **`save_to_oss`(registry)失败**:`logger.warning` 不抛,**summary 不带 had_processing_error**(agent summary 没这个字段)——registry 写挂了**静默**;下次启动 registry 从 0 version 重建,version 号会乱(从 0 而不是上次 version+1 开始)。生产建议 filebeat 拉 logger。
- **`delete_session_keys` 失败**:失败的 key 留在 storage,下次 drain 会再读进来——**可能重复处理**(agent 跑两遍,产生 2 个 evolution record)。**audit** 不写 `evolve_history.jsonl`。
- **`subprocess.run` `TimeoutExpired`**:OpenClaw 卡死超过 `timeout+30s` → `result.returncode=-1` + `result.stderr="TimeoutExpired: ..."` → `agent_returncode=-1` 写 `evolve_history.jsonl`,但 Step 8 collect_changes 仍跑(因为 `cwd` 已设好,workspace 文件是上一轮 step 7 中间状态)——**可能**正常 diff,**可能** diff 到半成品(取决于 OpenClaw 卡死在哪个文件写完之后)。
- **`run_periodic` Ctrl-C 退出要等 `interval_seconds`**:循环里 `await asyncio.sleep(self.config.interval_seconds)`,signal handler 设了 `self._running = False` 但 sleep 不能被打断——最坏等 600s。生产建议 `TimeoutStopSec=5s` 的 systemd unit 或起独立 stop 协程(31 章 §11 第 8 条同问题)。

### 8.3 待核 / 待对照官验证项

- [待核-R4] **`openclaw_home` 默认值**:`evolve_server/core/config.py:_PACKAGE_DIR = Path(__file__).resolve().parent` 解析到 `evolve_server/core/`(不是 `evolve_server/`),所以 `__post_init__` 写 `self.openclaw_home = str(_PACKAGE_DIR / ".openclaw_home")` 实际落点 `evolve_server/core/.openclaw_home`——变量名是 `_PACKAGE_DIR` 但解析到 `core/` 子目录,与命名暗示不符;30 章字段表里只写"`<_PACKAGE_DIR>/.openclaw_home`"不展开。**待核**是不是该 `Path(__file__).resolve().parent.parent`。
- [待核-R4] **`workspace_root` 默认值**:同上,落点 `evolve_server/core/agent_workspace`;41 章 §9.2 写"`<pkg>/agent_workspace`"是否指 `evolve_server/` 还是 `evolve_server/core/`。
- [待核-R4] **`--local` CLI flag vs `gateway.mode=local` JSON**:两者等价但 CLI 优先级更高——**待核** OpenClaw 是不是真的 CLI 优先。
- [待核-R4] **`subprocess.run` 的 stdout/stderr 持久化**:当前只在 `agent_returncode != 0` 时 `logger.warning` 截前 500 字符 stderr;成功的 stdout 拿到但丢弃。**待核**是否要加 `agent_stderr` 字段到 `evolve_history.jsonl`,或写独立 `<workspace>/agent_stderr.log`。
- [待核-R4] **`<workspace>/skills/<name>/history/` 不保留**:`prepare` 用 `clean=True` 重写,history/ 删——**待核**是不是该把 history/ 加到 preserve 列表里(配合 `--no-fresh` 模式)。
- [待核-R4] **`evolve_history.jsonl` 无界增长**:`_append_history` 只 append 不 rotate,生产 1 年会变 GB 级(30 章 §7 第 3 条 + 31 章 §11 第 6 条同问题)。
- [待核-R4] **`save_manifest` 并发安全**:多 agent 引擎实例并发时可能读到半写状态(无 file lock / 无 ETag)——目前设计假设单实例。
- [待核-R4] **`_id_registry` 跨 group 合并**:两 group 各自 registry,但 id 算法一致;**待核**是否要 `global registry` 跨 group 共享。
- [待核-R4] **`AnthropicClient` vision 支持**:`_AnthropicMessagesLLMClient` 不处理 `image` content block;**待核** agent 引擎未来是否需要看图。
- [待核-R4] **`memory-core` 定义**:`_EVOLVE_AGENTS_MD` 模板里写"You may use `memory/` and `MEMORY.md` for long-term notes across rounds",`memory-core` 是 OpenClaw 自己的模块(不在 SkillClaw 仓库);**待核** memory-core 是不是 OpenClaw 的包名,还是 SkillClaw 抽象。
- [待核-R4] **`HF_HUB_OFFLINE=1` + `TRANSFORMERS_OFFLINE=1` 是不是够**:OpenClaw 内部如果用别的下载器(比如 modelscope / 自家 hub),这俩 env 不挡——**待核** OpenClaw 用了哪些下载器。
- [待核-R4] **测试覆盖**:目前无单元测试覆盖 agent 引擎(强依赖 OpenClaw 子进程);建议在沙盒里跑 `--interval 60` 24 小时,人工 review `evolve_history.jsonl`(R1 起就标,本轮仍无测试)。
- [待核-R4] **`/status` 端点**:`create_http_app` 提供 `/trigger` / `/status` / `/health`,`/status` 返回 `{running, pending_sessions, registered_skills, skills, fresh_mode}`——**待核** `/status` 在多 engine instance 下的语义(`self._running` 是单实例标志位)。

---

→ **下一篇**:[33-共享层与存储适配](33-共享层与存储适配.md) — object store 抽象 + evolve server 怎么读写 storage
