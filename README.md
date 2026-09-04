# mac-cleanup

> 针对容量受限 Mac 的长期存储与工作区管家（Long-term Storage & Workspace Steward）。

## 核心使命（Mission）

本项目**不是**一个固定的脚本式一键垃圾清理器（Script-based Cleaner），而是一个以证据为驱动、持续治理的 **Storage / Workspace Steward**。

在 256GB 存储受限的硬件约束下，通过结构化的 Operating Model（`Observe → Diagnose → Cleanup / Organize → Verify → Promote durable knowledge`），达成以下长期目标：

1. **保障基础可用性**：维持足够的磁盘剩余空间，确保日常应用（如通信、浏览器、工作套件）稳定运行，杜绝因磁盘空间耗尽导致的进程崩溃与死锁；
2. **保障可扩展性**：确保系统具备即时安装新应用、拉取开发依赖及解压大型软件包的吞吐空间；
3. **保障可维护性**：确保系统随时有充足冗余空间执行 macOS 系统更新与固件升级；
4. **工作区持续治理**：持续理清 `Downloads`、`Documents` 与项目工作区的文件归属与生命周期，区分核心资产、历史归档与可丢弃缓存；
5. **开发环境与运行时治理**：持续监控 package manager、cache、虚拟环境与重复 runtime（如 Conda、Python、Node、Homebrew），消除同一工具或依赖的重复冗余安装；
6. **知识持久化沉淀**：在每一次真实的探测与治理后，将经过实证检验的安全规则沉淀为可复用的工程规则，实现长治久安。

---

## 核心操作模型（Operating Model）

所有清理与整理流程严格遵循五步闭环：

```
Observe (重新探测) → Diagnose (多维诊断) → Cleanup / Organize (决策执行) → Verify (实证复验) → Promote (规则沉淀)
```

- **动态决策**：不假设上一轮的最大空间消耗源本轮依然最大；
- **分级防护**：严格划分 `LOW RISK`、`REVIEW FIRST` 与 `HIGH RISK`；
- **整理即清理**：文件归属整理、开发环境去重与垃圾删除具有同等治理价值；
- **证据优先**：只有在实证支持下确认因果的规则，才能晋升为长期规则。

详见 [docs/operating-model.md](docs/operating-model.md)。

---

## 隐私与安全边界（Privacy & Safety Boundary）

本代码仓库为公开仓库（Public Repository）。为了严格保护个人隐私与机器安全：

- **代码与规则公开**：通用的治理逻辑、标准词汇、抽象模型与可复用规则公开透明；
- **机器遥测与快照绝对本地化**：所有包含用户个人路径、用户名、具体文件清单、微信/企业微信敏感数据、具体体积快照的诊断产物，严格隔离于本地专属区域（如 `local/`），并通过 `.gitignore` 彻底排除，绝不提交至版本库。

---

## 目录索引

- [CONTEXT.md](CONTEXT.md)：项目领域语言、核心实体及状态定义。
- [docs/operating-model.md](docs/operating-model.md)：长期运行模型、分级标准与知识晋升机制。
- [docs/machine-profile.md](docs/machine-profile.md)：当前设备的抽象硬件画像与长期约束描述。
- [AGENTS.md](AGENTS.md)：Agent 与工具链协作配置入口。
