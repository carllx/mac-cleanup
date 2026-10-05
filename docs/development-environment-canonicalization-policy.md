# Development Environment Canonicalization Policy

本文档为这台资源受限（256GB SSD）的 Mac 确立长期的**开发环境规范收敛策略 (Development Environment Canonicalization Policy)**。

本策略基于已验收的架构事实：
- [`docs/development-environment-topology.md`](./development-environment-topology.md)（Issue #4：运行时拓扑矩阵与必要多版本共存事实）
- [`docs/development-environment-placement-policy.md`](./development-environment-placement-policy.md)（Issue #15：存储分层与内置 APFS / 外置 exFAT 文件系统边界）
- [`docs/storage-health-and-cleanup-runbook.md`](./storage-health-and-cleanup-runbook.md)（运行健康缓冲模型与增量探测规范）
- [`docs/mole-integration-policy.md`](./mole-integration-policy.md)（三分法治理模型与用户最终授权边界）

---

## 1. 目的与核心准则 (Purpose & Guiding Principles)

本策略的目标**绝不是追求“全机只有一个 Python / 一个 Node”**，更不是以“尽量删除更多环境”为考量指标。现代软件工程、科研算法与多 Agent 系统的复杂度决定了运行时隔离是不可动摇的客观工程需求。

### 核心准则 (Guiding Principles)
1. **职责单一属主，而非物理单一实例 (Single Canonical Owner per Role, Not One Physical Copy)**：
   - 追求“一种核心职责拥有清晰唯一的权威管理者（Canonical Manager）”；
   - 允许项目隔离与专用工具环境内部保留合理的私有依赖副本（Private Dependency）。
2. **工具环境不得越权接管通用命名空间 (Tool Envs Must Not Own Generic Namespace)**：
   - 工具级专属虚拟环境（Tool-specific venv）可暴露其特定的 CLI 入口，但严禁通过终端全局 PATH 前置意外劫持通用的 `python` / `python3` 解释器。
3. **退役候选 $\neq$ 删除授权 (Retirement Candidate $\neq$ Deletion Authorization)**：
   - 孤儿、残缺或废弃环境必须经历严格的**零消费者门禁 (Zero-Consumer Gate)** 核验后方能列入候选；
   - 所有破坏性删除（Mutation）必须停在**用户最终授权边界 (User Authorization Boundary)** 前，严禁由脚本或项目策略自动执行。
4. **控制平面与离线稳健性保留在内置盘 (Keep Control Plane Internal)**：
   - 依据 Issue #15 决议，Homebrew、Miniconda Base、NVM / 活跃 Node、活跃 `.venv` 与全局 CLI 控制平面必须留在内置 SSD（APFS），确保拔除外置盘时开发底座完好可用。

---

## 2. 规范所有权模型 (Canonical Ownership Model)

本机各语言生态、工具链与包管理器的职责划分矩阵如下：

| 技术领域 (Domain) | 规范单一属主 (Canonical Manager) | 承载角色定位 | 辅助/专用隔离环境 (Allowed Secondary) |
| :--- | :--- | :--- | :--- |
| **通用与科学计算 Python** | **Miniconda Base (`/opt/miniconda3`)** | 用户终端默认交互式解释器，全局科学计算核心库（NumPy, Pandas, SciPy, Matplotlib） | 项目专用独立 Conda env（仅限需深度 C/C++ 绑定的专项项目，如 `nb-extract`） |
| **Python 项目级依赖** | **`uv` (`uv venv` / `pyproject.toml`)** | 现代代码库的独立虚拟环境管理器，优先负责高性能依赖解析与轻量隔离 | 标准 `python -m venv`（轻量独立项目备份方案） |
| **用户级独立 Python CLI** | **`uv tool` (`~/.local/bin`)** | 面向终端用户的独立 Python CLI 工具规范安装通道 | 历史 `pipx`（需逐步收敛，不推荐新增） |
| **系统底层 Python** | **Apple CLT (`/usr/bin/python3`)** | macOS 系统工具与 Xcode Command Line Tools 专用底座，SIP 保护 | 严禁用户层接管或修改 |
| **CLI 依赖型 Python** | **Homebrew (`/opt/homebrew/Cellar/python@*`)** | Homebrew formula 的下游二进制依赖专属运行时 | 仅服务于 Homebrew 内部，不对用户开放交互式符号链接 |
| **JavaScript / Node 体系** | **NVM (`~/.nvm`)** | 权威 Node 版本管理器，负责多版本切换与运行时生命周期管理 | `bun`（作为辅助快速执行器独立共存） |
| **用户级全局 Node CLI** | **NPM Global (`~/.npm-global/bin`)** | 承载全局 Agent CLI 工具（如 `gemini`, `opencli`, `mcporter` 等） | 随活跃 Node 版本独立绑定 |
| **系统级二进制 CLI** | **Homebrew (`/opt/homebrew/bin`)** | 编译型/跨平台基础设施（如 `git`, `gh`, `pandoc`, `ffmpeg`, `sox`, `mole` 等） | 严禁与 Python pip 混杂安装同名工具 |

---

## 3. Python 治理策略 (Python Policy)

### 3.1 各运行时角色边界
1. **Apple System Python (`/usr/bin/python3`)**：
   - 角色：操作系统自带基线；
   - 策略：**`KEEP / do not modify`**。禁止使用 `sudo pip` 向其安装任何包，禁止尝试重定向该路径。
2. **Homebrew Python (`python@3.12`, `python@3.13`)**：
   - 角色：服务于 Homebrew formula（如 `ffmpeg`, `media-info`, `pdftk-java` 等）的私有依赖库；
   - 策略：**`KEEP PACKAGE-MANAGER-OWNED`**。由 Homebrew 自行管理升级与修剪，不暴露到用户终端作为默认 `python3`。
3. **Miniconda Base (`/opt/miniconda3`)**：
   - 角色：用户全局规则指定的主科学计算底座；
   - 策略：**`CANONICAL INTERACTIVE PYTHON`**。终端输入 `python` / `python3` 应当默认解析至此环境，确保科研交互与算法包立即可用。
4. **项目级 `.venv`**：
   - 角色：具体项目的私有依赖沙盒；
   - 策略：**`PROJECT-SCOPED`**。位于代码仓库根目录（推荐 `.venv` 并加入 `.gitignore`），生命周期与项目生命周期完全绑定。
5. **专用工具虚拟环境 (Tool-Specific Venv)**：
   - 角色：特定工具（如 `agent-reach`）的私有运行时；
   - 策略：仅供该工具内部调用，**严禁将其 `bin/` 目录置于全局 PATH 前列**。

---

## 4. Node 治理策略 (Node Policy)

### 4.1 管理器与版本规范
1. **规范版本管理器**：
   - **NVM** 是本机唯一的 Node 版本管理器。禁止通过 Homebrew 再次安装 `brew install node`，杜绝多重管理器冲突。
2. **默认活跃版本策略**：
   - NVM 默认版本与当前活跃版本固定为 **LTS / Current 稳定版本（当前为 `v24.3.0`）**；
   - IDE 终端、系统全局脚本与 `~/.npm-global` 统一绑定该活跃版本。
3. **多版本共存准则**：
   - 仅当某特定遗留工程有明确 `.nvmrc` 且该工程处于活跃维护时，才允许保留对应的旧 Node 版本；
   - 严格禁止在 NVM 中无故保留残缺、无法执行或零消费者的历史版本。

---

## 5. 全局 CLI 治理策略 (Global CLI Policy)

### 5.1 单一规范属主原则 (Single Canonical Owner)
针对同一款 CLI 工具在多个包管理器中均可安装的情况，确立“单一规范属主”原则：

```text
[用户级全局 CLI 属主决议流]
      │
      ├─ 系统/多媒体/原生二进制 ───────► Homebrew (首选)
      ├─ 独立 Python 工具 ─────────────► uv tool (首选) / 独立单体二进制
      ├─ 独立 Node/TS CLI ─────────────► npm-global
      └─ 工具私有内部依赖 ─────────────► 保持 private dependency，不暴露全局符号
```

### 5.2 典型代表案例分析：`yt-dlp`
- **现状复核事实**：
  - 实例 A：`~/.agent-reach-venv/bin/yt-dlp`（轻量 venv stub）；
  - 实例 B：`~/.local/bin/yt-dlp`（35.4MB 独立可执行二进制，用户规则指定）。
- **长期策略决议**：
  - 确立面向用户 shell 的 **Canonical Owner 为独立的全局 CLI 路径**（未来推荐由 `uv tool` 统一管理，或保留受控独立的单体二进制）；
  - `agent-reach` 内部环境若需使用 `yt-dlp`，仅作为其私有内部依赖（Private Dependency），严禁将其作为全局 shell 的默认属主；
  - 现存历史副本定级为 **`REPAIR / CANONICALIZE FIRST`**，在建立标准属主前不进行盲目删除。

---

## 6. 全局 PATH 排序策略 (PATH Ordering Policy)

### 6.1 黄金排序准则 (Golden Rule of PATH)
终端环境变量 `PATH` 的解析顺序必须遵循：**系统控制平面 → 规范交互解释器 → 用户级独立 CLI → 系统底层基线**。

```text
[建议的长期 PATH 优先级流向]
1. ~/.npm-global/bin                       (用户级全局 Node CLI)
2. /opt/homebrew/bin, /opt/homebrew/sbin   (Homebrew 核心控制平面与系统 CLI)
3. /opt/miniconda3/bin                     (权威全局交互式 Python & 科学计算底座)
4. ~/.local/bin                            (规范用户独立 CLI，包含 uv tool / 独立脚本)
5. 系统底层路径 (/usr/bin, /bin, /usr/sbin) (Apple 系统基线)
```

### 6.2 严禁反模式 (Anti-Patterns Prohibited)
- ❌ **严禁前置工具专属 venv**：`export PATH="$HOME/.some-tool-venv/bin:$PATH"`（会导致该工具私有的 Python 解释器遮蔽系统的通用科学计算环境）。
- ❌ **严禁重复追加与循环嵌套**：保持 `.zshrc` 中 PATH 构建逻辑清晰幂等。

---

## 7. 项目隔离策略 (Project Isolation Policy)

新项目启动或现有项目维护时，遵循以下技术选型决策树：

```text
                          [ 创建新工程环境 ]
                                  │
                  该工程是否主要为纯 Python 开发？
                       ├───────────────┴───────────────┐
                       ▼ 是                            ▼ 否 (需 Node / 跨语言)
          是否需要复杂的深度非 Python C/C++ 库？          使用 NVM 对应 Node 版本
             ├─────────────────┴─────────────────┐       在工程根目录维持 node_modules
             ▼ 是                                ▼ 否
     使用 Conda 独立环境                    使用 uv 管理虚拟环境
   (conda create -n <name>)               (uv venv / uv pip install)
   仅限特定算法/多媒体项目                  ★ 推荐绝大多数新代码库默认采用
```

1. **Python 新项目默认规范**：
   - 优先采用 `uv venv` 创建位于项目根目录下的 `.venv`；
   - 依赖清单通过 `pyproject.toml` 或 `requirements.txt` 严格锁定；
   - 避免向 `/opt/miniconda3` base 环境直接无限制 `pip install` 杂乱业务依赖。
2. **Node 新项目默认规范**：
   - 项目根目录配置 `.nvmrc` 声明所需 Node 大版本；
   - 项目本地依赖严格锁定于 `node_modules`，通过 lockfile（`pnpm-lock.yaml` / `package-lock.json`）保障可重现性。

---

## 8. 缓存治理策略 (Cache Policy)

开发工具链的下载与包缓存具有**快速自然再生（Regenerable）**与**加速构建**的双重属性。严禁单纯为了追求数字而进行激进的日常清理。

### 8.1 缓存治理分级标准
1. **微小缓存放行准则（Sufficiency Stop）**：
   - 体积 `< 200 MB` 的包管理器缓存（如当前 Homebrew 58MB、Bun 106MB），属于健康运行缓冲，**一律判定为无需操作 (KEEP AS IS / NOT_WORTH_COMPLEXITY)**。
2. **增长触发式清理（Growth-Triggered Reclaim）**：
   - 仅当某单一包缓存发生异常膨胀（例如 `uv cache` 超过 **2.0 GB**，或全局缓存总和导致内置盘跌破 **🟢 GREEN (20 GiB)** 缓冲）时，才启动定向修剪。
3. **强制采用宿主原生命令（Owner-Native Cleaners Only）**：
   - 必须调用该工具官方内置的安全修剪命令，严禁直接执行 `rm -rf` 盲删缓存根目录：
     - `uv`：`uv cache prune`（修剪不可达对象）；
     - `conda`：`conda clean --tarballs`（仅清理下载压缩包，保留解压元数据）；
     - `npm`：`npm cache verify`（验证与整理，而非无脑强删）；
     - `Homebrew`：`brew cleanup --prune=30`（仅清理 30 天前的旧版本 bottle）。

---

## 9. 退役与零消费者门禁 (Retirement / Zero-Consumer Gate)

任何开发运行时、虚拟环境或全局 CLI，**绝不因为“看起来很老”或“体积较大”就自动具备删除资格**。进入退役执行前，必须严密满足以下 **七步安全闭环**：

```text
[候选实体]
   │
   ├─ 1. 零消费者排查 (Zero-Consumer Verification) ──► 活跃工程、IDE 配置中无依赖证据
   ├─ 2. 非系统/包管理器专属 (Non-OS Owned) ────────► 非 /usr/bin，无 brew formula 强依赖
   ├─ 3. 可重现性与恢复路径明确 (Reproducible) ─────► 具备清晰的重装或重新拉取方案
   ├─ 4. 零独立用户数据 (No Unique Data) ───────────► 纯依赖环境，内无用户未备份工程或权重
   ├─ 5. 无活跃进程运行 (Process Gate) ─────────────► ps 探测确认无关联后台进程或驻留服务
   ├─ 6. 预期释放收益值得操作 (Reclaim Worthwhile) ─► 回收收益显著大于操作复杂度与验证成本
   ▼
[RETIREMENT CANDIDATE (退役候选)]
   │
   ▼
[User Authorization Boundary (用户最终授权门禁)] ──► 必须经用户显式批复后方可执行删除
```

---

## 10. 当前候选对象处置表 (Candidate Disposition Table)

基于 2026-10-05 现场实测证据（VERIFIED CURRENT），对已知决策敏感实体做出如下权威定级：

| 候选实体 (Candidate Entity) | 现场实测状态 (Verified Fact) | 规模 (Footprint) | 权威分类 (Disposition) | 治理准则与后续路径 |
| :--- | :--- | :--- | :--- | :--- |
| **Conda `mybase`** | **物理路径已不存在 (Absent on disk)** | 0 B | **`RESOLVED / ABSENT`** | 现场确证 `~/.conda/envs/mybase` 在文件系统中已不存在，已在先前会话或本地清理中安全移出，不再作为待处理项。 |
| **NVM Node `v20.0.0`** | **残缺且无 node 可执行文件** | 49 MB | **`RETIREMENT CANDIDATE — HIGH CONFIDENCE`** | 仅存历史模块残留，NVM 默认已指向 v24.3.0，零消费者。列入高置信度退役候选，等待独立清理 Gate 授权移除。 |
| **Homebrew Python `3.12` / `3.13`** | **被核心 Homebrew 软件强依赖** | 150 MB | **`KEEP PACKAGE-MANAGER-OWNED`** | 经 `brew uses --installed` 实测被 `ffmpeg`, `libmediainfo`, `pdftk-java` 等依赖，严禁脱离包管理器删除。 |
| **Miniconda Base (`3.13.5`)** | **用户规则指定科学计算基座** | 713 MB | **`KEEP / CANONICAL INTERACTIVE`** | 承载基础科学计算与大多 CLI 的 shebang 底座，严格保留作为用户层通用 Python。 |
| **专用环境 `.agent-reach-venv`** | **活跃专用于 Agent Reach** | 75 MB | **`KEEP PROJECT-SCOPED`** | 工具私有环境保留，但后续需解耦全局 PATH 前置，消除对系统通用解释器的隐式遮蔽。 |
| **全局 CLI `yt-dlp` (多副本)** | **~/.local/bin 与 venv 均存有 2026.08.19** | 35.4 MB | **`REPAIR / CANONICALIZE FIRST`** | 目标建立以 `uv tool` 或独立单体二进制为 Canonical Owner 的机制，将专用 venv 内的副本降级为 private dependency。 |
| **孤儿链接 `gitingest` / `gitingest-agent`** | **软链接目标路径缺失 (Broken)** | < 1 KB | **`REPAIR / CANONICALIZE FIRST`** | 属已失效软链接，无活跃可执行目标。等待后续专项配置整理时安全解除软链接。 |
| **开发软链接 `doubao-web-bridge`** | **依赖 Downloads 临时目录源码** | 符号链接 | **`REPAIR / CANONICALIZE FIRST`** | 需先将源码移入正式代码库工作区并重新 `npm link`，解除对 Downloads 目录的隐式依赖。 |
| **`uv` 缓存目录 (`~/.cache/uv`)** | **实测体积 2.2 GB** | 2.2 GB | **`GROWTH CANDIDATE FOR NATIVE PRUNE`** | 突破 2GB 防御性关注线，建议在后续清理批次中调用官方 `uv cache prune` 执行无损修剪。 |

---

## 11. 明确的非目标 (Explicit Non-Goals)

在执行本收敛策略时，严格设立以下红线（Non-Goals）：
- ❌ **不强行统一机器为“单一 Python / 单一 Node”**；
- ❌ **不在本 Issue 中执行任何文件删除、环境卸载或包管理器清理**；
- ❌ **不在本 Issue 中重写用户 `~/.zshrc` 或修改全局 PATH**；
- ❌ **不在本 Issue 中将开发环境强行搬迁至 exFAT 外置盘（Samsung T7）**；
- ❌ **不依赖外部工具（如 Mole）的自动判定来代替本地策略决策**；
- ❌ **不脱离用户授权直接变更任何生产力工具链**。

---

## 12. 未来操作规程 (Future Operating Procedure)

当系统存储空间跌破健康缓冲（如 `< 20 GiB`），或例行维护窗口需要对开发环境执行优化时，遵循以下规程：

1. **事实重新采样 (Fresh Bounded Probe)**：
   运行最小化探测脚本核实候选对象（如 Node v20 残留、Broken Symlinks、uv cache 大小）的当前状态与消费者；
2. **比对退役门禁 (Gate Evaluation)**：
   逐项检查是否满足第 9 节的零消费者七步闭环要求；
3. **编制受控变更批次 (Bounded Mutation Plan)**：
   明确操作命令（如 `nvm rm 20.0.0`、`rm ~/.local/bin/broken-link`、`uv cache prune`）、预期释放容量与回滚方案；
4. **获取用户显式授权 (User Approval Gate)**：
   向用户呈递变更清单，在用户批准前严禁任何真实改动；
5. **执行、复验与容量记录 (Verify & Record)**：
   调用宿主原生指令执行，确认相关开发工具链与通信工具连通性正常，记录物理释放容量。
