# macOS 上 Codex CLI 的代理环境排查

适用场景：浏览器能访问服务，但从终端启动 Codex CLI 时连接失败；本机运行 Clash 等代理工具，希望确认命令行进程是否正确使用代理。

本文整理自一次用户提供的排查记录。原记录称：向 `~/.zshrc` 添加代理变量后，新 shell 中访问 OpenAI API 返回 HTTP 401。本次整理没有重现该网络现场，也没有修改读者的 shell 或桌面应用设置。

## 先区分事实与推断

浏览器使用了 macOS 系统代理，并不保证另一个程序使用相同路径。进程启动方式、继承的环境变量、客户端版本和网络实现都可能影响结果。

原记录把原因概括为“Codex 是 Rust 二进制，reqwest 只认环境变量，不读系统代理”。这不是本文确认的通用结论：语言本身不能决定代理行为；具体库版本、编译特性和调用方式也需要检查。本文查阅的 OpenAI CLI 产品说明不足以支持这一绝对说法。

更稳妥的诊断是：**检查出问题的进程实际继承了哪些代理配置，再用显式代理、环境变量和应用请求分别验证。** 即使本机二进制位于某个 `.app` 内，也不能仅凭路径或 `config.toml` 判断它是通过 GUI 还是终端启动的。

## 1. 确认本机代理监听

以下示例假设 HTTP/混合代理端口为 `7897`，请替换为自己的端口：

```bash
lsof -nP -iTCP:7897 -sTCP:LISTEN
command -v codex
curl --version
```

系统代理可以用 `scutil --proxy` 查看，但系统配置不等于当前 shell 已设置代理环境变量。

## 2. 先临时配置当前终端

```bash
export HTTP_PROXY='http://127.0.0.1:7897'
export HTTPS_PROXY='http://127.0.0.1:7897'
export http_proxy="$HTTP_PROXY"
export https_proxy="$HTTPS_PROXY"

export NO_PROXY='localhost,127.0.0.1,::1,.local'
export no_proxy="$NO_PROXY"
```

`HTTPS_PROXY` 的值仍可使用 `http://`：它描述的是本地代理协议，不是目标 URL 的协议。HTTP 代理可以通过 CONNECT 转发 HTTPS。

大小写同时设置是为了兼容不同客户端；例如 curl 的 HTTP 代理变量使用小写 `http_proxy`，不能只依赖大写版本。

如果混合端口支持 SOCKS，且其他工具需要通用后备代理，可另加：

```bash
export ALL_PROXY='socks5h://127.0.0.1:7897'
export all_proxy="$ALL_PROXY"
```

这不是配置 HTTP/HTTPS 的必需步骤。对 curl 而言，协议专用代理优先于 `ALL_PROXY`；`socks5h` 表示由代理解析目标域名，其他客户端是否支持该 scheme 需要单独确认。

### 正确理解 NO_PROXY

不要直接照抄 `10.*`、`192.168.*`、`*.local` 或包含 `...` 的示意字符串。`NO_PROXY` 没有一个被所有客户端一致实现的通配符规则。

上述基础示例使用 `.local` 作为域名后缀，仍应验证目标客户端的匹配行为。需要访问其他本地服务时，可补充确切的主机名或 IP。不要假设客户端一定会先做 DNS 解析，再用解析出的私网地址匹配排除规则。

对于**确认支持 CIDR 的客户端**，可使用完整私网段：

```bash
export NO_PROXY='localhost,127.0.0.1,::1,.local,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16'
export no_proxy="$NO_PROXY"
```

curl 从 7.86.0 起支持这里的 CIDR 写法；这不保证所有 Codex 版本及其调用的其他工具有同样行为。`172.16.0.0/12` 覆盖 `172.16` 到 `172.31`，仅写 `172.16.*` 不能表示完整范围。

## 3. 分别验证显式代理和环境变量

先明确指定代理，避免已有的排除规则干扰这个测试：

```bash
curl --proxy http://127.0.0.1:7897 --noproxy '' \
  --connect-timeout 10 --max-time 30 \
  -sS -o /dev/null -w 'HTTP_CODE=%{http_code}\n' \
  https://api.openai.com/v1/models
```

再去掉显式代理参数，测试当前 shell 配置：

```bash
curl --connect-timeout 10 --max-time 30 \
  -sS -o /dev/null -w 'HTTP_CODE=%{http_code}\n' \
  https://api.openai.com/v1/models
```

不要添加 `-k` 跳过证书校验。本例不带 API key，收到 API 的 401 认证错误是可预期的：它说明这一次 HTTPS 请求已取得 HTTP 响应，不是登录成功，也不证明 Codex 使用相同代理、认证方式或目标端点。

仅凭 401 状态码不能证明流量经过了 Clash；应结合显式代理测试、代理连接日志，必要时核对响应内容。没有显式代理的测试也可能经由 TUN 等其他路径联网。

最后从配置后的同一终端启动 `codex`，验证实际请求。curl 成功但 Codex 失败时，继续检查认证、实际服务端点、代理规则及应用错误日志；不能直接宣布 Codex 已修复。

## 4. 验证有效后再持久化

如果使用交互式 zsh，可把验证过的 export 配置加入 `~/.zshrc`，先备份并避免重复添加：

```bash
cp ~/.zshrc ~/.zshrc.backup-$(date +%Y%m%d-%H%M%S)
```

上面的备份命令假设文件已存在；不存在时先创建配置文件即可。完成编辑后，新开终端窗口，或在当前终端执行：

```bash
source ~/.zshrc
```

然后重新启动 Codex。环境变量不会自动注入已经运行的 Codex 进程；`~/.zshrc` 主要面向交互式 zsh，也不能保证定时任务和其他非交互进程加载它。

## 5. 桌面应用需要单独排查

从 Finder 或 Dock 启动的 GUI 进程不能假定会加载 `~/.zshrc`。桌面应用也可能有自己的系统代理处理或 shell 环境导入逻辑，应按实际版本验证。

可检查的方向包括应用自身网络设置、代理工具的 TUN 路由，以及启动环境。TUN 会影响更广的网络流量，需要同时考虑本地开发和局域网访问；它不是所有场景都必须启用的选项。

高级排查中，可对当前用户 launchd 上下文的后续进程设置环境变量：

```bash
launchctl setenv HTTP_PROXY http://127.0.0.1:7897
launchctl setenv HTTPS_PROXY http://127.0.0.1:7897
launchctl setenv http_proxy http://127.0.0.1:7897
launchctl setenv https_proxy http://127.0.0.1:7897
launchctl setenv NO_PROXY localhost,127.0.0.1,::1,.local
launchctl setenv no_proxy localhost,127.0.0.1,::1,.local
```

完全退出并重启目标应用后验证。此方法不是永久配置，不会修改已有进程，而且可能影响同一上下文启动的其他应用；不能保证某个应用启动链一定继承这些值，也不能保证应用采用它们。

## 6. 撤销配置

删除 `~/.zshrc` 中本次添加的配置块，并在当前 shell 取消变量，再重新启动受影响进程：

```bash
unset HTTP_PROXY HTTPS_PROXY ALL_PROXY NO_PROXY
unset http_proxy https_proxy all_proxy no_proxy
```

如果执行过上述 launchctl 设置，则撤销本次新增的键：

```bash
for key in HTTP_PROXY HTTPS_PROXY http_proxy https_proxy NO_PROXY no_proxy; do
  launchctl unsetenv "$key"
done
```

若变量原来已有值，应恢复原值。代理关闭后，仍指向本机端口的环境变量可能导致联网失败。

## 核验依据

- [OpenAI 官方：Codex CLI](https://learn.chatgpt.com/docs/codex/cli)——用于确认终端产品场景，不作为所有版本代理实现的证明。
- 本地 `curl --manual`：`ENVIRONMENT`、`--noproxy`、`--proxy` 部分；整理时核对版本为 curl 8.7.1。
- 本地 `man launchctl`：`setenv` / `unsetenv` 对后续进程的作用说明。
- 用户提供的原始排查记录：作为案例材料，其中网络根因和最终 Codex 可用性未在本次整理中独立复现。
