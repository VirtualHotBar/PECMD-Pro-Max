# PECMD 在 WinPE 启动中的应用

## 启动流程

在典型的 WinPE 环境中，PECMD 是内核和驱动程序初始化后第一个启动的用户模式程序。它用脚本控制的环境替代正常的 Windows Shell。

```
Windows Boot Manager (bootmgr / bootmgfw.efi)
    -> winload.exe / winload.efi (内核加载器)
        -> ntoskrnl.exe (内核)
            -> smss.exe (会话管理器)
                -> winlogon.exe
                    -> winpeshl.exe (WinPE Shell 启动器)
                        -> PECMD.EXE MAIN X:\Windows\System32\PECMD.INI
```

`winpeshl.exe` 是标准 WinPE Shell 启动器。它读取 `HKLM\SYSTEM\CurrentControlSet\Control\MiniNT` 下的 `WinPEShell` 值作为自定义 Shell 命令；如果该值不存在，则回退到 `cmd.exe`。如果 `SetupComplete.cmd` 存在，会先执行它，然后启动配置的 Shell。

### WinXShell 集成

许多现代 WinPE 构建使用 **WinXShell** 作为 PECMD 的替代或伴侣。WinXShell 提供完整的类 Explorer 桌面（任务栏、文件管理器、系统托盘），而 PECMD 处理启动脚本和底层自动化。典型集成方式：

```wcs
// 在 PECMD.INI 中：启动 WinXShell 作为桌面 Shell
EXEC* X:\Windows\System32\WinXShell.exe -winpe -wallpaper -desktop
```

WinXShell 从 `WinXShell.xml` 读取配置，可与 PECMD 共存——PECMD 管理启动脚本和驱动加载，WinXShell 提供面向用户的桌面环境。

## PECMD.INI — 标准入口点

`MAIN` 是标准 PE 入口命令。它加载配置文件**并**启动 Windows 消息循环（GUI 运行必需）。

```wcs
// PECMD.INI — 典型 WinPE 启动配置
#code=65001

// 1. 显示 Logo / 启动画面
LOGO %CurDir%\splash.jpg
TEXT 系统正在初始化，请稍候...#0xFFFFFF L20T20 $20

// 2. 初始化用户界面
INIT IU,3000

// 3. 加载 Shell
SHEL %SystemRoot%\explorer.exe

// 4. 初始化驱动程序
DEVI %CurDir%\Drivers\*.inf

// 5. 创建桌面快捷方式
LINK %Desktop%\命令提示符,%SystemRoot%\system32\cmd.exe
LINK %Desktop%\记事本,%SystemRoot%\system32\notepad.exe

// 6. 设置环境变量
ENVI $TEMP=%SystemDrive%\TEMP
ENVI $TMP=%SystemDrive%\TEMP

// 7. 注册热键
HKEY $Ctrl+Alt+#0x44, EXEC cmd.exe       // Ctrl+Alt+D -> 命令提示符（$ = 程序级热键，此 PECMD 实例的任何窗口响应）

// 8. 加载外部工具
LOAD %CurDir%\Tools\Network.ini
LOAD %CurDir%\Tools\DiskTools.ini

// 9. 执行启动程序
EXEC %SystemRoot%\system32\cmd.exe /c start /b PECMD.EXE TEAM WAIT 5000|LOAD %CurDir%\PostInit.ini

// 10. 清除 Logo
LOGO

// 11. 无限等待（保持 PE 运行，阻塞当前线程直到脚本终止）
WAIT -1
```

## 关键启动命令

### INIT — 初始化 PECMD 运行时

```wcs
INIT [选项列表],[等待时间],[USB起始盘符]
```

`INIT IU,3000` — 最常用形式。I=安装 PECMD 托盘菜单功能，U=检测 USB 移动硬盘并自动分配盘符，3000ms 超时。
`INIT IU,3000,U:` — 同上，但从 U: 开始分配 USB 盘符。

`INIT CIK` — C=将 CDROM 盘符写入环境变量，I=安装 PECMD 托盘菜单功能，K=执行 INIT 时立即安装低级键盘钩子
> ⚠ **注意**：公开发售的 WinPE 不建议带 `K` 选项（help.txt 原文），因为它安装低级键盘钩子，可能与某些软件冲突。

### SHEL — 设置 Windows Shell

```wcs
SHEL [-user|-sys] [-shel:"自动命令"] <文件名(含路径)|TEAM或EXEC开始的命令>[,密码BASE字符串][,重试次数]
```

- `-user`：强制配合 `MAIN -user` 使用
- `-sys`：直接强制为系统 Shell
- Shell 被杀时自动重新加载（自动锁定功能）
- 若使用 HOTK/HIDE 命令，SHEL **必须在其之后**（否则 HIDE 无法隐藏 PECMD 进程）

```wcs
SHEL %SystemRoot%\explorer.exe              // 使用 Explorer 作为 Shell
SHEL PECMD.EXE LOAD %CurDir%\MyShell.ini    // 使用 PECMD 脚本作为 Shell
SHEL -sys %SystemRoot%\explorer.exe         // 强制为系统 Shell
```

下一行缩进的命令在 Shell 变更时执行：
```wcs
SHEL %SystemRoot%\explorer.exe
    TEAM KILL Explorer.exe| KILL Explorer.exe
```

### LOGO — 显示/隐藏启动画面

支持 BMP/JPG/PNG/GIF 格式（需 GDI+ 支持）。标志：`-`（快速退出，无渐变）、`-top`（置顶）、`-enable`（ESC 退出）、`-wait`（等待动画结束）、`-trans:N`（透明度 0-255）。

```wcs
LOGO %CurDir%\logo.jpg           // 显示启动画面
LOGO %CurDir%\splash.jpg,0xFF00FF // 显示并设透明色
LOGO                             // 隐藏启动画面（渐隐淡出）
```

### TEXT — 显示状态文本

```wcs
TEXT 正在初始化系统...#0x00FF00 L100T200 R600B400 $18:Microsoft YaHei
```

格式：`TEXT 文本[#颜色][L左T上][R右B下][$字号[:字体名]]`

### DEVI — 安装驱动程序

```wcs
DEVI $%CurDir%\Drivers\NetCard.cab      // 从 CAB 安装（$ = 标准安装模式）
DEVI $$%CurDir%\Drivers\NetCard.inf     // 从 INF 标准安装（$$ = INF 安装模式）
DEVI %CurDir%\Drivers\*.inf             // 从 INF 文件安装（无需 $）
DEVI %CurDir%\Drivers                   // 安装目录中所有驱动（无需 $）
```

### LINK — 创建快捷方式

```wcs
LINK %Desktop%\我的工具,%CurDir%\mytool.exe
LINK %StartMenu%\Tools\分区编辑器,%CurDir%\part.exe,,%CurDir%\part.ico
```

## 常用 WinPE 模式

### 模式：等待可移动驱动器后加载工具

```wcs
_SUB WaitForUSB
    LOOP #1=1,
    {
        FDRV &盘符=*:
        FORX * &盘符,&&drv,
        {
            FORM -raw &type=%&drv%            // -raw 返回字符串常量（不区分USB/软驱）
            FIND $DRIVE_REMOVABLE=%&type%,    // 字符串比较（FORM 返回 "DRIVE_REMOVABLE" 等字符串）
            {
                IFEX %&drv%\PETOOLS\LOAD.INI, TEAM LOAD %&drv%\PETOOLS\LOAD.INI| EXIT _SUB
            }
        }
        WAIT 2000
    }
_END
```

### 模式：自动分配盘符

```wcs
SHOW -1:-1                           // 显示所有分区
DISK ,,,1,U:                         // 从 U: 开始分配 USB 驱动器
```

### 模式：设置虚拟内存（页面文件）

```wcs
PAGE C:\pagefile.sys 256 512         // C: 上最小 256MB，最大 512MB
```

### 模式：设置临时目录

```wcs
ENVI $TEMP=%SystemDrive%\TEMP
ENVI $TMP=%SystemDrive%\TEMP
```

### 模式：挂载 WIM 镜像加载外部程序

```wcs
MOUN %CurDir%\Tools.wim,%SystemDrive%\Tools,,1    // 挂载 WIM（带 TEMP）
IFEX %SystemDrive%\Tools\Setup.cmd, EXEC =!"%SystemDrive%\Tools\Setup.cmd"
```

## 最小 PECMD.INI

绝对最简 PE 启动脚本：

```wcs
#code=65001
INIT IU
SHEL %SystemRoot%\explorer.exe
WAIT -1
```

## 重要注意事项

1. PECMD.INI 末尾的 `WAIT -1` 保持脚本无限运行（否则 PE 启动后立即关闭）
2. `SHEL` 必须在 `INIT` **之后**——PECMD 运行时需先初始化，才能加载 Shell；若使用 HOTK/HIDE，SHEL 也须在其之后
3. PE 中注册表配置单元可能未完全加载。离线注册表访问用 `REGI .`（点前缀）
4. PE 中 `%SystemDrive%` 通常是 `X:`（RAM 磁盘），而非 `C:`
5. PE 环境通常缺少许多 DLL。在真实 PE 中测试脚本，或使用 `IFEX` 防护
6. `%CurDir%` 变量指向 PECMD.INI 所在目录，非常适合相对路径
7. 在 PECMD.INI 顶部使用 `LOGS * C:\pecmd.log` 调试启动问题

## PE 环境限制

PE 环境存在固有约束。理解这些是编写健壮 PECMD 脚本的关键。

### 1. 缺少运行时库
**约束**：未安装 VC++ 可再发行组件和 .NET Framework。
**缓解**：使用静态链接的 PECMD 工具（无需外部 DLL 依赖）。避免调用需要 MSVC 运行时 DLL 的程序，除非打包捆绑。

### 2. 只读介质
**约束**：`X:\` 是 RAM 磁盘（可写），但某些系统文件区域有写保护覆盖。启动介质本身（CD/DVD、USB）可能是只读的。
**缓解**：将 `%TEMP%`、`%SystemDrive%\TEMP` 或可写分区用于临时文件。切勿尝试写入 `X:\Windows\System32\` 等受保护路径。

### 3. 默认无网络
**约束**：未初始化网络适配器，未配置 DHCP。
**缓解**：在任何网络操作之前用 `PCIP` 命令显式初始化网络。无线/WiFi 用 `ADSL-wlan`；PPPoE 宽带拨号用 `ADSL`。

### 4. 临时注册表
**约束**：注册表加载到 RAM 中，重启后更改丢失。SYSTEM 和 SOFTWARE 配置单元从 WIM 加载，是只读叠加层。
**缓解**：将持久设置保存到 `HKCU`（映射到可写配置单元）或使用离线 HIVE 操作（`REGI .`）进行持久更改。使用 `%Desktop%` 或 `%TEMP%` 目录存储状态文件。

### 5. 单用户 SYSTEM 账户
**约束**：PE 以 SYSTEM 账户运行，无用户配置文件，无传统意义上的 `%USERPROFILE%` 目录，默认无用户特定的 HKCU 配置单元。
**缓解**：用 `%TEMP%` 存储临时数据。在 `INIT` 之前手动创建可写的用户配置文件目录：`PATH X:\Users\Default` 和 `ENVI $USERPROFILE=X:\Users\Default`。

### 6. 缺少驱动程序
**约束**：基础 PE 镜像可能不包含存储控制器、网络适配器和芯片组驱动。
**缓解**：使用 `DEVI` 在启动时注入所需驱动。对于持有启动介质的存储控制器，必须在启动前将驱动集成到 WIM 中（通过 DISM）。

### 7. 盘符不确定
**约束**：盘符分配不是确定性的。USB 启动盘可能是 `C:`、`D:` 或任何其他字母，而非预期的 `U:`。
**缓解**：使用 `FORX @\` 配合唯一标记文件定位正确盘符。例如：`FORX @\MyPETools.tag,&&usbDrv,1`——然后用 `%&usbDrv%` 作为基础路径。`@\` 前缀从 C: 到 Z: 搜索所有盘符。

### 8. 需要可写 USERPROFILE
**约束**：许多 Windows API 和 Shell 组件需要有效的可写 `%USERPROFILE%` 路径。没有它，Explorer 可能无法启动或行为异常。
**缓解**：在调用 `INIT` 或 `SHEL` 之前设置 `ENVI $USERPROFILE=X:\Users\Default` 并确保目录存在（`PATH X:\Users\Default`）。这是"Explorer 不启动"bug 的常见根源。

## PE 版本差异

### WinPE 3.x（Windows 7 内核）
- 基于 Windows 7 / Server 2008 R2 内核（NT 6.1）
- **特性**：支持 MBR/GPT 分区、基础 DISM、VHD 启动
- **限制**：无 DPI 缩放、USB 3.0 支持有限、WIM 挂载用 `wimgapi.dll`（用户模式，较慢，需临时空间）
- **PECMD 备注**：`INIT U` 对 USB 至关重要；许多现代存储驱动需通过 `DEVI` 注入

### WinPE 5.x（Windows 8.1 内核）
- 基于 Windows 8.1 / Server 2012 R2 内核（NT 6.3）
- **特性**：改进的 DISM（更快、更多命令）、原生 USB 3.0、更好的 SSD 支持
- **WIM 挂载**：默认仍用 `wimgapi.dll`；可选 `wimmount.sys`（内核模式，更快）
- **PECMD 备注**：开始支持 DPI 缩放（`-scale` 标志）；`PART -super -up` 可用于 GPT 分区类型更改

### WinPE 10.x（Windows 10/11 内核）
- 基于 Windows 10/11 内核（NT 10.0）
- **特性**：完整现代驱动支持、网络自动配置（Wi-Fi 配置文件）、NVMe 原生支持、exFAT 启动支持
- **WIM 挂载**：默认 `wimmount.sys`（内核模式）——比 `wimgapi.dll` 更快且使用更少 RAM
- **DPI 缩放**：通过 `-scale[:DPI]` 标志完全支持；`-scalef` 用于 XP 风格回退
- **UEFI Secure Boot**：完全支持；PECMD 脚本可在启用 Secure Boot 的环境中运行
- **PECMD 备注**：`INIT` 自动检测大多数硬件；Wi-Fi 可通过 `ADSL WLAN` 或原生 `netsh wlan` 设置

### 关键差异总结

| 特性 | WinPE 3.x | WinPE 5.x | WinPE 10.x |
|------|-----------|-----------|------------|
| WIM 挂载 | wimgapi.dll | wimgapi + 可选 wimmount.sys | wimmount.sys（默认） |
| DPI 缩放 | 无 | 基础（`-scale`） | 完整（`-scale[:DPI]`、`-scalef`） |
| USB 3.0 | 需驱动注入 | 原生 | 原生 |
| NVMe | 不支持 | 有限 | 原生 |
| UEFI Secure Boot | 不可靠 | 可用 | 完全支持 |
| 网络 | 仅手动 | 手动 + 基础自动 | 完整自动配置 |
| exFAT 启动 | 否 | 否 | 是 |
