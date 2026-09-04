# Machine Profile

本文档记录目标机器的相对稳定客观属性与系统级约束，供所有 Agent 与治理策略参考。

> [!NOTE]
> 本文件仅收录非敏感、低频变动的硬件与生态事实。具体的目录大小、剩余字节数及包含用户名的绝对路径属于动态快照（Dynamic Snapshot），绝不在此收录。

---

## 硬件与系统基线

| 维度 | 事实与约束说明 | 验证时间（Last Verified） |
| :--- | :--- | :--- |
| **芯片与架构** | Apple Silicon M1 (ARM64) | 2026-09-04 |
| **物理内存** | 8 GB 统一内存（RAM 限制使磁盘 Swap 空间对系统稳定性至关重要） | 2026-09-04 |
| **标称磁盘容量** | 256 GB 级别内置固态硬盘 | 2026-09-04 |
| **可用 APFS 容器** | 约 245 GB（APFS Container 物理上限） | 2026-09-04 |
| **核心系统约束** | **存储容量为长期强约束（Persistent Operating Constraint）**。系统对磁盘满溢极度敏感，必须保持安全余量。 | 2026-09-04 |

---

## 主要工作负载（Workloads）

本设备长期服务于多重交叉的工作流，治理策略必须确保这些核心负载不受破坏：

1. **高校教学与科研（Teaching & Academic Research）**：
   - 包含课程课件、多媒体课徒资料、学生课题材料；
   - 依赖 Zotero（学术文献与全文附件）与 Calibre（图书库）持续运转。
2. **多语言软件开发（Software Engineering）**：
   - 涉及 Web、数据处理、自动化脚本与 CLI 工具研发；
   - 活跃工程主要托管于本地代码仓库与 GitHub。
3. **AI 智能体与自动化研发（AI-Agent & Autonomous Tools）**：
   - 运行 Antigravity、IDE AI 辅助插件及 Agent 自动化调用；
   - 依赖快速的包下载、依赖解析与本地日志追踪。

---

## 观察到的主要开发环境生态（Observed Toolchains）

本设备观察到以下并存的技术栈生态（具体路径按各包管理器标准约定存放）：

- **系统包管理**：Homebrew (`/opt/homebrew`)
- **Python 生态**：
  - `uv` 作为现代化高效包安装与缓存工具；
  - 存在 Conda 家族的多套运行环境（包含数据科学与 R 语言关联的 miniconda 实例）；
  - Python 3 用户全局库。
- **JavaScript / TypeScript 生态**：
  - Node.js 运行时及全局管理工具（NVM、npm 全局包体系）。
- **主流代码编辑器与 IDE**：
  - Visual Studio Code、Windsurf、Antigravity IDE。

---

## 稳定性与变更规则

- 当 macOS 主版本升级、硬件变更或确立了新的技术栈标准时，更新本文档对应字段并修订 `Last Verified`；
- 严禁将单次探测到的剩余可用空间、具体文件体积或用户个人路径硬编码至此。
