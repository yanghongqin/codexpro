# CodexPro New Windows 固定地址配置记录

> 记录日期：2026-08-11  
> 用途：记录本机已经验证可用的 CodexPro New + Cloudflare Named Tunnel 固定地址配置，避免以后重启、换网或重新配置 ChatGPT Connector 时重复踩坑。

## 1. 当前已经验证的环境

本次实际使用环境：

- 系统：Windows，使用 PowerShell 启动 CodexPro。
- CodexPro 全局安装版本：`0.30.0`。
- 默认工作区：`D:\project`。
- CodexPro 源码仓库：`D:\project\codexpro`。
- Cloudflare Named Tunnel：`codexpro-new-pc`。
- 固定公网域名：`codexpro-new.seanyoungzone.com`。
- 固定 HTTP MCP Token 文件：`$env:USERPROFILE\.codexpro\http-token`。
- Bash 权限：`full`。

当前源码仓库里的 `package.json` 仍是 `0.29.0`，而电脑实际调用的全局 `codexpro` 是 `0.30.0`。因此排查 CLI 参数时，应优先以：

```powershell
codexpro --version
codexpro --help
```

的实际输出为准。

本机 `0.30.0` 已确认支持：

```text
--token-file <path>
```

而仓库里的 `0.29.0` 源码帮助文本还没有这个参数。不要因为旧仓库代码没写 `--token-file` 就误判当前命令不可用。

---

## 2. 这次最终解决的两个“不固定”问题

普通 Cloudflare Quick Tunnel 有两个会导致 ChatGPT Connector 经常失效的问题：

1. **公网 URL 会变。**
   普通 `cloudflare` quick tunnel 每次重新启动都可能拿到新的随机地址。

2. **MCP Token 也可能变。**
   如果不明确指定固定 token，隧道模式下 CodexPro 可以自动生成 HTTP MCP bearer token。重启后 token 变化，就算域名固定，ChatGPT Connector 里的完整 Server URL 仍然会变化。

本次配置把这两部分都固定下来：

```text
固定域名
codexpro-new.seanyoungzone.com
        +
固定 Named Tunnel
codexpro-new-pc
        +
固定 Token 文件
%USERPROFILE%\.codexpro\http-token
        =
ChatGPT Connector 地址长期不变
```

核心不是“只固定 Cloudflare URL”，而是 **hostname 和 MCP token 两个都固定**。

---

## 3. 唯一 MCP Token

这次配置明确只保留一份长期使用的 CodexPro HTTP MCP token，并放到：

```text
%USERPROFILE%\.codexpro\http-token
```

PowerShell 路径写法：

```powershell
$env:USERPROFILE + "\.codexpro\http-token"
```

如果以后需要重新生成一份新的 token，可以使用：

```powershell
$dir = Join-Path $env:USERPROFILE ".codexpro"
New-Item -ItemType Directory -Force $dir | Out-Null

$bytes = New-Object byte[] 32
[System.Security.Cryptography.RandomNumberGenerator]::Fill($bytes)
$token = [Convert]::ToHexString($bytes).ToLowerInvariant()

Set-Content -Path (Join-Path $dir "http-token") -Value $token -NoNewline
Remove-Variable token
```

**正常情况下不要重复执行这段生成命令。** 重新生成 token 就等于主动更换 Connector 密钥，ChatGPT 里原来的 Connector URL 也必须同步更新。

安全要求：

- 不要把 `http-token` 放进任何 Git 仓库。
- 不要把真实 token 写进本文档。
- 不要在截图、聊天记录、Issue 或日志里公开真实 token。
- Token 泄露时，应重新生成并同步更新 ChatGPT Connector。

---

## 4. Cloudflare Named Tunnel

这次使用的不是随机 Quick Tunnel，而是固定 Named Tunnel：

```text
Tunnel Name: codexpro-new-pc
Hostname:    codexpro-new.seanyoungzone.com
```

Cloudflare 侧只需要保证这个 Named Tunnel 和 DNS hostname 的映射一直存在。

如果以后完全重建 Cloudflare 配置，一般逻辑是：

```text
创建 Named Tunnel
        ↓
把 codexpro-new.seanyoungzone.com 路由到该 Tunnel
        ↓
以后一直复用同一个 Tunnel Name + Hostname
```

已经配置成功后，**日常启动不需要重新创建 tunnel，也不需要重新生成 URL**。

---

## 5. 以后日常就用这一条命令

在 Windows PowerShell 中运行：

```powershell
codexpro stable --root "D:\project" --hostname "codexpro-new.seanyoungzone.com" --tunnel-name "codexpro-new-pc" --token-file "$env:USERPROFILE\.codexpro\http-token" --bash full
```

参数含义：

| 参数 | 作用 |
|---|---|
| `stable` | 使用 Cloudflare Named Tunnel，而不是每次生成随机 URL 的 Quick Tunnel |
| `--root "D:\project"` | 把整个 `D:\project` 作为 CodexPro 可访问工作区 |
| `--hostname ...` | 固定公网 hostname |
| `--tunnel-name ...` | 固定使用 `codexpro-new-pc` Named Tunnel |
| `--token-file ...` | 每次都读取同一份 HTTP MCP token |
| `--bash full` | 给 ChatGPT/CodexPro 完整 bash 执行权限 |

`--bash full` 和“固定 URL”没有关系，只决定 shell 权限。如果以后希望更严格，可以改成 `--bash safe`，不会影响域名或 token 的稳定性。

**不要改回普通的：**

```powershell
codexpro start --root "D:\project"
```

默认 Cloudflare Quick Tunnel 的公网 URL 可能在重启后变化，这正是本次配置要避免的问题。

---

## 6. ChatGPT Connector 只配置一次

ChatGPT Connector 的 Server URL 使用：

```text
https://codexpro-new.seanyoungzone.com/mcp?codexpro_token=<http-token 文件里的真实 token>
```

认证方式保持 CodexPro 当前这种 URL token 方案即可。

只要下面两项没有改变：

```text
codexpro-new.seanyoungzone.com
%USERPROFILE%\.codexpro\http-token
```

以后关闭 CodexPro、重启电脑、重新启动 CodexPro，都不需要重新编辑 ChatGPT Connector。

本次已经实际验证：**连续重新启动后生成的 Connector URL 保持一致。**

---

## 7. 换网络以后会怎样

从 Wi-Fi 换到热点、换路由器、IP 地址变化时，当前 Cloudflare Tunnel 连接可能会短暂断开，这是正常的网络切换现象。

但 Named Tunnel 的优势是：

```text
本机公网 IP 可以变
网络可以变
Cloudflare 到本机的连接可以重连

但公网 hostname 不变：
codexpro-new.seanyoungzone.com
```

因此正常处理方式是：

1. 先看 CodexPro/cloudflared 是否自动恢复连接。
2. 如果没有恢复，结束当前进程。
3. 重新运行第 5 节同一条 `codexpro stable ...` 命令。
4. **不需要修改 ChatGPT Connector。**

换网真正可能变化的是“当前连接状态”，不是固定 URL 本身。

---

## 8. 目前不需要强制 HTTP/2

这次稳定化的关键是：

- Named Tunnel；
- 固定 hostname；
- 固定 `--token-file`。

**不是强制 HTTP/2。**

因此当前日常命令里不需要额外加“强制 HTTP/2”的配置。

只有以后在某个特定网络上反复出现 Cloudflare Tunnel 无法建立连接，并且确认是 QUIC/UDP 网络限制导致时，才把强制 HTTP/2 当作单独的网络兼容性排查手段。

不要为了“防止换网断开”默认强制 HTTP/2；换网导致旧连接瞬时中断和协议选择是两个不同问题。

---

## 9. 进程关闭、电脑重启以后

固定 hostname 并不代表本地服务永远在线。

如果运行 CodexPro 的 PowerShell 窗口被关闭，或者电脑关机：

```text
公网 hostname 仍然是原来的 hostname
但本地 CodexPro MCP 服务当前不在线
```

重新开机后，只需要再次执行：

```powershell
codexpro stable --root "D:\project" --hostname "codexpro-new.seanyoungzone.com" --tunnel-name "codexpro-new-pc" --token-file "$env:USERPROFILE\.codexpro\http-token" --bash full
```

服务恢复后，ChatGPT 继续使用原来的 Connector URL。

---

## 10. 常见问题快速判断

### A. 每次启动 URL 又变了

先确认是不是误用了 Quick Tunnel：

```powershell
codexpro start
```

应该使用：

```powershell
codexpro stable ...
```

并同时带固定的：

```text
--hostname
--tunnel-name
--token-file
```

### B. 域名没变，但 ChatGPT 提示未授权 / 401

优先检查 ChatGPT Connector URL 里的 `codexpro_token` 是否和：

```text
%USERPROFILE%\.codexpro\http-token
```

内容一致。

不要直接在公开日志里打印真实 token。

### C. `--token-file` 提示未知参数

运行：

```powershell
codexpro --version
codexpro --help
```

本机已经验证的全局版本是 `0.30.0`，并且 `--help` 明确包含：

```text
--token-file <path>       Read the HTTP MCP bearer token from a mode-0600 file.
```

如果没有这项，说明当前终端调用的很可能是另外一份旧 CodexPro CLI。

### D. 换网后暂时连不上

先重新运行同一条 `stable` 命令。不要重新生成 tunnel、hostname 或 MCP token。

### E. 在 WSL 里提示找不到 `cloudflared`

本次成功配置和启动是在 Windows PowerShell 环境完成的。Windows 和 WSL 的 PATH、二进制位置不是同一个环境。

因此不要仅凭 WSL 中：

```text
cloudflared: command not found
```

就判断 Windows PowerShell 下的 CodexPro Stable 配置失效。日常启动和故障验证优先使用本次已经验证成功的 Windows PowerShell 环境。

---

## 11. 本次配置的最终结论

以后正常情况下，只需要记住下面两件事。

### 启动 CodexPro New

```powershell
codexpro stable --root "D:\project" --hostname "codexpro-new.seanyoungzone.com" --tunnel-name "codexpro-new-pc" --token-file "$env:USERPROFILE\.codexpro\http-token" --bash full
```

### ChatGPT Connector

```text
https://codexpro-new.seanyoungzone.com/mcp?codexpro_token=<固定 token>
```

只要不主动修改 hostname、Named Tunnel 或 `http-token`，就不需要因为 CodexPro 重启、电脑重启或普通换网而重新配置 ChatGPT Connector。
