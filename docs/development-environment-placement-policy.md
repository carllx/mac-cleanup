# Development Environment Placement & Filesystem Boundary Policy

**Status:** Accepted architecture decision for Issue #15  
**Date:** 2026-09-05  
**Scope:** Establish the authoritative placement boundary between Internal SSD (APFS) and Samsung T7 (exFAT) for development runtimes, package managers, dependency trees, download caches, and large AI/toolchain assets. Exact local paths, filesystem probes, and machine-specific benchmarks are maintained in `local/development/development-placement-evidence.json`.

---

## 1. 决策目标与价值模型 (Decision Objective & Value Model)

在容量受限（256GB SSD）的 Mac 与 2TB 外置存储（Samsung T7）协同治理中，开发环境组件的放置**不以“盲目搬迁更多”为目标**。

本规范确立的核心决策模型为：

$$\text{净收益} = \text{空间回收收益} - (\text{系统复杂度} + \text{性能/IOPS风险} + \text{可用性/断连风险} + \text{数据完整性风险})$$

### 核心准则 (Guiding Principles)
1. **技术可行 $\neq$ 值得做 (Technical Feasibility $\neq$ Worth Doing)**：
   - 哪怕工具原生支持重定向缓存路径，若实际规模仅有几十至几百 MB，却引入“外置盘必须常驻插入”、“断连报错”、“额外环境变量污染”等维护负担，一律判定为 `NOT_WORTH_COMPLEXITY`。
2. **exFAT 与 POSIX 物理边界不可妥协 (Non-negotiable Filesystem Boundary)**：
   - 外置盘当前格式为 exFAT（簇大小 128KB，无原生 POSIX 权限、无硬链接、无 Unix Domain Socket、无安全文件锁）。
   - 涉及动态链接、Shebang 绝对路径、硬链接、高频小文件（IOPS 敏感）的组件，严禁放置于当前 exFAT 卷。
3. **拒绝无支持的符号链接黑魔法 (Avoid Unsupported Symlink Hacks)**：
   - 严禁通过 `ln -s /Volumes/T7/... ~/.cache/...` 欺骗包管理器或系统底座。无原生机制支撑的跨卷软链接在断连、更新或清理时极易诱发级联故障。

---

## 2. 状态分类词汇表 (Classification Vocabulary)

针对各组件的放置评估，统一采用以下 7 种决策枚举：

| 决策状态 (Decision) | 语义说明 | 典型适用对象 |
| :--- | :--- | :--- |
| **`KEEP_INTERNAL`** | 必须严格保留在内置 SSD (APFS)。因 POSIX 特性依赖、控制平面连续性或高频 I/O，绝不可外置。 | Homebrew 根树、Node/Python 活跃主运行时、活跃 `.venv`、`node_modules`、全局 CLI。 |
| **`SAFE_ON_T7`** | 在当前 exFAT 格式下具备原生兼容性，无须特殊配置或仅需标准下载路径指定。 | 静态只读归档、离线安装包（DMG/PKG/ZIP 暂存）。 |
| **`SAFE_ON_T7_WITH_NATIVE_CONFIG`** | 官方原生提供配置或环境变量支持，且在 exFAT 上性能与稳定性表现优异。 | 大模型权重（HuggingFace / PyTorch 模型）、大型离线多媒体数据集。 |
| **`APFS_ONLY_FUTURE_OPTION`** | 技术上只有在外置盘具备原生 APFS 容器/卷时才具备可行性，当前 exFAT 卷严格阻断。作为未来架构储备，目前不得实施重分区。 | 外置开发容器、便携式独立 POSIX 开发根。 |
| **`ARCHIVE_ONLY`** | 仅允许作为静态打包归档文件（`.tar.gz` / `.zip`）存放于外置盘，解压与活跃运行必须在内置盘。 | 历史项目源码包、冻结代码备份。 |
| **`NOT_WORTH_COMPLEXITY`** | 技术上虽支持路径重定向，但当前体积过小或存在跨卷硬链接降级性能惩罚，收益远低于维护复杂度。 | npm cache、pip cache、uv cache、Homebrew bottle cache、Conda pkgs cache。 |
| **`UNKNOWN`** | 暂未完成因果推演或缺乏兼容性证据的组件。 | 保持现状，不做任何假设。 |

---

## 3. 底层文件系统实证基线 (Empirical Filesystem Boundary)

针对当前内置盘（APFS）与 Samsung T7（exFAT）执行的现场实测特征对比如下：

| 特性 / 系统调用 | 内置 SSD (APFS) | Samsung T7 (exFAT) | 现场实测表现与开发环境影响 |
| :--- | :---: | :---: | :--- |
| **文件系统格式** | APFS (`disk3s1s1`) | exFAT (`disk4s1`, MBR) | T7 采用 128KB 大簇分配，存放海量小文件将遭受严重空间放大惩罚。 |
| **硬链接 (`link` / hardlink)** | 支持 | **不支持** (`Errno 45`) | **阻断 Conda / pnpm / uv 跨包硬链接机制**。强行外置会导致包重复解压或构建失败。 |
| **符号链接 (`symlink`)** | 原生支持 | 模拟支持 (Synthesized) | macOS 内核对 exFAT 模拟软链接，但在跨系统或断连恢复时极其脆弱。 |
| **POSIX 权限与 chmod** | 完整支持 (`0755`/`0644`) | **无效 (Synthesized)** | `chmod` 调用被静默忽略（固定回显 `0700`），无法为脚本赋予独立执行位。 |
| **Unix Domain Socket** | 原生支持 | **不支持** (`Errno 45`) | 开发服务、IPC 通信、LSP 语言服务器及调试器 Socket 无法在卷上创建。 |
| **文件锁与原子重命名** | 强一致性 | 弱语义 / 潜在竞态 | 多进程构建（如 cargo、webpack、esbuild）易引发文件损坏或死锁。 |

---

## 4. 开发环境组件放置评估矩阵 (Placement Matrix)

基于上述实证基线与各工具链官方规范，对候选组件做出的全面评估如下：

| 组件类别 (Component) | 观测规模 | 官方自定义路径机制 | exFAT 安全性 | 预期净空间收益 | 维护与断连复杂度 | 决策结论 (Decision) |
| :--- | :---: | :--- | :---: | :---: | :---: | :--- |
| **Homebrew 根树** (`/opt/homebrew`) | ~3.5 GB | `HOMEBREW_PREFIX` (Apple Silicon 强绑定 `/opt/homebrew`) | **否** (致命) | 0 (系统瘫痪) | 致命级 | **`KEEP_INTERNAL`** |
| **Miniconda Base** (`/opt/miniconda3`) | 713 MB | 安装器 Prefix 指定 | **否** | ~713 MB | 极高 (Shebang失效) | **`KEEP_INTERNAL`** |
| **NVM / Node v24** (`~/.nvm`) | 623 MB | `NVM_DIR` | **否** | ~623 MB | 极高 (IDE控制平面) | **`KEEP_INTERNAL`** |
| **活跃 `.venv` / 项目环境** | 75–200 MB/个 | `virtualenv` / `uv venv` path | **否** | < 300 MB | 高 (符号链接断裂) | **`KEEP_INTERNAL`** |
| **活跃 `node_modules`** | 300MB–1GB | 项目本地标准路径 | **否** | 负收益 (大簇浪费) | 极高 (IOPS暴跌) | **`KEEP_INTERNAL`** |
| **全局 npm 工具** (`~/.npm-global`) | 966 MB | `npm config set prefix` | **否** | ~966 MB | 高 (核心Agent CLI) | **`KEEP_INTERNAL`** |
| **npm 缓存** (`~/.npm`) | 231 MB | `npm config set cache <dir>` | 是 | 231 MB | 中 (外置依赖) | **`NOT_WORTH_COMPLEXITY`** |
| **uv 缓存** (`~/.cache/uv`) | 192 MB | `UV_CACHE_DIR` | **否** (失去硬链克隆) | 192 MB | 中 | **`NOT_WORTH_COMPLEXITY`** |
| **pip 缓存** (`~/Library/Caches/pip`)| 18 MB | `PIP_CACHE_DIR` | 是 | 18 MB | 低-中 | **`NOT_WORTH_COMPLEXITY`** |
| **Homebrew 缓存** (`~/Library/Caches/Homebrew`) | 81 MB | `HOMEBREW_CACHE` | 是 | 81 MB | 低 (brew自主管理) | **`NOT_WORTH_COMPLEXITY`** |
| **Conda 包缓存** (`pkgs`) | 451 MB | `conda config --add pkgs_dirs` | **否** (退化为深拷贝) | 451 MB | 高 | **`NOT_WORTH_COMPLEXITY`** |
| **Bun 缓存** (`~/.bun/install/cache`)| 106 MB | `BUN_INSTALL_CACHE_DIR` | 是 | 106 MB | 低-中 | **`NOT_WORTH_COMPLEXITY`** |
| **本地大模型权重** (`T7/llm`) | **37+ GB** | `HF_HOME` / 模型专用加载路径 | **是** (只读大张量) | **37+ GB** (已实现) | 低 | **`SAFE_ON_T7_WITH_NATIVE_CONFIG`** |
| **SDK/工具链安装包暂存** | 1–10 GB | 浏览器/下载器目标路径 | **是** (纯二进制静态包) | 1–10 GB (随用随清) | 低 | **`SAFE_ON_T7`** |
| **历史项目代码归档** | 数十 GB | 压缩打包归档 (`.tar.gz`) | **是** (包内封装元数据) | 数十 GB | 低 | **`ARCHIVE_ONLY`** |
| **T7 独立 APFS 分区/容器** | 架构选项 | macOS Disk Utility 动态卷 | N/A (原生 APFS) | 仅 2–4 GB (开发环境) | **极高风险** (破坏1.4TB数据) | **`APFS_ONLY_FUTURE_OPTION`** |

---

## 5. 核心分类深度分析与架构论据

### 5.1 强保留在内置盘类别 (A. Strong Keep-Internal Candidates)
1. **控制平面与离线可用性保障**：
   - 机器在拔除 Samsung T7 后，必须能够维持终端启动、Git 提交、IDE 运行、基础科学计算（Miniconda Python 3.13.5）及 Agent 交互能力。
   - 若将 Node v24、Miniconda Base 或全局 npm 目录迁移至 T7，一旦外置盘断连或睡眠唤醒延迟，将导致整个系统的开发控制平面直接崩溃。
2. **POSIX 特性硬性依赖**：
   - Homebrew 依赖 Mach-O 动态链接器的 `@rpath`、硬链接及精准的 Unix 权限体系；
   - Python 虚拟环境与 Node 模块包含大量层级深邃的符号链接（如 `.bin/*` 快捷执行脚本）。在 exFAT 上，这类依赖不仅丧失执行位保护，更在每次打包时产生异常。
3. **小文件 IOPS 与簇空间膨胀（Cluster Slack Waste）**：
   - `node_modules` 与构建缓存由数以万计的 < 4KB 小文件构成。
   - Samsung T7 的 exFAT 格式簇大小为 **128 KB**。一个 1KB 的文件在外置盘上必须强行占用 128KB 物理扇区，空间浪费高达 **128 倍**，且 USB 外部总线在密集小文件随机读写下的延迟远高于内置 APFS NVMe。

### 5.2 包管理器缓存重定向决策 (B. Package Download Caches)
- **实测容量现实**：
  现场测得全部包管理器缓存总和不足 **1.1 GB**（npm: 231MB, uv: 192MB, Bun: 106MB, Brew: 81MB, pip: 18MB, conda pkgs: 451MB）。
- **收益与复杂度失衡**：
  为回收数百兆可再生空间，而配置一整套跨盘缓存环境变量（`UV_CACHE_DIR`, `npm_config_cache` 等），不仅面临外置盘未挂载时的命令报错，更破坏了工具链的开箱即用体验。
- **治理结论**：全部判定为 `NOT_WORTH_COMPLEXITY`。缓存的治理应遵循原地生命周期管理（由定期 cleanup 触发），严禁外置。

### 5.3 真正安全且高收益的外置对象 (C. Large Immutable Assets)
- **大模型权重 (Model Weights)**：
  目前已在 `T7/llm/` 存放 34.02 GB 的模型权重，以及 3.02 GB 的音频模型。该类文件具备“超大单体文件（GB 级）、只读、吞吐受限低、零 POSIX 权限敏感”特征，是外置扩展层的最佳范例。
- **安装介质与工具链镜像 (Installers & DMGs)**：
  大体积 `.dmg`、`.pkg`、Android SDK 离线包等纯二进制安装包，可直接落盘 T7 暂存或解压。
- **历史工程静态冷冻 (Frozen Project Archives)**：
  历史代码如果直接以外置源码树方式存放，会污染 exFAT 并在 Windows/Mac 交叉读取时产生 `._` 垃圾文件；**最佳实践为在内置盘通过 `tar -czf` 或 `zip` 打包后，再将单一归档文件迁移至 `T7/Archives/` 或 `T7/PROJECTS/`**。

### 5.4 外置盘 APFS 容器/分区架构评估 (D. APFS-on-T7 Question)
- **现状物理约束**：
  当前 T7 分区表为 MBR (`FDisk_partition_scheme`)，文件系统为单一的 2.0TB exFAT 分区（已使用 1.4TB 真实数据）。
- **风险与收益核算**：
  1. 在非 GPT、含 1.4TB 存量数据的磁盘上尝试非破坏性压缩与创建 APFS 分区，存在极高的数据损坏与丢失风险；
  2. 即使成功切出 50GB APFS 容器，针对开发环境的理论空间回收上限仅为 **2–4 GB**（搬迁 Node/Conda base），净收益极低；
  3. 当前内置 SSD 可用空间已脱离最初的 2.6GiB 极端风险区，并不存在迫使系统冒险重构外置盘物理分区的紧急状态。
- **架构定论**：
  判定为 **`APFS_ONLY_FUTURE_OPTION`**。严禁在当前阶段为了开发环境重构或重分区 T7。

---

## 6. 与 Issue #8 (规范化与清理) 的清晰职责边界

Issue #15 仅负责划定**空间放置（Placement）的物理与逻辑边界**，明确告诉后续工单“什么能移、什么绝不能移”：

```
┌─────────────────────────────────────────────────────────────┐
│ Issue #15: 放置边界与文件系统约束 (Placement Boundary Policy)│
│ • 判定: Homebrew, Miniconda, Node, .venv 必须 KEEP_INTERNAL │
│ • 判定: Package Caches 规模小, NOT_WORTH_COMPLEXITY 留在内置  │
│ • 判定: 模型与归档 SAFE_ON_T7                                │
│ • 判定: 绝不创建跨卷 symlink，严禁在 T7 切 APFS 分区          │
└──────────────────────────────┬──────────────────────────────┘
                               │ 输出指导约束 (Authority Input)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ Issue #8: 开发环境规范化与冗余退役 (Canonicalization Policy)  │
│ • 在内置盘安全移除 1.2 GB 孤儿环境 mybase                      │
│ • 在内置盘退役 207 MB 历史冷冻 Node v20.0.0                  │
│ • 整合 pipx 冗余与失效 Python 3.14                           │
│ • 治理 PATH 解析顺序与 Downloads 隐式软链接依赖               │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. 最终执行建议 (Summary Recommendations)

1. **零搬迁开发运行时**：所有解释器、编译器、Node 运行时、虚拟环境原位留在内置 APFS SSD，不建立任何跨卷迁移工单。
2. **零外置包缓存**：不重定向 npm、uv、pip、Homebrew 缓存路径至 T7，保持本地开箱即用与断连健壮性。
3. **保持大模型外置模式**：持续利用 `T7/llm/` 承载模型大文件，确保主盘不被 GB 级张量文件挤占。
4. **代码外置规范化**：非活跃历史工程统一打包为 `.tar.gz` 放入 `T7/PROJECTS/` 或归档区，杜绝裸源码树散落 exFAT。
5. **严禁改动 T7 分区结构**：维持单一 exFAT 分区现状，不实施任何格式化或分区操作。
