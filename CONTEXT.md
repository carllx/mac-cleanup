# Domain Glossary & Language (CONTEXT.md)

本文档定义 `mac-cleanup` 项目专用的领域语言与实体概念，确保所有开发、治理和 Agent 交互使用统一定义的术语，避免概念漂移。

---

## 1. 状态与事实范畴

### Dynamic Snapshot（动态快照）
某一具体时间点针对系统或目录状态的只读切片。包含易失的绝对字节数、特定时刻的目录列表及临时日志。属于短期参考数据，必须物理隔离于本地（`local/`），不可作为持久性事实对待。

### Stable Machine Fact（稳定机器事实）
描述当前设备在数月甚至数年尺度上保持稳定的低频变动属性（如芯片架构、标称硬盘物理等级、APFS 容器总体积、主要业务负载场景）。收录于 `docs/machine-profile.md`。

---

## 2. 治理目标与规则范畴

### Cleanup Candidate（清理候选）
在某一治理周期中被探测识别出、具备释放空间潜力的目标实体（如特定包缓存、解压残留文件、浏览器缓存或废弃副本）。在未完成风险定级与审批前，不可执行任何变更。

### Validated Cleanup Rule（已验证清理规则）
经过实际执行并满足完整验证闭环的清理操作模式：具备明确的前置条件、非破坏性清理指令，且已实证确认该操作能有效释放空间同时绝不破坏关联应用的正常工作。

### Knowledge Promotion（知识晋升）
将临时观察（Dynamic Snapshot）或经验推断（Hypothesis），通过严格的因果验证与安全测试，提炼并升格为通用工程规范（如 Validated Cleanup Rule 或 Policy Candidate）的过程。

---

## 3. 安全分级标准

### LOW RISK（低风险）
指完全由系统或应用自动生成、且具备完全可再生性（Re-creatable）的数据。包括构建包缓存（如 `uv cache`）、下载器临时分片、应用网络缓存、已被证实且安装完毕的废弃 `.dmg`。清理这些数据绝不造成用户资产丢失或功能受损。

### REVIEW FIRST（需复核）
指可能包含历史有价值内容、或清理后可能需要用户重新配置/登录、或多份副本中需要人工指定主副本的实体。例如：多重 Conda 环境去重、历史项目备份快照、网盘离线缓存、大版本更新前的应用支持文件。必须经用户显式确认后方可处理。

### HIGH RISK（高风险 / 绝对禁动）
指用户的唯一真实工作资产、聊天记录数据库、个人文献图书库、核心配置及操作系统关键组件。例如：微信/企微核心 SQLite 数据库与聊天附件、语音备忘录录音原件、Zotero 与 Calibre 库、未备份的教学项目源码。在任何情况下均严禁自动删除或静默修改。

---

## 4. 架构治理动作

### Workspace Organization（工作区归属治理）
针对 Downloads、Documents 及桌面零散文件的生命周期管理动作。通过建立分类标准、将短期项目归档至外部介质或将游离资产收敛至规范目录，以达成减少主盘压力、提升系统条理的目的。“整理文件归属”在项目中与“物理删除”同属于清理范畴。

### Environment Canonicalization（开发环境规范收敛）
针对同类开发工具链（如 Python、Node、CLI）存在的多实例、重复包管理器、冗余虚拟环境进行统一标准化的过程。旨在确立单一事实来源（Single Source of Truth），避免同一工具在系统中被多次安装与占用空间。

---

## 5. 外部能力与知识上游范畴

### 三分法权威治理模型 (Three-Way Authority Model)
1. **Project Policy Authority（项目策略权威）**：`mac-cleanup` 本地代码库与治理规范拥有全部授权决策、风险定级（`HIGH RISK / REVIEW FIRST / LOW RISK`）与一票否决权，是机器状态治理的唯一最高权威。
2. **Tested Mole Runtime Dependency（已测试的 Mole 运行时依赖）**：本机通过 Homebrew 安装并固定的 Mole CLI 二进制（当前测试版本 `1.57.0`），拥有其实际的命令执行语义与物理行为（如 `mo clean` 默认物理删除、`mo uninstall` 默认移入废纸篓等）。
3. **Research Upstream（研究上游）**：`tw93/Mole` 开源社区与代码库，仅作为咨询、启发式规则探索与 macOS 新机制洞察的参考输入，不具备对本机的直接控制权。

### Observation / Telemetry（观察与遥测）vs Cleanup Candidate（清理候选）
外部引擎的输出不能无差别一概归入清理候选，必须严格区分：
- **Observation / Telemetry（系统观察与遥测）**：`mo status` 的健康评分、CPU/内存/磁盘读数，以及 `mo analyze` 输出的目录体积行，均属于只读观察事实，用于机器状态评估，不直接构成清理候选。
- **Cleanup Candidate（清理候选）**：`mo clean`、`mo uninstall`、`mo purge`、`mo installer` 及 `mo optimize` 显式列出的拟删除、拟优化目标实体，在未完成本地策略审查与人工批复前，属于受限的清理候选，严禁自动执行。

### 命令特定执行语义 (Command-Specific Execution Semantics)
Mole 工具不同子命令的删除与恢复语义存在根本差异，禁止将其泛化概括为“废纸篓缓冲”：
- **`mo clean`**：默认执行**直接永久删除 (Permanent deletion by default)**。因此，`--dry-run` 预览是至关重要的安全防护门禁。
- **`mo uninstall`**：默认**移入系统废纸篓 (Trash by default)**，除非显式指定 `--permanent`。
- **`mo analyze`**：交互界面中由用户选定的删除项目会路由至系统废纸篓。
- **非破坏性预览快照**：`mo clean --dry-run` 会向磁盘写入本地预览清单 `~/.config/mole/clean-list.txt`，该操作属于非破坏性只读预览，但涉及该临时快照文件的本地写入。

### Upstream Monitoring Owner（上游监控者角色）
负责周期性追踪外部引擎上游动态、新版本发布与安全通告的逻辑角色（如 Gemini Spark）。其职责严格限定于只读知识跟踪与差异提醒，不持有凭据，不直接调度或执行系统变更。
