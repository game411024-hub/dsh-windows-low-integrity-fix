# dsh-windows-low-integrity-fix
DeepSeek Harness Windows 桌面端无法启动的排查与修复：DSH 沙箱把安装目录标成低完整性(Low Integrity)

Fix DeepSeek Harness desktop crashing on Windows when DSH's sandbox marks its install directory as Low Integrity

# DeepSeek Harness Windows 无法启动修复指南

> 双击 `DeepSeek Harness.exe` 秒退 / 无响应？很可能是 **DSH 沙箱把安装目录标成了「低完整性（Low Integrity）」**。

本仓库记录了该问题的完整排查过程、根因、修复步骤和预防方法，并附带一键脚本。

---

## 目录

- [适用场景](#适用场景)
- [现象](#现象)
- [根因](#根因)
- [快速自检](#快速自检)
- [完整修复步骤](#完整修复步骤)
- [一键修复脚本](#一键修复脚本)
- [预防措施](#预防措施)
- [常见问题 FAQ](#常见问题-faq)
- [参考链接](#参考链接)
- [English Summary](#english-summary)
- [免责声明](#免责声明)

---

## 适用场景

- 系统：Windows 10 / 11（与具体版本号无关）
- 应用：DeepSeek Harness 桌面端（Electron 版）
- 特征：软件本身没坏，但**装在工作区目录内**，被 DSH 沙箱标成了低完整性

---

## 现象

- 双击 `DeepSeek Harness.exe`：
  - 完全没反应；
  - 或短暂加载后自动退出；
  - 有时弹 SmartScreen（点「运行」后依然没反应）。
- 日志中可能看到：
  - `crashpad ... not connected`、退出码 `0xFFFF7003`；
  - `Lock file can not be created: 拒绝访问 (0x5)`；
  - `Failed to open persistent cache files ... 拒绝访问 (0x5)`。
- 卸载器报错：
  - `Error writing temporary file. Make sure your temp folder is valid.`

> 注意：SmartScreen 弹窗只是网络连不上微软 SmartScreen 服务器，点「运行」即可，与本次故障无关。

---

## 根因

DSH 的沙箱机制在运行受限命令时，会给「**工作区根目录**」打上 NTFS 的
**低完整性标签（Low Mandatory Level）**，并且该标签**持久保留**。

如果 DSH 桌面端恰好安装在**工作区目录内部**，例如：

```
工作区根目录： D:\Deepseek_Harmess
软件安装目录： D:\Deepseek_Harmess\Deepseek_Harness\DeepSeek Harness.exe
```

那么整个安装目录（包括桌面端自己）都会被标成 Low，导致：

1. 进程以低完整性运行，**写不了**自己位于用户目录（Medium 完整性）里的配置 / 缓存
   → 报 `0x5` 拒绝访问；
2. 无法正常拉起 Chromium / Electron 子进程
   → `crashpad not connected`、秒退。

一句话总结：

> **DSH 沙箱把装在同一目录里的桌面端自己「锁」住了。**

因此，换版本、加 `--no-sandbox --disable-gpu`、删 AppData、改目录权限都无效——
因为根本不是那些问题。

---

## 快速自检

对安装目录（或工作区根目录）执行：

```bat
icacls "D:\Deepseek_Harmess"
```

看输出里有没有这一行：

```
Mandatory Label\Low Mandatory Level:(NW)
```

中文系统会显示为「强制标签\低强制级别」。**出现 Low = 就是它。**
正常应该是 `Medium`，或者干脆没有这一行。

### 快速验证（可选）

用**管理员身份**启动一次，如果能打开，基本坐实是低完整性导致
（管理员 = 高完整性，绕过了 Low 的限制）：

```powershell
Start-Process -FilePath 'D:\Deepseek_Harmess\Deepseek_Harness\DeepSeek Harness.exe' `
  -ArgumentList '--disable-gpu','--no-sandbox' -Verb RunAs
```

---

## 完整修复步骤

> 以下命令把 `D:\Deepseek_Harmess` 作为示例，请替换成你自己的安装目录 / 工作区根目录。

### 第 1 步：完全退出 DeepSeek Harness

右键托盘图标退出，并在任务管理器确认没有 `DeepSeek` / `dsh` 相关进程残留。

### 第 2 步：恢复完整性级别为 Medium（默认）

```bat
icacls "D:\Deepseek_Harmess" /setintegritylevel M /T /C /Q
```

- 如果报「拒绝访问 / Access is denied」，用**管理员终端**再执行一次。

### 第 3 步：清理沙箱残留的拒绝权限

```bat
icacls "D:\Deepseek_Harmess" /remove "Everyone" /T /C /Q
```

> 说明：这条会移除该目录树上所有 `Everyone` 的权限项。
> 该场景下最典型的是沙箱留下的 `Everyone:(CI)(DENY)(DC)`（会导致删不动文件），
> 删除后即可恢复正常。普通用户目录基本没有需要保留的 `Everyone` 项。

### 第 4 步：从工作区登记里移除该目录（防复发）

先备份：

```bat
copy "%USERPROFILE%\.dsh\storages\workspace.json" "%USERPROFILE%\.dsh\storages\workspace.json.bak"
```

打开这个文件：

```
%USERPROFILE%\.dsh\storages\workspace.json
```

做两件事：

1. 删除 `tables.workspaces` 里 `path` 指向该安装目录的那一项；
2. 删除 `global.workspaceIds` 数组里对应的 id。

也可以用下面这段 PowerShell 自动完成（同样会先备份）：

```powershell
$p = "$env:USERPROFILE\.dsh\storages\workspace.json"
Copy-Item $p "$p.bak-$(Get-Date -Format yyyyMMddHHmmss)" -Force

$j = Get-Content $p -Raw | ConvertFrom-Json
$targetPath = 'D:\Deepseek_Harmess'   # 改成你的工作区路径

$ids = @($j.tables.workspaces.PSObject.Properties |
         Where-Object { $_.Value.path -eq $targetPath } |
         ForEach-Object { $_.Name })

foreach ($id in $ids) {
  $j.tables.workspaces.PSObject.Properties.Remove($id)
  $j.global.workspaceIds = @($j.global.workspaceIds | Where-Object { $_ -ne $id })
}

$json = $j | ConvertTo-Json -Depth 30
[System.IO.File]::WriteAllText($p, $json, (New-Object System.Text.UTF8Encoding($false)))
Write-Host "已移除 $($ids.Count) 个工作区: $($ids -join ', ')"
```

### 第 5 步：验证并重新启动

```bat
icacls "D:\Deepseek_Harmess"
```

确认输出里：

- 不再有 `Mandatory Label\Low Mandatory Level`；
- 不再有 `Everyone:(CI)(DENY)(DC)`。

然后直接双击 `DeepSeek Harness.exe`，应能正常打开。

---

## 一键修复脚本

把下面内容保存为 `fix-dsh-low-integrity.ps1`，**用管理员 PowerShell 运行**：

```powershell
# 用法：先改下面这个路径，再以管理员身份运行
$root = 'D:\Deepseek_Harmess'   # ← 改成你的安装目录 / 工作区根目录

if (-not (Test-Path $root)) {
  Write-Host "[ERROR] 路径不存在: $root" -ForegroundColor Red
  exit 1
}

Write-Host "[1/3] 恢复完整性级别为 Medium..." -ForegroundColor Cyan
icacls $root /setintegritylevel M /T /C /Q

Write-Host "[2/3] 清理沙箱残留的 Everyone 权限..." -ForegroundColor Cyan
icacls $root /remove "Everyone" /T /C /Q

Write-Host "[3/3] 当前权限如下，请检查是否还有 Low / Everyone DENY：" -ForegroundColor Cyan
icacls $root

Write-Host "`n完成。请直接双击 DeepSeek Harness.exe 测试。" -ForegroundColor Green
```

> 提醒：脚本只处理「完整性级别 + 残留权限」两步；工作区登记（workspace.json）
> 请按上文第 4 步手动或脚本移除，防止复发。

---

## 预防措施

1. **不要把 DSH 桌面端或任何软件安装在工作区目录里。**
2. 用**单独的空目录**当工作区，例如：

```
D:\dsh-workspace
```

3. 使用 DSH 时，工作区选那个空目录，不要选软件安装目录或其上级目录。

一句话规则：

> **装软件的地方，不要拿来做 DSH 工作区；DSH 工作区里，不要放任何安装的软件。**

---

## 常见问题 FAQ

### 1. 为什么加 `--no-sandbox --disable-gpu` 没用？
因为问题不是 Chromium 沙箱，而是 **NTFS 完整性级别（Low Integrity）**。
这两个参数解决不了低完整性进程写不了配置 / 起不了子进程的问题。

### 2. 为什么管理员身份能打开？
管理员进程运行在**高完整性**，可以绕过 Low 标签的限制。
所以「管理员能开、普通双击不能开」是判断这个问题的关键特征。

### 3. 卸载器报 `Error writing temporary file` 怎么办？
不用卸载。直接按本文修复；若真要换版本，直接运行新安装包覆盖即可。

### 4. 输出里还看到一行 `S-1-4-...:(OI)(CI)(W,D,DC)` 要处理吗？
那是沙箱进程自己的授权项，**无害**，可以不管。它只对带该 SID 的沙箱进程有效，
普通程序用不到，不影响删除文件、更新或日常使用。

### 5. 删除工作区条目会不会丢会话历史？
不会。删除 `workspace.json` 里的条目只是解除目录与工作区的关联，
会话数据仍保存在 `.dsh\storages` 里。

### 6. 我的另一个工作区会不会也中招？
会。任何被 DSH 标过的工作区目录都可能被标成 Low。
如果发现该目录里的程序也异常，用同样的 `icacls ... /setintegritylevel M /T /C /Q` 处理即可。

---

## 参考链接

- 社区同类反馈：
  https://dsh-ai.com/discussions/10270/
- 桌面端相关修复讨论（`STARTUP_LOW_INTEGRITY`）：
  https://github.com/dsh-tauri/deepseek-harness-desktop/pull/892

---

## English Summary

**Title:** Windows DeepSeek Harness desktop app crashes on launch — DSH sandbox marks its install directory as Low Integrity

### Root cause

DSH's sandbox applies a persistent NTFS **Low Mandatory Level** label to the workspace root
when running restricted commands. If the desktop app is installed inside that workspace directory,
the whole install directory becomes Low integrity: the app cannot write its own Medium-integrity
config/cache (0x5 Access Denied) and cannot spawn Chromium child processes (crashpad not connected).

### Fix

```bat
icacls "your-install-directory" /setintegritylevel M /T /C /Q
icacls "your-install-directory" /remove "Everyone" /T /C /Q
```

Then back up and edit `%USERPROFILE%\.dsh\storages\workspace.json` to remove the workspace
entry pointing at the install directory. Restart the app.

### Prevention

Do not install DSH desktop (or any software) inside a DSH workspace directory.
Use a separate empty folder (e.g. `D:\dsh-workspace`) as your workspace.

---

## 免责声明

本文档与脚本仅供排查参考。执行前请确认路径正确，并备份 `workspace.json`。
涉及 NTFS 权限 / 完整性级别的操作请谨慎，后果自负。
