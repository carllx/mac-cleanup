# 调研报告：macOS 在日常运行、安装软件与系统更新时的安全余量模型

**票据关联**：[Issue #5](https://github.com/carllx/mac-cleanup/issues/5) (Part of [Wayfinder Map #1](https://github.com/carllx/mac-cleanup/issues/1))  
**研究日期**：2026-09-04  
**目标硬件画像**：Apple Silicon M1，8GB 统一内存，256GB 级内置 SSD（APFS 容器上限约 245.1GB，当前可用空间约 2.6 ~ 2.9GiB，处于严重告急状态）。  
**关联输出**：直接为 [Issue #6: Storage Tiering](https://github.com/carllx/mac-cleanup/issues/6) 与 [Issue #9: Operational Buffer Policy](https://github.com/carllx/mac-cleanup/issues/9) 提供严格由证据推导的安全阈值与触发准则。

---

## 1. 执行摘要（Executive Summary）

针对 256GB 级内置固态硬盘与 8GB 统一内存的硬件约束，本报告基于 **Apple 官方技术规范**、**macOS 系统架构机制（APFS、VM 动态分页与系统更新管道）** 以及 **只读本机事实探针（Read-only Fact Probe）**，建立了可解释的分级磁盘安全余量模型。

核心结论如下：

1. **严格区分三种空间概念，拒绝单一固定数字**：
   - **Apple 官方硬性门槛（Enforced Minimum / Staging Requirement）**：系统更新或安装程序启动前执行的前置检查空间（如大版本更新要求的 26~44.5GB，CC 桌面端要求的 4GB）。低于此值，安装器直接报错中断。
   - **安装暂存与膨胀开销（Staging / Expansion Overhead）**：下载包（Compressed Payload）+ 解压展开（Uncompressed Staging）+ 目标落地文件（Final Install Footprint）的重叠峰值。对于重度依赖系统 `$TMPDIR` 的安装器，实际瞬时峰值空间可能达到目标应用体积的 **2.0 ~ 2.5 倍**。
   - **工程动态操作余量（Engineering Operational Margin）**：为 8GB 内存设备上的动态 Swap 膨胀（常见 4~8GB）、APFS 写入写时复制（CoW）元数据开销、系统日志与应用本地 Cache 波动所预留的底线缓冲区。
2. **对当前本机约 2.6 ~ 2.9GiB 可用状态的定性**：
   - **定性为：【Critical / Emergency 状态】（严重告急）**。
   - 本机探针显示系统当前已产生 4 个各 1.0GB 的 swapfile（总计 4096MB，已用 3200MB+），当前物理剩余约 2.6GiB 甚至无法支撑系统在内存压力激增时再分配 3 个连续 swapfile。
   - 系统可随弃空间（Purgeable）与 Opportunistic Capacity 实测为 0.0 GiB，表明系统已失去自动削峰缓冲能力。任何中大型写入、解压或安装操作均极易引发内核 panic、应用静默崩溃或磁盘写满锁死。
   - **决策**：**必须立即停止所有大型写入、软件安装与系统更新，进入单向紧急空间恢复状态**。
3. **安装 Substance 3D Painter 前的准入 Gate**：
   - Adobe 官方要求 Substance Painter 拥有至少 30GB 可用 SSD 空间（推荐 50GB），CC 客户端基础缓冲要求 4GB，解压暂存额外需要约 5~8GB。
   - 即使未来将资产架与 SVT 缓存重定向至外置盘，本地主程序与安装暂存仍需至少 15GB 以上连续可用空间。在内置盘恢复至 **Healthy Working Buffer（≥ 20GB，建议 ≥ 25GB）** 之前，严禁尝试触发 Substance Painter 的本地安装。

---

## 2. 核心证据表（Evidence Table: Official & Verified Facts）

| 维度 / 场景 | 证据类型 | 官方一手依据 / 本机实测参数 | 提取的工程含义与约束 |
| :--- | :--- | :--- | :--- |
| **macOS 大版本升级 (Major Upgrade)** | **Verified** (Apple Support) | Apple 官方文档明确记录：macOS Big Sur 从早期版本升级需要高达 **44.5 GB** 可用空间；从 Sierra 升级需要 **35.5 GB**；Sequoia/Sonoma 普遍要求 **26 ~ 35 GB** 以上连续可用空间。 | 官方对“系统更新所需空间”存在硬性校验。这部分空间不仅包含约 12~14GB 的安装器镜像，还包含备用 APFS 快照、新 Sealed System Volume (SSV) 的构建与原子切换空间。 |
| **macOS 小版本增量更新 (Minor Update)** | **Verified** (Apple Support & 本机探针) | 本机实测 `softwareupdate --list` 探针发现增量更新包标称体积为 3.8GB（3831075KiB），但官方更新管道需要解压、验证系统快照并写入更新 staging，普遍要求 **10 ~ 15 GB** 的空闲空间。 | 低于 10GB 时系统更新管道极易报“Your disk does not have enough free space”，且断电或中断风险极高。 |
| **APFS 容器与快照空间共享机制** | **Verified** (Apple Developer & Man) | APFS 架构下，所有卷（System、Data、Preboot、Recovery、VM）共享同一个 Container 的物理未分配空间。系统根卷采用 Sealed System Volume (SSV) 快照启动。 | 用户 Data 卷、虚拟内存 VM 卷与更新下载缓冲区直接争夺同一池空间。一旦 Container 空间用尽，所有挂载卷的写入同时被阻断。 |
| **macOS 存储容量三级判定 (URLResourceValues)** | **Verified** (Apple Developer API) | Apple 官方 Foundation API 明确定义三层可用空间：<br>1. `volumeAvailableCapacityKey`<br>2. `volumeAvailableCapacityForImportantUsageKey`<br>3. `volumeAvailableCapacityForOpportunisticUsageKey` | 系统内部分级区分资源写入优先级。当 Opportunistic 降为 0 时，系统完全停止后台投机性缓存与同步；系统仅依靠 Important 空间勉强维持核心运转。 |
| **内存压力与虚拟内存 (VM Swap)** | **Verified** (macOS 本机探针 / sysctl) | - 硬件物理内存：8 GB；<br>- 当前 swapusage：`total = 4096.00M, used = 3216.62M, free = 879.38M`；<br>- `/System/Volumes/VM/` 实测已有 4 个 1.0GB 的 `swapfile` (0~4)。 | 8GB 内存设备在高负载下，macOS 动态分页机制（`dynamic_pager`）会以 1GB 为步长动态创建 swapfile。若磁盘剩余空间不足以容纳 swapfile 扩展，前台程序将直接崩溃（OOM / Out of VM space）或系统冻结。 |
| **Creative Cloud 桌面安装器** | **Verified** (Adobe Support) | Adobe 官方文档明确指出：Creative Cloud 桌面应用程序自身安装需要 **至少 4 GB 可用硬盘空间**，且必须安装在主操作系统盘。 | CC Desktop 核心启动器不接受外置，4GB 是启动器本身的最小底线，尚未计入任何具体创作套件。 |
| **Substance 3D Painter 空间需求** | **Verified** (Adobe Support) | Adobe 官方系统要求明确规定：<br>- 最低要求（Minimum）：**30 GB 可用 SSD 空间**；<br>- 推荐配置（Recommended）：**50 GB 可用 SSD 空间**；<br>- 极致配置（Optimal）：**70 GB 可用 SSD 空间**。 | 3D 纹理工作流涉及密集的贴图解压、虚拟纹理缓存（SVT）与网格烘焙，极度消耗磁盘 I/O 与空间。 |
| **大型应用安装 Staging 重叠峰值** | **Inferred** (工程推导与经验模型) | 下载的安装包（DMG/PKG/CCX 分片）与解压后的释放目录同时存在：`Peak = DownloadSize + ExtractedSize + TargetInstallSize`。 | 对于 3~5GB 的应用包，安装瞬间的系统临时盘消耗可达 8~12GB；在安装未完全 Commit 前，空间不会释放。 |

---

## 3. 本机只读事实探针审计（Local Fact Probe Audit）

为消除推测，针对当前机器进行了只读底层探针检测（去标识化）：

1. **APFS 容器与卷物理分配**：
   ```text
   APFS Container Reference: disk3 (Capacity Ceiling: 245.1 GB)
   Capacity In Use By Volumes: 242.5 GB (99.0% used)
   Capacity Not Allocated: 2.6 GB (1.0% free)
   ------------------------------------------------------------
   - System (disk3s1s1 - SSV Snapshot): 12.6 GB (Read-Only)
   - Preboot (disk3s2): 9.1 GB
   - Recovery (disk3s3): 1.3 GB
   - VM (disk3s6): 4.3 GB (Active Swap Storage)
   - Data (disk3s5): 215.1 GB (User & Application Data)
   ```
2. **Swift 运行时系统可用容量探针（`URLResourceValues`）**：
   ```text
   Volume Total Capacity: 228.27 GiB
   Normal Available (volumeAvailableCapacityKey): 2.62 GiB
   Important Available (volumeAvailableCapacityForImportantUsageKey): 2.92 GiB
   Opportunistic Available (volumeAvailableCapacityForOpportunisticUsageKey): 0.00 GiB
   ```
   - **可随弃空间（Purgeable）**：仅 `Important - Normal = 0.30 GiB`（约 300MB），几乎无可由系统主动回收的弹性空间。
   - **投机空间（Opportunistic）**：`0.0 GiB`，系统已彻底关闭一切非关键后台任务空间配额。
3. **虚拟内存与 Swap 活跃度**：
   ```text
   vm.swapusage: total = 4096.00M, used = 3216.62M, free = 879.38M
   /System/Volumes/VM/:
     swapfile0 (1.0G), swapfile1 (1.0G), swapfile2 (1.0G), swapfile4 (1.0G)
   vm_stat: Pages stored in compressor: 817849 (~13.1 GB uncompressed data in RAM cache)
   ```
   - 内存压缩器已满负荷，已转储 4GB 数据到磁盘交换卷，且当前已消耗了 3.2GB 的 swap 空间。
4. **APFS 本地快照探针**：
   ```text
   diskutil apfs listSnapshots /System/Volumes/Data -> No snapshots for disk3s5
   ```
   - Data 卷不存在积压的 Time Machine 本地快照，表明 **2.6GiB 是真实的物理未分配底线，不存在隐藏快照释放后可瞬间增加几十 GB 的幻想**。
5. **系统更新探针（OTA Pending Updates）**：
   ```text
   softwareupdate --list -> macOS Tahoe 26.6.2 (Size: 3,831,075 KiB, ~3.8 GiB)
   ```
   - 当前存在待安装的增量更新。即使仅 3.8GB 标称下载体积，由于当前剩余仅 2.6GiB，任何触发更新的行为都将直接失败，甚至可能在下载分片占满后导致系统崩溃。

---

## 4. 当前约 2.6 ~ 2.9GiB 状态的深度定性分析

根据上述事实，当前设备的存储状态定性为：**【严重告急（Critical / Emergency）】**。

### 4.1 为什么 2.6GiB 极其危险？
1. **Swap 扩展死锁风险**：
   本机物理内存仅 8GB，日常多任务（Chrome、VS Code、Antigravity、微信、系统服务）极易将物理 RAM 耗尽。macOS 内核的 `dynamic_pager` 会根据内存压力以 1GB 为单位动态创建新的 `swapfileN`。
   当前空闲仅 2.6GiB，若前台发生复杂任务（如同时打开大型文档、构建脚本或浏览器多标签），系统仅能再成功创建 2 个 swapfile。一旦第 3 个创建失败，内核将触发 `vm_pageout` 死锁，导致前台应用直接被强制终止（OOM Termination），严重时可能导致系统崩溃重启。
2. **写时复制（CoW）碎片化与元数据饥饿**：
   APFS 是强写时复制（Copy-on-Write）文件系统。当磁盘使用率达到 99% 时，空闲空间呈现高度碎片化，APFS 写入新数据时寻找连续空闲块的开销急剧增加，B-Tree 元数据节点更新可能因缺乏连续可用块而失败，极大地加速磁盘性能衰减。
3. **应用自动保存与 SQLite 事务挂起**：
   IDE、Zotero、微信等依赖 SQLite WAL 模式的应用，在事务写入与 Checkpoint 时需要临时日志空间。在仅剩 2.6GiB 且系统全局竞争时，可能出现写事务失败甚至损坏本地轻量数据库的风险。

---

## 5. 分级安全余量模型（Tiered Storage Buffer Model）

基于 256GB 级 SSD（实用空间约 245GB）与 8GB 内存基准，结合官方要求与工程实践，构建以下 5 级余量梯队：

```mermaid
graph TD
    subgraph 磁盘安全余量梯阶 (256GB Mac / 8GB RAM)
        L1["【等级 1: 严重告急 (Critical / Emergency)】<br>< 5 GiB (当前处于 ~2.6 GiB)<br>动作: 绝对熔断, 禁一切大型写入/安装/更新"]
        L2["【等级 2: 紧急恢复期 (Recovery Required)】<br>5 GiB ~ 12 GiB<br>动作: 仅限小文件操作, 禁止应用安装, 优先清淤归档"]
        L3["【等级 3: 最低日常运行余量 (Minimum Operating Buffer)】<br>12 GiB ~ 20 GiB<br>动作: 满足 8GB 内存日常 Swap 与浏览器/IDE 稳定运转"]
        L4["【等级 4: 健康工作余量 (Healthy Working Buffer)】<br>20 GiB ~ 35 GiB (约占总盘 10%~15%)<br>动作: 允许常规大型应用安装 staging, 支持日常创作"]
        L5["【等级 5: 系统升级就绪 (Install & Upgrade Ready)】<br>> 35 GiB (大版本推荐 45 GiB+)<br>动作: 满足 macOS Major Upgrade 与大型生产力套件安装"]
    end

    L1 -->|清理与外置分流| L2
    L2 -->|达成基础清淤| L3
    L3 -->|存储分层实施 #6| L4
    L4 -->|深度空间优化| L5
```

### 详细分级定义与触发策略表

| 等级名称 | 容量区间 (GiB) | 状态定性与系统特征 | 允许的操作 | 严厉禁止的操作 | 触发治理动作 (Policy Action) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **1. Critical / Emergency<br>(严重告急)** | **< 5.0 GiB**<br>*(本机当前处于此区间: ~2.6GiB)* | **极度危险**。<br>- VM Swap 面临枯竭；<br>- Opportunistic 空间归零；<br>- APFS CoW 严重降速；<br>- 存在系统死锁风险。 | - 只读事实探测；<br>- 文本编辑与配置微调；<br>- 紧急释放空间的删除/归档操作。 | - **严禁一切软件安装**；<br>- **严禁系统更新**；<br>- **严禁下载大文件/解压**；<br>- **严禁大型项目构建/烘焙**。 | **全局熔断**。<br>系统进入只读与单向清淤模式，任何非清理类写入动作必须被 Gate 拦截。 |
| **2. Recovery Required<br>(恢复观察期)** | **5.0 ~ 12.0 GiB** | **亚健康脆弱态**。<br>- 能够承受 1~2 个额外 swapfile；<br>- 但无法承受任何中型临时解压。 | - 正常命令行开发；<br>- 轻量 Git 操作；<br>- 常规浏览与文档编写。 | - 严禁安装任何大型软件；<br>- 严禁触发 macOS 小版本更新；<br>- 避免打开超大型媒体工程。 | **空间治理优先**。<br>推进 [Issue #6] 分层策略，将缓存、开发环境依赖及项目归档至外置盘或云端。 |
| **3. Minimum Operating<br>(最低日常运行)** | **12.0 ~ 20.0 GiB** | **日常稳定运转底线**。<br>- 充分保障 8GB RAM 的 Swap 波动（4~8GB）；<br>- 容纳常规浏览器 Cache、IDE 索引与临时编译文件。 | - 正常科研与高校教学工作流；<br>- Web/Python 日常软件工程开发；<br>- 小型独立应用（< 500MB）安装。 | - 禁止安装大型创作应用（如 Adobe、Unity Editor）；<br>- 禁止尝试 macOS 大版本升级。 | **维持防线**。<br>在此区间内，日常系统运行不会出现告警，属于长期运行的容忍下限。 |
| **4. Healthy Working<br>(健康工作余量)** | **20.0 ~ 35.0 GiB**<br>*(约占总盘 10% ~ 15%)* | **健康弹性状态**。<br>- 系统可自由使用 Opportunistic 空间；<br>- 具备承接常规大型应用安装 staging 的弹性缓冲。 | - 允许安装常规生产力工具；<br>- 允许进行中型项目 3D 渲染与轻量缓存；<br>- 允许小版本 macOS 增量更新。 | - 未经提前规划不建议直接在内置盘安装 30GB+ 极重型软件（如完整 Xcode）。 | **应用安装就绪门槛**。<br>达到此区间后，方可评估安装 Substance 3D Painter 等大型创作工具。 |
| **5. Install / Upgrade Ready<br>(升级就绪)** | **> 35.0 GiB**<br>*(大版本建议 45 GiB+)* | **高容量冗余状态**。<br>- 满足 Apple 官方对大版本系统升级（Major Upgrade）的最高检查门槛。 | - 全功能无限制操作；<br>- 执行 macOS 大版本操作系统升级（如跨大版本 OTA）。 | 无。 | **大版本升级通行证**。<br>升级前仍建议在外置盘留存完整备份。 |

---

## 6. 安装 Adobe Substance 3D Painter 的先决条件与 Gate 建议

结合 [Issue #11 大型应用拓扑调研](https://github.com/carllx/mac-cleanup/issues/11) 与本报告证据，对安装 Substance 3D Painter 设立专门的工程放行准则（Gate）：

### 6.1 空间开销解剖（Footprint Anatomy）
1. **主程序体积（Application Binary）**：约 3.0 ~ 4.0 GiB（必须安装在内置系统盘 `/Applications`，官方 Error 179 不支持外置）。
2. **下载与解压暂存（Download & Unpack Staging）**：Creative Cloud 下载的 dmg/zip 分片以及在 `$TMPDIR` 中解压的临时空间，需要额外约 **6.0 ~ 8.0 GiB**（安装完成后自动清除，但安装过程中必须共存）。
3. **资产与素材库（Assets & Shelves）**：官方启动附带基础笔刷材质约 1.5GiB，后续扩展可能达到 10~30GiB（**官方支持配置到 Samsung T7**）。
4. **纹理缓存与临时暂存（SVT Cache & Temp）**：烘焙高分辨率纹理时临时开销极大，数百 MB 至数十 GB 不等（**官方支持配置到 Samsung T7**）。

### 6.2 Substance Painter 安装放行准则（Installation Gate）

只有同时满足以下前置条件，才允许在系统上触发 Substance 3D Painter 的安装：

- [ ] **Gate 1（系统余量门槛）**：内置盘通过清理与分流，物理空闲空间必须稳定达到 **Level 4: Healthy Working Buffer（≥ 20.0 GiB，建议 ≥ 25.0 GiB）**。
- [ ] **Gate 2（外置拓扑就绪）**：外置 Samsung T7 已正常挂载，且已预先创建好专用目录（如 `<T7>/Creative/Substance3D/Libraries` 与 `<T7>/Creative/Substance3D/Temp`）。
- [ ] **Gate 3（安装后重定向机制）**：安装主程序一旦完成，必须在首次启动项目前，立即进入偏好设置将 Libraries 与 Temporary Files 重定向至外置盘，防止内置盘被资产和 SVT 缓存迅速蚕食跌回 Level 1。

---

## 7. 对后续架构与策略票据的输入规范（Inputs for #6 & #9）

本调研产出的分级余量与机制，作为下游票据的硬性设计输入：

### 7.1 对 [Issue #6: Storage Tiering Implementation](https://github.com/carllx/mac-cleanup/issues/6) 的输入：
1. **内置盘瘦身目标值（Target Free Space Goal）**：
   - 第一阶段目标（Phase 1）：必须将内置盘从当前的 **2.6GiB** 提升至 **Level 3 最低运行余量（≥ 15.0 GiB）**，彻底脱离死锁风险。
   - 第二阶段目标（Phase 2）：通过将大型资产、历史项目与开发包缓存外置到 Samsung T7 及云端，使内置盘常态保持在 **Level 4 健康工作余量（20 ~ 25 GiB）**。
2. **分层迁移优先序**：
   - 优先迁移占用空间大、非系统级绑定的静态实体（如 Unity Projects 15GB、Blender 资产与便携包、冗余 Conda 实例与未激活环境、历史多媒体）。

### 7.2 对 [Issue #9: Operational Buffer & Emergency Cleanup Policy](https://github.com/carllx/mac-cleanup/issues/9) 的输入：
1. **自动化监控与预警触发点（Threshold Triggers）**：
   - **Yellow Alert（黄色告警，触发条件：< 15 GiB）**：提醒用户关注空间，触发自动扫描清理低风险可再生缓存（如 `uv cache clean`、过期的日志、系统随弃缓存）。
   - **Orange Alert（橙色告警，触发条件：< 8 GiB）**：拦截一切新的应用安装与大文件下载，通知用户挂载外置盘进行数据归档。
   - **Red Alert（红色紧急熔断，触发条件：< 5 GiB）**：直接进入 Emergency 模式，禁止所有写操作，执行紧急清淤 Runbook。
2. **容量判定接口标准**：
   - 策略脚本中不得仅依据 Finder 报告的 Available（其包含未清理的 purgeable），必须以 POSIX `statvfs` 的 `f_bavail` 或 Foundation 的 `volumeAvailableCapacityKey` 作为真实物理未分配空间的权威依据。

---

## 8. 官方技术依据与参考源（Official Sources）

本报告的结论建立在以下一手权威资料与技术规范之上：

1. **Apple 官方支持文档：如何升级 macOS 并满足存储要求**:
   - Apple Support: *How to upgrade macOS and storage requirements*  
     [https://support.apple.com/HT201475](https://support.apple.com/HT201475)
2. **Apple 官方支持文档：macOS Big Sur 硬件技术规格与最低可用空间**:
   - Apple Support: *macOS Big Sur - Technical Specifications (Available storage requirements: 35.5GB to 44.5GB)*  
     [https://support.apple.com/kb/SP830](https://support.apple.com/kb/SP830)
3. **Apple Developer 文档：URLResourceValues 与存储可用容量分级 API**:
   - Apple Developer Documentation: *Checking Volume Storage Capacities (`volumeAvailableCapacityKey`, `volumeAvailableCapacityForImportantUsageKey`, `volumeAvailableCapacityForOpportunisticUsageKey`)*  
     [https://developer.apple.com/documentation/foundation/urlresourcevalues](https://developer.apple.com/documentation/foundation/urlresourcevalues)
4. **Apple 官方技术规范：关于 APFS 容器空间共享与卷角色**:
   - Apple Support: *Use Disk Utility to add APFS volumes and understand shared container space*  
     [https://support.apple.com/guide/disk-utility/add-apfs-volumes-dskuae1050c4/mac](https://support.apple.com/guide/disk-utility/add-apfs-volumes-dskuae1050c4/mac)
5. **Adobe 官方系统需求：Substance 3D Painter 硬件与可用空间标准**:
   - Adobe Help Center: *Substance 3D Painter System Requirements (Minimum 30GB, Recommended 50GB SSD space)*  
     [https://helpx.adobe.com/substance-3d-painter/getting-started/system-requirements.html](https://helpx.adobe.com/substance-3d-painter/getting-started/system-requirements.html)
6. **Adobe 官方系统需求：Creative Cloud 桌面应用程序安装与空间规范**:
   - Adobe Help Center: *Adobe Creative Cloud desktop app system requirements (4GB available hard-disk space)*  
     [https://helpx.adobe.com/creative-cloud/system-requirements.html](https://helpx.adobe.com/creative-cloud/system-requirements.html)
7. **Adobe 官方故障排查：错误 179（Error 179 - 本地主驱动器安装限制）**:
   - Adobe Help Center: *Troubleshoot Adobe Creative Cloud install issues - Error 179*  
     [https://helpx.adobe.com/creative-cloud/kb/troubleshoot-download-install-logs.html#error179](https://helpx.adobe.com/creative-cloud/kb/troubleshoot-download-install-logs.html#error179)
