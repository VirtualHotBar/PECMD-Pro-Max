# 进程/线程/系统信息/工具 — 代码配方

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

## 21. Resource Embedding & Extraction

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


### Icon and image resources

```wcs
IMAG Btn,L0T0W64H64,#1000                           // display image from resource #1000
TIPS* tooltip text,,,#1                              // use icon from resource #1
```
---

## 22. System Tray Icon (Full Handler)

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


### Full tray click handler with visibility toggle

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

## 30. SEND / WAIT -cont — Keyboard

```wcs
SEND {ENTER}                                        // send Enter key
SEND 0x11_,0x12_,0x2E,0x12^,0x11^                  // Ctrl+Alt+Del (press order)
WAIT -cont -1000,&&key                              // wait up to 1s for key, returns VK code
```

---

## 31. Multi-Part Color Format

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

---

## 34. Parameter Validation Guard

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

## 35. MSTR String Splitting — All Practical Variants

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

## 39. Struct Array Traversal (API Return Data)

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

## Additional Patterns

### Pattern 57: Win32 API Two-Call Buffer Pattern

Many Win32 APIs require calling twice: once to get the required buffer size, then allocate, then call again. This is the canonical reusable template:

```wcs
// Generic two-call buffer pattern
// Step 1: Call with NULL/0 to get required size
CALL $--qd --ret:&retSize DLL.dll,FunctionName,*#0,#0,...
// Step 2: Allocate buffer with returned size
SET$# &buffer=*%&retSize% 0
// Step 3: Call again with actual buffer
CALL $--qd --ret:&retSize DLL.dll,FunctionName,*&buffer,#%&retSize%,...
```

Used by: GetWindowsDirectoryW, GetSystemDirectoryW, GetTempPathW, GetComputerNameW, GetUserNameW, GetIfTable, QueryDosDeviceW, GetAdaptersInfo, GetModuleFileNameW.

Example - Get Computer Name:
```wcs
_SUB GetComputerName
    CALL $--qd --ret:&ret Kernel32.dll,GetComputerNameW,*#0,*#0
    SET$# &buf=*%&ret% 0
    CALL $--qd --ret:&ret Kernel32.dll,GetComputerNameW,*&buf,*&ret
    ENVI-ret %~1=%&buf%
_END
```

### Pattern 58: GUID Byte-Swap via SED Regex (CLSIDFromString Fallback)

When `ole32.dll!CLSIDFromString` is unavailable (common in minimal PE), construct GUID binary from string using regex byte-swap:

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

### Pattern 59: WndProc Binding for Win32 Callbacks

Register a PECMD `_SUB` function as a Win32 callback (e.g., for EnumResourceNames, EnumWindows):

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

### Pattern 60: WM_COMMAND + EN_CHANGE Edit Monitoring

Monitor edit control text changes via WM_COMMAND notification:

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

### Pattern 61: WM_MOUSEHOVER/LEAVE Hover Tooltips

Show tooltips on mouse hover over controls:

```wcs
SET &WM_MOUSEHOVER=0x02A1
SET &WM_MOUSELEAVE=0x02A3
ENVI @Label1.MSG=_%&WM_MOUSEHOVER%: TIPS Title,"Hover text\nLine 2",3000,1
ENVI @Label1.MSG=_%&WM_MOUSELEAVE%: TIPS *
```

### Pattern 62: WM_SIZE Responsive Layout with Saved Positions

Full DPI-aware window resize handling:

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

### Pattern 63: LoadLibraryExW for Resource-Only Loading

Load a DLL/EXE purely for resource extraction without executing code:

```wcs
SET &LOAD_LIBRARY_AS_DATAFILE=0x00000002
SET &LOAD_LIBRARY_AS_IMAGE_RESOURCE=0x00000020
CALC #&flags=%&LOAD_LIBRARY_AS_DATAFILE% | %&LOAD_LIBRARY_AS_IMAGE_RESOURCE%
CALL $--qd --ret:&hMod Kernel32.dll,LoadLibraryExW,$%&filePath%,#0,#%&flags%
// Now use EnumResourceNamesW, LoadResourceW etc. on &hMod
// Don't forget to free: CALL $--qd kernel32.dll,FreeLibrary,#%&hMod%
```

---

### 46. FVAR Secure Boot (Direct EFI Variable)

```wcs
// Method 1: Direct EFI global variable read (simplest)
ENVI ?&ret=FVAR,SecureBoot;{8be4df61-93ca-11d2-aa0d-00e098032b8c}
// Returns 0=off, 1=on

// Method 2: NtQuerySystemInformation #145 (see Boot Environment pattern above)
```


### 49. Display Mode Presets with Timeout

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
    FIND $0=%&&__YesNo%, DISP     // if still 0, auto-revert
_END
```

---

### 50. QueryDosDeviceW — All MS-DOS Devices

```wcs
ENVI$ &&buf=*0x100000 0
CALL $--qd --ret:&bret kernel32.dll,QueryDosDeviceW,#0,*&&buf,#0x80000
// Returns null-delimited, double-null terminated device list
// lpos* * for binary null pattern search
LPOS* * &&pos=0x00 0x00 0x00 0x00,1,&&buf
// Extract and decode: GETF -bin → MSTR * → SED -ex → CODE ***unicode
CODE ***unicode,**.buf,*uni,&&result
```


### 51. RtlGetNtVersionNumbers (Pointer-Based)

```wcs
ENVI$# &&Major=*4 0
ENVI$# &&Minor=*4 0  
ENVI$# &&Build=*4 0
CALL $--qd --ret:&bret ntdll.dll,RtlGetNtVersionNumbers,*&&Major,*&&Minor,*&&Build
SET?int &&Major=&&Major:0
SET?int &&Minor=&&Minor:0
SET?int &&Build=&&Build:0
CALC &BuildNumber=%&Build% & 0xFFFF   // mask high 16 bits
```


## 73. Thread Variable Async Conflict & Fix

### Problem: shared variable race condition

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

### Fix 1: copy to local before spawning

```wcs
{
    SET &I2=%I%                                       // copy to local
    THREAD* TEAM WAIT 100| MESS I=%&I2% J=%J%        // I2 is safe
    CALC I=%I% + 1
}
```

### Fix 2: use THREAD$ for pre-interpretation

```wcs
{
    THREAD*$ TEAM WAIT 100| MESS I=%&I% J=%J%        // %&I% resolved BEFORE thread starts
    CALC I=%I% + 1
}
```

### THREAD vs THREAD* variable sharing rules

| Context | THREAD (no *) | THREAD* (with *) |
|---------|---------------|-------------------|
| Regular function `{}` block | Copy (isolated) | Copy (isolated) |
| Window `_SUB` or `_SUB F,*` | Copy (isolated) | **Shared** (direct link) |
| `{}` inside window `_SUB` | Copy (isolated) | Copy (isolated — `{}` demotes to temporary) |

---

## 76. Random String Generation

```wcs
SET &CSet=0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz
STRL * &&LCSET=CSet
SET &V=
SET &n=10                                            // desired length
LOOP #%n%>0,
{
    CALC n=%n% - 1
    ^CALC &&i=%RANDOM% % %LCSET% + 1
    MSTR * &&vi=%i%,1,CSet
    SET< V=%vi%
}
// &V now contains 10 random alphanumeric characters
```

---

## 77. System Font Charset Constants

Useful when setting fonts via `ENVI @ctrl.Font=size:name:style:charset`.

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

## Non-Codebook Quick Reference (PECMD补充说明.doc extracts)

The `PECMD补充说明.doc` is the authoritative source for advanced patterns:

### THREAD* Stack Chain Rules
- In a window_SUB (persistent stack): THREAD* shares PE variables directly
- In a temporary function/block ({}) : THREAD* copies PE variables (isolated)
- `THREAD$` : pre-interpret once before launch (uses literal values, avoids async clash)
- `-link` : maintain parent-child window connection; wait for child thread end

### PE Variable Destructor
```wcs
SET-def ~CloseHandleX~h=0    // define h AND register destructor CloseHandleX
// When scope exits: CloseHandleX %&h% → PE var h released
// Destructors run in reverse order of definition
```

### Function Destructor (`_SUB Func,*,析构命令`)
```wcs
_SUB F1,*,IFEX #[ %&h%>0 ], CALL $kernel32.dll,CloseHandle,#%&h%
    // ... function body with early EXIT
_END  // destructor command runs automatically on exit
```

### #& Control Naming (Shared PE Variable)
```wcs
LIST #&L7,L410T55W46H23,1|2|3|4,,1,    // control name is #&L7, variable is %&L7%
// Access from parent/other pages: ENVI @Page1:#&L7.VAL=...
```

### ENVI^ Alias System
```wcs
ENVI^ Alias aliasName=[cmd prefix]
ENVI^ Alias * aliasName=cmd             // * = enable prefix+space syntax
```

### ENVI @@POSTMSG/SENDMSG Full Syntax
```wcs
ENVI @@SENDMSG=[:retVar;]windowID;messageID[;wParam[;lParam]]
// wParam,lParam: @PEvar (buffer), $string (SENDMSG only), number
// message with # prefix = PECMD custom message 1-N
// _ = second-half response mode (responds after system)
```

### PUT/GET Binary Resource Export
```wcs
// #.N = raw (original) resource data
PUTF -dd -bs=10M out.dat,0,"%MyName%""#.101|SCRIPT"
// #N = decompressed resource
PUTF -dd -bs=10M out.dat,0,"%MyName%""#2|INDATA"
// Resource type IDs: CURSOR=1 BITMAP=2 ICON=3 MENU=4 DIALOG=5, STRING=6,
//   FONTDIR=7 ACCELERATOR=9 RCDATA=10 GROUP_ICON=14 VERSION=16 MANIFEST=24
```

### FIND/IFEX Shortened Block Syntax (>=79N-59D)
```wcs
FIND $1=1,FIND body! ELSE body          // single-line TRUE + ;ELSE
FIND $1=1,
{   MESS TRUE
}! MESS FALSE                           // multi-line TRUE, single-line ELSE on }!
FIND $1=1, { MESS TRUE                  // TRUE block first line inline
}! { MESS FALSE }
```

### SED Regex Syntax (from PECMD2012正则表达式.doc)
```
.   = any char       [abc] = char class  [^abc] = negated class
?   = 0-1 times      + = 1+ times        * = 0+ times
??  = non-greedy ?   +? = non-greedy +   *? = non-greedy *
()  = group          {} = named group (reference via \1-\9)
^   = start anchor  $ = end anchor      | = alternation
\\a = [a-zA-Z0-9]   \\d = [0-9]          \\h = [0-9a-fA-F]
\\w = [a-zA-Z]+     \\z = [0-9]+          \\n = newline
Replacement: \\0=entire match \\1-\\9=group refs \\u=uppercase \\l=lowercase
```

### PECMD Variable → CMD Variable (3 methods)
```
// Method 1 (best): WRIT to stdout
WRIT -,$+0,a 111        // CMD FOR /F captures output

// Method 2: Temp file
WRIT %tmpf%,$+0,set a=%val%   // then CALL .\tmpf.CMD

// Method 3: Registry
REGI HKCU\PECMD_U\var=%val%   // CMD reads via reg query
```

---

