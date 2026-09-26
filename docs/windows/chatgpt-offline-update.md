# ChatGPT（原 Codex）商店更新失败：官方离线包升级策略

记录日期：2026-09-26。本文基于一次 Windows 11 x64 主机的实际处理；版本号是案例记录，不代表阅读时的最新版。

## 适用情形与决策

当 Microsoft Store 能识别新版，但反复下载失败时，先确认失败阶段。如果问题位于商店下载链路，可改用 OpenAI 官方提供的 MSIX 与配套离线许可证完成升级。

不要把“商店首页能打开”“应用提示更新检查完成”当作更新成功。最终应核对受影响用户的应用包版本，并验证新版启动。

本案例采用原位升级，没有先卸载应用、重置应用数据或清空聊天目录。离线升级解决了本次版本更新，不代表已经修复商店今后的自动下载问题。

## 本次证据

| 检查项 | 观察结果 |
| --- | --- |
| 已安装版本 | `26.917.8451.0`，应用包状态 `Ok` |
| 更新目标 | `26.924.1866.0` |
| 包标识 | `OpenAI.Codex`，包系列 `OpenAI.Codex_2p2nqsd0c76g0` |
| 商店产品 ID | `9PLM9XGG6VKS` |
| 商店更新 | 下载为 0 字节，多次出现 `0x80240440` |
| Windows Update 日志 | `GetExtendedUpdateInfo2` 失败，`FileLocations=0` |
| 应用内日志 | 检测到更新，并出现检查完成／下载完成事件，但当前用户版本没有变化 |

历史日志还出现过 `0x80073D02`（旧版应用占用），但对应的旧更新后来已经安装成功，不能把它直接当作本次失败原因。

Windows Update 日志显示部分请求使用用户代理；普通 HTTPS 请求在直连和代理下都取得了响应。**这不足以确认 Clash 是根因，也不能替代真实更新请求的验证。** 本次明确定位到下载信息获取失败，未完全确定其底层网络原因。

## 1. 确认安装渠道、版本和错误

在受影响用户的 PowerShell 中执行：

```powershell
Get-AppxPackage OpenAI.Codex |
    Select-Object Name, Version, Status, SignatureKind, InstallLocation
```

本案例 `SignatureKind` 为 `Store`。显示名称已是 ChatGPT，但包标识仍是 `OpenAI.Codex`，不要仅按显示名称寻找安装包。

重点检查事件查看器中的日志：

- `Microsoft-Windows-Store/Operational`：产品下载状态与错误。
- `Microsoft-Windows-WindowsUpdateClient/Operational`：后台下载失败。
- `Microsoft-Windows-AppXDeploymentServer/Operational`：安装、注册和应用占用错误。

需要底层细节时，可以使用 `Get-WindowsUpdateLog -LogPath <输出路径>` 导出日志。日志可能包含账户、设备信息或带参数的 URL，不应未经脱敏上传公开仓库。

## 2. 获取官方离线包

从 [OpenAI 官方 Windows 部署文档](https://learn.chatgpt.com/docs/enterprise/windows-deployment#install-from-an-offline-package) 获取匹配架构的 MSIX 和配套离线许可证。该文档也说明了管理员安装要求；执行前应以当时的官方说明为准。

本案例使用的官方文件链接：

- [x64 MSIX](https://persistent.oaistatic.com/codex-app-prod/ChatGPT-x64.msix)
- [离线许可证](https://persistent.oaistatic.com/codex-app-prod/ChatGPT-License.xml)

这些链接可能随发布更新内容。下载时记录实际包版本，不要依据文件名推断版本。ARM64 设备应从官方文档选择对应架构。

下载建议：

- 保存到独立目录，下载中使用 `.part` 后缀，成功后再改成正式文件名。
- 网络慢时可尝试断点续传，但须确认远端文件未变化，并在下载后重新校验。
- 目标机下载不稳定时，可在另一台可信设备下载，再通过局域网传输并比对 SHA-256。
- 本案例最终由 Windows 上的续传完成，Mac 上启动的备用下载被停止。

`winget` 使用 `-s msstore` 时仍走微软商店分发渠道，不能把它视为绕过此次下载故障的独立方案。

## 3. 校验后再安装

```powershell
# 示例目录，请换成实际下载目录
$dir = Join-Path $env:USERPROFILE 'Downloads\ChatGPT-Offline'
$package = Join-Path $dir 'ChatGPT-x64.msix'
$license = Join-Path $dir 'ChatGPT-License.xml'

$signature = Get-AuthenticodeSignature $package
$signature | Select-Object Status, StatusMessage, SignerCertificate
if ($signature.Status -ne 'Valid') {
    throw '安装包签名验证未通过，停止安装。'
}
Get-FileHash $package -Algorithm SHA256
[xml]$licenseXml = Get-Content $license -Raw
```

还应核对包内 `AppxManifest.xml` 的名称、发布者、架构及版本是否与预期一致。XML 可解析只说明文件格式可读取，不替代部署时的许可证校验。自行计算的哈希用于传输比对和留档，不能单独证明文件来源。

本案例核验记录：

- 大小：`876070899` 字节。
- 版本／架构：`26.924.1866.0` / `x64`。
- 数字签名：`Valid`。
- SHA-256：`ED58F738C31CFCABD96D072F0657269166E3F47DA51C6B3A864B682E30F598E1`。

以上大小和哈希仅对应本次文件，不能用于校验之后更新的安装包。

## 4. 部署新版

先保存工作并完全退出 ChatGPT，包括托盘中的后台实例。不要在仍有任务运行时强制结束进程。

在管理员 Windows PowerShell 中，设置好上一步的实际文件路径后执行：

```powershell
$ErrorActionPreference = 'Stop'
Add-AppxProvisionedPackage -Online `
    -PackagePath $package `
    -LicensePath $license `
    -Regions all
```

这一步属于设备级应用预配，会影响设备上用户的应用注册安排；不要误认为它只修改当前用户。本文案例返回 `RestartNeeded: False`。

## 5. 处理“部署成功，但当前用户仍是旧版”

设备级预配成功不等于当前已登录用户立即切换版本。本次部署后查询当前用户仍是 `26.917.8451.0`。

可以按官方说明注销再登录，或在受影响用户的交互式桌面会话中，以该用户的管理员权限执行当前用户安装：

```powershell
# 确认工作已保存；此参数会强制关闭占用应用
Add-AppxPackage -Path $package -ForceApplicationShutdown

Get-AppxPackage OpenAI.Codex |
    Select-Object Version, Status, InstallLocation
```

不要换成另一个管理员账户后，把另一个账户的注册结果当作原用户已更新。

本案例通过 SSH 发起工作，但当前用户注册步骤实际由一个临时计划任务在该用户桌面会话执行：`LogonType=Interactive`、`RunLevel=Highest`。任务执行结果写入本机日志，核对成功后删除任务。普通用户可以直接在桌面 PowerShell 执行，无需创建计划任务。

## 6. 验证并收尾

本次完成后的验证结果：

- 当前用户 `Get-AppxPackage` 显示 `26.924.1866.0`，状态 `Ok`。
- 从新版安装路径启动了 ChatGPT，相关进程响应正常。
- 无需重启 Windows。
- 临时安装和启动计划任务已清理，正式安装包与许可证保留。

仍应检查应用界面和原有项目、聊天是否正常。本次远程验证覆盖包状态与进程响应，没有逐项验证所有历史数据和功能。

如果只是预配成功而当前用户版本未变化，不要立即重复卸载重装。如果安装失败，保留错误码与部署日志，按失败阶段继续处理；不要用删除用户数据或修改 WindowsApps 权限作为默认修复。

## 与相关问题的区别

- [商店初始化失败](microsoft-store-initialization.md)：处理商店打开、应用注册和本机代理访问。
- 本文：商店可检测新版，但后台无法下载时，通过官方离线包完成当前更新。
- [macOS Codex CLI 代理环境](../macos/codex-cli-proxy.md)：适用于另一操作系统及进程启动场景，不能直接套用到 Windows 商店下载服务。
