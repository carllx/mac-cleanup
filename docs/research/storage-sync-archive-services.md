# 同步与归档服务选型调研报告：IDE Agent 自动化与 256GB Mac 内置盘减压

**票据关联**：[Issue #3](https://github.com/carllx/mac-cleanup/issues/3) (Part of [Wayfinder Map #1](https://github.com/carllx/mac-cleanup/issues/1))  
**研究日期**：2026-09-04  
**目标硬件基线**：Apple Silicon M1, 8GB 统一内存, 256GB 级内置 SSD（实际可用 APFS 容器约 245GB，内置盘可用空间已处临界低余量），配备 Samsung T7 2TB 外置 SSD。  
**目标工作负载**：高校教学科研（课件、多媒体、Zotero 文献库、Calibre 图书馆）、多语言软件开发、IDE Agent 自动化研发。

---

## 1. 执行摘要（Executive Conclusion）

本调研严格基于官方开发文档、API 规范与协议白皮书，评估了六大核心候选方案（百度网盘、iCloud Drive、Microsoft OneDrive、Dropbox、rclone 传输层、Syncthing）以及远端对象存储后端。核心结论如下：

1. **内置盘减压有效性**：
   - 消费级网盘（百度网盘、阿里云盘等）**完全不具备** macOS 原生按需占位符（Files-On-Demand）能力，开启双向同步即物理全量落盘，会迅速挤爆 256GB 内置盘。
   - 商业云盘中，**iCloud Drive 坚决不支持将主同步目录外置**，无法利用 Samsung T7 分流。**OneDrive** 官方支持外置盘，但除要求格式化为 APFS 外，还明确要求外置盘**不得被 macOS 识别为可移动介质（non-removable / non-ejectable drive）**，Removable USB drives 明确不受支持；当前 Samsung T7 是否满足该系统分类仍待本机事实探测。**Dropbox** 外置盘能力则依赖 macOS 15.4+、APFS (Encrypted)、<500,000 文件以及官方 eligibility/rollout 资格限制。
   - **rclone 作为传输/自动化引擎**，原生支持内存流式直传（Streaming Transfer），能直接将数据从本地/外置盘搬运到远端而完全不落地内置 SSD。
2. **IDE Agent 自动化操作可靠性**：
   - 国内消费级网盘（百度网盘等）无官方 CLI，缺乏开箱即用的标准开发环境接入能力；第三方逆向接口与爬虫存在滑动验证码（CAPTCHA）与风控阻断的次级观察（Secondary observation），**不适合引入确定性自动化脚本流水线**。
   - Syncthing 采用后台异步对等全量复制模型，冲突文件处理机制（`.sync-conflict-*`）具有非确定性，且常驻后台消耗 8GB RAM 机器宝贵的内存，**不适合作为 Agent 自动化归档后端**。
   - **rclone** 作为 Agent 操作层，具备确定性退出码（Exit Codes）、`--dry-run` 试运行、`lsjson` 结构化输出、内置标准 RC (Remote Control) HTTP API 与自动 Token 刷新机制，是 IDE Agent 最可靠的自动化操作介质。
3. **架构职责与分层分离（Engine vs Storage Backend & Sync ≠ Archive ≠ Backup）**：
   - **工具与后端分离**：`rclone` 本质是数据传输与自动化编排工具（Engine），本身不提供存储容量或数据安全保障；WORM（不可变防篡改）、Object Lock、Bucket Versioning、Lifecycle 冷归档策略、存储资费及取回延迟**均归属于具体的远端存储后端（如阿里云 OSS、腾讯云 COS、AWS S3 等）**。
   - **同步与备份分离**：所有主流网盘与 Syncthing 的核心定位均为**双向同步（Two-way Sync）**，本地删除会在云端及多端引发级联清除，绝不能等同于冷归档或只读备份。真正的冷归档与灾备必须依托后端的防篡改与单向增量链路。

---

## 2. 核心评估标准（Evaluation Criteria）

- **A. 能否真正降低 256GB Mac 内置盘压力**：是否存在无物理占用的按需占位符（Dataless files）；是否支持选择性同步；能否将同步或工作根目录安全指定在 Samsung T7 外置 SSD（含 macOS 可移动驱动器限制）。
- **B. IDE Agent / CLI 是否容易可靠操作**：是否有官方开放 API / CLI；自动化是否具备确定性退出码与结构化输出；避免依赖脆弱的 Web 登录态或风控不确定性。
- **C. 边界分离（Engine vs Backend & Sync vs Archive vs Backup）**：区分传输引擎能力与存储后端能力；能否防止双向同步误删级联；是否存在单向冷归档和防篡改能力。
- **D. 恢复与校验能力**：回收站期限、版本历史、勒索/批量误删一键时间点恢复（Point-in-time Rollback）；是否支持标准哈希校验。
- **E. 中国大陆现实可用性**：境内网络直连稳定性与访问门槛；无官方 SLA 或未实测项标记为 `Uncertain / requires local network probe`。
- **F. 成本与锁定**：免费与付费容量档位、到期后果、批量导出与迁移难度（Vendor Lock-in）。

---

## 3. 候选方案对比矩阵（Candidate Comparison Matrix）

| 评估维度 | 百度网盘 (Baidu Netdisk) | Apple iCloud Drive | Microsoft OneDrive | Dropbox | rclone (传输引擎) + 对象存储后端 | Syncthing |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **macOS 按需占位符** | **不支持**（全量物理落盘） | **支持**（原生 FileProvider） | **支持**（原生 FileProvider） | **支持**（原生 FileProvider） | 支持 mount（需 FUSE-T / nfsmount） | **不支持**（全量物理镜像） |
| **根目录能否设在 Samsung T7** | 支持（自定义下载/同步目录） | **不支持**（系统强制锁定内置盘） | **有条件支持**（要求 APFS 且非 removable/ejectable；T7 需实测） | **有条件支持**（macOS 15.4+、加密 APFS、<500k 文件、需 rollout 资格） | **原生直接操作外置挂载路径** | 支持（支持外置盘与保护标记） |
| **官方 CLI / 自动化 API** | **无官方 CLI**（OpenAPI 面向企业为主） | **无通用 REST API** / 无官方 CLI | **Graph API 完备** / 内置 CLI (`/unpin`) | **API v2 / 官方 SDK 完备** | **原生极丰富 CLI** / RC REST API | REST API / CLI 完备 |
| **Agent 操作可靠度** | **低**（无官方 CLI，逆向易遇风控验证） | **低**（无接口、brctl 属未归档私有工具易复弹） | **中高**（通过 API / 内置 /unpin） | **中**（通过 API，本地无官方 CLI） | **极高**（确定性 ExitCode/JSON） | **低**（冲突命名破坏路径确定性） |
| **服务本质定位** | 双向同步 / 单向备份 | 双向多端同步 | 双向多端同步 | 双向多端同步 | **数据传输引擎 (rclone) + 存储后端 (Cloud Storage)** | 双向持续异步对等同步 |
| **防勒索整库回滚** | **不支持** | **不支持** | **支持**（M365 专享 30 天回滚） | **支持**（Dropbox Rewind） | **支持**（依赖具体 Backend Versioning/Lock）| **不支持**（误删实时扩散） |
| **大陆网络可用性** | **直连良好**（非会员限速严苛） | **直连良好**（云上贵州运营） | **Uncertain / 需本机网络实测**（个人版直连波动） | **Uncertain / 需本机网络实测**（缺乏直连可用性保障） | **直连良好**（国内 OSS/COS 后端；境外后端需实测） | **Uncertain / 需本机网络实测**（公共节点连接性存疑）|
| **锁定与导出难度** | **极重度**（非会员下载极慢） | **重度**（无流式 API，打包耗时） | **低**（Graph API / rclone 导出） | **极低**（API 开放成熟） | **零锁定**（开放标准协议） | **零锁定**（本地开放文件） |
| **内存与系统开销** | 客户端常驻占内存 | 系统级守护进程 | 客户端常驻 | 客户端常驻 | **按需执行即释放，零常驻** | 常驻后台，大索引占用高内存 |

---

## 4. 各候选方案深度事实与官方一手证据

### 4.1 百度网盘 (Baidu Netdisk)
- **本地空间行为**：Mac 客户端未接入苹果 `FileProvider` 框架，未实现按需文件占位符（[百度网盘帮助中心](https://help.baidu.com/)）。开启同步空间会导致云端选定文件全量下载到本地磁盘。客户端支持将同步目录指定到外置硬盘自定义目录，但缓存保存在沙盒与 `~/Library/Caches/` 下，无硬性容量上限。
- **选择性同步**：官方 Mac 客户端在“同步空间”设置中支持勾选子文件夹进行选择性同步。
- **自动化能力与接入现状**：官方无命令行 CLI 工具。百度网盘开放平台官方文档中心（[pan.baidu.com/union/doc/](https://pan.baidu.com/union/doc/)）公开展示有 OAuth 2.0 规范与文件管理/上传下载 API；但对于个人开发者应用接入的审核门槛及存量维护状态，官方资料未给出绝对公开声明，**标记为 Uncertain**。社区使用逆向 Web 接口/Cookie 自动化常观察到滑块验证（CAPTCHA）与频率限制现象，属于次级观察（Secondary observation），非官方承诺行为。
- **安全与恢复**：回收站保留期普通用户 10 天，SVIP 30~180 天。无整库时间点回滚能力。个人版所有数据均存放在在线热存储池，无冷归档分层策略。
- **网络与锁定**：国内直连带宽充足，但官方服务协议明确对非会员实施下载带宽 QoS 限制（通常在百 KB/s 量级）。到期后超额空间冻结上传，因限速导致非会员导出 TB 级数据在现实中阻力极大（[空间容量与到期规则](https://pan.baidu.com/)）。

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

### 5.1 Strong Candidates（强烈推荐候选）
1. **rclone 作为 Agent 自动化传输层 + 经单独验证的对象存储（或外置 Samsung T7 归档目录）作为 Backend**：
   - *入选理由*：对 256GB 内置盘保护最彻底（流式直传零落地内置盘），对 8GB 内存最友好（按需执行零常驻），IDE Agent 适配性极佳（确定性 ExitCode、结构化 JSON、无头鉴权）；冷归档与防篡改能力由后端云存储底层保障。

### 5.2 Conditional Candidates（有条件适用候选）
1. **Microsoft OneDrive (M365 订阅)**：
   - *入选理由*：提供官方 `/unpin` CLI 释放空间；具备 30 天整库还原能力；M365 家庭版容量性价比高。
   - *前置条件*：外置盘使用受严格限制，需通过本机 Fact Probe 确认 Samsung T7 在 macOS 下是否满足非 removable/ejectable 分类；仅适合交互式同步，Agent 批量读写需防范 File Provider 水化风暴；境内网络稳定性待本机实测。
2. **Apple iCloud Drive（轻量系统配置级）**：
   - *入选理由*：系统底层集成，云上贵州运营有合规与速度保障。
   - *前置条件*：**严格限制在轻量个人文档与系统配置**；坚决不承载大型项目、科研数据集或外部归档，因其无法外置到 T7 且无可靠 CLI Evict。
3. **Dropbox**：
   - *前置条件*：需确认账号获得外置盘 rollout 资格，且系统满足 macOS 15.4+、加密 APFS 及 <500k 文件约束；需具备可靠网络连接配置。

### 5.3 Poor Fits（不合适方案）
1. **百度网盘 / 阿里云盘 / 夸克网盘消费级客户端**：
   - *排除理由*：无原生占位符（双向同步必挤爆内置盘）；无官方 CLI 工具；第三方接入存在风控与限速摩擦；数据导出成本高。
2. **Syncthing**：
   - *排除理由*：全量落盘无占位符；冲突命名破坏脚本路径确定性；本地误删不保留版本且实时传染；后台常驻索引挤占 8GB 内存。

---

## 6. 尚存未知（Remaining Unknowns）

1. **Samsung T7 在 macOS 下的驱动器分类属性（关键承重未知）**：Samsung T7 是否会被 macOS 识别为 removable/ejectable 介质，直接决定其能否被 OneDrive 官方支持作为外置同步根目录，需后续通过本机只读命令（如 `diskutil info`）进行事实探测（Fact Probe）。
2. **外置 SSD（Samsung T7）物理分区格式**：T7 当前为 APFS 还是 exFAT。
3. **公有云对象存储账号与网络偏好**：用户是否拥有或愿意开通国内合规对象存储（如阿里云 OSS、腾讯云 COS）作为 rclone 的冷归档后端。
4. **百度网盘存量资产结构**：用户当前百度网盘中是否存在科研教学唯一孤本，需由用户确认而非脚本探测。

---

## 7. 给后续决策（Issue #6: Storage Tiering）的输入建议

本报告不越权代替后续决策，但提供以下经过实证的边界约束：
- **热数据层（Hot Tier - 内置 256GB SSD）**：仅保留系统核心组件、当前最活跃工程与轻量配置；严禁挂载任何全量同步的消费级云盘。
- **温数据层（Warm Tier - 外置 Samsung T7 2TB）**：承接所有历史教学视频、大型开发环境及文献附件；可作为 rclone 本地归档目标，或在满足 non-removable/APFS 约束的前提下评估是否接入商业云盘外置域。
- **冷数据层（Cold / Archive Tier - 远端存储后端）**：由 rclone 作为 Agent 调度引擎，以单向增量（`--backup-dir`）方式将陈旧数据沉降至具备 Lifecycle / WORM 能力的对象存储后端，建立防误删防勒索的安全底座。
