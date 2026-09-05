# Storage Health & Cleanup Runbook

本运行手册（Runbook）沉淀了 `mac-cleanup` 项目在实践中验证的磁盘空间治理核心准则、分层存储模型、维护触发阈值、增量探测规范及跨介质迁移标准。旨在确保未来无论在 1 周、1 个月或更久后的例行或应急维护中，均能**基于基线增量与增长源直接定位**，彻底杜绝无差别的大规模全盘盲扫。

---

## 1. 核心治理原则 (Core Principles)

1. **增长优先，而非绝对体积优先 (Growth-First, Not Size-First)**：
   - 关注“自上次基线以来，究竟是哪个目录发生了异常膨胀”，而非机械列出“当前磁盘上最大的目录”。
2. **持久回收优先于再生清理 (Durable Reclaim > Regenerable Cleanup)**：
   - 清理过时运行时版本、孤立第三方框架、已结项归档外置等属于高价值持久回收；浏览器缓存、应用构建中间物等会快速自然再生，不计入长效可用储备。
3. **放置与保护正交 (Placement != Protection)**：
   - 数据存放在高速外置盘（如 T7）仅解决内置盘空间分配（Placement）问题；将数据移入 T7 并不等同于拥有了数据备份。真正的备份必须拥有独立的次级副本或远端镜像。
4. **运行时依赖门禁 (Consumer Gate Before Runtime/Framework Deletion)**：
   - 删除任何系统级 Framework、SDK 或环境之前，必须通过动态链接分析（`otool -L`）、全局元数据检索与活跃进程排查，拿到确凿的反向零消费（Zero-Consumer）证据。
5. **迁移三步验证法 (Copy → Verify → Delete Source)**：
   - 任何资产迁移必须先复制，通过数量、字节、哈希采样三重核对（`COPY_VERIFIED_PASS`）后，方可解除源端锁定并移除源文件。
6. **人工授权与不可变原则 (User Approval Before Mutation)**：
   - 任何涉及文件删除、路径变更、权限提权的操作必须获得用户明确批复，严禁后台顺手清理未授权目录。
7. **适度停止原则 (Sufficiency Stop)**：
   - 清理的目的是维持系统健康运行缓冲，而非无限追求最大剩余空间。一旦达成健康基线或预算目标，立即停止。

---

## 2. 存储分层模型 (Storage Tier Model)

| 存储层级 | 角色定位 | 典型承载内容 | 访问特征与策略 |
| :--- | :--- | :--- | :--- |
| **Tier 1: Internal SSD** | **控制平面 / 热核心 (Control Plane / Hot Core)** | macOS 核心、活跃 IDE/Toolchains、当前主要运行时 (Node, Python, Active R)、系统 Framework | 高速低延迟，保持系统基础环境稳固，禁止堆放大型冷归档 |
| **Tier 2: External T7** | **温数据 / 大型工作集 (Warm Capacity / Large Working Set)** | 重型引擎与应用 (Unity, Audition 等)、音视频与材质工程、历史教学与项目完整归档、大型只读知识库 | 大容量，便携外置；脱机时仅大型工程不可用，不影响核心系统运行 |
| **Tier 3: Cloud / Remote** | **冷归档 / 按需拉取 (Cold / On-demand Archive)** | 往期结项资料、已出版或已交付教材全量镜像、云端备份包 | 长期持久保存，本地不常驻，需要时按需下载 |
| **Orthogonal Plane: Backup** | **独立恢复平面 (Orthogonal Recovery Plane)** | GitHub 远程代码库、独立物理备份硬盘、结构化导出快照 | 与存储介质正交，专门用于灾难恢复 |

---

## 3. 运行健康缓冲定义 (Operational Buffer Model)

针对内置主系统盘（256 GB 级别 SSD），定义以下四个物理健康区间（基于 `df -h /System/Volumes/Data` 实测值）：

- **🔴 RED (`< 5 GiB`)**：
  - **严重告警**。系统面临 APFS 写入阻塞、虚拟内存换页衰竭、内核 Panic 风险。必须立即执行既定紧急清理，禁止任何编译与系统升级。
- **🟠 ORANGE (`5 – 12 GiB`)**：
  - **受限运行**。空间紧张，Xcode 构建、大型依赖安装（npm install、Docker 镜像拉取）极易失败。应冻结非必要增量，尽快安排持久释放。
- **🟡 YELLOW (`12 – 20 GiB`)**：
  - **关注预警**。日常开发可用，但缺乏突发工程准备空间。建议在空闲期执行轻量维护。
- **🟢 GREEN (`>= 20 GiB`)**：
  - **健康稳定**。系统、虚拟内存交换与日常开发均享有充裕安全冗余。

> **特别说明**：
> 普通日常运行进入 **🟢 GREEN (>= 20 GiB)** 即代表健康达标。
> **大型安装就绪目标（如 60 GiB+）** 属于特定工程预算（Planned-Install Target），切勿将其与日常 GREEN 混为一谈。

---

## 4. 维护触发与决策叠加层 (Maintenance Trigger Overlay)

在未来评估“当前是否需要启动清理”时，采用如下判断流：

```text
                  [ 获取当前真实可用容量 (df -k) ]
                                 │
         ┌───────────────────────┴───────────────────────┐
         ▼                                               ▼
   [ >= 40 GiB ]                                   [ < 40 GiB ]
         │                                               │
   有无大型安装计划？                                 [ 30 - 40 GiB ]
    ├─ 无  ──> [ NO_CLEANUP (保持现状) ]                 └─> 运行增量审查 (Growth Delta Review)
    └─ 有  ──> 评估 [ PLANNED_INSTALL_READY ] Gate       [ 20 - 30 GiB ]
               (工程预算 >= 60 GiB 专项核验)              └─> 只读盘点 (Read-Only Inventory)
                                                         [ < 20 GiB ]
                                                          └─> 进入正式清理规划 (Cleanup Required)
```

- **>= 40 GiB** 且无安装计划：状态极其充裕，**禁止盲目启动任何清理**。
- **30 – 40 GiB**：仅审查增量来源，不自动删除。
- **20 – 30 GiB**：执行只读目录盘点。
- **< 20 GiB**：跌破日常健康缓冲，启动标准清理批次。
- **工程预算目标**：
  - `PLANNED_INSTALL_READY`：最低门槛约为 **60 GiB**（用于承载大型 IDE、多系统 SDK 或重大版本升级）。
  - `SEMESTER_RESERVE`：理想学期冗余目标约为 **70 – 80 GiB**。
  - *注：上述预算均为工程实践设定的防御性缓冲，非 Apple 官方硬性指标。*

---

## 5. 基于增量的未来检查规范 (Delta-Based Future Probe)

未来在排查空间时，执行如下标准最小检查输出模板：

```text
=== STORAGE HEALTH DELTA PROBE ===
Timestamp:             YYYY-MM-DDTHH:MM:SS
Current Free:          XX.XX GiB (XXXXX KiB)
Previous Baseline:     YY.YY GiB (YYYYY KiB)
Delta Since Baseline:  +/- Z.ZZ GiB

Top Growth Sources (自基线以来的主要增量路径):
1. [Path/Category A] -> +X.X GiB (说明：如 Xcode 编译产物/模型文件)
2. [Path/Category B] -> +Y.Y GiB (说明：如本地开发临时包)

Known Regenerable Growth (已知可再生增量):
- 浏览器缓存 / 运行时日志 / 包管理器下载缓存: ~N GiB

Known Durable Candidates (已知潜在持久释放项):
- 往期结项归档 / 闲置独立工具链: ~M GiB

Maintenance Decision:
[ NO_CLEANUP / REVIEW / CLEANUP_REQUIRED / PLANNED_INSTALL_EVALUATION ]
===================================
```

---

## 6. 资产质量与处置模型 (Candidate Quality Model)

在考虑任何清理/归档候选时，从 **体积 (Size)**、**持久性 (Durability)** 与 **风险恢复成本 (Risk / Restore Cost)** 三个维度严格定级：

| 分类标识 (Classification) | 资产特征与定义 | 处理策略 |
| :--- | :--- | :--- |
| **`DURABLE_HIGH_VALUE`** | 孤立弃用的独立环境、废弃系统库、已结项无依赖的静态大归档 | **优先处理**：在消费证明与哈希校验后彻底移除或迁入 T7 |
| **`REGENERABLE_LOW_VALUE`** | 浏览器缓存、开发工作区 node_modules、缩略图缓存 | **低优先级**：删除后会迅速再生，不可作为核心持久空间依靠 |
| **`ACTIVE_KEEP`** | 正在使用的生产力工具、通信软件数据（QQ）、文献库（Zotero）、核心扩展 | **严格保护**：禁止列入删除池，必须在所有脚本中硬编码排除 |
| **`REVIEW_FIRST`** | 个人自制音频/临时项目、跨端未同步的工作副本、未结项电子书 | **人工确认**：必须由用户逐一检视，不可批量自动执行 |
| **`MOVE_TO_T7`** | 静态课件合包、历史教务材料、已归档论文、大型三维渲染工程资产 | **外置归档**：使用严格迁移协议移至 T7，释放内置盘容量 |
| **`DELETE_IF_APPROVED`** | 确认无宿主残留的辅助服务套件、无消费者且经用户授权的孤立配置 | **授权后删除**：必须由用户确认无需回滚后一次性清理 |

---

## 7. 实践沉淀经验法则 (Known Lessons Learned)

1. **APFS 容量只认真实 `df -k`**：
   - 绝不可将 `du` 统计的对象逻辑体积简单等同于文件系统的物理释放量。APFS 包含快照、延迟释放（Trimmer）、空间块共享与稀疏分配机制。
2. **不轻信瞬时释放承诺**：
   - 严禁假定“虽然现在没变，但系统稍后一定会全额释放出来”。评估以执行完毕后 fresh 采集的可用空间为唯一凭据。
3. **macOS 系统级突发增长**：
   - 系统升级下载（`com.apple.MobileSoftwareUpdate`）、Preboot 临时缓存及 Xcode 架构支持包可能在短时间内造成数 GiB 的突发占用，属于事件型波动。
4. **跨文件系统（exFAT）外置的特殊语义**：
   - 外置盘若为 exFAT 格式，不支持 Unix 本地 Socket、FIFO、部分特殊扩展属性及符号链接。
   - 归档包含精细权限或大量符号链接的代码仓库时，必须先打包为 `.tar.gz` 格式再迁入 exFAT，避免元数据失真。
5. **特殊文件的安全区分**：
   - 区分易失性运行时对象（如 Git FSMonitor 的 `.ipc` 套接字）与真实数据文件。易失性套接字属于安全忽略项（`SAFE_EPHEMERAL_OMISSION`），切忌因此阻断整体静态归档。

---

## 8. 标准化迁移验证协议 (Verified Migration Protocol)

任何文件由内置盘向外置盘迁移释放时，必须严密执行以下六步标准协议：

1. **Preflight Inventory（事前盘点）**：
   - 记录源端的确切路径、常规文件数、目录数、逻辑字节总数，并采样计算关键大文件的 SHA-256 哈希。
2. **Stream Copy（流式拷贝）**：
   - 采用对目标文件系统兼容的流式复制，保留文件修改时间戳（mtime），安全过滤不可移植的本地进程套接字。
3. **Double Verification Gate（双向核实门禁）**：
   - 校验目标端常规文件数必须与源端完全一致；
   - 校验逻辑字节总数必须完全一致；
   - 对比采样 SHA-256 哈希必须 100% 匹配；
   - 若包含 Git 仓库，执行 `git fsck --no-reflogs` 确认对象库完整无损；
   - **只有拿到 `COPY_VERIFIED_PASS`，才准许进入下一步**。
4. **Safe Source Removal（安全解除与删除）**：
   - 若源端存在 macOS 保护性 ACL（如 `everyone deny delete`）或只读属性，先以最小权限去除保护，随后彻底删除源路径。
5. **Post-Removal Check（事后状态复核）**：
   - 必须同时确认：**源端路径不存在 (Source Absent) 且 目标端路径完好可读 (Destination Intact)**。
6. **Fresh Capacity Capture（物理容量记录）**：
   - 重新执行 `df -k /System/Volumes/Data`，记录实际物理释放值。

---

## 9. 状态注册表语义标准 (Registry Semantics)

在项目跟踪记录与配置文件中，统一采用如下公开语义标签：

- `KEEP`：生产环境正在使用，严禁删除。
- `FUTURE_PROJECT_DEPENDENCY`：涉及其他并行项目或长期开发依赖，本清理流无权处置。
- `REVIEW_FIRST`：有潜在价值但存在歧义，需用户人工判定。
- `PROTECTED_BY_BACKUP_GATE`：尚无确切二级副本，必须在完成外部备份后方可进一步讨论。
- `REGENERABLE_NOT_DURABLE`：可快速再生资产，不作为核心治理目标。
- `EXTERNALIZED_TO_T7`：已完整迁移至外置存储设备并完成校验。
- `DELETE_CANDIDATE`：已完成消费者排查、具备完全独立性证据、等待最终授权执行。

---

## 10. 停止条件 (Stop Conditions)

维护工作严禁陷入无休止的“空间挖掘”。满足以下任一条件即可触发 **Sufficiency Stop** 结束会话：

- ✅ 内置盘可用空间已处于 **🟢 GREEN (>= 20 GiB)** 且近期无大型重型软件部署需求；
- ✅ 既定阶段的最小高置信回收批次已全部执行并验证闭环；
- ✅ 剩余候选对象的清理风险与人工审核成本已显著大于其可回收容量收益；
- ✅ 剩余占用主要来自用户正在进行的正常活跃工作流；
- ✅ 无任何无法解释的异常后台膨胀。
