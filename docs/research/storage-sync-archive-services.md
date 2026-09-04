# 同步与归档服务选型调研报告：IDE Agent 自动化与 256GB Mac 内置盘减压

**票据关联**：[Issue #3](https://github.com/carllx/mac-cleanup/issues/3) (Part of [Wayfinder Map #1](https://github.com/carllx/mac-cleanup/issues/1))  
**研究日期**：2026-09-04  
**目标硬件基线**：Apple Silicon M1, 8GB 统一内存, 256GB 级内置 SSD（实际可用 APFS 容器约 245GB，内置盘可用空间已处临界低余量），配备 Samsung T7 2TB 外置 SSD。  
**目标工作负载**：高校教学科研（课件、多媒体、Zotero 文献库、Calibre 图书馆）、多语言软件开发、IDE Agent 自动化研发。

---

## 1. 执行摘要（Executive Conclusion）

本调研严格基于官方开发文档、API 规范与协议白皮书，评估了六大核心候选方案（百度网盘、iCloud Drive、Microsoft OneDrive、Dropbox、rclone、Syncthing）以及对象存储中间层。核心结论如下：

1. **内置盘减压有效性**：
   - 消费级网盘（百度网盘、阿里云盘等）**完全不具备** macOS 原生按需占位符（Files-On-Demand）能力，开启同步即物理全量落盘，会迅速挤爆 256GB 内置盘。
   - 商业云盘中，**iCloud Drive 坚决不支持将主同步目录外置**，无法利用 Samsung T7 分流；而 **OneDrive**（格式化为 APFS 后）和 **Dropbox**（需 macOS 15.4+ 且加密 APFS）官方支持将同步根目录置于外置驱动器。
   - **rclone** 原生支持内存流式直传（Streaming Transfer），能直接将数据从本地/外置盘搬运到远端而完全不落地内置 SSD。
2. **IDE Agent 自动化操作可靠性**：
   - 国内消费级网盘（百度网盘等）对个人开发者关闭或限制 OpenAPI，无官方 CLI，开源逆向接口面临极高的人机验证（CAPTCHA）与封号风险，**属于 Poor Fit，严禁引入自动化脚本流水线**。
   - Syncthing 采用后台异步对等全量复制模型，冲突文件处理机制（`.sync-conflict-*`）具有非确定性，且常驻后台消耗 8GB RAM 机器宝贵的内存，**不适合作为 Agent 归档后端**。
   - **rclone** 具备确定性退出码（Exit Codes）、`--dry-run` 试运行、`lsjson` 结构化输出、内置标准 RC (Remote Control) HTTP API 与自动 Token 刷新机制，是 IDE Agent 最可靠的自动化操作介质。
3. **职责边界判定（Sync ≠ Archive ≠ Backup）**：
   - 所有主流网盘与 Syncthing 的核心定位均为**双向同步（Two-way Sync）**，本地删除会在云端及多端引发级联清除，绝不能等同于冷归档或只读备份。
   - 真正的冷归档与灾备能力必须依赖带有 **WORM (Object Lock)**、**时间点快照回滚（Point-in-Time Restore）** 或 **对象存储生命周期（Lifecycle Archive）** 的单向链路。

---

## 2. 核心评估标准（Evaluation Criteria）

- **A. 能否真正降低 256GB Mac 内置盘压力**：是否存在无物理占用的按需占位符（Dataless files）；是否支持选择性同步；能否将同步或工作根目录安全指定在 Samsung T7 外置 SSD。
- **B. IDE Agent / CLI 是否容易可靠操作**：是否有官方开放 API / CLI；rclone 等工具是否有稳定官方 backend；自动化是否依赖脆弱的 Web 登录态或易被风控的人机验证。
- **C. 边界分离（Sync vs Archive vs Backup）**：能否防止双向同步误删级联；是否存在单向冷归档和防篡改能力。
- **D. 恢复与校验能力**：回收站期限、版本历史、勒索/批量误删一键时间点恢复（Point-in-time Rollback）；是否支持标准哈希校验。
- **E. 中国大陆现实可用性**：境内网络直连稳定性、带宽保障、是否需要代理翻墙。
- **F. 成本与锁定**：免费与付费容量档位、到期后果、批量导出与迁移难度（Vendor Lock-in）。

---

## 3. 候选方案对比矩阵（Candidate Comparison Matrix）

| 评估维度 | 百度网盘 (Baidu Netdisk) | Apple iCloud Drive | Microsoft OneDrive | Dropbox | rclone (+ 对象存储) | Syncthing |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **macOS 按需占位符** | **不支持**（全量物理落盘） | **支持**（原生 FileProvider） | **支持**（原生 FileProvider） | **支持**（原生 FileProvider） | 支持 mount（需 FUSE-T） | **不支持**（全量物理镜像） |
| **根目录能否设在 Samsung T7** | 支持（自定义下载/同步目录） | **不支持**（系统强制锁定内置盘） | **支持**（需 APFS 卷） | **支持**（macOS 15.4+ 加密 APFS） | **原生直接操作挂载路径** | 支持（支持外置盘与保护标记） |
| **官方 CLI / 自动化 API** | **无官方 CLI** / 个人 API 暂停 | **无通用 REST API** / 无官方 CLI | **Graph API 完备** / 内置 CLI | **API v2 / 官方 SDK 完备** | **原生极丰富 CLI** / RC REST API | REST API / CLI 完备 |
| **Agent 操作可靠度** | **极低**（验证码风控、封禁） | **极低**（无接口、brctl 易复弹） | **中高**（通过 API / 内置 /unpin） | **中**（通过 API，本地无 CLI） | **极高**（确定性 ExitCode/JSON） | **低**（冲突命名破坏路径确定性） |
| **服务本质定位** | 双向同步 / 单向备份 | 双向多端同步 | 双向多端同步 | 双向多端同步 | **数据传输 / 归档调度引擎** | 双向持续异步对等同步 |
| **防勒索整库回滚** | **不支持** | **不支持** | **支持**（M365 专享 30 天回滚） | **支持**（Dropbox Rewind） | **支持**（依赖云存储版本与锁定）| **不支持**（误删实时扩散） |
| **大陆网络可用性** | **直连极佳**（非会员限速严苛）| **直连极佳**（云上贵州运营） | **不稳定**（个人版网页端被墙） | **不可用**（GFW 完全封锁） | **极佳**（直连国内 OSS/COS/S3） | **差**（公共中继/发现严重受阻）|
| **锁定与导出难度** | **极重度**（非会员下载极慢） | **重度**（无流式 API，打包耗时）| **低**（Graph API / rclone 导出）| **极低**（API 最开放成熟） | **零锁定**（开放标准协议） | **零锁定**（本地开放文件） |
| **内存与系统开销** | 客户端常驻占内存 | 系统级守护进程 | 客户端常驻 | 客户端常驻 | **按需执行即释放，零常驻** | 常驻后台，大索引占用高内存 |

---

## 4. 各候选方案深度事实与官方一手证据

### 4.1 百度网盘 (Baidu Netdisk)
- **本地空间行为**：Mac 客户端未接入苹果 `FileProvider` 框架，未实现按需文件占位符（[百度网盘帮助中心](https://help.baidu.com/)）。开启同步空间会导致云端选定文件全量下载到本地磁盘。缓存保存在沙盒与 `~/Library/Caches/` 下，无硬性容量上限。
- **自动化能力与风控**：官方开放平台已暂停个人开发者接入，仅支持企业认证主体（[百度网盘开放平台控制台](https://pan.baidu.com/union/)、[开放平台文档](https://pan.baidu.com/union/doc/)）。无官方 CLI。开源中间件（如通过 AList 转换 WebDAV）依赖逆向 Cookie (BDUSS)，极易触发人机滑块验证（CAPTCHA）或封禁。
- **安全与恢复**：回收站保留期普通用户 10 天，SVIP 30~180 天。无整库时间点回滚能力。个人版所有数据均存放在在线热存储池，无冷归档分层策略。
- **网络与锁定**：国内直连，但非 SVIP 用户遭受严苛 QoS 限速（100~300 KB/s）。到期后超额空间冻结上传，因限速导出 TB 级数据在现实中不可行，具有极重度厂商锁定（[空间容量与到期规则](https://pan.baidu.com/)）。

### 4.2 Apple iCloud Drive
- **本地空间行为**：基于系统级 `FileProvider` 架构，按需占位符（Dataless files）在 APFS 上物理占用为 0 字节，仅消耗文件系统 Inode 与扩展属性（[Apple Developer TN3150](https://developer.apple.com/documentation/technotes/tn3150-getting-ready-for-dataless-files)）。**关键缺陷**：苹果强制锁定本地路径在系统盘根目录，**完全不支持将主同步目录迁移至外置硬盘**，且软链接（Symlink）不被支持（[Apple Support: Optimize storage space on your Mac](https://support.apple.com/guide/mac-help/optimize-storage-space-sysp4ee93886/mac)）。
- **自动化能力**：苹果未提供面向第三方 CLI/脚本操作个人网盘的通用 REST API（仅面向 iOS/macOS App 沙盒的 CloudKit 容器，见 [CloudKit Documentation](https://developer.apple.com/icloud/cloudkit/)）。终端调试命令 `brctl evict` 属于内部未归档私有诊断工具，系统后台巡检或应用访问时会未经通知自动反弹重新拉取，不可作为稳定工程收回手段。
- **安全与恢复**：官方明确定义为多设备实时双向同步工具，并非 Mac 独立备份（[Apple Support: What is iCloud?](https://support.apple.com/guide/icloud/what-is-icloud-mm74e822f6de/icloud)）。已删除文件在 iCloud.com 保留 30 天，不支持整库时间点快照回滚。
- **网络与成本**：由云上贵州（GCBD）境内运营，国内直连极佳（[云上贵州 iCloud 条款](https://www.apple.com.cn/legal/internet-services/icloud/zh-cn/gcbd-terms.html)）。50GB ¥6/月，200GB ¥21/月，2TB ¥68/月。由于缺乏开放 API，迁移只能通过隐私门户批量申请打包，锁定程度重。

### 4.3 Microsoft OneDrive
- **本地空间行为**：采用 macOS File Provider 架构（[Inside the new Files On-Demand Experience on macOS](https://techcommunity.microsoft.com/t5/microsoft-onedrive-blog/inside-the-new-files-on-demand-experience-on-macos/ba-p/3058922)）。**关键优势**：官方正式支持将同步目录配置在外置硬盘（[Microsoft Support: Install OneDrive on an external drive](https://support.microsoft.com/en-us/office/install-onedrive-on-an-external-drive-6307e24b-d7a4-493f-bf43-5730527618d7)），要求外置硬盘格式化为 APFS。
- **自动化能力**：客户端内嵌官方 CLI 支持精确控制按需状态：`/Applications/OneDrive.app/Contents/MacOS/OneDrive /unpin <path> /r` 可递归释放本地物理空间（[Microsoft Learn: Set Files On-Demand states on Mac](https://learn.microsoft.com/en-us/sharepoint/files-on-demand-mac)）。具备开放的 Microsoft Graph API（[Microsoft Graph Drive API](https://learn.microsoft.com/en-us/graph/api/resources/drive?view=graph-rest-1.0)），支持个人 OAuth2 鉴权；rclone 原生支持 `onedrive` backend。
- **安全与恢复**：个人回收站保留 30 天。全类型文件支持 30 天历史版本。M365 订阅用户专享“Restore your OneDrive”功能，支持将整库一键回退到过去 30 天内的任意时间点，有效防御勒索软件（[Microsoft Support: Restore your OneDrive](https://support.microsoft.com/en-us/office/restore-your-onedrive-fa231298-759d-41cf-bcd0-e5f828acac78)）。
- **网络与成本**：个人版服务器部署在境外，网页端在大陆被阻断，客户端同步偶有波动；世纪互联运营版仅限企业租户（[Microsoft Learn: 21Vianet 服务说明](https://learn.microsoft.com/zh-cn/microsoft-365/admin/services-in-china/services-in-china?view=o365-worldwide)）。M365 家庭版 6TB（6 人每人 1TB，约 ¥498/年），性价比极高。

### 4.4 Dropbox
- **本地空间行为**：采用 File Provider 架构（[Dropbox for macOS Changes](https://help.dropbox.com/installs/macos-support-for-expected-changes)）。官方支持移动到外置盘，但**门槛苛刻**：需 macOS 15.4+、文件数少于 50 万且磁盘必须为加密 APFS（[Dropbox: Mac external drive support](https://help.dropbox.com/installs/mac-external-drive-support)）。无 macOS 官方 CLI，释放空间仅限 Finder 上下文菜单。
- **自动化能力**：拥有成熟的 Dropbox API v2 与官方 Python SDK（[Dropbox Developer Overview](https://www.dropbox.com/developers/documentation/http/overview)），支持长期 Refresh Token。但若在本地 File Provider 目录下直接执行脚本遍历，易触发水化风暴和 I/O 阻塞。
- **安全与恢复**：Basic/Plus 保留 30 天，Pro 180 天。支持 **Dropbox Rewind** 倒带恢复（[Dropbox Rewind 帮助文档](https://help.dropbox.com/delete-restore/rewind)）。
- **网络与成本**：中国大陆被 GFW 完全屏蔽，所有请求必须依赖系统代理。2TB 档位为 $119.88/年，成本最高。

### 4.5 rclone (+ 对象存储)
- **本地空间行为**：传输命令（`copy`, `sync`, `move`）采用流式内存缓冲模型（由 `--buffer-size` 控制，默认 16MiB，见 [rclone buffer-size](https://rclone.org/docs/#buffer-size)），**完全不落地内置 SSD 磁盘**。原生直接操作外置盘 `/Volumes/SamsungT7/...`（[rclone Local Backend](https://rclone.org/local/)）。
  - *注意防护*：若使用 `rclone mount`，必须通过 `--cache-dir` 显式重定向到外置 SSD，并配合 `FUSE-T` 规避 Apple Silicon 内核扩展风险（[rclone mount 文档](https://rclone.org/commands/rclone_mount/)）。
- **自动化能力**：专为命令行与自动化设计。具备确定性退出码（0~8 详尽定义，见 [rclone exit-code](https://rclone.org/docs/#exit-code)）、`--dry-run` 安全模拟、`lsjson` 标准元数据输出、`--use-json-log`、内置标准 RC REST API（[rclone RC](https://rclone.org/rc/)）。支持 40+ 种后端，可基于环境变量零交互注入 AccessKey。
- **安全与恢复**：配合 `--backup-dir` 参数实现零流量开销的版本切片增量归档（[rclone backup-dir](https://rclone.org/docs/#backup-dir-dir)）。对接公有云对象存储时，可直接继承云厂商的 **Lifecycle Rules**（下沉至冷归档/深度冷冻）、**Object Lock (WORM)** 与 **Bucket Versioning**（[rclone S3 文档](https://rclone.org/s3/)），天然免疫勒索病毒与误删。
- **网络与成本**：工具开源免费。国内直连阿里云 OSS、腾讯云 COS 速度极快（<20ms 延迟，带宽充足）。存储成本极低（以国内对象存储冷归档为例，月费仅约 ¥0.015~0.03/GB）。无常驻守护进程，按需调用后立即释放所有 RAM，对 8GB 内存设备零负担。

### 4.6 Syncthing
- **本地空间行为**：设计定位为去中心化全量文件镜像，**原生不支持任何虚拟占位符（Files-On-Demand）**（[Syncthing FAQ](https://docs.syncthing.net/intro/faq.html)）。若在 256GB 内置盘同步超量目录，会触发 `minDiskFree` 保护暂停甚至塞爆磁盘导致系统卡死。支持将根目录设在外置盘，并具备 `.stfolder` 丢失保护机制（[Syncthing Folder Marker Missing](https://docs.syncthing.net/users/foldertypes.html#folder-marker-missing)）。
- **自动化能力**：提供 REST API 与 CLI，但**冲突文件处理非确定性**：冲突时静默重命名为 `.sync-conflict-*`，无错误退出码，脚本无法判断执行原子性（[Syncthing Conflicts](https://docs.syncthing.net/users/syncing.html#conflicts)）。
- **安全与恢复**：官方明文声明 **“Syncthing is not a backup solution”**（[Is Syncthing my backup solution?](https://docs.syncthing.net/intro/faq.html#is-syncthing-my-backup-solution)）。核心痛点：双向同步导致本地误删向集群实时扩散；且**版本控制仅针对远端同步过来的变更生效，本地主动修改或删除根本不保留版本**（[Syncthing File Versioning](https://docs.syncthing.net/users/versioning.html)）。
- **网络与维护开销**：国内公网环境下，官方发现与中继服务器常年被阻断，非局域网必须自建穿透。后台常驻进程及其 LevelDB 文件索引对 8GB RAM 机器形成显著内存负担，加剧 Swap 压力。

---

## 5. 候选分级分类（Classification）

### 5.1 Strong Candidates（强烈推荐候选）
1. **rclone + 国内对象存储（如阿里云 OSS / 腾讯云 COS）/ 本地 Samsung T7 归档**：
   - *入选理由*：对 256GB 内置盘保护最彻底（流式传输零落地），对 8GB 内存最友好（零常驻进程），IDE Agent 适配性最强（确定性 ExitCode、JSON、无头鉴权），并可依托云端实现真实的冷归档与 WORM 防勒索。

### 5.2 Conditional Candidates（有条件适用候选）
1. **Microsoft OneDrive (M365 订阅)**：
   - *入选理由*：支持将 File Provider 根目录正规迁移至 Samsung T7 APFS 分区；提供官方 `/unpin` CLI 释放空间；具备 30 天整库还原能力；M365 家庭版容量性价比极高。
   - *前置条件*：仅作为交互式工作区或文档同步；Agent 读写时需通过 Graph API 或 rclone 操作以避开 File Provider 水化风暴；需接受国内直连偶发的网络波动。
2. **Apple iCloud Drive（轻量系统配置级）**：
   - *入选理由*：系统底层深度集成，国内云上贵州高速稳定。
   - *前置条件*：**严格限制在 5GB~50GB 的轻量日常文档与系统元数据**；坚决不承载大型项目、科研数据集或外部归档，坚决不将其根目录通过软链接重定向。

### 5.3 Poor Fits（不合适方案）
1. **百度网盘 / 阿里云盘 / 夸克网盘消费级客户端**：
   - *排除理由*：无原生占位符（双向同步必挤爆内置盘）；对个人无官方 CLI 与 OpenAPI；逆向接口受制于高频滑动验证码风控与封号风险；极重度厂商锁定与非会员限速。
2. **Syncthing**：
   - *排除理由*：全量落盘无占位符；冲突命名破坏脚本路径确定性；本地误删不保留版本且实时传染；后台常驻索引挤占 8GB 内存。
3. **Dropbox**：
   - *排除理由*：中国大陆全域被墙；外置盘门槛苛刻；缺乏官方 macOS CLI；美元计价成本高昂。

---

## 6. 尚存未知（Remaining Unknowns）

1. **用户历史百度网盘资产结构**：用户当前百度网盘内到底存放了多少历史数据，是否包含教学资产的主版本，需在后续决策中通过用户授权核对，而非直接由脚本遍历。
2. **外置 SSD（Samsung T7）文件系统格式**：是否已格式化为 APFS（若为 exFAT，将无法直接作为 OneDrive File Provider 根目录）。
3. **公有云对象存储账号偏好**：用户是否拥有或愿意开设阿里云/腾讯云账号以开通极低成本的冷归档 Bucket。

---

## 7. 给后续决策（Issue #6: Storage Tiering）的输入建议

本报告不越权代替后续决策，但提供以下经过实证的边界约束：
- **热数据层（Hot Tier - 内置 256GB SSD）**：仅保留系统核心组件、当前最活跃工程与轻量配置；严禁挂载任何全量同步的云盘。
- **温数据层（Warm Tier - 外置 Samsung T7 2TB）**：承接所有历史教学视频、大型开发环境及文献附件；可作为 OneDrive 外置同步域或 rclone 本地归档目标。
- **冷数据层（Cold / Archive Tier - 云端对象存储）**：通过 rclone 定期以单向增量（`--backup-dir`）方式将 T7 上的陈旧项目沉降至云端冷归档，建立防误删防勒索的安全底座。
