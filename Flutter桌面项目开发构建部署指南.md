# Flutter 桌面项目开发、构建、部署指南

> 以 FlClash 为实例，适用于 Flutter + Go + Rust 多语言桌面项目

---

## 1. 项目结构一图看懂

```
FlClash/
├── lib/                    ← Dart/Flutter 代码 (GUI 界面)
│   ├── pages/              ← 页面
│   ├── views/              ← 视图组件
│   ├── manager/            ← 管理器 (窗口、核心、托盘等)
│   ├── providers/          ← 状态管理 (Riverpod)
│   ├── common/             ← 工具类、常量
│   └── core/               ← 与 Go 内核通信的 IPC 层
│
├── core/                   ← Go 代码 (代理内核)
│   ├── server.go           ← IPC 服务端
│   └── Clash.Meta/         ← Git 子模块 (mihomo 引擎)
│
├── windows/                ← Windows 平台原生代码 (C++/Rust)
│   └── runner/             ← Windows 入口
│
├── build/                  ← 构建输出 (自动生成，不要手动改)
│   └── windows/x64/runner/
│       ├── Debug/          ← Debug 构建产物
│       └── Release/        ← Release 构建产物
│
├── pubspec.yaml            ← Dart 依赖配置 (类似 package.json)
├── pubspec.lock            ← 依赖锁定文件
├── setup.dart              ← 自定义构建脚本
└── .gitmodules             ← Git 子模块配置
```

---

## 2. 三种语言，三个可执行文件

FlClash 由三种语言编写，最终生成三个 exe：

| 文件 | 语言 | 作用 | 单独编译 |
|---|---|---|---|
| `FlClash.exe` | Dart/Flutter | GUI 界面 | `flutter build windows` |
| `FlClashCore.exe` | Go | 代理内核 | `go build` |
| `FlClashHelperService.exe` | Rust | Windows 服务 | `cargo build` |

**关键理解**：改了哪部分代码，就只需要编译哪部分。

---

## 3. 开发环境搭建

### 3.1 必装工具

| 工具 | 版本 | 用途 | 安装 |
|---|---|---|---|
| Flutter SDK | 3.x | 编译 GUI | https://flutter.dev |
| Go | 1.20+ | 编译内核 | https://go.dev |
| Rust | stable | 编译服务 | https://rustup.rs |
| Git | 最新 | 版本控制 | https://git-scm.com |
| Visual Studio | 2022 | C++ 编译工具链 | 安装"C++桌面开发"工作负载 |

### 3.2 获取源码

```powershell
# 克隆项目（含子模块）
git clone https://github.com/chen08209/FlClash.git
cd FlClash

# 子模块默认用 SSH URL，没有 SSH key 需改为 HTTPS
git config submodule.core/Clash.Meta.url https://github.com/chen08209/Clash.Meta.git
git submodule update --init --recursive

# 安装 Dart 依赖
flutter pub get
```

### 3.3 环境验证

```powershell
flutter doctor      # 检查 Flutter 环境
go version          # 检查 Go
rustc --version     # 检查 Rust
```

---

## 4. Debug vs Release — 什么区别？

| | Debug | Release |
|---|---|---|
| **编译速度** | 快 | 慢 |
| **运行速度** | 慢（有断言检查） | 快（优化后） |
| **文件大小** | 大（含调试符号） | 小（压缩优化） |
| **用途** | 开发调试 | 正式使用 |
| **构建命令** | `flutter build windows --debug` | `flutter build windows --release` |

**开发流程**：改代码 → Debug 构建 → 测试 → 确认没问题 → Release 构建 → 部署

---

## 5. 构建命令速查

### 5.1 只改了 Dart/Flutter 代码（GUI、布局、逻辑）

```powershell
# Debug 版（开发调试用）
flutter build windows --debug
# 产物在: build\windows\x64\runner\Debug\

# Release 版（正式使用）
flutter build windows --release
# 产物在: build\windows\x64\runner\Release\
```

### 5.2 只改了 Go 代码（代理内核）

```powershell
cd core
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build -ldflags="-w -s" -tags=with_gvisor -o FlClashCore_patched.exe .
# 产物: core\FlClashCore_patched.exe
```

只需替换这一个文件，不需要重新编译 Flutter。

### 5.3 完整构建（所有组件）

```powershell
dart ./setup.dart windows --arch amd64
# 产物: dist\FlClash-setup-amd64.exe (安装包)
#       dist\FlClash-amd64.zip (便携版)
```

---

## 6. 部署（把构建产物装到电脑上）

### 6.1 手动部署流程

```powershell
# 第 1 步：关闭正在运行的程序
taskkill /F /IM FlClash.exe
taskkill /F /IM FlClashCore.exe

# 第 2 步：备份（首次）
copy "C:\Program Files\FlClash\FlClashCore.exe" "C:\Program Files\FlClash\FlClashCore.exe.bak"

# 第 3 步：复制新文件
copy "D:\git\FlClash-git\build\windows\x64\runner\Debug\*" "C:\Program Files\FlClash\" -Recurse -Force

# 第 4 步：启动
Start-Process "C:\Program Files\FlClash\FlClash.exe"
```

### 6.2 常见 copy 报错

| 错误 | 原因 | 解决 |
|---|---|---|
| "文件正在被使用" | 程序没关干净 | 先 `taskkill /F` 关掉所有相关进程 |
| "用户映射区域" | 字体/资源文件被锁 | 关闭程序后重试，或重启电脑 |
| "拒绝访问" | 需要管理员权限 | 用管理员 PowerShell 运行 |

### 6.3 快速测试（不部署，直接运行）

开发时推荐直接从构建目录运行，不需要复制：

```powershell
# 先关掉已安装版本
taskkill /F /IM FlClash.exe
taskkill /F /IM FlClashCore.exe

# 直接运行 Debug 版
D:\git\FlClash-git\build\windows\x64\runner\Debug\FlClash.exe
```

优点：快，不怕 copy 出错。
缺点：FlClashCore.exe 的路径不在 Program Files，HelperService 可能找不到（会 fallback 到直接启动）。

---

## 7. 开发调试技巧

### 7.1 增量构建

Flutter 的构建是增量的 — 只重新编译改动的部分。第二次 `flutter build` 比第一次快很多。

### 7.2 热重载 (Hot Reload)

开发时用 `flutter run` 而不是 `flutter build`：

```powershell
flutter run -d windows
```

改了代码保存后自动刷新界面，不需要重新编译。这是开发效率最高的方式。

> 注意：热重载只支持 Dart 代码变更。改了 Go/Rust/C++ 代码必须重新编译。

### 7.3 查看调试日志

```powershell
# Flutter Debug 模式自带控制台输出
# 直接在终端看日志

# Go Core 的 stderr 输出
# 在 FlClash 设置中开启日志
```

---

## 8. Git 子模块

FlClash 的 Go 内核依赖 `Clash.Meta` 作为 Git 子模块（仓库中的仓库）。

```powershell
# 初始化子模块
git submodule update --init --recursive

# 更新子模块到最新
git submodule update --remote

# 查看子模块状态
git submodule status
```

### SSH vs HTTPS 问题

`.gitmodules` 可能配置了 SSH URL (`git@github.com:...`)，没有 SSH key 会失败。手动改为 HTTPS：

```powershell
git config submodule.core/Clash.Meta.url https://github.com/chen08209/Clash.Meta.git
git submodule update --init
```

---

## 9. 依赖管理对比

| 语言 | 配置文件 | 锁文件 | 安装命令 | 类比 |
|---|---|---|---|---|
| Dart/Flutter | `pubspec.yaml` | `pubspec.lock` | `flutter pub get` | npm install |
| Go | `go.mod` | `go.sum` | `go mod download` | npm install |
| Rust | `Cargo.toml` | `Cargo.lock` | `cargo build` | npm install + build |

**关键**：这些命令都需要联网下载依赖。确保代理可用（见《Windows代理配置指南》）。

离线模式（已下载过依赖时）：
```powershell
dart pub get --offline        # Dart
go build ./...                # Go (自动用缓存)
cargo build --offline         # Rust (需要 vendor 目录)
```

---

## 10. 完整开发工作流示例

以本次 FlClash 最大化按钮修复为例：

```
1. 定位问题
   → 阅读源码，找到 lib/common/window.dart

2. 修改代码
   → setMaximizable(false) 改为 setMaximizable(true)

3. 构建 Debug 版
   → flutter build windows --debug

4. 快速测试
   → 直接运行 build\...\Debug\FlClash.exe
   → 点最大化，验证按钮位置正常

5. 构建 Release 版
   → flutter build windows --release

6. 部署
   → 关闭旧版 → 复制新版到 Program Files → 启动

7. 验证
   → 日常使用确认无副作用
```

---

## 11. 常见问题

### Q: 构建报网络错误？
A: 确保代理可用。参考《Windows代理配置指南》设置 `$env:HTTP_PROXY`。

### Q: 改了代码但构建产物没变？
A: 试试清除缓存重新构建：
```powershell
flutter clean
flutter pub get
flutter build windows --debug
```

### Q: 替换 FlClashCore.exe 后 HelperService 启动失败？
A: 正常现象。替换后 SHA256 不匹配，FlClash 会 fallback 到 `Process.start` 直接启动 Core。

### Q: Debug 版和 Release 版能混用吗？
A: 不建议。Debug 版 Flutter DLL 和 Release 版不同，混用可能崩溃。要么全用 Debug，要么全用 Release。
