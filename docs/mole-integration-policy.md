# Mole Integration & Execution Authority Policy

本文档规范开源系统清理与优化引擎 **Mole** 在 `mac-cleanup` 项目中的受控集成架构、权限边界、上游知识源流转机制以及与现有运行手册（Runbook）的职责划分。

---

## 1. 背景与准入动因 (Context & Rationale)

在历史运行中，本机曾部署闭源商业软件 CleanMyMac X（版本 4.14.2）。该软件作为重型图形界面套件，存在后台常驻守护组件较多、缺乏适合自动化审计的标准化 CLI 接口等局限。在本次替代验证中，通过 AppleScript 无障碍树探测该定制界面亦引发了系统端 `System Events` 的句柄膨胀与超时，进一步表明非受控 GUI 软件不适宜作为可追溯工程治理的基础设施。

为达成“**透明可审计、脚本化交互、最小特权原则与持久可用缓冲**”的工程目标，本项目决定将 CleanMyMac 安全下线，引入现代开源 CLI 工具 **Mole**，并正式建立其在 `mac-cleanup` 中的角色定位与协作规范。

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

## 3. 架构主权体系：三分法治理模型 (Three-Way Authority Architecture)

在 `mac-cleanup` 项目中，严格确立三方分离的治理体系：

```text
┌────────────────────────────────────────────────────────────────────────┐
│                   1. Project Policy Authority (项目策略权威)            │
│  - 本地 mac-cleanup 代码库与运行手册 (Single Source of Truth)           │
│  - 拥有全部授权决策、HIGH RISK / REVIEW FIRST / LOW RISK 定级与一票否决权│
└──────────────────────────────────┬─────────────────────────────────────┘
                                   │ 严格门禁审查与策略过滤
                                   ▼
┌────────────────────────────────────────────────────────────────────────┐
│             2. Tested Mole Runtime Dependency (已测试运行时依赖)         │
│  - 本机固定的 Mole 1.57.0 CLI 二进制执行实体                            │
│  - 拥有其实际命令级执行语义 (clean 默认物理删除 / uninstall 默认进废纸篓)│
│  - 执行非破坏性预览探测与授权后的原子操作                               │
└────────────────────────────────────────────────────────────────────────┘
                                   ▲
                                   │ 参考与知识输入 (只读 / 咨询性质)
┌──────────────────────────────────┴─────────────────────────────────────┐
│                     3. Research Upstream (研究上游)                     │
│  - tw93/Mole 开源社区、代码库与规则集                                   │
│  - 提供 macOS 新特性清理规则、启发式模型与维护建议                      │
└────────────────────────────────────────────────────────────────────────┘
```

1. **Project Policy Authority（项目策略权威）**：
   本地策略体系拥有最高且唯一的解释权与决策权。任何外部探测结果必须接受本土安全策略的严格检视。
2. **Tested Mole Runtime Dependency（已测试运行时依赖）**：
   作为受控下发的执行层，明确遵循其具体的底层执行语义。严禁对 Mole 的恢复机制做宽泛化假设。
3. **Research Upstream（研究上游）**：
   仅作为规则输入与研究参考，不具备对本机的运行时控制权。

---

## 4. 实体属性区分：观察值 vs 清理候选 (Observations vs Cleanup Candidates)

严禁不加区分地将外部工具的所有输出统称为“清理候选”：

- **System Observation / Telemetry（系统观察与遥测）**：
  - `mo status --json` 采集的主机健康分、CPU 负载、内存/交换分区利用率、磁盘挂载等度量，属于系统状态的只读事实观察。
  - `mo analyze --json` 输出的文件系统各层级占用大小，属于结构化的容量分布观察，不自动构成清理候选。
- **Cleanup Candidate（清理候选）**：
  - 仅指代 `mo clean`、`mo uninstall`、`mo purge`、`mo installer` 及 `mo optimize` 显式识别出的具体拟删除或拟优化目标文件/路径。
  - 清理候选在获得本土策略核准与用户明确授权前，绝对禁止执行真实变更（Mutation）。

---

## 5. 命令特定执行与恢复语义 (Command Deletion & Recovery Semantics)

根据 `tw93/Mole @ V1.57.0` 规范，Mole 各命令具有完全不同的物理删除与恢复特征，**切勿将 Mole 清理泛化理解为“废纸篓缓冲 (Trash-First)”**：

1. **`mo clean`**：
   - **默认行为为永久直接删除 (Permanent deletion by default)**，目标文件不会进入 macOS 废纸篓。
   - **安全要求**：必须以 `--dry-run` 预览作为强制前置门禁；`mo clean --dry-run` 会生成本地预览清单 `~/.config/mole/clean-list.txt`，供人工与规则逐项复核。
2. **`mo uninstall`**：
   - **默认行为为移入系统废纸篓 (Trash by default)**，具备一定可恢复性；
   - 只有显式添加 `--permanent` 标志时才会彻底物理粉碎。
3. **`mo analyze`**：
   - 交互模式下由用户显式标记并确认删除的项目，默认路由至废纸篓。
4. **`mo purge` 治理策略准则 (Purge Policy Guardrails)**：
   - 本项目不预设未经测试的技术拦截层，而是通过**严格的工程策略进行治理规范**：
     - **严禁无范围限制的全局真实清理 (No unscoped real `mo purge`)**；
     - 任何执行必须以 `--dry-run` 为第一步；
     - 涉及工程构建物清理时，必须优先使用显式指定的 `--paths` 参数限定边界；
     - 用户创作项目、科研教学源码与高价值代码仓库始终受 `mac-cleanup` 最高策略防护，严禁整盘粗暴扫描。

---

## 6. 运行手册职责映射：三分类治理 (Three-Way Architecture Mapping)

对照现行 `docs/storage-health-and-cleanup-runbook.md`，将各项清理与健康检查逻辑划分为三类：

| 治理归属 | 适用范畴与具体项目 | 架构设计考量 |
| :--- | :--- | :--- |
| **`DELETE`**<br>*(本土废弃 / 淘汰)* | • 原 CleanMyMac 专用维护脚本与专有配置探测<br>• 本土硬编码维护的通用应用 Cache/Log 静态绝对路径列表<br>• 无 CLI 接口的手工 GUI 流程 | 避免本土策略退化为易失的路径碎片黑名单，将通用应用生态的缓存变动追踪委托给活跃开源上游。 |
| **`DELEGATE_TO_MOLE`**<br>*(委托给 Mole 处理)* | • 系统标准缓存与日志扫描（以 `mo clean --dry-run` 预览为前提）<br>• 孤立应用支持残留（已卸载 App 的残留支持文件）探测<br>• 安全系统状态刷新（DNS、Spotlight、QuickLook 缩略图等，`mo optimize`）<br>• 孤立安装包镜像探测（`mo installer`）<br>• 磁盘空间大文件快照（`mo analyze --json`） | 充分发挥 Mole 在 macOS 原生环境下的高效探测与原子执行能力。所有委托操作执行前必须进行非破坏性预览。 |
| **`KEEP_LOCAL`**<br>*(本土保留核心主权)* | • **健康缓冲与停工基准**：日常 >= 20 GiB (GREEN)、大型安装准备 >= 60 GiB、学期储备 70–80 GiB 的工程防御缓冲模型<br>• **高价值核心资产保护**：Zotero、Calibre、Obsidian、微信/企微数据库、教学源码的绝对禁动门禁<br>• **跨介质分层存储与迁移**：Tier 1 (内置 SSD) 与 Tier 2 (外置 T7) 的分层模型与 6 步 Verified Migration Protocol<br>• **开发环境拓扑与包管理器规范收敛**：Python/Conda/Node/NVM 的动态链接分析（`otool -L`）与零消费者门禁（Zero-Consumer Gate）<br>• **工程代码目录边界保护**：遵循上述 `mo purge` 策略，严防误删活跃工作区依赖 | 涉及机器架构、运行时兼容性、科研教学命脉以及多介质物理分布的决策，必须全部由本土工程策略掌控。 |

---

## 7. 迁移与非破坏性探测实证基线 (Baseline & Verification Evidence)

### 7.1 卸载前基线 (Pre-Uninstall Baseline)
- **启动盘物理可用空间**：`43,014,852 KiB`（~`41.02 GiB` / 80% 容量占用，处于日常 GREEN 状态）
- **卸载目标**：CleanMyMac X（版本 `4.14.2`，Bundle ID `com.macpaw.CleanMyMac4`，独立 DMG 安装源）
- **关联常驻服务与特权项**：
  - 用户进程：`com.macpaw.CleanMyMac4.HealthMonitor`、`com.macpaw.CleanMyMac4.Menu`
  - 特权进程：`/Library/PrivilegedHelperTools/com.macpaw.CleanMyMac4.Agent`（PID 4788）
  - 启动项配置：`/Library/LaunchDaemons/com.macpaw.CleanMyMac4.Agent.plist`

### 7.2 卸载实施核验与偏离说明 (Uninstaller Evidence: DEVIATION)
- **执行方式核验 (DEVIATION)**：
  - 原计划通过 CleanMyMac 自带内置卸载器进行移除；
  - 实际执行中，因无交互式 GUI 且无障碍树探测阻塞，经用户确认授权，偏离原路径，改由**受控终端下线流程 (Controlled Terminal Decommissioning)** 执行：安全注销并终止用户级 LaunchAgents 与常驻进程，移除 `/Applications/CleanMyMac X.app` 包，确认特权 Helper 终止。
- **卸载后状态复核**：
  - 主程序：`/Applications/CleanMyMac X.app` 已彻底移除（`app absent`）。
  - 常驻进程：MacPaw 相关监控与菜单进程全部退出（`relevant processes absent`）。
  - 系统特权项：`/Library/LaunchDaemons` 及 `/Library/PrivilegedHelperTools` 中的对应项已清空。
- **残留组件分类盘点（遵照只读分类原则，严禁手工盲目删除存疑项）**：
  - *孤立应用配置 (REVIEW FIRST)*：`~/Library/Application Support/CleanMyMac X*`、`~/Library/Preferences/com.macpaw.CleanMyMac4*`
  - *历史日志与崩溃记录 (LOW RISK)*：`~/Library/Logs/CleanMyMac X Menu`、`~/Library/Logs/com.macpaw.CleanMyMac4/`

### 7.3 Mole 机器可读探测与非破坏性预览实证 (Probes Evidence)
所有首轮探测均以非破坏性方式执行，未执行任何真实删除：
1. **`mo status --json` (系统遥测观察)**：
   - 采集到单次健康快照（健康分 80/Good，内存使用 7.0GB，Swap 占用 1.9GB，根磁盘使用率 81.9%）。
2. **`mo analyze --json` (磁盘占用观察与覆盖度报告)**：
   - 探测总大小约 `8.48 GB`；
   - **扫描覆盖度披露**：如实反映扫描状态为 **`"scan_status": "partial"`**，其中 Xcode Simulators、Homebrew Cache、System Logs 报告为 `complete`，Home 与 Downloads 报告为 `partial`，而 `/Applications` 与 `/Library` 等系统目录因受限而呈现 `unavailable`。不将进程成功退出误判为全盘全量覆盖。
3. **`mo clean --dry-run` (清理候选预览)**：
   - 识别出 7 个类别的可清理候选，估算潜在空间约 `6.97 GB`（共 `7,830` 个候选文件）。
   - 候选文件列表完整保存在本地物理隔离路径 `~/.config/mole/clean-list.txt`，未对目标路径执行任何删除。
4. **`mo optimize --dry-run` (系统优化预览)**：
   - 预览 4 项安全优化动作，零配置修改。
5. **`mo purge --dry-run` (构建产物候选探测)**：
   - 识别出默认扫描工程目录边界，已确立本地策略保护准则。
6. **`mo history --json` (操作历史实证)**：
   - 实证确认：`actions.removed = 0`，`actions.trashed = 0`，`deletions = []`，确凿证明无任何目标被物理删除或移入废纸篓。
- **最新物理可用空间**：`42,310,092 KiB`（~`40.35 GiB`，稳健维持在日常 GREEN 缓冲范围）。

---

## 8. Gemini Spark 上游监控者职责规范 (Upstream Monitoring Owner)

为确保持续吸收 Mole 上游的高价值规则，定义 **Gemini Spark** 作为逻辑上的上游监控所有者：

1. **逻辑职责**：
   - 定期比对 `tw93/Mole` 仓库的 Release Note、Tag 变更与规则更新。
   - 提取新版针对 macOS 新增架构的缓存特征与安全优化建议，向 `mac-cleanup` 提炼为规则候选（Rule Promotion）。
2. **安全与隐私硬边界**：
   - 本地代码库、提交记录及公开文档中**严禁保存任何 Google 凭据、私有密钥或 Token**。
   - 本地 IDE 与 Agent **不负责创建、配置或管理 Spark 的底层自动化运行调度器 (No Local Scheduler Management)**。所有监控活动为松耦合的外部只读审视。

---

## 9. 后续演进门禁 (Subsequent Gate)

- 本次集成以**完成下线替代、建立非破坏性基线、规范项目权威集成并推送版本库**为阶段结束标志（Gate Closed）。
- **真实 Mole 清理（Real Mutation）作为独立的下一阶段授权 Gate**：在未获得用户针对具体候选批次的显式批复前，严禁在自动化脚本中移除 `--dry-run` 保护。
