---
chapter: 04
title: 模块详解 - 目录
depends_on: [02-整体架构, 03-核心流程]
estimated_read_min: 1
---

# 模块详解 - 目录

> **模块计数对照**:本章表格列了 14 行——**client 端 11 个(04.01-04.11)+ server 端 3 个(04.12-04.14)= 总 14 个**。
> 02 章顶视图按"client 11"统计(行 2.1),03 章按"6 条流程"组织,本章按"14 个模块"组织——这是同一套系统在不同视角下的三种切分,数字差异是粒度差异,不是统计错误。

14 个核心模块逐一深读,按"client 优先 → 跨进程 → server 优先 → 可选"排列。

| # | 模块 | 关键文件 | 行数 | 难度 |
|---|---|---|---|---|
| 04.01 | [API 代理服务](./04.01-api代理服务.md) | `skillclaw/api_server.py` | 3438 | ★★★★★ |
| 04.02 | [协议适配层](./04.02-协议适配层.md) | `skillclaw/protocols/` | 2 文件 | ★★★★ |
| 04.03 | [Skill 管理器](./04.03-skill管理器.md) | `skillclaw/skill_manager.py` | 851 | ★★★ |
| 04.04 | [Skill 共享中心](./04.04-skill共享中心.md) | `skillclaw/skill_hub.py` + `object_store.py` | 837 + 255 | ★★★★ |
| 04.05 | [Nacos Skill Registry 适配](./04.05-nacos适配.md) | `skillclaw/nacos_skill_hub.py` | 586 | ★★★ |
| 04.06 | [Agent 框架适配层](./04.06-代理适配层.md) | `skillclaw/claw_adapter.py` | 1708 | ★★★★ |
| 04.07 | [配置中心](./04.07-配置中心.md) | `skillclaw/config.py` + `config_store.py` | 143 + 546 | ★★ |
| 04.08 | [守护进程与启动](./04.08-守护进程与启动.md) | `cli.py` / `launcher.py` / `setup_wizard.py` | 950 + 213 + 400 | ★★★ |
| 04.09 | [PRM 评分](./04.09-prm评分.md) | `skillclaw/prm_scorer.py` | 222 | ★★ |
| 04.10 | [后台验证](./04.10-后台验证.md) | `validation_worker.py` + `validation_store.py` | 338 + 191 | ★★★ |
| 04.11 | [可视化面板](./04.11-可视化面板.md) | `dashboard_*.py` | 1375 + 726 + 666 | ★★★ |
| 04.12 | [Evolve Server 核心](./04.12-evolve_server核心.md) | `evolve_server/core/` | ~600 | ★★★ |
| 04.13 | [三阶段流水线](./04.13-三阶段流水线.md) | `evolve_server/pipeline/` | ~1500 | ★★★★ |
| 04.14 | [服务端引擎](./04.14-服务端引擎.md) | `evolve_server/engines/` | ~1300 | ★★★★ |

> **注**:server 端除了上表三个 `engines/` / `core/` / `pipeline/` 子包,还有第 4 个子包 `evolve_server/storage/`(`oss_helpers.py` + `mock_bucket.py`),其工具函数(`list_session_keys` / `save_manifest` / `fetch_skill_content` / `save_version_bundle` 等)按需出现在 [04.04 (Skill 共享中心)](./04.04-skill共享中心.md) 的 "service 端 `storage/oss_helpers.py`" 段落,以及 04.12 / 04.13 / 04.14 各章节的"关键文件"表里。`storage/` 子包**无**独立 04.x 模块页。

## 阅读顺序

- **"通读全部"**:04.01 → 04.02 → 04.03 → 04.04 → 04.06 → 04.07 → 04.08 → 04.09 → 04.10 → 04.11 → 04.12 → 04.13 → 04.14 → 04.05(Nacos 特殊)
- **"只读 client"**:跳过 04.12-04.14
- **"只读 server"**:04.12 → 04.13 → 04.14,前 11 个章节只需要看 04.01(api 协议)和 04.04(共享存储)

每章固定结构:**设计动机 → 核心概念 → 实现要点 → 关键文件 → 常见误区**。
