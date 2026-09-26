# Microsoft Store 初始化失败排查笔记

适用场景：Windows 11 打开 Microsoft Store 时提示初始化失败，尤其是使用 Clash 等本机代理的环境。以下步骤应按检查结果选用，不是每台电脑都需要全部执行。

## 现象与本次观察

- 商店应用包存在，状态为 `Ok`，相关服务没有被禁用。
- 历史部署日志出现 `0x80073D02`，提示应用正在运行，阻止注册操作。
- 系统启用了 `127.0.0.1:7897` 本机代理，但 Microsoft Store 不在环回豁免列表中。这里的端口仅为示例，应以实际配置为准。
- SSH 会话中重新注册遇到 `0x80070005`（拒绝访问）；换到同一用户的交互式桌面会话后注册成功。该错误本身并不能证明所有情况都由 SSH 导致。

修复后确认：重新注册成功，商店启动并响应，且建立了到本机代理的 TCP 连接。记录时尚未取得用户对首页正常显示的确认，因此不把进程与网络恢复等同于完整功能恢复。

## 1. 检查应用和服务

在受影响用户的 Windows PowerShell 中执行：

```powershell
Get-AppxPackage Microsoft.WindowsStore |
    Select-Object Name, PackageFullName, Status, InstallLocation

Get-Service AppXSvc, ClipSVC, InstallService, wuauserv, BITS |
    Select-Object Name, Status, StartType
```

按需启动的服务处于 `Stopped` 不一定异常。不要仅凭这一状态更改全部服务的启动类型。

部署错误可在事件查看器的 `Microsoft-Windows-AppXDeploymentServer/Operational` 日志中检查。

## 2. 修复应用注册

如果日志提示 `0x80073D02`，先关闭商店。随后在受影响用户的桌面 PowerShell 中运行；权限不足时以该用户身份提升权限，不要换成另一个账户：

```powershell
Get-Process WinStore.App -ErrorAction SilentlyContinue | Stop-Process -Force
$store = Get-AppxPackage Microsoft.WindowsStore
if ($null -eq $store) {
    throw '当前用户未安装 Microsoft Store，需要另行检查安装状态。'
}
Add-AppxPackage -DisableDevelopmentMode -Register (
    Join-Path $store.InstallLocation 'AppxManifest.xml'
)
```

此操作重新注册现有商店包，不是卸载全部 Windows 应用。远程执行遇到 `0x80070005` 时，优先在受影响用户的桌面会话中重试并检查具体日志，不要直接修改 WindowsApps 文件夹权限。

## 3. 重置商店缓存

在桌面按 `Win + R`，运行：

```text
wsreset.exe
```

等待缓存重置完成，商店通常会自动打开。

## 4. 检查本机代理访问

如果系统使用本机代理，检查配置与端口监听状态：

```powershell
Get-ItemProperty 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings' |
    Select-Object ProxyEnable, ProxyServer, AutoConfigURL

# 将 7897 换成实际代理端口
Get-NetTCPConnection -LocalPort 7897 -State Listen -ErrorAction SilentlyContinue

CheckNetIsolation.exe LoopbackExempt -s
```

Windows 的应用容器网络隔离可能阻止商店访问本机代理。在确认代理正常运行、商店缺少豁免且确实需要使用该代理时，以管理员身份运行：

```powershell
CheckNetIsolation.exe LoopbackExempt -a '-n=Microsoft.WindowsStore_8wekyb3d8bbwe'
```

该设置允许 Microsoft Store 访问本机环回地址；仅为此包添加例外，不要批量放开所有应用。PowerShell 中将完整 `-n=...` 参数加引号，避免参数解析问题。

添加后关闭并重新打开商店：

```powershell
Get-Process WinStore.App -ErrorAction SilentlyContinue | Stop-Process -Force
Start-Process 'ms-windows-store:'
```

如需撤销本次添加的例外：

```powershell
CheckNetIsolation.exe LoopbackExempt -d '-n=Microsoft.WindowsStore_8wekyb3d8bbwe'
```

## 5. 验证结果

```powershell
$storeProcess = Get-Process WinStore.App -ErrorAction SilentlyContinue
$storeProcess | Select-Object Id, Responding
if ($storeProcess) {
    Get-NetTCPConnection -OwningProcess $storeProcess.Id -ErrorAction SilentlyContinue |
        Select-Object RemoteAddress, RemotePort, State
}
```

依次确认商店首页加载、搜索可用，以及需要时的下载功能。到本机代理的 `Established` 连接只证明这一段连接建立，不保证代理上游请求或商店全部功能正常。

本次同时进行了应用注册、缓存重置和代理豁免修复。代理权限是较强的原因线索，但缺少逐项对照验证，不能断言它是唯一原因。

## 参考资料

- [Microsoft 支持：Microsoft Store 打不开](https://support.microsoft.com/zh-cn/accounts-billing/microsoft-store-doesn-t-open)
- [Microsoft：Diagnosing Network Isolation Issues](https://techcommunity.microsoft.com/blog/coreinfrastructureandsecurityblog/diagnosing-network-isolation-issues/2511562)
- [Microsoft Learn：排查 Windows 防火墙中的 UWP 应用连接问题](https://learn.microsoft.com/zh-cn/windows/security/operating-system-security/network-security/windows-firewall/troubleshooting-uwp-firewall)
