# SkillClaw 源码书 · 目录

> 面向"完全没读过 SkillClaw 源码"的工程师，按"先总后分 + 设计思路导向 + 数据流串讲"的笔法，把这个让 AI agent skills 从真实交互中集体演化的系统讲透。
> 读完本书，删掉源码只留这一套文档，另一位工程师能照着重写出行为一致的 SkillClaw。

SkillClaw 是 AMAP-ML 开源的项目（MIT 协议，仓库地址 `https://github.com/AMAP-ML/SkillClaw`，论文 arXiv:2604.08377）。本书源码以 `https://github.com/AMAP-ML/SkillClaw` 当前 `main` 分支为准（基于本地克隆 `code/oss/SkillClaw/`）。

---

## 阅读顺序

整套书分五层目录（"总览层 / 模块层 / 细节层"），从前往后、从抽象到具体。**读者不需要先懂代码再读本书**——每一章都假设你"还没看过相关文件"，从章首的三段全景讲起。

### 00 · 总览层（先读这三章建立心智模型）

| 序 | 标题 | 一句话简介 |
|----|------|-----------|
| 00 | [序章：这本书怎么读](00-总览/00-序章.md) | 阅读约定、术语一览、章节依赖图、哪些章节需要"先读再读" |
| 01 | [全景与设计哲学](00-总览/01-全景与设计哲学.md) | 什么是 SkillClaw、为什么存在、Two Loops 怎么走、目录全貌、两个组件怎么分工 |
| 02 | [核心概念词典](00-总览/02-核心概念词典.md) | SKILL.md / AgentSkill / Claw / Two Loops / 共享存储后端 / Sharing Backend / Skill Backend / Session / Turn / PRM / Effectiveness / Publish Mode / Skill Verifier / Session Judge / 等等 |

### 01 · 客户端运行时（你本机跑的那一段）

| 序 | 标题 | 一句话简介 |
|----|------|-----------|
| 10 | [CLI 与启动生命周期](01-客户端运行时/10-CLI与启动.md) | `skillclaw` 命令、daemon 拉起、`SkillClawLauncher`、`runtime_state` 锁、信号 |
| 11 | [配置系统与字段归一化](01-客户端运行时/11-配置系统.md) | `~/.skillclaw/config.yaml` 怎么写、`ConfigStore` 怎么读、`to_skillclaw_config` 怎么把分散字段归一到一个 dataclass、`SetupWizard` 的全部提问 |
| 12 | [代理服务器总览](01-客户端运行时/12-代理服务器总览.md) | `SkillClawAPIServer` 这个类在 FastAPI 上挂了哪些端点、健康检查、internal reload、OpenAI/Responses/Anthropic 三个端点之间的协议分工 |
| 13 | [请求处理与协议适配](01-客户端运行时/13-请求处理与协议适配.md) | `/v1/chat/completions`、`/v1/responses`、`/v1/messages`、`/v1/messages/count_tokens` 进入后经过的每一步钩子、协议互转（OpenAI Chat ↔ Responses ↔ Anthropic Messages）、system prompt 压缩、context 截断 |
| 14 | [Session 与 Turn 簿记](01-客户端运行时/14-Session与Turn簿记.md) | 怎么把一堆 stateless HTTP 请求串成"一次会话"、TUI 边界启发式、idle sweeper、record 文件、`_close_session` 的全流程、PRM 怎么和 turn 联动 |
| 15 | [技能库与注入](01-客户端运行时/15-技能库与注入.md) | `SkillManager` 从 SKILL.md 读出来的内部模型、`build_injection_prompt` 怎么把 N 个技能压成系统提示、template/embedding 两种检索模式、stats/feedback 怎么流回去 |
| 16 | [PRM 与反馈回路](01-客户端运行时/16-PRM与反馈.md) | `PRMScorer` 怎么打分、`prm_m` 多数投票、turn 反馈怎么写回 `skill_stats.json`、Bedrock provider 分支 |
| 17 | [适配器矩阵](01-客户端运行时/17-适配器矩阵.md) | `claw_adapter` 怎么把 12 种 CLI agent（OpenClaw / Hermes / Codex / Claude Code / OpenCode / QwenPaw / IronClaw / PicoClaw / ZeroClaw / NanoClaw / NemoClaw / none）的本地配置改写、备份、回滚 |

### 02 · 客户端共享层（多端 + 多人 + 可观测）

| 序 | 标题 | 一句话简介 |
|----|------|-----------|
| 20 | [共享存储与同步](02-客户端共享层/20-共享存储与同步.md) | 共享存储布局（`{group_id}/skills/` / `manifest.jsonl` / `sessions/` / `validation_*` / `candidate_skills/` / `evolve_skill_registry.json`）、三种对象存储后端（local / S3 / OSS）、`SkillHub` 增量同步、Nacos 适配器、bundle 哈希协议 |
| 21 | [后台验证工作流](02-客户端共享层/21-后台验证工作流.md) | `validation.enabled` 触发的 idle-time 验证器：replay 打分、quota、job/result/decision 三段状态、publish_mode=validated 时的"先验证再发布"完整流程 |
| 22 | [Dashboard 与可观测性](02-客户端共享层/22-Dashboard与可观测性.md) | `skillclaw dashboard sync` / `serve` 命令、SQLite 投影、`DashboardService` 的全部 HTTP 端点、把本地 skill/session/validation 聚合到只读视图 |

### 03 · Evolve 服务（后台那一段）

| 序 | 标题 | 一句话简介 |
|----|------|-----------|
| 30 | [Evolve 服务端与引擎选择](03-Evolve服务/30-Evolve服务端与引擎选择.md) | `python -m evolve_server` 的入口、`EvolveServerConfig` 全部字段、env vs skillclaw-config 两种来源、workflow vs agent 两种引擎的取舍 |
| 31 | [Workflow 引擎详解](03-Evolve服务/31-Workflow引擎详解.md) | 6 步流水线：Drain → Summarize → Session Judge → Aggregate → Evolve/Create → Upload，含 `_queue_validation_job` 旁路 + `_finalize_validation_jobs` 收尾 + skill verifier 闸门 + merge 冲突 |
| 32 | [Agent 引擎详解](03-Evolve服务/32-Agent引擎详解.md) | OpenClaw 子进程化、`AgentWorkspace` 文件级 diff、`EVOLVE_AGENTS.md` 协议、bootstrap 文件预写、change detection、`--no-fresh` 跨轮记忆 |
| 33 | [共享层与存储适配](03-Evolve服务/33-共享层与存储适配.md) | `storage.oss_helpers` 的 manifest / session 读写、bundle v1 布局、`mock_bucket` 本地替代、对象存储多后端选型（OSS/S3/local/Nacos-only） |

### 04 · 端到端（细节层：数据流与部署）

| 序 | 标题 | 一句话简介 |
|----|------|-----------|
| 40 | [数据流总览](04-端到端/40-数据流总览.md) | 一条 `/v1/chat/completions` 请求从 Hermes 进来、经过代理、注入技能、转发 LLM、回写 record、触发 PRM、上传 session、被 evolve 读取、改写 skill、再被下次同步拉回本地——完整生命周期 |
| 41 | [部署形态](04-端到端/41-部署形态.md) | 单机自用 / 团队共享 / validated 三种部署模式、各自的最小配置、运行手册、可观测与回滚 |

---

## 层级分布

- **总览层（overview）**：3 章
- **模块层（module）**：15 章
- **细节层（detail）**：2 章
- **合计**：20 篇正文 + 1 篇 INDEX

---

## 覆盖映射（源码目录 → 负责讲的章节）

> 这张表是给评审官和后续轮次的"漏没漏"做兜底用的——每个源码目录 / 子系统都至少有一章认领。**没有出现"未认领"的代码块**（`assets/` 图片资源、`docs/` 空目录、`tests/`、`requirements*.txt`、`pyproject.toml` 已在"全景"中提过；代码层所有目录都覆盖到了）。

| 源码目录 / 文件 | 负责章节 |
|---|---|
| `skillclaw/__init__.py` | 01 / 10 |
| `skillclaw/__main__.py` | 10 |
| `skillclaw/cli.py` | 10 / 22 |
| `skillclaw/launcher.py` | 10 |
| `skillclaw/runtime_state.py` | 10 |
| `skillclaw/log_color.py` | 10 |
| `skillclaw/utils.py` | 12 |
| `skillclaw/setup_wizard.py` | 11 / 41 |
| `skillclaw/config.py` | 11 |
| `skillclaw/config_store.py` | 11 |
| `skillclaw/api_server.py` | 12 / 13 / 14 |
| `skillclaw/skill_manager.py` | 15 |
| `skillclaw/skill_bundle.py` | 15 / 20 |
| `skillclaw/skill_hub.py` | 20 |
| `skillclaw/nacos_skill_hub.py` | 20 |
| `skillclaw/nacos_versions.py` | 20 |
| `skillclaw/object_store.py` | 20 / 33 |
| `skillclaw/claw_adapter.py` | 17 |
| `skillclaw/prm_scorer.py` | 16 |
| `skillclaw/bedrock_client.py` | 13 / 16 |
| `skillclaw/validation_worker.py` | 21 |
| `skillclaw/validation_store.py` | 21 |
| `skillclaw/dashboard_ingest.py` | 22 |
| `skillclaw/dashboard_store.py` | 22 |
| `skillclaw/dashboard_server.py` | 22 |
| `skillclaw/protocols/common.py` | 13 |
| `skillclaw/protocols/anthropic_messages.py` | 13 |
| `skillclaw/protocols/openai_responses.py` | 13 |
| `evolve_server/__init__.py` | 30 |
| `evolve_server/__main__.py` | 30 |
| `evolve_server/core/constants.py` | 30 |
| `evolve_server/core/utils.py` | 31 / 32 |
| `evolve_server/core/config.py` | 30 |
| `evolve_server/core/llm_client.py` | 30 / 31 |
| `evolve_server/core/skill_registry.py` | 31 |
| `evolve_server/storage/mock_bucket.py` | 33 |
| `evolve_server/storage/oss_helpers.py` | 31 / 33 |
| `evolve_server/pipeline/summarizer.py` | 31 |
| `evolve_server/pipeline/aggregation.py` | 31 |
| `evolve_server/pipeline/execution.py` | 31 |
| `evolve_server/pipeline/session_judge.py` | 31 |
| `evolve_server/pipeline/skill_verifier.py` | 31 |
| `evolve_server/engines/common.py` | 30 / 31 / 32 |
| `evolve_server/engines/workflow.py` | 31 |
| `evolve_server/engines/agent.py` | 32 |
| `evolve_server/engines/agent_workspace.py` | 32 |
| `evolve_server/engines/openclaw_runner.py` | 32 |
| `evolve_server/engines/agents_md.py` | 32 |
| `evolve_server/engines/EVOLVE_AGENTS.md` | 32 |
| `scripts/install_skillclaw.sh` | 41 |
| `scripts/install_skillclaw_server.sh` | 41 |
| `scripts/demo_nacos_skill_lifecycle.py` | 20 |
| `pyproject.toml` / `requirements*.txt` | 01 / 41 |
| `evolve_server/evolve_server.env.example` | 30 / 41 |
| `client_env.example.sh` | 41 |
| `assets/*`（8 张 SVG/PNG） | 01 |
| `tests/`（24 个测试文件） | （不在本书范围——本书只覆盖产品代码；测试在 `tests/` 单独维护） |

---

## 阅读建议

- **第一次读**：按章节顺序从 00 → 04-端到端/41 走一遍，建立完整心智模型。
- **查特定功能**：直接看 INDEX 的覆盖映射，跳到对应章节。
- **排查 bug**：优先读 13（请求处理）、14（Session 簿记）、31/32（Evolve 引擎），这三章是 Bug 集中地。
- **接入新 CLI agent**：先读 17（适配器矩阵），再读 11（配置系统）。
- **部署到团队**：直接看 41（部署形态）。
