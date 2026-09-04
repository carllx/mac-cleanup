# 同步与归档服务选型调研报告：IDE Agent 自动化与 256GB Mac 内置盘减压

**票据关联**：[Issue #3](https://github.com/carllx/mac-cleanup/issues/3) (Part of [Wayfinder Map #1](https://github.com/carllx/mac-cleanup/issues/1))  
**研究日期**：2026-09-04  
**目标硬件基线**：Apple Silicon M1, 8GB 统一内存, 256GB 级内置 SSD（实际可用 APFS 容器约 245GB，内置盘可用空间已处临界低余量），配备 Samsung T7 2TB 外置 SSD。  
**目标工作负载**：高校教学科研（课件、多媒体、Zotero 文献库、Calibre 图书馆）、多语言软件开发、IDE Agent 自动化研发。

---

## 1. 执行摘要（Executive Conclusion）

本调研严格基于官方开发文档、API 规范与协议白皮书，评估了存储后端（内置 SSD、Samsung T7、百度网盘、OneDrive、iCloud Drive、Dropbox、对象存储 OSS/COS/S3）与 Agent 自动化操作层（rclone、baidu-drive Agent Skill、Baidu Netdisk MCP、服务原生 API、桌面同步客户端）。核心结论如下：

1. **架构模型解耦：存储后端（Storage Backend/Tier）与自动化操作层（Agent Automation Layer）分离**：
   - **后端与引擎分离**：存储服务提供空间、耐久性与底层安全（如 WORM 不可变防篡改、Bucket Versioning、冷归档 Lifecycle、配额与网络 SLA）；操作层（如 `rclone`、`baidu-drive` Skill）负责在各后端间执行受控的数据流转。**`rclone` 与 `baidu-drive` 并非互斥方案**，它们分别面向不同后端、承担不同工程职责。
   - **同步、归档与备份分离**：主流网盘桌面客户端与 Syncthing 的核心定位均为**双向同步（Two-way Sync）**，本地删除会在云端及多端引发级联清除，绝不能等同于冷归档或只读备份。真正的冷归档（Archive）与灾备必须依托后端的防篡改、单向增量链路或严格隔离的安全作用域。
2. **百度网盘结论细化（拒绝单一实体化评判）**：
   - **Baidu desktop sync client（桌面全量同步客户端）**：**仍不适合作为 256GB Mac 的主同步层**。未接入 macOS 原生按需占位符（Files-On-Demand），开启双向同步会导致物理全量落盘，挤爆内置盘，且常驻后台占用内存。
   - **baidu-drive Agent Skill (`baidu-netdisk/bdpan-storage`)**：**Conditional / Promising Agent-native Candidate**。提供官方封装的 Agent Skill 与 `bdpan` CLI，支持 macOS arm64/amd64 与结构化 JSON 输出；其最关键特性是**安全操作边界严格限制在 `/apps/bdpan/` 目录**。因此，它非常适合作为由 Agent 受控调度的归档隔离区（Agent-managed archive zone），但**绝不能推导为可用于直接操作或整理用户既有的整个网盘资产**。具体运行时兼容性已拆分至原型验证票据 [Issue #10](https://github.com/carllx/mac-cleanup/issues/10)。
   - **Baidu Netdisk MCP Server (`baidu-netdisk/mcp`)**：**Conditional / Higher-friction Candidate**。具备更广的能力（含语义搜索、配额等），但官方明确个人用户目前仅属“限时体验”，且目前存在个人 Access Token 入口失效及 SSE endpoint 405 兼容性 open Issues，摩擦较高，暂不列入首轮运行时原型验证。
3. **其他候选系统与内置盘减压有效性**：
   - 商业云盘中，**iCloud Drive 坚决不支持将主同步目录外置**，无法利用 Samsung T7 分流。**OneDrive** 官方支持外置盘，但除要求 APFS 外，明确要求外置盘**不得被 macOS 识别为可移动介质（non-removable / non-ejectable drive）**；Samsung T7 是否满足该分类仍待本机事实探测。**Dropbox** 外置盘能力则依赖 macOS 15.4+、APFS (Encrypted)、<500,000 文件及 rollout 资格限制。
   - **rclone 作为多后端自动化引擎**：原生支持内存流式直传（Streaming Transfer），能直接将数据从本地/外置盘搬运到远端而完全不落地内置 SSD；具备确定性退出码、`--dry-run`、`lsjson` 与 RC API，是跨存储调度的强力基础设施。

---

## 2. 核心评估标准与架构分层模型（Architecture Model & Evaluation Criteria）

### 2.1 架构模型：存储后端与自动化操作层解耦
为彻底避免“将客户端与存储服务混为一谈”的分析缺陷，本报告将评估严格解耦为两个独立维度：
1. **存储后端 / 分层（Storage Backend / Tier）**：
   - 物理与云端载体：内置 256GB SSD、外置 Samsung T7 2TB SSD、百度网盘（Baidu Netdisk 云端存储）、Microsoft OneDrive、Apple iCloud、Dropbox、对象存储（阿里云 OSS / 腾讯云 COS / AWS S3 等）。
   - 核心职责：提供数据耐久性、容量、安全策略（WORM / Bucket Versioning / Lifecycle）、资费与网络可用性。
2. **Agent 自动化 / 访问层（Agent Access / Automation Layer）**：
   - 操作工具与接口：`rclone`、`baidu-drive` Agent Skill (`bdpan`)、Baidu Netdisk MCP Server、服务原生 API（Microsoft Graph API / Dropbox API 等）、桌面同步客户端（Desktop Sync Clients）。
   - 核心职责：在各存储后端之间安全调度数据，提供结构化输出、确定性退出码及凭据隔离。
   - **关键洞察**：`rclone` 与 `baidu-drive` Agent Skill 绝非互斥方案。它们可以分别操作不同的存储后端、在系统分层中承担不同的专业职责（例如 `rclone` 调度对象存储与本地外置盘，`baidu-drive` 调度百度网盘隔离归档区）。

### 2.2 评估标准
- **A. 能否真正降低 256GB Mac 内置盘压力**：是否存在无物理占用的按需占位符（Dataless files）；是否支持选择性同步；能否将同步或工作根目录安全指定在 Samsung T7 外置 SSD（含 macOS 可移动驱动器限制）。
- **B. IDE Agent / CLI 是否容易可靠操作**：是否有官方开放 API / CLI；自动化是否具备确定性退出码与结构化输出；避免依赖脆弱的 Web 登录态或风控不确定性。
- **C. 边界分离（Engine vs Backend & Sync vs Archive vs Backup）**：区分传输引擎能力与存储后端能力；能否防止双向同步误删级联；是否存在单向冷归档和防篡改能力。
- **D. 恢复与校验能力**：回收站期限、版本历史、勒索/批量误删一键时间点恢复（Point-in-time Rollback）；是否支持标准哈希校验。
- **E. 中国大陆现实可用性**：境内网络直连稳定性与访问门槛；无官方 SLA 或未实测项标记为 `Uncertain / requires local network probe`。
- **F. 成本与锁定**：免费与付费容量档位、到期后果、批量导出与迁移难度（Vendor Lock-in）。

---

## 3. 候选方案对比矩阵（Candidate Comparison Matrix）

| 评估维度 | 百度桌面同步客户端 (Baidu Desktop Sync) | baidu-drive Agent Skill (`bdpan-storage`) | Baidu Netdisk MCP Server (`baidu-netdisk/mcp`) | Apple iCloud Drive | Microsoft OneDrive | Dropbox | rclone (传输引擎) + 对象存储后端 | Syncthing |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **所属层级** | 桌面客户端 (Sync) | **Agent 自动化工具 (Skill/CLI)** | **Agent 自动化协议 (MCP)** | 桌面/系统同步客户端 | 桌面同步 + API/CLI | 桌面同步 + API | **数据传输引擎 (rclone) + 存储后端 (Cloud)** | 持续对等同步引擎 |
| **macOS 按需占位符** | **不支持**（全量物理落盘） | **不涉及本地镜像**（按需显式传输） | **不涉及本地镜像**（按需显式传输） | **支持**（原生 FileProvider） | **支持**（原生 FileProvider） | **支持**（原生 FileProvider） | 支持 mount（需 FUSE-T / nfsmount） | **不支持**（全量物理镜像） |
| **根目录能否设在 Samsung T7** | 支持（自定义下载/同步目录） | 原生直接操作外置路径（通过 CLI） | 原生直接操作外置路径（stdio 上传） | **不支持**（系统强制锁定内置盘） | **有条件支持**（要求 APFS 且非 removable/ejectable；T7 需实测） | **有条件支持**（macOS 15.4+、加密 APFS、<500k 文件、需 rollout 资格） | **原生直接操作外置挂载路径** | 支持（支持外置盘与保护标记） |
| **官方 CLI / 自动化接口** | **无官方 CLI** | **官方 Agent Skill + bdpan CLI** (BETA) | **官方 MCP Server** (限时体验/企业为主) | **无通用 REST API** / 无官方 CLI | **Graph API 完备** / 内置 CLI (`/unpin`) | **API v2 / 官方 SDK 完备** | **原生极丰富 CLI** / RC REST API | REST API / CLI 完备 |
| **操作作用域与安全边界** | 整个网盘（GUI 交互） | **严格限制在 `/apps/bdpan/`** | 整个网盘（权限更广） | 个人 iCloud Drive | 个人 OneDrive | 个人 Dropbox | 授权的 Bucket / 本地指定路径 | 选定同步目录 |
| **Agent 操作可靠度** | **极低**（无 CLI，依赖 GUI） | **高（理论）**（支持 JSON，Token 不暴露，兼容多 Agent） | **中低（当前）**（Token 入口失效、SSE 405 open issues） | **低**（无接口、brctl 属未归档私有工具易复弹） | **中高**（通过 API / 内置 /unpin） | **中**（通过 API，本地无官方 CLI） | **极高**（确定性 ExitCode/JSON） | **低**（冲突命名破坏路径确定性） |
| **服务/工具定位** | 双向同步 / 单向备份客户端 | **Agent 托管归档区操作工具** | **Agent 宽范围交互协议层** | 双向多端同步 | 双向多端同步 | 双向多端同步 | **通用传输引擎 (rclone) + 存储后端 (Cloud Storage)** | 双向持续异步对等同步 |
| **防勒索整库回滚** | **不支持** | **不支持** | **不支持** | **不支持** | **支持**（M365 专享 30 天回滚） | **支持**（Dropbox Rewind） | **支持**（依赖具体 Backend Versioning/Lock）| **不支持**（误删实时扩散） |
| **大陆网络可用性** | **直连良好**（非会员限速严苛） | **直连良好**（百度网盘服务端直连） | **直连良好**（官方接口服务端直连） | **直连良好**（云上贵州运营） | **Uncertain / 需本机网络实测**（个人版直连波动） | **Uncertain / 需本机网络实测**（缺乏直连可用性保障） | **直连良好**（国内 OSS/COS 后端；境外后端需实测） | **Uncertain / 需本机网络实测**（公共节点连接性存疑）|
| **锁定与导出难度** | **极重度**（非会员下载慢） | 依赖 CLI 下载（受网盘限速与接口限制） | 依赖 API 传输（受接口限制） | **重度**（无流式 API，打包耗时） | **低**（Graph API / rclone 导出） | **极低**（API 开放成熟） | **零锁定**（开放标准协议） | **零锁定**（本地开放文件） |
| **内存与系统开销** | 客户端常驻占内存 | **按需执行即释放，零常驻** | 按需启动 stdio / SSE 进程 | 系统级守护进程 | 客户端常驻 | 客户端常驻 | **按需执行即释放，零常驻** | 常驻后台，大索引占用高内存 |

---

## 4. 各候选方案深度事实与官方一手证据

### 4.1 百度网盘多层级生态与 Agent 原生能力拆分

必须坚决避免将“百度网盘”作为单一实体进行粗暴评判。官方生态当前实际包含三大截然不同的操作介质：

#### 4.1.1 百度桌面全量同步客户端 (Baidu Desktop Sync Client)
- **本地空间行为**：Mac 客户端未接入苹果 `FileProvider` 框架，未实现按需文件占位符（[百度网盘帮助中心](https://help.baidu.com/)）。开启同步空间会导致云端选定文件全量下载到本地磁盘。客户端支持将同步目录指定到外置硬盘自定义目录，但缓存保存在沙盒与 `~/Library/Caches/` 下，无硬性容量上限。
- **选择性同步**：官方 Mac 客户端在“同步空间”设置中支持勾选子文件夹进行选择性同步。
- **系统资源负担**：桌面客户端常驻后台消耗内存与 CPU，且在后台维护同步状态，对 8GB RAM 机器造成常态化资源挤占。
- **结论**：**不适合作为 256GB Mac 内置盘的主同步层**。

#### 4.1.2 官方 Agent Skill: `baidu-drive` (`baidu-netdisk/bdpan-storage`)
- **仓库与定位**：开源仓库 `baidu-netdisk/bdpan-storage`，官方定位为面向 AI Agent 的存储交互 Skill。README 声明已兼容 Cursor、Codex CLI、Gemini CLI 等 Agent。
- **提供能力**：明确提供 `upload`（上传）、`download`（下载）、`transfer`（转存）、`share`（分享）、`list/search`（列表/搜索）、`move/copy/rename`（移动/复制/重命名）、`mkdir`（创建目录）。
- **架构与环境兼容性**：底层基于官方独立编译的 `bdpan` CLI；安装方式为 `npx skills add baidu-netdisk/bdpan-storage --skill baidu-drive`；支持 macOS `arm64` 与 `amd64` 平台。
- **鉴权与自动化契约**：首次使用引导安装 `bdpan` CLI 工具并通过 OAuth 流程完成授权。官方明确要求 Token 由底层安全管理，不得被 Agent 读取或明文输出。CLI 命令支持输出 JSON，便于 Agent 进行确定性解析与条件判断。
- **核心安全边界**：**操作权限严格限制在专属目录 `/apps/bdpan/`**。
  - *工程推导*：这意味着该 Skill 天然形成了沙盒隔离，非常适合作为 **Agent-managed archive zone（Agent 托管安全归档区）**。
  - *明确非目标与边界*：**绝对不能据此推导该 Skill 可以用于直接整理或操作用户既有的整个百度网盘资产**。
- **当前状态与分级**：当前官方仍标注为 **BETA**。分级判定为 **Conditional / Promising Agent-native Candidate**。具体在 macOS + Antigravity 环境下的运行时最小闭环验证，已独立拆分至原型验证票据 [Issue #10](https://github.com/carllx/mac-cleanup/issues/10)。

#### 4.1.3 官方 MCP Server: `baidu-netdisk/mcp`
- **仓库与定位**：开源仓库 `baidu-netdisk/mcp`，基于模型上下文协议（Model Context Protocol）暴露百度网盘能力。
- **提供能力**：相较 Skill 提供更广泛的接口能力，包括：file list、metadata、mkdir、copy、delete、move、rename、local upload、URL upload、keyword search、semantic search（语义搜索）、share、quota（容量查询）。
- **传输与连接模式**：本地文件上传明确仅支持 `stdio` 传输模式；其他控制与查询能力支持 `SSE` (Server-Sent Events) 传输。
- **准入门槛与稳定性摩擦（高摩擦原因）**：
  - MCP README 明确声明：开放平台正式调用接入目前主要面向**企业开发者**；个人用户当前仅属于“限时体验”。
  - 仓库存在关键未决缺陷（Open Issues）：个人 Access Token 授权入口失效的问题尚未完全修复；SSE endpoint 与 Streamable HTTP transport 存在 405 兼容性问题。
- **当前状态与分级**：判定为 **Conditional / Higher-friction Candidate**。由于鉴权入口与传输协议的客观摩擦，暂不作为第一轮运行时原型验证对象。

#### 4.1.4 百度网盘通用后端属性（存储与网络底座）
- **安全与恢复**：回收站保留期普通用户 10 天，SVIP 30~180 天。无整库时间点回滚能力。个人版数据均存放在在线热存储池，无多层冷归档策略。
- **网络与锁定**：国内直连带宽充沛，但官方服务协议明确对非会员实施严格的下载带宽 QoS 限制（百 KB/s 级）。到期后超额空间冻结上传，导出 TB 级数据存在现实速度摩擦。

### 4.2 Apple iCloud Drive
- **本地空间行为**：基于系统级 `FileProvider` 架构，按需占位符（Dataless files）在 APFS 上物理占用为 0 字节，仅消耗文件系统 Inode 与扩展属性（[Apple Developer TN3150](https://developer.apple.com/documentation/technotes/tn3150-getting-ready-for-dataless-files)）。**关键缺陷**：苹果强制锁定本地路径在系统盘根目录，**完全不支持将主同步目录迁移至外置硬盘**，且软链接（Symlink）不被支持（[Apple Support: Optimize storage space on your Mac](https://support.apple.com/guide/mac-help/optimize-storage-space-sysp4ee93886/mac)）。
- **自动化能力**：苹果未提供面向第三方 CLI/脚本操作个人网盘的通用 REST API（仅面向 iOS/macOS App 沙盒的 CloudKit 容器，见 [CloudKit Documentation](https://developer.apple.com/icloud/cloudkit/)）。终端调试命令 `brctl evict` 属于内部未归档私有诊断工具，系统后台巡检或应用访问时会未经通知自动反弹重新拉取，不可作为稳定工程收回手段。
- **安全与恢复**：官方明确定义为多设备实时双向同步工具，并非 Mac 独立备份（[Apple Support: What is iCloud?](https://support.apple.com/guide/icloud/what-is-icloud-mm74e822f6de/icloud)）。已删除文件在 iCloud.com 保留 30 天，不支持整库时间点快照回滚。
- **网络与成本**：由云上贵州（GCBD）境内运营，国内直连有官方合规协议保障（[云上贵州 iCloud 条款](https://www.apple.com.cn/legal/internet-services/icloud/zh-cn/gcbd-terms.html)）。50GB ¥6/月，200GB ¥21/月，2TB ¥68/月。由于缺乏开放 API，迁移只能通过隐私门户批量申请打包，锁定程度重。

### 4.3 Microsoft OneDrive
- **本地空间行为与外置盘限制**：采用 macOS File Provider 架构（[Inside the new Files On-Demand Experience on macOS](https://techcommunity.microsoft.com/t5/microsoft-onedrive-blog/inside-the-new-files-on-demand-experience-on-macos/ba-p/3058922)）。
  - **外置盘关键约束（官方严格规范）**：根据 [Microsoft Support: Install OneDrive on an external drive](https://support.microsoft.com/en-us/office/install-onedrive-on-an-external-drive-6307e24b-d7a4-493f-bf43-5730527618d7)，外置盘必须格式化为 **APFS**，且必须为**非可弹出/非可移动驱动器（non-ejectable / non-removable）**。微软文档明确强调：**“Removable USB drives are not supported”**。外置盘若在 macOS 底层被识别为 Removable 卷，OneDrive 客户端将直接拒绝或报错。Samsung T7 作为 USB 移动固态硬盘是否能被系统识别为非可移动卷，**保持为待本机事实探测（Fact Probe）状态**，不得直接推导为立即可用。
- **自动化能力**：客户端内嵌官方 CLI 支持精确控制按需状态：`/Applications/OneDrive.app/Contents/MacOS/OneDrive /unpin <path> /r` 可递归释放本地物理空间（[Microsoft Learn: Set Files On-Demand states on Mac](https://learn.microsoft.com/en-us/sharepoint/files-on-demand-mac)）。具备开放的 Microsoft Graph API（[Microsoft Graph Drive API](https://learn.microsoft.com/en-us/graph/api/resources/drive?view=graph-rest-1.0)），支持个人 OAuth2 鉴权；rclone 原生支持 `onedrive` backend。
- **安全与恢复**：个人回收站保留 30 天。全类型文件支持 30 天历史版本。M365 订阅用户专享“Restore your OneDrive”功能，支持将整库一键回退到过去 30 天内的任意时间点，有效防御勒索软件（[Microsoft Support: Restore your OneDrive](https://support.microsoft.com/en-us/office/restore-your-onedrive-fa231298-759d-41cf-bcd0-e5f828acac78)）。
- **网络与成本**：个人版服务器部署在境外，国内直连稳定性官方未作 SLA 承诺，标记为 `Uncertain / requires local network probe`；世纪互联运营版仅限企业租户（[Microsoft Learn: 21Vianet 服务说明](https://learn.microsoft.com/zh-cn/microsoft-365/admin/services-in-china/services-in-china?view=o365-worldwide)）。M365 家庭版 6TB（6 人每人 1TB，约 ¥498/年），性价比极高。

### 4.4 Dropbox
- **本地空间行为与外置盘门槛**：采用 File Provider 架构（[Dropbox for macOS Changes](https://help.dropbox.com/installs/macos-support-for-expected-changes)）。
  - **外置盘条件性支持**：根据官方说明，外置盘支持处于逐步开放状态（rollout / eligibility-dependent），且具备极其严格的硬件与系统前置门槛：需 **macOS 15.4 或更高版本**、Dropbox 同步文件总数需**少于 500,000 个**，且外置磁盘必须格式化为**已加密的 APFS（APFS Encrypted）**（[Dropbox Help: External drive support](https://help.dropbox.com/installs/mac-external-drive-support)）。无 macOS 官方 CLI，释放空间仅限 Finder 上下文菜单。
- **自动化能力**：拥有成熟的 Dropbox API v2 与官方 Python SDK（[Dropbox Developer Overview](https://www.dropbox.com/developers/documentation/http/overview)），支持长期 Refresh Token。但若在本地 File Provider 目录下直接执行脚本遍历，易触发水化风暴和 I/O 阻塞。
- **安全与恢复**：Basic/Plus 保留 30 天，Pro 180 天。支持 **Dropbox Rewind** 倒带恢复（[Dropbox Rewind 帮助文档](https://help.dropbox.com/delete-restore/rewind)）。
- **网络与成本**：官方无中国境内直连服务保障，实际访问通常需要代理通道，标为 `Uncertain / requires local network probe`。2TB 档位为 $119.88/年，成本最高。

### 4.5 rclone（传输/自动化引擎）与对象存储（Backend）
必须严格明确两者的职责与边界：
- **rclone 的角色（Transfer & Automation Engine）**：
  - rclone 是开源的命令行数据传输与同步调度引擎，本身不提供任何存储空间。
  - **本地空间行为**：核心传输命令（`copy`, `sync`, `move`）采用基于 RAM 的流式内存缓冲模型（由 `--buffer-size` 控制，默认 16MiB，见 [rclone buffer-size](https://rclone.org/docs/#buffer-size)），**完全不落地内置 SSD 磁盘**。原生直接操作外置盘 `/Volumes/SamsungT7/...`（[rclone Local Backend](https://rclone.org/local/)）。若使用 `rclone mount`，必须通过 `--cache-dir` 显式重定向到外置 SSD 并结合 FUSE-T 规避内核扩展。
  - **自动化能力**：具备确定性退出码（0~8 详尽定义，见 [rclone exit-code](https://rclone.org/docs/#exit-code)）、`--dry-run` 安全模拟、`lsjson` 标准元数据输出、`--use-json-log`、内置标准 RC REST API（[rclone RC](https://rclone.org/rc/)）。支持 40+ 种后端，可基于环境变量零交互注入 AccessKey。配合 `--backup-dir` 参数实现零本地流量开销的版本切片增量归档（[rclone backup-dir](https://rclone.org/docs/#backup-dir-dir)）。
- **存储后端（Storage Backend，如阿里云 OSS / 腾讯云 COS / AWS S3 等）的角色**：
  - **数据安全性与不可变性**：WORM（Object Lock）、Bucket Versioning、防篡改机制均由云厂商底座提供，而非 rclone 自带功能（[rclone S3 文档](https://rclone.org/s3/)）。
  - **生命周期与成本**：归档存储（Archive / Deep Archive）、自动冷沉降生命周期规则（Lifecycle Rules）、每 GB 存储费用（如低至几分钱/GB/月）、流量出站费（Egress）以及解冻取回等待时间（Minutes to Hours）完全由各云存储后端自身的商业规范决定。

### 4.6 Syncthing
- **本地空间行为**：设计定位为去中心化全量文件镜像，**原生不支持任何虚拟占位符（Files-On-Demand）**（[Syncthing FAQ](https://docs.syncthing.net/intro/faq.html)）。若在 256GB 内置盘同步超量目录，会触发 `minDiskFree` 保护暂停甚至塞爆磁盘导致系统卡死。支持将根目录设在外置盘，并具备 `.stfolder` 丢失保护机制（[Syncthing Folder Marker Missing](https://docs.syncthing.net/users/foldertypes.html#folder-marker-missing)）。
- **自动化能力**：提供 REST API 与 CLI，但**冲突文件处理非确定性**：冲突时静默重命名为 `.sync-conflict-*`，无错误退出码，脚本无法判断执行原子性（[Syncthing Conflicts](https://docs.syncthing.net/users/syncing.html#conflicts)）。
- **安全与恢复**：官方明文声明 **“Syncthing is not a backup solution”**（[Is Syncthing my backup solution?](https://docs.syncthing.net/intro/faq.html#is-syncthing-my-backup-solution)）。核心痛点：双向同步导致本地误删向集群实时扩散；且**版本控制仅针对远端同步过来的变更生效，本地主动修改或删除根本不保留版本**（[Syncthing File Versioning](https://docs.syncthing.net/users/versioning.html)）。
- **网络与维护开销**：中国大陆公网环境下，依赖的公共发现与中继服务器连通性缺乏保障，标为 `Uncertain / requires local network probe`；非局域网通常需自建穿透。后台常驻进程及其 LevelDB 文件索引对 8GB RAM 机器形成显著内存负担，加剧 Swap 压力。

---

## 5. 候选分级分类（Classification）

必须基于“自动化引擎/操作层”与“存储后端/物理层”的分离原则，避免出现单一笼统的判断。

### 5.1 Strong Agent Automation Engine（强推荐 Agent 自动化引擎）
1. **rclone**：
   - *入选理由*：对 256GB 内置盘保护最彻底（基于 RAM 的流式传输零落地内置盘），对 8GB 内存最友好（按需触发、执行完毕即释放，零常驻守护进程），具备极其成熟的确定性 Exit Codes、`lsjson` 结构化输出与 RC REST API。与经单独验证的境内外合规对象存储（如阿里云 OSS、腾讯云 COS）或外置 Samsung T7 配合，构成最具确定性的归档底座。

### 5.2 Promising / Requires Prototype（有前景，待原型验证候选）
1. **`baidu-drive` Agent Skill (`baidu-netdisk/bdpan-storage`)**：
   - *入选理由*：官方原生支持 Agent 交互，提供 macOS arm64/amd64 编译版 `bdpan` CLI，操作限制在 `/apps/bdpan/` 目录形成天然隔离，具备结构化 JSON 输出与受控鉴权。
   - *候选定位*：**适合作为 Agent 托管安全归档区（Agent-managed archive zone）的潜在介质**；但绝不能用于整理用户既有的整个百度网盘。
   - *验证闭环*：其具体在 macOS + Antigravity 环境下的运行时最小闭环已交由 [Issue #10: 验证：百度网盘 baidu-drive Agent Skill 能否在 Antigravity 完成安全归档最小闭环？](https://github.com/carllx/mac-cleanup/issues/10) 进行验证。Issue #3 仅完成研究事实梳理与架构分级，不作运行时断言。

### 5.3 Conditional Candidates（有条件适用候选）
1. **Microsoft OneDrive (M365 订阅)**：
   - *前置条件*：需通过本机事实探测确认 Samsung T7 在 macOS 下是否符合“non-removable / non-ejectable”要求；批量读写需防范 File Provider 水化风暴；大陆个人版网络稳定性待实测。
2. **Apple iCloud Drive（轻量系统配置级）**：
   - *前置条件*：**严格限制在轻量个人文档与系统配置**；坚决不承载大型工程、科研附件或外部归档，因其无法外置到 T7 且缺乏可靠的 CLI 释放手段。
3. **Dropbox**：
   - *前置条件*：需确认账号获得外置盘 rollout 资格，且环境满足 macOS 15.4+、加密 APFS 及 <500k 文件约束；大陆网络需具备可靠访问通道。
4. **Baidu Netdisk MCP Server (`baidu-netdisk/mcp`)**：
   - *入选理由*：提供更宽能力（语义搜索、配额等）。
   - *前置条件与摩擦*：官方声明面向企业为主，个人仅限时体验；当前存在个人 Access Token 入口失效及 SSE 405 open issues。属于 **Higher-friction Candidate**，暂不进入第一轮运行时原型验证。

### 5.4 Poor Fit for this machine as primary full-sync layer（不适合作为本机主全量同步层）
1. **百度网盘桌面同步客户端 (Baidu Desktop Full Sync)**：
   - *排除理由*：未接入 macOS 原生按需占位符（开启同步即全量物理落盘，挤爆 256GB 内置盘）；无官方 CLI 工具；常驻后台挤占 8GB 内存。
2. **Syncthing（在 256GB 内置盘上）**：
   - *排除理由*：全量物理落盘无占位符；冲突重命名（`.sync-conflict-*`）破坏脚本执行的原子确定性；本地误删不保留版本且实时级联扩散；常驻后台索引挤占 8GB 内存。

---

## 6. 尚存未知（Remaining Unknowns）

1. **Samsung T7 在 macOS 下的驱动器分类属性（关键承重未知）**：Samsung T7 是否会被 macOS 识别为 removable/ejectable 介质，直接决定其能否被 OneDrive 官方支持作为外置同步根目录，需后续通过本机只读命令（如 `diskutil info`）进行事实探测（Fact Probe）。
2. **外置 SSD（Samsung T7）物理分区格式**：T7 当前为 APFS 还是 exFAT。
3. **公有云对象存储账号与网络偏好**：用户是否拥有或愿意开通国内合规对象存储（如阿里云 OSS、腾讯云 COS）作为 rclone 的冷归档后端。
4. **百度网盘存量资产结构**：用户当前百度网盘中是否存在科研教学唯一孤本，需由用户确认而非脚本探测。

---

## 7. 给后续决策（Issue #6: Storage Tiering）的输入建议

本报告不越权代替后续决策，但提供以下经过实证的边界约束与分层逻辑：
- **热数据层（Hot Tier - 内置 256GB SSD）**：仅保留系统核心组件、当前最活跃工程与轻量配置；严禁挂载任何全量同步的消费级云盘客户端。
- **温数据层（Warm Tier - 外置 Samsung T7 2TB）**：承接所有历史教学视频、大型开发环境及文献附件；可作为 rclone 本地归档目标，或在满足 non-removable/APFS 约束的前提下评估是否接入商业云盘外置域。
- **冷数据/归档层（Cold / Archive Tier - 远端存储后端）**：
  - **通用对象存储路径**：由 rclone 作为 Agent 调度引擎，以单向增量（`--backup-dir`）方式将陈旧数据沉降至具备 Lifecycle / WORM 能力的对象存储后端，建立防误删防勒索的安全底座。
  - **百度网盘隔离归档候选区**：若 [Issue #10](https://github.com/carllx/mac-cleanup/issues/10) 原型验证通过，可在安全隔离作用域 `/apps/bdpan/` 内作为备选的 Agent-managed archive zone；若 Issue #10 验证存在阻断性摩擦，则维持 rclone + 对象存储/外置盘为主线方案。
  - **重要原则区分**：Issue #3 本研报仅负责梳理官方证据、架构解耦与分级评估；具体的运行时兼容性与闭环成立与否，完全交由独立拆分的 Wayfinder 原型验证票据 [Issue #10](https://github.com/carllx/mac-cleanup/issues/10) 判定。

