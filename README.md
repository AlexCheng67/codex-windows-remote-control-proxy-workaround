# Windows Codex 远程控制启用失败：本地代理环境的临时修复

> 记录日期：2026-10-10。以下方法已使桌面端显示“启用远程控制”成功；尚未测试第二台设备配对及完整远程会话。此案例只说明一次具体环境中的排障结果。

## 症状与环境

- Windows 桌面包 `OpenAI.Codex` 版本 `26.930.7945.0`；检测到的 Codex CLI 版本为 `0.160.1`。
- Windows 用户代理已启用，指向已有的本机 HTTP 代理；WinHTTP 为直连。最初没有设置 `HTTP_PROXY`、`HTTPS_PROXY`、`ALL_PROXY`、`NO_PROXY` 环境变量。
- 桌面端点击“启用远程控制”后提示“无法启用远程控制，请重试”。

## 诊断证据

1. 直连 `https://chatgpt.com` 超时。通过现有本机代理时，TCP 和 TLS 能建立，但 HTTP 返回 `403`。这只说明代理链路可达，不能证明远程控制已连通。
2. 桌面日志出现 `remoteControl/enable` 失败（`errorCode=null`），随后持久 WebSocket 在建立前关闭（`1006`、`open_timeout`）。更早的日志还出现过 `net::ERR_PROXY_CONNECTION_FAILED`。
3. 默认环境运行 `codex doctor` 时，Responses WebSocket 超时；仅为该诊断进程临时设置 `HTTP_PROXY`、`HTTPS_PROXY` 后得到 HTTP `101 Switching Protocols`。这是 Responses WebSocket 的测试，**不是**远程控制 WebSocket 的握手验证。
4. 当前 CLI 能识别 `respect_system_proxy`（开发中，默认 `false`）和 `system_proxy_fallback`（稳定，默认 `true`）。临时开启前者没有解决诊断 WebSocket 超时，因此未将两项写入 `config.toml`。
5. 系统 DNS 与经代理查询的公共 DNS 结果不同，直连公共 DNS 查询也失败；hosts 文件没有 OpenAI 相关映射。现有证据不足以认定 DNS 污染是根因。

这些证据支持“桌面进程未正确使用现有代理”这一判断，但尚不能确定远程控制内部失败的精确环节。

## 已验证的临时方法

先正常退出桌面客户端。打开普通 Windows PowerShell，将下面的 `PORT` 替换为**你自己已有且信任的本机 HTTP 代理端口**，然后执行：

```powershell
$proxy = 'http://127.0.0.1:PORT'
if (Get-Process ChatGPT -ErrorAction SilentlyContinue) { throw '请先退出桌面客户端。' }
if (-not (Test-NetConnection 127.0.0.1 -Port PORT -InformationLevel Quiet)) { throw '本机代理端口未监听。' }

$exe = Join-Path (Get-AppxPackage OpenAI.Codex).InstallLocation 'app\ChatGPT.exe'
$oldHttp = $env:HTTP_PROXY
$oldHttps = $env:HTTPS_PROXY
try {
    $env:HTTP_PROXY = $proxy
    $env:HTTPS_PROXY = $proxy
    Start-Process -FilePath $exe
} finally {
    $env:HTTP_PROXY = $oldHttp
    $env:HTTPS_PROXY = $oldHttps
}
```

使用该方式启动后，桌面端成功启用了远程控制；运行中的应用也与本机代理建立了 TCP 连接。在此之前，正常手动重开客户端仍会报错。

这段命令只在启动应用时临时设置进程环境，不会永久更改 Windows 代理、注册表、DNS、防火墙、认证或 Codex 功能配置。若后续正常启动又出现同样问题，需要再次使用此方式启动。代理成功并不等于远程控制全链路已验证；请以桌面端实际启用结果及后续配对结果为准。

