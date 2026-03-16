# Windows 下 Markdown 文件完全共享与同步方案

> 场景：同一份内容需要同时出现在多个路径，修改任意一个，其他路径自动同步。  
> 典型用例：`~/.claude/CLAUDE.md` 与 `~/.config/opencode/AGENTS.md` 共享同一份配置。

---

## 方案对比

| 方案 | 原理 | 需要权限 | 跨磁盘 | 断链风险 | 推荐度 |
|------|------|---------|--------|---------|--------|
| **Hardlink** | 多个文件名指向同一 inode | 无需管理员 | ❌ 同磁盘 | 低（原地写入安全） | ⭐⭐⭐ 首选 |
| **Symlink** | 文件名指向另一个路径 | 需管理员或开发者模式 | ✅ | 低 | ⭐⭐ 次选 |
| **手动 copy** | 定期复制 | 无需 | ✅ | 无（但不自动同步） | ❌ 不推荐 |
| **OneDrive/同步盘** | 云端同步 | 无需 | ✅ | 网络依赖 | ⚠️ 有延迟 |

---

## 方案一：Hardlink（推荐）

### 原理

多个文件名（路径）指向磁盘上**同一块数据（inode）**。  
修改任意一个路径的内容 → 所有路径立即同步，因为本质是同一份数据。

```
master-config.md ──┐
CLAUDE.md        ──┼──► [磁盘 inode: 同一块数据]
AGENTS.md        ──┘
```

### 限制

- **必须在同一磁盘分区内**（不能跨 C: D: 等不同盘）
- 删除其中一个文件名，其他仍然有效（数据不丢）
- 编辑器「安全保存」（write-to-temp-then-rename）会断链（见下文处理）

### 创建步骤

```powershell
# 1. 准备真实文件（唯一维护点）
$master = "C:\Users\mi\.claude\master-config.md"

# 2. 删除旧文件（如果已存在）
Remove-Item "C:\Users\mi\.claude\CLAUDE.md" -Force
Remove-Item "C:\Users\mi\.config\opencode\AGENTS.md" -Force

# 3. 创建 hardlink
New-Item -ItemType HardLink -Path "C:\Users\mi\.claude\CLAUDE.md"   -Target $master
New-Item -ItemType HardLink -Path "C:\Users\mi\.config\opencode\AGENTS.md" -Target $master
```

### 验证

```powershell
# 查看所有 hardlink（应显示 3 个路径）
fsutil hardlink list "C:\Users\mi\.claude\master-config.md"

# 验证文件大小一致
(Get-Item "C:\Users\mi\.claude\master-config.md").Length
(Get-Item "C:\Users\mi\.claude\CLAUDE.md").Length
(Get-Item "C:\Users\mi\.config\opencode\AGENTS.md").Length
```

输出示例：
```
\Users\mi\.config\opencode\AGENTS.md
\Users\mi\.claude\master-config.md
\Users\mi\.claude\CLAUDE.md
---
3911
3911
3911
```

### 断链检测与修复

某些工具（如 VSCode 默认安全保存）会「删旧建新」导致断链：

```powershell
# 检测是否断链（hardlink 数量应 = 3，若为 1 则已断链）
(fsutil hardlink list "C:\Users\mi\.claude\master-config.md" | Measure-Object -Line).Lines

# 修复断链（重新建立 hardlink）
Remove-Item "C:\Users\mi\.claude\CLAUDE.md" -Force
New-Item -ItemType HardLink -Path "C:\Users\mi\.claude\CLAUDE.md" -Target "C:\Users\mi\.claude\master-config.md"
# AGENTS.md 同理
```

---

## 方案二：Symlink（次选）

### 原理

文件名 A 是一个「快捷方式」，指向文件名 B 的路径。  
读写 A 时，操作系统自动重定向到 B。

```
CLAUDE.md  ──► master-config.md（真实文件）
AGENTS.md  ──► master-config.md（真实文件）
```

### 前置条件（二选一）

**方式 1：开启 Windows 开发者模式**
```
设置 → 系统 → 开发者选项 → 开发者模式 → 开启
```

**方式 2：以管理员身份运行 PowerShell**

### 创建步骤

```powershell
$master = "C:\Users\mi\.claude\master-config.md"

Remove-Item "C:\Users\mi\.claude\CLAUDE.md" -Force
Remove-Item "C:\Users\mi\.config\opencode\AGENTS.md" -Force

New-Item -ItemType SymbolicLink -Path "C:\Users\mi\.claude\CLAUDE.md"   -Target $master
New-Item -ItemType SymbolicLink -Path "C:\Users\mi\.config\opencode\AGENTS.md" -Target $master
```

### 验证

```powershell
Get-Item "C:\Users\mi\.claude\CLAUDE.md" | Select-Object LinkType, Target
# 输出：LinkType=SymbolicLink  Target=C:\Users\mi\.claude\master-config.md
```

### Symlink vs Hardlink 行为差异

| 行为 | Hardlink | Symlink |
|------|---------|---------|
| 删除 master 文件 | 其他路径仍可用 | 其他路径变死链 |
| `Get-Item` 显示 | 普通文件（看不出链接） | 显示 LinkType=SymbolicLink |
| 跨磁盘 | ❌ | ✅ |
| 需要权限 | ❌ | ✅（管理员或开发者模式） |

---

## 编辑工具兼容性

| 工具 | 保存方式 | Hardlink 安全 | Symlink 安全 |
|------|---------|--------------|-------------|
| OpenCode (Edit tool) | 原地写入 | ✅ | ✅ |
| `Add-Content` / `Set-Content` | 原地写入 | ✅ | ✅ |
| VSCode（默认） | 安全保存（rename） | ⚠️ 会断链 | ✅ |
| VSCode（关闭安全保存） | 原地写入 | ✅ | ✅ |
| Notepad | 原地写入 | ✅ | ✅ |

VSCode 关闭安全保存（settings.json）：
```json
{
  "files.safeSave": false
}
```

---

## 本项目实际配置

```
真实文件：C:\Users\mi\.claude\master-config.md
    │
    ├── hardlink → C:\Users\mi\.claude\CLAUDE.md        (Claude Code 加载)
    └── hardlink → C:\Users\mi\.config\opencode\AGENTS.md (OpenCode 加载)
```

日常维护：只需编辑 `master-config.md`，两个工具链自动同步。

---

## 参考命令速查

```powershell
# 查看文件的所有 hardlink
fsutil hardlink list <文件路径>

# 创建 hardlink
New-Item -ItemType HardLink -Path <新路径> -Target <已有文件路径>

# 创建 symlink（需权限）
New-Item -ItemType SymbolicLink -Path <新路径> -Target <目标路径>

# 查看链接类型
Get-Item <路径> | Select-Object LinkType, Target

# 检查 hardlink 数量
(fsutil hardlink list <路径> | Measure-Object -Line).Lines
```
