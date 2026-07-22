---
chapter: 03
title: 核心流程 - 目录
depends_on: [02-整体架构]
estimated_read_min: 1
---

# 核心流程 - 目录

端到端的 6 条关键流程,按"client 优先 → 跨进程 → server 优先 → 可选"排列:

| # | 流程 | 阅读用时 | 关键代码位置 |
|---|---|---|---|
| 03.01 | [LLM 请求拦截与 skill 注入](./03.01-llm请求拦截与skill注入.md) | 12 min | `api_server.py:1570` (`/v1/chat/completions`) |
| 03.02 | [会话录制、归档与 PRM 评分](./03.02-会话录制与归档.md) | 10 min | `api_server.py` (后置钩子) + `prm_scorer.py` |
| 03.03 | [共享 skill 同步:本地 ↔ 共享存储](./03.03-共享skill同步.md) | 8 min | `skill_hub.py` (pull/push) + `object_store.py` |
| 03.04 | [Workflow 引擎的 session → skill 进化](./03.04-workflow引擎进化.md) | 15 min | `engines/workflow.py:894` (`run_once`) + `pipeline/execution.py` |
| 03.05 | [Agent 引擎的 skill 编辑(OpenClaw 驱动)](./03.05-agent引擎编辑.md) | 8 min | `engines/agent.py:306` + `engines/openclaw_runner.py` |
| 03.06 | [后台 validation worker(可选)](./03.06-后台验证.md) | 5 min | `validation_worker.py:204` (`run_once`) + `validation_store.py` |

## 阅读顺序

- **"只想懂 client"**:03.01 → 03.02 → 03.03
- **"只想懂 server"**:03.04 → 03.05
- **"完整链路"**:03.01 → 03.02 → 03.03 → 03.04(3 阶段管线)→ 03.05(另一种实现)→ 03.06(回路)

每条流程都按"设计动机 → 核心概念 → 实现流程 → 关键文件 → 常见误区"五段式展开,见各章首页。
