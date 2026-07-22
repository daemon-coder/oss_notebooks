---
title: SkillClaw 源码书 - 总目录
---

# SkillClaw 源码书

> 读源码,理解一个让 AI Agent 技能"集体进化"的系统是怎么搭起来的。

SkillClaw 是一个面向 LLM Agent 的**技能(skill)生命周期管理框架**。它由两个相对独立的进程组成:

- **Client Proxy**(`skillclaw/`):本地 API 代理,拦截 Agent 的 LLM 请求,按需注入相关 skill,记录完整会话,按需把会话同步到共享存储。
- **Evolve Server**(`evolve_server/`):周期性地从共享存储读出会话,调用 LLM 把一批会话凝练/演进成可复用的 skill 文档,再写回共享存储。

两套进程**只通过共享对象存储通讯**(local / Alibaba OSS / S3 / Nacos Skill Registry),所以用户本机只需要装 Client,运维侧才需要部署 Evolve Server。

---

## 阅读路径

如果你想按下面的顺序读,大约 90 分钟能通读全本:

| # | 章节 | 目的 |
|---|---|---|
| 1 | [00-前言.md](./00-前言.md) | 文档目标 + 读者画像 + 阅读节奏 |
| 2 | [01-项目概述.md](./01-项目概述.md) | 一句话定位、核心特性、典型场景 |
| 3 | [02-整体架构.md](./02-整体架构.md) | 双进程架构、共享存储桥、模块划分 |
| 4 | [03-核心流程/](./03-核心流程/INDEX.md) | 端到端走 6 条关键流程(请求注入、session 录制、pull/push、workflow 进化、agent 引擎、后台验证) |
| 5 | [04-模块详解/](./04-模块详解/INDEX.md) | 12 个核心模块逐一深读 |
| 6 | [05-关键设计决策.md](./05-关键设计决策.md) | 一些"为什么这样写"的设计权衡 |
| 7 | [06-附录.md](./06-附录.md) | 术语表 + 引用文件清单 + 待确认项 |

---

## 章节清单

### 顶层

- [00-前言.md](./00-前言.md)
- [01-项目概述.md](./01-项目概述.md)
- [02-整体架构.md](./02-整体架构.md)
- [05-关键设计决策.md](./05-关键设计决策.md)
- [06-附录.md](./06-附录.md)

### 03 - 核心流程

- [03-核心流程/INDEX.md](./03-核心流程/INDEX.md)
- [03.01 - LLM 请求拦截与 skill 注入](./03-核心流程/03.01-llm请求拦截与skill注入.md)
- [03.02 - 会话录制、归档与 PRM 评分](./03-核心流程/03.02-会话录制与归档.md)
- [03.03 - 共享 skill 同步:本地 ↔ 共享存储](./03-核心流程/03.03-共享skill同步.md)
- [03.04 - Workflow 引擎的 session → skill 进化](./03-核心流程/03.04-workflow引擎进化.md)
- [03.05 - Agent 引擎的 skill 编辑(OpenClaw 驱动)](./03-核心流程/03.05-agent引擎编辑.md)
- [03.06 - 后台 validation worker(可选)](./03-核心流程/03.06-后台验证.md)

### 04 - 模块详解

- [04-模块详解/INDEX.md](./04-模块详解/INDEX.md)
- [04.01 - API 代理服务 `api_server.py`](./04-模块详解/04.01-api代理服务.md)
- [04.02 - 协议适配层 `protocols/`](./04-模块详解/04.02-协议适配层.md)
- [04.03 - Skill 管理器 `skill_manager.py`](./04-模块详解/04.03-skill管理器.md)
- [04.04 - Skill 共享中心 `skill_hub.py` + `object_store.py`](./04-模块详解/04.04-skill共享中心.md)
- [04.05 - Nacos Skill Registry 适配 `nacos_skill_hub.py`](./04-模块详解/04.05-nacos适配.md)
- [04.06 - 代理适配层 `claw_adapter.py`](./04-模块详解/04.06-代理适配层.md)
- [04.07 - 配置中心 `config.py` + `config_store.py`](./04-模块详解/04.07-配置中心.md)
- [04.08 - 守护进程与启动链路 `launcher.py` / `cli.py` / `setup_wizard.py`](./04-模块详解/04.08-守护进程与启动.md)
- [04.09 - PRM 评分 `prm_scorer.py`](./04-模块详解/04.09-prm评分.md)
- [04.10 - 后台验证 `validation_worker.py` + `validation_store.py`](./04-模块详解/04.10-后台验证.md)
- [04.11 - 可视化面板 `dashboard_*.py`](./04-模块详解/04.11-可视化面板.md)
- [04.12 - Evolve Server 核心 `evolve_server/core/`](./04-模块详解/04.12-evolve_server核心.md)
- [04.13 - 三阶段流水线 `evolve_server/pipeline/`](./04-模块详解/04.13-三阶段流水线.md)
- [04.14 - 服务端引擎 `evolve_server/engines/`](./04-模块详解/04.14-服务端引擎.md)

---

## 全局图(50 词版)

```
[LLM Agent: Hermes/Codex/Claude Code/...]
        │ HTTPS (OpenAI / Anthropic 协议)
        ▼
[Client Proxy  ·  FastAPI  ·  skillclaw/]
   ├─ 协议规范化 + skill 检索注入
   ├─ session 录制 + PRM 打分
   └─ 守护进程、setup 向导、dashboard
        │
        │ (可选) shared object storage
        │   local / OSS / S3 / Nacos Skill
        ▼
[Evolve Server  ·  asyncio  ·  evolve_server/]
   ├─ 摘要 → 聚合 → 进化 LLM
   ├─ Workflow 引擎(3 阶段 LLM)
   └─ Agent 引擎(OpenClaw 子进程)
```

详细解读见 [02-整体架构.md](./02-整体架构.md)。

---

## 文档约定

- 文件引用采用 `path/to/file.py:42` 形式,行号指向**关键实现位置**而非复制整段代码。读者可自行打开源码定位。
- API 端点、配置项、CLI 命令保留原始英文名,**不**做中文翻译,以便和源码对账。
- 涉及"为什么这样设计"的讨论,统一汇总到 [05-关键设计决策.md](./05-关键设计决策.md)。
- 源码读不透的项,会在 [06-附录.md](./06-附录.md) 的"待确认清单"中标出。
