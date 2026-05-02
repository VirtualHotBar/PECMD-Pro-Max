# PECMD Code Pattern Cookbook

Each pattern below is distilled from real PECMD code in the wild. Patterns marked with `[CHINESE]` use Chinese variable names, the de facto standard in the PECMD community.

---

## 1. Disk Enumeration & Information

### List all physical disks with details

```wcs
PART list disk,&&全部磁盘
FORX * %&全部磁盘%,&&磁盘,
{
    PART list disk %&磁盘%,&&磁盘信息
    MSTR &&sz,&&bus,&&mbr,&&sign=<2><9><10><9>%&磁盘信息%
    MESS Disk#%&磁盘%: Size=%&sz% bytes, Bus=%&bus%, Type=%&mbr%
}
```

### List all partitions on a disk (GPT + MBR compatible)

```wcs
SET &dsk=0    // 0 = first disk
// Step 1: get partition numbers (plain list)
PART list part %&dsk%,&&全部分区
// Step 2: iterate and get detailed info per partition
FORX * %&全部分区%,&&pt,
{
    FIND $%&pt%=0, EXIT -                         // skip partition 0 (SD card issue)
    PART -hextp -phy# list part %&dsk%#%&pt%,&&分区信息
    MSTR &&tp,&&start,&&len,&&attr,&&drv=<2><4><5><7><9>%&分区信息%
    // For GPT: MSTR has different field layout
    // MSTR &&guid,&&attr,&&start,&&len,&&nr=<2><3><4><5><8>%&分区信息%
    CALC &&lenGB=%&len%/1G#1
    IFEX $%&lenGB%G<1G, TEAM CALC &&lenMB=%&len%/1M#1| SET lenDisplay=%&lenMB%M! SET lenDisplay=%&lenGB%G
    MESS Part#%&pt%: Type=%&tp%, Size=%&lenDisplay%, Drv=%&drv%
}
```

### Map drive letters to disk numbers

```wcs
FDRV &Drvs=*:
FORX * %&Drvs%,&&D,
{
    PART list drv %&D%,&&V
    MSTR &&hd=<9>%&V%
    ^SET &Drv[%&hd%]=%%&Drv[%&hd%]%%%%&D%      // append with delayed expansion
}
// &Drv[0], &Drv[1], etc. now hold concatenated drive letters per disk
```

### Get volume label and filesystem (with auto-mount fallback)

```wcs
_SUB GetVol
    TEAM ENVI %1=| ENVI %2=| ENVI &&v=| ENVI &&b=0| ENVI &&VL=%~3| ENVI &&dsk=%~4| ENVI &&pt=%~5
    SET &r1=
    SET &r2=
    FIND $X=X%&VL%,! FDRV *vol r1,r2=%&VL%      // vol exists: direct query
    FIND $X=X%&VL%,                               // vol doesn't exist: auto-assign letter
    { LOCK #pecmd
        SET b=1
        FDRV *idle *rsort &&VL=*:
        LSTR VL=2,%&VL%
        SHOW & %&dsk%#%&pt%,%&VL%                // assign temp letter
        FDRV *vol r1,r2=%&VL%                     // query
        SHOW & ,%&VL%                              // release letter
    }
    ENVI-ret %1=%&r1%
    ENVI-ret %2=%&r2%
_END
```

---

## 2. Device Enumeration via Win32 API

### Enumerate physical disks via SetupAPI

```wcs
// Canonical method: CLSIDFromString produces binary GUID
SET &GUID_STR={53f56307-b6bf-11d0-94f2-00a0c91efb8b}
SET$# &guid=*16 0
CALL $--qd --ret:&&r ole32.dll,CLSIDFromString,${%&GUID_STR%},*&guid
// Alternative: construct GUID from hex bytes
SET &GUID_HEX=53 f5 63 07 b6 bf 11 d0 94 f2 00 a0 c9 1e fb 8b
CODE *,%&GUID_HEX%,*HEX,&guid
CALL $--qd --ret:&&hSetup Setupapi.dll,SetupDiGetClassDevsW,*&guid,#0,#0,#0x12
IFEX $%&hSetup%<>-1,
{
    SET &i=0
    LOOP #1=1,
    {
        IFEX $%&::bX64%<3, SET &dataSz=28! SET &dataSz=32
        SET$# &devData=*%&dataSz% 0
        SET-long &devData=%&dataSz%:0
        CALL $--qd --bool --ret:&&ok Setupapi.dll,SetupDiEnumDeviceInterfaces,#%&hSetup%,#0,*&guid,#%&i%,*&devData
        IFEX $%&ok%<>1, EXIT LOOP
        // ... get detail data, open device, send IOCTL ...
        CALC &i=%&i% + 1
    }
    CALL $--qd --bool Setupapi.dll,SetupDiDestroyDeviceInfoList,#%&hSetup%
}
```

### Get disk performance counters (IOCTL_DISK_PERFORMANCE)

```wcs
SET &disk=\\.\PhysicalDrive0
CALC &&access=0x80000000 | 0x40000000
CALC &&share=0x00000001 | 0x00000002
CALL $--qd --ret:&&h Kernel32.dll,CreateFileW,$%&disk%,#%&access%,#%&share%,#0,#3,#0,#0
IFEX $%&h%<>-1,
{
    SET &ioctl=0x70020
    SET$# &buf=*0x58 0
    SET$# &retSz=*8 0
    CALL $--qd --ret:&&r Kernel32.dll,DeviceIoControl,#%&h%,#%&ioctl%,#0,#0,*&buf,#0x58,*&retSz,#0
    SET?longlong &buf=&&bytesRead:0
    SET?longlong &buf=&&bytesWritten:8
    CALL $--qd --bool Kernel32.dll,CloseHandle,#%&h%
}
```

---

## 3. Boot Environment & System Info

### Detect BIOS vs UEFI

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

### Detect Secure Boot state

```wcs
SET$ &SSBI=*2 0
CALL $--qd --ret:&&r ntdll.dll,NtQuerySystemInformation,#145,*&SSBI,#2,#0
SET?char &SSBI=&&enabled:0
IFEX #%&enabled%=0,MESS Secure Boot: Disabled! MESS Secure Boot: Enabled
```

### Get Windows version (RtlGetVersion)

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

### Check if running in WinPE

```wcs
ENVI ?WinPE=&&isPE
IFEX #%&isPE%>0, MESS Running in WinPE! MESS Normal Windows
```

### Get file version

```wcs
ENVI ?FVER &&ver,%SystemRoot%\System32\shell32.dll
MESS Shell32 version: %&ver%
```

---

## 4. File & Config Operations

### Read entire file (text / binary)

```wcs
READ C:\config.ini,**,&content      // ** = DOS CRLF -> native, *r = raw, * = LF only
GETF# C:\data.bin,0#*,&raw          // binary read
```

### Parse INI-style config into variables [CHINESE]

```wcs
READ %CurDir%\config.ini,*,&cfg
FORX *NL &cfg,&&line,
{
    SED &&key=1,=.*,,%&line%
    SED &&val=1,.*=,,%&line%
    FIND $=%&key%,! SET %&key%=%&val%         // skip empty key lines
    // Alternative idiom (canonical in 代码大全 source):
    // FIND *<>var,...  = "var is NOT empty"  (execute if var has content)
    // FIND *=var,...   = "var IS empty"      (execute if var is blank)
}
```

### Write to file

```wcs
WRIT C:\output.txt,$0,First line          // $ = ANSI, 0 = overwrite
WRIT C:\output.txt,$+0,Second line        // + = append
PUTF -dd -len=0 C:\file.bin,0,zero        // create/truncate
PUTF C:\file.bin,%offset%,#%&data%        // write at offset
```

### Get file timestamp and size

```wcs
SIZE &&sz=C:\file.txt
MESS Size: %&sz% bytes
```

---

## 5. Registry Operations

### Read/write (all types)

```wcs
// Read
REGI $HKLM\SOFTWARE\App\Version,&str        // REG_SZ
REGI #HKLM\SOFTWARE\App\Count,&num           // REG_DWORD
REGI @HKLM\SOFTWARE\App\Binary,&data         // REG_BINARY
REGI ~HKLM\SOFTWARE\App\Path,&expand         // REG_EXPAND_SZ
REGI +HKLM\SOFTWARE\App\Big,&qword           // REG_QWORD
REGI .HKLM\SOFTWARE\App\OfflineKey,&val      // offline hive (PE mounted Windows)

// Write
REGI $HKLM\SOFTWARE\App\Version=1.2.3
REGI #HKLM\SOFTWARE\App\Count=#0x100         // hex DWORD

// Delete
REGI $HKLM\SOFTWARE\App\OldKey=              // empty = delete

// Enumerate
REGI HKCU\Software\,&&keys                   // subkeys (NL-delimited)
REGI HKCU\Software\MyApp,&vals               // values in key
```

---

## 6. Process & Program Execution

### Run and capture output

```wcs
EXEC* &output=!cmd.exe /c dir C:\ /b
SET &count=0
FORX *NL &output,&&line, CALC &count=%&count%+1
MESS %&count% lines:%&NL%%&output%
```

### Run with real-time output callback [CHINESE]

```wcs
EXEC* -cmd:::OnLine -err+ &output=!"%program%" %args%

_SUB OnLine
    READ -,0,&output,%&output%              // read captured content so far
    SED &&pct=1,\(,,%&output%
    SED &&pct=1,\).*,,%&pct%
    MSTR &&w1,&&w2=<1><2>%&output%
    ENVI @ProgressBar=%&w2% %&pct%          // update GUI
_END
```

### Run sub-PECMD and wait

```wcs
EXEC =!"%MyNAME%" TEAM WAIT 1000|LOAD other.ini
ENVI ?WinPE=&&isPE
EXEC* &&ver=*PECMD                              // * prefix = internal PECMD command
```

### Kill a process

```wcs
KILL process.exe            // by name
KILL *12345                 // by PID
KILL \                      // kill current script's windows
KILL \WindowName            // kill specific window
```

---

## 7. GUI Patterns

### Complete window template [CHINESE]

```wcs
#code=936T950
ENVI^ EnviMode=1
ENVI^ ForceLocal=1
SET$ &NL=0d 0a
SET$ &TAB=09

CALL @主窗口

_SUB 主窗口,W400H300,我的工具,,#1,,
    ENVI @this.Font=12:Microsoft YaHei
    ENVI @this.bkcolor=0xF0F0F0
    ENVI @this.MSG=0x0010:CALL OnClose           // WM_CLOSE (window-level, no _)
    LABE -vcenter -trans 标签1,L20T20W360H30,欢迎使用本工具,,0x000000,14
    EDIT 编辑1,L20T60W360H120,,CALL OnEdit,,12
    ITEM 确定,L150T200W100H35,确定,CALL OnOK
    ITEM 取消,L260T200W100H35,取消,KILL \
_END

_SUB OnOK
    MESS 你点击了确定@提示#OK
_END

_SUB OnClose
    KILL \
_END
```

### DPI-aware window with manual layout

```wcs
_SUB DPIWindow,W600H400,DPI Demo,,,,, -scalef -scale
    TEAM SET &dpi=| REGI #HKCU\Control Panel\Desktop\WindowMetrics\AppliedDPI,&dpi|
    FIND $%&dpi%=, SET dpi=96
    // Check if system already auto-scaled (if actual = requested, skip manual scaling)
    ENVI @this.POS=?::&actualW:&actualH
    IFEX [ %&actualW%=600 & %&actualH%=400 ], SET dpi=96
    SET &DPI=%&dpi%/96
    // Save initial positions for resize handling
    ENVI @this.POS=?::&InitW:&InitH
    CALC #&x=20*%&DPI%
    ITEM Btn,L%&x%T120W100H30,DPI Button,CALL OnClick
    ENVI @this.MSG=0x0005::&&wp,&&lp,CALL OnResize %&wp% %&lp%
_END

_SUB OnResize
    IFEX $%1=1, EXIT _SUB                        // SIZE_MINIMIZED, skip
    CALC #&newW= %2 & 0xFFFF                     // LOWORD of lParam
    CALC #&newH= %2 / 0x10000                    // HIWORD of lParam
    IFEX [ %&newW%<600 | %&newH%<400 ],         // enforce minimum
    {
        ENVI @DPIWindow.POS=::%&InitW%:%&InitH%
    }
_END
```

### TABL data operations (comprehensive)

```wcs
// Title with formatting flags
ENVI &&Title=100:名称%&TAB%=60:PID%&TAB%+80:内存%&TAB%*0:隐藏列%&TAB%*200:路径
//   default=left, =  =right-align, +  =center, *0: =hidden column

TABL Table1,L10T10W500H300,%&Title%,,0x10040

// Bulk-set from variable (typical after FIND --pid*@)
FIND --pid*@ &&processList,
ENVI @Table1.Val=1*;%&processList%

// Read individual cells
ENVI @Table1.Sel=?;&sel
ENVI @Table1.Val=?%&sel%.1;&name
ENVI @Table1.Val=?%&sel%.2;&pid

// Row count
ENVI @Table1.Val=?*;&count

// Clear all
ENVI @Table1.Val=-*

// Select/deselect
ENVI @Table1.Sel=3
ENVI @Table1.Sel=3;0

// Set full row (tab-separated columns)
ENVI @Table1.Val=%&row%;%&col1%%&TAB%%&col2%%&TAB%%&col3%

// Sort rows by reading and swapping values
ENVI @Table1.Val=?%&a%;&rowA
ENVI @Table1.Val=?%&b%;&rowB
ENVI @Table1.Val=%&a%;%&rowB%
ENVI @Table1.Val=%&b%;%&rowA%
```

### LIST dropdown operations

```wcs
LIST -h List1,L10T10W200H200,,CALL OnSelect,,0x100
ENVI @List1.VAL=                        // clear all
ENVI @List1.ADD=选项1                    // add item
ENVI @List1.ADD=选项2
ENVI @List1.ADDSEL=默认选项              // add and select
ENVI @List1.DEL=选项1                    // remove item
ENVI @List1.isel=2                       // select by 1-based index
ENVI @List1.QUERY=;&allItems             // get all items (NL-delimited)
```

### Popup context menu on right-click

```wcs
ENVI @Table1.MSG=_%&::WM_RBUTTONDOWN%: CALL @--popmenu MyMenu

_SUB MyMenu
    MENU 打开位置,打开文件位置,TEAM ENVI @Table1.Sel=?&s| CALL FileDir
    MENU -
    MENU 复制,复制到剪贴板,CALL Clipboard
    MENU -
    MENU 结束进程,结束进程,CALL KillProc
_END
```

### SWIN sub-window (tab pages)

```wcs
SWIN :Page1,L10T40W380H200
SWIN :Page2,L10T40W380H200,,0x100       // 0x100 = initially hidden

_SUB Page1
    ENVI @this.bkcolor=0xFFFFFF
    LABE ,L10T10W200H20,This is page 1
_END
// Switch: ENVI @Page1.Visible=0 | ENVI @Page2.Visible=1
```

### Dynamic label creation (command in variable)

```wcs
ENVI &&cmdString=LABE -vcenter -trans Lbl%&i%,L%&x%T%&y%W%&w%H%&h%,%&text%,CALL OnClick %&i%,0x000000,14
%&cmdString%                                      // execute the dynamically-built command
```

### Checkbox with linked control

```wcs
CHEK Check1,L20T50W200H20,Auto refresh,CALL OnCheck,1

_SUB OnCheck
    IFEX $%Check1.Check%=1, ENVI @Timer1=1000! ENVI @Timer1=0
_END
```

---

## 8. Threading & Async

### Background thread with UI update via POSTMSG

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

### Shared flags for thread coordination

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

### Multiple workers + completion check

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

## 9. Single Instance / Mutex Pattern

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

## 10. Hotkey Registration

```wcs
HKEY Ctrl+Shift+#0x41, CALL OnHotkeyA            // Ctrl+Shift+A
HKEY #0x0D, MESS Enter pressed                    // Enter (window scope)
HKEY $#0x0D, MESS Enter pressed globally           // $ = system-wide
HKEY Ctrl+Shift+#0x41,--del                       // unregister
```

---

## 11. String Manipulation

### MSTR — field and substring extraction

```wcs
MSTR &&a,&&b=<1><3>%&data%                         // fields 1 and 3 (space-delimited)
MSTR &&rest=<~5>%&data%                             // fields 5 through end
MSTR &&last=<-1>%&data%                             // last field (negative index)
MSTR &&prefix=1,5,%&str%                           // chars 1-5
MSTR &&suffix=6,0,%&str%                           // from position 6 to end
MSTR -delims:. &&a,&&b,&&c,&&d=<1*>%&ip%          // split by colon (IP: 192.168.1.1)
```

### SED — regex substitution

```wcs
SED &&r=0,pat,rep,%&source%                        // replace ALL occurrences
SED &&r=1,find,replace,%&source%                   // replace FIRST only
SED &&ext=-1,.*\.,,%&filename%                      // get extension (from end)
SED &&name=1,.*\\,,%&fullpath%                      // remove directory path
SED &&clean=0,[^0-9], ,%&str%                       // remove all non-digits
SED &&r=0:0,%&NL%,%&NL%ENVI ,%&source%             // transform newlines to commands (regex mode)
```

### LPOS / RPOS — substring search

```wcs
LPOS &&pos=needle,,%&haystack%                     // find first (case-insensitive)
LPOS &&pos=needle,1,%&haystack%                    // case-sensitive search
RPOS &&pos=needle,,%&haystack%                     // find last
```

### RSTR — zero-padding

```wcs
RSTR &&padded=3,000%num%                            // pad to 3 digits: 8 -> 008
```

---

## 12. Dynamic Variables & Arrays

### Indirect dereference (pseudo-arrays)

```wcs
SET Arr.1.1=row1col1
SET Arr.1.2=row1col2
SET~ &&val=Arr.%&row%.%&col%                        // indirect read
```

### Dynamic variable names with delayed expansion

```wcs
SET &hd=0
SET &D=C:
^SET &Drv[%&hd%]=%%&Drv[%&hd%]%%%%&D%             // append to Drv[0]
// ^SET defers variable expansion: %% inside ^SET -> single % at execution time
```

### Dynamic code execution

```wcs
SET$ &NA=0a
READ %CurDir%\rules.ini,**,&A
SED &&A=0:0,%&NA%,%&NA%ENVI ,{*ENVI %A%            // convert lines to ENVI commands
SET< A=%&NL%}
%&A%                                                // execute generated code block
```

---

## 13. Timer & Scheduler Patterns

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

## 14. Compound Conditions & Flow Control

### IFEX compound conditions

```wcs
IFEX [ %file% & %var%<> ], command                  // file exists AND var not empty
IFEX [ %n%<=6 & %m%<6000 ], command                // numeric AND
IFEX [| %a%<>%b% | %c%<>%d% ], command             // OR
```

### FIND compound conditions

```wcs
FIND [ $1=%&RET% & %WID%>0 ], command               // AND in FIND
```

### EXIT variants

```wcs
EXIT LOOP       // break loop
EXIT FORX       // break FORX
EXIT BLOCK      // exit {} code block
EXIT _SUB       // return from function
EXIT FILE       // terminate entire script
EXIT -          // continue (skip to next iteration)
```

---

## 15. Encryption & Hashing

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

## 16. Network Operations

### Get network adapter IP

```wcs
EXEC* &&ipcfg=*ipconfig                         // * prefix = internal PECMD command
FORX *NL &ipcfg,&&line,
{
    SED &&found=1,IPv4,,%&line%
    IFEX $1=%&found%,
    {
        MSTR &&ip=<-1>%&line%                   // last field = IP address
        MSTR &&ip=<2>%&ip%                       // strip leading space
        MESS IP: %&ip%
    }
}
```

### WiFi scan & connect

```wcs
ADSL-wlan ,,list,&&wifiInfo                     // scan nearby networks
ADSL-wlan ,,scan,&&detail                        // detailed scan
BASE "SSID",&&encSSID
BASE "password",&&encPSK
ADSL-wlan %&encSSID%,%&encPSK%                  // connect
```

### Ping check

```wcs
EXEC* &result=!ping -n 1 192.168.1.1
FIND TTL=,%&result%,MESS Host reachable             // substring search (no $ = contains)
```

---

## 17. Date/Time

```wcs
DATE &&dateVar                                     // get current date
DATE &&dateVar -now                                // get date+time
TIME &&timeVar                                     // get current time
DTIM &&ts,&&dateVar,&&timeVar                       // combine to timestamp (seconds)
CALC &&ts=%&ts% + 3600                              // add 1 hour
DTIM &&newDate,%&ts%                                // convert back to date string
```

---

## 18. Math & Calculation

```wcs
CALC #&result=1000 * 2 + 300                        // integer math (#)
CALC &result=3.14 * 2.5 #2                          // float, 2 decimal places
CALC -base=16 #&hex=shl(0x07,16) | 0x20             // hex bitwise
CALC &sz=%&bytes%/1G#3                              // bytes to GB, 3 decimal places
CALC &&pct=100 - 100 * %&used% / %&total% ##1       // percentage, force decimals
```

---

## 19. Cross-Process Window Control

```wcs
// From second instance, restore first instance:
ENVI @@Visible=%&windowID%:2                         // SW_RESTORE
ENVI @@POS=%&windowID%:::::::1                       // bring to foreground + activate

// In Window, save its HWND:
ENVI @window.POS=?::&InitW:&InitH
```

---

## 20. PART Operations (full toolkit patterns)

```wcs
// Change partition type
PART -super -up -xup %&dsk%#%&pt% %&newType%

// Toggle active flag
FIND $%&ac%=1, SET a=-a! SET a=a
PART -super -up %&dsk%#%&pt% %&a%

// Toggle hide (bit 4)
CALC -base=16 #&ntp=%&tp% @ 0x10                   // XOR with 0x10 to toggle

// Delete partition (with safety)
MESS 确定要删除该分区吗？@删除分区#YN*8000$N
FIND $%&YESNO%<>YES, EXIT _SUB
SHOW *- %&dsk%#%&pt%,                               // unload first (3 times for safety)
SHOW *- %&dsk%#%&pt%,
PART -super -up del %&dsk%#%&pt%

// Swap physical partition numbers
PART -up -hup -swap:%&v1% %&dsk%#%&v2%
```

---

## 21. Resource Embedding (Internal Scripts & EXEs)

### Load embedded scripts from PECMD resources

```wcs
LOAD #102 arg1 arg2                                  // execute script at resource ID 102
LOAD #103 /l zh-CN                                   // with language parameter
```

### Execute embedded binaries from PECMD resources

```wcs
EXEC* -exe:#1003 &&out=*bcdboot64.exe %sysdir% /l %lang% /s %esp% /f uefi
EXEC* -exe:#1005 =*MountESP64                        // wait for completion (=)
```

Resource IDs are embedded in the PECMD executable at build time. This is how PECMD-based tools bundle dependencies.

---

## 22. System Tray Icon (TIPS*)

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

---

## 23. System Power & Display Control

### Shutdown / reboot / logoff

```wcs
SHUT                        // shutdown (SHUTDOWN)
SHUT R                      // reboot (RESTART)
SHUT L                      // logoff
SHUT E                      // standby
SHUT H                      // hibernate
```

### Display resolution

```wcs
DISP W1920 H1080 B32 F60    // set 1920x1080, 32-bit color, 60Hz
DISP W1024 H768 B16 F60     // set 1024x768, 16-bit color
DISP                        // auto-detect best mode (no arguments)
// After DISP, restart Explorer to refresh taskbar positioning:
TEAM DISP| KILL explorer
```

### Screen dimensions

```wcs
SCRN &scrW,&scrH
CALC &&rightEdge=%&scrW% - 300
```

### Query taskbar height

```wcs
FIND --class:Shell_TrayWnd --wid*@ &tbars
FORX *NL &tbars,&&tb,
{
    MSTR &tbtype=<7>&&tb
    FIND $%&tbtype%=Shell_TrayWnd,
    {
        MSTR &tbWid=<2>&&tb
        ENVI @@POS=?%&tbWid%::::&TB_H
    }
}
MESS Taskbar height: %&TB_H% px
```

---

## 24. Directory Search Across All Drives (FORX @)

```wcs
FORX @\Windows,&&winDir,1,                          // search ALL drives for \Windows
{
    MSTR &&drive=1,2,%&winDir%                       // extract drive letter
    IFEX %&winDir%\System32\config\SOFTWARE,
    {
        MESS Found Windows on %&drive%: %&winDir%
    }
}
// @\DirName iterates ALL root drives looking for DirName
// The 3rd param 1 returns the FIRST match only (vs 0 = all)
```

---

## 25. Offline Registry Manipulation (offreg.dll)

Critical for PE system deployment. Uses the full offreg.dll API for create/read/write/enumerate on offline Windows hives.

```wcs
// Open offline hive
SET &hHive=
CALL $--qd --ret:&&ret offreg.dll,OROpenHive,$%&HiveFile%,*&hHive
IFEX $%&ret%<>0, TEAM MESS Failed to open hive@错误#OK| EXIT

// Open or create key
SET &hKey=
CALL $--qd --ret:&&ret offreg.dll,OROpenKey,#%&hHive%,$%&SubKey%,*&hKey
IFEX #%&ret%=0,                                       // key already exists
{
    // Read value
    SET$# &Data=*8192 0
    SET$# &pdwType=&PtrSz% 0
    SET$# &pcbData=&PtrSz% 0
    SET-long &pcbData=8192:0
    CALL $--qd --ret:&&ret offreg.dll,ORGetValue,#%&hKey%,#0,$%&Value%,*&pdwType,#0,*&pcbData
    SET?int &Data=&&ValueLength:0
    SET-make &&Value=&Data@4;%&ValueLength%
}! IFEX #%&ret%=234,                                    // key doesn't exist
{
    CALL $--qd --ret:&&ret offreg.dll,ORCreateKey,#%&hHive%,$%&SubKey%,#0,$,#0,*&hKey,#0
}

// Set value
CALL $--qd --ret:&&ret offreg.dll,ORSetValue,#%&hKey%,$%&Value%,#1,*&Data,#%&DataSize%

    // Enumerate subkeys
    SET$# &keyCount=*4 0
    SET$# &maxSubKeyLen=*4 0
    SET$# &valCount=*4 0
    CALL $--qd --ret:&&ret offreg.dll,ORQueryInfoKey,#%&hKey%,#0,#0,*&keyCount,*&maxSubKeyLen,#0,*&valCount,#0,#0,#0,#0
    SET?int keyCount=&&nKeys:0

// Save and close — use actual NT version numbers for best compatibility
// Use $0 for generic (current OS version implied)
CALL $--qd --ret:&&ret offreg.dll,ORSaveHive,#%&hHive%,$%&HiveFile%,$0,$0
CALL $--qd offreg.dll,ORCloseKey,#%&hKey%
CALL $--qd offreg.dll,ORCloseHive,#%&hHive%
```

---

## 26. Window Style Flags & Window Hiding

```wcs
// Common window creation flags
_SUB MyWin,W400H300,Title,,,,, -trap -nocap -ntab -nfocus
//   -trap: close button doesn't exit (window survives close)
//   -nocap: no title bar (frameless window)
//   -ntab: hidden from taskbar
//   -nfocus: no keyboard focus on creation
//   -nosysmenu: no system menu (no icon, no min/max/close)
//   -top: always on top (TOPMOST)
//   -maxb: enable maximize button
//   -minb: enable minimize button

// Cross-process window hide/show
ENVI @@Visible=%&WID%:0                           // SW_HIDE (hide to tray)
ENVI @@Visible=%&WID%:1                           // SW_SHOW (show)
ENVI @@Visible=%&WID%:2                           // SW_RESTORE (restore from minimize)
ENVI @@Visible=%&WID%:*4                          // SW_MINIMIZE
ENVI @@Visible=?%&WID%:&&state                    // query visibility state
```

## 27. Resource Extraction & Embedded Tools

```wcs
// EXEC* -exe:#resourceID extracts and runs a binary embedded in PECMD resources
EXEC* -exe:#1000 &&out=*mv.exe Z: /S               // run mv.exe from resource #1000
EXEC* -exe:#1001 &&out=*mv64.exe Z: /S              // run mv64.exe from resource #1001

// Resource scripts (LOAD #ID)
LOAD #102                                            // run script at resource 102

// Icon resources
IMAG Btn,L0T0W64H64,#1000                           // display image from resource #1000
TIPS* tooltip text,,,#1                              // use icon from resource #1
```

## 28. BROW — File/Directory Browse Dialog

```wcs
BROW &saveFile,&%Desktop%\output.iso,Save ISO file,iso           // save dialog
BROW &openFile,,Select a file,INI|*.INI|All Files|*.*|           // open dialog with filter
BROW &folder,*C:\,Select a folder                                // folder browser (* prefix)
// Additional flags: 0x200=multi-select, 0x10=edit box
```

## 29. SUBJ — Mount/Unmount Drive Letters

```wcs
SUBJ -X:                                            // remove drive letter X:
SUBJ G:,\Device\HarddiskVolume3                     // mount volume as G:
```

## 30. NET & Network Card Operations

```wcs
// Full network adapter query
PCIP ?* IP,MASK,GW,DNS,%NIC%?NAME,MAC,LINK,DHCP,bDHCP,STATUS,MEDIA,DESC,TYPE

// Set static IP
PCIP 192.168.1.100,255.255.255.0,192.168.1.1,192.168.1.1

// DHCP
PCIP DHCP
```

## 31. SEND / WAIT -cont — Keyboard

```wcs
SEND {ENTER}                                        // send Enter key
SEND 0x11_,0x12_,0x2E,0x12^,0x11^                  // Ctrl+Alt+Del (press order)
WAIT -cont -1000,&&key                              // wait up to 1s for key, returns VK code
```

## 32. Multi-Part Color Format

PECMD supports a 4-part color string for hover-aware controls:
```
textColor#backgroundColor#hoverTextColor#hoverBackgroundColor
```

```wcs
LABE -center -vcenter Btn,L0T0W100H30,Refresh,CALL OnRefresh,0xffffff#0x0066CC#0xffffff#0x0088EE
// Normal: white text on blue, Hover: white text on lighter blue
ENVI @Btn.color=0x000000#0xFFF0E0#0xFF0000#0xFFE0C0
// Updates colors at runtime
```

## 33. Dynamic Control Creation (Command String in Variable)

```wcs
// Create controls programmatically from a command string in a variable
ENVI &&cmd=LABE -vcenter -trans Lbl%&i%,L%x%T%y%W%w%H%h%,%&text%,,0x000000,14
%&cmd%                                              // execute the command to create the control

// Delete controls dynamically
ENVI @Lbl%A.*del=                                   // delete label A
ENVI @Edit%B.*del=                                  // delete edit field B
// This is essential for dynamic GUIs that rebuild control sets
```

## 34. WM_TRAYNOTIFY Pattern (Full Tray Icon Handler)

```wcs
SET &::WM_TRAYNOTIFY=1109
ENVI @this.MSG=_%&::WM_TRAYNOTIFY%::&&wp,&&lp, CALL OnTray %&wp% %&lp%

_SUB OnTray
    IFEX $%&::WM_LBUTTONDOWN%=%2, TEAM CALL OnSwitch| EXIT _SUB
    IFEX $%&::WM_RBUTTONDOWN%=%2, CALL @--popmenu TrayMenu
_END

_SUB OnSwitch                                // toggle window visibility
    ENVI @@Visible=?%&WID%:&&view
    FIND |%&view%=0, ENVI @@Visible=%&WID%:1! ENVI @@Visible=%&WID%:0
_END
```

## 35. SWIN Nested Windows (Tab Pages)

Complete property-page pattern: define each sub-window as a `_SUB`, embed them with `SWIN` in the parent, use `TABS.SEL` to switch visible page on tab click.

```wcs
#code=936T950
ENVI^ EnviMode=1
ENVI^ ForceLocal=1
SET$ &NL=0d 0a

CALL @主窗口

_SUB 主窗口,W500H380,Tabbed Settings,,#1,,
    ENVI @this.Font=12:Microsoft YaHei
    TEAM ENVI &&nTab=0| ENVI &&TabSel=1

    // --- TABS control (page buttons) ---
    TABS TabMain,L10T10W480H40,CALL OnTabSel

    // --- SWIN containers for each page ---
    SWIN :PageGeneral,L10T55W480H290
    SWIN :PageNetwork,L10T55W480H290,,0x100        // hidden initially
    SWIN :PageAdvanced,L10T55W480H290,,0x100       // hidden initially

    // --- Populate tabs ---
    LOOP #%&nTab%<3,
    {
        CALC &nTab=%&nTab%+1
        IFEX $%&nTab%=1, ENVI @TabMain.ADD=General
        IFEX $%&nTab%=2, ENVI @TabMain.ADD=Network
        IFEX $%&nTab%=3, ENVI @TabMain.ADD=Advanced
    }
    ENVI @TabMain.SEL=%&TabSel%                     // select first tab by default

    ITEM BtnOK,L200T352W100H30,OK,CALL OnOK
_END

_SUB OnTabSel
    ENVI @TabMain.SEL=?&&sel
    ENVI @PageGeneral.Visible=0
    ENVI @PageNetwork.Visible=0
    ENVI @PageAdvanced.Visible=0
    IFEX $%&sel%=1, ENVI @PageGeneral.Visible=1
    IFEX $%&sel%=2, ENVI @PageNetwork.Visible=1
    IFEX $%&sel%=3, ENVI @PageAdvanced.Visible=1
_END

_SUB PageGeneral
    ENVI @this.bkcolor=0xFFFFFF
    LABE ,L10T10W200H20,General Settings,,0x000000,10
    CHEK ChkAutoRun,L10T40W200H20,Auto start with Windows,,1
    EDIT EdtName,L10T70W460H24,MyApp,,12
_END

_SUB PageNetwork
    ENVI @this.bkcolor=0xFFFFFF
    LABE ,L10T10W200H20,Network Configuration,,0x000000,10
    EDIT EdtIP,L10T40W200H24,192.168.1.100,,12
    EDIT EdtPort,L10T74W200H24,8080,,12
_END

_SUB PageAdvanced
    ENVI @this.bkcolor=0xFFFFFF
    LABE ,L10T10W200H20,Advanced Options,,0x000000,10
    CHEK ChkDebug,L10T40W200H20,Enable debug logging,,1
    CHEK ChkEncrypt,L10T70W200H20,Encrypt traffic,,0
_END

_SUB OnOK
    ENVI @PageGeneral.Visible=?
    ENVI @PageNetwork.Visible=?
    ENVI @PageAdvanced.Visible=?
    IFEX $%ChkAutoRun.Check%=1, MESS AutoRun: ON
    MESS IP=%&EdtIP%:%&EdtPort%@Info#OK
    KILL \
_END
```

Key points:
- `SWIN` containers must be placed BEFORE their `_SUB` definitions.
- Use `0x100` flag to hide a page initially.
- `TABS.SEL=<n>` selects a tab; `TABS.SEL=?` queries the current selection.
- Toggle visibility with `ENVI @PageName.Visible=0` / `=1`.

---

## 36. Dynamic Row Creation & Batch Deletion

Build controls at runtime from variable-expanded command strings. Batch-delete groups in loops.

```wcs
_SUB CreateRow
    SET &row=%~1
    CALC #&y=50 + %&row% * 35
    ENVI &&cmd=LABE -vcenter LblName%&row%,L20T%&y%W150H30,Item %&row%,,0x000000,12
    %&cmd%
    ENVI &&cmd=EDIT EdtVal%&row%,L180T%&y%W100H24,value%&row%,,12
    %&cmd%
    ENVI &&cmd=ITEM BtnDel%&row%,L290T%&y%W60H28,Del,CALL OnDelRow %&row%
    %&cmd%
_END

_SUB OnDelRow
    SET &row=%~1
    ENVI @LblName%&row%.*del=                    // destroy label
    ENVI @EdtVal%&row%.*del=                     // destroy edit
    ENVI @BtnDel%&row%.*del=                     // destroy button
_END

_SUB ClearAllRows
    SET &i=0
    LOOP #%&i%<10,
    {
        CALC &i=%&i%+1
        ENVI @LblName%&i%.*del=
        ENVI @EdtVal%&i%.*del=
        ENVI @BtnDel%&i%.*del=
    }
_END

// --- Usage in a window context ---
// Create 5 rows:
SET &n=0
LOOP #%&n%<5,
{
    CALC &n=%&n%+1
    CALL CreateRow %&n%
}
```

Key points:
- `ENVI @CtrlName.*del=` destroys a control and frees its resources.
- Command strings in variables (`%&cmd%`) are the only way to use computed control names.
- Batch deletion loops must be careful not to skip indices when controls are pre-indexed.

---

## 37. Parameter Validation Guard

Early-exit guard pattern at function entry. Checks argument count, non-empty values, and format validity.

```wcs
_SUB SafeFunc
    // --- Guard 1: argument count ---
    FIND $%#<3, EXIT _SUB                            // need at least 3 args

    // --- Guard 2: non-empty checks ---
    FIND $X=X%~1, EXIT _SUB                           // arg1 empty
    FIND $X=X%~2, EXIT _SUB                           // arg2 empty

    // --- Guard 3: format validation (IP) ---
    SET &ip=%~3
    MSTR -delims:. &&a,&&b,&&c,&&d=<1*>&ip
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

`FIND $X=X%&var%` is the idiomatic PECMD "is empty" test: if `%&var%` is empty, the left side collapses to `X=X` which matches the right side `X=X` (both empty after `X=`), so the command executes.

---

## 38. MSTR String Splitting — All Practical Variants

Real parsing examples covering last-field extraction, N-th field, trimmed split, and custom delimiters.

```wcs
// === Last segment (negative index) ===
// Typical use: extract filename from path, IP from "IPv4 ... x.x.x.x"
SET &line=   IPv4 Address. . . . . . . . : 192.168.1.100
MSTR &&ip=<-1>%&line%                             // "192.168.1.100"
MSTR &&ip=<2>%&ip%                                 // strip leading space → first field after trim

// === N-th field and range ===
SET &data=DISK 0 500107862016 GPT F6E0B 2048 976773127
MSTR &&name,&&nr,&&sz=<1><2><3>%&data%              // "DISK" "0" "500107862016"
MSTR &&rest=<~4>%&data%                              // everything from field 4 onward
MSTR &&mid=<3~5>%&data%                              // fields 3 through 5

// === Trimmed split (default whitespace strips leading blanks) ===
SET &line=     Label:    BOOT      FS: NTFS
MSTR &&label=<2>%&line%                              // "BOOT" (whitespace collapsed)

// === Custom delimiter: IP address parsing ===
SET &ip=192.168.1.100
MSTR -delims:. &&a,&&b,&&c,&&d=<1><2><3><4>%&ip%    // "192" "168" "1" "100"
MSTR -delims:. &&net=<1~3>%&ip%                       // "192.168.1"

// === Custom delimiter: PATH parsing ===
SET &path=C:\Windows\System32\drivers\etc\hosts
MSTR -delims:\ &&root,&&sub=<1><2>%&path%             // "C:" "Windows"
MSTR -delims:\ &&file=<-1>%&path%                      // "hosts"
MSTR -delims:\ &&ext=<-1.>%&path%                      // "hosts" (same; last segment before dot use <->)

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

## 39. TABL Scrollbar Control via LVM Messages

Use `SENDMSG` to send list-view messages for scroll control. Messages apply to the underlying SysListView32 control.

```wcs
#code=936T950
SET$ &NL=0d 0a
SET$ &TAB=09

// LVM constants
SET &::LVM_FIRST=0x1000
SET &::LVM_SCROLL=%&::LVM_FIRST% + 20               // 0x1014
SET &::LVM_ENSUREVISIBLE=%&::LVM_FIRST% + 19        // 0x1013
SET &::LVM_GETITEMCOUNT=%&::LVM_FIRST% + 4          // 0x1004
SET &::LVM_GETCOUNTPERPAGE=%&::LVM_FIRST% + 40      // 0x1028
SET &::LVM_GETTOPINDEX=%&::LVM_FIRST% + 39          // 0x1027

CALL @主窗口

_SUB 主窗口,W600H400,Table Scroll Demo,,#1,,
    ENVI @this.Font=12:Microsoft YaHei
    ENVI &&Title=100:Name%&TAB%=60:PID%&TAB%+80:Memory

    TABL Table1,L10T10W580H300,%&Title%,,0x10040

    // Fill table with sample data
    SET &i=0
    LOOP #%&i%<100,
    {
        CALC &i=%&i%+1
        ENVI @Table1.Val=%&i%;Process%&i%%&TAB%%&i%00%&TAB%%&i%M
    }

    ITEM BtnBottom,L10T320W140H35,Scroll to Bottom,CALL ScrollBot
    ITEM BtnSel,L160T320W140H35,Scroll to Selection,CALL ScrollSel
    ITEM BtnPageDn,L310T320W140H35,Page Down,CALL ScrollPgDn
    ITEM BtnEnsure,L460T320W130H35,Ensure Row 77,CALL EnsureRow
_END

_SUB ScrollBot
    // Get item count, scroll by that many lines down
    ENVI @Table1.POSTMSG=%&::LVM_GETITEMCOUNT%
    WAIT 50
    ENVI @Table1.SENDMSG=%&::LVM_SCROLL%,0,50                     // 0=dx, 50=dy (lines)
    // Alternative: scroll to very large line offset
    // ENVI @Table1.SENDMSG=%&::LVM_SCROLL%,0,999999
_END

_SUB ScrollSel
    ENVI @Table1.Sel=?&&sel
    IFEX $%&sel%<1, EXIT _SUB
    ENVI @Table1.SENDMSG=%&::LVM_ENSUREVISIBLE%,%&sel% - 1,0       // 0-based index, partialOK=0
_END

_SUB ScrollPgDn
    // Get visible rows per page, scroll by that amount
    // Simplistic: send a fixed vertical scroll
    ENVI @Table1.SENDMSG=%&::LVM_SCROLL%,0,20                       // scroll down 20 lines
_END

_SUB EnsureRow
    ENVI @Table1.SENDMSG=%&::LVM_ENSUREVISIBLE%,76,0               // row 77 (0-based)
_END
```

Note: `SENDMSG` sends to the window that last received focus / the foreground window. In a GUI context, you may need to focus the table first or use `ENVI @Table1.SENDMSG` (PECMD 2012 extension). For cross-process or precise control, use `CALL $ user32.dll,SendMessageW`.

---

## 40. TABL In-Row Sorting (Bubble Sort)

Read all rows into memory, bubble-sort by a target column, rewrite the table.

```wcs
_SUB SortTableByCol
    SET &col=%~1                                    // 1-based column to sort by
    ENVI @Table1.Val=?*;&&rows                       // total rows

    // --- Read all rows into array variables ---
    SET &i=0
    LOOP #%&i%<%&rows%,
    {
        CALC &i=%&i%+1
        ENVI @Table1.Val=?%&i%;&&Row[%&i%]
    }

    // --- Numeric sort: bubble sort ---
    SET &i=0
    LOOP #%&i%<%&rows%,
    {
        CALC &i=%&i%+1
        SET &j=%&i%
        LOOP #%&j%<%&rows%,
        {
            CALC &j=%&j%+1
            MSTR &&vA=<%&col%>%%&Row[%&i%]%%
            MSTR &&vB=<%&col%>%%&Row[%&j%]%%
            // Numeric comparison (use # for number):
            SED &&nA=0,[^0-9],,%&vA%
            SED &&nB=0,[^0-9],,%&vB%
            FIND $=&nA=, SET nA=0
            FIND $=&nB=, SET nB=0
            // Swap if a > b (ascending)
            IFEX $%&nA%>%&nB%,
            {
                SET &tmp=%%&Row[%&i%]%%
                ENVI &Row[%&i%]=%%&Row[%&j%]%%
                ENVI &Row[%&j%]=%&tmp%
            }
        }
    }

    // --- String sort alternative (by column text) ---
    // LOOP ...
    //     MSTR &&sA=<%&col%>%%&Row[%&i%]%%
    //     MSTR &&sB=<%&col%>%%&Row[%&j%]%%
    //     FIND $%&sA%>%&sB%,                             // lexicographic "greater" for ascending
    //     {
    //         SET &tmp=%%&Row[%&i%]%%
    //         SET &Row[%&i%]=%%&Row[%&j%]%%
    //         SET &Row[%&j%]=%&tmp%
    //     }

    // --- Rewrite table ---
    SET &i=0
    LOOP #%&i%<%&rows%,
    {
        CALC &i=%&i%+1
        ENVI @Table1.Val=%&i%;%%&Row[%&i%]%%
    }
_END

// Usage:
// CALL SortTableByCol 3          // sort by 3rd column
```

Key points:
- Double-percent `%%&Row[%&i%]%%` dereference: first `%%` evaluates to `%`, then `%&Row[3]%` reads the variable.
- `FIND $str1>str2` does lexicographic "greater" comparison (compare two strings, true if first > second). Use `|` prefix instead for numeric compare.
- Remove non-digits with `SED` for numeric extraction before numeric compare.

---

## 41. Custom Title Bar Window (Frameless + Manual Caption)

Borderless window with fake title bar built from LABE controls. Handles minimize, close, hover color effects, and window dragging via `WM_NCHITTEST`.

```wcs
#code=936T950
ENVI^ EnviMode=1
ENVI^ ForceLocal=1

SET &::WM_LBUTTONDOWN=0x0201
SET &::WM_LBUTTONUP=0x0202
SET &::WM_MOUSEMOVE=0x0200
SET &::WM_NCHITTEST=0x0084
SET &::HTCAPTION=2

CALL @CustomWin

_SUB CustomWin,W500H350,My Custom Tool,-trap -nocap,#1,,
    ENVI @this.Font=12:Microsoft YaHei
    ENVI @this.bkcolor=0xF0F0F0

    // --- Fake title bar background ---
    LABE -center TitleBar,L0T0W500H32,,,0xFFFFFF#0x2D2D30

    // --- Fake icon ---
    LABE -center -vcenter LblIcon,L8T4W24H24,&#x1f4bb;,,0xFFFFFF#0x2D2D30#0xFFFFFF#0x3D3D40
    ENVI @LblIcon.MSG=%&::WM_LBUTTONDOWN%: CALL @--popmenu SysMenu

    // --- Title text ---
    LABE -center -vcenter LblTitle,L36T4W360H24,My Custom Tool,,0xFFFFFF#0x2D2D30

    // --- Minimize button ---
    LABE -center -vcenter BtnMin,L412T4W36H24,&#x2500;,CALL OnMin,0xCCCCCC#0x2D2D30#0xFFFFFF#0x3D3D40

    // --- Close button ---
    LABE -center -vcenter BtnClose,L452T4W40H24,&#x2715;,CALL OnClose,0xCCCCCC#0x2D2D30#0xFFFFFF#0xE81123

    // --- Drag support: whole title bar reports as HTCAPTION ---
    ENVI @TitleBar.MSG=%&::WM_NCHITTEST%: ENVI @TitleBar.POSTMSG=%&::HTCAPTION%
    ENVI @LblTitle.MSG=%&::WM_NCHITTEST%: ENVI @LblTitle.POSTMSG=%&::HTCAPTION%

    // --- Main content area ---
    LABE -vcenter LblContent,L20T50W460H280,Content goes here,,0x000000,12

    // --- Fake status bar ---
    LABE -vcenter LblStatus,L0T330W500H20,Ready,,0xAAAAAA#0x2D2D30
_END

_SUB SysMenu
    MENU 关于,About,CALL OnAbout
    MENU -
    MENU 退出,Exit,KILL \
_END

_SUB OnMin
    ENVI @@POS=%__WinID%:::::::6                    // SW_MINIMIZE via window ID
_END

_SUB OnClose
    KILL \
_END

_SUB OnAbout
    MESS Custom Title Bar Demo v1.0@About#OK
_END
```

Key points:
- `-nocap` removes the system title bar; `-trap` prevents close-button auto-exit.
- `WM_NCHITTEST` on LABE controls returning `HTCAPTION` makes them draggable.
- 4-part color format `textColor#bgColor#hoverTextColor#hoverBgColor` enables hover effects.
- Unicode symbols (`&#x2500;` = ─, `&#x2715;` = ✕) provide button glyphs via HTML entities.

---

## 42. Struct Array Traversal (API Return Data)

Walk a struct array returned from a Win32 API call. Allocate a buffer, enumerate indices, calculate field offsets, read typed values, check the termination condition.

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

Key points:
- `SET-ptr` + `SET-make` reads a pointer value at an offset and copies the data it points to.
- `SET-long` writes a 32-bit integer to a buffer at a specific offset (used for `cbSize`).
- `SET?int` reads a 32-bit signed integer; `SET?longlong` reads 64-bit.
- Struct byte offsets are manually calculated from the MSDN layout.
- `LOOP #1=1` with `EXIT LOOP` is the standard pattern for an arena-style iteration when the count isn't known upfront.

---

## Pattern Index

| # | Pattern | Description |
|---|---------|-------------|
| 1 | Disk Enumeration & Information | List disks, partitions, map drive letters |
| 2 | Device Enumeration via Win32 API | SetupAPI enumeration, IOCTL calls |
| 3 | Boot Environment & System Info | BIOS/UEFI, Secure Boot, WinPE detection |
| 4 | File & Config Operations | READ, WRITE, GETF#, INI parsing |
| 5 | Registry Operations | REGI read/write/enum, all types |
| 6 | Process & Program Execution | EXEC*, callbacks, sub-PECMD |
| 7 | GUI Patterns | Window templates, TABL, LIST, SWIN |
| 8 | Threading & Async | THREAD*, POSTMSG, shared flags |
| 9 | Single Instance / Mutex | LOCK mutex, window restore |
| 10 | Hotkey Registration | HKEY, system-wide shortcuts |
| 11 | String Manipulation | MSTR, SED, LPOS, RPOS, RSTR |
| 12 | Dynamic Variables & Arrays | Indirect deref, delayed expansion |
| 13 | Timer & Scheduler | TIME, one-shot, variable callbacks |
| 14 | Compound Conditions & Flow Control | IFEX/FIND compound, EXIT variants |
| 15 | Encryption & Hashing | BASE, HASH, CMPS |
| 16 | Network Operations | IP config, WiFi scan/connect, ping |
| 17 | Date/Time | DATE, TIME, DTIM |
| 18 | Math & Calculation | CALC integer/float/hex |
| 19 | Cross-Process Window Control | ENVI @@Visible, ENVI @@POS |
| 20 | PART Operations | Partition management toolkit |
| 21 | Resource Embedding | LOAD #ID, EXEC* -exe:#ID |
| 22 | System Tray Icon | TIPS*, WM_TRAYNOTIFY handler |
| 23 | System Power & Display Control | SHUT, DISP, SCRN |
| 24 | Directory Search Across All Drives | FORX @ |
| 25 | Offline Registry Manipulation | offreg.dll API |
| 26 | Window Style Flags & Hiding | -nocap, -trap, ENVI @@Visible |
| 27 | Resource Extraction & Embedded Tools | EXEC* -exe, LOAD #ID, icons |
| 28 | BROW — File/Directory Browse Dialog | Save/open/folder dialogs |
| 29 | SUBJ — Mount/Unmount Drive Letters | Drive letter assignment |
| 30 | NET & Network Card Operations | PCIP query/set |
| 31 | SEND / WAIT -cont — Keyboard Input | Send keys, wait for keypress |
| 32 | Multi-Part Color Format | 4-part color for hover effects |
| 33 | Dynamic Control Creation (Command String) | Variable-expanded command strings |
| 34 | WM_TRAYNOTIFY Pattern | Full tray icon handler |
| 35 | SWIN Nested Windows (Tab Pages) | TABS + SWIN property pages |
| 36 | Dynamic Row Creation & Batch Deletion | Runtime control creation & loop deletion |
| 37 | Parameter Validation Guard | Early-exit argument guards |
| 38 | MSTR String Splitting — All Variants | Complete MSTR parsing cookbook |
| 39 | TABL Scrollbar Control via LVM | LVM_SCROLL, LVM_ENSUREVISIBLE messages |
| 40 | TABL In-Row Sorting | Bubble sort on table data |
| 41 | Custom Title Bar Window | Frameless window with manual caption |
| 42 | Struct Array Traversal | API struct array enumeration |

---

## Notes on Chinese Variable Naming

Real PECMD code from the Chinese WinPE community overwhelmingly uses Chinese variable names. This is the de facto standard. When writing scripts for this ecosystem, use Chinese names like `全部磁盘`, `分区信息`, `盘符`, `磁盘类型`, etc. However, PECMD fully supports English variable names and you may use either convention depending on the target audience. The code patterns above intentionally mix both to show both styles are valid.
