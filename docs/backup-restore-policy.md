# Backup & Restore Policy v1 — Protection Orthogonal to Storage Tiering

**Status:** Accepted architecture decision for Issue #14  
**Date:** 2026-09-05  
**Scope:** Establish the protection and disaster recovery architecture orthogonal to physical placement (Tier 0/1/2). This policy defines protection classifications, failure modes, recovery value tiers, minimal backup policies, and restore verification gates. Exact private file paths, credentials, and user data lists remain strictly isolated in local-only registries (`local/workspace/current-workspace-registry.json`).

---

## 1. 架构定位：保护与存储位置的正交性 (Protection Orthogonal to Placement)

在以往的运维与文件管理中，极易产生一种危险的认知混淆：**“文件只要换了个地方存（或被同步了），它就已经安全了”**。

在经历 Issue #6（存储分层）、Issue #7（工作区架构）以及 Issue #13（应急空间恢复）后，本项目必须确立一个根本原则：

> **“某份数据当前放在哪里 (Placement / Tier)” 与 “这份数据发生损坏、误删、勒索或设备损毁后如何恢复 (Protection / Recovery)” 是两个完全正交的维度。**

```
┌─────────────────────────────────────────────────────────────┐
│ 存储位置维度 (Storage Tiering)                              │
│ • Tier 0 (Internal SSD): Hot Core / Control Plane           │
│ • Tier 1 (Samsung T7): Warm Capacity / Large Working Set    │
│ • Tier 2 (Cloud Archive): Cold / On-demand Archive          │
└──────────────────────────────┬──────────────────────────────┘
                               │  正交解耦 (Orthogonal)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ 数据保护维度 (Protection & Recovery Architecture)           │
│ • UNPROTECTED / REPLICATED / BACKED_UP / VERSIONED          │
│ • OFFDEVICE / OFFSITE / RESTORE_VERIFIED                    │
└─────────────────────────────────────────────────────────────┘
```

### 概念边界与核心语义澄清

为杜绝混淆，本策略对 7 个核心术语做出严格定义：

1. **Working Copy（工作副本）**：用户或应用日常读写、修改的活跃文件实体。
2. **Canonical Copy（权威副本）**：特定资产在逻辑上的**唯一真实来源（Single Source of Truth）**。标记为 `CANONICAL` 仅表示其具备组织与工作流上的权威性，**绝不表示物理上只能存在这唯一一份文件**。
3. **Sync（同步）**：保持两处或多处副本状态一致的自动化通道。**同步不是备份**：误删、逻辑损毁或勒索加密会实时双向传播。
4. **Archive（归档）**：将不再处于高频修改的冷数据移出活跃热层，以释放昂贵空间，保留检索能力。**归档不自动等同于备份**：若原件移入归档后删除，该归档仍然是单点副本。
5. **Backup（备份）**：与工作副本**物理隔离、独立保留、不随工作副本实时变动**而自动覆盖的冗余副本，专为灾难恢复设计。
6. **Version History（版本历史）**：能够在时间线上回退到过去某一特定健康时间点的差异或快照记录，用于防御误改与静默损坏。
7. **Recovery Copy（恢复副本）**：在灾难发生时，经过验证能够真正重新投入生产使用的有效副本。

---

## 2. 保护状态分类词汇表 (Protection Classification Vocabulary)

为 Current Workspace Registry 与资产治理引入轻量且充分的保护状态标识（正交于 `status` 和 `lifecycle_stage`）：

| 保护级别枚举 | 核心定义与保障能力 | 风险特征 |
| :--- | :--- | :--- |
| **`UNPROTECTED`** | 仅有当前 1 份物理副本；无论在主盘还是 T7，一旦介质损坏即彻底灭失。 | **极高危**：单点故障即造成数据永久丢失。 |
| **`REPLICATED`** | 存在第 2 份副本，但缺乏独立介质隔离、版本控制或处于实时同步链条中。 | **中危**：可防简单硬件坏道，但无法防御误删传播或静默损坏。 |
| **`BACKED_UP`** | 存在独立于主工作介质的备份副本，写入后与主工作副本解耦。 | **良好**：具备基本的硬件故障容灾能力。 |
| **`VERSIONED`** | 具备不可变历史版本（如 Git commit、快照增量、带时间戳的归档包）。 | **优良**：可抵御逻辑误删、误覆盖与内容篡改。 |
| **`OFFDEVICE`** | 至少有一份副本存放在完全独立的另一台物理硬件（如外置硬盘或另一台电脑）。 | **抗单机损毁**：主电脑主板/SSD 报废不影响数据生存。 |
| **`OFFSITE`** | 至少有一份副本位于异地网络或不同容灾域（如远程云端），跨物理环境。 | **抗物理灾难**：火灾、失窃、涉水灾害下仍有火种。 |
| **`RESTORE_VERIFIED`** | 该副本已经过实际的端到端“还原与校验演练”，证明数据可读、完整且业务可用。 | **最高确信度**：恢复能力已被事实证明，而非停留在假设中。 |

---

## 3. 数据恢复价值分级 (Data Classification by Recovery Value)

**严禁仅按“文件体积大小”决定备份优先级。** 180MB 的个人笔记与证件价值，远超 100GB 的可重新下载安装包。确立以下 3 级恢复价值体系：

```
                    ┌──────────────────────────────┐
                    │ 1. Irreplaceable / Critical  │ ──► 绝不允许单副本，必须异地+多版
                    ├──────────────────────────────┤
                    │ 2. Reproducible but Costly   │ ──► 轻量保护，恢复成本高需备关键元数据
                    ├──────────────────────────────┤
                    │ 3. Replaceable / Disposable  │ ──► 无需长期备份，坏了直接重构/重下
                    └──────────────────────────────┘
```

### 3.1 绝不可替代 / 极高价值 (Irreplaceable / High Value — CRITICAL)
- **定义**：一旦损坏或丢失，无法通过任何公开渠道、重新编译或花时间重新下载获取的资产。
- **代表类别**：
  - **Personal Knowledge**：`Documents/obsdiannote` 知识库、双链卡片、日记；
  - **Original Research & Literature DB**：Zotero 核心 SQLite 数据库、批注元数据、个人学术大纲；
  - **Personal & Legal Documents**：个人身份证件扫描件、合同协议、教务评审材料、聘任表格；
  - **Unpublished Creative Works & Unique Originals**：个人原创 3D 模型原始工程、未发布的剧本与设计源文件；
  - **Unpushed Active Git Commits / Private Keys**：未 push 到远端的代码分支、本地未提交的修改、SSH 密钥与环境凭据。
- **治理要求**：**绝对不能只有一份！** 必须达到 `BACKED_UP` + `OFFDEVICE`/`OFFSITE`，并逐步具备 `VERSIONED` 与 `RESTORE_VERIFIED`。

### 3.2 可复现但代价高昂 (Reproducible but Costly — IMPORTANT)
- **定义**：理论上能通过重做、从学生/教务处重新索要、或花费数天时间重新整理恢复的数据。
- **代表类别**：
  - **Git Repositories (Committed & Pushed)**：源码已在 GitHub，但本地未纳入 Git 的重型素材、大型虚拟环境配置或 `.env` 配置；
  - **Historical Teaching Courseware**：历年课件合集、学生作业归档（网盘或学校已有分发，但本地检索成本极高）；
  - **Calibre Library**：电子书原书（网络能重新搜寻，但数千本藏书的标签、书目元数据与书签重构繁琐）；
  - **Render Outputs & Asset Packages**：可耗费机器时间重新烘焙/渲染的中间资产包。
- **治理要求**：采用轻量防护；确保远程仓库状态健康，必要时对核心元数据制作独立 snapshot。

### 3.3 可替代 / 即用即弃 (Replaceable — MINIMAL)
- **定义**：可随时从官方服务器下载、重新构建或系统能自动再生的数据。
- **代表类别**：
  - 应用程序安装镜像 (`.dmg`, `.pkg`)；
  - 依赖与编译缓存 (`node_modules`, `.venv`, Homebrew 缓存, Unity SVT/Shader 缓存)；
  - 浏览器临时下载物与过期截屏；
  - Windows 回收站残留 (`$RECYCLE.BIN`)。
- **治理要求**：**明确不列入备份计划**，避免消耗宝贵的备份介质与带宽。

---

## 4. 现状保护审计：狭窄实证盘点 (Current Protection Audit)

基于现场探测，针对当前主要逻辑根的真实保护现状进行审计：

| Logical Root (去隐私化) | Canonical Medium | Current Copies | Independent? | Versioned? | Restore Tested? | Protection Status | Current Risk |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Canonical Obsidian Vault** | Internal SSD (APFS) | 2 份 (本地 + GitHub 远程) | 是 (GitHub 独立) | 是 (Git commit) | 否 (尚未形式化演练) | `BACKED_UP`, `OFFSITE`, `VERSIONED` | **低**。核心笔记全量受 Git 追踪且已推送到远程仓库，状态健康。 |
| **Internal Git Repositories** | Internal SSD (APFS) | 多数 2 份 (本地 + GitHub)；但有 15 个库存在未提交改动 | 部分独立 | 仅 committed 部分 | 否 | `REPLICATED` / `VERSIONED` (部分未提交代码 `UNPROTECTED`) | **中**。源码基本安全，但未提交修改（如 BMAD 58 项、ObsAI 64 项）面临单点丢失风险。 |
| **T7 PROJECTS Root** | Samsung T7 (exFAT) | **仅 1 份** (733GB 集中于 T7) | **否** | **否** | **否** | **`UNPROTECTED`** | **极高危**。T7 硬件损毁或文件系统故障将导致 733GB 商业/学术工程永久灭失！ |
| **T7 Teaching Historical Roots** | Samsung T7 (exFAT) | **仅 1 份** (171GB 集中于 T7) | **否** | **否** | **否** | **`UNPROTECTED`** | **高危**。多年教学多媒体课件与学生资料无第二副本。 |
| **Zotero Core Database** | Internal SSD (APFS) | 1 份活跃库 + 1 份本地 `.bak` | **否** (均在内置 SSD) | 伪版本 (仅本地1天备份) | 否 | **`UNPROTECTED`** (或同盘弱镜像) | **高危**。主盘故障或误删将导致学术图谱与多年批注全灭。 |
| **Calibre Ebook Library** | Internal SSD (APFS) | 1 份 (3.0GB 集中于内置盘) | **否** | **否** | **否** | **`UNPROTECTED`** | **中高危**。图书元数据与个人批注无外部保护。 |
| **Personal & Admin Documents** | 散落在 Desktop / Documents | 1 份 (且无统一目录收敛) | **否** | **否** | **否** | **`UNPROTECTED`** | **极高危**。证件合同裸露，且缺乏异地与外置备份。 |
| **Creative / Loose Media** | Samsung T7 (exFAT) | 1 份 (集中在外置盘) | **否** | **否** | **否** | **`UNPROTECTED`** | **中**。无第二副本，依赖硬件存活。 |

---

## 5. 现有机制与外部介质能力深度分析

### 5.1 Internal SSD
- **定位**：Hot working copy / Control Plane。
- **判定**：**内置 SSD 不是备份介质**。任何将文件从主盘复制到主盘另一文件夹的行为，均属于同介质冗余，无法抵御 SSD 硬件损坏或整机丢失。

### 5.2 Samsung T7 (2TB exFAT)
- **定位**：Warm Capacity / Large Working Set。
- **判定**：**T7-only = One copy = NOT a backup**。
- **边界**：当文件从 Mac 剪切到 T7 并释放内置空间后，该文件不仅没有被“备份”，反而转变成了“T7 单点孤本”。必须彻底纠正“移到移动硬盘等于安全备份”的错觉。

### 5.3 GitHub (Remote Code Hosting)
- **优势**：为 Git 追踪的代码提供了出色的 `OFFDEVICE`、`OFFSITE` 与 `VERSIONED` 保护能力。
- **致命盲区**：
  1. **未提交与未推送内容**：现场实证表明，本地至少 15 个代码库存在未 commit 代码，这些变动对 GitHub 完全不可见；
  2. **`.gitignore` 忽略的文件**：本地 `.env` 密钥配置、私有数据包、权重等不在远程保护范围内；
  3. **非纯代码大资产**：Git 对大型 3D 资产与媒体不友好，绝大多数工程的大素材并未上传。
- **结论**：**Git 远程仓库 ≠ 完整工程备份**。

### 5.4 百度网盘 `/apps/bdpan/`
- **优势**：已在 Issue #10 验证具备 Agent 自动化上传、哈希校验、目录管理与下载能力，具备天然的 `OFFSITE` 异地容灾属性。
- **判定条件（为何当前只能作为 Cold Archive 而非 Complete Backup）**：
  1. **缺乏版本控制与不可变性 (Immutability)**：普通上传容易被覆盖或误删，缺乏防勒索锁定（Object Lock / Versioning）；
  2. **非结构化容灾**：缺乏对权限、软链接、扩展属性（xattr）及 POSIX 元数据的保真封装；
  3. **账号与网络可用性依赖**：受限于 API 配额、网络连通性及第三方账号风控策略。
- **结论**：在当前阶段，`/apps/bdpan/` **非常适合作为重要归档包的“异地火种库（Secondary Cold Archive）”**，但不能作为实时的全机或数据库级灾难恢复系统。

### 5.5 macOS 原生机制现状
- **Time Machine**：实测命令 `tmutil destinationinfo` 返回 `No destinations configured`。**当前本机完全未配置 Time Machine 目标盘**，自动系统快照处于未激活状态。
- **iCloud Drive**：实测显示 iCloud 仅配置了部分苹果原生套件与少量历史目录，**未开启** Desktop & Documents 自动同步，且现有 256GB 存储空间与网络限制也不适合接纳全量同步。

---

## 6. 故障场景推演与容灾矩阵 (Failure Scenarios Analysis)

针对现实中最易发生的 6 大故障场景，进行系统性推演：

| 故障场景 (Failure Scenario) | 哪些数据现在能恢复？ | 从哪里恢复？ | 哪些数据目前绝对恢复不了？(致命断裂点) |
| :--- | :--- | :--- | :--- |
| **Scenario A: 内置 SSD 硬件损坏 / 主板烧毁** | • Obsidian 知识库 (来自 GitHub)<br>• 已 push 的 Git 源码 (来自 GitHub)<br>• T7 上的大项目与素材 (T7 独立存活) | • GitHub 远程仓库<br>• Samsung T7 外置盘 | **• 15个本地 Git 项目的未提交代码**<br>**• Zotero 数据库与批注元数据 (全灭)**<br>**• Calibre 电子书库 (全灭)**<br>**• 桌面与文稿中的个人证件与合同 (全灭)** |
| **Scenario B: Samsung T7 硬件损坏 / 跌落损坏** | • 内置 SSD 上的所有核心代码与笔记依然完好 | • 内置 SSD 活跃副本 | **• 733GB PROJECTS 大项目资产库 (全灭)**<br>**• 171GB 历年教学课件与学生资料 (全灭)**<br>**• T7 上的外置 Unity 编辑器与 34GB 大模型 (需重装重下)** |
| **Scenario C: 逻辑误删或错误同步蔓延** | • Obsidian 笔记 (可通过 Git commit log 回滚) | • Git 版本历史 | **• Zotero 库内文献误删 (无版本回退能力)**<br>**• T7 上的工程误删或误改 (exFAT 无快照回退)**<br>**• 桌面文件误按 Command+Delete 清空** |
| **Scenario D: 文件静默损坏 (Bit Rot / 坏道)** | • 有 Git SHA 校验的代码文件能察觉异常 | • Git remote | **• T7 上存储数年的多媒体视频、3D 贴图与课件包 (exFAT 无法察觉静默位翻转，直到文件无法打开)** |
| **Scenario E: 本地未 push 代码丢失 (误执行 reset 等)** | • 历史已提交且 push 的代码可重新 clone | • GitHub | **• 本地工作区当前修改（如 BMAD-METHOD 58 个修改文件）永久丢失** |
| **Scenario F: 百度网盘账号暂时受限 / 离线** | • 本地所有工作不受影响 (Control Plane 与 T7 均在本地) | • 本地物理介质 | **• 无法从云端获取历史归档包（但在本地无依赖的情况下不会阻塞核心工作流）** |

---

## 7. 最小充分备份策略设计 (Minimal Backup Policy)

不直接追求复杂的企业级“3-2-1”备份系统（当前机器既无 NAS，也无多余的 4TB 硬盘）。基于现场真实设备约束（Mac SSD + T7 SSD + Baidu `/apps/bdpan/` + GitHub），设计**最低充分（Minimal Sufficient）**的弹性策略：

```
┌────────────────────────────────────────────────────────────────────────┐
│ 最小充分保护拓扑 (Minimal Sufficient Protection Topology)              │
├──────────────────────┬──────────────────────┬──────────────────────────┤
│ Critical Core (核资产)│ • Obsidian: GitHub 自动持续同步 (已满足)                │
│                      │ • Zotero: 定期导出 SQLite Snapshot + 归档至 T7/云端      │
│                      │ • Personal/证件: 收敛至专用目录并生成加密压缩包异地备份    │
│                      │ • Git 活跃工程: 建立 git status 每日/每周合规推送习惯     │
├──────────────────────┼──────────────────────┼──────────────────────────┤
│ Important (重要资产) │ • T7 PROJECTS: 针对高价值完成态工程，打包生成 SHA-256     │
│                      │   校验包，按需上传至 `/apps/bdpan/projects/` 作为火种副本 │
│                      │ • Teaching: 历史课件大包生成压缩包上传至 `/apps/bdpan/`   │
├──────────────────────┼──────────────────────┼──────────────────────────┤
│ Replaceable (可替代) │ • 应用、缓存、模型权重、已发布工具：保持单副本，不浪费备份带宽│
└──────────────────────┴──────────────────────┴──────────────────────────┘
```

### 7.1 核心防护规则体系

#### 规则 1：关键数据库与证件的“双介质”原则
- **Zotero 元数据与数据库**：必须定期将 `zotero.sqlite` 生成带时间戳的只读快照包，至少复制一份至 T7，并选择性上传至 `/apps/bdpan/`；
- **个人敏感档案与证件**：收敛至 `~/Documents/Personal/` 后，打包加密存入 T7 专属备份目录，绝不允许仅在桌面裸露单份。

#### 规则 2：Git 代码库的“Committed & Clean”准则
- 只有被 `git commit` 并 `git push` 到远程仓库的代码，才算完成基础异地保护；
- 建立定期排查机制，防止工作区长期滞留数十个未提交文件。

#### 规则 3：T7 重型资产的“冻结火种（Frozen Seed）”策略
- 鉴于 T7 容量达 733GB，无法对其实施廉价的全盘每日镜像；
- **策略**：对已结项的商业工程、个人代表作与历史教学大包，实施**“结项冻结归档”**：
  1. 打包为 `.tar.gz` 或 `.zip`；
  2. 生成 SHA-256 Manifest 清单；
  3. 通过 `baidu-drive` 工具上传至 `/apps/bdpan/`；
  4. 验证哈希无误后，标记为已具备 `OFFSITE` 恢复能力。

#### 规则 4：恢复验证制度 (Restore Verification Gate)
- **未经验证的备份只是愿望，不是容灾。**
- 对所有核心资产，每季度或在重大迁移前，必须执行一次**沙箱恢复测试**：
  - 从 GitHub 全新 clone 知识库并验证双链；
  - 从备份包解压 Zotero 数据库，验证条目数与附件索引是否完整；
  - 从百度网盘下载归档包，执行 SHA-256 校验和对齐。

---

## 8. Current Registry 保护字段扩展方案 (Registry Integration)

将保护与恢复维度正式纳入本地事实注册表 `local/workspace/current-workspace-registry.json`，在每个资产项中扩展以下规范字段：

```json
{
  "id": "internal-zotero-root",
  "domain": "Research & Academic",
  "path": "/Users/yamlam/Zotero",
  "medium": "Internal SSD",
  "role": "Research Literature Library",
  "status": "DEFERRED",
  "lifecycle_stage": "ACTIVE",
  
  "protection": {
    "protection_class": "UNPROTECTED",
    "copy_count": 1,
    "secondary_location": null,
    "versioned": false,
    "off_device": false,
    "off_site": false,
    "restore_status": "UNTESTED",
    "last_backup_verified": null,
    "recovery_priority": "CRITICAL"
  }
}
```

### 关键语义解释
- **`status: CANONICAL` 与 `copy_count`**：
  - `status: CANONICAL` 表达其**权威编辑位置**；
  - `copy_count` 表达**物理副本数量**；
  - 拥有 3 份备份副本完全不影响其唯一的 `CANONICAL` 地位，备份副本不争夺工作权威。
- **根目录与子资产继承隔离**：
  - `T7/PROJECTS` 被标记为 `status: CANONICAL` 且 `protection_class: UNPROTECTED`，仅代表该容器本身的总体状态；
  - 容器内部的具体项目（如某已推送到远端的子开源库）可以单独评估为具备独立保护能力，不被根容器简单粗暴地一刀切。

---

## 9. 真实备份实施待办清单 (Implementation Backlog)

本政策**严格不授权任何真实的数据上传、删除或迁移**。后续涉及真实保护落地的动作，必须单独建立 GitHub 工单并在用户明确授权后推进：

1. **[TASK] 固化本地注册表保护状态扩展**：将 Section 8 定义的 `protection` 模式写入 `local/workspace/current-workspace-registry.json`，完成全量根目录盘点。
2. **[TASK] 排查与收敛本地未提交 Git 代码**：对 BMAD-METHOD、ObsAI 等 15 个存在未提交改动的活跃代码库进行审查与安全推送，消除本地单点丢失风险。
3. **[TASK] Zotero 核心数据库快照与保护原型**：设计免中断的 `zotero.sqlite` 本地快照脚本，建立定期的 T7 与云端火种转存通道。
4. **[TASK] 个人档案与敏感证件安全打包方案**：在 `~/Documents/Personal/` 建立收敛规范，制定加密备份与异地存储机制。
5. **[TASK] T7 高价值项目与历史教学大包冷冻归档清单**：梳理 T7 上具备永久保留价值的完结项目，制定分批次 SHA-256 打包与 `/apps/bdpan/` 上传演习。

---

## 10. 决策结论 (Decision Statement)

**Accepted:**
> 确立数据保护（Protection / Recovery）与存储分层（Storage Tiering）的完全正交性；建立 7 类保护状态与 3 级恢复价值词汇表；确认当前 T7 上的 733GB 项目集、教学课件与主盘 Zotero/个人证件处于单点 `UNPROTECTED` 高风险状态；明确当前条件下的最小充分保护架构与恢复演练规范。
> 
> 本决策不授权进行任何实际数据变动、云端上传或介质格式化。
