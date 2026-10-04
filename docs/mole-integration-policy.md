# Mole Integration & Execution Authority Policy

本文档规范开源系统清理与优化引擎 **Mole** 在 `mac-cleanup` 项目中的受控集成架构、权限边界、上游知识源流转机制以及与现有运行手册（Runbook）的职责划分。

---

## 1. 背景与准入动因 (Context & Rationale)

在历史运行中，本机曾部署闭源商业软件 CleanMyMac X（版本 4.14.2），该软件存在后台常驻守护进程较多、图形界面深度遍历易引发系统无障碍树挂起与内存异常膨胀、以及缺乏受控 CLI 接口等问题。

为达成“**透明可审计、脚本化交互、最小特权原则与持久可用缓冲**”的工程目标，本项目完成了从 CleanMyMac 到现代开源 CLI 工具 **Mole** 的受控迁移，并将 Mole 正式确立为 `mac-cleanup` 的**外部执行引擎 (External Execution Engine)** 与**上游研究知识源 (Research Upstream)**。

---

## 2. 上游定位器与已测试不可变元数据 (Upstream Locator & Provenance)

为确保执行环境的可重现性与审计溯源，记录当前基准版本的不可变元数据：

- **工具名称**：Mole CLI（对应命令 `mo` / `mole`）
- **官方主页**：[https://mole.fit](https://mole.fit)
- **开源代码仓库**：[tw93/Mole](https://github.com/tw93/Mole)
- **当前测试稳定版本**：`1.57.0`
- **发布标签 (Release Tag)**：`V1.57.0`
- **不可变上游提交 (Immutable Upstream Commit)**：`6bca4812acd6a3d54ffe97291734c3556a174057`
- **安装通道**：Homebrew Stable Formula (`brew install mole`)
- **二进制执行路径**：`/opt/homebrew/bin/mo`（符号链接指向 `/opt/homebrew/Cellar/mole/1.57.0/bin/mo`）
- **更新策略 (Non-Goal)**：禁止自动静默更新；禁止追踪 nightly/main 分支；版本升级需经过独立的兼容性验证与门禁审批。

---

## 3. 架构主权体系：运行时权威 vs 研究上游 (Runtime Authority vs Research Upstream)

在 `mac-cleanup` 项目中，严格确立以下权限与决策边界：

```text
               ┌─────────────────────────────────────────────────────────┐
               │              Upstream Research (Mole / tw93)            │
               │  - macOS 新特性清理规则 / 启发式探测模型 / 优化实践     │
               └───────────────────────────┬─────────────────────────────┘
                                           │ 提供探测能力与规则参考 (只读输入)
                                           ▼
┌────────────────────────────────────────────────────────────────────────┐
│                   Runtime Authority (mac-cleanup 本地权威)              │
│  - 唯一最高决策权 (Single Source of Truth)                              │
│  - 强制执行 HIGH RISK / REVIEW FIRST / LOW RISK 策略最高优先级          │
│  - 外部扫描结果一律降级为 Cleanup Candidate                            │
│  - 严禁外部工具享有静默执行或自动删除特权                               │
└──────────────────────────────────┬─────────────────────────────────────┘
                                   │ 严格门禁批复后受控下发
                                   ▼
┌────────────────────────────────────────────────────────────────────────┐
│               External Execution Engine (Mole CLI 实例)                 │
│  - 受控执行探测 (mo analyze --json / mo clean --dry-run)                │
│  - 经人工授权后执行原子级清理/优化动作                                 │
└────────────────────────────────────────────────────────────────────────┘
```

1. **运行时权威唯一性 (Runtime Authority Sole Ownership)**：
   `mac-cleanup` 代码库、策略文件与 Runbook 是本机治理的唯一主权代表。Mole 仅为受控的外部能力提供者，绝不拥有自主治理权。
2. **上游候选强制降级 (Cleanup Candidate Downgrade)**：
   Mole 输出的所有扫描项目（包含 `mo clean`、`mo analyze`、`mo purge` 等产生的列表），在本项目中严格定级为 **Cleanup Candidate（清理候选）**。任何外部工具的一键标记、默认勾选在本项目中均无权直接触发真实删除（Mutation）。
3. **安全分级最高优先级 (Policy Precedence)**：
   本项目的 `HIGH RISK / REVIEW FIRST / LOW RISK` 防护分级绝对优先于 Mole 的分类。例如：即使 Mole 判定某开发项目为“旧构建产物”，若该路径触及本机的核心文献库（Zotero/Calibre）、教学源码、未备份分支或个人工作区，本土策略拥有绝对一票否决权，严禁误删。

---

## 4. 运行手册职责映射：三分类治理 (Three-Way Architecture Mapping)

对照现行 `docs/storage-health-and-cleanup-runbook.md`，将各项清理与健康检查逻辑划分为三类：

| 治理归属 | 适用范畴与具体项目 | 架构设计考量 |
| :--- | :--- | :--- |
| **`DELETE`**<br>*(本土废弃 / 淘汰)* | • 原 CleanMyMac 专用维护脚本与专有配置探测<br>• 本土硬编码维护的通用应用 Cache/Log 静态绝对路径列表<br>• 无 CLI 接口的手工 GUI 点击指引 | 避免本土策略退化为易失的路径碎片黑名单，将通用应用生态的缓存变动追踪委托给活跃开源上游。 |
| **`DELEGATE_TO_MOLE`**<br>*(委托给 Mole 处理)* | • 系统标准缓存与日志清理（`~/Library/Caches`, `~/Library/Logs`）<br>• 孤立应用支持残留（已卸载 App 的残留支持文件）探测<br>• 安全系统优化（DNS 缓存刷新、QuickLook 缩略图刷新、IconServices 缓存重建）<br>• 孤立安装镜像探测（`mo installer`）<br>• 磁盘空间大文件快照（`mo analyze --json`） | 充分发挥 Mole 在 macOS 原生环境下的高效探测、垃圾箱安全缓冲（Trash-First）与原子执行能力。所有委托操作执行前必须加 `--dry-run` 预览。 |
| **`KEEP_LOCAL`**<br>*(本土保留核心主权)* | • **健康缓冲与停工基准**：日常 >= 20 GiB (GREEN)、大型安装准备 >= 60 GiB、学期储备 70–80 GiB 的工程防御缓冲模型<br>• **高价值核心资产保护**：Zotero、Calibre、Obsidian、微信/企微数据库、教学源码的绝对禁动门禁<br>• **跨介质分层存储与迁移**：Tier 1 (内置 SSD) 与 Tier 2 (外置 T7) 的分层模型与 6 步流式哈希核验迁移协议（Verified Migration Protocol）<br>• **开发环境拓扑与包管理器规范收敛**：Python/Conda/Node/NVM 的动态链接分析（`otool -L`）与零消费者门禁（Zero-Consumer Gate）<br>• **工程代码目录边界保护**：`mo purge` 默认禁止对主开发目录执行盲删，必须由本土显式配置隔离白名单 | 涉及机器架构、运行时兼容性、科研教学命脉以及多介质物理分布的决策，必须全部由本土工程策略掌控。 |

---

## 5. 迁移与只读探测实证基线 (Baseline & Verification Evidence)

### 5.1 卸载前基线 (Pre-Uninstall Baseline)
- **启动盘物理可用空间**：`43,014,852 KiB`（~`41.02 GiB` / 80% 容量占用，处于日常 GREEN 状态）
- **卸载目标**：CleanMyMac X（版本 `4.14.2`，Bundle ID `com.macpaw.CleanMyMac4`，独立 DMG 安装源）
- **检测到的关联服务与特权项**：
  - 用户进程：`com.macpaw.CleanMyMac4.HealthMonitor`、`com.macpaw.CleanMyMac4.Menu`
  - 特权进程：`/Library/PrivilegedHelperTools/com.macpaw.CleanMyMac4.Agent`（PID 4788）
  - 启动项配置：`/Library/LaunchDaemons/com.macpaw.CleanMyMac4.Agent.plist`

### 5.2 卸载后状态复核与残留只读分类 (Post-Uninstall Audit)
- **主程序状态**：`/Applications/CleanMyMac X.app` 已彻底移除（`app absent`）。
- **进程状态**：相关常驻后台进程及特权 Helper 全部安全终止（`relevant processes absent`）。
- **残留组件分类盘点（遵照只读分类原则，严禁手工盲目强删）**：
  - *孤立应用配置 (REVIEW FIRST)*：`~/Library/Application Support/CleanMyMac X*`、`~/Library/Preferences/com.macpaw.CleanMyMac4*`
  - *历史日志与崩溃记录 (LOW RISK)*：`~/Library/Logs/CleanMyMac X Menu`、`~/Library/Logs/com.macpaw.CleanMyMac4/`
  - *系统级特权项*：`/Library/LaunchDaemons` 及 `/Library/PrivilegedHelperTools` 中的 MacPaw 项已清空。

### 5.3 Mole 首轮只读 Dry-Run 探测结果 (Probe Evidence)
- `mo clean --dry-run`：
  - 识别出 7 个类别的可清理候选，潜在可用空间约 `6.97 GB`，共 `7,830` 个候选文件。
  - 候选清单安全物理隔离于本地 `~/.config/mole/clean-list.txt`，未对系统产生任何真实修改。
- `mo optimize --dry-run`：
  - 诊断出当前性能与内存压力（WindowServer 渲染负荷、Swap 交换分区使用状态）。
  - 预览 4 项安全优化动作，未写入任何配置。
- `mo purge --dry-run`：
  - 探测到 10 个默认扫描工程目录，明确识别出 `node_modules` 与 `.venv`。因其默认范围触及复杂工作区，已明确建立本地边界保护，禁止在未指定子工程时盲目执行。
- `mo history --json`：
  - 确凿实证：`actions.removed = 0`，`actions.trashed = 0`，`deletions = []`，完全符合只读审计门禁。
- **最新物理可用空间**：`42,310,092 KiB`（~`40.35 GiB`，稳健保持在日常 GREEN 缓冲范围）。

---

## 6. Gemini Spark 上游监控者职责规范 (Upstream Monitoring Owner)

为确保持续吸收 Mole 上游的高价值规则，定义 **Gemini Spark** 作为逻辑上的上游监控所有者：

1. **逻辑职责**：
   - 定期比对 `tw93/Mole` 仓库的 Release Note、Tag 变更与规则更新。
   - 提取新版针对 macOS 新增架构（如 Tahoe / Sequoia）的缓存特征与安全优化建议，向 `mac-cleanup` 提炼为规则候选（Rule Promotion）。
2. **安全与隐私硬边界**：
   - 本地代码库、提交记录及公开文档中**严禁保存任何 Google 凭据、私有密钥或 Token**。
   - 本地 IDE 与 Agent **不负责创建、配置或管理 Spark 的底层自动化运行调度器 (No Local Scheduler Management)**。所有监控活动为松耦合的外部只读审视。

---

## 7. 后续演进门禁 (Subsequent Gate)

- 本次集成以**完成卸载迁移、建立只读基线、规范项目权威集成并推送版本库**为阶段结束标志（Gate Closed）。
- **真实 Mole 清理（Real Mutation）作为独立的下一阶段授权 Gate**：在未获得用户针对具体候选批次的显式批复前，严禁在自动化脚本中移除 `--dry-run` 保护。
