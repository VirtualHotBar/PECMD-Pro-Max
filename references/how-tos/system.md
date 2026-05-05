# 进程/线程/系统信息/工具 — 写法示例

> **版本兼容性：** 以下示例使用 DLL 调用（`CALL $--qd`）获取系统信息。DLL 调用需要 PECMD2012 v1.88+ 完整版支持（32-bit 和 64-bit 行为一致）。
> 如果 DLL 调用返回空，说明当前版本不支持，可使用 `REGI`、`EXEC*`、`FIND --pid*@` 等内置命令替代。

## 3. 启动环境与系统信息

### 检测 BIOS 与 UEFI

```wcs
_SUB GetBootEnv
    SET$# &buf=*16 0 *4 0 *8 0
    CALL $--qd --ret:&&r ntdll.dll,NtQuerySystemInformation,#90,*&buf,#32,#0
    IFEX #%&r%<>0, TEAM ENVI-ret %~1=Unknown| EXIT _SUB
    SET?int &buf=&&type:16
    IFEX #%&type%=1, ENVI-ret %~1=BIOS
    IFEX #%&type%=2, ENVI-ret %~1=UEFI
_END
```

### 检测 Secure Boot 状态

```wcs
SET$ &SSBI=*2 0
CALL $--qd --ret:&&r ntdll.dll,NtQuerySystemInformation,#145,*&SSBI,#2,#0
SET?char &SSBI=&&enabled:0
IFEX #%&enabled%=0,MESS Secure Boot: Disabled! MESS Secure Boot: Enabled
```

### 获取 Windows 版本（RtlGetVersion）

```wcs
SET$# &verBuf=*4 0 *4 0 *4 0 *4 0 *4 0 *256 0
SET-long &verBuf=276:0
CALL $--qd --ret:&&r ntdll.dll,RtlGetVersion,*&verBuf
SET?int &verBuf=&&major:4
SET?int &verBuf=&&minor:8
SET?int &verBuf=&&build:12
SET-make &&sp=&verBuf@16;256
MESS Windows %&major%.%&minor% build %&build%
```

### 检测是否在 WinPE 中运行

```wcs
ENVI ?WinPE=&&isPE
IFEX #%&isPE%>0, MESS Running in WinPE! MESS Normal Windows
```

### 获取文件版本

```wcs
ENVI ?FVER &&ver,%SystemRoot%\System32\shell32.dll
MESS Shell32 version: %&ver%
```

---

## 6. 进程与程序执行

### 运行并捕获输出

```wcs
EXEC* &output=!cmd.exe /c dir C:\ /b
SET &count=0
FORX *NL &output,&&line, CALC &count=%&count%+1
MESS %&count% lines:%&NL%%&output%
```

### 带实时输出回调的执行

```wcs
EXEC* -cmd:::OnLine -err+ &output=!"%program%" %args%

_SUB OnLine
    READ -,0,&output,%&output%              // read captured content so far
    SED &&pct=1,\(.*\),,%&output%                    // remove parenthesized content (e.g. "50%")
    MSTR &&w1,&&w2=<1><2>%&output%
    ENVI @ProgressBar=%&w2% %&pct%          // update GUI
_END
```

### 运行子 PECMD 并等待

```wcs
EXEC =!"%MyNAME%" TEAM WAIT 1000|LOAD other.ini
ENVI ?WinPE=&&isPE
EXEC* &&ver=*PECMD                              // * prefix = capture stdout to variable
```

### 终止进程

```wcs
KILL process.exe            // by name
KILL *12345                 // by PID
KILL \                      // kill current script's windows
KILL \WindowName            // kill specific window
```

---

## 8. 线程与异步

### 通过 POSTMSG 更新 UI 的后台线程

```wcs
ENVI @MainWindow.MSG=#1: CALL OnThreadDone         // register custom message handler
THREAD* CALL LongTask
MESS This runs immediately, without waiting@多线程#OK

_SUB LongTask
    SET &result=
    FORX /S C:\*.dll,&&f,0, SET< result=%&f%
    ENVI @MainWindow.POSTMSG=#1                    // notify main thread when done
_END
```

### 线程协调共享标志

```wcs
SET &::bWork=1
THREAD* CALL Worker
WAIT 5000
SET &::bWork=0                                      // signal thread to stop

_SUB Worker
    LOOP #%&::bWork%=1,
    {
        // do work...
        WAIT 100
    }
_END
```

### 多工作线程 + 完成检查

```wcs
SET &::task1Done=0
SET &::task2Done=0
THREAD* CALL Task1
THREAD* CALL Task2

_SUB CheckDone
    FIND $%&::task1Done%%&::task2Done%=11, MESS Both tasks completed!
_END
```

---

## 9. 单实例 / 互斥体模式

```wcs
_SUB CheckSingle *
    { LOCK #pecmd                                   // atomic scope
        LOCK --exist #MyApp_UniqueName,&&exists
        REGI $HKCU\Software\MyApp\WID,&&wid
    }
    IFEX $1=%&exists%,
    {
        IFEX $%&wid%>0,
        {
            ENVI @@Visible=%&wid%:2                 // restore window
            ENVI @@POS=%&wid%:::::::1               // bring to foreground
        }
        EXIT FILE
    }
    LOCK #MyApp_UniqueName,&&ret
_END

// On window creation, save window ID:
{ LOCK #pecmd
    REGI $HKCU\Software\MyApp\WID=%&__WinID%
}

// On exit, clean up:
{ LOCK #pecmd
    REGI $HKCU\Software\MyApp\WID=
}
```

---

## 10. 热键注册

```wcs
HKEY Ctrl+Shift+#0x41, CALL OnHotkeyA            // Ctrl+Shift+A
HKEY #0x0D, MESS Enter pressed                    // Enter (window scope)
HKEY $#0x0D, MESS Enter pressed globally           // $ = system-wide
HKEY Ctrl+Shift+#0x41,--del                       // unregister
```

---

## 11. 字符串操作

### MSTR — 字段与子串提取

```wcs
MSTR &&a,&&b=<1><3>%&data%                         // fields 1 and 3 (space-delimited)
MSTR &&rest=<5->%&data%                             // fields 5 through end (`-` = to end)
MSTR &&last=<-1>%&data%                             // last field (negative index)
MSTR &&prefix=1,5,%&str%                           // chars 1-5
MSTR &&suffix=6,0,%&str%                           // from position 6 to end
MSTR -delims:. &&a,&&b,&&c,&&d=<1><2><3><4>%&ip%  // split by dot (IP: 192.168.1.1)
```

### SED — 正则替换

```wcs
SED &&r=0,pat,rep,%&source%                        // replace ALL occurrences
SED &&r=1,find,replace,%&source%                   // replace FIRST only
SED &&ext=-1,.*\.,,%&filename%                      // get extension (from end)
SED &&name=1,.*\\,,%&fullpath%                      // remove directory path
SED &&clean=0,[^0-9], ,%&str%                       // remove all non-digits
SED &&r=0:0,%&NL%,%&NL%ENVI ,%&source%             // transform newlines to commands (regex mode)
```

### LPOS / RPOS — 子串查找

```wcs
LPOS &&pos=needle,,%&haystack%                     // find first (case-insensitive)
LPOS &&pos=needle,1,%&haystack%                    // ,1, flag (case sensitivity unconfirmed)
RPOS &&pos=needle,,%&haystack%                     // find last
```

### RSTR — 取右边字符

```wcs
RSTR &&last3=3,%&str%                               // 从右边取3个字符: "12345" -> "345"
```

---

## 12. 动态变量与数组

### 间接解引用（伪数组）

```wcs
SET Arr.1.1=row1col1
SET Arr.1.2=row1col2
SET~ &&val=Arr.%&row%.%&col%                        // indirect read
```

### 延迟展开的动态变量名

```wcs
SET &hd=0
SET &D=C:
^SET &Drv[%&hd%]=%%&Drv[%&hd%]%%%%&D%             // append to Drv[0]
// ^SET defers variable expansion: %% inside ^SET -> single % at execution time
```

### 动态代码执行

```wcs
SET$ &NA=0a
READ %CurDir%\rules.ini,**,&A
SED &&A=0:0,%&NA%,%&NA%ENVI ,{*ENVI %A%            // convert lines to ENVI commands
SET< A=%&NL%}
%&A%                                                // execute generated code block
```

---

## 13. 定时器与调度模式

```wcs
TIME Timer1,1000, CALL OnTick                        // periodic timer
TIME -t:1 Timer1,1000, CALL Once                     // one-shot
ENVI @Timer1=5000;3                                   // 5s interval, run 3 times only
ENVI @Timer1=0                                        // stop
ENVI @Timer1=-del                                     // destroy

// Variable callback pattern
SET &callback=CALL OnTimerA
TIME Timer1,1000, %&callback%
// ... later ...
SET callback=CALL OnTimerB                            // switch handler on next tick
```

---

## 14. 复合条件与流程控制

### IFEX 复合条件

```wcs
IFEX [ %file% & %var%<> ], command                  // file exists AND var not empty
IFEX [ %n%<=6 & %m%<6000 ], command                // numeric AND
IFEX [ %a%<>%b% | %c%<>%d% ], command              // OR
```

### FIND 复合条件

```wcs
FIND [ $1=%&RET% & %WID%>0 ], command               // AND in FIND
```

### EXIT 变体

```wcs
EXIT LOOP       // break loop (同 EXIT BREAK)
EXIT FORX       // break FORX (同 EXIT BREAK)
EXIT CONTINUE   // continue to next iteration (LOOP/FORX)
EXIT BLOCK      // jump to current {} block tail
EXIT _SUB       // return from function
EXIT FILE       // terminate entire script
EXIT -          // same as EXIT BLOCK
```

---

## 15. 加密与哈希

```wcs
BASE "string",&&encoded                     // PECMD custom base64 (for ADSL passwords)
BASE* "string",&&encoded                    // standard base64
BASE* -u "%&encoded%",&&decoded             // standard decode
HASH C:\file.exe,&&md5,MD5
HASH $hello,&&sha1,SHA1
HASH C:\file.dat,&&crc,CRC32
CMPS -m source.wcs,dest.wcz                 // compress
CMPS -u source.wcz,dest.wcs                 // decompress
```

---

## 17. 日期/时间

```wcs
// 默认格式: "年-月-日|星期|时:分:秒"（如 "2025-1-15|3|14:30:0"）
DATE &&dateVar
MSTR &&yr,&&mo,&&dy=<1><2><3>%&dateVar%            // 提取年、月、日
MSTR &&hr,&&min,&&sec=<5><6><7>%&dateVar%          // 提取时、分、秒（<4>是星期）

// -space 标志用空格分隔（更易解析）：
DATE -space &&dateVarSp                             // "2025 1 15 3 14 30 0"
MSTR &&yr,&&mo,&&dy=<1><2><3>%&dateVarSp%

// 排序用字符串：
SET &&dateSort=%&yr%%&mo%%&dy%%&hr%%&min%%&sec%
```

---

## 18. 数学与计算

```wcs
CALC #&result=1000 * 2 + 300                        // integer math (#)
CALC &result=3.14 * 2.5 #2                          // float, 2 decimal places
CALC -base=16 #&hex=shl(0x07,16) | 0x20             // hex bitwise
CALC &sz=%&bytes%/1G#3                              // bytes to GB, 3 decimal places
CALC &&pct=100 - 100 * %&used% / %&total% ##1       // percentage, force decimals
```

---

## 19. 跨进程窗口控制

```wcs
// From second instance, restore first instance:
ENVI @@Visible=%&windowID%:2                         // SW_RESTORE
ENVI @@POS=%&windowID%:::::::1                       // bring to foreground + activate

// In Window, save its HWND:
ENVI @window.POS=?::&InitW:&InitH
```

---

## 21. 资源嵌入与提取

### 从 PECMD 资源加载嵌入脚本

```wcs
LOAD #102 arg1 arg2                                  // execute script at resource ID 102
LOAD #103 /l zh-CN                                   // with language parameter
```

### 从 PECMD 资源执行嵌入二进制文件

```wcs
EXEC* --exe:#1003 &&out=*bcdboot64.exe %sysdir% /l %lang% /s %esp% /f uefi
EXEC* --exe:#1005 =*MountESP64                       // wait for completion (=)
```

资源 ID 在构建时嵌入到 PECMD 可执行文件中。这是基于 PECMD 的工具打包依赖项的方式。


### 图标和图像资源

```wcs
IMAG Btn,L0T0W64H64,#1000                           // display image from resource #1000
TIPS* tooltip text,,,#1                              // use icon from resource #1
```
---

## 22. 系统托盘图标（完整处理器）

```wcs
// Create tray icon
TIPS* tooltip text,,,,shell32.dll#41                 // use system icon #41
TIPS* tooltip text,,,#1                              // use icon #1 from EXE resources

// Update tray icon dynamically
TIPS* ,%&newStatus%,,,%&trayIconHandle%
TIPS* ,%&newStatus%,,,-%&trayIconHandle%            // remove previous tip
TIPS*                                                // remove tray icon entirely

// WM_TRAYNOTIFY message handler for tray events
ENVI @this.MSG=_%&::WM_TRAYNOTIFY%::&&wp,&&lp, CALL OnTrayMenu %&wp% %&lp%
```

WM_TRAYNOTIFY message values:
```wcs
SET &::WM_TRAYNOTIFY=1109
SET &::WM_LBUTTONDOWN=0x0201
SET &::WM_RBUTTONDOWN=0x0204
```


### 完整托盘点击处理器（含可见性切换）

```wcs
_SUB OnTray
    IFEX $%&::WM_LBUTTONDOWN%=%2, TEAM CALL OnSwitch| EXIT _SUB
    IFEX $%&::WM_RBUTTONDOWN%=%2, CALL @--popmenu TrayMenu
_END

_SUB OnSwitch                                // toggle window visibility
    ENVI @@Visible=?%&WID%:&&view
    FIND |%&view%=0, ENVI @@Visible=%&WID%:1! ENVI @@Visible=%&WID%:0
_END
```
---

## 23. 系统电源与显示控制

### 关机 / 重启 / 注销

```wcs
SHUT                        // shutdown (SHUTDOWN)
SHUT R                      // reboot (RESTART)
SHUT L                      // logoff
SHUT S                      // standby (suspend)
SHUT H                      // hibernate
SHUT E                      // eject optical drive
SHUT O                      // eject optical drive + wait 10s
SHUT K                      // lock workstation
```

### 显示分辨率

```wcs
DISP W1920 H1080 B32 F60    // set 1920x1080, 32-bit color, 60Hz
DISP W1024 H768 B16 F60     // set 1024x768, 16-bit color
DISP                        // auto-detect best mode (no arguments)
// After DISP, restart Explorer to refresh taskbar positioning:
TEAM DISP| KILL explorer
```

### 屏幕尺寸

```wcs
SCRN &scrW,&scrH
CALC &&rightEdge=%&scrW% - 300
```

### 查询任务栏高度

```wcs
FIND --class:Shell_TrayWnd --wid*@ &tbars
FORX *NL &tbars,&&tb,
{
    MSTR* &tbtype=<7>%&tb%
    FIND $%&tbtype%=Shell_TrayWnd,
    {
        MSTR* &tbWid=<2>%&tb%
        ENVI @@POS=?%&tbWid%;;;;&TB_H
    }
}
MESS Taskbar height: %&TB_H% px
```

---

## 30. SEND / WAIT -cont — 键盘

```wcs
SEND VK_RETURN                                      // send Enter key
SEND 0x11_,0x12_,0x2E,0x12^,0x11^                  // Ctrl+Alt+Del (press order)
WAIT -cont -1000,&&key                              // wait up to 1s for key, returns VK code
```

---

## 31. 多段颜色格式

PECMD 支持 4 段颜色字符串，用于悬停感知控件：
```
文本色#背景色#悬停文本色#悬停背景色
```

```wcs
LABE -center -vcenter Btn,L0T0W100H30,Refresh,CALL OnRefresh,0xffffff#0x0066CC#0xffffff#0x0088EE
// Normal: white text on blue, Hover: white text on lighter blue
ENVI @Btn.color=0x000000#0xFFF0E0#0xFF0000#0xFFE0C0
// Updates colors at runtime
```

---

## 34. 参数验证守卫

函数入口处的提前退出守卫模式。检查参数个数、非空值和格式有效性。

```wcs
_SUB SafeFunc
    // --- Guard 1: argument count ---
    FIND $%#<3, EXIT _SUB                            // need at least 3 args

    // --- Guard 2: non-empty checks ---
    FIND $X=X%~1, EXIT _SUB                           // arg1 empty
    FIND $X=X%~2, EXIT _SUB                           // arg2 empty

    // --- Guard 3: format validation (IP) ---
    SET &ip=%~3
    MSTR -delims:. &&a,&&b,&&c,&&d=<1><2><3><4>%&ip%
    IFEX [ $%&a%<0 | $%&a%>255 | $%&b%<0 | $%&b%>255 | $%&c%<0 | $%&c%>255 | $%&d%<0 | $%&d%>255 ], EXIT _SUB
    FIND $X=X%&a%, EXIT _SUB
    FIND $X=X%&b%, EXIT _SUB

    // --- Guard 4: numeric range ---
    SET &port=%~4
    SED &&cleanPort=0,[^0-9],,%&port%
    FIND $0=%&cleanPort%, SET isNum=1
    IFEX [ $%&isNum%=0 | $%&port%<1 | $%&port%>65535 ], EXIT _SUB

    // --- Only now: do the actual work ---
    MESS All valid! Processing %&ip%:%&port%@Info#OK
_END

// Top-level script guard
_SUB MainGuard *
    FIND $%~1=, TEAM MESS Usage: script.wcs <file> <option>@Error#OK| EXIT FILE
    IFEX %~1,! TEAM MESS File not found: %~1@Error#OK| EXIT FILE

    SET &opt=%~2
    FIND $%&opt%=start, CALL OnStart %~1
    FIND $%&opt%=stop, CALL OnStop %~1
    FIND |%&opt%<>start & %&opt%<>stop, TEAM MESS Unknown option: %&opt%@Error#OK| EXIT FILE
_END
```

`FIND $X=X%&var%` 是惯用的 PECMD "为空"测试：如果 `%&var%` 为空，左侧折叠为 `X=X` 与右侧 `X=X` 匹配（`X=` 后两者皆空），因此命令执行。

---

## 35. MSTR 字符串拆分 — 所有实用变体

真实解析示例，涵盖末字段提取、第 N 字段、去空白拆分和自定义分隔符。

```wcs
// === Last segment (negative index) ===
// Typical use: extract filename from path, IP from "IPv4 ... x.x.x.x"
SET &line=   IPv4 Address. . . . . . . . : 192.168.1.100
MSTR &&ip=<-1>%&line%                             // "192.168.1.100"
MSTR &&ip=<1>%&ip%                                 // strip leading space → first field after trim

// === N-th field and range ===
SET &data=DISK 0 500107862016 GPT F6E0B 2048 976773127
MSTR &&name,&&nr,&&sz=<1><2><3>%&data%              // "DISK" "0" "500107862016"
MSTR &&rest=<4->%&data%                              // everything from field 4 onward (`-` = to end)
MSTR &&mid=<3->%&data%                               // fields 3 through end

// === Trimmed split (default whitespace strips leading blanks) ===
SET &line=     Label:    BOOT      FS: NTFS
MSTR &&label=<2>%&line%                              // "BOOT" (whitespace collapsed)

// === Custom delimiter: IP address parsing ===
SET &ip=192.168.1.100
MSTR -delims:. &&a,&&b,&&c,&&d=<1><2><3><4>%&ip%    // "192" "168" "1" "100"
MSTR -delims:. &&oct1,&&oct2,&&oct3=<1><2><3>%&ip%    // "192" "168" "1"

// === Custom delimiter: PATH parsing ===
SET &path=C:\Windows\System32\drivers\etc\hosts
MSTR -delims:\ &&root,&&sub=<1><2>%&path%             // "C:" "Windows"
MSTR -delims:\ &&file=<-1>%&path%                      // "hosts" (最后一个段)
// 去掉扩展名：先取最后段，再用 SED 去掉 .xxx
MSTR -delims:\ &&basename=<-1>%&path%                   // "hosts"
SED &&nameNoExt=0,\.[^.]*$,,%&basename%                 // "hosts"（无扩展名时不变）

// === Custom delimiter: Multi-char (PART output parsing) ===
PART list disk 0,&&info
MSTR &&sz,&&bus,&&mbr,&&sign=<2><9><10><9>%&info%     // field 2=size, 9=bus, 10=MBR/GPT, 9=sign
// Note: field indices can repeat — <9> appears twice (bus = field 9, sign = field 9 — this is a quirk)

// === Extract extension via SED + MSTR ===
SET &filename=backup.2025.tar.gz
SED &&ext=-1,.*\.,,%&filename%                        // get last dot → everything after
MESS Extension: %&ext%                                // "gz"

// === Split lines into key=value pairs ===
SET &cfg=NAME=MyApp%&NL%VER=2.0%&NL%PORT=8080
FORX *NL &cfg,&&line,
{
    MSTR -delims:= &&key,&&val=<1><2>%&line%
    SET %&key%=%&val%
}
```

---

## 39. 结构体数组遍历（API 返回数据）

遍历 Win32 API 调用返回的结构体数组。分配缓冲区、枚举索引、计算字段偏移、读取带类型的值、检查终止条件。

```wcs
_SUB EnumDiskDrives
    // GUID_DEVINTERFACE_DISK = 53f56307-b6bf-11d0-94f2-00a0c91efb8b
    SET &GUID_HEX=53 f5 63 07 b6 bf 11 d0 94 f2 00 a0 c9 1e fb 8b
    SET$# &guid=*16 0
    CODE *,%&GUID_HEX%,*UNI,&guid

    SET &::DIGCF_PRESENT=0x02
    SET &::DIGCF_DEVICEINTERFACE=0x10

    CALC &&flags=%&::DIGCF_PRESENT% | %&::DIGCF_DEVICEINTERFACE%
    CALL $--qd --ret:&&hSetup Setupapi.dll,SetupDiGetClassDevsW,*&guid,#0,#0,#%&flags%
    IFEX $%&hSetup%=-1, EXIT _SUB

    // --- Struct sizes ---
    // SP_DEVICE_INTERFACE_DATA: cbSize(4) + InterfaceClassGuid(16) + Flags(4) + Reserved(8)
    //   x86: 32 bytes, x64: 40 bytes (due to alignment)
    IFEX $%&::bX64%<3, SET &elemSz=32! SET &elemSz=40
    // SP_DEVICE_INTERFACE_DETAIL_DATA_W: cbSize(4) + DevicePath(variable, ~260 WCHARs)
    //   Header is cbSize(4), then DevicePath starts at byte 4
    SET &detailHeader=4

    SET &i=0
    SET$# &devData=*%&elemSz% 0                         // one element buffer
    SET-long &devData=%&elemSz%:0                         // write cbSize

    LOOP #1=1,
    {
        SET$# &devData=*%&elemSz% 0
        SET-long &devData=%&elemSz%:0
        CALL $--qd --bool --ret:&&ok Setupapi.dll,SetupDiEnumDeviceInterfaces,#%&hSetup%,#0,*&guid,#%&i%,*&devData
        IFEX $%&ok%<>1, EXIT LOOP

        // --- Read field at offset ---
        SET?int &devData=&&flags:%&detailHeader% + 16   // Flags at offset 20 (cbSize=4 + guid=16)
        MESS Device #%&i%: Flags=0x%&flags%

        // --- Get detail data (contains DevicePath) ---
        // First call: get required size
        SET$# &detailBuf=*%&detailHeader% 0
        SET-long &detailBuf=%&detailHeader%:0
        SET$# &reqSize=*4 0
        CALL $--qd --ret:&&r Setupapi.dll,SetupDiGetDeviceInterfaceDetailW,#%&hSetup%,*&devData,#0,#0,*&reqSize,#0
        SET?int &reqSize=&&sz:0
        IFEX $%&sz%<1, EXIT LOOP

        // Allocate full buffer and call again
        SET$# &detailBuf=*%&sz% 0
        SET-long &detailBuf=%&detailHeader%:0
        CALL $--qd --bool Setupapi.dll,SetupDiGetDeviceInterfaceDetailW,#%&hSetup%,*&devData,*&detailBuf,#%&sz%,#0,#0

        // DevicePath is at offset 4 (right after cbSize)
        SET?ptr &detailBuf=&&pathPtr:4
        SET-make &&devicePath=&pathPtr;0                    // null-terminated copy
        MESS DevicePath: %&devicePath%
        // Trim null terminator if present:
        SED &&devicePath=0,\0.*,,%&devicePath%

        CALC &i=%&i%+1
    }

    CALL $--qd --bool Setupapi.dll,SetupDiDestroyDeviceInfoList,#%&hSetup%
    MESS Total devices found: %&i%@Enum Complete#OK
_END

// --- SIMPLE EXAMPLE: DISK_GEOMETRY via DeviceIoControl ---
_SUB ReadDiskGeometry
    SET &dsk=\\.\PhysicalDrive%~1
    SET &::IOCTL_DISK_GET_DRIVE_GEOMETRY=0x70000

    // Open disk
    CALC &&access=0x80000000 | 0x40000000    // GENERIC_READ | GENERIC_WRITE
    CALC &&share=1 | 2                        // FILE_SHARE_READ | FILE_SHARE_WRITE
    CALL $--qd --ret:&&h Kernel32.dll,CreateFileW,$%&dsk%,#%&access%,#%&share%,#0,#3,#0,#0
    IFEX $%&h%=-1, EXIT _SUB

    // DISK_GEOMETRY struct layout:
    //   Cylinders(8) + MediaType(4) + TracksPerCylinder(4) + SectorsPerTrack(4) + BytesPerSector(4)
    //   = 24 bytes base; varies by SDK version
    SET$# &geom=*48 0                              // generous allocation
    SET$# &retSz=*8 0
    CALL $--qd --ret:&&r Kernel32.dll,DeviceIoControl,#%&h%,#%&::IOCTL_DISK_GET_DRIVE_GEOMETRY%,#0,#0,*&geom,#48,*&retSz,#0

    // --- Read fields at known offsets ---
    SET?longlong &geom=&&cylinders:0
    SET?int &geom=&&mediaType:8
    SET?int &geom=&&tracksPerCyl:12
    SET?int &geom=&&sectorsPerTrack:16
    SET?int &geom=&&bytesPerSector:20

    CALC &&totalBytes=%&cylinders% * %&tracksPerCyl% * %&sectorsPerTrack% * %&bytesPerSector%
    CALC &&totalGB=%&totalBytes%/1G#2

    MESS Cylinders: %&cylinders%%&NL%MediaType: %&mediaType%%&NL%Tracks/Cyl: %&tracksPerCyl%%&NL%Sectors/Track: %&sectorsPerTrack%%&NL%Bytes/Sector: %&bytesPerSector%%&NL%Total: %&totalGB% GB@Disk Geometry#OK

    CALL $--qd --bool Kernel32.dll,CloseHandle,#%&h%
_END
```

关键点：
- `SET-ptr` + `SET-make` 在指定偏移处读取指针值并复制其指向的数据。
- `SET-long` 在缓冲区指定偏移处写入 32 位整数（用于 `cbSize`）。
- `SET?int` 读取 32 位有符号整数；`SET?longlong` 读取 64 位。
- 结构体字节偏移根据 MSDN 布局手动计算。
- `LOOP #1=1` 配合 `EXIT LOOP` 是计数未知时竞技场式迭代的标准模式。

---

## 附加模式

### 模式 57：Win32 API 两次调用缓冲区模式

许多 Win32 API 需要调用两次：一次获取所需缓冲区大小，然后分配，再调用一次。这是标准的可复用模板：

```wcs
// Generic two-call buffer pattern
// Step 1: Call with NULL/0 to get required size
CALL $--qd --ret:&retSize DLL.dll,FunctionName,*#0,#0,...
// Step 2: Allocate buffer with returned size
SET$# &buffer=*%&retSize% 0
// Step 3: Call again with actual buffer
CALL $--qd --ret:&retSize DLL.dll,FunctionName,*&buffer,#%&retSize%,...
```

适用函数：GetWindowsDirectoryW、GetSystemDirectoryW、GetTempPathW、GetComputerNameW、GetUserNameW、GetIfTable、QueryDosDeviceW、GetAdaptersInfo、GetModuleFileNameW。

示例 — 获取计算机名：
```wcs
_SUB GetComputerName
    SET$# &nSize=*4 0
    CALL $--qd --ret:&ret Kernel32.dll,GetComputerNameW,*#0,*&nSize
    ENVI?int &nSize=&bufSize
    IFEX #%&bufSize%=0, EXIT _SUB
    SET$# &buf=*%&bufSize% 0
    CALL $--qd --ret:&ret Kernel32.dll,GetComputerNameW,*&buf,*&nSize
    ENVI-ret %~1=%&buf%
_END
```

### 模式 58：通过 SED 正则进行 GUID 字节交换（CLSIDFromString 回退方案）

当 `ole32.dll!CLSIDFromString` 不可用时（最小 PE 中常见），使用正则字节交换从字符串构造 GUID 二进制：

```wcs
_SUB MakeGuid
    // Try CLSIDFromString first
    CALL $--qd --ret:&ret ole32.dll,CLSIDFromString,${%~2},*%~1
    IFEX #%&ret%>=0, EXIT _SUB
    // Fallback: regex byte-swap for little-endian GUID layout
    // Input: {XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX}
    SED &&hex=0,
      {\a\a}{\a\a}{\a\a}{\a\a}-{\a\a}{\a\a}-{\a\a}{\a\a}-{\a\a}{\a\a}-{\a\a}{\a\a}{\a\a}{\a\a}{\a\a}{\a\a},
      0x\4 0x\3 0x\2 0x\1 0x\6 0x\5 0x\8 0x\7 0x\9 0x\10 0x\11 0x\12 0x\13 0x\14 0x\15 0x\16,
      %~2
    CODE *,%&hex%,*HEX,%~1
_END
```

### 模式 59：Win32 回调的 WndProc 绑定

将 PECMD `_SUB` 函数注册为 Win32 回调（例如用于 EnumResourceNames、EnumWindows）：

```wcs
// Bind function to execution stack and get its address
SET^ CallbackFunc,&&callbackAddr
// Pass address to Win32 API as callback
CALL $--qd --ret:&ret Kernel32.dll,EnumResourceNamesW,
    #%&hModule%,#3,#%&callbackAddr%,#0
// Unbind when done
SET^ CallbackFunc=0

_SUB CallbackFunc
    // %1=hModule, %2=lpType, %3=lpName, %4=lParam
    // Return 1 to continue enumeration, 0 to stop
    EXIT _SUB 1
_END
```

### 模式 60：WM_COMMAND + EN_CHANGE 编辑框监控

通过 WM_COMMAND 通知监控编辑框文本变更：

```wcs
SET &WM_COMMAND=0x0111
SET &EN_CHANGE=0x0300
// Get the edit control's HWND
ENVI @Edit1.ID=?;&Edit1Hwnd
// Compute expected wParam: (EN_CHANGE << 16) | controlID
CALC -base=16 #&ExpectedWP=%&EN_CHANGE% * 0x10000 + %&Edit1Hwnd%
// Register for WM_COMMAND on the window
ENVI @this.MSG=_%&WM_COMMAND%::&wp,&lp, CALL OnEditChange

_SUB OnEditChange
    IFEX $%&wp%=%&ExpectedWP%, {
        // Edit1 text changed - read new value
        ENVI @Edit1.VAL=?;&newText
        // ... handle change
    }
_END
```

### 模式 61：WM_MOUSEHOVER/LEAVE 悬停提示

在鼠标悬停控件时显示提示：

```wcs
SET &WM_MOUSEHOVER=0x02A1
SET &WM_MOUSELEAVE=0x02A3
ENVI @Label1.MSG=_%&WM_MOUSEHOVER%: TIPS Title,"Hover text\nLine 2",3000,1
ENVI @Label1.MSG=_%&WM_MOUSELEAVE%: TIPS *
```

### 模式 62：WM_SIZE 响应式布局与保存位置

完整的 DPI 感知窗口调整大小处理：

```wcs
_SUB MainWindow,L200T100W600H400,My App,-trap,-size
    TABL Table1,L10T10W580H300,...
    ITEM Btn1,L10T320W80H28,Refresh
    ITEM Btn2,L100T320W80H28,Close
    // Save initial window and control sizes
    ENVI @this.POS=?::&initW:&initH
    ENVI @Table1.POS=?::&tblW:&tblH
    ENVI @Btn2.POS=?&btn2L:&btn2T
    // Register resize handler
    SET &WM_SIZE=0x0005
    ENVI @this.MSG=_%&WM_SIZE%: CALL OnResize
_END

_SUB OnResize
    // Extract new width/height from wParam
    CALC #&newW= %2 & 0xFFFF          // LOWORD
    CALC #&newH= %2 >> 16              // HIWORD
    // Calculate delta from initial size
    CALC #&dw= %&newW% - %&initW%
    CALC #&dh= %&newH% - %&initH%
    // Resize table proportionally
    CALC #&tw= %&tblW% + %&dw%
    CALC #&th= %&tblH% + %&dh%
    ENVI @Table1.POS=::%&tw%:%&th%
    // Reposition buttons (anchor to bottom-right)
    CALC #&b2L= %&btn2L% + %&dw%
    CALC #&b2T= %&btn2T% + %&dh%
    ENVI @Btn2.POS=%&b2L%:%&b2T%::
_END
```

### 模式 63：LoadLibraryExW 仅加载资源

加载 DLL/EXE 纯用于资源提取，不执行代码：

```wcs
SET &LOAD_LIBRARY_AS_DATAFILE=0x00000002
SET &LOAD_LIBRARY_AS_IMAGE_RESOURCE=0x00000020
CALC #&flags=%&LOAD_LIBRARY_AS_DATAFILE% | %&LOAD_LIBRARY_AS_IMAGE_RESOURCE%
CALL $--qd --ret:&hMod Kernel32.dll,LoadLibraryExW,$%&filePath%,#0,#%&flags%
// Now use EnumResourceNamesW, LoadResourceW etc. on &hMod
// Don't forget to free: CALL $--qd kernel32.dll,FreeLibrary,#%&hMod%
```

---

### 46. FVAR Secure Boot（直接 EFI 变量）

```wcs
// Method 1: Direct EFI global variable read (simplest)
ENVI ?&ret=FVAR,SecureBoot;{8be4df61-93ca-11d2-aa0d-00e098032b8c}
// Returns 0=off, 1=on

// Method 2: NtQuerySystemInformation #145 (see Boot Environment pattern above)
```


### 49. 带超时的显示模式预设

```wcs
ENVI &&curDisp=
SUBM * &&curDisp
DISP W%&w%H%&h%B%&b%F%&f% T10    // apply with 10s countdown
FIND $%&YesNo%=NO,
{   // User cancelled or timeout → revert
    KILL explorer.exe
}

// Or multi-try with timer fallback:
TIME &TM,2000,CALL OnTwoSeconds   // 2-second timer
_SUB OnTwoSeconds
    FIND $0=%&YesNo%, DISP     // if still 0, auto-revert
_END
```

---

### 50. QueryDosDeviceW — 所有 MS-DOS 设备

```wcs
SET$# &&buf=*0x100000 0
CALL $--qd --ret:&bret kernel32.dll,QueryDosDeviceW,#0,*&buf,#0x80000
// Returns null-delimited, double-null terminated device list
// lpos* * for binary null pattern search
LPOS* * &&pos=0x00 0x00 0x00 0x00,1,&&buf
// Extract and decode: GETF -bin → MSTR * → SED -ex → CODE ***unicode
CODE ***unicode,**.buf,*uni,&&result
```


### 51. RtlGetNtVersionNumbers（基于指针）

```wcs
SET$# &&Major=*4 0
SET$# &&Minor=*4 0
SET$# &&Build=*4 0
CALL $--qd --ret:&bret ntdll.dll,RtlGetNtVersionNumbers,*&Major,*&Minor,*&Build
SET?int &&Major=&&Major:0
SET?int &&Minor=&&Minor:0
SET?int &&Build=&&Build:0
CALC &BuildNumber=%&Build% & 0xFFFF   // mask high 16 bits
```


## 73. 线程变量异步冲突与修复

### 问题：共享变量竞态条件

```wcs
// BUG: I is shared in persistent stack, child thread may see stale value
SET &I=1
LOOP %I%<10,
{
    SET &J=%I%
    THREAD* TEAM WAIT 100| MESS I=%&I% J=%J%       // I may be wrong!
    CALC I=%I% + 1
}
```

### 修复 1：创建线程前复制到局部变量

```wcs
{
    SET &I2=%I%                                       // copy to local
    THREAD* TEAM WAIT 100| MESS I=%&I2% J=%J%        // I2 is safe
    CALC I=%I% + 1
}
```

### 修复 2：使用 THREAD$ 进行预解释

```wcs
{
    THREAD*$ TEAM WAIT 100| MESS I=%&I% J=%J%        // %&I% resolved BEFORE thread starts
    CALC I=%I% + 1
}
```

### THREAD 与 THREAD* 变量共享规则

| 上下文 | THREAD（无 *） | THREAD*（带 *） |
|---------|---------------|-------------------|
| 普通函数 `{}` 块 | 复制（隔离） | 复制（隔离） |
| 窗口 `_SUB` 或 `_SUB F,*` | 复制（隔离） | **共享**（直接链接） |
| 窗口 `_SUB` 内的 `{}` | 复制（隔离） | 复制（隔离——`{}` 降级为临时） |

---

## 76. 随机字符串生成

```wcs
SET &CSet=0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz
STRL * &&LCSET=CSet
SET &V=
SET &n=10                                            // desired length
LOOP #%&n%>0,
{
    CALC &n=%&n% - 1
    ^CALC &&i=%RANDOM% % %&LCSET% + 1
    MSTR * &&vi=%&i%,1,CSet
    SET< &V=%&vi%
}
// &V now contains 10 random alphanumeric characters
```

---

## 77. 系统字体字符集常量

通过 `ENVI @ctrl.Font=size:name:style:charset` 设置字体时非常有用。

| Constant | Value | Description |
|----------|-------|-------------|
| ANSI_CHARSET | 0 | Western |
| DEFAULT_CHARSET | 1 | System default |
| GB2312_CHARSET | 134 | Simplified Chinese |
| CHINESEBIG5_CHARSET | 136 | Traditional Chinese |
| SHIFTJIS_CHARSET | 128 | Japanese |
| HANGEUL_CHARSET | 129 | Korean |
| OEM_CHARSET | 255 | OEM codepage |

---

## 非代码手册快速参考（PECMD补充说明.doc 摘录）

`PECMD补充说明.doc` 是高级模式的权威来源：

### THREAD* 栈链规则
- 在窗口 _SUB（持久栈）中：THREAD* 直接共享 PE 变量
- 在临时函数/块（`{}`）中：THREAD* 复制 PE 变量（隔离）
- `THREAD$`：启动前预解释一次（使用字面值，避免异步冲突）
- `-link`：维护父子窗口连接；等待子线程结束

### PE 变量析构器
```wcs
SET-def ~CloseHandleX~h=0    // 定义 h 并注册析构器 CloseHandleX
// 作用域退出时：CloseHandleX %&h% → PE 变量 h 被释放
// 析构器按定义逆序执行
```

### 函数析构器（`_SUB Func,*,析构命令`）
```wcs
_SUB F1,*,IFEX #[ %&h%>0 ], CALL $kernel32.dll,CloseHandle,#%&h%
    // ... 函数体，可提前 EXIT
_END  // 退出时自动执行析构命令
```

### #& 控件命名（共享 PE 变量）
```wcs
LIST #&L7,L410T55W46H23,1|2|3|4,,1,    // 控件名是 #&L7，变量是 %&L7%
// 从父窗口/其他页面访问：ENVI @Page1:#&L7.VAL=...
```

### ENVI^ Alias 别名系统
```wcs
ENVI^ Alias aliasName=[cmd prefix]
ENVI^ Alias * aliasName=cmd             // * = 启用 prefix+空格 语法
```

### ENVI @@POSTMSG/SENDMSG 完整语法
```wcs
ENVI @@SENDMSG=[:retVar;]windowID;messageID[;wParam[;lParam]]
// wParam,lParam：@PE变量（缓冲区）、$字符串（仅 SENDMSG）、数字
// 带 # 前缀的消息 = PECMD 自定义消息 1-N
// _ = 后半响应模式（系统处理后响应）
```

### PUT/GET 二进制资源导出
```wcs
// #.N = 原始（未压缩）资源数据
PUTF -dd -bs=10M out.dat,0,"%MyName%""#.101|SCRIPT"
// #N = 已解压资源
PUTF -dd -bs=10M out.dat,0,"%MyName%""#2|INDATA"
// 资源类型 ID：CURSOR=1 BITMAP=2 ICON=3 MENU=4 DIALOG=5、STRING=6、
//   FONTDIR=7 ACCELERATOR=9 RCDATA=10 GROUP_ICON=14 VERSION=16 MANIFEST=24
```

### FIND/IFEX 简写块语法（>=79N-59D）
```wcs
FIND $1=1,FIND body! ELSE body          // 单行 TRUE + ;ELSE
FIND $1=1,
{   MESS TRUE
}! MESS FALSE                           // 多行 TRUE，单行 ELSE 在 }! 后
FIND $1=1, { MESS TRUE                  // TRUE 块首行内联
}! { MESS FALSE }
```

### SED 正则语法（来自 PECMD2012正则表达式.doc）
```
.   = 任意字符       [abc] = 字符类    [^abc] = 否定类
?   = 0-1 次         + = 1+ 次         * = 0+ 次
??  = 非贪婪 ?       +? = 非贪婪 +     *? = 非贪婪 *
()  = 分组           {} = 命名组（通过 \1-\9 引用）
^   = 行首锚点       $ = 行尾锚点      | = 或
\\a = [a-zA-Z0-9]   \\d = [0-9]        \\h = [0-9a-fA-F]
\\w = [a-zA-Z]+     \\z = [0-9]+        \\n = 换行
替换：\\0=完整匹配 \\1-\\9=分组引用 \\u=大写 \\l=小写
```

### PECMD 变量 → CMD 变量（3 种方法）
```
// 方法1（最佳）：WRIT 到 stdout
WRIT -,$+0,a 111        // CMD FOR /F 捕获输出

// 方法2：临时文件
WRIT %tmpf%,$+0,set a=%val%   // 然后 CALL .\tmpf.CMD

// 方法3：注册表
REGI HKCU\PECMD_U\var=%val%   // CMD 通过 reg query 读取
```

---

