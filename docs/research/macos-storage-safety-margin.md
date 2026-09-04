# 调研报告：macOS 在日常运行、安装软件与系统更新时的安全余量模型

**票据关联**：[Issue #5](https://github.com/carllx/mac-cleanup/issues/5) (Part of [Wayfinder Map #1](https://github.com/carllx/mac-cleanup/issues/1))  
**研究日期**：2026-09-04  
**目标硬件画像**：Apple Silicon M1，8GB 统一内存，256GB 级内置 SSD（APFS 容器上限约 245.1GB，当前可用空间处于约 2.6 ~ 2.9GiB 严重告急状态）。  
**关联输出**：直接为 [Issue #6: Storage Tiering](https://github.com/carllx/mac-cleanup/issues/6) 与 [Issue #9: Operational Buffer Policy](https://github.com/carllx/mac-cleanup/issues/9) 提供严格由证据推导的安全阈值与触发准则。

---

## 1. 执行摘要（Executive Summary）

针对 256GB 级内置固态硬盘与 8GB 统一内存的硬件约束，本报告基于 **Apple 官方技术规范**、**macOS 系统架构机制（APFS、VM 动态分页与系统更新管道）** 以及 **只读本机事实探针（Read-only Fact Probe）**，建立了可解释的分级磁盘安全余量模型。

核心结论如下：

1. **严格区分三种空间概念，拒绝单一固定数字**：
   - **Apple 官方硬性门槛（Enforced Minimum / Staging Requirement）**：系统更新或安装程序启动前执行的前置检查空间（如历史大版本更新曾要求 35.5~44.5GB，CC 桌面端自身要求 4GB）。低于此值，安装器直接报错中断。
   - **安装暂存与膨胀开销（Staging / Expansion Overhead）**：下载包（Compressed Payload）+ 解压展开（Uncompressed Staging）+ 目标落地文件（Final Install Footprint）的重叠峰值。对于重度依赖系统临时目录的安装器，实际瞬时峰值空间往往是目标应用体积的数倍。
   - **工程动态操作余量（Engineering Operational Margin）**：为 8GB 内存设备上的动态 Swap 膨胀、APFS 写入写时复制（CoW）元数据开销、系统日志与应用本地 Cache 波动所预留的工程缓冲区。
2. **对当前本机约 2.6 ~ 2.9GiB 可用状态的定性**：
   - **定性为：【Critical / Emergency 状态】（严重告急）**。
   - 本机探针显示系统当前已分配 4 个各 1.0GB 的 swapfile（总计 4096MB，已用 3200MB+），极低的物理剩余容量显著压缩了 swap、temp、logs、cache 与 CoW metadata 的运行余量。
   - API 测量值显示当前 `volumeAvailableCapacityForOpportunisticUsage` 为 0.0 GiB，表明系统在此 API 下已无可用空间分配给非核心/投机性存储（nonessential resources）。
   - **决策**：**必须立即停止所有大型写入、软件安装与系统更新，进入单向紧急空间恢复状态**。
3. **关于 Substance 3D Painter 的先决条件与硬件门槛**：
   - Adobe 官方要求 Substance Painter 拥有至少 30GB 可用 SSD 空间（推荐 50GB，极致 70GB），CC 客户端基础缓冲要求 4GB。
   - **关键硬件事实**：Adobe 官方文档明确规定 macOS 下 Substance 3D Painter 的**最低内存要求为 16 GB RAM**，而**本机仅具备 8 GB RAM**。
   - 因此，**磁盘可用空间达到健康余量仅是必要条件，并非充分条件**。在进入任何安装尝试前，必须在 [Issue #12](https://github.com/carllx/mac-cleanup/issues/12) 中独立评估该机器在 8GB 内存下的硬件兼容性与可行性，本票据绝不做出“达到空间即可安装”的结论。

---

## 2. 核心证据表（Evidence Table: Official & Verified Facts）

| 维度 / 场景 | 证据类型 | 官方一手依据 / 本机实测参数 | 提取的工程含义与约束 |
| :--- | :--- | :--- | :--- |
| **macOS 大版本升级 (Major Upgrade - 历史基准)** | **Verified (Historical Apple Support)** | Apple 官方技术规格（如 macOS Big Sur）明确记录：从早期版本升级需要 **44.5 GB** 可用空间，从 Sierra 升级需要 **35.5 GB**。当前 Tahoe 等现代版本指导指出更新流程已优化空间使用，但未公布单一通用数值。 | 大版本系统升级对系统可用空间存在明确的前置硬性校验，需同时容纳安装器镜像、新 Sealed System Volume (SSV) 构建与快照原子切换空间。历史 35.5~44.5GB 作为高容量升级场景的典型参考。 |
| **macOS 小版本增量更新 (Minor Update Payload)** | **Reported local fact** (本机探针) | 本机实测 `softwareupdate --list` 探针报告当前待装增量更新（macOS Tahoe 26.6.2）payload 标称体积约为 **3.8 GiB**（3,831,075 KiB）。 | 证实当前存在真实的系统增量更新待装包；当前 2.6GiB 甚至低于更新包本身的压缩下载体积。 |
| **macOS 增量更新工程余量建议 (Minor Update Margin)** | **Inferred engineering headroom** (工程推导) | 增量更新管道包含下载校验、包解压、系统快照生成与更新应用 staging，工程上建议预留 **10 ~ 15 GiB** 的空闲空间以保障流程顺畅与断电容错。 | 明确此 10~15GiB 为工程安全建议，而非 Apple 官方公布的绝对硬性门槛。低于此余量时更新失败风险极高。 |
| **APFS 容器空间共享机制** | **Verified** (Apple Developer & Documentation) | APFS 架构下，所有卷（System、Data、Preboot、Recovery、VM）共享同一个 Container 的物理未分配空间。系统根卷采用 Sealed System Volume (SSV) 快照挂载。 | 用户 Data 卷、虚拟内存 VM 卷与更新下载缓冲区直接竞争同一物理未分配池。一旦 Container 耗尽，所有挂载卷的写入均受阻。 |
| **macOS 存储容量分级 API (URLResourceValues)** | **Verified** (Apple Developer API & 本机探针) | Apple 官方 Foundation API 明确提供多层容量查询：<br>- `volumeAvailableCapacityKey` (本机测得 2.62 GiB)<br>- `volumeAvailableCapacityForImportantUsageKey` (本机测得 2.92 GiB)<br>- `volumeAvailableCapacityForOpportunisticUsageKey` (本机测得 0.00 GiB) | 官方文档将 Opportunistic 定义为用于存储非关键资源（nonessential resources）的容量。本机测得该值为 0，狭义表明当前 API 报告已无容量可分配给投机性/非关键写入。两个 API 读数分别记录，不简单算术等同于精确 purgeable 字节数。 |
| **内存压力与虚拟内存 (VM Swap)** | **Reported local fact** (macOS 本机探针) | - 硬件物理内存：8 GB；<br>- 当前 swapusage：`total = 4096.00M, used = 3216.62M, free = 879.38M`；<br>- VM 交换卷中当前存在 4 个各 1.0GB 的 swapfile。 | 8GB 内存设备在高负载下，动态分页机制处于高度活跃状态。当前系统已实质性占用超过 3.2GB 的 swap 空间。 |
| **Creative Cloud 桌面安装器** | **Verified** (Adobe Support) | Adobe 官方文档明确指出：Creative Cloud 桌面应用程序自身安装需要 **至少 4 GB 可用硬盘空间**，且必须安装在主操作系统盘。 | CC Desktop 核心启动器不接受外置，4GB 是启动器管理程序自身的硬性底线，尚未计入任何具体创作套件。 |
| **Substance 3D Painter 空间需求** | **Verified** (Adobe Support) | Adobe 官方系统要求明确规定：<br>- 最低要求（Minimum）：**30 GB 可用 SSD 空间**；<br>- 推荐配置（Recommended）：**50 GB 可用 SSD 空间**；<br>- 极致配置（Optimal）：**70 GB 可用 SSD 空间**。 | 3D 纹理工作流涉及密集的贴图解压、虚拟纹理缓存（SVT）与网格烘焙，对磁盘可用容量与连续写入性能要求极高。 |
| **Substance 3D Painter 内存门槛** | **Verified** (Adobe Support) | Adobe 官方文档明确规定：macOS 环境下 Substance 3D Painter 的**最低硬件内存要求为 16 GB RAM**。 | **关键阻断事实**：本机仅有 8GB 统一内存，低于官方最低硬件支持门槛。 |
| **大型应用安装 Staging 重叠峰值** | **Inferred** (工程推导与经验模型) | 下载的安装包（DMG/PKG/CCX 分片）与解压后的释放目录同时存在：`Peak = DownloadSize + ExtractedSize + TargetInstallSize`。 | 对于 3~5GB 的应用包，安装瞬间的系统临时盘消耗可达 8~12GB；在安装未完全 Commit 前，空间不会释放。 |

---

## 3. 本机只读事实探针审计（Local Fact Probe Audit - 聚合脱敏）

为保护隐私并符合公开仓库安全规范，本节仅保留聚合技术事实，不收录底层设备节点编号及私有卷结构：

1. **APFS 容器与容量现状**：
   - 属于 **256GB 级内置 APFS 容器**（物理上限约 245.1 GB）；
   - 容器已分配比例约为 **99.0%**，未分配物理空间（Capacity Not Allocated）约为 **2.6 GB**；
   - 核心卷布局遵循现代 macOS 标准结构（只读 SSV 系统快照卷、Data 用户数据卷、VM 交换卷、Preboot 与 Recovery 卷）；各卷共享底层物理未分配池。
2. **容量 API 测量值（Swift `URLResourceValues` 探针）**：
   - `volumeTotalCapacity`: 约 228.27 GiB
   - `volumeAvailableCapacity` (Normal): **2.62 GiB**
   - `volumeAvailableCapacityForImportantUsage`: **2.92 GiB**
   - `volumeAvailableCapacityForOpportunisticUsage`: **0.00 GiB**
   - *说明*：当前系统报告无可用容量分配给非关键/投机性数据；两个 Available 接口测量值独立记录，不推导为精确的系统可随弃（purgeable）字节。
3. **虚拟内存与 Swap 活跃度**：
   - 物理统一内存：8 GB；
   - `sysctl vm.swapusage`：`total = 4096.00M, used = 3216.62M, free = 879.38M`；
   - VM 卷内实测存在 4 个活跃的 1.0GB swapfile，表明在 8GB 物理内存约束下，系统已产生显著的磁盘虚拟内存交换。
4. **APFS 本地快照探针**：
   - 只读探测用户 Data 卷未发现积压的 Time Machine 本地快照，表明 **2.6 ~ 2.9GiB 属于当前的真实可用空间范围，不存在隐藏快照释放后可瞬间增加几十 GB 的空间**。
5. **系统更新探针（OTA Pending Updates）**：
   - `softwareupdate --list` 报告存在 `macOS Tahoe 26.6.2` 待装更新，标称更新体积约 **3.8 GiB**（3,831,075 KiB）。

---

## 4. 当前约 2.6 ~ 2.9GiB 状态的工程定性分析

根据上述事实，当前设备的存储状态定性为：**【严重告急（Critical / Emergency）】**。

### 4.1 工程风险评估（Engineering Risk Assessment）
1. **虚拟内存交换空间极度紧缩**：
   本机物理内存仅 8GB，日常多任务极易引发内存置换。当前系统已使用超过 3.2GB 的 swap 空间。在物理剩余仅约 2.6GiB 的情况下，系统为应对突发内存压力而进一步扩展 swapfile 的空间极其受限，在密集工作负载下显著增加了前台应用被系统终止（OOM / Jetsam / Memory Pressure Termination）或执行挂起的风险。
2. **临时文件与写时复制（CoW）开销风险**：
   低剩余容量显著压缩了系统临时文件（`$TMPDIR`）、应用缓存、系统运行日志以及 APFS 写时复制（Copy-on-Write）元数据更新的运行余量。在高写入压力或磁盘接近满溢时，可能引发 I/O 阻塞或应用写入失败。
3. **更新与安装不可行性**：
   当前可用容量（2.6~2.9GiB）甚至无法完整下载 3.8GiB 的 macOS 增量更新包，更无法承接任何大型创作软件的解压暂存。

---

## 5. 分级安全余量模型（Inferred Engineering Policy for this 256GB / 8GB Machine）

> [!IMPORTANT]
> **本模型为针对当前 256GB 存储 / 8GB 内存 Mac 的【工程策略建议（Inferred Engineering Policy）】，并非 Apple 官方公布的绝对规则。** 阈值区间的设定综合考量了当前剩余容量、观测到的 4GB swap 占用、已知大型创作工具解压开销以及 macOS 升级的典型历史需求。

```mermaid
graph TD
    subgraph 工程建议安全余量梯阶 (256GB Mac / 8GB RAM)
        L1["【等级 1: 严重告急 (Critical / Emergency)】<br>< 5 GiB (当前处于 ~2.6–2.9 GiB)<br>策略: 熔断大型写入, 禁安装/更新, 优先空间恢复"]
        L2["【等级 2: 恢复观察期 (Recovery Required)】<br>5 GiB ~ 12 GiB<br>策略: 允许轻量日常开发, 仍禁止应用安装与系统更新"]
        L3["【等级 3: 最低日常运行余量 (Minimum Operating Buffer)】<br>12 GiB ~ 20 GiB<br>策略: 满足 8GB 内存日常 Swap 与浏览器/IDE 基础运行"]
        L4["【等级 4: 健康工作余量 (Healthy Working Buffer)】<br>20 GiB ~ 35 GiB (约占总盘 10%~15%)<br>策略: 具备常规应用 staging 缓冲, 支持日常创作"]
        L5["【等级 5: 升级就绪冗余 (Upgrade Ready)】<br>> 35 GiB (大版本升级建议 ≥ 45 GiB)<br>策略: 具备大版本系统升级或大型套件前置门槛余量"]
    end

    L1 -->|清理与外置分流| L2
    L2 -->|达成基础清淤| L3
    L3 -->|存储分层实施 #6| L4
    L4 -->|深度空间优化| L5
```

### 分级定义与触发策略表

| 等级名称 | 容量区间 (GiB) | 状态定性与系统特征 | 建议允许的操作 | 建议禁止的操作 | 触发治理动作 (Policy Action) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **1. Critical / Emergency<br>(严重告急)** | **< 5.0 GiB**<br>*(本机当前处于此区间: ~2.6–2.9GiB)* | **严重告急态**。<br>- VM Swap 扩展空间极度受限；<br>- Opportunistic 容量 API 读数为 0；<br>- 写入失败与内存置换受阻风险显著上升。 | - 只读事实探测；<br>- 文本编辑与代码查看；<br>- 紧急释放空间的删除/归档操作。 | - **禁止一切软件安装**；<br>- **禁止系统更新**；<br>- **禁止下载大文件/大型解压**；<br>- **禁止执行重度渲染/项目烘焙**。 | **全局熔断**。<br>系统进入只读与单向清淤模式，除清理归档脚本外，拦截任何可能引发大额写入的操作。 |
| **2. Recovery Required<br>(恢复观察期)** | **5.0 ~ 12.0 GiB** | **脆弱恢复态**。<br>- 具备一定 Swap 缓冲区；<br>- 但仍难以容纳多任务激增或中型解压 staging。 | - 正常命令行开发与轻量构建；<br>- 轻量 Git 代码提交；<br>- 常规网页浏览与文档阅读。 | - 禁止安装任何大型软件；<br>- 禁止触发系统更新；<br>- 避免打开超大型工程。 | **空间治理优先**。<br>推进 [Issue #6] 存储分层，将非活跃资产、历史依赖与大型项目有序迁移至外置存储或云端。 |
| **3. Minimum Operating<br>(最低日常运行)** | **12.0 ~ 20.0 GiB** | **最低日常运行底线**。<br>- 为 8GB 内存设备提供较为可靠的 Swap 弹性（4~8GB）；<br>- 容纳常规浏览器缓存、IDE 索引与临时小文件。 | - 高校教学与科研工作流；<br>- Web/Python 日常软件工程开发；<br>- 小型独立应用（< 500MB）安装。 | - 暂缓安装大型创作套件（Adobe、Unity Editor）；<br>- 暂缓触发系统大版本升级。 | **维持防线**。<br>属于日常运维建议保持的下限，在此区间内应维持谨慎的写入习惯。 |
| **4. Healthy Working<br>(健康工作余量)** | **20.0 ~ 35.0 GiB**<br>*(约占总盘 10% ~ 15%)* | **健康弹性工作态**。<br>- 具备承接常规应用下载解压（staging）的弹性缓冲；<br>- 适应多任务并行及临时缓存波动。 | - 允许安装常规中型工具；<br>- 允许进行日常 3D 渲染与工程缓存；<br>- 在评估后允许进行小版本系统更新。 | - 未经提前规划不建议在内置盘直接部署 30GB+ 极重型软件。 | **应用评估就绪门槛**。<br>达到此区间后，方可启动对候选应用安装的工程可行性评估。 |
| **5. Upgrade Ready<br>(升级就绪冗余)** | **> 35.0 GiB**<br>*(大版本升级建议 ≥ 45 GiB)* | **高冗余储备态**。<br>- 达到历史大版本 macOS 升级与大型生产力工具典型的前置检查空间量级。 | - 全功能常规操作；<br>- 满足系统升级对可用空间的充裕需求。 | 无特定限制（仍建议定期维护）。 | **系统升级通行门槛**。<br>在执行系统大版本跨代升级前，建议达到此余量并在外置介质做好数据备份。 |

---

## 6. 关于 Adobe Substance 3D Painter 的安装前置门槛评估

结合官方文档与本机硬件配置，对 Substance 3D Painter 进行系统化准入评估：

### 6.1 空间需求与硬件门槛解耦分析
1. **存储空间要求（Storage Requirements）**：
   - 官方最低要求：**30 GB 可用 SSD 空间**（推荐 50 GB，极致 70 GB）；
   - Creative Cloud 桌面端基础框架要求：**4 GB 可用空间**；
   - 安装下载与解压 staging：约需额外 **6 ~ 8 GB** 临时重叠空间。
2. **关键硬件不满足：内存约束（RAM Barrier）**：
   - **官方文档事实**：Adobe 明确规定 macOS 版 Substance 3D Painter 的**最低物理内存配置为 16 GB RAM**；
   - **本机稳定画像**：本项目目标设备仅具备 **8 GB 统一内存**。
3. **结论**：
   - **充足的磁盘安全余量（如达到 Level 4 ≥ 20~25GB）仅仅是安装的必要前提，但绝非充分条件**。
   - 即使内置盘通过清理释放出充裕空间，本机在硬件规格上仍处于低于官方最低配置的状态。
   - **决策**：本报告（Issue #5）严格限定于存储安全余量研究，不越界决定应用安装；关于该机器是否能在 8GB RAM 下运行 Substance Painter、是否会引发极端 swap 抖动等硬件兼容性问题，必须由 **Issue #12** 独立进行调研与评估。在 #12 形成正式结论前，严禁尝试触发安装。

---

## 7. 对后续架构与策略票据的输入规范（Inputs for #6 & #9）

本调研产出的分级余量模型与事实证据，为下游票据提供明确边界与输入：

### 7.1 对 [Issue #6: Storage Tiering Implementation](https://github.com/carllx/mac-cleanup/issues/6) 的输入：
1. **分阶段容量治理目标（Tiering Targets）**：
   - **Phase 1（紧急脱困）**：将内置盘从当前的 **2.6~2.9 GiB** 提升至 **Level 3 最低运行余量（≥ 12~15 GiB）**，缓解 Swap 扩展与临时写入压力；
   - **Phase 2（常态健康）**：通过将大型项目（如 15GB Unity Projects）、外部资产库（Blender/Painter 资源）及开发包缓存分层外置到 Samsung T7 与云端，使内置盘常态稳定维持在 **Level 4 健康工作余量（20 ~ 25 GiB）**。
2. **外置分流优先级**：
   - 优先将完全解耦、支持原生外部路径的资产与工程移至外置盘，严禁对系统级核心服务执行跨卷软链接。

### 7.2 对 [Issue #9: Operational Buffer & Emergency Cleanup Policy](https://github.com/carllx/mac-cleanup/issues/9) 的输入：
1. **预警与策略触发阈值（Policy Triggers）**：
   - **Yellow Alert（黄色预警，候选触发点：< 15 GiB）**：提示用户关注空间，允许触发低风险可再生缓存治理；
   - **Orange Alert（橙色告警，候选触发点：< 8 GiB）**：拦截新的大型软件安装与增量更新，建议挂载外置盘归档；
   - **Red Alert（红色熔断，候选触发点：< 5 GiB）**：进入紧急状态，除清理脚本外阻断一切写入，执行应急清淤手册。
2. **容量判定口径规范**：
   - 治理脚本在检查磁盘可用空间时，应使用 POSIX `statvfs`（`f_bavail`）或 Foundation `volumeAvailableCapacityKey` 作为物理真实空闲判定依据，不得单一依赖混杂了未清理 purgeable 的 Finder 视觉读数。

---

## 8. 官方技术依据与参考源（Official Sources）

本报告结论直接依托以下一手官方文档与系统技术接口：

1. **Apple 官方支持文档：macOS 存储空间与升级指导**:
   - Apple Support: *Upgrade to macOS Tahoe (Storage guidance)*  
     [https://support.apple.com/en-la/122727](https://support.apple.com/en-la/122727)
2. **Apple 官方支持文档：macOS Big Sur 历史技术规格与可用空间基准**:
   - Apple Support: *macOS Big Sur - Technical Specifications (Available storage requirements: 35.5GB to 44.5GB)*  
     [https://support.apple.com/en-in/111980](https://support.apple.com/en-in/111980)
3. **Apple Developer 文档：URLResourceValues 与存储容量分级 API**:
   - Apple Developer Documentation: *Checking Volume Storage Capacities (`volumeAvailableCapacityKey`, `volumeAvailableCapacityForImportantUsageKey`, `volumeAvailableCapacityForOpportunisticUsageKey`)*  
     [https://developer.apple.com/documentation/foundation/checking-volume-storage-capacity](https://developer.apple.com/documentation/foundation/checking-volume-storage-capacity)
4. **Apple 官方技术规范：关于 APFS 容器空间共享机制**:
   - Apple Support: *Use Disk Utility to add APFS volumes and understand shared container space*  
     [https://support.apple.com/guide/disk-utility/add-apfs-volumes-dskuae1050c4/mac](https://support.apple.com/guide/disk-utility/add-apfs-volumes-dskuae1050c4/mac)
5. **Adobe 官方系统需求：Substance 3D Painter 硬件、内存与可用空间标准**:
   - Adobe Substance 3D Documentation: *Substance 3D Painter System Requirements (Minimum 16GB RAM for macOS; 30GB Minimum, 50GB Recommended, 70GB Optimal SSD space)*  
     [https://experienceleague.adobe.com/en/docs/substance-3d-painter/using/getting-started/system-requirements](https://experienceleague.adobe.com/en/docs/substance-3d-painter/using/getting-started/system-requirements)
6. **Adobe 官方系统需求：Creative Cloud 桌面应用程序安装与空间规范**:
   - Adobe Help Center: *Adobe Creative Cloud desktop app system requirements (4GB available hard-disk space)*  
     [https://helpx.adobe.com/creative-cloud/system-requirements.html](https://helpx.adobe.com/creative-cloud/system-requirements.html)
7. **Adobe 官方故障排查：错误 179（Error 179 - 本地主驱动器安装限制）**:
   - Adobe Help Center: *Troubleshoot Adobe Creative Cloud install issues - Error 179*  
     [https://helpx.adobe.com/creative-cloud/kb/troubleshoot-download-install-logs.html#error179](https://helpx.adobe.com/creative-cloud/kb/troubleshoot-download-install-logs.html#error179)
