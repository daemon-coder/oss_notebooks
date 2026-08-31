# 32 · Agent 引擎详解

## 这一章讲什么

这一章回答"agent 引擎在 workflow 引擎之外,怎么把'让 LLM 自己改 SKILL.md'这件事做出来"——覆盖 OpenClaw 子进程调度、`AgentWorkspace` 文件级 diff、`EVOLVE_AGENTS.md` 协议、跨轮记忆机制、SkillIDRegistry 细节、可观测层。

## 术语速查（读本章前 1 分钟过一遍）

> 本章新引入 7 个核心术语(下表)。"两个 HOME"(`workspace_root` / `openclaw_home`)的概念因反复出现且容易混淆,独立成 [§1.1 两个 HOME 的区分](#11-两个-home-的区分) 作为正式内容,不在本速查表内。

- **OpenClaw** —— 一个**外部的 agent 运行时**(AMAP-ML 同团队开源,仓库 `https://github.com/AMAP-ML/openclaw`;**不是** SkillClaw 内部子模块;`--llm-api-type=anthropic-messages` 时调 `openclaw` 自带的 chat 客户端,workflow 引擎调 Anthropic 走 `bedrock_client`,agent 引擎和 workflow 引擎是两条独立路径)。SkillClaw **用 `subprocess.run` 调它的 binary**(`openclaw agent --session-id ... --message ... --json --local --timeout ...`),把"读 sessions + 改 SKILL.md"这件事外包给它。**这跟 17 章 12 种 CLI agent 适配是两类不同的事**——那些是 SkillClaw **替**它们配(`configure_openclaw` 等),这个是 SkillClaw **反过来调**它。(OpenClaw 兼容性详见 [02 章 §4](../00-总览/02-核心概念词典.md))
- **EVOLVE_AGENTS.md** —— SkillClaw 喂给 OpenClaw 的**完整指令文件**(`evolve_server/engines/EVOLVE_AGENTS.md`,407 行;`wc -l engines/EVOLVE_AGENTS.md` 可核对),告诉 OpenClaw "你的任务是改 SKILL.md;请按这个流程走"。**要改 agent 行为,改这份 markdown,不用动 Python**。§5 给出 7 步流程概览,完整 prompt 文本以原文件为准。
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

**场景**:周期任务或 HTTP 触发 `run_once`——与 workflow 引擎不同,**agent 引擎是"让 LLM 自己改 SKILL.md"**。它把"提炼 session → 改 skill"这件事**外包给一个 OpenClaw 子进程**——给子进程一个"工作区"目录(装 sessions + skills + EVOLVE_AGENTS.md 协议),子进程自己读、改、产出新 skill 文件;evolve server 只负责"装 / 收 / 上传"。

**为什么是 10 步而不是更少**:

- 步骤 1-2 与 workflow 引擎一样(drain + summarize)——都是"准备数据"
- 步骤 3-6 是 agent 引擎特有——**准备给子进程的工作区**(`workspace` 目录:复制 manifest / 写 EVOLVE_AGENTS.md / snapshot 当前 skill 集)
- 步骤 7-8 是 agent 引擎核心——**让 OpenClaw 子进程跑**(异步阻塞),**收集它改了什么**(`collect_changes` 对比 snapshot)
- 步骤 9-10 是收尾——**逐个上传** + **写 manifest / 清 sessions / 追加 history**

**整体流程图**:

```
run_once()
  │
  ├─→ 1. Drain (拉 sessions/*.json)
  ├─→ 2. Summarize (LLM 给 session 加 _trajectory / _summary)
  ├─→ 3. Reset(可选) if self.config.fresh: workspace.reset()
  ├─→ 4. Fetch all skills (load manifest + fetch 所有 skills 落到 workspace)
  ├─→ 5. Prepare workspace (写 sessions + skills + manifest + EVOLVE_AGENTS.md)
  ├─→ 6. Snapshot (记下当前 workspace 里所有 skill 文件的 sha256)
  ├─→ 7. Run agent (asyncio.to_thread 调 OpenClawRunner.run, 阻塞等)
  │       # OpenClaw 读 EVOLVE_AGENTS.md, 自己改 SKILL.md
  ├─→ 8. Collect changes (对比 snapshot 与现状, 产出 changed_skills 列表)
  ├─→ 9. Upload (逐个 change 跑 _upload_skill → record_update)
  │       # 失败跳下一个 change, 不打断 cycle
  └─→ 10. Finalize (save_to_oss + delete_session_keys + cleanup_sessions + _append_history)

return summary_dict
```

**10 步对照表**(与 [30 · 4.2 agent](30-Evolve服务端与引擎选择.md) 一致):

| # | 步 | 在源码里叫什么 | 在哪一节讲 | 这一步的"输入"和"产出" |
|---|----|---------------|----------|-------------------|
| 1 | Drain | `self._drain_sessions()` | 见 31 章 §2 | 产出 `sessions` list |
| 2 | Summarize | `self._summarize_sessions(sessions)` | §6 | 给每个 session 加 _trajectory / _summary |
| 3 | Reset(可选) | `if self.config.fresh: self._workspace.reset()` | §2 / §3.2 | workspace 目录清空 |
| 4 | Fetch all skills | `_load_remote_skills()` + `_fetch_all_skills(manifest)` | §2.2 | workspace 里有完整 skill 集 |
| 5 | Prepare workspace | `self._workspace.prepare(...)` | §2.2 | 写 sessions / skills / manifest / EVOLVE_AGENTS.md |
| 6 | Snapshot | `self._workspace.snapshot_skills()` | §2.3 | 记下当前所有 skill 文件的 sha256 |
| 7 | Run agent | `await asyncio.to_thread(self._runner.run, ...)` | §3.5 | OpenClaw 子进程跑完, 它改了哪些文件 |
| 8 | Collect changes | `self._workspace.collect_changes(before_snapshot)` | §2.4 | 产出 `changed_skills: list[SkillChange]` |
| 9 | Upload(逐个) | `self._upload_skill(...)` × N | §4 | 写 `skills/<name>/versions/v<N>/` + manifest |
| 10 | Finalize | `save_to_oss` + `delete_session_keys` + `cleanup_sessions` + `_append_history` | §4.4 / §4.6 / §2.5 | registry 更新 + sessions 删除 + history 加一行 |

**为什么 Step 7 / 8 / 9 / 10 是串行流水线**:

```
OpenClaw 跑完 → 才知道它改了哪些文件 (Step 8)
                ↓
知道改了哪些 → 才能 upload (Step 9)
                ↓
upload 完 → 才能 finalize (Step 10)
```

中间失败的处理:

- **Step 7 失败**(OpenClaw 子进程崩了)→ Step 8 / 9 / 10 仍跑但 `changed_skills=[]` → upload 空集 + finalize(只清理 sessions)
- **Step 9 内部单 change 失败** → 跳到下一个 change, 不打断 cycle(其它 change 仍上传, 失败的进 `evolve_history.jsonl` 的 `error` 字段)
- **Step 10 finalize 失败** → evolve_history.jsonl 记 error, 但已上传的 skill 仍在(下一轮 cycle 可以 reconcile)

**与 workflow 引擎的关键差异**:

- **workflow** 是"evolve server 自己调 LLM 炼 skill"——LLM 调用的 prompt / 输出解析全在 evolve server 里
- **agent** 是"evolve server 把工作区打包丢给 OpenClaw 子进程"——LLM 调用在子进程里,evolve server 只负责"装 / 收 / 上传"

`--engine workflow` 是固定流水线;`--engine agent` 是把"炼 skill"这件事外包给一个能 SKILL.md 协议的 agent framework(目前只支持 OpenClaw)。

**关键边界**:

- **空 sessions**:跳过 Step 2-9,直接进 Step 10(只跑 finalize)
- **`config.fresh=True`**:Step 3 reset workspace,Step 4 fetch 是空(没有 manifest),Step 7 跑的是"从零开始演化"
- **`config.fresh=False`** (默认):Step 3 跳过,Step 4 fetch 当前 manifest,Step 7 跑的是"基于现有 skill 集做增量演化"
- **OpenClaw 进程超时**:`asyncio.to_thread` 等不到结果 → 终止子进程,Step 7 标记 error,Step 8-9 跑空集
- **workspace 大小**:Step 4 把 manifest 里所有 skill 拉到 workspace——如果 skill 集很大(几百个 skill),workspace 目录会很大,Step 7 OpenClaw 启动会慢

**走完之后**:

- `evolve_skill_registry.json` 更新了所有 Step 9 成功上传的 version
- `manifest.jsonl` 多了 N 行(Step 9 的 N 个新 bundle)
- `sessions/*.json` Step 10 删了(避免下轮重复炼)
- `evolve_history.jsonl` 加了一行 cycle 记录
- OpenClaw 自己的 MEMORY.md(在 openclaw_home 里)如果 `--no-fresh` 保留——下轮 OpenClaw 启动时能读
- workspace_root 目录**下次 cycle 会重写**——不保留状态

### 1.1 两个 HOME 的区分

agent 引擎里有**两个 HOME,正交两件事**——`workspace_root`(SkillClaw 喂给 OpenClaw 的工作区)和 `openclaw_home`(OpenClaw 自己的 HOME)。下面把这两件事讲清,免得后续 §3 / §4 反复提到时混淆。

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
                   不是 agent.py 启动时的 cwd;30 章用 _PACKAGE_DIR 是另一条路径)

--fresh=True  → _prepare_home wipe openclaw_home(OpenClaw 自己的 MEMORY.md 等)
              → workspace_root 每次 prepare() 都重写,与 --fresh 无关
--no-fresh    → openclaw_home 保留 → OpenClaw 能读上轮 MEMORY.md
```

**关键决策点**:`--fresh` 只影响 `openclaw_home`,**不**影响 `workspace_root`——后者每次 `prepare()` 都重建。`--no-fresh` + 跨轮 `session_id` 是 OpenClaw 跨轮记忆的两个支撑。

**两个 HOME 路径的微妙差异**:`AgentEvolveServer.__init__` 默认把 `openclaw_home` 设为 `<_PACKAGE_DIR>/.openclaw_home`(与 `workspace_root` 同目录),而 `OpenClawRunner._prepare_home` 在 `openclaw_home` 为空时**用 `Path.cwd() / ".openclaw_home"` 取 OpenClawRunner 构造时的 cwd**——这两个路径在生产部署(从包目录起服务)下通常是同一处,但开发模式下(从仓库根起)会分开。

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

**场景**:`run_once` Step 5 调 `AgentWorkspace.prepare(...)`——给 OpenClaw 子进程准备"工作区"。这个工作区**完全新建**(每次 cycle 重建),OpenClaw 启动时读 `EVOLVE_AGENTS.md` + `sessions/` + `skills/` 然后做决策。

**为什么是 7 步按顺序写而不是 1 步**:

- **sessions/ + skills/ 是 OpenClaw 必读的内容**(见 §2.1)——必须先写好
- **manifest.json + skill_registry.json 是只读视图**——OpenClaw 用来知道"当前 skill 集的状态"
- **EVOLVE_AGENTS.md 是协议文件**——OpenClaw 启动后第一件事就是读它
- **AGENTS.md / SOUL.md / IDENTITY.md / USER.md 是 bootstrap**——预写好让 OpenClaw 跳过(避免它浪费 tokens 自己写)
- **TOOLS.md / MEMORY.md / memory/ 是 OpenClaw 自己管**——不预写

**怎么走**(`prepare` 7 步):

```python
def prepare(self, sessions, existing_skills, manifest, agents_md, skill_registry_info):
    # 段 1:写 sessions/
    shutil.rmtree(self._sessions_dir, ignore_errors=True)
    for s in sessions:
        compact = _to_compact_session(s)  # 9 字段(见下表)
        (self._sessions_dir / f"{s['session_id']}.json").write_text(
            json.dumps(compact, ensure_ascii=False, indent=2)
        )

    # 段 2:写 skills/(清老 skill 子目录,按 existing_skills 重新写)
    existing_names = {sk["name"] for sk in existing_skills}
    for skill_dir in self._skills_dir.iterdir():
        if skill_dir.name not in existing_names:
            shutil.rmtree(skill_dir)  # 清多余的
    for skill in existing_skills:
        write_skill_bundle(self._skills_dir / skill["name"], skill, clean=True)

    # 段 3:写 manifest.json(workspace 视图,只读)
    (self._root / "manifest.json").write_text(json.dumps(manifest, indent=2))

    # 段 4:写 skill_registry.json(workspace 视图,只读)
    (self._root / "skill_registry.json").write_text(json.dumps(skill_registry_info, indent=2))

    # 段 5:写 EVOLVE_AGENTS.md(完整 407 行)
    (self._root / "EVOLVE_AGENTS.md").write_text(agents_md)

    # 段 6:预写 4 个 bootstrap 文件(被 OpenClaw writeFileIfMissing 'wx' 跳过)
    for name in ["AGENTS.md", "SOUL.md", "IDENTITY.md", "USER.md"]:
        path = self._root / name
        if not path.exists():  # writeFileIfMissing 语义
            path.write_text(_bootstrap_content(name))

    # 段 7:不预写 TOOLS.md / MEMORY.md / memory/(留给 OpenClaw 自己管)
    # openclaw_home 里的 MEMORY.md 是 --no-fresh 跨轮记忆的载体
```

**compact session 的 9 个字段**(写进 `sessions/<sid>.json`):

| 字段 | 含义 | 来源 |
|------|------|------|
| `session_id` | session UUID | session 顶层 |
| `task_id` | 任务的业务 ID(不是 session_id) | session.metadata.task_id(详见 14 章) |
| `num_turns` | session 内 turn 数 | session.turns 长度 |
| `aggregate` | session 维度聚合指标 dict | `_summarize_sessions` 阶段填 |
| `_skills_referenced` | list[str],session 内引用的 skill name | session 累积(去重 + 排序) |
| `_avg_prm` | float,session 平均 PRM 分 | session 累积 |
| `_has_tool_errors` | bool,session 是否有 tool 错误 | session 累积 |
| `_trajectory` | list[dict],step-by-step LLM 调用记录 | summarizer 程序生成(无损) |

`aggregate` dict 包含 `avg_prm` / `skill_usage_count` / `tool_error_count` 等——agent 用来快速判断 session 整体健康度,**不**是给 LLM 看的(LLM 看 `_summary`)。

**关键边界**:

- **`write_skill_bundle` 用 `clean=True`**:删除 `skills/<name>/` 整个目录再重建——这意味着 `--no-fresh` 模式下,`skills/<name>/history/` 也会被删(OpenClaw 之前加的 evidence 文件没了)
- **`_prepare_home` 单独管理 `openclaw_home`**:workspace 重建不影响 openclaw_home 里的 MEMORY.md——跨轮记忆的真正载体
- **bootstrap 文件用 `writeFileIfMissing` 语义**:`if not path.exists()` 判定,OpenClaw 自己用 `'wx'` flag 也用同样语义——不覆盖已存在
- **`existing_skills` 之外的 skill 被删**:本轮没引用的 skill 不在 workspace 里——OpenClaw 看不到它们(下一次 prepare 又会被加回来,如果 manifest 里有)
- **写文件失败**(权限 / 磁盘满):prepare 抛异常,run_once 整个 cycle 失败(下轮 cycle 重试)

**走完之后**:

- workspace 目录是**完整自洽的快照**:OpenClaw 启动后能**独立**判断"现状是什么、该改什么"
- sessions/ + skills/ + manifest.json + skill_registry.json 是 OpenClaw 读的所有内容
- EVOLVE_AGENTS.md 是协议文件——OpenClaw 读它知道"该做什么"
- bootstrap 文件已就位(避免 OpenClaw 浪费时间写) |
| `_summary` | str,LLM 生成的 session 摘要 | summarizer LLM 生成 |

### 2.3 `snapshot_skills()`

**场景**:`run_once` Step 6 在"OpenClaw 跑之前"拍 workspace 当前状态——给 Step 8 `collect_changes` 做 diff 用。这是"前后对比"模式的"前"快照。

**为什么需要 snapshot**:

`collect_changes` 要知道"OpenClaw 改了哪些 skill"——但**没有**"前"状态就不知道"什么是新的"。`snapshot_skills` 在 OpenClaw 启动前算每个 skill 的 `tree_sha256` 指纹,OpenClaw 跑完后再算一次,两次 diff 就是 OpenClaw 的改动。

**怎么走**(`snapshot_skills` 2 步):

```python
def snapshot_skills(self) -> dict[str, str]:
    snapshot = {}
    for skill_dir in sorted(self.skills_dir.iterdir()):
        if not skill_dir.is_dir():
            continue
        bundle = read_skill_bundle(skill_dir)  # 读 SKILL.md + references/ + scripts/ + assets/
        if "SKILL.md" in bundle:  # 没 SKILL.md 视为空 skill,跳过
            snapshot[skill_dir.name] = bundle_tree_sha256(bundle)
    return snapshot
```

**`bundle_tree_sha256` 怎么算**:把 bundle 里所有文件按 path 排序后算 `sha256(join(content_bys))`——**任意子文件变动都会让 hash 变**(包括 SKILL.md / references/ / scripts/ / assets/)。

**为什么只看"`SKILL.md` 在 bundle 里"**:

有些 skill 目录可能没有 SKILL.md(只放参考文件)——这些不算"可演化 skill",跳过。如果 OpenClaw 创建了"没 SKILL.md 的目录",也算空 skill。

**关键边界**:

- **`skills/` 是空目录**:返 `{}` 空 dict,Step 8 diff 出"全部新增"
- **skill 目录没 SKILL.md**:跳过(不计入 snapshot)
- **bundle 读失败**(权限/IO 错):`read_skill_bundle` 抛异常,run_once 整个 cycle 失败
- **bundle 有但 SKILL.md 解析失败**:bundle dict 不含 "SKILL.md" key,跳过

**走完之后**:

- 返 `{skill_name: tree_sha256}` 字典给 Step 8 `collect_changes` 用
- snapshot 是 Step 6 的"前"状态,Step 7 OpenClaw 跑完后再算一次得"后"状态

### 2.4 `collect_changes(before_snapshot)`

**场景**:`run_once` Step 8 在"OpenClaw 跑之后"扫 workspace,与 Step 6 的 `before_snapshot` diff——得出"OpenClaw 改了哪些 skill"。每个 change 会被 Step 9 循环 `_upload_skill` 上传到共享存储。

**为什么是 2 步而不是 1 步**:

- **第一步算"后"快照**:走和 `snapshot_skills` 一样的逻辑
- **第二步 diff before vs after**——根据 before_snapshot 里有没有、sha 变没变,决定 action(create / improve / ignore)

**怎么走**(`collect_changes` 2 步):

```python
def collect_changes(self, before_snapshot) -> list[dict]:
    # 段 1:算"后"快照
    after_snapshot = {}
    for skill_dir in sorted(self.skills_dir.iterdir()):
        if not skill_dir.is_dir():
            continue
        bundle = read_skill_bundle(skill_dir)
        if "SKILL.md" in bundle:
            after_snapshot[skill_dir.name] = bundle_tree_sha256(bundle)

    # 段 2:diff before vs after
    changes = []
    after_names = set(after_snapshot.keys())
    before_names = set(before_snapshot.keys())

    for name in after_names | before_names:
        before_sha = before_snapshot.get(name)
        after_sha = after_snapshot.get(name)

        if name not in before_names:
            # 新增
            action = "create"
        elif after_sha is None:
            # 删了——不支持 delete action
            logger.warning(f"skill {name} deleted by agent, not supported; ignored")
            continue
        elif before_sha != after_sha:
            # 修改
            action = "improve"
        else:
            # 没变
            continue

        # 读 bundle 准备上传
        bundle = read_skill_bundle(self.skills_dir / name)
        skill = parse_skill_md(bundle["SKILL.md"])  # 解析 frontmatter
        changes.append({
            "name": name,
            "action": action,
            "skill": skill,
            "bundle_files": bundle,  # {file_path: content} 字典
            "tree_sha256": after_sha,
        })
    return changes
```

**`action` 判定矩阵**:

| before_snapshot | after_snapshot | action |
|---|---|---|
| **没这个 name** | 有 | `create` |
| 有 | **没**(OpenClaw 删了目录) | `ignore`(`logger.warning`,不支持 delete) |
| 有 | 有 + sha 不同 | `improve` |
| 有 | 有 + sha 同 | **跳过**(没变) |

**`deleted` 不算 change**(设计选择):SkillClaw 故意不实现 delete action,只 create / improve。OpenClaw 如果"删了"某个 skill 目录,SkillClaw 记 warning 但忽略——下轮 cycle 又会被 prepare 加回来(因为 manifest 里有)。

**`conflict` 不处理**(`collect_changes` vs workflow 引擎 31 章 §6):

| 路径 | conflict 处理 |
|---|---|
| **workflow 引擎**(31 章 §6) | `_detect_conflict` + `_resolve_and_upload` + `_execute_merge` LLM 合并 |
| **agent 引擎**(本段 32 章) | **不**调 `_detect_conflict`——agent 改了 skill X 但 server 端 sha 已被别人改过,**没有 merge**,agent 的版本**直接覆盖**(见 §8.2 错误路径) |

**这是设计选择**——agent 引擎"信任 agent 的决定",不做 merge。如果团队部署(多 evolve server 并发)有并发改同一 skill 的风险,**不**用 agent 引擎,改用 workflow 引擎。

**关键边界**:

- **`deleted` skill**:忽略,记 warning,不上传
- **OpenClaw 没改任何 skill**:返 `[]` 空 list,Step 9 不上传,Step 10 finalize 仍走(清 sessions)
- **skill 解析失败**(frontmatter 错):跳过该 skill,记 error
- **多文件 bundle**:全部打进 `bundle_files: dict`,Step 9 `_upload_skill` 走 bundle_v1 协议(详见 20 章)

**走完之后**:

- 返 change 列表,每条 `{name, action, skill, bundle_files, tree_sha256}` 5 字段
- Step 9 循环每个 change 调 `_upload_skill`
- Step 10 finalize 收尾

### 2.5 `cleanup_sessions()`

删 `sessions/` 整个目录(保留其它数据,**不** reset workspace)——下次 cycle 的 `prepare` 会重建。这是 Step 10 的 3 个 finalize 子动作之一,另外两个见 §4.4。

## 3. `OpenClawRunner`(`openclaw_runner.py`,209 行)

对应 10 步里的 Step 3(`_prepare_home`)/ Step 7(`run`)/ Step 4 间接(`_write_config` 把 workspace 路径告诉 OpenClaw)。

### 3.1 构造

`OpenClawRunner` 的构造参数 8 个:`openclaw_bin` 默认 `"openclaw"`(在 PATH;env key `AGENT_EVOLVE_OPENCLAW_BIN`);`openclaw_home` 默认 `Path("")`(实际为 `<runner 构造时 cwd>/.openclaw_home`);`fresh` 默认 `True`(env key `AGENT_EVOLVE_FRESH`,`"1"`/`"true"` 视为 True);`timeout` 默认 `600` 秒(env key `AGENT_EVOLVE_TIMEOUT`);再加 4 个 LLM 客户端参数 `llm_api_key` / `llm_base_url` / `llm_model` / `llm_api_type`。

**`_openclaw_dir` / `_config_path`**:`self._openclaw_dir = self.openclaw_home / ".openclaw"`,`self._config_path = self._openclaw_dir / "openclaw.json"`。`openclaw_home` 是 OpenClaw 自己的 HOME,`_openclaw_dir` 才是 OpenClaw 真正读 config / sessions 的子目录。

**`config.group_id` 来源**:`self._agent_session_id = f"evolve-{config.group_id}"`——`config.group_id` 来自 `EvolveServerConfig.group_id`,用户启动 evolve server 时通过 `EVOLVE_GROUP_ID` env 传入(默认 `"default"`,详见 30 章)。

### 3.2 `_prepare_home()`

**场景**:`run_once` Step 7 调 `OpenClawRunner.run(...)` 之前,`_prepare_home` 准备 OpenClaw 自己的 HOME——`openclaw_home` 目录(与 workspace_root **正交**,见 §1.1)。

**为什么是 2 步而不是 1 步**:

- **第一步按 `fresh` 决定是否 wipe**:`fresh=True` 彻底清空(下一轮从零开始),`fresh=False` 保留上轮 OpenClaw 写的所有内容
- **第二步 mkdir 重建**:无论是否 wipe,都要确保目录存在

**怎么走**:

```python
def _prepare_home(self) -> None:
    # 段 1:按 fresh 决定 wipe
    if self.fresh and self.openclaw_home.exists():
        shutil.rmtree(self.openclaw_home, ignore_errors=True)
        # ignore_errors=True: 删失败的子文件不抛错,避免部分失败导致整个 cycle 挂

    # 段 2:mkdir 重建(无条件)
    self.openclaw_home.mkdir(parents=True, exist_ok=True)
    self._openclaw_dir.mkdir(parents=True, exist_ok=True)  # <openclaw_home>/.openclaw/
```

**`fresh=True` 时彻底 wipe `<openclaw_home>`**:

OpenClaw 自己的 memory / sessions / state 全部清零——下一轮 OpenClaw 启动时是"全新状态",看不到任何上轮的 MEMORY.md / memory/ / .openclaw/agents/main/sessions/。

**`fresh=False` 时保留**:

后续轮可以读上一轮 OpenClaw 的 memory:

- `MEMORY.md`(主记忆文件)
- `memory/` 目录(memory-core 管的辅助记忆)
- `.openclaw/agents/main/sessions/sessions.json`(OpenClaw 自己的 session 记录)

`mkdir(parents=True, exist_ok=True)` **不删**——这是 `--no-fresh` 跨轮记忆的核心机制:**保留什么**取决于上轮 OpenClaw 自己写的内容(OpenClaw 内部的 memory-core 在管),SkillClaw 这边只做"如果存在就复用"。

**关键边界**:

- **`fresh=True` + `openclaw_home` 不存在**:跳过 wipe 段(不存在无需删)
- **`fresh=True` + 删子文件失败**:`ignore_errors=True` 静默吞——可能漏删但 mkdir 时不会失败
- **`fresh=False` + 目录被外部删了**:`mkdir(parents=True, exist_ok=True)` 重建——OpenClaw 启动看不到上轮内容(等价于 fresh=True 效果)
- **openclaw_home 路径不在可写位置**:`mkdir` 抛 `PermissionError`,run_once 整个 cycle 失败

**走完之后**:

- `<openclaw_home>/` 和 `<openclaw_home>/.openclaw/` 两个目录都存在
- `fresh=True` 时目录是空的(只 mkdir 出来)
- `fresh=False` 时目录有上轮 OpenClaw 写的所有内容(MEMORY.md / memory/ / sessions/)——下轮 OpenClaw 启动能读
- 接下来 `_write_config(workspace_path)` 写 `openclaw.json`——告诉 OpenClaw 用哪个 LLM provider + 哪个 workspace

### 3.3 `_write_config(workspace_path)`

`_write_config` 写 `<openclaw_home>/.openclaw/openclaw.json`——SkillClaw **真正决策的字段**只有 6 个:第一组 `gateway.mode=local` + `gateway.bind=loopback`(本地模式、不暴露端口);第二组 `models.providers.evolve-llm={api, baseUrl, apiKey, models: [{id, name}]}`(LLM 客户端配置,`api` 字段从 `llm_api_type` 拿);第三组 `agents.defaults.model.primary = "evolve-llm/<llm_model>"`(指定 agent 用哪个 LLM provider);第四组 `agents.defaults.sandbox.mode=off`(关掉沙盒);第五组 `agents.defaults.workspace = <workspace_path absolute>`(OpenClaw 切到 SkillClaw 的工作区)。其它 OpenClaw schema 字段是默认值,SkillClaw 不动。

**6 个决策字段**:`gateway.mode=local` / `gateway.bind=loopback` / `models.providers.evolve-llm.{api,baseUrl,apiKey,models[0].id,name}` / `agents.defaults.model.primary` / `agents.defaults.sandbox.mode=off` / `agents.defaults.workspace`。

**其它字段都是 OpenClaw schema 默认值,SkillClaw 不动**(包括 `models[*].contextWindow=200000` / `maxTokens=16384` / `cost.{input,output,cacheRead,cacheWrite}=0` / `reasoning=false` / `input=["text"]`)。SkillClaw 在 `_write_config` 里**只**设上面 6 个决策字段,如果 `openclaw.json` 已有这些字段先 `setdefault`,不覆盖;之后用 `config.setdefault("gateway", {})` / `config["agents"] = ...` 写入 SkillClaw 决策值。

**`gateway.mode=local` + `bind=loopback`**:不让 OpenClaw 暴露端口(本地执行即可);`--local` CLI flag(见 §3.5)和 `gateway.mode=local` JSON 配置**等价**——CLI flag 优先级更高(后解析覆盖先解析),所以 `openclaw agent --local` 是 source of truth,JSON 是兜底。

**`agents.defaults.workspace`**:让 OpenClaw 把工作区切到 SkillClaw 的 workspace 路径(`workspace_path.resolve()` 绝对路径)。

**`sessions.json` bootstrap**:`agents/main/sessions/sessions.json` 如果不存在则写空对象 `{}`——OpenClaw 启动需要这个文件存在。

### 3.4 `_build_env()`

`_build_env` 走"拷父进程 env + 覆写 5 个 key"两步:第一步 `env = dict(os.environ)` 拷一份父进程环境;第二步 5 个赋值——`env["HOME"] = str(self.openclaw_home)`(隔离,见下)+ `env["OPENCLAW_HOME"] = str(self.openclaw_home)` + `env["OPENCLAW_CONFIG_PATH"] = str(self._config_path)` + `env["HF_HUB_OFFLINE"] = "1"`(禁用 HF Hub 联网)+ `env["TRANSFORMERS_OFFLINE"] = "1"`(禁用 transformers 联网)。

**`HOME` 覆盖**:`env["HOME"] = str(self.openclaw_home)` —— **有意为之**,隔离 OpenClaw 子进程对用户 home 目录的访问(避免 OpenClaw 误读 `~/.aws/credentials` / `~/.bashrc` 等)。如果 OpenClaw 真的需要访问用户真实 home,**必须**用绝对路径硬编码而不是依赖 `$HOME`。

**`HF_HUB_OFFLINE=1` + `TRANSFORMERS_OFFLINE=1`**:禁用 Hugging Face Hub 和 transformers 库的联网检查——保证 OpenClaw 不会在子进程启动时去拉新模型(网络不可用时直接报错而不是 hang)。

**`OPENCLAW_HOME` / `OPENCLAW_CONFIG_PATH`**:OpenClaw 自己读这两个 env(可能跟 `HOME` 重复,但 OpenClaw 内部优先用这两个)——确保 OpenClaw 找到 SkillClaw 写的 `openclaw.json` 而不是用户 home 下的同名文件。

### 3.5 `run(workspace_path, message, session_id=None)`

**场景**:`run_once` Step 7 调 `OpenClawRunner.run(...)`——这是**真正调 OpenClaw 子进程**的入口,把工作区路径 + 一句触发消息(通常是"演化这些 session")丢给 OpenClaw,等它跑完(改 SKILL.md 文件),再回来看 workspace 里变了什么(Step 8 `collect_changes`)。

**为什么是 3 步而不是 1 步**:

- **段 1 默认 session_id**:`fresh=True` 给 None 走 UUID(新 session 每次),`fresh=False` 复用 stable id(跨轮记忆的载体)
- **段 2 准备 home + config**:`_prepare_home` wipe/保留 openclaw_home + `_write_config` 写 `openclaw.json` 告诉 OpenClaw 用哪个 LLM provider + 哪个 workspace
- **段 3 跑 subprocess**:构造 cmd 列表 + 同步 `subprocess.run` 阻塞等 OpenClaw 跑完

**怎么走**:

```python
def run(self, workspace_path, message, session_id=None):
    # 段 1:默认 session_id
    if session_id is None:
        session_id = f"evolve-{uuid.uuid4().hex[:12]}"

    # 段 2:准备 home + config
    self._prepare_home()  # §3.2:按 fresh wipe/保留 openclaw_home
    self._write_config(workspace_path)  # §3.3:写 openclaw.json 6 字段

    # 段 3:跑 subprocess
    cmd = [
        self.openclaw_bin, "agent",
        "--session-id", session_id,
        "--agent", "main",
        "--message", message,
        "--json",
        "--local",  # 本地模式(不暴露 gateway)
        "--timeout", str(self.timeout),
    ]
    env = self._build_env()  # 隔离 env(只传 LLM 必要的 env vars)
    result = subprocess.run(
        cmd,
        cwd=str(workspace_path),  # OpenClaw 在 workspace 跑
        env=env,
        capture_output=True,
        text=True,  # stdout/stderr 是 str 而非 bytes
        check=False,  # 不抛 returncode 异常
        timeout=self.timeout + 30,  # 30s grace(给 OpenClaw 清理机会)
    )
    return result
```

**`session_id` 跨轮记忆**:`AgentEvolveServer._agent_session_id = f"evolve-{config.group_id}"`——同一 group 跨多次 cycle 用**相同**的 OpenClaw session_id(`run_once` 里 `session_id = self._agent_session_id if not self.config.fresh else None`):

| `fresh` | `session_id` | 跨轮记忆 |
|---|---|---|
| `True` | None → UUID | 每次新 session(不跨轮) |
| `False` | 复用 `evolve-<group_id>` | 跨轮维持 conversation history(结合 `openclaw_home/MEMORY.md` 读上轮 context) |

**`subprocess.run` 与 `asyncio.to_thread` 的关系**:

- `OpenClawRunner.run` 是**同步** `def`(不是 `async def`)
- **调用方** `AgentEvolveServer.run_once` 用 `await asyncio.to_thread(self._runner.run, ...)` 包装
- event loop **不**被同步阻塞(因为 `to_thread` 把它丢到线程池)

**stdout / stderr 持久化**:

- `subprocess.run` 本身带 `capture_output=True` + `text=True` 捕获 OpenClaw 输出
- **没有**把 stdout/stderr 持久化到 `evolve_history.jsonl` 或独立 log
- 只在 `agent_returncode != 0` 时 `logger.warning` 截前 500 字符 stderr
- `stdout` 拿到但丢弃(成功时只截前 500 字符 stderr 写 warning)

**`--local`**:OpenClaw 本地模式(不暴露 gateway / 不用远程 agent)。与 `openclaw.json` 里的 `gateway.mode=local` 等价——CLI flag 优先级更高。

**`timeout=self.timeout+30`**:

subprocess 自己 30s grace(如果 OpenClaw 卡住,先 `TimeoutExpired` 30s 后杀)。

**TimeoutExpired 处理**:`try/except` 捕到后返回 `subprocess.CompletedProcess(returncode=-1, stderr=f"TimeoutExpired: ...")`——cycle summary 里 `agent_returncode=-1`,Stage 8 finalize 记 error。

**`cwd=str(workspace_path)`**:OpenClaw 在 workspace 目录跑——保证所有相对路径(`skills/<name>/SKILL.md` 等)正确。

**`return type` 是 `subprocess.CompletedProcess[str]`**:`text=True` 让 stdout/stderr 是 str 而不是 bytes。`agent.py` 的 summary 只用 `result.returncode`,不用 `result.stdout` / `result.stderr`——这两个值是"看完即弃"的。

**关键边界**:

- **`openclaw` 不在 PATH**:`subprocess.run` 抛 `FileNotFoundError`,run_once 整个 cycle 失败
- **OpenClaw 子进程 timeout**:`TimeoutExpired` 抛出,30s grace 后杀子进程,cycle summary 记 `-1` returncode
- **OpenClaw 返回非零 returncode**:`subprocess.run(check=False)` 不抛,Stage 7 检查 returncode 决定是否继续 cycle
- **OpenClaw 改了文件但没改 SKILL.md**:`collect_changes` 仍能 diff(snapshot 拿前,bundle 读后)
- **`session_id` 跨多 evolve server 并发**:`evolve-<group_id>` 撞名——后启动的会覆盖前一个的 conversation history(团队部署要避免)

**走完之后**:

- OpenClaw 跑了,可能改了 `skills/<name>/SKILL.md` + 一些 bundle 文件
- subprocess 返回 `CompletedProcess`(returncode + stdout + stderr)
- `AgentEvolveServer` 拿到 result 进 Step 8 `collect_changes`
- 失败信息(非零 returncode)在 logger.warning 截前 500 字符 stderr

## 4. Step 9 · Upload 与 Registry 更新(`_upload_skill` + SkillIDRegistry)

> **本节是 §1 顶层 10 步里的 Step 9 + Step 10(部分)+ 可观测层**。从原 §3(OpenClawRunner)抽出独立成章——Step 9 跟 OpenClaw 子进程无关(子进程只是改了 workspace 文件,真正上传是 SkillClaw 自己做)。

每个 change 独立跑 "上传 bundle + 写 manifest + 写 registry" 三步,中间失败跳到下一个 change(不打断 cycle);最后所有 change 跑完才 `save_manifest`(全量重写一次)。

### 4.1 `_upload_skill(skill, bundle_files, action)` 流程

**场景**:`run_once` Step 9 对 `collect_changes` 返回的每个 change 调一次 `_upload_skill`——把"workspace 里的 SKILL.md + 辅助文件"上传到共享存储。这一步是"agent 引擎改的 skill"真正进入"团队共享"流程的入口。

**为什么是 11 步而不是 1 步**:

- **步骤 1-3**:前置检查(名字合法、分配 ID、补 SKILL.md)
- **步骤 4-5**:上传 SKILL.md + 辅助文件 + 删 stale files
- **步骤 6**:算 hash + bundle_record
- **步骤 7-8**:更新 registry + 写 v<N>/ 历史目录
- **步骤 9-11**:重新读 manifest + 写新条目 + 全量重写 manifest

**怎么走**:

```python
async def _upload_skill(self, skill, bundle_files, action):
    # 段 1:名字检查
    name = skill.get("name")
    if not name:
        return  # 空名跳过

    # 段 2:分配 skill_id
    skill_id = self._id_registry.get_or_create(name)  # 12-char hex,确定性

    # 段 3:补 SKILL.md
    if "SKILL.md" not in bundle_files:
        bundle_files["SKILL.md"] = build_skill_md(skill)

    # 段 4:上传 SKILL.md
    skill_md_key = f"{self._prefix}skills/{name}/SKILL.md"
    await self._call_storage(put_object, self._bucket, skill_md_key, bundle_files["SKILL.md"])

    # 段 5:上传其它文件 + 删 stale
    keep_keys = set(bundle_files.keys()) - {"SKILL.md"}
    for rel_path, content in bundle_files.items():
        if rel_path == "SKILL.md":
            continue
        file_key = f"{self._prefix}skills/{name}/files/{rel_path}"
        await self._call_storage(put_object, self._bucket, file_key, content)
    # 删 stale files
    existing_keys = await self._call_storage(list_object_keys, self._bucket, f"{self._prefix}skills/{name}/files/")
    for key in existing_keys:
        rel = key.split(f"skills/{name}/files/", 1)[1]
        if rel not in keep_keys:
            await self._call_storage(delete_object, self._bucket, key)

    # 段 6:算 hash + bundle_record
    content_sha = sha256(bundle_files["SKILL.md"].encode())
    tree_sha = bundle_tree_sha256(bundle_files)
    bundle_record = {
        "format": "bundle_v1",
        "entrypoint": "SKILL.md",
        "tree_sha": tree_sha,
        "files": sorted(bundle_files.keys()),
    }

    # 段 7:更新 registry
    version = self._id_registry.record_update(name, content_sha, action, bundle_record)
    # version + 1 + history 追加

    # 段 8:写 v<N>/ 目录
    await self._save_version_bundle(name, version, bundle_files, bundle_record)

    # 段 9:重新读 manifest
    manifest = await self._load_remote_skills()

    # 段 10:写新条目
    manifest[name] = {
        "name": name,
        "skill_id": skill_id,
        "version": version,
        "sha256": content_sha,
        "tree_sha256": tree_sha,
        "format": "bundle_v1",
        "entrypoint": "SKILL.md",
        "files": sorted(bundle_files.keys()),
        "uploaded_by": "evolve_server",
        "uploaded_at": iso_now(),
        "description": skill.get("description", ""),
        "category": skill.get("category", ""),
    }

    # 段 11:全量重写 manifest
    await self._save_manifest(self._bucket, self._prefix, manifest)
```

**11 步"为什么这么排"**:

| # | 步 | 失败处理 | 这一步的"为什么" |
|---|---|---|---|
| 1 | 名字检查 | 空名跳过 | 防止 LLM 给空 name 污染 storage |
| 2 | 分配 skill_id | get_or_create 必有返 | 确定性 ID 算法 `sha256(name)[:12]` |
| 3 | 补 SKILL.md | 没就构造 | 保证 bundle 必有 SKILL.md |
| 4 | 上传 SKILL.md | 失败抛错 | skill 入口文件必须成功上传 |
| 5 | 上传其它文件 + 删 stale | 失败记 warning | 不阻塞 cycle |
| 6 | 算 hash + record | 抛错 | 关键,registry 用 |
| 7 | 更新 registry | 失败只 warning | 见 §4.3 失败行为 |
| 8 | 写 v<N>/ 目录 | 失败记 warning | 历史版本完整保留 |
| 9-10 | 重读 + 写 manifest | 失败抛错 | 客户端 poll 这文件 |
| 11 | save_manifest | 失败抛错 | 全量重写是覆盖语义 |

**关键边界**:

- **`name=""`**:段 1 早返,不上传
- **`bundle_files` 没 `SKILL.md`**:段 3 用 `build_skill_md(skill)` 构造
- **stale files 删除失败**:段 5 记 warning,继续 cycle(下轮 cycle 还会重试)
- **save_manifest 失败**:抛错,这一 change 上传失败,Step 9 跳下一个 change
- **`record_update` 失败**(registry 写盘):只 logger.warning,**不**抛——见 §4.3 "失败只 logger.warning 不抛"
- **并发 agent 引擎**:多实例同时 `_upload_skill` 同一 skill,**没有** OSS ETag 串行化,可能丢变更(见 §4.2)

**走完之后**:

- 共享存储 `skills/<name>/SKILL.md` 写好
- 共享存储 `skills/<name>/files/<rel_path>` 写好
- 共享存储 `skills/<name>/versions/v<N>/` 历史目录写好
- 共享存储 `manifest.jsonl` 全量重写
- `evolve_skill_registry.json` 更新(name → skill_id, version, history)
- 客户端 30s 内 poll 到新 manifest,拉新 skill

### 4.2 `save_manifest` / 整文件覆盖

`<bucket>/<group_id>/manifest.jsonl` 的写盘策略(见 [33 · 共享层与存储适配](33-共享层与存储适配.md) §3):

- **写**:每行一条 skill 记录,`save_manifest` **全量重写** 整个文件(`"\n".join(lines) + "\n"`),不是行级 append。
- **读**:`load_manifest` 按行解析,`{skill_name: record}` 字典,后写覆盖前写(因为重名时 dict 赋值会覆盖)。
- **并发**:`bucket.put_object` 自身在 OSS/S3 后端是覆盖语义,**没有** atomic rename;并发场景(多 agent 引擎实例同时跑)可能读到半写状态。当前设计假设**单实例**——多实例时是否需要 OSS 的 ETag / If-Match 串行化,暂留(部署方按需决策)。
- **Nacos 路径**:`save_manifest` **不**调,改走 `nacos_skill_client.upload_skill_zip` + `submit()` + 视 `nacos_publish_mode` 决定 `publish()`(见 20 章 Nacos 适配器)。

### 4.3 `SkillIDRegistry` 细节

**类位置**:`evolve_server/core/skill_registry.py`(`SkillIDRegistry`)。agent 引擎在 `__init__` 里 `self._id_registry = SkillIDRegistry()` + `self._id_registry.load_from_oss(self._bucket, self._prefix)`(从 `<bucket>/<group_id>/evolve_skill_registry.json` 读已有 mapping)。

**`skill_id` 分配规则**:`get_or_create(name)` 走"查表 + 算新 id"两步:第一步 `entry = self._map.get(skill_name)`,命中就返 `entry["skill_id"]`(复用老 ID);第二步 `sid = hashlib.sha256(skill_name.encode()).hexdigest()[:12]` 算 12-char hex 作为新 ID(确定性 + 全局唯一 + 可重算)。
- **算法**:`sha256(name)[:12]`(前 12 个 hex 字符),**确定性**——同一个 name 永远映射到同一个 id;**全局唯一**(sha256 碰撞概率忽略不计);**可重算**(不需要集中协调)。
- **跨 group 共享**:**全 cluster 共享一份**——`evolve_skill_registry.json` 存在 `<bucket>/<group_id>/` 下,**每个 group 一份**,但**ID 分配函数对所有 group 一样**。所以**两个 group 各自有独立的 SkillIDRegistry 实例**(load 自己的 `evolve_skill_registry.json`),但同一 name 算出的 id 在两边是一样的。**"跨 group 共享 ID 空间"**指的是 id 算法一致,不是 registry 实例共享——是否需要跨 group registry 合并,暂留(部署方按需决策)。

**何时分配 ID**:`get_or_create(name)` 在每次 `record_update` 之前调一次(确保 entry 存在)。**首次 `record_update` 之前** `get_or_create` 就会创建 entry(`version=0, content_sha=""`),所以**整个 ID 在第一次 upload 之前就已经确定**——`record_update` 只把 `version+1`。

**`record_update(name, content_sha, action, bundle_record)` 行为**:
- `version += 1`,`content_sha` 更新;
- `history` 数组追加 `{version, content_sha, timestamp, action}`(**最多 20 条**,超过截尾);
- 返回新 version 号。

**`save_to_oss` / `load_from_oss`**:整个 dict 一次性 `put_object` 写到 `<bucket>/<group_id>/evolve_skill_registry.json`;**失败只 logger.warning 不抛**——所以 registry 写挂了 cycle summary 仍正常,只是下次启动 registry 缺数据(会从 0 version 重新建 entry,version 号会乱——见 §8.2)。

### 4.4 失败重试 / 部分成功

- **`put_object` 失败**:`agent.py` 的 Step 9 循环里 `try/except Exception as e: logger.error(...)`——**不**重试,跳到下一个 change;`skills_evolved` 计数跳过这个失败的。**部分成功**:`save_manifest` 在所有 change 跑完才写(成功 + 失败的子集都反映进去——失败的 change 没进 manifest)。
- **`delete_session_keys` 失败**:`oss_helpers.py:delete_session_keys` 内部 `try/except` 单独处理每个 key,返回成功删除数。**不抛**——失败的 key 留在 storage,下次 drain 会再读进来(可能重复处理)。**audit**:`delete_session_keys` **不**写 `evolve_history.jsonl`——是否加 audit 字段,暂留(部署方按需决策)。
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

**每行 schema**(agent 引擎 summary,见 `agent.py:run_once` 末尾的 `summary = {...}`):6 类字段——`timestamp` (ISO8601 UTC) / `elapsed_seconds` (单 cycle 耗时,float) / `sessions` (本轮 drain 的 session 数) / `skills_evolved` (本轮成功上传的 skill 数) / `agent_returncode` (OpenClaw 子进程 exit code,`TimeoutExpired` 时是 -1) + `evolutions` 列表(每个成功上传的 change 一条,内含 `action: create | improve` / `skill_name` / `skill_id` (12-char hex) / `version` (int) / `source: "agent"`)。

**与 workflow summary 的差异**(见 31 章 §10):agent summary **没有** `had_processing_error` / `skill_groups` / `validation_publish` / `session_judge` / `skill_verifier` 字段——agent 引擎的 10 步里没有 session_judge / skill_verifier / validation publish,所以 summary 也不带这些。

**与 `evidence.md` 的区别**:`history/v<N>_evidence.md` 是 OpenClaw 写在 workspace 里(`<workspace>/skills/<name>/history/`),记录 self-validation 结果;`evolve_history.jsonl` 是 SkillClaw 写在 evolve server cwd 里,记录每个 cycle 的宏观 summary——**两个文件不同,关注点不同**。

**生产监控**:41 章部署形态建议"定期 `cat evolve_history.jsonl` + `curl /status`"——配合 filebeat 拉走 + 磁盘告警(因为无 rotation,1 年会变 GB 级,见 §8.3 第 5 条)。

## 5. `EVOLVE_AGENTS.md` 协议(agent 行为宪法,407 行)

`evolve_server/engines/EVOLVE_AGENTS.md`(407 行)——**agent 引擎的"宪法"**。OpenClaw agent 读 `AGENTS.md`(SkillClaw 写的)被指向这个文件,然后严格按它的流程工作。

> **完整 407 行请直接读 `evolve_server/engines/EVOLVE_AGENTS.md`**——`wc -l engines/EVOLVE_AGENTS.md` 可确认行数。本节**不**重抄全文,只列出最关键的 7 步流程 + 4 条禁区——重写 SkillClaw 的 agent 引擎时,先看本节定位**流程边界**,再看 `EVOLVE_AGENTS.md` 拿**完整 prompt 文本**。

### 5.0 agent 的 7 步主流程(从 `EVOLVE_AGENTS.md` 协议体中提取)

OpenClaw agent 接到 prompt 后按 7 步走(对应 10 步里的 Step 4 部分 + Step 7 内部):**Step 1 · Read & Understand Session Data** ——读 `sessions/<session_id>.json` 的 `_summary`(快速概览)+ 必要时 `_trajectory`(step-by-step 细节);**Step 2 · Analyze Patterns** ——哪些 skill 被用了 / 哪些有效 / 哪些失败 / 哪些 skill 被创建了但没用;**Step 3 · Decide Action** ——对每个 skill / pattern 决策 SKIP / IMPROVE / OPTIMIZE_DESCRIPTION / CREATE 之一;**Step 4 · Read Existing Skill State** ——读 `skills/<name>/SKILL.md`(当前内容)+ 读 `skills/<name>/history/v<N>_evidence.md`(自检证据,必有)+ 可能读 `history/v<N>.md`(决策记录,可选);**Step 5 · Edit Skill** ——写 `skills/<name>/SKILL.md`,可改 body / description / frontmatter,**不可改** name(改名 = 创新 skill);**Step 6 · Self-Validate** ——检查 SKILL.md 是否符合 AgentSkills 协议(frontmatter / name / description / category),可选跑一次 LLM 验证评估改进质量,失败就 revert 或继续改;**Step 7 · Record** ——写 `skills/<name>/history/v<N>_evidence.md`(注:workspace `prepare()` 用 `clean=True` 重写,`history/` 实际不保留;OpenClaw 跨轮记忆的真正载体是 `openclaw_home/MEMORY.md`,见术语速查"两个 HOME")。

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

**`system` 字段处理**:`_AnthropicMessagesLLMClient` 走"分离 system + 重构 messages"两步:第一步开两个 list `system_parts` 和 `body_messages`,遍历 `messages` 每一项——拿 `role` 和 `content`(字符串化,空字符串兜底),`role == "system"` 且 `content` 非空就 append 到 `system_parts`(`continue`);`role in {"user", "assistant"}` 就 append `{"role": role, "content": content}` 到 `body_messages`;其它 role 丢弃。第二步组装 `request_body`——`model = self.model` + `messages = body_messages or [{"role": "user", "content": ""}]`(Anthropic API 要求 messages 非空,所以空时 fallback 一条空 user) + `max_tokens` / `temperature` 从 kwargs pop(用 self 的默认值兜底);最后如果 `system_parts` 非空,`request_body["system"] = "\n\n".join(system_parts)`(把 OpenAI Chat 风格的 system 提到 Anthropic 的顶层 `system` 字段)。
- `messages` 里 `role='system'` 的项被**提取合并**到 `body['system']` 字符串(`"\n\n"` join);
- `role in {'user', 'assistant'}` 的项保留在 `body['messages']`;
- 其它 role 丢弃;
- 如果 `body_messages` 空(全是 system),fallback `[{role:'user', content:''}]`(Anthropic API 要求 messages 非空)。

**重试与 backoff**:`_AnthropicMessagesLLMClient` 的 HTTP 调走"起 client + 6 次重试 + 指数 backoff"三步:第一步 `async with httpx.AsyncClient(timeout=httpx.Timeout(600.0, connect=30.0)) as client` 起异步 client(读超时 600s、连超时 30s);第二步 `for attempt in range(max_retries=6)` 跑 6 次循环(`attempt ∈ {0,1,2,3,4,5}`),循环体内 `try: resp = await client.post(self._messages_url(), json=request_body, headers=headers)` + `resp.raise_for_status()` + `payload = resp.json()` + 从 `payload.content` 提取所有 `type=='text'` 的 `text` 字段 `return "".join(parts)`;第三步异常处理——`except Exception:` 时如果 `attempt < max_retries - 1`(还没到最后一轮)就 `wait = min(2**attempt + random.uniform(0, 1), 30)` 算指数 backoff 等待时间(2^attempt + 0-1 随机抖动,最大 30s),`await asyncio.sleep(wait)` 等完 `continue` 进下一轮;否则 `raise` 把最后一轮的异常抛出去(由 `summarize_sessions_parallel` 处理)。
- **共 6 次**(含首试),`attempt ∈ {0, 1, 2, 3, 4, 5}`;
- **最大等待**:`min(2**attempt + random.uniform(0, 1), 30)`——`2**5 = 32`,被 30s 截掉,实际最大等 30s;
- 失败抛异常(`raise` 最后一轮的 exception),由 `summarize_sessions_parallel` 处理。

**不支持**:`vision` / `tool_use` / `cache_control` / `thinking`——只用 `messages: [{role, content}]` + `system` 字符串。agent 引擎未来是否需要 vision 给 agent 看图,暂留(按需扩展)。

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

## 8. 待确认 / 已知限制(分 3 类: 设计限制 / 错误路径 / 待对照验证)

> 本节分 3 类内容:**已知设计限制**(SkillClaw 当前实现就这样的,不是 bug) / **错误路径与边界**(生产 incident 时会发生的事) / **待对照验证**(还没在样本流量或部署环境实测的设计选择)。

### 8.1 已知设计限制

- **`openclaw` 必须装且能调 LLM**:`AgentEvolveServer` 构造时**不**校验 OpenClaw 是否能跑通——subprocess 启动失败才报错(`FileNotFoundError: 'openclaw'`,整个 cycle 抛异常,`run_periodic` 抓 Exception 走下一轮)。**这是设计选择**——减少启动时间 / openclaw 升级可能 break CLI flag,出错让 subprocess 失败自然浮上来。
- **`_AnthropicMessagesLLMClient` 不支持 vision / tool_use / cache_control / thinking**:只用 `messages: [{role, content}]` + `system` 字符串。**待确认**对 vision 任务(agent 看图片)够不够用。
- **`AnthropicClient` 不持久化连接**:每次 `_summarize_sessions` 都新建 `httpx.AsyncClient`(在 `async with` 里),不连接池——性能调优点。
- **`--no-fresh` + OpenClaw memory 累积**:OpenClaw 的 `openclaw_home/MEMORY.md` 会**无界**增长(OpenClaw 内部 memory-core 在管,SkillClaw 不介入)。**待确认** OpenClaw 自己有没有 rotation。
- **`_id_registry` 跨 group 共享 ID 空间**:**id 算法一致**(`sha256(name)[:12]`),**registry 实例独立**(每 group 自己的 `evolve_skill_registry.json`)。同一 name 在两个 group 算出同一 id,但 registry 不合并——是否需要跨 group registry 合并,暂留(部署方按需决策)。
- **`_notify_proxy_reload` agent 引擎不调**:workflow 引擎 Step 10.5 会 `if uploaded_skills > 0: _notify_proxy_reload()`(31 章 §10 步 11);agent 引擎**没**实现这一步——是设计遗漏还是有意,暂留。
- **agent 引擎不实现 Nacos 路径**:`_load_remote_skills` 在 workflow 里有 Nacos override,`AgentEvolveServer` **没** override——`skill_storage_backend="nacos"` 走 agent 会失败(底层用 default `_load_remote_skills` 拉 `manifest.jsonl`,Nacos 后端没有这个文件)。

### 8.2 错误路径与边界

- **agent 改完但 LLM `_summarize_sessions` 调用失败**(网络挂 / LLM 5xx):`summarize_sessions_parallel` 内部会失败,session 写不进 workspace → `prepare` 时 compact 字段空(`_summary=""` / `_trajectory=""`)→ agent 看不到 summary。**部分成功**:`agent.py` 的 try/except 包了 Step 5 之后的所有步骤,但 Step 2 `_summarize_sessions` 失败会让 `agent.py:run_once` 抛异常,Step 3-10 **不**跑。
- **agent 改了 skill X,server 端 sha 已被别人改过**(并发场景):`collect_changes` **不**调 `_detect_conflict`,Step 9 直接覆盖——**last-write-wins**,agent 改的可能丢掉别人的改动。
- **未知 `llm_api_type` fallback**:退化路径只填 `_trajectory`(程序生成的),`_summary=""`——agent 看不到 LLM 摘要,只能看 trajectory。
- **`save_to_oss`(registry)失败**:`logger.warning` 不抛,**summary 不带 had_processing_error**(agent summary 没这个字段)——registry 写挂了**静默**;下次启动 registry 从 0 version 重建,version 号会乱(从 0 而不是上次 version+1 开始)。生产建议 filebeat 拉 logger。
- **`delete_session_keys` 失败**:失败的 key 留在 storage,下次 drain 会再读进来——**可能重复处理**(agent 跑两遍,产生 2 个 evolution record)。**audit** 不写 `evolve_history.jsonl`。
- **`subprocess.run` `TimeoutExpired`**:OpenClaw 卡死超过 `timeout+30s` → `result.returncode=-1` + `result.stderr="TimeoutExpired: ..."` → `agent_returncode=-1` 写 `evolve_history.jsonl`,但 Step 8 collect_changes 仍跑(因为 `cwd` 已设好,workspace 文件是上一轮 step 7 中间状态)——**可能**正常 diff,**可能** diff 到半成品(取决于 OpenClaw 卡死在哪个文件写完之后)。
- **`run_periodic` Ctrl-C 退出要等 `interval_seconds`**:循环里 `await asyncio.sleep(self.config.interval_seconds)`,signal handler 设了 `self._running = False` 但 sleep 不能被打断——最坏等 600s。生产建议 `TimeoutStopSec=5s` 的 systemd unit 或起独立 stop 协程(31 章 §11 第 8 条同问题)。

### 8.3 暂留(待对照验证)

> 下面是还没在样本流量或部署环境实测过的设计选择,部署方按需决定是否验证 / 修复。

- **`openclaw_home` 默认值**:`evolve_server/core/config.py:_PACKAGE_DIR = Path(__file__).resolve().parent` 解析到 `evolve_server/core/`(不是 `evolve_server/`),所以 `__post_init__` 写 `self.openclaw_home = str(_PACKAGE_DIR / ".openclaw_home")` 实际落点 `evolve_server/core/.openclaw_home`——变量名是 `_PACKAGE_DIR` 但解析到 `core/` 子目录,与命名暗示不符;30 章字段表里只写"`<_PACKAGE_DIR>/.openclaw_home`"不展开。**待实测**是不是该 `Path(__file__).resolve().parent.parent`。
- **`workspace_root` 默认值**:同上,落点 `evolve_server/core/agent_workspace`;41 章 §9.2 写"`<pkg>/agent_workspace`"是否指 `evolve_server/` 还是 `evolve_server/core/`。
- **`--local` CLI flag vs `gateway.mode=local` JSON**:两者等价但 CLI 优先级更高——**待实测** OpenClaw 是不是真的 CLI 优先。
- **`subprocess.run` 的 stdout/stderr 持久化**:当前只在 `agent_returncode != 0` 时 `logger.warning` 截前 500 字符 stderr;成功的 stdout 拿到但丢弃。**待实测**是否要加 `agent_stderr` 字段到 `evolve_history.jsonl`,或写独立 `<workspace>/agent_stderr.log`。
- **`<workspace>/skills/<name>/history/` 不保留**:`prepare` 用 `clean=True` 重写,history/ 删——**待实测**是不是该把 history/ 加到 preserve 列表里(配合 `--no-fresh` 模式)。
- **`evolve_history.jsonl` 无界增长**:`_append_history` 只 append 不 rotate,生产 1 年会变 GB 级(30 章 §7 第 3 条 + 31 章 §11 第 6 条同问题)。
- **`save_manifest` 并发安全**:多 agent 引擎实例并发时可能读到半写状态(无 file lock / 无 ETag)——目前设计假设单实例。
- **`_id_registry` 跨 group 合并**:两 group 各自 registry,但 id 算法一致;**待实测**是否要 `global registry` 跨 group 共享。
- **`AnthropicClient` vision 支持**:`_AnthropicMessagesLLMClient` 不处理 `image` content block;**待实测** agent 引擎未来是否需要看图。
- **`memory-core` 定义**:`_EVOLVE_AGENTS_MD` 模板里写"You may use `memory/` and `MEMORY.md` for long-term notes across rounds",`memory-core` 是 OpenClaw 自己的模块(不在 SkillClaw 仓库);**待实测** memory-core 是不是 OpenClaw 的包名,还是 SkillClaw 抽象。
- **`HF_HUB_OFFLINE=1` + `TRANSFORMERS_OFFLINE=1` 是不是够**:OpenClaw 内部如果用别的下载器(比如 modelscope / 自家 hub),这俩 env 不挡——**待实测** OpenClaw 用了哪些下载器。
- **测试覆盖**:目前无单元测试覆盖 agent 引擎(强依赖 OpenClaw 子进程);建议在沙盒里跑 `--interval 60` 24 小时,人工 review `evolve_history.jsonl`。
- **`/status` 端点**:`create_http_app` 提供 `/trigger` / `/status` / `/health`,`/status` 返回 `{running, pending_sessions, registered_skills, skills, fresh_mode}`——**待实测** `/status` 在多 engine instance 下的语义(`self._running` 是单实例标志位)。

---

→ **下一篇**:[33-共享层与存储适配](33-共享层与存储适配.md) — object store 抽象 + evolve server 怎么读写 storage
