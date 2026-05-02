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
ENVI @Ctrl.ID=?&hwnd               // get control HWND
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
FIND *=var, command              // IDIOM: var is empty (primary source pattern)
FIND *<>var, command             // IDIOM: var is NOT empty
FIND $str1=str2,! cmd1! cmd2   // if else (separated by !)
FIND $str1=str2,!! command      // else only
FIND |num1>num2, command        // numeric comparison
FIND $'%var%'='', command       // safe empty check (single quotes)
FIND [$][A & B], command         // compound AND (& between conditions)
FIND [$][A | B], command         // compound OR (| between conditions)
FIND --pid &var,                // get process/CPU ticks
FIND --pid*@[.ext|#parentPID] &var,  // process list (opt: extension filter or parent PID)
FIND --wid*@[parentWID] &var,[title] // window list (opt: parent window filter)
FIND --class:ClassName --wid*@ &var   // window list filtered by window class
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
FORX * list,&&item,                              // space-delimited iteration
FORX *NL &multiLine,&&line,                      // newline-delimited
FORX *v &a &b &c,&&name,                          // iterate variable names
FORX /S[:depth] path\*.ext,&&name,0               // file enumeration (0=files, 1=dirs)
FORX /S:3 /O:N path\*.ext,&&f,0                  // max depth 3, sorted by name
FORX /S /O:-N path\*.ext,&&f,0                   // depth unlimited, reverse sort
FORX /S /size:0:1048576:512 path\*.ext,&&f,0     // size 0-1MB, 512-aligned
FORX @\Windows,&&dir,1                            // search for directory root
FORX !\*.ext,&&f,0                                // reverse directory order
FORX @\*.ext,&&d,1                                // dirs only (@ prefix)
FORX *ab \*.ext,&&f,0                             // exclude A/B removable drives
FORX *cur \*.ext,&&f,0                            // current drive preferred in search
FORX *qu[~] \*.ext,&&name,0                       // support quoting in paths
FORX *off \*.ext,&&name,0                         // return changed portion only
FORX *bf \*.ext,&&name,0                          // breadth-first directory search
FORX *L start step end,&&val,                     // numeric loop: FORX *L 0 2 10,&&val,
FORX . \*.ext,&&f,0                               // ; can replace , as separator
FORX : \*.ext,&&f,0                               // : can replace , as separator
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
// List/info operations
PART list disk,&var                         // list all disk numbers
PART list disk N,&var                       // disk info (size, cylinders, heads, media type, signature, bus, type, removable)
PART list part N,&var                       // list partition numbers on disk N
PART -hextp list part N#M,&var              // partition info (hex type 0xNN)
PART -hextp -phy list part N#M,&var         // partition info with physical numbering (1-4 primary, 5-N logical)
PART -hextp -phy# list part N#M,&var        // partition info + physical# field appended
PART -fill list part N#M,&var               // use * placeholder for empty drive letter
PART -devid list disk N,&var                // device path/ID (like \\.\PHYSICALDRIVE0)
PART -devidx list disk N,&var               // model + serial
PART -devidn list disk N,&var               // name only
PART -devida list disk N,&var               // full: product# + serial + version + DeviceType + RemovableMedia + CommandQueueing + VendorId + ProductRevision
PART -iv=N list disk N,&var                 // query sub-field N of disk info
PART -raw list disk N,&var                  // raw disk info (device path, media GUID, volume name)
PART list drv D:,&var                       // drive letter → disk# partition# type bus drive media
PART list volume volumeName,&var            // volume info
PART -drv list volume N,&var                // volumes by drive number
PART -report[:retvar][diskNum]              // show/list report (ignores other args)
PART -floppy list disk N,&var               // list floppy devices

// Modify operations
PART -super -up -xup N#M type [attr]        // set partition type+attribute (both -super -up required)
PART -super -up -axup N#M type [attr]       // enhanced xupdate for removable disks
PART -super -up -swap:M N#P                 // swap physical partition numbers
PART -super -up N#M a|A|-a|-A type start len// create (a=active, A=extended, -a=inactive)
PART -super -up del N#M                     // delete partition
PART -super -up N#M a|A|-a|-A               // toggle active on existing
PART -super -up -fs0 N#M init               // initialize as raw (no filesystem)
PART -super -up -force N#M ...              // force dangerous operation
PART update N                               // refresh disk info from system
PART hupdate[f] N                           // hard disk refresh (f=force with renumber)
PART -ahup -up N#M ...                      // additional hard update for removable renumber

// MBR/PBR operations
PART /mbr[=nt6|=win|=nt5|=dos|=file] N      // rewrite MBR on disk N
PART /pbr[=nt6|=win|=nt5|=dos|=file] N#M    // rewrite PBR on partition
PART -img=[*offs*len*]file|disk[/mbr|/pbr]  // operate on image file instead of physical disk

// GPT operations
PART -gpt init N                            // initialize as GPT
PART -super -up -gpt N#M a type start len guid attr name  // create GPT partition
PART -super -up -gpt -fs0 -mbr init N       // init GPT+MBR hybrid, raw FS
PART -gpt -cmp N                            // compress GPT table (make 1-based, contiguous)
PART fix N                                  // fix GPT: correct checksums, flags, partition count

// Smart drive letter control
PART -lock[:\\\\.\D:] N                     // lock drive letter (prevent auto-assign)
PART -locku[:\\\\.\D:] N                    // unlock
PART -lock *                                // lock all volumes
PART -dvol N#M,&volGUID                     // dynamic corrected VolumeGUID
PART -mount-                                // don't show labels for unassigned partitions
PART -fill                                  // fill empty drive letter slots

// Utility
PART -gui                                   // launch GUI partition manager
PART -usb                                   // USB-only mode
PART -admin                                 // advanced mode (dangerous)
PART -align[=size]                          // alignment (default or specify)
PART -CHS=C:H:S                             // override cylinder/head/sector geometry
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
| DRIVE_CDROMUSB | 7 | USB optical disc |
| DRIVE_USBFLASH | 8+ | USB flash drive |
| DRIVE_USBDISK | 9+ | USB disk |
| FUNCTION_ERROR | -1 | API error |

### Bus Type Constants (FORM -raw &busType,&bus,drvType=&id,D:)
| Constant | Value | Description |
|---|---|---|
| BusTypeUnknown | 0x00 | Unknown |
| BusTypeScsi | 0x01 | SCSI |
| BusTypeAtapi | 0x02 | ATAPI |
| BusTypeAta | 0x03 | ATA |
| BusType1394 | 0x04 | 1394 (FireWire) |
| BusTypeSsa | 0x05 | SSA |
| BusTypeFibre | 0x06 | Fibre Channel |
| BusTypeUsb | 0x07 | USB |
| BusTypeRAID | 0x08 | RAID |
| BusTypeiScsi | 0x09 | iSCSI |
| BusTypeSas | 0x0A | SAS |
| BusTypeSata | 0x0B | SATA |
| BusTypeSd | 0x0C | SD |
| BusTypeMmc | 0x0D | MMC |
| BusTypeVirtual | 0x0E | Virtual |
| BusTypeFileBackedVirtual | 0x0F | File-backed Virtual |
| BusTypeSpaces | 0x10 | Storage Spaces |
| BusTypeNvme | 0x11 | NVMe |
| BusTypeSCM | 0x12 | SCM |
| BusTypeUfs | 0x13 | UFS |
| BusTypeMax | 0x14 | |
| BusTypeMaxReserved | 0x7F | Reserved max |

### DFMT — Format
```
DFMT d:,NTFS,label,quick
DFMT d:,FAT32,,quick
```

### EJEC — Eject
```
EJEC D:                       // eject optical drive D:
EJEC * D:                     // eject removable USB disk D:
EJEC C-                       // close all optical drive trays
EJEC U-                       // eject all USB disks
EJEC C- X:                    // close tray on X:
EJEC U- X:                    // eject USB disk X:
EJEC C- HDD#1                 // close tray on disk 1
EJEC U- HDD#1                 // eject USB disk on disk 1
```

### DISK — Disk operations
```
DISK [varName],[diskNum],[partNum],function,[USBDriveLetters][,options]
  // function 1=allocate, 2=free, 3=reallocate, 22=first primary partition
  // USBDriveLetters: e.g. "UW" means start from W: for USB (drive letter table)
```
| Function | Description |
|---|---|
| 1 | Allocate drive letters |
| 2 | Free drive letters |
| 3 | Reallocate (rearrange + allocate) |
| 22 | First primary partition allocation |
| **varName special forms:** | |
| `&drvLetter,diskNum,partNum` | Get drive letter of specific partition |
| `uAllPart,diskNum,partNum` | Assign USB → new partition letter |
| `Vol:volLabel,diskNum,partNum` | Find by volume label |
| `Part:partName,diskNum,partNum` | Find by partition name |
| `\Windows\|\WinXP\|\WinNT\|` | Search system dirs across drives |
| **Options (0x**):** ||
| 0x1 | Only rearrange already-drive-lettered partitions |
| 0x2 | Verify partition validity |
| 0x4 | Skip 0xEE/0xEF partitions |
| 0x10 | Hidden partitions too |
| 0x20 | CDROM too |
| 0x40 | Limit drive letter table |
| **Flags:** ||
| `-check` | Skip if already loaded |
| `-skiptp:tp1;tp2` | Skip partition types |
| `-skippt:hd:pt` | Skip specific harddisk:partition |
| `-from:D:` | Start drive letter from D: |
| `-from:UW` | USB drive letter table "UW" |
| `-cdrom` | Include CDROM |
```

---

## MOUNT (WIM / VHD / UDM)

### WIM mounting
```
MOUN[-svr] [!] wimFile,mountDir,[imageID],[tempDir]    // mount (read-write)
MOUN[-svr] -w [!] wimFile,mountDir,[imageID],[tempDir]  // mount writable
MOUN[-svr] -m [!] wimFile,mountDir,[imageID],[tempDir]  // mount read-only (no -w)
MOUN -u mountDir                                         // unmount
MOUN -query &var                                        // query mounted images
MOUN[-svr] -u [!] wimFile,mountDir,[imageID],[tempDir]   // unmount with commit
MOUN[-svr] -rw [!] wimFile,mountDir,...                  // mount read-write (alias)
```
Option: `-dll WIMDLLpath:` to specify wimgapi.dll location.

### VHD/VHDX mounting
```
MOUN-vhd -c[x] file.vhd,size                            // create (x=expand to size first)
MOUN-vhd -c[x] -d file.vhd,size                         // create dynamic (sparse)
MOUN-vhd -c[x] -s:512 file.vhd,size                     // sector size override
MOUN-vhd -r file.vhd,mountDir                           // mount read-only
MOUN-vhd -d file.vhd,mountDir                           // mount dynamic VHD
MOUN-vhd -u mountDir                                    // unmount
MOUN-vhd -iso file.iso,mountDir                         // mount ISO
MOUN-vhd -query file.vhd,&var                          // query VHD info
```
PECMD-private PE var: when var goes out of scope, auto-unmount. Use `PEvar` as 3rd parameter.

### UDM (Ultra Deep Mount) — hidden partition mounting
```
MOUN-udm [flags] \\\\.PhysicalDriveN                     // mount all or specific
MOUN-udm -findboot -ret:&retVar                         // find and mount boot device
MOUN-udm -u mountDir                                    // unmount
```
| Flag | Description |
|---|---|
| `-ud` | UD partition |
| `-uh` | UD high |
| `-muh` | Mount UD high |
| `-u+` | U+ partition |
| `-udfs` | UD filesystem |
| `-udm-` | Disable UDM |
| `-mall` | Mount ALL (not just hidden) |
| `-mhide` | Mount hidden only |
| `-mhide1` | Mount hidden only (variant 1) |
| `-onlys` | Only mount specific system types |
| `-findboot` | Auto-find boot device |
| `-ret:` | Return device path to variable |
| `-CheckFile[+]:path` | Verify by file existence |
| `-CheckVol[R]` | Verify by volume label |
| `-CheckUuid[R]` | Verify by UUID |
| `-CheckPtType` | Verify by partition type |
| `-check[-]` | Only mount valid filesystem partitions |
| `-tag[+]:name` | Tag identification for matching |
| `-opts:`/`-opt:` | Mount options (separate or combined) |
| `-nbrd[-]` | Don't broadcast drive letter |
| `-ainf:var` | Store partition table buffer to variable |
| `-udmid:pt#physicalNum` | Soft mount by physical partition number (read-only default) |
| `-udmdev:device` | Specify boot device and UDM |

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
SHUT                              // shutdown
SHUT R                            // restart
SHUT S                            // suspend/standby
SHUT H                            // hibernate
SHUT L                            // logoff
SHUT K                            // lock workstation
SHUT E                            // eject optical drive
SHUT C                            // close optical drive
SHUT O                            // eject optical + wait 10s
SHUT -force R                     // force restart
SHUT -- [scriptFile]              // run script on shutdown
SHUTDOWN -s|-r|-f|-t 秒           // pass raw args to shutdown.exe
  // -s=shutdown -r=reboot -f=force --f=cancel force -t=delay
```
// OnShutdown.wcs hook 操作码: shutdown reboot logout suspend hiber poweroff unknown lock

### DISP — Display settings
```
DISP W1024H768B32F60                             // width, height, color bits, refresh rate
DISP                                             // auto-detect best mode
DISP =N W1024H768B32F60                         // target display N (0-based)
DISP W1024H768B32F60 T15                         // apply with 15s timeout (auto-restore)
DISP W1024H768B32F60 P                           // set as primary display
DISP W1024H768B32F60 O0                          // orientation (0=default, 1=90, 2=180, 3=270)
DISP -confirm W1024H768B32F60                    // confirmation prompt
DISP -nwb W1024H768B32F60                        // no broadcast wait
DISP -delay W1024H768B32F60                      // registry-only (don't apply), wait for broadcast
DISP @X0:Y0:X1:Y1:...                            // multi-monitor positions (matrix)
DISP S0x84                                        // multi-display: 0x81=single, 0x82=clone, 0x84=extend, 0x88=dual
DISP ?[?*] [=N] &var                             // query current (*=all possible) modes
DISP -reset                                       // reset to defaults
DISP -bright[?]:value/&var                        // brightness control
DISP -ori [?] &var                                // query orientation
DISP -guis                                        // graphical interface
DISP -sort[-r|-n]                                 // sort modes (r=reverse, n=by name)
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
SERV servicename                             // start service
SERV !servicename                            // stop service (! prefix)
SERV ?servicename,&var                       // query status
SERV -create name,path,type,start            // create service
SERV -delete [-stop-] name                   // delete (-stop-=auto-stop before delete)
```
Start types: `-boot`, `-system`, `-auto`, `-demand`, `-disabled`, `-delayed-auto`

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
DATE &var                                  // get current date (yyyy mm dd HH MM SS ms weekday format)
DATE &var yyyy-mm-dd-HH-MM-SS-ms-wd        // set system date/time (partial ok)
DATE -h &var                               // high-precision timer (microseconds)
DATE -r &var                               // sync + read high-precision timer
DATE -space0 &var                          // space-delimited, 0-padded
DATE -space &var                           // space-delimited (default compact)
DATE -bsys &var                            // output system time
DATE -utc:UTCtime &var                     // convert FROM UTC time
DATE -gmt:GMTtime &var                     // convert FROM GMT time
DATE -local:LOCALtime &var                 // convert FROM local time
DATE -sys:internTime &var                  // international/UTC time
DATE -us &var                              // microseconds (4 decimal places)
```
Sub-items (use `MSTR` or direct `%&var:item%` syntax):
| Sub-item | Meaning |
|---|---|
| `y` / `year` | Year (4-digit) |
| `mon` / `month` | Month (2-digit) |
| `d` / `day` | Day (2-digit) |
| `w` / `weekday` | Day of week (1=Mon..7=Sun) |
| `h` / `hour` | Hour (24h, 2-digit) |
| `min` / `minute` | Minute (2-digit) |
| `s` / `second` | Second (2-digit) |
| `ms` / `msec` | Milliseconds (3-digit) |
| `ws[1]` | Week of year ([1]=Sunday as weekend boundary) |
| `ds` / `daysofyear` | Day of year (1-366) |
| `Freq` / `frequency` | Counter frequency |
| `Counter` / `counter` | Hardware timer counter value |
| `gmt` | Seconds since 1970-01-01 |
| `uptime` / `uptime_ms` | Milliseconds since boot |
| `utc` | 100ns units since 1601-01-01 |
| `uptimens` | Nanoseconds since boot |

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

// File version query
SITE ?fileVerVar[,prodVerVar]=FVER,filePath  // query file version (e.g. "1.2.3.4")

// File time query
SITE ?[-local -ws -link] [[*]creationVar,[*]writeVar,[*]accessVar]=FTIME,filePath
  // * prefix → returns UTC time integer (directly comparable)
  // no * → returns "yyyy mm dd HH MM SS us weekday" (fixed-width fields)
  // -local → local time (default: UTC)
  // -ws → append week-of-year; -ws1 → Sunday as weekend boundary
  // -link → follow symbolic links

// File attribute query
SITE ?[attrVar][,hidVar][,roVar][,sysVar][,fullVar]=FATTR,filePath

// Update file timestamp
SITE *touch[:[cr][*local:|*local0:|*sys:|*sys0:|*utc:]time],<file>[,retVar]
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

// Advanced operations
REGI --ak HKCU\Software\Key\,&all             // enumerate ALL values for key
REGI --av HKCU\Software\Key\,&all             // enumerate ALL subkeys  
REGI .?\HKLM\SOFTWARE\Key\Val,&type           // query value type (dot+question)
REGI --16 ...                                 // hex data input
REGI --su path\val=value                      // run elevated (SYSTEM) — for 32bit on 64bit
REGI --init path\val,&var                     // return empty string on read failure
REGI --name path\val,&var                     // data variable name mode
REGI --k path\key\                            // only create key (don't set value)
REGI --byte path\val,&var                     // byte stream mode
REGI --v[-] path\val,&var                     // don't save changes (read snapshot)
REGI --qk path\val,&var                       // quick mode
REGI --r10 path\val,&var                      // output decimal (for DWORD)
REGI --t:NUM path\val,&var                    // specify arbitrary registry type by number
REGI --0[:N] path\key\                        // clear key: 1=clear default, 2=delete subkeys, 4=delete values (combine: 5=1+4)

// Query existence (returns ERROR if not found)
REGI ?HKLM\SOFTWARE\Key\,&&VT
FIND $%&VT%=ERROR, MESS Key not found! MESS Key exists
REGI ?HKLM\SOFTWARE\Key\Val,&&VT           // check value existence
REGI ?HKLM\SOFTWARE\Key\,&&VT               // check key existence
FIND $%&VT%=NI, MESS Data not set!          // NI = key exists but no data
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
ENVI @LIST.QUERY=row;&line         // get specific row
ENVI @LIST.isel=N                  // select by index
ENVI @LIST.ADD1=item1|item2        // bulk-add (pipe-delimited)
ENVI @LIST.VAL=item1|item2         // reset and add all
ENVI @LIST.VAL=:+item              // insert at top
ENVI @LIST.VAL=:-item              // insert at bottom
ENVI @LIST.VAL=:=item              // replace current selection
```

### CHEK / RADI — Checkbox / Radio
```
CHEK [-right] [-center] [-scale[:[res][:icon]]] Name,Shape,Text,[EventCmd],[State]
RADI [-right] Name,Shape,Text,[EventCmd],[State]
```
State: `1`|`-1`=checked, `0`|`2`|`-2`=unchecked, `<0`=grayed, `±16`=invisible

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

### TIPS — Tray icon / bubble notification
```
// Tray icon (with * suffix: private tray control, requires window)
TIPS* WindowName,[Content],[timeout],[iconStyleID],[trayIcon],[#WID]

// Bubble notification
TIPS Title,Content,[timeout],[iconStyleID],[@[A]LxTy]
TIPS# Title,#Content,#timeout,[iconStyleID],[@[A]LxTy]

// Params:
//   WindowName  - window name for private tray (* suffix)
//   Title       - bubble title / tray label (max 64 chars)
//   Content     - body text (max 256 chars), \n for line breaks
//   timeout     - display ms (default 10s, 0=permanent)
//   iconStyleID - 0=none, 1=info, 2=warning, 3=error, 4+=tray icon
//   trayIcon    - icon file or #resID (shell32.dll#94 etc.)
//   @[A]LxTy    - screen position (@=rectangular, @A=arrow style)
//   #WID        - window handle for association
```

Examples:
```wcs
// Tray icon with WM_TRAYNOTIFY message for click handling
SET &WM_TRAYNOTIFY=1109
CALL @WinMain
_SUB WinMain,#
    ENVI @this.MSG=_%&WM_TRAYNOTIFY%::wp,lp,CALL DoTrayClick %wp% %lp%
    TIPS* WinMain,MyTool,,,shell32.dll#94
_END
_SUB DoTrayClick
    IFEX $0x0204=%2, CALL @--popmenu MyMenu    // right-click → popup
_END

// Bubble notification
TIPS MyTitle,Hello World\nLine 2,5000,1          // info icon, 5 seconds
TIPS* WinMain,Status update,,2,#1                 // warning icon, built-in res#1

// Clear
TIPS -                                            // clear bubble
TIPS *                                            // clear all tray icons + bubble
```

### Other GUI commands
```
BROW &result,[*|&]path,[prompt],[filter],[flags] // file/dir browser
MESS [text][+iconN] [\n...] [@title][#buttons][*ms][$default]  // message box
// iconN: +6=info, +32=question, +16=error, +48=warning, +0=none
// #buttons: #OK=OK, #YN=YesNo, #YNC=YesNoCancel, #YNCD=with default, #IC=IgnoreCancel
// *ms: auto-close timeout in ms. $N/$Y: default button (No/Yes)
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
// Dial-up (PPPoE)
ADSL userEncoded,passEncoded,[retries],[name|*|retVar]     // dial-up
ADSL start[+] userEncoded,passEncoded,[retries],[retVar]    // start (connect)
ADSL stop,connectionName                                    // hang up
ADSL list[on],connectionName                                // list connections

// WiFi (ADSL-wlan)
ADSL-wlan SSID|&profileVar,password,encType,[index]        // connect (enc default=WPA2PSK AES)
ADSL-wlan -start SSID|&profileVar,password,encType,[index]  // explicit start
ADSL-wlan index,,list,&&result                              // list WiFi profiles
ADSL-wlan index,,query[all],&&result                        // query details (序号 guid State Desc)
ADSL-wlan index,,scan,&&result                              // scan networks
ADSL-wlan index,,-list,&&result                             // net-broadcast scan
// Result format (list): SSID SignalQuality Flags BssType NumBssid bConnectable ...
// Result format (query[all]): index guid State Description
// Flags & 1 = currently connected
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
-incmd                  // run command in a fresh PECMD instance (no message loop)
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
MSTR* &a,&b=<1><2>%&data%               // TAB-delimited (prefix * on command)
MSTR$ &a,&b=<1><2>%&data%               // space-delimited, consecutive spaces = single
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
CMPS -m source.wcs,dest.wcz             // compress (encrypted)
CMPS -m -u source.wcz,dest.wcs           // decompress
CMPS -f -m source.wcs,dest.wcz           // compress (no encryption, -m after -f)
CMPS -bin source.exe,dest.wcz            // compress binary (NOT script)
CMPS -src[:flags] source.wcs,dest.wcz    // source compression with clean flags:
  // -src:1 = remove comment lines
  // -src:2 = convert line endings
  // -src:4 = compress empty lines
  // -src:8 = remove inline comments
  // Combine: -src:15 = all of above
CMPS -utf8 source.wcs,dest.wcz           // encode as UTF-8
```

### WAIT — Pause / key wait
```
WAIT ms                                    // pause milliseconds
WAIT -1                                    // wait forever
WAIT -cont [-timeout],[&var]              // non-blocking key press wait
WAIT *pid|*tid                             // wait for process/thread completion
WAIT **                                     // wait for grandparent process
WAIT =tid                                   // wait for specific thread ID
WAIT -del file1 [-del file2]               // delete files after wait (with retry)
WAIT -delms:N                               // delay between delete retries (ms)
WAIT -scanall|scan:key,&var                // get keyboard scan state table
WAIT -sys[0] [switch] -cmd                 // system proxy agent execution
WAIT -sys[0]cmd                            // system direct agent execution
WAIT -thread                                // wait for all child threads
WAIT $handle                                // wait for handle
WAIT -freemem                               // free memory
WAIT -pad                                   // distinguish numpad keys
WAIT -ncd                                   // don't change directory during wait
WAIT &&PressKey.Hex                         // sub-var for hex key code
WAIT time1 time2                            // time1>0&<1: *100000 = pending message count
```

### KILL — Terminate
```
KILL process.exe                           // by name
KILL *12345                                // by PID
KILL \                                     // current script's windows
KILL \WinName                              // specific window title
KILL process.exe|username                   // by name + owner
KILL \[windowTitle]                        // by window title
KILL @[windowName]                         // by window class name
KILL @@windowID                            // by window ID
KILL **tid                                 // by thread ID (async kill)
KILL *&hpid                                // by process HANDLE
KILL **&htid                               // by thread HANDLE (async)
KILL -force process.exe                    // force terminate
KILL -explorer process.exe                 // prevent explorer auto-restart
KILL -gui                                  // process manager GUI
KILL -tree process.exe                     // terminate process tree
KILL -svr2                                 // for MESS-svr2
KILL -exitcode:NUM process.exe             // set exit code
KILL ** process.exe                        // force synchronous kill
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
