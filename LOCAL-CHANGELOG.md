# 本地构建更新日志（LOCAL-CHANGELOG）

> 本文件记录相对上游 `PCL-Community/PCL-CE`（dev 分支，2026-10-30 克隆）的全部本地改动。
> 所有改动在源码中以 `// [Local Patch]` 注释标记。**请勿以 PCL-CE 名义分发本构建。**

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
