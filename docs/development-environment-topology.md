# Development Environment Topology — As-Is Architecture & Dependency Mapping

**Status:** Accepted architecture specification for Issue #4  
**Date:** 2026-09-05  
**Scope:** Public logical mapping of macOS development runtimes, package managers, dependency trees, and necessary multiplicity. Exact private filesystem paths and user project inventories are strictly isolated in local-only registries (`local/development/development-environment-topology.json`).

---

## 1. 核心目标与原则 (Objectives & Classification Rules)

在资源受限（256GB SSD）的 Mac 上，开发环境的治理**绝不等于盲目追求“单一 Python”或“单一 Node”**。现代开发与 Agent 工作流对运行时隔离有客观要求。

本项目确立以下分类原则：

1. **必要共存（Necessary Multiplicity）**：
   - **系统级归属（OS / Apple ownership）**：由系统更新与 SIP 保护，严禁改动；
   - **包管理器依赖（Package-manager ownership）**：Homebrew 安装的工具链所依赖的独立运行时，不能随意脱离包管理器删除；
   - **项目级隔离（Project isolation）**：特定 Agent 或算法库依赖特定 Python 次版本（如 Python 3.11），与主科学计算环境（Python 3.13）共存属于必要隔离；
   - **规范指定核心（Authoritative Base）**：用户全局规则明确指定的科学计算环境（Miniconda 3.13.5）和 Node 运行时（Node v24.3.0）。
2. **重复候选判定（Duplicate Candidates）**：
   - 仅当满足 **`同一职责 + 无独占消费者 + 可完全由主运行时替代/安全删除`** 三重要件时，才标记为 `DUPLICATE CANDIDATE`；
3. **安全边界（Non-mutation Gate）**：
   - Issue #4 仅建立只读事实拓扑，**严禁执行任何 uninstall、upgrade、brew cleanup、npm cache clean、conda clean 或环境删除**；清理与规范化统一留待 Issue #8 与 Issue #15 推进。

---

## 2. 状态分类词汇表 (Classification Status Vocabulary)

所有运行时、包管理器与虚拟环境在盘点时，统一使用以下 6 个标准状态枚举：

| 状态标识 (Status) | 定义与治理语义 | 默认处置策略 (Default Policy) |
| :--- | :--- | :--- |
| **`REQUIRED`** | 系统底层、全局核心工作流、Homebrew 依赖链或核心 Agent 明确强绑定的运行时。 | **原位保留（KEEP）**。严禁任何破坏性操作。 |
| **`PROJECT-SCOPED`** | 项目专属的虚拟环境或特定版本解释器，服务于特定代码库的兼容性要求。 | **项目级保留（KEEP PROJECT-LOCAL）**。生命周期随项目走。 |
| **`SHARED`** | 作为跨项目公用的替代轻量运行时（如 Bun），独立运行无冲突。 | **共享保留（KEEP SHARED）**。维持现状。 |
| **`DUPLICATE CANDIDATE`** | 同一 CLI 或工具链因不同渠道重复安装（如 pipx vs agent-reach venv）。 | **候选收敛（CANDIDATE FOR CANONICALIZATION）**。交由 #8 决策主副本。 |
| **`LEGACY / REVIEW`** | 历史遗留版本（如未使用的旧 Node）或损坏的孤儿环境（无 Python 二进制的环境）。 | **复核确认（REVIEW & RETIRE CANDIDATE）**。在后续工单中经授权退役。 |
| **`UNKNOWN`** | 暂未探测到明确调用者，或处于动态脚本引用中的未知组件。 | **保持观察（OBSERVE）**。在获得明确证据前不予操作。 |

---

## 3. 总体运行时拓扑矩阵 (Runtime Topology Matrix)

以下为当前机器上经现场实测建立的逻辑运行时拓扑（去隐私化汇总）：

```
[System Plane]
  └── Apple System Python 3.9.6 (/usr/bin/python3) ──────────────► REQUIRED (Apple / CLT associated)

[Homebrew Plane] (/opt/homebrew)
  ├── Homebrew Python 3.12 (75 MB) ──────────────────────────────► REQUIRED (Dep of libmms, pdftk-java)
  ├── Homebrew Python 3.13 (75 MB) ──────────────────────────────► REQUIRED (Dep of ffmpeg, media-info, etc.)
  ├── Homebrew uv 0.7.5 ─────────────────────────────────────────► REQUIRED (Modern packaging engine)
  └── Homebrew CLI Tools (git, gh, pandoc, ffmpeg, sox...) ─────► REQUIRED (Core CLI Tooling)

[Conda Plane] (/opt/miniconda3 & ~/.conda)
  ├── Miniconda Base Python 3.13.5 (713 MB) ─────────────────────► REQUIRED (User Rule Scientific Base)
  ├── Conda Env: specialized extractor (159 MB) ─────────────────► PROJECT-SCOPED (Notebook tools)
  └── Conda Env (Dangling): ~1.2 GB orphaned packages ──────────► LEGACY / REVIEW (High-confidence candidate)

[uv & Python Tooling Plane] (~/.local)
  ├── Dedicated Agent Reach Venv (75 MB) ────────────────────────► REQUIRED (Active Shell & Agent Tools)
  ├── uv-managed Python 3.11 (61 MB) ────────────────────────────► PROJECT-SCOPED (Dep of Serena Agent)
  └── pipx venvs (~77 MB) ───────────────────────────────────────► DUPLICATE CANDIDATE (yt-dlp, gitingest)

[Node & JavaScript Plane] (~/.nvm, ~/.bun, ~/.npm-global)
  ├── NVM Node v24.3.0 (623 MB) ─────────────────────────────────► REQUIRED (Active Default Node)
  ├── NVM Node v20.0.0 (207 MB) ─────────────────────────────────► LEGACY / REVIEW (Inactive since 2023)
  ├── Global NPM CLI Tools (966 MB) ─────────────────────────────► REQUIRED (gemini, opencli, doubao, mcporter)
  └── Bun Standalone Runtime (163 MB) ───────────────────────────► SHARED (Fast JS runtime)
```

---

## 4. 各领域深度拓扑与证据链 (Detailed Domains & Evidence)

### 4.1 Python Topology

1. **System / Apple Python 3.9.6** (`/usr/bin/python3`)
   - **Manager**: Apple-managed / Command Line Tools associated runtime.
   - **Classification**: `REQUIRED`.
   - **Policy**: `KEEP / do not modify`. SIP-protected system baseline.
2. **Homebrew Python 3.12 & 3.13** (`/opt/homebrew/Cellar/python@*`)
   - **Classification**: `REQUIRED`.
   - **Evidence**: 经 `brew uses --installed` 实测：
     - `python@3.12` 被 `libmms`、`pdftk-java` 强依赖；
     - `python@3.13` 被 `ffmpeg@4`、`gobject-introspection`、`libmediainfo`、`media-info`、`openjdk@11` 强依赖。
     - **重要结论**：Homebrew 下的两个 Python 版本均有确定下游依赖，不是随意安装的冗余，必须原位保留。
3. **Miniconda Base Python 3.13.5** (`/opt/miniconda3`)
   - **Classification**: `REQUIRED`.
   - **Footprint**: 基础包 392 MB + 缓存 321 MB。
   - **Evidence**: 用户规则（User Rules）明确指定的全局科学计算核心环境（包含 `numpy` 2.4.1、`pandas` 2.3.3、`scipy` 1.16.3 等 264 个包）。同时作为活跃工程（如语音算法库）与专用 Agent 虚拟环境的基础底座。
4. **Conda Envs** (`~/.conda/envs/`)
   - **nb-extract (159 MB)**: `PROJECT-SCOPED`。专用于 Notebook 知识提取工具链。
   - **mybase (~1.2 GB)**: `LEGACY / REVIEW — high-confidence retirement candidate`。经只读探测：
     - Python executable missing；
     - No normal conda-meta history observed；
     - Current bounded probe found no consumer；
     - 规模约 1.2 GB。作为高置信度退役候选留待 Issue #8 执行门禁决策，本轮不予删除。
5. **专用 Agent 虚拟环境与 uv Python**
   - **`.agent-reach-venv` (75 MB)**: `REQUIRED`。当前在 `.zshrc` 中被提升至 PATH 顶层，承载 `agent-reach` (v1.5.0) 及活跃的 `yt-dlp`。
   - **uv Python 3.11.12 (61 MB)**: `PROJECT-SCOPED`。由 uv 管理并锁定，专供核心 Agent 库（要求 `python >=3.11, <3.12`）与 `graphifyy` 工具使用。

### 4.2 Node & JavaScript Topology

1. **NVM Node v24.3.0 (623 MB)**: `REQUIRED`。
   - 机器权威主 Node 运行时，与当前 IDE 和活跃代码库直接对接。
2. **NVM Node v20.0.0 (207 MB)**: `LEGACY / REVIEW` (Confidence: MEDIUM/HIGH)。
   - 安装于 2023-04-30。在当前 bounded probe 下未发现活跃调用者（No consumer found in current bounded probe）。不视为已授权退役，留待后续复核。
3. **Global NPM Directory (`~/.npm-global`, 966 MB)**: `REQUIRED`。
   - 承载多款核心 Agent CLI：`@google/gemini-cli` (481MB)、`netlify-cli` (316MB)、`@iflow-ai/iflow-cli` (77MB)、`mcporter` (60MB)、`opencli` (29MB)。
   - **关键发现**：`doubao-web-bridge` 当前符号链接指向 `Downloads/doubao-web-bridge`。这是一个典型的“开发环境对临时下载目录形成隐式依赖”，后续在治理 Downloads 时必须先完成源码安全归集。
4. **Bun Standalone (`~/.bun`, 163 MB)**: `SHARED`。
   - 独立极速运行器，在 `.zshrc` 中配置为辅助工具，占用较小且无冲突。

### 4.3 Homebrew Topology

- **Prefix**: `/opt/homebrew` (Apple Silicon 原生路径).
- **Leaves**: 48 个（包含核心 CLI：`git`, `gh`, `pandoc`, `ffmpeg`, `sox`, `exiftool`, `media-info`, `uv` 等）。
- **Casks**: 10 个（如 `mactex`, `libreoffice`, `mpv`, `powershell`, `wine-stable` 等）。
- **Brew Cache**: 仅 81 MB。
- **治理结论**：Homebrew 体系健康，未发现巨额缓存膨胀，formulae 依赖关系严密，不宜进行激进的 `brew autoremove`。

### 4.4 Global CLI 重复与遮蔽分析 (Shadowing & Duplicates)

现场探测发现以下工具存在跨包管理器重复或遮蔽现象：

1. **`yt-dlp`**:
   - 实例 A：`~/.agent-reach-venv/bin/yt-dlp`（活跃，处于 PATH 最前端，版本 2026.8.19）；
   - 实例 B：`~/.local/bin/yt-dlp`（独立二进制/pipx 历史残留，37MB）。
   - **结论**：实例 A 处于活跃维护与调用中，实例 B 属于 `DUPLICATE CANDIDATE`。
2. **`python` / `python3` 终端解析优先级**:
   - 当前交互式 Shell (`.zshrc`) 中：
     `~/.agent-reach-venv/bin` 排在 `/opt/miniconda3/bin` 之前。
   - 这导致在终端直接输入 `python3` 时，默认命中 `.agent-reach-venv`（虽然它的基底同样来自 miniconda 3.13.5）。建议在 #8 规范化时评估是否需要解耦。
3. **损坏的孤儿 CLI 软链接**:
   - `~/.local/bin/gitingest-agent` 损坏，指向已失效的 Downloads 临时目录。已标定为 `LEGACY / REVIEW`。

---

## 5. 当前各 Runtime / Cache 观测容量分布 (Selected Observed Footprint)

> **容量声明与统计口径**：
> - 以下列表仅反映**本次 bounded probe 重点观测到的特定运行时与缓存容量分布（Selected observed runtime/cache footprint）**，总和约 4.5 GB。
> - **非整机开发环境完整容量总计**：未穷举全部项目的 `node_modules`、虚拟环境、构建产物或完整的 Homebrew Cellar 树；
> - 部分统计维度可能存在少量嵌套重叠；
> - 该数值仅用于把握各组件的相对数量级与物理分布，**绝不作为后续清理验收的绝对数学基准**。

| 类别 / 目录 | 架构逻辑路径 | 观测体积 | 评估分类 | 治理建议 (针对 #8 / #15) |
| :--- | :--- | :--- | :--- | :--- |
| **Miniconda 孤儿环境** | `~/.conda/envs/mybase` | **~1.2 GB** | `LEGACY / REVIEW` | 高置信退役候选。需在 #8 授权后安全移除。 |
| **全局 NPM 扩展包** | `~/.npm-global` | **966 MB** | `REQUIRED` | 核心 CLI 依赖，原位保留在内置 SSD。 |
| **Miniconda Base & Pkgs**| `/opt/miniconda3` | **713 MB** | `REQUIRED` | 科学计算基石，原位保留。 |
| **NVM Node v24.3.0** | `~/.nvm/versions/.../v24` | **623 MB** | `REQUIRED` | 主 Node 引擎，原位保留。 |
| **NVM Node v20.0.0** | `~/.nvm/versions/.../v20` | **207 MB** | `LEGACY / REVIEW` | 历史版本，后续可复核退役。 |
| **NPM 缓存** | `~/.npm` | **231 MB** | `CACHE` | 可再生缓存，后续按需释放。 |
| **uv 缓存与工具** | `~/.cache/uv` & `~/.local/share/uv` | **462 MB** | `REQUIRED / CACHE` | uv 引擎缓存与 Python 3.11 运行时。 |
| **Bun 引擎与缓存** | `~/.bun` | **163 MB** | `SHARED` | 独立运行器，原位保留。 |
| **Homebrew Python 双版本**| `/opt/homebrew/Cellar/python@*`| **150 MB** | `REQUIRED` | Homebrew 核心依赖，原位保留。 |
| **pipx 虚拟环境** | `~/.local/pipx` | **77 MB** | `DUPLICATE CANDIDATE`| 依赖失效 Python 3.14，可整合。 |
| **Homebrew 下载缓存** | `~/Library/Caches/Homebrew` | **81 MB** | `CACHE` | 规模微小，保持现状。 |
| **Pip 缓存** | `~/Library/Caches/pip` | **18 MB** | `CACHE` | 规模微小，保持现状。 |

---

## 6. 为 Issue #15 (T7 外置可行性) 提供的证据输入 (Evidence Inputs for #15)

根据 `docs/storage-tiering-policy.md` 与现场实测环境，#4 仅提供底层依赖、POSIX 语义、符号链接及 I/O 特征证据；**具体组件的最终放置决策统一由 Issue #15 独立验证与决定**：

1. **强保留在内置盘候选 / 在当前 exFAT T7 上具有高风险 (Strong keep-internal candidates / high-risk on current exFAT T7)**：
   - **Homebrew 根树 (`/opt/homebrew`)**：高度依赖硬链接（hardlink）、Unix 权限与 POSIX 语义；
   - **Miniconda 主树与运行中环境 (`/opt/miniconda3`)**：内部二进制广泛采用绝对路径硬编码与动态链接；
   - **活跃项目的 `.venv` 与 `node_modules`**：包含极高频的小文件 I/O、符号链接与跨平台权限敏感属性；
   - **NVM 与 Node 运行时 (`~/.nvm`)**：直接承载系统 IDE 控制平面。
2. **当前最明显的低风险 T7 放置候选 (Low-risk T7 placement candidates)**：
   - **只读大模型权重与离线数据集**（当前已在 `T7/llm/` 承载，运行良好）；
   - **部分独立隔离的大型免编译安装包下载缓存（Package Download Caches）**；
   - **历史不再活跃修改的代码冻结归档包（如压缩归档）**。
   - *说明：上述候选仅作为事实证据输入，最终 placement 结论由 Issue #15 决定。*

---

## 7. 结论与下一步路标 (Roadmap & Next Steps)

本轮只读探测已达成 Issue #4 的阶段使命：
1. **实证了多版本共存的合理性**：论证了 Homebrew 双 Python、Miniconda Base、uv Python 3.11 的客观必要性；
2. **定位了潜在冗余与退役候选**：
   - 发现了 **~1.2 GB 缺失 Python 二进制的孤儿环境 `mybase`**；
   - 发现了 **207 MB 的冷冻旧版本 `Node v20.0.0`**；
   - 发现了 **77 MB 基于失效 Python 3.14 构建的 pipx 残留**；
   - 发现了 **`doubao-web-bridge` 对 Downloads 临时目录的隐式依赖**。
3. **安全准则达成**：全过程零数据变动、零配置修改，严格遵守 Non-mutation Gate。

**后续路线图**：
- **Issue #15**：基于本拓扑提供的实证，深入验证哪些开发环境组件可迁移到 T7，哪些因 exFAT / POSIX 约束必须留内置盘；
- **Issue #8**：制定开发环境规范化策略（Canonicalization Policy），在取得用户显式授权后，有序清理上述定位的冗余候选。
