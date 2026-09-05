# Current Workspace Topology & Classification (As-Is Architecture)

**Status:** Accepted architecture specification for Issue #7 Addendum  
**Date:** 2026-09-05  
**Scope:** Public logical mapping of current (As-Is) workspace roots, canonical classification vocabulary, and migration status. Exact private filesystem paths and user inventory are strictly isolated in local-only registries (`local/workspace/`).

---

## 1. 架构定位与双地图模型 (Dual-Map Model)

在资源受限 Mac 的长期治理中，仅规划理想的未来状态（Target Architecture）会导致后续 Agent 在面对真实物理文件时缺乏判断上下文。本项目正式确立 **`Current State → Classification → Target State`** 的双地图模型：

```
┌───────────────────────────────┐
│ Current Workspace Topology    │  (现状拓扑与实证根目录地图)
└──────────────┬────────────────┘
               │  查询状态分类 (Status Classification)
               ▼
┌───────────────────────────────┐
│ Classification Decision       │  (CANONICAL / ACCEPTED / TRANSITIONAL / ...)
└──────────────┬────────────────┘
               │  根据状态决定处置策略 (Policy & Action Gate)
               ▼
┌───────────────────────────────┐
│ Target Workspace Architecture │  (目标治理规范与归宿)
└───────────────────────────────┘
```

> **核心治理准则**：
> - **现状优先**：对任何已有资产执行整理、迁移或清理前，必须先查阅现状拓扑与分类；
> - **零无收益迁移**：标定为 `CANONICAL` 或 `ACCEPTED_EXISTING` 的路径默认原位保持，禁止为了命名对称制造搬迁成本；
> - **动态流转范围**：只有标记为 `TRANSITIONAL`、`INBOX / TRANSIENT`、`LEGACY / REVIEW` 的资产才进入迁移与归宿流程。

---

## 2. 状态分类词汇表 (Classification Status Vocabulary)

所有用户目录、存储根与工作区资产在分类时，统一使用以下 7 个标准状态枚举：

| 状态标识 (Status) | 定义与治理语义 | 默认处置策略 (Default Policy) |
| :--- | :--- | :--- |
| **`CANONICAL`** | 已实证确认的系统权威位置，承载核心活跃工作流，具备唯一真实来源属性。 | **保持原位（KEEP）**。默认严禁为了目录整齐或美观进行重构或迁移。 |
| **`ACCEPTED_EXISTING`** | 当前路径或名称虽非最理想设计，但已承载大量既有数据，重构迁移成本过高且缺乏明显净收益。 | **保持现状（KEEP AS-IS）**。承认其事实有效性，避免大范围变动。 |
| **`TRANSITIONAL`** | 当前处于有效使用中，但由于空间占用或生命周期已过，在 Target Architecture 中已定义了更合理的归宿。 | **计划流转（PLAN RELOCATION）**。列入迁移 backlog，在后续独立工单中渐进迁移。 |
| **`INBOX / TRANSIENT`** | 临时接收、解压、查看与中转的摄取层（如 Downloads / Desktop）。 | **定期清空（TRANSIENT INBOX）**。不得被认作长期工作区，遵循收拢→分流启发规则。 |
| **`DEFERRED`** | 涉及底层复杂数据库、专用应用绑定或跨介质同步机制，当前决策条件不成熟的领域。 | **暂缓变动（DEFER）**。留待独立专题（如 Issue #4 开发环境、Issue #14 备份）研究。 |
| **`LEGACY / REVIEW`** | 历史遗留目录、历史测试暂存、已退役的快照副本或系统残留。 | **复核确认（REVIEW FIRST）**。须完成内容核实并取得显式授权后，方可退役或清理。 |
| **`RETIRE`** | 经因果验证已完全失效、无独立参考价值且具备可替代证据的废弃目录。 | **授权退役（RETIRE & PURGE）**。在留存必要基线后执行物理清理。 |

---

## 3. 现状权威与接受根目录逻辑映射 (Current Accepted Roots)

以下为当前经过现场实证确立的逻辑存储根与分类映射（去隐私化表述）：

| 逻辑根实体 (Logical Root) | 领域 (Domain) | 物理介质 (Medium) | 承担角色 (Role) | 分类状态 (Status) | 保持理由 (Why Kept) | 与 Target 的映射关系 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Internal Active Git Root** | Software & Agents | Internal SSD (APFS) | 核心活跃代码与控制平面 | `CANONICAL` | 承载 35 个活跃代码库与 Agent 项目，具有强 POSIX 敏感性与高频修改需求。 | 对应 `Target: ~/Documents/GitHub/`，保持核心地位不变。 |
| **Canonical Obsidian Vault** | Knowledge Base | Internal SSD (APFS) | 唯一真实笔记与双链库 | `CANONICAL` | 经完整审计健康的唯一知识库，原位保持以保护双链拓扑与脚本稳定性（架构特例）。 | 对应 `Target: obsdiannote` 原位特例保留，不进行二级迁移。 |
| **T7 Large-Project Root** | Creative & Projects | Samsung T7 (exFAT) | 重型大项目与混合工程根 | `CANONICAL` | 承载 733GB 规模的历史与大项目资产（含已外置工程），无收益改名成本过高。 | 对应 `Target: T7/PROJECTS/` 保持原名不变，后续仅做内部梳理。 |
| **T7 Applications Root** | Applications | Samsung T7 (exFAT) | 外置与便携应用程序根 | `ACCEPTED_EXISTING` | 承载外置 Unity Editor 与模块，运行良好，无须为追求命名改名。 | 对应 `Target: T7/Applications/` 保持原名与外置应用职责。 |
| **T7 AI/Model Root** | AI & Machine Learning | Samsung T7 (exFAT) | 本地大模型权重与环境 | `ACCEPTED_EXISTING` | 34GB 本地模型权重，exFAT 承载良好，避免挤占主盘。 | 对应 `Target: T7/llm/` 保持现状，等待模型专项治理。 |
| **Downloads Inbox** | Ingestion & Temp | Internal SSD (APFS) | 浏览器与系统临时下载接收 | `INBOX / TRANSIENT` | 系统默认下载落地层，不可避免。 | 对应 `Target: Downloads (Inbox)`，须剥离游离代码与活跃授课。 |
| **Desktop Workbench** | Ingestion & Temp | Internal SSD (APFS) | 单日即时作业台面与临时查看 | `INBOX / TRANSIENT` | 满足用户日常随手截屏与即时对照。 | 对应 `Target: Desktop`，须剥离高价值档案与 3D 资产。 |
| **Teaching Historical Roots** | Teaching & Courses | Samsung T7 (exFAT) | 历史大型多媒体课件库 | `TRANSITIONAL` | 积累了 171GB+ 往届历史课件，但缺乏统一架构归集。 | 对应 `Target: T7/Teaching/`，后续规划平滑收拢。 |
| **Zotero / Calibre Roots** | Research & Books | Internal SSD (APFS) | 学术文献库与电子图书库 | `DEFERRED` | 包含复杂 SQLite 数据库与上万条附件，涉及核心知识资产。 | 对应 `Target: DEFERRED`，等待备份与附件外置专项评估。 |
| **Legacy Staging Zones** | System & Staging | Internal SSD (APFS) | 历史重构暂存与测试目录 | `LEGACY / REVIEW` | 历史治理中生成的过渡测试副本，主库稳定后属于候选退役项。 | 对应 `Target: RETIRE`，核实无差异后清理。 |
| **T7 Residual Recycle Bin** | System Residuals | Samsung T7 (exFAT) | 历史 Windows 回收站残留 | `LEGACY / REVIEW` | 占有 12GB 空间，但可能包含用户历史数据。 | 对应 `Target: PURGE`，经内容确认与用户授权后释放。 |

---

## 4. 本地精确事实注册表 (Local Exact Registry)

为在公开版本库中彻底杜绝用户隐私路径与敏感文件名的泄露，本项目采用**逻辑公开 + 本地精确隔离**机制：

- **公共规范**：本文档仅记录逻辑名称、领域划分与架构决策；
- **本地精确注册表**：真实绝对路径、详细体积数据、文件哈希与私人目录清单由本地专属注册表维护：
  ```
  local/workspace/current-workspace-registry.json
  ```
- **配置保证**：该文件受根目录 `.gitignore` 全面覆盖，物理隔离于任何 git commit / push 之外。
- **Agent 执行准则**：任何执行具体分类、分析或生成迁移候选的 Agent，必须优先读取本地 `local/workspace/current-workspace-registry.json` 获取精确路径事实，不得凭空猜测或直接对未注册目录实施批量变更。
