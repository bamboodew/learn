# Windows 代理配置指南 — 开发者必知

> 适用场景: 企业环境 (CorpLink VPN) + FlClash 代理 + 开发工具链

---

## 1. 代理是什么？一句话理解

你的电脑要访问外网 (如 github.com)，不是直接连，而是**先把请求交给一个"中间人"（代理服务器）**，由它帮你转发。

```
你的电脑 → 代理 (127.0.0.1:7890) → 外网 (github.com)
```

FlClash 就是这个代理，它在本地 `127.0.0.1:7890` 开了个门，所有走这个门的流量会被转发出去。

---

## 2. Windows 上的 4 层代理配置

不同的程序从不同的地方读代理设置，这就是你"懵"的根源。一共 4 层，**互相独立**：

```
┌─────────────────────────────────────────────────┐
│  第 4 层: 各工具自己的代理配置                      │
│  (git config、.npmrc、gradle.properties 等)       │
├─────────────────────────────────────────────────┤
│  第 3 层: 环境变量                                 │
│  (HTTP_PROXY、HTTPS_PROXY、ALL_PROXY、NO_PROXY)  │
├─────────────────────────────────────────────────┤
│  第 2 层: Windows 系统代理 (注册表)                 │
│  (ProxyEnable + ProxyServer, 浏览器/WinHTTP 用)   │
├─────────────────────────────────────────────────┤
│  第 1 层: 网络出口                                 │
│  (CorpLink VPN、公司防火墙、物理网络)               │
└─────────────────────────────────────────────────┘
```

### 每层详解

| 层级 | 配置位置 | 谁在读 | 控制命令 |
|:---:|---|---|---|
| **4** | `git config --global http.proxy` | git | 见下文 |
| **4** | `.npmrc` 中 `proxy=` | npm | `npm config set proxy` |
| **3** | `$env:HTTP_PROXY` 等 | curl, dart, flutter, pip, go, wget 等大部分 CLI 工具 | `$env:HTTP_PROXY="..."` |
| **2** | 注册表 `Internet Settings` | 浏览器(Chrome/Edge)、部分 Windows 应用 | FlClash 自动管理 |
| **1** | CorpLink VPN | 所有流量 | IT 管理员控制 |

---

## 3. 各层查看与设置命令

### 第 4 层: Git 代理

```powershell
# 查看
git config --global http.proxy
git config --global https.proxy

# 设置
git config --global http.proxy http://127.0.0.1:7890
git config --global https.proxy http://127.0.0.1:7890

# 清除
git config --global --unset http.proxy
git config --global --unset https.proxy
```

> Git 只读自己的配置，不读环境变量 HTTP_PROXY（大部分情况下）。

### 第 3 层: 环境变量

```powershell
# 查看 (当前 PowerShell 会话)
echo $env:HTTP_PROXY
echo $env:HTTPS_PROXY

# 设置 (仅当前会话，关窗口就没了)
$env:HTTP_PROXY="http://127.0.0.1:7890"
$env:HTTPS_PROXY="http://127.0.0.1:7890"

# 清除
$env:HTTP_PROXY=""
$env:HTTPS_PROXY=""
```

> **重点**: Flutter、Dart、Go、pip、curl 等命令行工具主要读这一层。
> 每开一个新 PowerShell 窗口，环境变量都是空的，需要重新设置。

### 第 2 层: Windows 系统代理

```powershell
# 查看
Get-ItemProperty "HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings" |
  Select-Object ProxyEnable, ProxyServer

# 开启
Set-ItemProperty "HKCU:\...\Internet Settings" -Name ProxyEnable -Value 1
Set-ItemProperty "HKCU:\...\Internet Settings" -Name ProxyServer -Value "127.0.0.1:7890"

# 关闭
Set-ItemProperty "HKCU:\...\Internet Settings" -Name ProxyEnable -Value 0
```

> FlClash 启动时自动设 ProxyEnable=1，退出时设为 0。
> Chrome/Edge 读这个设置，命令行工具一般**不读**这个。

### 第 1 层: CorpLink 企业 VPN

```
你无法控制这一层。
CorpLink 会拦截所有 HTTPS 连接，做 SSL 中间人检查。
如果没有代理 (FlClash) 帮你中转，很多国外站点直接连不上或被拦截。
```

---

## 4. 常见场景速查

### 场景 A: 正常开发 (FlClash 运行中)

一切正常，不需要额外配置。FlClash 自动管理系统代理(第2层)，Git 代理(第4层)通常你已设好。

唯一需要注意: **新开 PowerShell 窗口**跑 flutter/dart/go 前，要手动设环境变量:

```powershell
$env:HTTP_PROXY="http://127.0.0.1:7890"
$env:HTTPS_PROXY="http://127.0.0.1:7890"
```

### 场景 B: FlClash 内核断了，命令行报连接拒绝

**症状**: `SocketException: 远程计算机拒绝网络连接, port = xxxx`

**原因**: 代理配置还指向 127.0.0.1:7890，但 FlClash 没在监听。

**解决**:
1. 先启动 FlClash，等内核连接
2. 或者临时清掉所有代理直连:
```powershell
$env:HTTP_PROXY=""
$env:HTTPS_PROXY=""
git config --global --unset http.proxy
git config --global --unset https.proxy
```

### 场景 C: 清掉代理后还是报错，端口随机变

**原因**: CorpLink VPN 在拦截连接。没有 FlClash 帮忙，直连外网被企业防火墙挡了。

**解决**: 必须先启动 FlClash + 设好代理，没有其他办法。

### 场景 D: 只需要编译不需要联网

如果依赖包已经下载过，可以离线构建:
```powershell
# Dart/Flutter 离线
dart pub get --offline
flutter build windows --debug

# Go 离线 (依赖已在 vendor 或 module cache)
go build ./...
```

---

## 5. 完整开发环境代理设置模板

把以下内容保存为 `proxy-on.ps1`，每次开新窗口执行:

```powershell
# proxy-on.ps1 — 开启代理
$proxy = "http://127.0.0.1:7890"
$env:HTTP_PROXY=$proxy
$env:HTTPS_PROXY=$proxy
$env:ALL_PROXY=$proxy
$env:NO_PROXY="localhost,127.0.0.1,::1"
git config --global http.proxy $proxy
git config --global https.proxy $proxy
Write-Host "Proxy ON: $proxy"
```

关闭代理的脚本 `proxy-off.ps1`:

```powershell
# proxy-off.ps1 — 关闭代理
$env:HTTP_PROXY=""
$env:HTTPS_PROXY=""
$env:ALL_PROXY=""
$env:NO_PROXY=""
git config --global --unset http.proxy
git config --global --unset https.proxy
Write-Host "Proxy OFF"
```

---

## 6. 快速诊断流程

遇到网络问题时，按顺序检查:

```
1. FlClash 在运行吗？内核已连接吗？
   → 看 GUI 状态 / Get-Process FlClashCore

2. 7890 端口在监听吗？
   → Get-NetTCPConnection -LocalPort 7890 -State Listen

3. 环境变量设了吗？(flutter/dart/go 需要)
   → echo $env:HTTP_PROXY

4. Git 代理设了吗？(git clone/push 需要)
   → git config --global http.proxy

5. 系统代理开了吗？(浏览器需要)
   → Get-ItemProperty "HKCU:\...\Internet Settings" | Select ProxyEnable
```

**一句话总结**: FlClash 是水龙头，代理配置是水管。水龙头没开(FlClash没运行)，水管接得再好也没水。水龙头开了，但水管没接(没设环境变量)，水也流不到你的工具里。
