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
2. **exFAT 与 POSIX 物理边界感知 (Filesystem Boundary Awareness)**：
   - 外置盘当前格式为 exFAT（簇大小 128KB，无原生 POSIX 权限、无硬链接、无 Unix Domain Socket）。
   - 涉及动态链接、Shebang 绝对路径、硬链接优化、高频小文件（IOPS 敏感）的组件，建议留在内置盘以保障效率与稳健性。
3. **拒绝无支持的跨卷符号链接黑魔法 (Avoid Unsupported Symlink Hacks)**：
   - 严禁通过 `ln -s /Volumes/T7/... ~/.cache/...` 欺骗包管理器或系统底座。无原生机制支撑的跨卷软链接在断连、更新或清理时极易诱发级联故障。
4. **实测与因果纪律声明 (Probe Mutation & Verification Discipline)**：
   - 本次调研严格遵守无用户数据突变纪律（No user-data or runtime mutation）；
   - 现场仅在 T7 创建了一个一次性探测目录用于检验系统调用支持能力，并在测试完成后立即清理（One disposable filesystem capability probe directory was created on T7 and removed immediately after testing）。

---

## 2. 状态分类词汇表 (Classification Vocabulary)

针对各组件的放置评估，统一采用以下 7 种决策枚举：

| 决策状态 (Decision) | 语义说明 | 典型适用对象 |
| :--- | :--- | :--- |
| **`KEEP_INTERNAL`** | 保留在内置 SSD (APFS)。主要依据包括控制平面连续性、外置盘断连依赖、exFAT POSIX 限制以及迁移净收益微小。 | Homebrew 根树、Node/Python 活跃主运行时、活跃 `.venv`、`node_modules`、全局 CLI。 |
| **`SAFE_ON_T7`** | 在当前 exFAT 格式下具备原生兼容性，无须特殊配置或仅需标准下载路径指定。 | 静态只读归档、离线安装包（DMG/PKG/ZIP 暂存）。 |
| **`SAFE_ON_T7_WITH_NATIVE_CONFIG`** | 官方原生提供配置或环境变量支持，且在 exFAT 上大文件存储性能与稳定性表现优异。 | 大模型权重（HuggingFace / PyTorch 模型）、大型离线多媒体数据集。 |
| **`APFS_ONLY_FUTURE_OPTION`** | 技术上只有在外置盘具备原生 APFS 容器/卷时才值得考虑。作为未来架构储备，目前严禁实施重分区。 | 外置开发容器、便携式独立 POSIX 开发根。 |
| **`ARCHIVE_ONLY`** | 仅允许作为静态打包归档文件（推荐 `.tar` / `.tar.gz`）存放于外置盘，解压与活跃运行留在内置盘。 | 历史项目源码包、冻结代码备份。 |
| **`NOT_WORTH_COMPLEXITY`** | 官方原生支持路径重定向，但当前体积较小，或存在跨卷硬链接降级复制等性能开销，收益远低于维护复杂度。 | npm cache、pip cache、uv cache、Homebrew bottle cache、Conda pkgs cache。 |
| **`UNKNOWN`** | 暂未完成因果推演或缺乏兼容性证据的组件。 | 保持现状，不做任何假设。 |

---

## 3. 底层文件系统实证基线 (Empirical Filesystem Boundary)

针对当前内置盘（APFS）与 Samsung T7（exFAT）执行的现场实测特征对比如下：

| 特性 / 系统调用 | 内置 SSD (APFS) | Samsung T7 (exFAT) | 现场实测表现与开发环境影响 |
| :--- | :---: | :---: | :--- |
| **文件系统格式** | APFS (`disk3s1s1`) | exFAT (`disk4s1`, MBR) | 实测确认。T7 采用 128KB 簇分配，海量小文件将承受空间放大开销。 |
| **硬链接 (`link` / hardlink)** | 支持 | **不支持** (`Errno 45`) | 实测 `os.link()` 报错 `Operation not supported`。阻断 Conda / pnpm / uv 跨文件系统硬链共享。 |
| **符号链接 (`symlink`)** | 原生支持 | **实测成功** (Supported) | 实测 `os.symlink()` 在当前 macOS 挂载下成功创建。跨卷链接虽技术可行，但非受支持架构。 |
| **POSIX 权限与 chmod** | 完整支持 (`0755`/`0644`) | **无差异保留 (Synthesized)** | 实测 `os.chmod()` 设置 755 与 644 后，`stat` 均回显固定模式 `0100700`。 |
| **Unix Domain Socket** | 原生支持 | **不支持** (`Errno 45`) | 实测 `AF_UNIX` 套接字绑定报错 `Operation not supported`。开发服务与调试 Socket 无法建立。 |
| **文件锁与原子重命名** | 强一致性 | *未现场实测 (Not probed)* | *架构风险推断：非原生 POSIX 文件系统在多进程并发构建下的锁语义需单独实证，本轮不作确定性断言。* |

---

## 4. 开发环境组件放置评估矩阵 (Placement Matrix)

基于上述实证基线与各工具链官方规范，对候选组件做出的全面评估如下：

| 组件类别 (Component) | 观测规模 | 官方自定义路径机制 | exFAT 安全性 / 机制影响 | 预期净空间收益 | 复杂度与决策考量 | 决策结论 (Decision) |
| :--- | :---: | :--- | :---: | :---: | :---: | :--- |
| **Homebrew 根树** (`/opt/homebrew`) | ~3.5 GB | Apple Silicon 官方规范路径为 `/opt/homebrew` | 失去官方预编译 Bottle 支持；依赖底层 Unix 属性 | 0 (维护失控) | 包管理器控制平面留在内置盘；非默认路径失去正常支持 | **`KEEP_INTERNAL`** |
| **Miniconda Base** (`/opt/miniconda3`) | 713 MB | 安装器 Prefix 指定 | 跨盘会导致 shebang 与动态库软链接断裂风险 | ~713 MB | 离线核心底座；断连直接影响科学计算工具链可用性 | **`KEEP_INTERNAL`** |
| **NVM / Node v24** (`~/.nvm`) | 623 MB | 官方支持 `NVM_DIR` | 理论兼容，但引入外置盘常驻依赖 | ~623 MB | 承载 IDE 控制平面与终端环境；断连导致开发环境失效 | **`KEEP_INTERNAL`** |
| **活跃 `.venv` / 项目环境** | 75–200 MB/个 | virtualenv / uv venv path | 包含 shebang 绝对路径与高频小文件 | < 300 MB | 破坏断连可用性；分散治理成本高 | **`KEEP_INTERNAL`** |
| **活跃 `node_modules`** | 300MB–1GB | 项目本地标准路径 | 128KB 簇带来严重空间放大；高频小文件 IOPS 拖慢构建 | 负收益 (空间膨胀) | 项目本地依赖；随代码仓库保持就近 | **`KEEP_INTERNAL`** |
| **全局 npm 工具** (`~/.npm-global`) | 966 MB | 官方支持 `npm config set prefix` | 理论兼容，但断连丢失 CLI 访问 | ~966 MB | 承载多款核心 Agent CLI (gemini, opencli, doubao) | **`KEEP_INTERNAL`** |
| **npm 缓存** (`~/.npm`) | 231 MB | 官方支持 `npm config set cache <dir>` | exFAT 兼容 | 231 MB | 规模微小，引入常驻外置依赖得不偿失 | **`NOT_WORTH_COMPLEXITY`** |
| **uv 缓存** (`~/.cache/uv`) | 192 MB | 官方支持 `UV_CACHE_DIR` | cache 与 env 跨卷时失去硬链克隆优化，fallback to copy | 192 MB | 仅 192 MB；跨卷复制损失性能且增加断连报错风险 | **`NOT_WORTH_COMPLEXITY`** |
| **pip 缓存** (`~/Library/Caches/pip`)| 18 MB | 官方支持 `PIP_CACHE_DIR` | exFAT 兼容 | 18 MB | 仅 18 MB；收益微不足道 | **`NOT_WORTH_COMPLEXITY`** |
| **Homebrew 缓存** (`~/Library/Caches/Homebrew`) | 81 MB | 官方支持 `HOMEBREW_CACHE` | exFAT 兼容 | 81 MB | 仅 81 MB；Homebrew 自主管理轮转与清理 | **`NOT_WORTH_COMPLEXITY`** |
| **Conda 包缓存** (`pkgs`) | 451 MB | 官方支持 `pkgs_dirs` | hardlink 不可用时 fallback to copied files，失去包共享收益 | 451 MB | 将 pkgs 与 env 分开会破坏硬链接去重；规模不大 | **`NOT_WORTH_COMPLEXITY`** |
| **Bun 缓存** (`~/.bun/install/cache`)| 106 MB | 官方支持 `BUN_INSTALL_CACHE_DIR` | exFAT 兼容 | 106 MB | 仅 106 MB；收益微不足道 | **`NOT_WORTH_COMPLEXITY`** |
| **本地大模型权重** (`T7/llm`) | **37+ GB** | 官方支持 `HF_HOME` / 模型专用路径 | **优秀** (只读大单体张量文件) | **37+ GB** (已实现) | 无 POSIX 权限敏感性，低 IOPS 压力，大容量收益显著 | **`SAFE_ON_T7_WITH_NATIVE_CONFIG`** |
| **SDK/工具链安装包暂存** | 1–10 GB | 下载器/浏览器目标路径 | **优秀** (纯二进制安装镜像) | 1–10 GB (随用随清) | 临时接收层与暂存归档 | **`SAFE_ON_T7`** |
| **历史项目代码归档** | 数十 GB | 压缩打包归档 (推荐 `.tar` / `.tar.gz`) | **优秀** (包内封装完整 POSIX 元数据) | 数十 GB | 冻结旧项目，避免海量源码散落 exFAT 导致元数据丢失 | **`ARCHIVE_ONLY`** |
| **T7 独立 APFS 分区/容器** | 架构选项 | macOS 磁盘工具原生 APFS 卷 | N/A (原生 APFS) | 仅 2–4 GB (开发环境) | **极高操作风险** (MBR/exFAT已存 1.4TB 真实数据) | **`APFS_ONLY_FUTURE_OPTION`** |

---

## 5. 核心分类深度分析与架构论据

### 5.1 强保留在内置盘类别 (A. Strong Keep-Internal Candidates)
1. **控制平面与离线可用性 (Control-Plane Continuity)**：
   - 机器在拔除 Samsung T7 后，必须能够维持系统终端、Git 提交、IDE 运行、基础科学计算（Miniconda Python 3.13.5）及 Agent 交互能力。
   - NVM、Node v24、Miniconda Base 或全局 npm 工具虽然理论上支持自定义路径，但一旦绑定外置盘，断连或睡眠唤醒延迟将导致整个开发环境与编辑工具链不可用。
2. **包管理器官方规范约束**：
   - Apple Silicon 架构下，Homebrew 官方明确要求安装在原生 `/opt/homebrew`。更改 prefix 会失去对绝大多数预编译二进制 Bottle 的支持，迫使本地大量从源码编译，带来巨大的维护负担。
3. **小文件 IOPS 与空间开销**：
   - 活跃项目的 `node_modules` 与虚拟环境包含海量小文件。Samsung T7 的 128KB 大簇分配会导致严重的物理存储放大（Slack Space），且外置总线在高频随机小文件读写时的延迟显著高于内置 NVMe。

### 5.2 包管理器缓存重定向决策 (B. Package Download Caches)
- **实测容量现实**：
  现场实测全部包管理器下载缓存总和不足 **1.1 GB**（npm: 231MB, uv: 192MB, Bun: 106MB, Brew: 81MB, pip: 18MB, conda pkgs: 451MB）。
- **收益与复杂度失衡**：
  uv 与 Conda 官方均支持自定义缓存路径，但它们在同一文件系统内通常利用硬链接实现瞬时安装与空间去重。一旦将缓存外置到 exFAT，跨文件系统硬链接无法建立，包管理器必须 fallback 到深拷贝（copying files），不仅丧失去重优势，更拖慢安装速度。
- **治理结论**：全部判定为 `NOT_WORTH_COMPLEXITY`。缓存属于可再生资产，应通过定期清理与生命周期管理治理，而非跨盘迁移。

### 5.3 真正安全且高收益的外置对象 (C. Large Immutable Assets)
- **大模型权重 (Model Weights)**：
  目前已在 `T7/llm/` 存放 34.02 GB 的模型权重，以及 3.02 GB 的音频模型。该类文件具备“超大单体文件（GB 级）、只读、吞吐受限低、零 POSIX 权限敏感”特征，是外置扩展层的最佳范例。
- **安装介质与工具链镜像 (Installers & DMGs)**：
  大体积 `.dmg`、`.pkg`、Android SDK 离线包等纯二进制安装包，可直接落盘 T7 暂存或解压。
- **历史工程静态冷冻 (Frozen Project Archives)**：
  若要将历史代码迁移至 T7，**强烈推荐使用 `tar` 或 `tar.gz` 格式进行打包归档**。因为标准的 `tar` 归档能够完整保留 Unix 权限、执行位与符号链接等 POSIX 元数据；而直接解压存放或使用普通 ZIP 格式容易在跨平台读取时丢失执行权限或产生 `._` 垃圾文件。

### 5.4 外置盘 APFS 容器/分区架构评估 (D. APFS-on-T7 Question)
- **现状物理约束**：
  当前 T7 分区表为 MBR (`FDisk_partition_scheme`)，文件系统为单一的 2.0TB exFAT 分区（已使用 1.4TB 真实数据）。
- **风险与收益核算**：
  1. 在非 GPT、含 1.4TB 存量数据的磁盘上尝试非破坏性压缩与创建 APFS 分区，存在极高的数据损坏与丢失风险；
  2. 即使成功切出 APFS 卷，开发环境全部外置的理论空间回收上限仅为 **2–4 GB**，净收益极低；
  3. 当前主要矛盾并不在于开发环境挤占空间，重构外置盘物理分区弊远大于利。
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

1. **开发运行时全量留在内置盘**：所有解释器、编译器、Node 运行时、虚拟环境原位留在内置 APFS SSD，不建立任何跨卷迁移工单。
2. **包缓存不外置**：不重定向 npm、uv、pip、Homebrew 缓存路径至 T7，保持本地开箱即用与断连健壮性。
3. **保持大模型外置模式**：持续利用 `T7/llm/` 承载模型大文件，确保主盘不被 GB 级张量文件挤占。
4. **代码归档推荐 tar 打包**：非活跃历史工程统一打包为 `.tar.gz` 放入 `T7/PROJECTS/` 或归档区，完整保护 POSIX 元数据。
5. **严禁改动 T7 分区结构**：维持单一 exFAT 分区现状，不实施任何格式化或分区操作。
