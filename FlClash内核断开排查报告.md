# FlClash 内核频繁断开 — 全排查报告

test

> 环境: Windows 11 Pro / FlClash v0.8.92 / 企业安全 (Threatbook OneSEC + ESET Endpoint + Windows Defender)
> 排查日期: 2026-03-10
> 排查工具: PowerShell + Windows Security Audit + 源码分析

---

## 1. 问题描述

FlClash 代理工具在日常使用中频繁出现 **"内核断开"** 提示，代理服务中断，需要手动点击重启内核才能恢复。每天发生多次，严重影响工作。

**症状表现:**

- FlClash GUI 显示 "内核断开"
- 系统代理 (127.0.0.1:7890) 失效，网络不通
- FlClashCore.exe 进程消失，FlClash.exe (GUI) 仍在运行
- 手动重启内核后恢复，但过段时间再次断开

---

## 2. 排查过程总览

| 阶段 | 排查方向                  | 手段                          | 结论                                                                                  |
| :--: | ------------------------- | ----------------------------- | ------------------------------------------------------------------------------------- |
|  1   | Threatbook (微步) 查杀    | AegisCore 日志分析            | **确认** — 日志命中 `扫描状态[成功命中威胁] FlClashCore.exe`，管理员加白后解决此路径  |
|  2   | ESET 拦截                 | FlClash 日志 + ESET 日志      | **部分确认** — 日志出现 `403 Blocked by ESET Security`，加白后仍复现                  |
|  3   | WSL/Hyper-V VmSwitch 干扰 | VmSwitch 事件关联分析         | **排除** — 禁用 WSL 服务后问题仍复现，VmSwitch 每 15 分钟周期事件与死亡时间吻合属巧合 |
|  4   | FlClash 程序自身 Bug      | Security 4689 审计 + 源码分析 | **确认根因** — Exit Status = `0x0`，无外部 4688 进程创建，程序自行退出                |

---

## 3. 详细排查过程

### 3.1 阶段一: 安全软件排查 — Threatbook (微步 OneSEC)

**操作:**

- 检查 Threatbook AegisCore 日志目录: `C:\Program Files (x86)\Threatbook\Agent\data\gol\AegisCore\*.log`
- 搜索关键词: `FlClash`、`成功命中威胁`

**证据:**

```
扫描状态[成功命中威胁] 文件路径[...FlClashCore.exe]
```

**处置:** 联系管理员将 FlClashCore.exe 加入 Threatbook 白名单。

**结果:** 该路径杀进程问题解决，但内核断开仍间歇性复现 → 排查继续。

### 3.2 阶段二: 安全软件排查 — ESET Endpoint Security

**操作:**

- 检查 FlClash 自身日志，搜索 ESET 相关拦截
- 检查 ESET 日志路径: `C:\ProgramData\ESET\` 系列目录
- 检查 ESET Inspect Connector 日志

**证据:**

```
403 Blocked by ESET Security
```

**处置:** 将 FlClash 相关程序加入 ESET 排除列表。

**结果:** 减少了部分触发，但核心问题仍未根除 → 排查继续。

### 3.3 阶段三: WSL / Hyper-V VmSwitch 网络干扰假说

**假说:** WSL 的 Hyper-V VmSwitch 虚拟网络适配器周期性重置，可能导致 FlClash 的 localhost TCP 连接中断。

**操作:**

1. 运行 `check_wsl_correlation.ps1` 分析全天 VmSwitch 事件分布
2. 对比 FlClashCore 死亡时间与 VmSwitch 事件时间
3. 运行 `shutdown_wsl.ps1` 关闭 WSL
4. 运行 `disable_wsl.ps1` 彻底禁用 WslService
5. 验证 WSL 进程、VmSwitch 事件均已清零
6. 继续监控 FlClashCore 存活状态

**证据:**

- VmSwitch 事件约每 15 分钟一轮，与部分死亡时间存在时间吻合
- 但禁用 WSL 后，VmSwitch 事件归零，**FlClashCore 仍然断开**

**结论:** **排除** — 相关性 ≠ 因果性。VmSwitch 是巧合关联。

**交付物:**

- `~/.claude/scripts/check_wsl_correlation.ps1` — VmSwitch 事件关联分析
- `~/.claude/scripts/shutdown_wsl.ps1` — 快速关闭 WSL
- `~/.claude/scripts/disable_wsl.ps1` — 彻底禁用 WSL 服务

### 3.4 阶段四: Windows 安全审计 — 定位真正死因

**关键发现:** Windows 默认不记录进程终止事件，必须手动开启审计。

**操作:**

1. 使用 GUID 开启进程创建/终止审计 (避免中英文系统名差异):
   ```powershell
   auditpol /set /subcategory:"{0CCE922B-69AE-11D9-BED3-505054503030}" /success:enable /failure:enable
   auditpol /set /subcategory:"{0CCE922C-69AE-11D9-BED3-505054503030}" /success:enable /failure:enable
   ```
2. 运行 `flclash_killer_trace.ps1` 持续监控，等待下一次死亡
3. 死亡发生时自动捕获 Security 4689/4688、Application 1000/1001/1002 事件

**铁证:**

```
Security Event 4689:
  进程名: FlClashCore.exe
  Exit Status: 0x0
```

- `Exit Status = 0x0` → 程序**正常退出** (不是崩溃、不是被杀)
- 同时间窗口无 Security 4688 创建 taskkill/cmd 等外部进程
- 无 Application 1000 崩溃事件

**结论:** **FlClashCore.exe 是自己退出的**，根因在程序代码内部。

**交付物:**

- `~/.claude/scripts/flclash_killer_trace.ps1` — 死因追踪脚本 (核心取证工具)
- `~/.claude/scripts/check_death_2100.ps1` — 特定时间窗口事件分析
- `~/.claude/scripts/flclash_monitor.ps1` — 早期版本进程监控
- `~/.claude/scripts/check_flclash_events.ps1` — FlClash 相关事件日志搜索
- `~/.claude/scripts/check_flclash_full.ps1` — 全面状态检查

---

## 4. 源码分析与根因定位

### 4.1 FlClash 架构

```
FlClash.exe (Flutter/Dart GUI)
  |-- TCP ServerSocket (127.0.0.1, 随机端口)
  |     ↕ JSON-over-TCP (行分隔协议)
  |-- FlClashCore.exe (Go/mihomo 内核)
  |     └── 监听 port 7890 (HTTP/SOCKS 代理)
  └── FlClashHelperService.exe (Rust Windows Service)
        └── HTTP API :47890 (/start /stop /ping)
```

### 4.2 IPC 通信流程

1. Dart 创建 `ServerSocket.bind(127.0.0.1, 0)` 获取随机端口
2. 通过 Helper Service (或直接 `Process.start`) 启动 Go Core，传入端口号
3. Go Core `net.Dial("tcp", "127.0.0.1:{port}")` 连回 Dart
4. 双向 JSON 行协议: Dart 发 Action → Go 返回 ActionResult

### 4.3 Go Core 退出路径 (`core/server.go`)

唯一的 exit code 0 路径:

```
main() → startServer(arg)
  └── for { reader.ReadString('\n') } 循环
      ├── ReadString error (EOF / 网络断开) → return → exit 0
      └── json.Unmarshal error (数据损坏) → return → exit 0
```

**任何读取错误都会导致静默退出，无日志、无重试。**

### 4.4 Dart 侧检测 (`lib/core/service.dart`)

```dart
socket.transform(...).listen(...).onDone(() {
    _handleInvokeCrashEvent();  // 触发 CoreEvent.crash → GUI 显示 "内核断开"
});
```

### 4.5 发现的代码缺陷

| 严重度 | 缺陷                     | 文件                            | 影响                                                  |
| :----: | ------------------------ | ------------------------------- | ----------------------------------------------------- |
|   P0   | Go 并发写无锁            | `core/server.go` send()         | 多 goroutine 同时写 socket → 数据交叉 → Dart 解析失败 |
|   P0   | 读循环静默退出           | `core/server.go:74`             | TCP 断开无任何日志，无法追踪死因                      |
|   P0   | 无 TCP KeepAlive         | `core/server.go`                | 空闲连接可能被 OS/防火墙静默回收                      |
|   P0   | goroutine 无 recover     | `core/server.go:93`             | handler panic 直接杀死整个进程                        |
|   P1   | Dart socket 流无 onError | `lib/core/service.dart:74`      | transform 错误未处理                                  |
|   P1   | Windows 桌面端无自动重连 | `lib/manager/core_manager.dart` | crash 后不重启 Core (Android 有)                      |

---

## 5. 修复方案与实施

### 5.1 Go Core 修复 (`core/server.go`) — 已完成

```go
var (
    conn     net.Conn
    connLock sync.Mutex  // [新增] 保护并发写
)

func send(data []byte) {
    connLock.Lock()           // [新增] 加锁
    defer connLock.Unlock()
    if conn == nil { return }
    _, err := conn.Write(append(data, []byte("\n")...))
    if err != nil {
        fmt.Fprintf(os.Stderr, "send error: %v\n", err)  // [新增] 错误日志
    }
}

func startServer(arg string) {
    // ... 连接建立 ...

    // [新增] TCP KeepAlive 防空闲回收
    if tcpConn, ok := conn.(*net.TCPConn); ok {
        tcpConn.SetKeepAlive(true)
        tcpConn.SetKeepAlivePeriod(30 * time.Second)
    }

    for {
        data, err := reader.ReadString('\n')
        if err != nil {
            fmt.Fprintf(os.Stderr, "read error: %v\n", err)  // [新增] 读错误日志
            return
        }
        // ...
        // [新增] goroutine recover 防 panic 杀进程
        go func() {
            defer func() {
                if r := recover(); r != nil {
                    fmt.Fprintf(os.Stderr, "handleAction panic: %v, method=%s\n", r, action.Method)
                    result.error(fmt.Sprintf("panic: %v", r))
                }
            }()
            handleAction(action, result)
        }()
    }
}
```

### 5.2 Dart 桌面端自动重连 (`lib/manager/core_manager.dart`) — 已完成

```dart
@override
Future<void> onCrash(String message) async {
    // ... 原有清理逻辑 ...
    await coreController.shutdown(false);
    // [新增] 桌面端自动重连 (与 Android tryStartCore 对齐)
    if (system.isDesktop) {
        await Future.delayed(const Duration(seconds: 2));
        await appController.tryStartCore(ref.read(isStartProvider));
    }
}
```

### 5.3 构建 Debug 版

```bash
# 仅编译 Go Core (最小测试)
cd D:\git\FlClash-git\core
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build -ldflags="-w -s" -tags=with_gvisor -o FlClashCore_patched.exe .

# 完整构建 (Flutter GUI + Go Core + Helper Service)
dart ./setup.dart windows --arch amd64
# 输出: dist/FlClash-setup-amd64.exe + dist/FlClash-amd64.zip
```

### 5.4 部署修复版

```powershell
# 停止进程
taskkill /F /IM FlClash.exe; taskkill /F /IM FlClashCore.exe

# 备份原版
copy "C:\Program Files\FlClash\FlClashCore.exe" "C:\Program Files\FlClash\FlClashCore.exe.bak"

# 复制 Release 构建产物
copy "D:\git\FlClash-git\build\windows\x64\runner\Release\*" "C:\Program Files\FlClash\" -Recurse -Force

# 重启
Start-Process "C:\Program Files\FlClash\FlClash.exe"
```

> **注意:** 替换 Core 后 SHA256 与 HelperService 内置 TOKEN 不匹配，Helper 启动 Core 会失败。FlClash 有 fallback: Helper 失败后走 `Process.start` 直接启动 Core (`lib/core/service.dart:110`)。

---

## 6. 应急方案: 看门狗

在源码修复验证期间，部署看门狗脚本作为兜底保障。

### 6.1 看门狗核心逻辑

```
每 5 秒轮询 FlClashCore 进程
  ├── 存活 → 跳过 (每 10 分钟心跳日志)
  │    └── 检查系统代理是否意外关闭 → 自动恢复
  └── 死亡 → 等 3 秒确认 → 执行恢复:
       ├── 方法 1: Restart-Service FlClashHelperService
       └── 方法 2: 重启 FlClash GUI
       冷却 60 秒防止重启风暴, 最多重试 3 次
```

### 6.2 注册为计划任务 (开机自启)

```powershell
powershell -ExecutionPolicy Bypass -File ~/.claude/scripts/register_watchdog_task.ps1
```

### 6.3 管理命令

```powershell
Get-Content D:\download\flclash_watchdog.log -Tail 20   # 查看日志
Stop-ScheduledTask -TaskName FlClashWatchdog              # 停止
Start-ScheduledTask -TaskName FlClashWatchdog             # 启动
Unregister-ScheduledTask -TaskName FlClashWatchdog        # 删除
```

---

## 7. 全部交付物清单

### 7.1 排查/监控脚本

| 文件                                          | 用途                | 说明                                                                    |
| --------------------------------------------- | ------------------- | ----------------------------------------------------------------------- |
| `~/.claude/scripts/flclash_killer_trace.ps1`  | **死因追踪** (核心) | 开启审计 + 轮询监控 + 死亡时自动抓取 4689/4688/1000 事件 + 安全软件快照 |
| `~/.claude/scripts/flclash_monitor.ps1`       | 进程监控 (早期版)   | 轮询 + WMI 事件 + 死亡时检查安全日志和应用崩溃日志                      |
| `~/.claude/scripts/full_investigation.ps1`    | 全面状态采集        | WSL/VmSwitch/Threatbook/ESET/防火墙/端口/代理 一站式检查                |
| `~/.claude/scripts/deep_investigation.ps1`    | 深度调查            | ESET trace.log/Inspect Connector/VmSwitch 时间窗口详查                  |
| `~/.claude/scripts/check_death_2100.ps1`      | 特定死亡事件分析    | 21:00 时段 4689/4688/Application Error + Threatbook/ESET 检查           |
| `~/.claude/scripts/check_flclash_events.ps1`  | 事件日志搜索        | Application/System/Sysmon 中 FlClash 相关事件                           |
| `~/.claude/scripts/check_flclash_full.ps1`    | 进程/端口/代理状态  | 所有 FlClash 进程、监听端口、命令行、代理开关状态                       |
| `~/.claude/scripts/check_proxy_status.ps1`    | 代理状态检查        | 注册表 ProxyEnable/ProxyServer + 端口 7890/3519 监听                    |
| `~/.claude/scripts/check_parent.ps1`          | 父进程溯源          | FlClashCore/FlClash/HelperService 进程树 + 版本信息                     |
| `~/.claude/scripts/check_wsl_correlation.ps1` | WSL 关联分析        | 全天 VmSwitch 事件分布 + 多时段死亡关联 + Docker/WSL 检查               |

### 7.2 应急恢复脚本

| 文件                                           | 用途              | 说明                                                   |
| ---------------------------------------------- | ----------------- | ------------------------------------------------------ |
| `~/.claude/scripts/flclash_watchdog.ps1`       | **看门狗** (核心) | 自动检测死亡 + 重启 Core/GUI + 恢复系统代理 + 日志轮转 |
| `~/.claude/scripts/register_watchdog_task.ps1` | 注册计划任务      | 将看门狗注册为开机自启的 Windows 计划任务              |
| `~/.claude/scripts/shutdown_wsl.ps1`           | 关闭 WSL          | wsl --shutdown + 停止服务 + 验证                       |
| `~/.claude/scripts/disable_wsl.ps1`            | 禁用 WSL          | 设置 WslService 为 Disabled + 完全关闭 + 验证          |

### 7.3 开源项目代码改进

| 文件                                               | 修改内容                                                                          |
| -------------------------------------------------- | --------------------------------------------------------------------------------- |
| `D:\git\FlClash-git\core\server.go`                | Go Core: sync.Mutex 并发写保护 + TCP KeepAlive + 读写错误日志 + goroutine recover |
| `D:\git\FlClash-git\lib\manager\core_manager.dart` | Dart: 桌面端 onCrash 自动重连 (对齐 Android 行为)                                 |
| `D:\git\FlClash-git\lib\core\service.dart`         | Dart: socket 流 onError 处理 (建议项)                                             |

### 7.4 构建产物

| 产物             | 路径                                                   | 说明                   |
| ---------------- | ------------------------------------------------------ | ---------------------- |
| Debug 版 Go Core | `D:\git\FlClash-git\core\FlClashCore_patched.exe`      | 单独编译的修复版 Core  |
| 完整 Release 版  | `D:\git\FlClash-git\build\windows\x64\runner\Release\` | Flutter 完整构建输出   |
| 原版备份         | `C:\Program Files\FlClash\FlClashCore.exe.bak`         | 修改前的原版 Core 备份 |

### 7.5 知识沉淀

| 文件                                                         | 用途                                                             |
| ------------------------------------------------------------ | ---------------------------------------------------------------- |
| `~/.claude/skills/windows-process-killer-diagnosis/SKILL.md` | 方法论 SKILL: EPAV 排查框架 + 诊断决策树 + 看门狗方案 + 构建指南 |

---

## 8. 关键教训

1. **先开审计再排查** — 没有 Security 4689 事件一切都是盲猜
2. **auditpol 必须用 GUID** — 中文 Windows 下用英文子类别名 `"Process Creation"` 会报 `0x00000057` 错误
3. **相关性 ≠ 因果性** — VmSwitch 每 15 分钟一轮的事件与死亡时间吻合是纯巧合
4. **Exit Status 是铁证** — `0x0` 明确表示正常退出，不是被杀、不是崩溃
5. **多因素叠加** — 同一问题可能有多个原因 (Threatbook 查杀 + ESET 拦截 + 自身 Bug)
6. **Bash 转义陷阱** — 从 Git Bash 执行 PowerShell 时 `$_.Message` 会被 bash extglob 破坏，必须写 .ps1 文件

---

## 9. 相关 GitHub Issues

- [#927](https://github.com/chen08209/FlClash/issues/927) — HelperService 不重启 Core (not planned)
- [#1331](https://github.com/chen08209/FlClash/issues/1331) — Core 反复重启循环
- [#1090](https://github.com/chen08209/FlClash/issues/1090) — 运行中偶尔崩溃/断网
