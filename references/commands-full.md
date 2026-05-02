# PECMD Command Reference

Full command reference organized by category. All commands are case-insensitive.
Syntax conventions: `<required>`, `[optional]`, `|` = alternatives.

---

## SCRIPT STRUCTURE

### _END
End of a `_SUB` function or code block.
```
_END
```
Must be on its own line. Each `_SUB` requires one `_END`.

### _ENDFILE
End of script file. Code after this line is never loaded.
```
_ENDFILE[-IMPORT]
```
`-IMPORT`: Only effective when the file is IMPORTed.

### _SUB — Define function / class / window
```
_SUB FuncName [*]                   // function (* = this-call, caller's stack)
_SUB FuncName,*,,destroyCmd         // function with destructor command
_SUB WinName,<shape>,[title],[closeCmd],[icon],[style],[mask],[flags]  // window
```
Window shape: `LleftTtopWwidthHheight`. Omit L/T for centered.
Window flags:
`-top` (always on top), `-nocap` (no title bar), `-nosysmenu` (no system menu),
`-trap` (close doesn't exit), `-size` (resizable), `-maxb` (enable maximize),
`-minb` (enable minimize), `-disminb` (disable minimize button),
`-discloseb` (disable close button), `-nfocus` (no keyboard focus),
`-ntab` (no tab stop), `-disaltmv` (disable ALT-drag),
`-forcenomin` (prevent minimize), `-scalef` (XP-style DPI scaling),
`-scale[:DPI]` (Win8+ DPI scaling), `-nxp` (no XP visual style),
`-csize` (size=client area), `-na` (don't activate on creation)
Style: `[#][$]`number for transparency, `,#` for hidden window.
Mask: `[color][*][w:h]bmpname` for shaped window.

### CALL — Call function/window/DLL
```
CALL FuncName [args...]               // call function
CALL *FuncName [args...]              // this-call (caller's stack)
CALL @WinName [args...]              // create/show window (modal, blocks)
CALL @*WinName [args...]             // parallel window
CALL @-WinName [args...]             // background window
CALL @~WinName [args...]             // background, non-blocking
CALL @+WinName [args...]             // abandoned child window
CALL @^WinName [args...]             // parallel, parent doesn't block child
CALL @--popmenu WinName [x.y[:align]] // popup menu
CALL @--WinName                      // destroy Win environment
CALL @WinName                        // initialize Win environment

// DLL calling
CALL $[? --cd --nrcd --c --ret:retVar] DLL|*hDll,Func,[#]p1,[#]p2...
CALL $--ret:retVar [--cd],[--nrcd],-LoadLibrary,[^]DLLpath     // load DLL (^=auto-free)
CALL $--ret:retVar [&&memVar],-LoadLibrary,*[file]#resID[|type] // load from memory
CALL $--ret:retVar ,-GetProcAddress,*hDll,FuncName             // get function address
CALL $[--ret:retVar] ,-FreeLibrary,*hDll                        // free DLL
CALL $--win [--qd@ --cd --nrcd --ret:retVar] DLL,Func,cmdLine   // rundll32
CALL $--cpl CPLpath                                             // control panel
```
DLL flags: `--cd`=chdir, `--nrcd`=don't restore, `--c`=C convention,
`--bool`=BOOL return, `--ret:*`=return via pointer, `--m`=in-memory,
`--1`=all remaining as one param, `--qd@`/`--qd#`/`--qd$`/`--qd*`=per-param type

### EXIT — Terminate
```
EXIT FILE      // terminate entire script
EXIT _SUB      // exit current function
EXIT LOOP      // break out of loop
EXIT FORX      // break out of FORX
EXIT BLOCK     // exit current {} block
EXIT -         // continue (skip to next iteration)
```

### IMPORT — Include library
```
IMPORT path\to\library.wcs
```
Imports functions from another file. `_ENDFILE-IMPORT` in the imported file excludes trailing content.

### LOAD — Execute script file
```
LOAD path\to\script.ini [args]
LOAD #101 [args]                           // built-in script from EXE resources
LOAD --mem &var [args]                     // execute code stored in variable
LOAD --Local --EnviMode path.ini           // run as ForceLocal=1 + EnviMode=1
```

### THREAD / THRD — Create thread
```
THREAD[*][&][+][$][#] [-exp] [-wait[x][-here]] [-tid:var] [--st:stackSize] command
```
`*` = immediate, `&` = force PE var mode, `$` = pre-interpret, `+` = abandoned thread,
`-wait` = wait for completion, `-tid:var` = get thread ID

---

## VARIABLES & DATA

### ENVI / SET — Set / query variables
```
ENVI (SET) [&][$][@]VarName=Value
ENVI^ EnviMode=1|ForceLocal=1|FORCELOCAL=1|LoadEnvi [...]
ENVI $var=val              // set environment variable
ENVI &var=val  (alias: SET var=val)   // set local PE variable
SET &::var=val             // set class/global PE variable
ENVI-def var=val  (alias: SET-def var=val)  // only if not already defined
ENVI-ret[level] %~1=%val%  // return-by-reference (default level=1)
ENVI~ &&Dst=Source.Key     // ~ = indirect expansion
```
Control operations:
```
ENVI @Ctrl=Text                   // set control text
ENVI @Ctrl.Enable=0|1             // disable/enable
ENVI @Ctrl.Visible=0|1|*4        // hide/show/minimize
ENVI @Ctrl.POS=l:t:w:h            // move/size
ENVI @Ctrl.POS=?&L:&T:&W:&H     // query position
ENVI @Ctrl.Check=0|1|2|-1|-2       // checkbox state (1/-1=checked, 0/2/-2=unchecked, <0=grayed, ±16=invisible)
ENVI @Ctrl.Val=data               // set content
ENVI @Ctrl.Val=?row.col;&var      // get cell (semicolon)
ENVI @Ctrl.Val=?*;&count          // get row count
ENVI @Ctrl.Val=-*                 // clear all rows
ENVI @Ctrl.Val=1*;%&data%         // bulk-set from variable
ENVI @Ctrl.Sel=idx|idx;0          // select/deselect
ENVI @Ctrl.MSG=_msgId:cmd         // message map (control notifications use _)
ENVI @Ctrl.POSTMSG=#msg;wp;lp     // async post message
ENVI @Ctrl.SENDMSG=#msg;wp;lp     // sync send message
ENVI @Ctrl.*del=                  // destroy control
ENVI @Ctrl.Font=size:name
ENVI @Ctrl.bkcolor=0xRRGGBB
ENVI @Ctrl.Cursor=32649           // hand cursor
ENVI @@POS=wid:l:t:w:h:layer:trans:front:activate
ENVI @@Visible=wid:0|1|*4        // cross-process visibility
```

### CALC — Calculate / evaluate
```
CALC [#][变量=]表达式[#[#][小数位][E|F|G]]
```
`#` prefix = integer mode. Supports: `+ - * / % ^`, bitwise `& | @`, comparison `= <> > >= < <=`,
logic `&& || lnot`. Functions: `abs sin cos tan sqrt ln lg log pow exp pow10`,
`floor ceil round int frac div mod rand shl shr xor not`,
`arcsin arccos arctan arcctg deg rad hypot max min`.
`CALC -base=16 #&hex=shl(0x07,16)|0x20` — hex bitwise.
`CALC &sz=%&bytes%/1G#3` — bytes to GB, 3 decimal places.

### CODE — Encoding conversion
```
CODE -srcFmt,srcFile,-dstFmt,dstFile
CODE **-GBK,&src,**-UNI,&dst
```
Formats: `-ANSI`, `-UNICODE`, `-UTF8`, `-UTF7`, `-GBK`, `-BIG5`, `-UNICODEB`, `-BOM`

### SET$ / ENVI$ — Create string from hex
```
SET$ &Var=0d 0a                   // variable = CR+LF (Unicode wide string)
SET$# &Buf=*4096 0                 // allocate zero-filled raw byte buffer
ENVI$ &data=*1M 30 0d 0a          // variable-length allocation
```

### SET-def — Set only if not defined
```
SET-def Var=DefaultValue
```

### SET-copy — Raw byte copy
```
SET-copy Dst=&src;srcOff;len;dstOff
```

### SET-long / SET-short / SET-ptr — Write typed values
```
SET-long &buf=value:offset         // 32-bit int at buffer offset
SET-short &buf=value:offset        // 16-bit
SET-ptr &buf=value:offset           // pointer-sized
```

### SET?int / SET?longlong / SET?char — Read typed values
```
SET?int &buf=&&Var:offset
SET?longlong &buf=&&Var:offset
SET?char &buf=&&Var:offset
```

### SET-make / ENVI-make — Substring from buffer
```
SET-make &&Str=&buf@offset;$length
```

### SET< / ENVI< — Append to variable
```
SET< Var=text to append
```

### ENVI-addr — Get buffer address
```
ENVI-addr &&ptr=&buf
```

### ENVI-mkdummy — Dummy pointer/length descriptor
```
ENVI-mkdummy &&Name=&buf@offset;length
```

---

## FLOW CONTROL

### FIND — String comparison
```
FIND $str1=str2, command        // equal (case-sensitive)
FIND $str1<>str2, command       // not equal
FIND $=%var%, command           // var is empty
FIND $%var%=, command           // var is empty (same, variable on left)
FIND $str1=str2,! cmd1! cmd2   // if else (separated by !)
FIND $str1=str2,!! command      // else only
FIND |num1>num2, command        // numeric comparison
FIND $'%var%'='', command       // safe empty check (single quotes)
FIND [$][A & B], command         // compound AND (& between conditions)
FIND [$][A | B], command         // compound OR (| between conditions)
FIND --pid &var,                // get process/CPU ticks
FIND --pid*@[.ext|#parentPID] &var,  // process list (opt: extension filter or parent PID)
FIND --wid*@[parentWID] &var,[title] // window list (opt: parent window filter)
```

### IFEX — File test / numeric comparison / system query
```
IFEX path\|file, command         // file exists
IFEX path\|file,! command        // NOT exists
IFEX x:\, command                // drive letter exists AND has filesystem
IFEX $num1>=num2, command        // numeric comparison
IFEX #num1=#num2, command        // force integer
IFEX [ cond1 & cond2 ], command  // AND compound (& and | or @ between conditions)
IFEX [ cond1 | cond2 ], command  // OR compound
IFEX [ cond1 @ cond2 ], command  // XOR compound
IFEX MEMU=?,&var                 // query free memory
IFEX MEMA=?,&var                 // query total memory
IFEX drv:\=?,&var               // query disk free space
IFEX KEY=?                       // wait for key press
```

### LOOP — While loop
```
LOOP [#]condition,
{
    // body
}
// BREAK: EXIT LOOP or EXIT -
// CONTINUE: EXIT -
```

### FORX — Iterate
```
FORX * list,&&item,              // space-delimited
FORX *NL &multiLine,&&line,      // newline-delimited
FORX *v &a &b &c,&&name,         // iterate variable names
FORX /S path\*.ext,&&name,0      // file enumeration (0=files, 1=dirs)
FORX @\Windows,&&dir,1           // search for directory root
```

### TEAM — Multi-command
```
TEAM cmd1 | cmd2 | cmd3 ...
```
Nested separators: `|` (level 1), `||` (level 2), `|||` (level 3).

### LOCK — Critical section / mutex
```
LOCK #lockName,&retVar           // create/acquire named lock
LOCK --exist #lockName,&retVar   // check if lock exists (1=yes, 0=no)
{ LOCK #pecmd ... }              // wrap in braces for atomic scope
```

---

## FILE I/O

### READ — Read file
```
READ path,*r,&var     // raw (no line-end conversion)
READ path,*,&var      // UNIX LF -> native
READ path,**,&var     // DOS CRLF -> native
READ -,-1,&count,&var // get line count
READ -,lineNo,&line,&var  // read specific line
```

### WRIT — Write to file
```
WRIT path,$0,text      // $ = ANSI, 0 = line 0 (overwrite)
WRIT path,$+0,text     // append
```

### GETF — Binary file read
```
GETF# path,offset#size,&var       // read raw bytes at offset, size bytes
GETF# path,0#*,&var               // read entire file
```

### PUTF — Binary file write
```
PUTF -dd -len=0 path,0,zero       // create/truncate
PUTF path,offset,#&data            // write at offset
PUTF -dd -bs=1M src,0,dst,0,-len=size  // disk-to-disk copy
```

### FILE — File/directory operations
```
FILE src -> dst               // copy
FILE -force src -> dst        // force overwrite
FILE src                      // delete
FILE -r dir                   // recursive delete
FILE -md dir                  // create directory
FILE -simpleprogress src -> dst  // with progress
```

### DIR — List directory
```
DIR &var /s /b path            // get listing into variable
```

### FDIR / FEXT / FNAM / NAME — Path parts
```
FDIR &var=fullPath              // directory part
FEXT &var=fullPath              // extension (e.g. "EXE")
FNAM &var=fullPath              // filename with extension
NAME &var=fullPath              // filename without extension
```

### SIZE — File size
```
SIZE &var=filePath
```

### HASH — Compute hash
```
HASH filePath,&var,MD5|SHA1|SHA256|CRC32
HASH $string,&var,SHA1          // hash string content
```
Default algorithm: MD5. Without variable, displays result in message box and copies to clipboard.

### MDIR — Create directory
```
MDIR dirPath
```

### FLNK — Symbolic/hard link
```
FLNK linkPath,targetPath
FLNK -h linkPath,targetPath     // hard link
```

---

## DISK & PARTITION

### PART — Partition management (comprehensive)
```
// List operations
PART list disk,&var                         // list all disk numbers
PART list disk N,&var                       // disk info (size, cylinders, heads, etc.)
PART list part N,&var                       // list partition numbers on disk N
PART -hextp -phy# list part N#M,&var        // detailed partition info (hex type + physical#)
PART -devid list disk N,&var                // device path/ID
PART -devidx list disk N,&var               // physical serial number
PART list drv D:,&var                       // info about specific drive letter
PART list volume volumeName,&var            // info about volume

// Modify operations
PART -super -up -xup N#M type [attr]        // set partition type+attribute (use both -super -up)
PART -super -up -swap:M N#P                // swap physical partition numbers
PART -super -up N#M a|A|-a|-A type start len // create partition (a=active, A=extended)
PART -super -up del N#M                    // delete partition
PART -super -up N#M a|A                     // set active/inactive on existing partition
PART update N                               // refresh disk info from system

// MBR/PBR operations
PART /mbr[=nt6|=win|=nt5|=dos|=file] N      // rewrite MBR on disk N
PART /pbr[=nt6|=win|=nt5|=dos|=file] N#M    // rewrite PBR on partition

// GPT operations
PART -gpt init N                            // initialize as GPT
PART -super -up -gpt N#M a type start len guid attr name  // create GPT partition
PART -gpt -cmp N                            // compress GPT partition table

// Other
PART -gui                                   // launch GUI partition manager
PART -usb                                   // USB-only mode
PART -admin                                 // advanced mode (dangerous)
PART -align[=size]                          // alignment (default or specify)
PART -fill                                  // fill empty drive letter slots
```

PART MBR output fields: `分区号 类型(hex) 激活 起始(字节) 长度(字节) 隐藏扇区 结束(字节) 物理# 盘符`
PART GPT output fields: `分区号 GUID 属性 起始(字节) 长度(字节) 结束(字节) 物理# 盘符`

### SHOW — Show/hide partitions
```
SHOW -1:-1                                // show all partitions
SHOW * N#M,driveLetter                     // assign drive letter
SHOW *- N#M,                              // remove drive letter
SHOW & N#M,driveLetter                     // local-mode assign
SHOW =1 * N#M,driveLetter                  // skip if already loaded
SHOW -check * N#M,driveLetter              // skip if no valid filesystem
```

### SUBJ — Mount/unmount
```
SUBJ D:,\Device\Harddisk0\Partition1       // mount
SUBJ -D:                                   // unmount
```

### FDRV — Drive enumeration
```
FDRV &var=*:                               // all drive letters with volumes
FDRV *idle &var=*:                         // idle (unassigned) drive letters
FDRV *vol &label,&fs=D:                   // get volume label and filesystem
FDRV *rsort &var=*:                        // reversed sort order
```

### FORM — Drive type (comprehensive)
```
// Basic filesystem query
FORM &var=D:                               // filesystem type string (e.g. "NTFS", "FAT32", "CDFS")
FORM -raw &var=D:                          // drive type constant (see table below)
FORM TYPE,&var,BUS=D:                      // bus type string (e.g. "USB", "SATA", "SCSI", "NVMe")
FORM -raw &type,&bus,&drvType=&dsk,<drive>  // comprehensive: all type info in one call

// Drive type constants (returned by FORM -raw)
```
| Constant | Value | Description |
|---|---|---|
| DRIVE_UNKNOWN | 0 | Unknown drive type |
| DRIVE_NO_ROOT_DIR | 1 | Invalid/not mounted |
| DRIVE_REMOVABLE | 2 | Removable media (USB flash, floppy) |
| DRIVE_FIXED | 3 | Fixed disk (HDD, SSD) |
| DRIVE_REMOTE | 4 | Network/mapped drive |
| DRIVE_CDROM | 5 | Optical disc (CD/DVD/BD) |
| DRIVE_RAMDISK | 6 | RAM disk |

### DFMT — Format
```
DFMT d:,NTFS,label,quick
DFMT d:,FAT32,,quick
```

### EJEC — Eject
```
EJEC d:                                    // eject CD/DVD
EJEC * d:                                  // eject removable disk
```

### DISK — Disk operations
```
DISK 1,,,3                                 // initialize disk
DISK 0,,,1,U:                              // assign USB starting from U:
```

---

## MOUNT (WIM / VHD / UDM)

### WIM mounting
```
MOUN wimFile,mountDir,[index],[tempDir]     // mount WIM image
MOUN -u mountDir                            // unmount WIM
MOUN -query &var                            // query mounted images
MOUN -svr wimFile,mountDir                  // persistent mount (server mode)
```

### VHD/VHDX mounting
```
MOUN-vhd -c[x] file.vhd,size                // create (x=expand first)
MOUN-vhd -c[x] -d file.vhd,size             // create dynamic (sparse)
MOUN-vhd -r file.vhd,mountDir               // mount read-only
MOUN-vhd -u mountDir                        // unmount
MOUN-vhd -iso file.iso,mountDir             // mount ISO
MOUN-vhd -query file.vhd,&var              // query VHD info
```

### UDM (Ultra Deep Mount) — hidden partition mounting
```
MOUN-udm [flags] \\.\PhysicalDriveN
MOUN-udm -findboot -ret:&retVar             // find and mount boot device
MOUN-udm -mhide \\.\PhysicalDriveN D:       // mount hidden partition
MOUN-udm -udimg:file.img                    // mount UD image file
MOUN-udm -u mountDir                        // unmount
```
Flags: `-ud` (UD partition), `-uh` (UD high), `-muh` (mount UD high),
`-u+` (U+ partition), `-udfs` (UD filesystem), `-udm-` (disable UDM),
`-mall` (mount all), `-mhide` (mount hidden only),
`-findboot` (auto-find boot device), `-ret:var` (return device path),
`-CheckFile:path` (verify by file existence), `-tag:name` (tag to identify)

---

## SYSTEM

### MAIN — WinPE entry point
```
MAIN path\to\PECMD.INI
```
Starts the desktop, hooks Ctrl+Alt+Del, runs the config file, and enters the message loop. This is the standard PE boot entry command.

### INIT — Initialize
```
INIT [options],[timeout]
```
Options: `I`=keyboard, `U`=USB, `C`=disable Ctrl+Alt+Del, `K`=kill explorer, `P`=pagefile
Common: `INIT IU,3000`

### SHEL — Set Windows shell
```
SHEL %SystemRoot%\explorer.exe
SHEL PECMD.EXE LOAD MyShell.ini
    cmd_on_shell_change
```
The indented line(s) execute when the shell transition happens.

### SHUT — Shutdown/restart
```
SHUT                        // shutdown
SHUT R                      // restart
SHUT S                      // suspend/standby
SHUT H                      // hibernate
SHUT L                      // logoff
SHUT K                      // lock workstation
SHUT E                      // eject optical drive
SHUT C                      // close optical drive
SHUT O                      // eject optical + wait 10s
```

### DISP — Display settings
```
DISP W1024H768B32F60       // width, height, color bits, refresh rate
DISP                       // auto-detect best mode
```

### PAGE — Virtual memory
```
PAGE C:\pagefile.sys 256 512   // min 256MB, max 512MB
```

### RAMD — RAM disk (ImDisk)
```
RAMD ImDisk,L100,FAT32,C:,MyRam      // create
RAMD ImDisk* -D -m G:                 // remove
```

### SERV — Service management
```
SERV servicename                     // start service
SERV -stop servicename               // stop
SERV -query servicename,&var         // query status
SERV -create name,path,type,start   // create service
SERV -delete name                    // delete service
```
Start types: `-boot`, `-system`, `-auto`, `-demand`, `-disabled`

### HOTK — System-wide hotkey
```
HOTK Ctrl+Alt+#0x41,execPath                // register (global, system-wide)
HOTK Ctrl+Shift+Alt+Win+#0x42,command       // multi-modifier
HOTK #0x0D,--del                            // unregister by key code
HOTK --del:keyname                          // unregister by name
HOTK -wait [timeout],&var                   // NOTE: non-standard extension. Use standard WAIT -cont [-timeout],[&var] instead.
```
Modifiers: `Ctrl`, `Alt`, `Shift`, `Win`. Combine with `+`.
Virtual key codes use `#` prefix (decimal or hex: `#0x41`).

### HKEY — Window/program-scope hotkey
```
HKEY #0x41,command                          // window-active only (responds when main window has focus)
HKEY $#0x41,command                         // $ = program-level global (any window of this PECMD instance)
HKEY Ctrl+Shift+#0x42,command               // multi-modifier
HKEY #0x0D,--del                            // unregister by key code
HKEY --del:keyname                          // unregister by name
```

### DATE — Date/time variables and sub-variables
```
DATE &var                                  // get current date (yyyy-mm-dd format)
DATE &var yyyy-mm-dd                       // set system date
DATE *[name] &var                          // get named day of week (e.g. "Monday")
DATE &var -now                             // get current date+time
DATE &var -utc                             // get UTC date+time
```
Sub-variables extracted from a DATE result (use `MSTR` or direct sub-variable syntax):
| Sub-variable | Meaning | Example |
|---|---|---|
| `%&var:ym%` | Year-Month | `2026-05` |
| `%&var:y%` | Year (4-digit) | `2026` |
| `%&var:mon%` | Month (2-digit) | `05` |
| `%&var:day%` | Day (2-digit) | `02` |
| `%&var:h%` | Hour (24h, 2-digit) | `14` |
| `%&var:min%` | Minute (2-digit) | `30` |
| `%&var:s%` | Second (2-digit) | `45` |
| `%&var:ms%` | Milliseconds (3-digit) | `123` |
| `%&var:w%` | Day of week (1=Mon..7=Sun) | `6` |
| `%&var:wd%` | Day of week name | `Saturday` |

### Various system commands
```
RUNS prog,Name                       // add to Run registry key
TEMP @SetTemp                        // set temp directory
PATH C:\Tools;%PATH%                 // set search path
RECY C:|*                            // empty recycle bin
USER username,password               // create user
HOME C:\Users\name                   // set home directory
```

---

## AUDIO & DISPLAY

### SCRN — Screenshot / capture screen
```
SCRN &w,&h                             // get screen width and height
SCRN -desk &w,&h                       // desktop resolution (all monitors combined, virtual screen)
SCRN -cur &x,&y                        // get cursor position
SCRN -cap scrn.bmp,&wid                // capture full screen to BMP, return window ID
SCRN -cap scrn.bmp,&wid,WxH            // capture at specific resolution
SCRN -cap scrn.bmp,&wid,WxH,x,y        // capture region at (x,y) of size WxH
SCRN -cap -capwid:WID scrn.bmp,&wid    // capture specific window
SCRN -cap -cur scrn.bmp,&wid           // capture with cursor included
SCRN -cap scrn.jpg,&wid,0,0,0,80       // capture as JPG (quality 80)
```
Capture formats: `.bmp`, `.jpg`, `.png` determined by file extension.

### FONT — Load / register fonts
```
FONT fontPath                           // register a single font file
FONT fontPath,fontName                  // register with specific name
FONT -reg fontPath                      // permanent registration (survives reboot)
FONT -unreg fontPath                    // unregister font
FONT fontDir\*                          // register all fonts in directory
FONT -list &var                         // list registered font names
FONT ?fontName,&var                     // query font info
```

### WALL — Set desktop wallpaper
```
WALL imagePath                           // set wallpaper (BMP, JPG, PNG, GIF)
WALL %SystemRoot%\Web\Wallpaper\img.jpg  // absolute path
WALL -center imagePath                   // centered (not stretched)
WALL -tile imagePath                     // tiled
WALL -stretch imagePath                  // stretched (default)
WALL -fit imagePath                      // fit to screen
WALL -fill imagePath                     // fill to screen
WALL -span imagePath                     // span across monitors
WALL ""                                  // clear wallpaper (solid color)
```

### SITE — File attribute query/set
```
// Query
SITE ?filePath                           // show attributes in message box
SITE &var,filePath                       // get attributes to variable (e.g. "A--RHS-")
SITE ?-attr,&var=&attr                   // query raw attribute flags

// Set
SITE +R,filePath                         // set Read-only
SITE +H,filePath                         // set Hidden
SITE +S,filePath                         // set System
SITE +A,filePath                         // set Archive
SITE -R,filePath                         // remove Read-only
SITE +R+H+S+A,filePath                   // combine multiple
SITE +R-H,filePath                       // set Read-only AND remove Hidden
SITE -R-H-S-A,filePath                   // clear all attributes
SITE +R+H,dirPath\*                      // apply to all files in directory (add trailing backslash)

// Encode variable (security)
SITE ?-all,VAR=variable                  // encode variable (PECMD-style)
SITE ?-sys,VAR=variable                  // encode with system flag
SITE ?H:hWnd,variable1[,variable2]       // copy to clipboard
```
Attribute flags in query result: `R`=Read-only, `H`=Hidden, `S`=System, `A`=Archive,
`N`=Normal, `D`=Directory, `C`=Compressed, `E`=Encrypted, `T`=Temporary, `O`=Offline.

---

## DRIVERS & DEVICES

### DEVI — Device driver installation
```
// Install from CAB/INF/folder
DEVI [$]<CAB文件>[,匹配级别[,解压目录]]         // install from CAB
DEVI [*nocheck] <INF文件>[,DevClass]           // install from INF
DEVI [*rescan] <含有INF的目录>[,DevClass]       // install from directory
DEVI $<INF文件>,[安装节],[操作码]               // advanced install
DEVI *extract <CAB>[,匹配级别],解压目录          // extract only

// List devices
DEVI listdev:var [*devclass:Class] [*ALL] [*listdev=i|c|+]
DEVI listdev:var *many *devid:PCI\VEN_14E4*     // list with filter

// Control devices
DEVI *enable:[h|c|+:]devID                      // enable device
DEVI *disable:[h|c|+:]devID                     // disable device
DEVI *remove:[h|c|+:]devID                      // remove device
DEVI *restart:[h|c|+:]devID                     // restart device
DEVI *status:retVar:[h|c|+:]devID               // query status
DEVI *update:hardwareID:INF                      // update driver
DEVI *install:hardwareID:INF                     // install driver

// Other
DEVI *rescan[:Fun]                               // rescan devices
DEVI buildcache:[-a:arch] dir                    // build driver cache
```

### FBWF — FBWF cache control
```
FBWF [Ppercent] [Lmin] [Hmax] [Fremain]
```
All values in MB. Example: `FBWF P50 L200 H300` — 50% of memory, min 200MB, max 300MB.

---

## REGISTRY

### REGI — Read/write registry
```
// Read (all type prefixes)
REGI $HKLM\SOFTWARE\Key\Val,&var          // REG_SZ
REGI #HKLM\SOFTWARE\Key\Val,&var          // REG_DWORD
REGI @HKLM\SOFTWARE\Key\Val,&var          // REG_BINARY
REGI *HKLM\SOFTWARE\Key\Val,&var          // REG_MULTI_SZ
REGI **HKLM\SOFTWARE\Key\Val,&var         // REG_MULTI_SZ (special)
REGI *$HKLM\SOFTWARE\Key\Val,&var         // multi-line REG_MULTI_SZ
REGI ~HKLM\SOFTWARE\Key\Val,&var          // REG_EXPAND_SZ
REGI ~~HKLM\SOFTWARE\Key\Val,&var         // REG_EXPAND_SZ (variant)
REGI +HKLM\SOFTWARE\Key\Val,&var          // REG_QWORD
REGI ^HKLM\SOFTWARE\Key\Val,&var          // REG_LINK
REGI bHKLM\SOFTWARE\Key\Val,&var          // REG_QWORD_BIG_ENDIAN
REGI uHKLM\SOFTWARE\Key\Val,&var          // REG_MUI_SZ
REGI nHKLM\SOFTWARE\Key\Val,&var          // REG_NONE
REGI .HKLM\SOFTWARE\Key\Val,&var          // offline registry (offline Windows/system)
REGI HKCU\Software\Key\,&&keys            // enumerate subkeys (NL-delimited)

// Write
REGI $HKLM\SOFTWARE\Key\Val=string         // write REG_SZ
REGI #HKLM\SOFTWARE\Key\Val=#0x100        // write REG_DWORD (hex)
REGI $HKLM\SOFTWARE\Key\Val=               // delete value

// Advanced
REGI --ak HKCU\Software\Key\,&all         // enumerate ALL values for key
REGI --av HKCU\Software\Key\,&all         // enumerate ALL subkeys
REGI .? \HKLM\SOFTWARE\Key\Val,&type      // query type (dot+question)
REGI --16 ...                              // hex data input
```

### HIVE — Load/unload offline registry hive (comprehensive)
```
// Mount an offline hive to a mount point under HKLM (or HKU)
HIVE E:\Windows\System32\config\SOFTWARE,HKLM\PE-SYS     // load offline SOFTWARE hive
HIVE E:\Windows\System32\config\SYSTEM,HKLM\PE-SYS       // load offline SYSTEM hive
HIVE E:\Users\Default\NTUSER.DAT,HKU\PE-DEF             // load offline user hive
HIVE HKLM\PE-SYS,                                        // unload (empty path, same mount point)

// Load with security descriptor (preserve ACLs)
HIVE E:\...\SOFTWARE,HKLM\PE-SYS,ACL                    // load with security/ACL

// Load as temporary hive (changes discarded on unload)
HIVE -tmp E:\...\SOFTWARE,HKLM\PE-TMP                   // use temp hive (read-only intent)

// Load with restore on unload (save changes back to hive file)
HIVE -restore E:\...\SOFTWARE,HKLM\PE-SYS               // restore (write-back) on unload
HIVE -restore HKLM\PE-SYS,                               // unload with restore (saves changes)
```
Mount point syntax: `HKLM\PE-SYS` = mount the hive at `HKEY_LOCAL_MACHINE\PE-SYS`.
After mounting, access with `REGI .HKLM\PE-SYS\...` (dot prefix for offline registry).
To unload, provide the same mount point with an empty path (or `-restore` prefix to save).

---

## GUI CONTROLS

### ITEM — Button
```
ITEM [-font:N] [-def] [-round] [-na] [-right] [-center]
     Name,LxTyWwHh,Text,Command,[State],[Style]
```
`-def`=default button, `-round`=rounded, `-na`=no activate

### LABE — Static text
```
LABE [-vcenter] [-left|-center|-right] [-trans] [-3D] [-ncmd]
     Name,Shape,Text,[Command],[Color],[FontSize]
```
`-trans`=transparent, `-vcenter`=vertical center

### EDIT — Text input
```
EDIT [-3D] [-vcenter] [-rich] [-font:N]
     Name,LxTyWwHh,Text,[EventCmd],[Style],[FontSize]
```
Style: `0x224`=multiline+password, `0x10`=hidden, `0x400`=number only

### MEMO — Multi-line text
```
MEMO [-rich] Name,Shape,Text,[EventCmd],[Style],[FontSize]
```

### LIST — Dropdown
```
LIST [-h] Name,Shape,item1|item2|item3,[EventCmd],[Sel],[Style],[FontSize]
```
`-h`=always show dropdown height. Operations:
```
ENVI @LIST.VAL=                    // clear
ENVI @LIST.ADD=item                // add item
ENVI @LIST.ADDSEL=item             // add and select
ENVI @LIST.DEL=item                // delete item
ENVI @LIST.QUERY=;&all             // get all (NL-delimited)
ENVI @LIST.isel=N                  // select by index
```

### CHEK / RADI — Checkbox / Radio
```
CHEK [-right] Name,Shape,Text,[EventCmd],[State]
RADI [-right] Name,Shape,Text,[EventCmd],[State]
```
State: `1`=checked, `0`=unchecked, `-1`=toggle, `-2`=grayed

### TABL — Table / grid
```
TABL [-font:N] Name,Shape,Title,[Event],[Style]
```
Title format: `100:Name%&TAB%=90:Size%&TAB%+50:Count`
Column flags: default=left, `*`=left (default), `=`=right-align, `+`=center, `*0:`=hidden column
Style: `0x10040`=full row select+checkboxes, `0x10200`=full row select,
`0x4000`=grid lines, `0x16000`=single select+no header
Operations:
```
ENVI @TABL.Val=1*;%&data%            // bulk-set all rows
ENVI @TABL.Val=?row.col;&var         // get cell
ENVI @TABL.Val=?*;&count             // get row count
ENVI @TABL.Val=-*                    // clear all
ENVI @TABL.Sel=row                   // select row
ENVI @TABL.Sel=row;0                 // deselect
```

### Other controls
```
SWIN [-] [:]SubName,LxTyWwHh,[title],[style]   // sub-window (tab pages)
TABS Name,Shape,[Titles],[Style]               // tab control
GROU Name,Shape,Text,[Style]                   // group frame
PBAR Name,Shape,Value,[Style]                  // progress bar
IMAG Name,Shape,filePath|#resID,[Event],[Style] // image (BMP,JPG,GIF)
SLID Name,Shape,min:max,[EventCmd],[InitVal]    // slider
SPIN Name,Shape,min:max,[EventCmd],[InitVal]    // spin control
```

### MENU — Popup menu items
```
MENU ItemName,DisplayText,Command
MENU -                                        // separator
CALL @--popmenu MenuWindow [x.y[:align]]       // show popup
```

### TIPS — Tray icon / notification
```
// Create tray icon
TIPS* WindowName,iconFile,tooltip,leftClickHandler,rightClickHandler,timeout

// Params:
//   WindowName        - window to receive messages
//   iconFile          - icon path (or #resID for built-in)
//   tooltip           - hover tooltip text
//   leftClickHandler  - command on left-click (or 0 for none)
//   rightClickHandler - command on right-click (or 0 for none)
//   timeout           - display duration in ms (0=permanent)

// Examples
TIPS* MAINWIN,#1,My Tool,TEAM MESS Clicked!,0,0        // built-in icon, left-click only
TIPS* MAINWIN,%SystemRoot%\my.ico,Title,0,popmenuCmd,0  // right-click popup menu
TIPS* MAINWIN,#2,Hello World,CALL OnLeft,CALL OnRight,5000  // auto-dismiss after 5s

// Remove tray icon
TIPS.DEL=WindowName                                     // delete specific
TIPS.DEL=*                                              // delete all icons

// Multiple tray icons (different WindowName for each)
TIPS* Icon1,#1,Tool 1,cmd1,0,0
TIPS* Icon2,#2,Tool 2,cmd2,0,0

// Bubble / balloon notification
TIPS* MAINWIN,#1,Title\nMessage text,,,10000             // multi-line via \n
TIPS* MAINWIN,#3,Warning!\nDisk full!,CALL OnClick,,0   // with click handler

// Update existing tray icon
TIPS* WindowName,newIcon,newTooltip,newLeftCmd,newRightCmd,newTimeout
```

### Other GUI commands
```
BROW &result,[*|&]path,[prompt],[filter],[flags] // file/dir browser
MESS [text][+iconN] [\n...] [@title][#buttons][*ms][$default]  // message box
LOGO imagePath                                 // show/hide splash
TEXT text[#color][LxTy][RxBy][$size:font]     // display status text
HIDE                                           // hide PECMD.EXE process
WALL imagePath                                 // set desktop wallpaper
WALL -center imagePath                          // centered
WALL -tile imagePath                            // tiled
WALL -stretch imagePath                         // stretched
WALL -fit imagePath                             // proportional fit
WALL -fill imagePath                            // fill (crop to fit)
WALL -span imagePath                            // span across monitors
SEND {ENTER}                                   // send keystrokes
NUMK 1|0                                       // NumLock on/off
```

---

## NETWORK

### ADSL — Broadband/WiFi
```
ADSL userEncoded,passEncoded,[retries],[name|*|retVar]   // dial-up
ADSL stop|list[on],connectionName                        // stop/list
ADSL-wlan SSID|&var,password,encType,[index]              // WiFi connect (WPA2PSK default)
ADSL-wlan ,,list,&&result                                 // WiFi scan (basic)
ADSL-wlan ,,[?][^|*|-]list|query[all]|scan,&&result       // detailed scan/query
```

### PCIP — IP configuration
```
PCIP 192.168.1.100,255.255.255.0,192.168.1.1,[DNS1],[DNS2]
PCIP DHCP
```

### Other network
```
NTPC time.server.com                                // time sync
SITE ftp://user:pass@server/path,local,get|put        // FTP (download/upload)
UPNP add|del TCP|UDP,port,internalIP                  // port forwarding
```

---

## EXTERNAL EXECUTION

### EXEC — Execute program (comprehensive)
```
EXEC [=][!][@][^][&][*] [flags] program [args]
```
Basic: `=`=wait, `!`=hidden, `@`=no wait+hidden, `^`=don't wait, `*`=don't wait

Additional flags (most commonly used):
```
-err+                  // capture stderr into same output
-err                   // capture stderr separately
-cmd:::Callback        // real-time output line callback
-wd:path               // set working directory
-pid:var               // get process PID
-su[acde]              // run as SYSTEM
-doc:mode              // open/edit/print/properties document
-min|-max|-show|-hide  // window state
-user:name -passwd:pwd // run as user
-timeout:ms[:code]     // timeout
-waiti                 // wait for UI init
-raw                   // capture raw (no recoding)
-nowin                 // CREATE_NO_WINDOW
```

### EXEC* — Capture output
```
EXEC* [*1|*N|*-] [-catch] [-cmd:::Callback] [-err+] [&]outputVar=program [args]
```
`*1`=only first line, `*N`=merge lines, `*-`=trim trailing newline

### SOCK — Windows sockets / IPC
```
SOCK &sock                             // create socket
SOCK &sock connect host port            // connect TCP
SOCK &sock send data                    // send
SOCK &sock recv &var                    // receive
SOCK --pipe &pipeName                   // named pipe
SOCK --mailslot &slotName               // mailslot
SOCK --shm &varName                     // shared memory
SOCK --event &eventName                 // event object
SOCK --sem &semName                     // semaphore
SOCK --mutex &mutexName                 // mutex
```

### PINT — Pin to taskbar/start
```
PINT %Desktop%\Name.lnk,TaskBand         // pin to taskbar
PINT %Desktop%\Name.lnk,StartMenu        // pin to start menu
```

### LINK — Create shortcut
```
LINK %Desktop%\Name.lnk,target,[args],[icon],[iconIdx],[workDir]
LINK [?]Name.lnk                          // query shortcut info
```

---

## STRING MANIPULATION

### MSTR — Multi-string extraction
```
MSTR &a,&b=<1><3>%&data%              // extract fields 1 and 3
MSTR &rest=<5*>%&data%                 // fields 5 through end (use <N*> or <N->)
MSTR &restq=<~5>%&data%                 // field 5 with outer quotes stripped (~ = strip quotes)
MSTR &s=pos,len,%&str%                 // substring at position
MSTR &last=<-1>%&data%                 // last field (negative index)
MSTR -delims:. &a,&b,&c=<1*>%&ip%     // split by custom delimiter
```

### SED — Regex substitution
```
SED &r=count,pattern,replacement,%&source%
SED &r=0,pat,rep,%&s%                  // replace ALL
SED &r=1,pat,rep,%&s%                  // replace FIRST
SED &ext=-1,.*\.,,%&filename%          // get extension (negative=from end)
SED &r=0:0,pat,rep,%&s%                // regex mode
```

### Other string operations
```
LSTR &left=N,%&str%                     // first N chars
RSTR &right=N,%&str%                    // last N chars
SSTR &mid=M,N,%&str%                    // N chars from position M
LPOS &pos=needle,[1],%&haystack%        // find first (case-insensitive; add ,1, for case-sensitive)
RPOS &pos=needle,[1],%&haystack%        // find last
STRL &len=%&str%                        // string length
RAND &var                               // random 63-bit integer
```

---

## OTHER COMMANDS

### TIME / DTIM
```
TIME &var                               // get current time
DTIM &ts,&date,&time                    // combine to timestamp
DTIM &dateStr,&ts                       // timestamp to string
```
> Note: See SYSTEM section for the comprehensive DATE command.

### BASE — Base64
```
BASE string,&var                         // PECMD variant (for ADSL security)
BASE* string,&var                        // standard base64
BASE* -u string,&var                     // standard decode
```

### CMPS — Compression
```
CMPS -m source.wcs,dest.wcz             // compress
CMPS -u source.wcz,dest.wcs             // decompress
```

### WAIT — Pause / key wait
```
WAIT ms                                 // pause milliseconds
WAIT -1                                 // wait forever
WAIT -cont [-timeout],[&var]            // non-blocking key press wait
```

### KILL — Terminate
```
KILL process.exe                        // by name
KILL *12345                             // by PID
KILL \                                  // current script's windows
KILL \WinName                           // specific window
```

### LOGS — Debug logging
```
LOGS * C:\log.txt                       // start logging
LOGS                                    // stop
```

### COME / NOTE — Comment toggle
```
COME 0 / NOTE OFF                       // disable comments
COME 1 / NOTE ON                        // enable comments
```

### HELP
```
HELP                                    // show full help
HELP commandName                        // show command-specific help
```

### TIME — Timer control
```
TIME TimerName,interval,command          // periodic timer
TIME -t:1 TimerName,interval,command     // one-shot
ENVI @TimerName=0                        // stop
ENVI @TimerName=interval;count           // run N times
ENVI @TimerName=-del                     // destroy
```

### IPAD — IP address display control
```
IPAD Name,LxTyWwHh,IP,[Style]
```

### ENVI ? — System queries
```
ENVI ?WinPE=ispe                          // check if running in WinPE
ENVI ?FVER &ver,path\to\file.dll          // get file version
ENVI ?ReturnValue=FVAR,varName;{GUID}     // query firmware variable (UEFI)
```

---

## Appendix: Virtual Key Codes

Common Windows virtual key codes used with `HKEY`, `HOTK`, `WAIT -cont`, and `SEND`:

| Key Name | Decimal | Hex | Description |
|---|---|---|---|
| VK_LBUTTON | 1 | 0x01 | Left mouse button |
| VK_RBUTTON | 2 | 0x02 | Right mouse button |
| VK_CANCEL | 3 | 0x03 | Ctrl+Break |
| VK_MBUTTON | 4 | 0x04 | Middle mouse button |
| VK_BACK | 8 | 0x08 | Backspace |
| VK_TAB | 9 | 0x09 | Tab |
| VK_RETURN | 13 | 0x0D | Enter |
| VK_SHIFT | 16 | 0x10 | Shift |
| VK_CONTROL | 17 | 0x11 | Ctrl |
| VK_MENU | 18 | 0x12 | Alt |
| VK_PAUSE | 19 | 0x13 | Pause/Break |
| VK_CAPITAL | 20 | 0x14 | Caps Lock |
| VK_ESCAPE | 27 | 0x1B | Esc |
| VK_SPACE | 32 | 0x20 | Spacebar |
| VK_PRIOR | 33 | 0x21 | Page Up |
| VK_NEXT | 34 | 0x22 | Page Down |
| VK_END | 35 | 0x23 | End |
| VK_HOME | 36 | 0x24 | Home |
| VK_LEFT | 37 | 0x25 | Left Arrow |
| VK_UP | 38 | 0x26 | Up Arrow |
| VK_RIGHT | 39 | 0x27 | Right Arrow |
| VK_DOWN | 40 | 0x28 | Down Arrow |
| VK_INSERT | 45 | 0x2D | Insert |
| VK_DELETE | 46 | 0x2E | Delete |
| VK_0 | 48 | 0x30 | 0 |
| VK_1 | 49 | 0x31 | 1 |
| VK_2 | 50 | 0x32 | 2 |
| VK_3 | 51 | 0x33 | 3 |
| VK_4 | 52 | 0x34 | 4 |
| VK_5 | 53 | 0x35 | 5 |
| VK_6 | 54 | 0x36 | 6 |
| VK_7 | 55 | 0x37 | 7 |
| VK_8 | 56 | 0x38 | 8 |
| VK_9 | 57 | 0x39 | 9 |
| VK_A | 65 | 0x41 | A |
| VK_B | 66 | 0x42 | B |
| VK_C | 67 | 0x43 | C |
| VK_D | 68 | 0x44 | D |
| VK_E | 69 | 0x45 | E |
| VK_F | 70 | 0x46 | F |
| VK_G | 71 | 0x47 | G |
| VK_H | 72 | 0x48 | H |
| VK_I | 73 | 0x49 | I |
| VK_J | 74 | 0x4A | J |
| VK_K | 75 | 0x4B | K |
| VK_L | 76 | 0x4C | L |
| VK_M | 77 | 0x4D | M |
| VK_N | 78 | 0x4E | N |
| VK_O | 79 | 0x4F | O |
| VK_P | 80 | 0x50 | P |
| VK_Q | 81 | 0x51 | Q |
| VK_R | 82 | 0x52 | R |
| VK_S | 83 | 0x53 | S |
| VK_T | 84 | 0x54 | T |
| VK_U | 85 | 0x55 | U |
| VK_V | 86 | 0x56 | V |
| VK_W | 87 | 0x57 | W |
| VK_X | 88 | 0x58 | X |
| VK_Y | 89 | 0x59 | Y |
| VK_Z | 90 | 0x5A | Z |
| VK_LWIN | 91 | 0x5B | Left Windows key |
| VK_RWIN | 92 | 0x5C | Right Windows key |
| VK_APPS | 93 | 0x5D | Application/Menu key |
| VK_NUMPAD0 | 96 | 0x60 | Numpad 0 |
| VK_NUMPAD1 | 97 | 0x61 | Numpad 1 |
| VK_NUMPAD2 | 98 | 0x62 | Numpad 2 |
| VK_NUMPAD3 | 99 | 0x63 | Numpad 3 |
| VK_NUMPAD4 | 100 | 0x64 | Numpad 4 |
| VK_NUMPAD5 | 101 | 0x65 | Numpad 5 |
| VK_NUMPAD6 | 102 | 0x66 | Numpad 6 |
| VK_NUMPAD7 | 103 | 0x67 | Numpad 7 |
| VK_NUMPAD8 | 104 | 0x68 | Numpad 8 |
| VK_NUMPAD9 | 105 | 0x69 | Numpad 9 |
| VK_MULTIPLY | 106 | 0x6A | Numpad * |
| VK_ADD | 107 | 0x6B | Numpad + |
| VK_SUBTRACT | 109 | 0x6D | Numpad - |
| VK_DECIMAL | 110 | 0x6E | Numpad . |
| VK_DIVIDE | 111 | 0x6F | Numpad / |
| VK_F1 | 112 | 0x70 | F1 |
| VK_F2 | 113 | 0x71 | F2 |
| VK_F3 | 114 | 0x72 | F3 |
| VK_F4 | 115 | 0x73 | F4 |
| VK_F5 | 116 | 0x74 | F5 |
| VK_F6 | 117 | 0x75 | F6 |
| VK_F7 | 118 | 0x76 | F7 |
| VK_F8 | 119 | 0x77 | F8 |
| VK_F9 | 120 | 0x78 | F9 |
| VK_F10 | 121 | 0x79 | F10 |
| VK_F11 | 122 | 0x7A | F11 |
| VK_F12 | 123 | 0x7B | F12 |
| VK_NUMLOCK | 144 | 0x90 | Num Lock |
| VK_SCROLL | 145 | 0x91 | Scroll Lock |
| VK_LSHIFT | 160 | 0xA0 | Left Shift |
| VK_RSHIFT | 161 | 0xA1 | Right Shift |
| VK_LCONTROL | 162 | 0xA2 | Left Ctrl |
| VK_RCONTROL | 163 | 0xA3 | Right Ctrl |
| VK_LMENU | 164 | 0xA4 | Left Alt |
| VK_RMENU | 165 | 0xA5 | Right Alt |
| VK_BROWSER_BACK | 166 | 0xA6 | Browser Back |
| VK_BROWSER_FORWARD | 167 | 0xA7 | Browser Forward |
| VK_VOLUME_MUTE | 173 | 0xAD | Volume Mute |
| VK_VOLUME_DOWN | 174 | 0xAE | Volume Down |
| VK_VOLUME_UP | 175 | 0xAF | Volume Up |
| VK_MEDIA_NEXT_TRACK | 176 | 0xB0 | Media Next Track |
| VK_MEDIA_PREV_TRACK | 177 | 0xB1 | Media Previous Track |
| VK_MEDIA_STOP | 178 | 0xB2 | Media Stop |
| VK_MEDIA_PLAY_PAUSE | 179 | 0xB3 | Media Play/Pause |
| VK_OEM_1 | 186 | 0xBA | ;: key (US) |
| VK_OEM_PLUS | 187 | 0xBB | =+ key (US) |
| VK_OEM_COMMA | 188 | 0xBC | , key |
| VK_OEM_MINUS | 189 | 0xBD | -_ key (US) |
| VK_OEM_PERIOD | 190 | 0xBE | . key |
| VK_OEM_2 | 191 | 0xBF | /? key |
| VK_OEM_3 | 192 | 0xC0 | `~ key |
| VK_OEM_4 | 219 | 0xDB | [{ key |
| VK_OEM_5 | 220 | 0xDC | \| key |
| VK_OEM_6 | 221 | 0xDD | ]} key |
| VK_OEM_7 | 222 | 0xDE | '" key |

Usage with PECMD commands:
```
HKEY Alt+#0x41,TEAM MESS Pressed Alt+A!
HOTK Ctrl+Win+#0x53,MYFUNC         // Ctrl+Win+S
WAIT -cont -3000,&key               // wait 3s for key, store code in &key
SEND #0x0D                          // send Enter key
```
