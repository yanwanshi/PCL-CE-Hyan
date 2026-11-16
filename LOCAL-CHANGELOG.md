# 本地构建更新日志（LOCAL-CHANGELOG）

> 本文件记录相对上游 `PCL-Community/PCL-CE`（dev 分支，2026-10-30 克隆）的全部本地改动。
> 所有改动在源码中以 `// [Local Patch]` 注释标记。**请勿以 PCL-CE 官方名义分发本构建。**

## local.11 — 2026-11-16

### 修复：以微软档案点「启动」时抛裸异常（发布版本号 2.15.1-local.3）

- local.9 已给「微软登录」页加了未配置 Client ID 的友好拦截，但**启动路径**（选中微软档案后点「启动」）绕过该拦截，仍抛「Microsoft OAuth client id is not configured」裸异常。
- 修复：`ModLaunch.McLoginStart` 在派发微软登录加载器前检查 `Secrets.MSOAuthClientId`，未配置时以 `$` 前缀约定抛出本地化指引（`Launch.Account.Microsoft.ClientIdMissing`），启动流程会以友好弹窗展示而非错误详情页。
- 同时修复构建管线两处缺陷：① 补丁导出漏掉未跟踪新文件（`git add -N .` 全量 intent-to-add，防止 `git clean` 误删无备份）与已暂存删除项（导出改用 `git diff HEAD`）；② 汇总步骤读取 metadata.json 补 `-Encoding UTF8`（PS 5.1 默认 GBK 解码中文导致 JSON 解析失败）。

## local.10 — 2026-11-16

### 更新通道首版发布 + 内置更新模块实测

- 构建版本升为 **2.15.1-local.2**（versionCode 525），已发布：https://github.com/yanwanshi/PCL-CE-Hyan/releases/tag/v2.15.1-local.2（zip + update-meta.json 双资产，sha256 校验一致）。
- 更新仓库 main 分支已同步最新补丁集（39 文件，含 local.1–local.9 全部改动）与 LOCAL-CHANGELOG；已废弃的 apiv2 旧格式文件移除。
- **内置更新模块实测通过**（2.15.1-local 实例实测）：启动 → 自动检测到 2.15.1-local.2 → 从 `releases/download` 下载 76.1MB 更新包（分段重试正常）→ 提取 exe → SHA-256 校验通过 → 等待重启安装。全程约 14 秒。

## local.9 — 2026-11-14

### 微软登录引导与构建密钥注入

- 问题：本地构建不含 `MS_CLIENT_ID` 密钥（正式版由 CI 注入），点「微软登录」会抛裸异常「Microsoft OAuth client id is not configured」。
- 修复：登录页检测到未配置 Client ID 时给出明确指引（改用离线登录 / 自行注册 Azure 应用并在构建时注入），不再抛裸异常。
- 构建支持密钥注入：`build.ps1` 现在会加载同目录的 `.secrets.env`（`PCL_*` 键值对），配合 `PCL_WRITE_SECRET=1` 由源码生成器把密钥编译进 exe；新增 `.secrets.env.example` 模板（含 Azure 应用注册步骤说明）。
- 新增文案键 `Launch.Account.Microsoft.ClientIdMissing`（zh-CN/zh-TW/en-US/en-GB）。

## local.8 — 2026-11-14

### 修复：Fabric loader 安装失败（Fabric meta 下载失败）

- 实测定位（对 26.3 与 1.21.1 双版本验证）：`bmclapi2.bangbang93.com/fabric-meta/v2/versions/loader/{mc}/{loader}/profile/json` 对所有版本 404（镜像路径已失效）；`meta.fabricmc.net` 同形 URL 带 "loader" 别名一律 400（官方 meta 已不接受该形式）。带具体 loader 版本号的三段式 URL 则两源均 200。
- 修复（`McDownloadFabricLoader`）：安装时先解析实际 loader 版本——传入版本无效（如字面量 "loader"）或为空时，自动从 `meta.fabricmc.net/v2/versions/loader/{mc}` 取最新版本（如 26.3 → 0.20.1），再以具体版本构造 profile 地址（bmclapi 镜像 + 官方双源保留）；解析失败且无有效传入版本时给出明确错误。
- 所有 Fabric 安装入口（版本安装页、实例设置页、启动补装、整合包内嵌 Fabric）统一受益。
- 修正解析器形状：per-game loader 列表条目为 `{"loader":{"version":"..."},...}`（版本嵌套在 loader 键下），与聚合列表的扁平形状并存兼容。已用真实端点端到端验证：26.3 → 最新 loader 0.20.1，官方源与 BMCLAPI 镜像 profile 均获取成功。
- 触发场景补充：用户重打包的 CurseForge 格式包 manifest 中 Fabric 声明为裸 `fabric-loader`（无版本号），`id.Replace("fabric-","")` 推导出 "loader"——解析器自动改用该 MC 版本最新 loader 安装。

## local.7 — 2026-11-14

### 整合包安装流程重做：拖入即自动安装

- 旧流程：识别后弹「选择一个空文件夹」→ 用户选择 → 整包解压到所选文件夹 → AddFolder。两个问题：弹窗反复出现被感知为「循环」；且平铺包解压后是散装布局，PCL 实例扫描（只认 versions/ 下实例）找不到实例，表现为「未找到可用 Minecraft 实例」。
- 新流程：识别命中后**自动安装到当前游戏文件夹的 `versions\实例名\`**（版本隔离布局）：
  - 实例名取版本 json 文件名（过滤非法字符）；
  - 解压按命中目录前缀剥离包裹层，嵌套包与根平铺包统一处理；命中目录之外的条目跳过；
  - `versions\实例名\` 已存在时确认覆盖；
  - 解压带 GB18030→UTF-8 编码重试与 zip-slip 目录包含校验；
  - 完成后强制刷新实例列表、跳转实例选择页并弹成功提示（含安装路径）。
- 新增文案键 `Minecraft.Download.Modpack.OverwriteConfirm(.Title)` / `InstallSuccess`（zh-CN/zh-TW/en-US/en-GB）。
- 平铺包安装结果即版本隔离实例：`versions\Catgirl Server 26.3\` 内含 json/jar/mods/config/PCL 等，与源实例布局一致。

## local.6 — 2026-11-14

### 修复：平铺式游戏文件夹包无法识别 + 诊断路径 NRE

- 真实案例：直接打包 PCL 隔离实例文件夹得到的平铺 zip（版本 `json+jar` 在根目录，无 `versions/` 层级），原兜底正则只认 `versions/实例名/实例名.json`，两种编码下均识别失败；且识别失败路径上存在 NRE（archive 被置空后仍被顶层条目诊断逻辑引用）。
- 修复：兜底识别泛化为「任意目录下同名 json+jar 对」（兼容 versions 层级与平铺，含根目录平铺）；探测循环不再置空 archive，NRE 消除且诊断信息可靠。
- 已用真实包与构造的 versions 层级包双样本验证命中：平铺包基目录为空、实例名取 json 文件名；versions 层级包行为与原版一致。

## local.5 — 2026-11-14

### 自有更新通道上线（GitHub Releases 直连源）

- 网络实测（本机）：`api.github.com` 与 Release 资产对象存储域可达；`cdn.jsdelivr.net`/`raw.githubusercontent.com`/`github.com` 主站均不可达或严重劣化。据此新增 **`UpdatesGitHubModel`** 更新源：
  - 版本检查 `GET {apiBase}releases/latest`（api.github.com），元数据与更新包走 Release 资产（update-meta.json + zip，对象存储域）；
  - Release 需附带 `update-meta.json`：`{"versionCode":int,"exeSha256":"...","changelog":"<summary>...</summary>"}`；
  - 公告返回空列表（该源不提供公告）。
- `SystemUpdateSourceUrl` 默认值烘焙为 `https://api.github.com/repos/yanwanshi/PCL-CE-Hyan/`——所有副本开箱即用内部更新；URL 含 `api.github.com/repos/` 时走 GitHub 模型，其余仍走 Minio 多镜像。
- 首版已发布：https://github.com/yanwanshi/PCL-CE-Hyan/releases/tag/v2.15.1-local（zip + update-meta.json），冒烟验证「远程最新版本: 2.15.1-local」判定通过。
- 新增 `publish-update.ps1`：打包 exe → 计算 sha256 → 同步补丁/日志到仓库 → 创建 Release → 上传资产。（注意：本版脚本的 apiv2 JSON 流程已废弃待改，见待办。）

### 许可合规（公开分发要求落实）

- 依据《PCL 存储库合理使用指南》重度使用条款：
  - 产品显示名改为 **「PCL2CE 个人修改版」**（保留 PCL2CE 字样 + 修改后缀，条款 1/2）；
  - 恢复启动时**非阻断**正版购买劝导（条款 4；不恢复阻断式验证弹窗，离线登录不受影响）；
  - 更新仓库附带补丁集与本日志 = 公开源代码（条款 3）；关于页署名继承 CE（条款 3）。
- 更新仓库：https://github.com/yanwanshi/PCL-CE-Hyan（公开，README 含署名与许可说明）。

## local.4 — 2026-11-14

### 修复：GBK 文件名整合包无法识别（「未能找到适合的文件结构」）

- 根因：`ModpackInstall` 打开 ZipArchive 时未指定条目名编码，国内老压缩工具（WinRAR 旧版/好压等）以 GBK 编码打包的 zip 条目名全部乱码，`manifest.json`、`versions/实例名/实例名.json` 等关键文件全部匹配不到 → 落到兜底后同样失败 → 抛出「未能找到适合的文件结构」。已用最小实验证实（构造 GBK 无标志位 zip：默认编码下条目名全乱、GB18030 下完全正确）。
- 修复：整合包类型识别改为**双编码探测**——先按 GB18030（覆盖国内 GBK 包，纯 ASCII 名不受影响），失败再按默认 UTF-8（覆盖无标志位的 UTF-8 包）；任一编码识别成功即采用。
- 兜底结构探测（`versions/实例名/实例名.json`）从 `InstallPackCompress` 前移进识别循环参与编码重试（`packType = 8`），`InstallPackCompress` 改用共享正则字段。
- 两种编码均识别失败时，错误信息附加压缩包顶层条目列表，便于定位包的实际内容。
- 已知边界：解压阶段（`ExtractModpackFiles`）保持 GB18030 优先 + 异常时切 UTF-8 的原有策略，对 GBK 包正确；极罕见的「无标志位 UTF-8 文件名」包识别后解压可能乱码（修复前则完全无法安装）。

## local.3 — 2026-10-31

### 补丁化工程管理

- 全部本地改动导出为单一补丁文件：`..\patches\pcl2-ce-local.patch`（37 个文件，+735/−457），已验证可完整应用于纯净上游 dev。
- 新增一键构建脚本 `..\build.ps1`：
  - `build.ps1`＝拉取上游 dev → 重置工作树 → 套补丁 → 发布自包含版（含未导出手改的安全拦截）；
  - `build.ps1 -ExportPatches`＝把当前工作树改动重新导出为补丁；
  - `-NoFetch` 离线重建、`-Force` 跳过安全检查、`-Configuration/-Platform` 可选。
- `.gitignore` 增加 `artifact/`。

### 安全加固

- 通用解压助手（`ModBase`）的 zip-slip 防护由前缀比较改为 `Files.IsPathWithinDirectory` 严格目录包含校验，修复兄弟目录（`C:\a` 误匹配 `C:\ab`）误放行的理论缺陷。

## local.2 — 2026-10-31

### 自有更新通道（配置驱动）

- 移除全部官方更新源（社区公益源 Pysio/Naids、GitHub 服务器源）。更新源改为读取配置项 `SystemUpdateSourceUrl`：
  - **未配置（默认）**：不发出任何更新相关网络请求，自动更新/公告/版本检查全部跳过，本地构建不可能被官方版覆盖；
  - **已配置**：指向自己的 Minio 格式发布仓库（如 GitHub 仓库 raw 地址），成为自己的发布通道。
- 配置项位于 `PCL/config.v1.yml` 的 `SystemUpdateSourceUrl` 字段（设置页暂无输入框，直接编辑配置文件即可）。
- 受守卫的调用点：启动自动检查（`ScheduleBasedOnConfig`）、手动检查（设置页 `IsLatestAsync`）、公告加载（`AnnouncementService`）、版本状态判定（`GetVersionStatus`，未配置时视为最新）。
- 仓库文件布局（与社区服务器格式一致）：`apiv2/cache.json`、`apiv2/updates/updates-sr{x64|arm64}.json`（稳定）/`updates-fr{x64|arm64}.json`（测试）、`apiv2/announcement.json`、更新包下载地址（写在 updates JSON 的 `downloads` 数组中，可以是同仓库的 Release/文件直链）。

### 版本号区分

- `metadata.json` 的 `suffix` 设为 `local`，本地构建版本号显示为 **`2.15.1-local`**，与官方发布明确区分。

## local.1 — 2026-10-31

基于上游 v2.15.1（dev）。

### 解除限制

- **下载限制**：`RestrictedFeaturePolicy` 区域判定不再拦截任何功能（原逻辑保留于 `Assess()`，可恢复）。受影响的 9 处调用点全部放行：MC 客户端下载、版本/加载器安装、整合包安装、启动补全、实例修复/重置等。
- **离线登录**：`CanCreateOtherProfile()` 始终允许创建离线/第三方档案（原发布版要求已有正版档案）；移除启动时无正版档案的「账号验证」强制弹窗（原会强制 `--demo` 或中止启动）。

### 便携化

- 首次运行或已存游戏文件夹失效时，默认使用启动器目录下的 `.minecraft`（不存在则自动创建含 `launcher_profiles.json`）；不再自动接管 `%APPDATA%\.minecraft` 等其他文件夹。启动器目录不可写时回退为扫描结果。

### 崩溃日志诊断器扩展

- 新增 15 个崩溃原因类别 + 21 条检测规则 + 提取器通用原生崩溃兜底：
  - Java 环境（JVM 参数错误、Java 安装损坏）、栈溢出、压缩文件损坏、磁盘空间不足、文件被占用/杀软拦截、路径非法字符、存档锁占用、数据包不兼容、会话失效、服务器无法连接、模组侧向装错、无效模组文件、显存耗尽、原生访问冲突兜底。
- 全部规则使用真实日志签名；文案覆盖 zh-CN（基础）/zh-TW/en-US/en-GB/fr-FR。

### Java 下载增强

- 无可用 Java 时的一键下载现支持**自由选择版本**：弹出 Mojang 运行时索引中所有可下载组件（Java 8/17/21 等，官方源优先、BMCLAPI 回退），与实例需求匹配的版本预选；列表获取失败时降级为原自动下载行为。三处调用点（启动检测、下载安装 ×2）全部接入。

### 构建与性能

- **内置 .NET**：自包含发布，无需安装 .NET 10 桌面运行时。
- **启动速度**：实测对比后采用**不压缩自包含单文件**（热启动 ~0.9s，较压缩版快约 19%；代价 196 MB）。R2R 预编译实测无收益，未采用。
- 后续重打命令：`dotnet publish "Plain Craft Launcher 2/Plain Craft Launcher 2.csproj" -c Release -p:Platform=x64 -p:SelfContained=true -o ./artifact`

### 关闭联机功能

- `LobbyService` 移除生命周期自动启动注册（后台轮询/世界发现不再随应用运行）。
- 删除「更多」工具页「联机大厅」栏、「工具」分类标题；工具页默认落点改为「实用工具」。
- 删除设置页「联机」条目及空置的「工具」分类。
- 删除 设置 → 个性化 → 隐藏页面 中对应的「联机」「大厅」勾选框（同行复选框已补位），并清理 `HiddenRefresh` 协同逻辑。
- `PCL.Core/Link` 代码与 `Resources/linkd.exe` 保留在工程中但不可达（最小化与上游 diff）。

### 移除 MirrorChyan（镜像酱）

- 删除 `UpdatesMirrorChyanModel` 更新源、更新源列表中的引用、`SystemMirrorChyanKey` 配置项。
- 删除设置页「Mirror 酱 CDK」输入区（标签/密码框/获取按钮）与相关事件处理。
- 删除 4 个语言文件中的 CDK 文案与 2 个 MirrorChyan CI 工作流，清理 `release-publish.yml` 触发步骤。
- 更新通道现仅剩社区公益源（Pysio/Naids Minio）与 GitHub。

### 依赖清理（漏洞检查结果）

- `dotnet list package --vulnerable --include-transitive`：**两个工程均无易受攻击的依赖**。
- 移除 5 个 NU1510 冗余引用（框架已内置）：`System.Net.Http` 4.3.4、`System.Text.RegularExpressions` 4.3.1（应用）；`Microsoft.CSharp`、`System.Security.Cryptography.ProtectedData`、`System.Text.Json`（Core）。

### 已知风险与待办

- ⚠️ **更新通道仍指向官方**：更新检查仍会拉取上游 Release，点击更新会被官方原版覆盖全部本地改动。处理方式见「后续更新方向」。
- 通用解压助手（`ModBase.cs`）的 zip-slip 防护使用前缀比较（`StartsWithF`），存在兄弟目录前缀误匹配的理论缺陷，建议改用 `Files.IsPathWithinDirectory`。

## 已改文件清单（相对上游 dev）

`.github/workflows/release-publish.yml`（删除 MirrorChyan 触发）｜删除 `.github/workflows/mirrorchyan_*.yml`｜`PCL.Core/App/Config.cs`｜`PCL.Core/App/Localization/Languages/*.xaml` ×5｜`PCL.Core/Link/Lobby/LobbyService.cs`｜`PCL.Core/PCL.Core.csproj`｜`CrashAnalysis/*` ×4｜`Modules/Minecraft/ModJava.cs`、`ModLaunch.cs`、`ProfileUi.cs`、`RestrictedFeaturePolicy.cs`｜`Modules/Updates/UpdateManager.cs`｜删除 `Modules/Updates/UpdatesMirrorChyanModel.cs`｜`Pages/PageDownload/ModDownloadLib.cs`｜`Pages/PageLaunch/PageLaunchLeft.xaml.cs`｜`Pages/PageSetup/*` ×5｜`Pages/PageTools/PageToolsLeft.xaml(.cs)`｜删除 `Plain Craft Launcher 2.csproj` 冗余引用
