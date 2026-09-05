# Target Workspace Architecture v1 — 256GB Mac / Samsung T7 / Cloud Archive

**Status:** Accepted architecture decision for Issue #7 (Revision 1.1 — Evidence-Hardened)  
**Date:** 2026-09-05  
**Scope:** Define user workspace directory hierarchy, domain mapping, file lifecycle heuristics, ingestion flow, and migration backlog. This document does not authorize moving, deleting, or altering user data or applications.

---

## 1. 架构目标与核心原则

### 1.1 核心目标
解决容量受限（256GB 级 SSD）环境下，用户在**创建、下载、编辑、归档**项目与文件时的归属迷茫问题，扭转资产散落在 `Desktop`、`Documents`、`Downloads`、用户 `Home (~)`、外置盘 `T7` 及云端多处的失控现状，建立**“自然知晓其所、存取井然有序、生命周期清晰、迁移成本感知”**的长期可维护工作区结构。

### 1.2 核心原则
1. **关系优先与迁移成本感知（Relationship-First & Migration-Cost-Aware）**：
   - 尊重已形成事实标准且健康运作的路径。对已验证的 Canonical 根目录（如 `~/Documents/obsdiannote` 与 `/Volumes/T7-carllx2T/PROJECTS`）予以确认为架构例外（Architecture Exception），不为了追求形式上的命名对称而进行高成本、高风险的巨额目录重命名或整体搬迁。
2. **职责分明（Strict Separation of Concerns）**：
   - `Downloads` 与 `Desktop` 是**临时收件箱（Inbox）与工作台面**，绝非持久工作区或仓库；
   - 内置 SSD 是**控制平面与热核心（Control Plane / Hot Core）**，严禁堆放大型离线资产；
   - Samsung T7 是**温层扩展与重型工作集（Warm Capacity / Large Working Set）**，而非无限倾倒场；
   - 云端（`/apps/bdpan/`）是**冷归档（Cold / Archive candidate）**与分发渠道；
   - **备份正交性**：独立备份（Backup / Restore）承担独立的数据韧性与灾难恢复职责，归档和同步不等同于备份，云端归档目前不得宣称为完整备份，备份策略由独立课题（Issue #14）决定。
3. **文件系统边界尊重（Filesystem Boundary Awareness）**：
   - 鉴于当前 Samsung T7 格式为 **exFAT**，严格禁止将 POSIX 敏感型工具链（如 Homebrew 主树、Python/Node 虚拟环境、Unix 权限敏感源码）直接迁移至外置盘；
   - 包含密集编译产物（如 `node_modules`、`.venv`、Rust `target`）的活跃代码保留在内置 APFS 容器，重型多媒体数据与数据集外置。
4. **启发式生命周期与显式授权准则（Heuristic Lifecycle & Explicit Authorization）**：
   - 文档中定义的各类时间流转规则均为**默认生命周期启发规则（Default lifecycle heuristics）**，绝非自动触发删除或移动的后台任务（Non-automatic mutation triggers）；
   - 真实的文件迁移必须按项目授权策略（Authorization Policy）推进，建立独立的追踪工单（Execution Ticket），遵循“识别 → 授权 → 校验 → 归档”闭环。

---

## 2. 事实基础与容量基线 (Authority & Evidence)

本文档承接并固化项目已有架构决策与实测证据，明确区分不同阶段的历史度量与最新状态：
- **Issue #2 存储拓扑调查**：内置盘处于临界低容量；`Downloads` 事实演化为多门活跃课程与 Git 项目的“伪工作区”；`~` 根目录下存在数十个游离目录与历史快照。
- **Issue #5 存储余量模型**：内置盘需常态维持 **YELLOW（12–20GiB）** 与 **GREEN（20–25GiB+）** 安全余量。
- **Issue #6 三层存储策略 (`storage-tiering-policy.md`)**：确立 Tier 0 (内置 SSD / Hot)、Tier 1 (T7 / Warm)、Tier 2 (Cloud / Cold) 及 Backup 独立边界。
- **Issue #11 大型应用拓扑 (`large-applications-placement-topology.md`)**：五层解耦模型，确认 Unity Editor/Blender 为 T7 强候选，Creative Cloud 核心授权留内置盘。
- **容量基线区分（明确时间点事实）**：
  - **Issue #13 历史验收基线**：2026-09-05 应急恢复完成时，实测可用空间回升至 **14.403 GiB**，达到 YELLOW 区间并触发 Sufficiency Stop；
  - **最新现场运行时状态**：当前实测 Internal SSD 可用空间约为 **19.97 GiB**（接近 20GiB GREEN 边界），治理重心已彻底转入预防性架构治理。

---

## 3. 真实领域识别与分布现状 (Real Domains & Current Topology)

基于现场探测，用户日常工作与数字资产并非机械对应 macOS 的默认文件夹，而是分布在以下 7 个核心领域：

| 领域分类 (Domain) | 包含内容与工作负载 | 当前散落分布现状 | 事实存在的核心 (De Facto Root) | 结构问题诊断 |
| :--- | :--- | :--- | :--- | :--- |
| **1. Teaching & Courses<br>(教学与教务)** | 当前学期教案、授课课件、学生作业、实验报告、教务申报材料、评审打分表 | `Downloads/` (5GB+ 活跃课程及实验报告)<br>`Desktop/` (打分表、高修、学术周)<br>`Documents/nfu - 教务/`<br>`T7/Courseware_NFU/` (160G)<br>`T7/Courseware_GAFA/` (11G)<br>`T7/作业归档...` (3.3G) | T7 承担了历史海量课件；内置盘无统一教学入口，随手存入 Downloads/Desktop | **极度碎片化**。当前活跃教学与历史教案断联，Downloads 被当作授课工作目录。 |
| **2. Software & Agent Projects<br>(代码与智能体工程)** | Git 仓库、Agent Skills、自动化脚本、研究原型、跨平台工具 | `Documents/GitHub/` (35 个主工程)<br>`~/Projects/`<br>`~/AprilTools/`<br>`~/` 根目录独立工程<br>`Downloads/` (**8 个**带 `.git` 仓库)<br>`T7/PROJECTS/` (733G，含大工程与历史代码) | `~/Documents/GitHub/` 是核心代码区；T7 `PROJECTS/` 是重型工程区 | **边界模糊与代码游离**。轻量代码在 Downloads 随意 clone/init；缺乏区分纯代码与重型数据集项目的标准。 |
| **3. Research & Academic<br>(学术研究与文献库)** | 论文 PDF、Zotero 库、Calibre 图书馆、调研报告、脑图大纲、文献检索导出 | `~/Zotero/` (2.2G)<br>`~/Calibre Library/` (3.0G)<br>`Desktop/` (文献综述、思维导图)<br>`Documents/` (大量散落 .pdf.md5)<br>`Downloads/` (CNKI 导出、论文 PDF)<br>`T7/BOOK/` (925M)<br>`T7/脑电资料合包/` (1.1G) | `~/Zotero/` 与 `~/Calibre Library/` 放置于用户根目录；文献零散分布 | **高价值资产暴露且占主盘**。5.2GB 文献库压在内置盘根目录，论文元数据与正文未规范收敛。 |
| **4. Knowledge Base<br>(Obsidian 个人知识库)** | Markdown 笔记、日记、学术卡片、双链知识网络 | `Documents/obsdiannote/` (已确立 Verified Canonical Vault，187MB)<br>`Documents/ObsAI/` (789MB)<br>`~/obsdiannote-*` (5个历史恢复/暂存副本) | `Documents/obsdiannote/` 已被验证为 Canonical Vault | **历史候选残留**。主库已健康且轻量，但用户 `~` 目录下残留多个演化快照和暂存目录。 |
| **5. 3D, Creative & Media<br>(3D资产、设计与媒体)** | Blender 工程、Maya 场景、Substance 贴图、音频剪辑、影视原片、录像、渲染输出 | `Desktop/` (blend 文件、w9 资产拆解)<br>`Documents/maddalenna...`<br>`T7/PROJECTS/` (3D 雕塑、VR、航拍等)<br>`T7/Movie/` (109G)<br>`T7/教程/` (69G)<br>`T7/林昕真实多媒体归档库/` (8.4G)<br>`T7 根目录` (散落数十GB视频/音频) | T7 是事实上的唯一主力媒体承载介质；内置盘仅作临时渲染/查看 | **容量黑洞外溢至内置盘**。在 Desktop/Documents 随意放置 3D 资产导致主盘骤紧；T7 根目录缺乏收拢。 |
| **6. Administrative & Personal<br>(个人档案与行政文件)** | 学位申请证件、签约合同、人事表格、简历、财务与进修资料 | `Desktop/学位申请证件/` (422MB)<br>`Desktop/~$咨询雅思合同...`<br>`Documents/林昕 - 简历...pdf` | `Desktop/` 承担了高频证件暂存 | **高风险暴露与单点误删隐患**。关键证件与法律合同裸露在桌面上，极易误删且缺乏独立备份保护。 |
| **7. Ingestion & Temporary<br>(临时摄取与交换层)** | 浏览器下载物、聊天附件、临时截屏、安装包镜像 (DMG/ZIP)、过渡文本 | `~/Downloads/` (11GB)<br>`~/Desktop/` (数十张截图、临时脚本、未命名文件) | `~/Downloads` 与 `~/Desktop` | **生命周期缺失**。只进不出，长期驻留并演化成事实上的第二工作区。 |

---

## 4. Target Architecture v1 目录体系拓扑设计

根据 Storage Tiering Policy 及“成本感知”原则，确立三层物理介质上的规范目录树。

### 4.1 Tier 0: 内置 SSD (Hot Core / Control Plane)
保持精炼与高流动性，常态维持 20–25GiB 以上可用空间。

```
/Users/yamlam/
├── .config/, .zshrc, ...                # 标准用户配置文件
├── Applications/                        # 用户特权级轻量应用
├── Desktop/                             # 【临时作业台】仅允许当天活跃临时文件，定期清空
├── Downloads/                           # 【收件箱 Inbox】仅作临时中转，严禁作为长期工程目录
├── Documents/                           # 【核心热数据工作区】
│   ├── GitHub/                          # 核心活跃 Git 代码仓库 (POSIX 敏感、轻量、无巨型二进制)
│   ├── Teaching/                        # 统一教学根目录
│   │   └── Current/                     # 当前学期授课资料、教案、轻量课件、教务表格 (学期末归档至 T7)
│   ├── Research/                        # 学术研究、论文草稿、思维导图、调研汇总
│   ├── obsdiannote/                     # 【架构特例 Architecture Exception】已验证的唯一 Canonical Vault (187MB)，原路径保留！
│   └── Personal/                        # 个人行政、证件、简历、人事归档 (组织收敛目标，非独立备份方案)
└── Library/                             # 操作系统与应用支持文件 (受限治理)
```

> **架构特例说明（Architecture Exception）**：
> - `~/Documents/obsdiannote` 已经作为本机的 Verified Canonical Vault。根据“无收益迁移坚决不作”原则，保持原路径不变，不迁入二级子目录，以规避 Obsidian 内部双链、脚本、快捷方式及第三方同步引用的潜在破坏。

> **内置盘绝对禁放清单**：
> - 任何大于 1GB 的离线视频素材、音视频教程、渲染原片；
> - 任何 3D 资产库（Blender Asset Pack、大型 FBX/OBJ 扫描件）；
> - 历史学期课程归档大包（应在学期结束后迁入 T7）；
> - 预训练模型权重（LLM 权重、Diffusers 权重）；
> - 安装完毕后的 `.dmg` / `.pkg` / `.zip` 安装包；
> - 游离于项目外的冗余虚拟环境或废弃临时解压副本。

---

### 4.2 Tier 1: Samsung T7 (Warm Capacity / Large Working Set)
exFAT 格式外置 SSD，容量 2TB。作为大容量活跃工作区、资产库与重型工程承载盘。

```
/Volumes/T7-carllx2T/
├── PROJECTS/                            # 【大型与混合项目 Canonical Root】保持既有 733GB 目录不变，承载 HealthBridgePrivate 等大型工程
├── Teaching/                            # 【教学历史与重型大包】收敛 Courseware_NFU (160G)、Courseware_GAFA (11G)、学生作业大包
├── Assets/                              # 【共享创作素材库】收敛 3D 扫描模型、STL 打印件、PBR 材质贴图、音频音效
├── Media/                               # 【多媒体影视与教程库】收敛原 Movie/ (109G)、教程/ (69G)、演示录像
├── llm/                                 # 【大模型与机器学习资源】保持既有模型权重与环境目录 (34G)
├── Applications/                        # 【自包含外置应用】保持既有目录名，承载 Unity Editors 与便携工具 (41G)
└── Archives/                            # 【本地冻结归档包】压缩打包的历史旧项目 (.tar.gz / .zip)
```

> **T7 准入与治理规则**：
> - **保持已有根目录稳定性**：已有 `T7/PROJECTS`（733GB）与 `T7/Applications`（41GB）保持原名称，不进行大规模重命名；
> - **坚决清理 T7 根目录零散文件**：禁止在 T7 根目录随意堆放音视频与大文件，逐步分流收敛至 `Media/`、`Assets/` 与 `Teaching/`；
> - **exFAT 约束**：禁止通过 symlink 将内置盘的 Node/Python 系统环境硬链接至 T7；大工程源码与资产置于 T7，依赖管理遵循应用原生配置。

---

### 4.3 Tier 2: Cloud Archive (Cold / Archive Candidate)
以百度网盘 `/apps/bdpan/` 为 Agent 授权归档区，作为按需冷存储候选：

```
/apps/bdpan/ (Baidu Netdisk Agent Archive Zone)
├── teaching/                            # 已结课超过 1 年以上的课程与学生作业完整大包
├── projects/                            # 商业/学术历史工程冷冻包 (附带 SHA-256 校验和)
└── publications/                        # 个人已发表论文、学术专著、原始实证数据归档
```

> **备份边界声明（Backup Boundary Statement）**：
> - `/apps/bdpan/` 属于**冷存储/归档候选层（Cold / Archive Candidate）**，当前**不得被宣称为完整备份系统**；
> - 完整备份必须具备防篡改、多版本历史、跨介质灾备与可实证恢复能力，高价值数据的 Backup / Restore Policy 由独立课题（Issue #14）设计。

---

## 5. Downloads 与 Desktop 摄取层治理机制 (Ingestion Lifecycle)

用户空间频频告急的根源在于：`Downloads` 和 `Desktop` 扮演了“永久工作区”，文件“只进不出”。确立以下**默认生命周期启发规则（Default lifecycle heuristics）**：

```
[ Inflow (下载/截图/AirDrop) ]
            │
            ▼
┌───────────────────────────────┐
│     Inbox (Downloads/Desktop) │
└───────────────────────────────┘
            │
      审阅分流 (Review)
            │
    ┌───────┴───────────────────────────────┐
    ▼                                       ▼
【即用即弃】 (Ephemeral)              【沉淀资产】 (Durable)
• 安装完毕的 .dmg / .pkg             • 代码工程 → ~/Documents/GitHub/
• 临时解压的临时包                    • 课程课件 → ~/Documents/Teaching/Current/
• 随手截屏 (已粘贴)                   • 3D/视频资产 → /Volumes/T7-carllx2T/
• 临时查看的草稿文档                  • 个人档案 → ~/Documents/Personal/
    │                                       │
    ▼                                       ▼
  清理 (Trash / 手动确认)              进入规范目录工作
```

### 5.1 Downloads 启发规则 (Heuristics)
1. **严禁在 `Downloads/` 内常驻项目或执行 `git init`**：
   - 临时 clone 的开源库，试用后若需二次开发，**应及时剪切至 `~/Documents/GitHub/`**；若不再使用，应及时删除。
2. **教学材料与课程包禁止直接在 `Downloads/` 中展开教学**：
   - 收到教务处或学生打包文件，解压后第一时间移入 `~/Documents/Teaching/Current/`，原始压缩包清空。
3. **DMG / 安装包默认周期启发**：
   - 软件安装完成并验证可用后，安装包原则上 24 小时内移入废纸篓（非自动静默删除，作为用户清理提示）。

### 5.2 Desktop 桌面启发规则 (Heuristics)
1. **桌面仅作为“单日作业台”**：下班或关机前，桌面除少数系统快捷方式外，原则上鼓励保持零临时文件。
2. **系统截屏（Screenshots）治理**：默认截屏定期归集或清理，避免数十张 png 长期铺满桌面。
3. **个人档案安全收敛**：`Desktop/学位申请证件/` 等重要个人资料，应收敛至 `~/Documents/Personal/` 统一管理，并等待 Issue #14 确立备份机制。

---

## 6. 项目全生命周期流转 (Project Lifecycle Heuristics)

定义项目在不同阶段的物理位置流转启发规则（非自动突变触发器，须经用户确认执行）：

```
[ 诞生 (Creation) ]
  ├─ 轻量/纯代码/日常文档 ─────────► [ Tier 0: 内置 SSD ] (Active)
  └─ 重型 3D/视音频/巨型工程 ───────► [ Tier 1: Samsung T7 / PROJECTS ] (Active/Warm)
                                          │
                        项目结束 / 启发阈值: 30/90 天未修改
                                          │
                                          ▼
                                 [ Tier 1: Samsung T7 ] (Warm Storage)
                                 • 结课教案归入 T7/Teaching/
                                 • 旧代码工程移入 T7/PROJECTS/
                                 • 内置盘清理依赖 (.venv/node_modules)
                                          │
                                结项超过 1 年 / 极低频访问
                                          │
                                          ▼
                                 [ Tier 2: Cloud Cold Archive ]
                                 • 本地生成 tar.gz 打包
                                 • 生成 SHA-256 Manifest
                                 • 移交 /apps/bdpan/，释放本地空间
```

### 6.1 各阶段定义与启发阈值
- **Active（活跃期）**：
  - 定义：当前正在频繁编辑的课程、代码或论文。
  - 存放：轻量项目放内置盘 `~/Documents/` 下对应子目录；重量级项目放 `T7/PROJECTS/`。
- **Warm（温存期 - 默认启发：30/90 天无修改）**：
  - 定义：学期结束的结课课程、已完成主要开发进入维护期的软件工程。
  - 动作：从内置盘迁移至 T7 对应归档目录；若项目留存内置盘，应冻结 lockfile 并清理庞大的 `node_modules` 或 `.venv` 缓存。
- **Cold（冷冻期 - 默认启发：1 年以上未触碰）**：
  - 定义：往年完成的商业项目、历史毕业生材料、过往学术成果。
  - 动作：打包压缩，计算哈希校验值，上传至百度网盘 `/apps/bdpan/` 冷区；T7 本地可仅保留只读压缩副本或在空间紧张时下线。
- **Retire（退役/销毁）**：
  - 定义：临时构建中间件、废弃无价值的测试环境副本、已被证明无用的重复 snapshot。
  - 动作：在留存基线证据并取得用户显式确认后，从磁盘清除。

---

## 7. 现状对比与迁移差距分析 (Migration Gap Analysis)

本章对现场状态与 Target 架构进行严格差距比对，形成后续治理的迁移清单（Backlog）。**本轮只建立规划清单，不执行任何实际迁移。**

### 7.1 差距对照与推荐动作清单

| 当前资产类别 | 当前物理位置 | 目标规范位置 (Target) | 核心问题 / 隐患 | 推荐动作 (Recommendation) | 风险定级 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **活跃与历史课程混合** | `~/Downloads/` (多门课程及实验报告)<br>`~/Desktop/22高修/` 等 | `~/Documents/Teaching/Current/`<br>`/Volumes/T7-carllx2T/Teaching/` | 授课目录与下载杂物混合；历史课件长期占用 Downloads 空间 | **结构整理**：本学期活跃课程移入 Documents/Teaching/Current；往期报告与作业移入 T7/Teaching | `REVIEW FIRST` |
| **游离 Git 仓库** | `~/Downloads/` (**8 个工程**)<br>`~/Projects/`、`~/AprilTools/` | `~/Documents/GitHub/`<br>或 `T7/PROJECTS/` | 随时面临下载清理被误删风险；开发环境散落无法统一治理 | **仓库归位**：排查未提交代码后，轻量工程归入 GitHub/，重型或历史工程归入 T7/PROJECTS | `REVIEW FIRST` |
| **个人高价值档案与证件** | `~/Desktop/学位申请证件/`<br>`Desktop/~$咨询雅思合同...` | `~/Documents/Personal/` (待 #14 规划备份) | 裸露在桌面台面，单点误删风险高（单纯移入目录不等于具备备份） | **安全收敛**：移入 Personal 统一目录；真实异地备份与恢复等待 Issue #14 决策 | `HIGH RISK` |
| **3D 工程与临时渲染散件** | `~/Desktop/*.blend`<br>`Desktop/w9-digital-sculpt/` | `/Volumes/T7-carllx2T/PROJECTS/` | 3D 资产动辄数百兆，占用紧张的内置 SSD 缓冲空间 | **外部迁移**：移入 T7/PROJECTS/ 对应子目录，在应用中重新关联资产路径 | `LOW RISK` |
| **桌面日常截屏堆积** | `~/Desktop/Screenshot *.png` (数十张) | 移入废纸篓 或 `~/Pictures/Screenshots/` | 桌面极度混乱，占用桌面渲染与索引资源 | **批处理清理**：审查无用临时截图批量移入废纸篓，需长期保留的重命名归集 | `LOW RISK` |
| **Home 根目录幽灵 node_modules** | `~/node_modules` (4.1KB 索引，多项依赖) | 废纸篓删除 (Trash) | 过去在 ~ 误执行 npm install 产生，污染根目录命名空间 | **依赖剥离**：确认根目录无全局 package 依赖后清除残留 node_modules | `LOW RISK` |
| **已退役 Obsidian 快照残留** | `~/obsdiannote-*` (5 个历史测试目录) | 外部备份介质 或 彻底清理 (Retire) | Issue #13 已退役且已确认 Canonical vault，但历史暂存仍滞留内置盘 | **快照退役**：核对历史快照无新变动后，移至 T7 或清除释放空间 | `LOW RISK` |
| **Zotero & Calibre 知识库** | `~/Zotero` (2.2GB)<br>`~/Calibre Library` (3.0GB) | 保持内置盘或评估数据目录外置配置 | 占用 5.2GB 内置空间；直接裸露在用户根目录 | **路径评估**：作为独立课题评估其附件目录外置可行性，核心数据库保留内置 | `REVIEW FIRST` |
| **T7 根目录散落音视频与文档** | `/Volumes/T7-carllx2T/` 根目录散落视频、音频、PDF (数十GB) | `/Volumes/T7-carllx2T/Media/` 及 `Teaching/` | T7 根目录变成第二个垃圾桶，难以按类别管理与备份 | **分类收敛**：在 T7 内部进行物理移动，分流至 Media、Teaching 和 Assets 目录 | `LOW RISK` |
| **T7 Windows 回收站残留** | `/Volumes/T7-carllx2T/$RECYCLE.BIN` (12GB) | 内容核实后删除 (Purge) | 12GB 可释放潜力；但可能包含历史可恢复文件，必须内容级确认后方可清除 | **审阅清除**：检查其内容，经用户明确授权后清除 | `REVIEW FIRST` |

---

## 8. 现状最大结构问题与治理路线图

### 8.1 当前最大的 4 个结构性问题
1. **Downloads 伪工作区化（The Downloads-as-Workspace Anti-pattern）**：
   - 现场实测发现 `~/Downloads`（11GB）中存在多达 **8 个带 `.git` 的代码仓库**（`carllx-skills`, `practices-personal-portfolio-website - 副本`, `agent-project-system`, `2025-2026-2 课程`, `22 毕设创作`, `教务材料`, `doubao-tts-api`, `doubao-web-bridge`），以及多门活跃课程。这是导致空间黑洞无法受控、随时有误删风险的头号架构顽疾。
2. **教学资产跨介质断代撕裂（Bifurcated Teaching Assets）**：
   - 当前学期资产在内置盘的 Downloads/Desktop 随手堆积，历史课件（170GB+）堆在 T7。缺乏统一的 `Teaching` 领域根目录与学期结课归档流转。
3. **高价值个人档案与证件单点暴露（Single-Point Vulnerability of Critical Documents）**：
   - `学位申请证件`、咨询合同直接放于 Desktop，未纳入规范目录，也无独立备份机制。
4. **根目录无序蔓延（Pollution of User Home & T7 Root）**：
   - 用户 `~` 目录下存在 30+ 游离目录（包括直接安装的 `node_modules`、废弃 Obsidian 暂存）；T7 根目录下散落巨型未分类媒体文件与 12GB Windows 回收站残留。

### 8.2 推荐迁移顺序（分波次推进路线图）
- **Wave 0: 低阻力安全收敛（Quick Wins）**：
  - 在 `~/Documents/` 建立标准目录骨干：`Teaching/`、`Research/`、`Personal/`；
  - 确认 `~/Documents/obsdiannote` 架构特例地位，不做位置变更；
  - 将 `Desktop/学位申请证件/` 及个人合同收敛至 `~/Documents/Personal/`（真实备份策略交由 #14）；
  - 批量审查并清空 `Desktop` 上数十张过期截图；
  - 清理 `~` 根目录下误装的无主 `node_modules`。
- **Wave 1: 解救 Downloads 核心（Inbox Restoration）**：
  - 针对 `~/Downloads` 中 8 个 Git 仓库进行状态审查，确保无未提交更改后分别迁入 `~/Documents/GitHub/` 或 `T7/PROJECTS/`；
  - 将当前活跃教学目录移入 `~/Documents/Teaching/Current/`；
  - 将安装完毕的 `.dmg` 彻底清退。
- **Wave 2: T7 内部规范整理（T7 Structure Normalization）**：
  - 维持 `T7/PROJECTS` 与 `T7/Applications` 根目录名称；
  - 审查 T7 上 12GB 的 `$RECYCLE.BIN` 内容，取得用户授权后清除；
  - 将 T7 根目录散落的音视频分流移入 `Media/` 与 `Assets/`。
- **Wave 3: 知识库与重型依赖专项（Specialized Deep Dives）**：
  - 彻底退役清理 `~` 根目录下 5 个已废弃的 `obsdiannote-*` 历史测试暂存；
  - 启动 Zotero / Calibre 存储与附件外置可行性评估。

### 8.3 推荐建立的后续 GitHub 跟踪工单
1. **[TASK] 治理 Downloads 伪工作区：迁移活跃教学目录与 8 个 Git 仓库至规范路径**
2. **[TASK] 治理 Desktop 与 Home 根目录游离资产：收敛敏感证件至 Personal 并清除误装 node_modules**
3. **[TASK] 整理 Samsung T7 外部存储结构：核实清理 $RECYCLE.BIN 并归集根目录散落媒体**
4. **[TASK] 彻底退役清理 Home 根目录残留的 5 个 Obsidian 历史迁移暂存目录**
5. **[RESEARCH] 评估 Zotero 与 Calibre 文献库的存储布局与附件外置可行性**

### 8.4 现阶段绝对不可动边界（Guardrails & Non-goals）
为保证系统稳定与零数据风险，以下项目在本轮及后续常规整理中**严格禁止盲目改动**：
- **禁止在没有单个仓库状态核实前批量移动 Downloads 中的 Git 目录**（防范未 commit / 未 push 提交丢失）；
- **禁止改动当前已稳定运作的 `Documents/obsdiannote` Canonical Vault**；
- **禁止移动或清理 Zotero / Calibre 核心数据库**（等待 Issue #14 备份策略建立）；
- **禁止在未核实内容且未获得用户显式授权前清空 T7 的 `$RECYCLE.BIN`**；
- **禁止在没有应用自身设置支持的情况下跨卷硬链接应用程序资产**；
- **禁止在当前决策轮次执行任何真实文件系统的删除或批量重命名（保持 Zero Mutations）**。
