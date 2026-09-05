# Backup & Restore Policy v1 — Protection Orthogonal to Storage Tiering

**Status:** Accepted architecture decision for Issue #14 (Revision 1.1 — Rigorous & Privacy-Sanitized)  
**Date:** 2026-09-05  
**Scope:** Establish the protection and disaster recovery architecture orthogonal to physical placement (Tier 0/1/2). This policy defines decomposed protection schemas, failure modes, recovery value tiers, minimal backup policies, secrets boundaries, and restore verification gates. Exact private file paths, repo names, credentials, and user data lists remain strictly isolated in local-only registries (`local/workspace/current-workspace-registry.json`).

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
│ • Base state: UNPROTECTED | REPLICATED | BACKED_UP          │
│ • Capabilities: versioned | off_device | offsite            │
│ • Evidence: restore_status | last_restore_verified          │
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

## 2. 保护模型解耦与分类词汇表 (Decomposed Protection Schema)

保护状态不再被设计为单一互斥枚举，而是拆解为三个正交的子维度：**基础保护状态 (Base State)**、**保护能力特征 (Capabilities)** 与 **恢复验证实证 (Restore Evidence)**。

### 2.1 基础保护状态 (Base Protection State)
* **`UNPROTECTED`**：当前仅有 1 份有效物理副本；无论在主盘还是 T7，介质损坏即导致永久灭失。
* **`REPLICATED`**：存在第 2 份副本，但缺乏独立物理隔离、无版本历史、或属于双向实时同步链路。
* **`BACKED_UP`**：存在独立于主工作介质的备份副本，写入后与工作副本解耦，且具备灾难恢复用途。

### 2.2 保护能力特征 (Protection Capabilities)
* **`versioned` (bool)**：是否具备不可变历史版本能力（如 Git commit、快照增量、带时间戳的归档），用于防御逻辑误删与内容篡改。
* **`off_device` (bool)**：是否至少有 1 份副本存放在完全独立的另一台物理硬件介质上（抗单机硬件故障）。
* **`offsite` (bool)**：是否至少有 1 份副本位于异地网络或不同的物理容灾域（抗设备被盗、火灾等物理损毁）。

### 2.3 恢复验证实证 (Restore Evidence & Audit)
* **`restore_status`**：`UNTESTED`（未经验证）或 `VERIFIED`（已实际验证过还原可用）。
* **`last_restore_verified` (date|null)**：上一次实际执行端到端恢复演练的日期；未经验证时为 `null`。
* **`last_protection_audited` (date)**：上一次对该资产保护状态进行只读核查审计的日期。**只读审计绝不等于恢复验证**。

---

## 3. 根容器聚合 vs 子资产独立分类原则 (Root Aggregate vs Child Assets)

为避免出现“某个顶层目录很大，就必须全量备份整个目录”的机械推论，确立以下分层治理原则：

1. **容器聚合状态 (Aggregate Representation)**：
   - 类似 `T7/PROJECTS`（733GB）、`T7/Courseware_*`（171GB）或 `Documents/GitHub`（35个代码库）属于多资产根容器；
   - 顶层根目录的保护状态标记为 **`aggregate_protection: MIXED / UNPROTECTED DEFAULT`**，表明该容器缺乏统一完备的整根灾备；
   - **绝不代表该容器下的全部数百 GB 都是 Critical 或必须全部上传云端**。
2. **状态不向下自动继承**：
   - Root 标记为 `status: CANONICAL` 仅表示该根目录是工作区的权威物理容器，其内部子目录/子项目不自动继承相同的保护级别；
   - 具体资产必须按自身的生命周期、使用状态与恢复价值单独建立记录。
3. **按子资产恢复价值按需建档**：
   - 后续仅对真正具备高恢复价值的具体完结项目、代表作或核心工程建立精确注册项，实施按需备份。

---

## 4. 数据恢复价值分级 (Data Classification by Recovery Value)

**严禁仅按“文件体积大小”决定备份优先级。** 确立以下 3 级恢复价值体系：

### 4.1 绝不可替代 / 极高价值 (Irreplaceable / High Value — CRITICAL)
- **定义**：一旦损坏或丢失，无法通过任何公开渠道、重新编译或花费机器时间重新下载获取的资产。
- **代表类别**：
  - **Personal Knowledge**：`Documents/obsdiannote` 个人知识库与笔记网络；
  - **Academic Literature & Annotations**：Zotero 核心数据目录（含 SQLite 数据库、批注与本地唯一附件）；
  - **Personal & Legal Documents**：个人身份证件扫描件、签约合同协议、人事档案表格；
  - **Unpublished Creative Works & Unique Originals**：个人独创 3D 模型原始源文件、未发布的工程创作；
  - **Unpushed Active Git Branches**：活跃项目中尚未推送到远端仓库的本地工作成果。
- **治理目标**：**不能只有一份**；必须逐步达到 `BACKED_UP` + `off_device`/`offsite`，并落实恢复验证。

### 4.2 可复现但代价高昂 (Reproducible but Costly — IMPORTANT)
- **定义**：理论上能通过重做、向合作方重新索要、或花费大量人工重新整理恢复的数据。
- **代表类别**：
  - **Git Repositories (Committed & Pushed)**：已全量推送到 GitHub 的源码（但需注意排除的重型配置）；
  - **Historical Teaching Courseware**：历年教学课件与往届学生作业（已结课，多媒体体积极大）；
  - **Calibre Library**：电子书原书与藏书库元数据；
  - **Render Outputs & Asset Packages**：可花费机器算力重新烘焙/渲染的中间资产包。
- **治理目标**：轻量保护，定期制作冷冻快照或火种副本。

### 4.3 可替代 / 即用即弃 (Replaceable — MINIMAL)
- **定义**：可随时从官方或镜像站重新下载、重新构建或系统能自动再生的数据。
- **代表类别**：
  - 应用程序安装包镜像 (`.dmg`, `.pkg`)；
  - 依赖与编译缓存 (`node_modules`, `.venv`, Homebrew 缓存, Unity SVT/Shader 缓存)；
  - 临时下载物、已使用截屏、Windows 回收站残留 (`$RECYCLE.BIN`)。
- **治理目标**：**明确不列入备份计划**，避免浪费外置介质容量与网络带宽。

---

## 5. 敏感凭据与机密边界 (Secrets Boundary)

在以往的备份实践中，将代码目录或桌面全部“打包上传”极易导致安全灾难。本项目严格划定 **`SECRET_MATERIAL`** 边界：

### 5.1 涵盖范围
* SSH 私钥 (`~/.ssh/id_*`)；
* 项目环境变量与配置文件中的私钥 (`.env`, `.env.local`)；
* API Tokens、云平台访问密钥、第三方服务凭据；
* 个人银行、财务与敏感法律凭证。

### 5.2 核心安全防护规则
1. **绝对禁放普通归档包**：严禁将上述机密材料自动打入普通的 T7 明文工程包或未加密的百度网盘 `/apps/bdpan/` 归档包中；
2. **严禁在公开文档暴露真实密钥**：公共政策与版本库中只能出现抽象定义，不得留存任何真实凭证信息；
3. **独立安全恢复机制**：机密材料的容灾必须采用受信任的独立加密机制（如 GPG 加密包、密码管理器或 macOS 原生加密钥匙串）；
4. **显式授权门禁**：任何针对秘密材料的备份、加密或轮换行为，必须单独建立安全工单，经用户显式批准后方可执行。

---

## 6. 现场实证审计与机制能力分析 (Current Mechanisms Audit)

### 6.1 现有物理与云端机制事实判定
* **Internal SSD**：Hot working copy / Control Plane，**不是备份介质**。同盘复制不能抵御主控损坏或整机损毁。
* **Samsung T7 (exFAT)**：Warm Capacity，**T7-only = One copy = NOT a backup**。文件从 Mac 移至 T7 并删除原件后，属于单点孤本。
* **GitHub (Remote Code Hosting)**：
  - 覆盖范围仅限于 **已 commit 并已 push 的受追踪源码** (`coverage_scope: tracked_committed_pushed_content_only`)；
  - 现场审计证实：多个活跃代码库存在未 commit 本地修改，且被 `.gitignore` 忽略的文件（私有配置、本地数据）对 GitHub 完全不可见；**Git Remote 不等同于整项目灾备**。
* **百度网盘 `/apps/bdpan/`**：
  - 具备 Agent 自动化上传与 SHA-256 验证能力，具备天然异地属性；
  - 但缺乏对象版本锁定、不支持 POSIX 权限保真封装，目前定性为 **“重要归档包的异地冷冻火种库 (Secondary Cold Archive)”**，而非全自动灾备系统。
* **macOS Time Machine 事实**：
  - 命令 `tmutil destinationinfo` 返回 `No destinations configured`（未配置备份目标盘）；
  - 命令 `tmutil listlocalsnapshots /` 当前仅观察到系统更新相关快照，未观察到 Time Machine 用户数据本地快照。当前又未配置 Time Machine backup destination，因此 Backup / Restore Policy 不依赖本地 snapshots 作为有效灾备层。
* **macOS iCloud 事实**：
  - 目录仅有部分应用沙盒，命令行证据**无法证明已开启 Desktop & Documents 同步**（标记为 `Not verified / no evidence of Desktop & Documents coverage`），不做无证据猜测。

### 6.2 现状逻辑根保护矩阵 (去隐私化)

| Logical Root (去隐私化) | Canonical Medium | Base State | Capabilities (Ver/OffDev/OffSite) | Restore Evidence | Protection Coverage / Risk |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Canonical Obsidian Vault** | Internal SSD | `BACKED_UP` | `[T, T, T]` | `UNTESTED` (Audited: 2026-09-05) | 核心笔记全量已推送到 GitHub 远端；状态健康。 |
| **Internal Git Repositories Root** | Internal SSD | `MIXED` | `[T, T, T]` (仅限已 push) | `UNTESTED` (Audited: 2026-09-05) | `coverage_scope: tracked_committed_pushed_content_only`。存在未提交改动的仓库有本地丢失风险。 |
| **T7 PROJECTS Root (733GB)** | Samsung T7 | `MIXED` | `[F, F, F]` (Aggregate缺口大) | `UNTESTED` (Audited: 2026-09-05) | 容器级缺乏统备。后续按子项目价值独立建档，不要求整根全量备份。 |
| **T7 Teaching Historical Roots** | Samsung T7 | `UNPROTECTED` | `[F, F, F]` | `UNTESTED` (Audited: 2026-09-05) | 历年课件无第二副本。计划后续对高价值大包制作异地火种。 |
| **Zotero Data Directory** | Internal SSD | `UNPROTECTED` | `[F, F, F]` | `UNTESTED` (Audited: 2026-09-05) | 完整数据目录（含数据库与 storage 附件）仅在主盘，同盘 bak 无法跨介质容灾。 |
| **Calibre Ebook Library** | Internal SSD | `UNPROTECTED` | `[F, F, F]` | `UNTESTED` (Audited: 2026-09-05) | 图书与元数据无外置与异地保护。 |
| **Personal & Admin Documents** | Desktop/Docs | `UNPROTECTED` | `[F, F, F]` | `UNTESTED` (Audited: 2026-09-05) | 裸露在桌面台面，缺乏统一归集与备份。 |

---

## 7. 故障场景推演矩阵 (Failure Scenarios Analysis)

| 故障场景 (Failure Scenario) | 哪些数据现在能恢复？ | 从哪里恢复？ | 哪些数据目前绝对恢复不了？(致命断裂点) |
| :--- | :--- | :--- | :--- |
| **Scenario A: 内置 SSD 硬件损坏** | • Obsidian 知识库<br>• 已 push 的 Git 仓库源码<br>• T7 上的外置项目与素材 | • GitHub 远程仓库<br>• Samsung T7 外置盘 | **• 本地各仓库未 commit 的代码修改**<br>**• Zotero 数据库与 storage 附件 (全灭)**<br>**• Calibre 电子书库 (全灭)**<br>**• 桌面与文稿中的个人资质证件与表格 (全灭)** |
| **Scenario B: Samsung T7 硬件损坏** | • 内置 SSD 上的所有核心代码与笔记依然完好 | • 内置 SSD 活跃副本 | **• T7 上未做归档火种的独创工程源文件 (全灭)**<br>**• 171GB 历年教学课件与学生归档 (全灭)**<br>**• 外置 Unity 编辑器与大模型权重 (需重新下载)** |
| **Scenario C: 逻辑误删或错误同步** | • Obsidian 笔记 (可通过 Git commit log 回滚) | • Git 版本历史 | **• Zotero 库内文献误删 (无版本快照回退)**<br>**• T7 上的工程误删或误改 (exFAT 无快照回退)**<br>**• 桌面文件误按 Command+Delete 清空** |
| **Scenario D: 文件静默损坏 (Bit Rot)** | • 有 Git SHA 校验的代码文件能察觉异常 | • Git remote | **• T7 上存储数年的多媒体视频与工程资产包 (exFAT 无法察觉静默损坏)** |
| **Scenario E: 本地未 push 代码丢失** | • 历史已提交且已 push 的代码可重新 clone | • GitHub | **• 本地工作区当前修改（未 commit 部分）永久丢失** |
| **Scenario F: 百度网盘账号暂时受限** | • 本地所有工作不受影响 (Control Plane 与 T7 均在本地) | • 本地物理介质 | **• 无法从云端提取历史归档火种（但在本地无依赖的情况下不阻塞日常工作）** |

---

## 8. 专项备份策略：Zotero 完整数据目录方案 (Zotero Strategy)

**严禁将“在线定时快照 zotero.sqlite”作为推荐备份方案。**

依据 Zotero 官方灾备指南：
1. **数据不仅包含数据库**：Zotero 数据由 `zotero.sqlite`（元数据/目录树/卡片）与 `storage/`（所有 PDF 文献全文、附件与批注）共同构成，仅备数据库会导致附件全量断链；
2. **静态复制原则**：在执行文件级备份与复制前，**必须先退出 Zotero 进程**，以防止 SQLite 写入时产生数据库碎片或不一致状态；
3. **导出与同步的局限**：导出 RDF/BibTeX 会丢失阅读进度与收藏夹层级，Zotero Sync 亦不能替代脱机完整备份。

### 推荐的最小 Zotero 备份规程 (Standard Procedure)
```
[ 退出 Zotero 应用 ]
         │
         ▼
[ 完整复制 Data Directory ] ──► (包含 zotero.sqlite + storage/ + translators)
         │
         ▼
[ 验证本地副本完整性 ] ──────► 复制到 Samsung T7 专属备份目录
         │
         ▼
[ 打包加密异地转存 ] ────────► 生成带时间戳归档包，选择性上传至 /apps/bdpan/
```

---

## 9. 最小充分备份架构与实施清单 (Minimal Backup Implementation Backlog)

本政策**严格不授权任何真实的数据上传、删除、复制或迁移**。后续落地动作全部拆解为独立 GitHub 工单，在获得用户明确授权后推进：

1. **[TASK] 排查与收敛本地未提交 Git 代码**：对存在本地修改的活跃代码库进行审查与安全推送，消除本地孤本风险；
2. **[TASK] Zotero complete data-directory backup and restore prototype**：依据官方指南，设计退出应用后的整目录备份与沙箱还原验证方案；
3. **[TASK] 个人档案与敏感证件收敛与加密保护**：在 `~/Documents/Personal/` 建立安全归集，研究本地离线加密机制，排除普通明文云上传；
4. **[TASK] T7 高价值项目按需冷冻归档清单**：针对 T7 上已结题的高价值项目与历史教学大包，制定单项目 SHA-256 打包与异地火种转存计划；
5. **[TASK] 首次端到端恢复演练 (Restore Verification Drill)**：针对核心资产在隔离测试目录验证解压、校验与业务可用性。

---

## 10. 决策结论 (Decision Statement)

**Accepted:**
> 确立数据保护与存储分层的正交性；建立解耦的基础保护状态、保护能力与恢复验证模式；明确多资产根容器聚合（Aggregate）与具体子资产独立分类原则；确立机密材料（`SECRET_MATERIAL`）安全边界；确立 Zotero 退出应用并完整备份数据目录的官方规范。
> 
> 本决策不授权进行任何实际数据变动、云端上传或介质格式化。
