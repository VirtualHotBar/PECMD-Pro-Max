---
name: pecmd-pro-max
description: |
  PECMD2012 scripting for WinPE — lightweight Windows GUIs, system
  tools, boot/init scripts, and automation. Use for .wcs/.wci/.wce files,
  disk partitioning, batch-to-PECMD conversion, system info collectors,
  PE/pre-install environment utilities, and any PECMD code debugging.
  References: commands-full.md, codebook.md, pe-startup.md, pecmd-gui.md.
compatibility: Requires PECMD2012 v1.88+ interpreter. Scripts are written for the PECMD/WinCMD runtime, not for cmd.exe.
---

# PECMD Pro Max

You are an expert PECMD programmer. PECMD is a WinPE command interpreter and scripting language evolved from XCMD V2.2 — think of it as a domain-specific language for Windows PE system administration and lightweight GUI tools.

## $1 MENTAL MODEL

PECMD has two execution modes:
- **Command-line mode**: `PECMD.EXE ENVI $PPPoE=OK` — single command, comments OFF by default
- **Script mode**: `PECMD.EXE LOAD C:\PECMD.INI` — multi-command file, comments ON by default

A PECMD script is a flat list of top-level statements. Execution flows top-to-bottom. `_SUB` blocks are declared at parse time and only run when CALLed. All GUI is created by `_SUB` blocks defining window functions (called via `CALL @WindowName`).

Key runtime facts:
- `PECMD.EXE MAIN path\to\PECMD.INI` — the standard WinPE entry point, executes the INI and also starts the message loop
- You can run PECMD inside itself: `EXEC =!"%MyNAME%" <command>` — useful for isolated sub-operations
- `EXIT FILE` terminates the entire script; `EXIT _SUB` returns from the current function
- `EXIT -` in a loop body continues to the next iteration; `EXIT LOOP` / `EXIT FORX` breaks out
- Script files typically use `.wcs` extension (or `.wci`, `.wce`). INI files also work. Start with `#code=936T950` on line 1 for GBK-encoded files (the default for Chinese PECMD scripts).

## $2 VARIABLE SYSTEM — THE MOST CRITICAL SECTION

PECMD has a **three-tier** variable system. Getting this wrong causes the majority of bugs.

| Tier | Syntax | Set with | Scope |
|------|--------|----------|-------|
| Environment | `%var%` (or `%%var%%` in CMD) | `ENVI var=val` or `ENVI $var=val` | Process-level, shared with child processes |
| PE variables (local) | `%&var%` | `ENVI &var=val` or `SET var=val` | Current `_SUB` or `{}` block |
| PE class/global | `%&::var%` | `ENVI &::var=val` or `SET &::var=val` | Cross-function, cross-thread, file-level |

### The & naming convention

```wcs
SET localVar=hello           // PE variable (SET = ENVI & by default)
ENVI var=hello                // environment variable (default without ForceLocal=1)
SET &::globalVar=hello       // class-level PE variable — accessible from any _SUB in file
```

**Critical rules:**
1. `ENVI^ ForceLocal=1` (or `ENVI^ FORCELOCAL=1`) at top of file — forces `ENVI` and `SET` to default to local PE variables. Use this ALWAYS.
2. `ENVI^ EnviMode=1` — empty-variable references return empty string instead of error. Always use this.
3. `SET` is **always** equivalent to `ENVI &` (SET always creates PE variables, regardless of ForceLocal). With ForceLocal=1, bare `ENVI var=val` also creates a PE variable (instead of environment).
4. `%Desktop%` is the **environment variable** version; `%&Desktop%` is the **PE variable** version. Both have the same value but different scopes.
5. In multi-threaded code, ALWAYS use PE variables (`&var`) — environment variables are shared across threads and will race. Use `&::` class variables for inter-thread communication.
6. `SET~ &&dest=Source.Key` or `ENVI~ &&dest=&Src.%Key%` — the `~` operator performs **indirect dereferencing**: expands the right side to a variable name, then reads that variable's value. Essential for pseudo-arrays.
7. `^SET` / `^ENVI` (with `^` prefix) — defers variable expansion to execution time, not parse time. Essential for dynamic variable names in loops.
8. `ENVI-ret %~1=%var%` — sets the variable whose *name* is in `%~1` (return-by-reference). Default return level is 1; use `ENVI-ret2 %~1=...` for deeper stack returns.
9. `SET-def var=value` — only sets if the variable is not already defined (safe default).

### Hex data and raw memory

```wcs
SET$ &NL=0d 0a              // hex to wide string (Unicode)
SET$# &buf=*4096 0           // hex to raw bytes (binary buffer)
ENVI$ &data=*1M 30 0d 0a    // variable length hex allocation
```

### Standard file header

```wcs
#code=936T950                // GBK encoding (omit for English-only scripts)
ENVI^ EnviMode=1
ENVI^ ForceLocal=1
SET$ &NL=0d 0a
SET$ &TAB=09
```

### ENVI^ Control Commands

`ENVI^` provides extended control over PECMD's runtime behavior:

| Command | Purpose |
|---------|---------|
| `ENVI^ EXPORTLOCAL=1\|0\|&1` | Control PE variable inheritance to sub-levels. `1` = propagate all PE vars to child `_SUB`/`CALL`; `0` = isolate; `&1` = propagate to all sub-levels recursively |
| `ENVI^ EnviBroad=0\|1\|-` | Control `$`/`#` environment variable broadcast. `0` = no broadcast; `1` = broadcast; `-` = query current mode |
| `ENVI^ Clipboard=text` | Write text to the Windows clipboard |
| `ENVI^ Clipboard?=var` | Read clipboard content into a variable (use `?=` to query) |
| `ENVI^ DisX64=1` | Disable WOW64 filesystem redirection (when 32-bit PECMD runs on 64-bit Windows, prevents automatic `System32`→`SysWOW64` path redirection) |
| `ENVI^ LoadEnvi=file` | Load environment variables from a file into the process environment block |

**Environment variable `$`/`#` prefixes**: When setting environment variables, the prefix controls scope:
- `ENVI $var=value` — system-level environment variable (broadcast to HKLM, visible to all processes)
- `ENVI #var=value` — user-level environment variable (broadcast to HKCU, visible to user processes)
- `ENVI var=value` — process-level only (no broadcast, unless `EnviBroad=1` changes default)

### Function parameter references

When a `_SUB` function is called with arguments, these special variables are available:
```
%0  = function name
%1..%n = argument 1..n
%#  = argument count
%*  = all arguments from %1 onward (space-separated)
%~0..%~n = same as %0..%n but with outer quotes stripped
```
Always use `%~1`, `%~2` etc. for function parameters — they strip quotes safely.

## $3 CODE ORGANIZATION

### Functions

```wcs
_SUB FunctionName [*]              // * = this-call (runs in caller's stack)
    SET &param1=%~1                 // %~1 strips outer quotes
    SET-def &result=                // defensive init
    // ... function body ...
    ENVI-ret %~3=%&result%          // return value via reference parameter
_END
```

### CALL variants

```
CALL FuncName [args]               // call function
CALL *FuncName [args]              // this-call (caller's stack)
CALL @WinName [args]              // create/show window (modal, blocks)
CALL @*WinName [args]             // parallel window (both active)
CALL @-WinName [args]             // background window (continues below)
CALL @~WinName [args]             // background, fully non-blocking
CALL @+WinName [args]             // abandoned child (program exits without waiting)
CALL @^WinName [args]             // like @* but parent doesn't block child
CALL @--WinName                   // destroy Win environment
CALL @--popmenu WinName [x.y[:align]] // popup menu at position
```

### Windows (GUI)

```wcs
_SUB WindowName,L200T100W400H300,Window Title,[close-command],[icon],[style],[mask],[flags]
    // controls go here
_END
```

Window shape: `L<left>T<top>W<width>H<height>`. Omit L/T for centered.
Common flags: `-trap` (close button doesn't exit), `-nocap` (no title bar), `-nosysmenu`, `-top`, `-size`, `-maxb` (enable maximize), `-minb` (enable minimize), `-disminb` (disable minimize button), `-discloseb` (disable close button), `-nfocus` (no keyboard focus), `-ntab` (no tab focus), `-disaltmv` (disable ALT+mouse move), `-forcenomin` (prevent minimize), `-scalef` (XP-style scaling), `-scale[:DPI]` (Win8+ DPI scaling), `-nxp` (no XP visual style), `-csize` (size = client area), `-na` (don't activate), `,#` (hidden window)

### Classes and nesting

```wcs
_SUB ClassName
    _SUB SubFunc              // nested: ClassName.SubFunc
        // ...
    _END
_END
```

Call nested: `CALL ClassName.SubFunc`. Dot-notation (`ENVI aa.bb.cc=1`) is a naming convention for member-like variables. Access parent class with `::` prefix.

### IMPORT

```wcs
IMPORT path\to\library.wcs     // loads reusable functions into current script
```

To make a file safe for importing (prevent execution when loaded as library):
```wcs
_ENDFILE-IMPORT    // everything below this line is discarded when IMPORTed
```

### Code blocks

```wcs
{                               // block with its own PE variable stack
    SET temp=only here
}

{*                              // extended-command block (allows certain syntax extensions)
    // ...
}
```

For file-level and function-level, `{` must start at column 1. For command-group level, `{` must be first character of the command group, `}` before `|` or at line end.

## $4 FLOW CONTROL

### FIND — string comparison (case-sensitive by default)

```wcs
FIND $%var%=hello, command           // equal
FIND $%var%<>hello, command          // not equal
FIND $%var%=hello,!! command         // !! = else with no if-body
FIND $%var%=hello, cmd1! cmd2       // ! separates if-body from else-body
FIND $=%var%, command               // "is not empty" test
FIND $'%var%'='', command           // "is empty" (single-quote protects special chars)
FIND |%a%>%b%, command              // | prefix = numeric comparison (NOT string)
FIND [ $A & $B ], command          // compound AND (same syntax as IFEX)
FIND [| $A | $B ], command         // compound OR
FIND --pid &var,                    // get process/CPU info
FIND --pid*@[.ext|#parentPID] &var, // process list for TABL (opt: filter by .ext or #parent)
FIND --wid*@[parentWID] &var,[titleFilter]  // window list
```

### IFEX — file test / numeric comparison / system query

```wcs
IFEX C:\boot.ini, command           // file/directory exists
IFEX C:\boot.ini,! command          // NOT exists
IFEX x:\, command                   // drive letter exists AND has filesystem
IFEX x:, command                    // drive letter exists (may be unformatted)
IFEX $%val%>=5, command             // numeric comparison
IFEX #%a%<#%b%, command             // # prefix = force integer
IFEX [ cond1 & cond2 ], command     // AND compound condition (use &)
IFEX [| cond1 | cond2 ], command    // OR compound condition (use |)
IFEX MEMU=?,&MemU                   // query free memory into variable
IFEX C:\=?,&FreeSpace               // query free disk space
```

### LOOP — while loop

```wcs
LOOP #%I%<=10,
{
    MESS iteration %I%
    CALC &I=%I% + 1
}
```
Break out: `EXIT LOOP`. Continue to next iteration: `EXIT -`.

### FORX — iteration

```wcs
FORX * %&list%,&&item,            // * = space-delimited
{
    MESS item=%&item%
}

FORX *NL &multiLineVar,&&line,    // *NL = newline-delimited
{
    // process each line
}

FORX /S C:\Windows\*.exe,&&file,0  // file system enumeration (0=files only, 1=dirs only)
{
    MESS found: %&file%
}

FORX @\Windows,&&winDir,1          // search ALL drives for directory (1=first match, 0=all)
{
    MESS Windows at: %&winDir%
}

FORX *v &var1 &var2 &var3,&&v,    // *v = iterate variable names, not values
```

Break out: `EXIT FORX`. Continue: `EXIT -`.

### TEAM — multi-command chaining

```wcs
TEAM SET &a=1| SET &b=2| CALC &c=%&a% + %&b%
```

The `|` pipe inside TEAM is the command separator. For nested TEAM: outermost uses `|`, one level nested uses `||`, two levels nested uses `|||`.

### LOCK — critical section / mutex

```wcs
{ LOCK #pecmd                                   // wrap in braces for atomic scope
    LOCK --exist #MyLock,&&ret                   // check if lock exists (1=yes, 0=no)
}
LOCK #MyLock,&&ret2                             // create/acquire named lock
```

### LAMBDA — anonymous functions

LAMBDA creates first-class function values that can be assigned to variables, passed as arguments, or called directly.

**Syntax**:
```wcs
[] P1 P2 ... Pn { body }
//   ^^ parameter list       ^^ function body
```

**Assignment**:
```wcs
&funcName = [] P1 P2 { body }
// Equivalent to: _SUB funcName P1,P2 ... body ... _END
```

**Direct call**:
```wcs
CALL [] P1 P2 { body } arg1 arg2
// Creates anonymous function and calls it immediately
```

**Variable capture**: PE variables from the outer scope are visible inside the LAMBDA body (closure-like behavior). This is a major advantage over `_SUB` which starts with a clean variable stack.

**Important TEAM/FIND caveat**: When using LAMBDA inside `TEAM` or `FIND` commands, you must use `%%` to escape `%` signs within the LAMBDA body:
```wcs
TEAM &f = [] x { CALC &result=%%&x%% + 1 }| ...
//                             ^^ double % to survive TEAM's own % expansion pass
```

**Common patterns**:
```wcs
// Store LAMBDA in a variable
SET &add = [] a b { CALC &result=%&a% + %&b% }

// Call indirectly via variable
CALL %&add% 3 5

// Pass LAMBDA as callback
CALL ProcessItems %&list%, [] item { MESS Processing: %&item% }

// Store in array-like pattern
SET &handlers[0] = [] { MESS First handler }
SET &handlers[1] = [] { MESS Second handler }
CALL %&handlers[%&index%]%
```

## $5 GUI PROGRAMMING

### Control types

| Control | Command | Purpose |
|---------|---------|---------|
| Button | `ITEM` | Clickable button |
| Label | `LABE` | Static text (can be clickable) |
| Edit | `EDIT` | Text input |
| CheckBox | `CHEK` | On/off toggle |
| Radio | `RADI` | Radio button |
| List | `LIST` | Dropdown / selection |
| Table | `TABL` | Multi-column data grid |
| Progress | `PBAR` | Progress bar |
| Group | `GROU` | Visual grouping box |
| Image | `IMAG` | Picture display |
| Memo | `MEMO` | Multi-line text |
| Sub-window | `SWIN` | Embedded sub-window (tab pages) |
| Timer | `TIME` | Periodic callback |
| Tray Icon | `TIPS*` | System tray icon and tooltip |
| Tabs | `TABS` | Tab control |
| Slider | `SLID` | Range slider |
| Spin | `SPIN` | Up/down spinner |

### Control syntax

```wcs
ITEM [-font:N] [-def] [-right] [-round] [-na] Name,LxTyWwHh,Text,[Command],[State],[Style]

LABE [-vcenter] [-left|-center|-right] [-trans] [-3D] [-ncmd] Name,Shape,Text,Command,[Color],[FontSize]

LIST [-h] Name,Shape,item1|item2|item3,[EventCmd],[Style],[FontSize]

CHEK [-right] Name,Shape,Text,[EventCmd],[State]   // State: 1=checked, 0=unchecked
```

### Message mapping

```wcs
ENVI @Control.MSG=_%&WM_LBUTTONDOWN%: command     // _ = post-system handler (control notifications use _)
ENVI @Window.MSG=0x0010: CALL OnClose              // no _ = window-level messages (WM_CLOSE)
ENVI @Control.MSG=$msg#: command                   // $ = replace system handler
ENVI @Control.POSTMSG=#1                           // post custom message #1
ENVI @Control.SENDMSG=msg#;wParam;lParam           // synchronous send
```

The `_` prefix on MSG is critical: use `_` for control notifications (WM_COMMAND/WM_NOTIFY subtypes), omit `_` for direct window messages.

Common Win32 message values (declare at file top):
```wcs
SET &::WM_LBUTTONDOWN=0x0201
SET &::WM_LBUTTONUP=0x0202
SET &::WM_LBUTTONDBLCLK=0x0203
SET &::WM_RBUTTONDOWN=0x0204
SET &::WM_RBUTTONUP=0x0205
SET &::WM_KEYDOWN=0x0100
SET &::WM_COMMAND=0x0111
SET &::WM_NOTIFY=0x004E
SET &::WM_SIZE=0x0005
SET &::WM_DROPFILES=0x0233
SET &::WM_DEVICECHANGE=0x0219
SET &::WM_MOUSEHOVER=0x02A1
SET &::WM_MOUSELEAVE=0x02A3
SET &::WM_MOUSEENTER=0x1000
SET &::WM_TRAYNOTIFY=1109
```

### Control manipulation

```wcs
ENVI @ControlName=New Text                       // set text
ENVI @ControlName.Enable=0                       // disable (1=enable)
ENVI @ControlName.Visible=0                      // hide (1=show, *4=minimize)
ENVI @ControlName.POS=left:top:width:height      // move/size
ENVI @ControlName.POS=?;&L:&T:&W:&H             // query position
ENVI @ControlName.bkcolor=0xFF0000               // set background color
ENVI @ControlName.Font=12:Microsoft YaHei        // set font
ENVI @ControlName.Check=1                        // check/uncheck
ENVI @ControlName.Val=data                       // set TABL/LIST content
ENVI @ControlName.Val=?row.col;&var              // get TABL cell (semicolon for result var)
ENVI @ControlName.Val=?*;&count                  // get TABL row count
ENVI @ControlName.Val=-*                         // clear all TABL rows
ENVI @ControlName.Val=1*;%&allData%              // bulk-set all rows from variable
ENVI @ControlName.Sel=index                      // select item
ENVI @ControlName.Sel=index;0                    // deselect
ENVI @ControlName.Sel=?;&var                     // get selection
ENVI @ControlName.*del=                          // destroy control
ENVI @ControlName.Cursor=32649                   // hand cursor
```

### LIST operations

```wcs
ENVI @ListName.VAL=                             // clear all items
ENVI @ListName.ADD=item                          // add one item
ENVI @ListName.ADDSEL=item                      // add and select
ENVI @ListName.DEL=item                          // delete item
ENVI @ListName.QUERY=;&allItems                  // get all items (NL-delimited)
ENVI @ListName.isel=N                            // select by 1-based index
```

### Popup menu

```wcs
_SUB MyMenu
    MENU ItemName,Display Text,CALL Handler
    MENU -                                       // separator
    MENU Item2,Exit,KILL \
_END
// Trigger from right-click on control:
ENVI @Control.MSG=_%&::WM_RBUTTONDOWN%: CALL @--popmenu MyMenu
```

### Window manipulation (cross-process)

```wcs
ENVI @WindowName.Visible=0|1|*4                  // hide/show/minimize
ENVI @@Visible=windowID:0|1|*4                   // cross-process visibility control
ENVI @@POS=windowID:::::::1                       // bring window to foreground
```

## $6 DLL CALLING

```wcs
CALL $--qd --ret:&&ret DLLPath,FunctionName,[#]Param1,[#]Param2,...
CALL $--qd --bool --ret:&&ret DLL,Func,...       // --bool: function returns BOOL
CALL $--qd --cd --ret:&&ret DLL,Func,...         // --cd: chdir to DLL dir first
CALL $--qd --ret:&&ret ,-LoadLibrary,[^]DLLPath  // load DLL (^ = self-releasing)
CALL $--qd --ret:&&ret ,-FreeLibrary,*hDll       // free loaded DLL
CALL $--qd --ret:&&ret ,-GetProcAddress,*hDll,FuncName  // get function address
CALL $--win DLL,Func,cmdLine                     // rundll32-style
CALL $--cpl CPLpath                              // control panel applet
```

DLL params: `#N`=integer, `$s`=wide string, `@s`=narrow string, `*buf`=buffer pointer, `=s`=raw string
Type override per-param: `--qd#` (all int), `--qd*` (all PE var), `--qd$` (all string), `--qd@` (all narrow)

### Memory buffer operations

```wcs
SET$# &buf=*4096 0                    // allocate 4096 zero-filled raw bytes
SET-long &buf=value:offset            // write 32-bit int at offset
SET-ptr &buf=value:offset             // write pointer-sized value at offset
SET-short &buf=value:offset           // write 16-bit short at offset
SET-copy &buf=&src;srcOff;len;dstOff  // raw byte copy between buffers
ENVI-addr &&ptrVar=&buf               // get buffer address
ENVI-ptr %~1=&buf:offset              // write pointer to return buffer
ENVI-mkdummy &&Name=&buf@offset;len   // create "view" variable from buffer slice
SET-make &&Str=&buf@offset;$len      // extract string from buffer at offset
SET?int &buf=&&Val:offset             // read 32-bit int from buffer
SET?longlong &buf=&&Val:offset        // read 64-bit from buffer
SET?char &buf=&&Val:offset            // read byte from buffer
```

### Platform detection for struct sizes

```wcs
IFEX #%&::bX64%=3, SET &PtrSz=8! SET &PtrSz=4    // 3=PECMD64, 1=32on64, 0=32on32
```

## $7 COMMON IDIOMS (Quick Reference)

All patterns below are from real PECMD code. Full expanded versions with explanations in `references/codebook.md`.

### Disk & partition

```wcs
FDRV &drives=*:                                          // enumerate all drive letters
FDRV *vol &label,&fs=C:                                  // get volume label + filesystem
FORM -raw &&type=D:                                       // get drive type constant
FORM TYPE,&&bus,BUS=D:                                    // get bus type (USB, SATA, etc.)
PART list disk,&&disks                                    // enumerate all disks
PART list disk %&dsk%,&&info                              // get disk info
PART list part %&dsk%,&&parts                             // list partition numbers on disk
PART -hextp -phy# list part %&dsk%#%&pt%,&&info           // get detailed partition info (GPT+MBR)
PART -super -up -xup %&dsk%#%&pt% %&type%                 // change partition type (use -super -up together)
SHOW * %&dsk%#%&pt%,%&drv%                                // assign drive letter
SHOW *- %&dsk%#%&pt%,                                     // remove drive letter
```

PART output fields (MBR): `分区号 类型 激活 起始 长度 隐藏扇区 结束 物理号 盘符`
PART output fields (GPT): `分区号 GUID 属性 起始 长度 结束 物理号 盘符`

### Registry operations

```wcs
REGI $HKLM\SOFTWARE\App\Key,&var                 // read REG_SZ
REGI #HKLM\SOFTWARE\App\Count,&var               // read REG_DWORD
REGI @HKLM\SOFTWARE\App\Data,&var                // read REG_BINARY
REGI ~HKLM\SOFTWARE\App\Path,&var                // read REG_EXPAND_SZ
REGI +HKLM\SOFTWARE\App\BigNum,&var              // read REG_QWORD
REGI $HKLM\SOFTWARE\App\Key=value                 // write REG_SZ
REGI #HKLM\SOFTWARE\App\Count=#0x100             // write REG_DWORD
REGI $HKLM\SOFTWARE\App\Key=                      // delete value
REGI .HKLM\SOFTWARE\App\,&&keys                   // offline hive: enumerate subkeys
REGI .HKLM\SOFTWARE\App\Key,&var                  // offline hive: read value
```

### File I/O

```wcs
READ %path%,**,&content           // read entire file (text, DOS CRLF -> native)
READ %path%,*r,&content           // raw read (no line-end conversion)
GETF# %path%,0#*,&raw             // read entire file as raw bytes
WRIT %path%,$0,first line         // write to file ($ = ANSI, 0 = line 0 =overwrite)
WRIT %path%,$+0,append line       // append
BROW &result,[*|&]initPath,[prompt],[filter],[flags]  // file/browse dialog
// * = folder browser, & = save dialog, 0x200 = multi-select, 0x10 = edit box

// Parse INI-style config
READ %curdir%\config.ini,*,&cfg
FORX *NL &cfg,&&line,{SED &&k=1,=.*,,%&line%|SED &&v=1,.*=,,%&line%|SET %&k%=%&v%}

// Win32 INI API
CALL $--qd --ret:&r Kernel32.dll,GetPrivateProfileStringW,$sect,$key,$def,*&buf,#65535,$%file%
CALL $--qd --ret:&r Kernel32.dll,WritePrivateProfileStringW,$sect,$key,$val,$%file%
```

### Execute & capture

```wcs
EXEC* &output=!cmd.exe /c dir /b                        // capture all output (preserves line endings)
EXEC*1 &firstLine=!program.exe                          // capture only first line (then kill program)
EXEC*N &oneLine=!program.exe                           // join all lines, strip newlines
EXEC*- &trim=!program.exe                              // strip trailing newline only
EXEC* -cmd:::OnLine -err+ &out=!"program.exe" args      // real-time line callback + stderr
EXEC =!"%MyNAME%" TEAM WAIT 1000|LOAD other.ini         // run sub-PECMD, wait for finish
EXEC* &&out=*IPCONFIG                                   // * prefix = internal PECMD command
EXEC* -exe:#101 &&out=*embedded.exe                     // run EXE from PECMD internal resources
```

### Detect boot mode (BIOS/UEFI)

```wcs
SET$# &buf=*16 0 *4 0 *8 0
CALL $--qd --ret:&&r ntdll.dll,NtQuerySystemInformation,#90,*&buf,#32,#0
SET?int &buf=&&type:16
IFEX #%&type%=1,MESS BIOS!IFEX #%&type%=2,MESS UEFI!MESS Unknown
```

### Single-instance (mutex)

```wcs
{ LOCK #pecmd
    LOCK --exist #MyAppLock,&&exists
}
IFEX $1=%&exists%,
{
    REGI .HKCU\Software\MyApp\WID,&&wid
    IFEX $%&wid%>0, TEAM ENVI @@Visible=%&wid%:2| ENVI @@POS=%&wid%:::::::1
    EXIT FILE
}
LOCK #MyAppLock,&ret2
```

### Window with background thread + timer

```wcs
THREAD* CALL LongTask                  // starts immediately, non-blocking
TIME Timer1,500, CALL CheckProgress     // UI timer for periodic updates
ENVI @Window.POSTMSG=#1                 // thread signals completion via custom message
ENVI @Timer1=0                          // stop timer
ENVI @Timer1=-del                       // destroy timer
```

### DPI-aware layout

```wcs
REGI #HKCU\Control Panel\Desktop\WindowMetrics\AppliedDPI,&dpi
FIND $%&dpi%=, SET dpi=96
SET &DPI=%&dpi%/96
// Check if system already auto-scaled:
ENVI @this.POS=?::&actualW:&actualH
IFEX [ %&actualW%=%&designedW% & %&actualH%=%&designedH% ], SET DPI=1
CALC #&w=400*%&DPI%
ITEM Btn,L%x%T%y%W%&w%H30,Text
```

### Hotkey

```wcs
HKEY Ctrl+Shift+#0x41, CALL OnHotkeyA            // register
HKEY #0x0D,--del                                  // unregister
```

### TABL data operations

```wcs
ENVI @TABL.Val=1*;%&data%                          // bulk-set all rows
ENVI @TABL.Val=%row%;col1%TAB%col2...              // set single row
ENVI @TABL.Val=?%row%.%col%;&var                   // get cell (semicolon before var name)
ENVI @TABL.Val=?*;&rowCount                        // get row count
ENVI @TABL.Val=-*                                   // clear all rows
ENVI @TABL.Sel=%row%                                // select row
ENVI @TABL.Sel=%row%;0                              // deselect row
// Column format flags in title: 100:磁盘  =90:大小  +50:磁头数
//   (default=left, =  =center, +  =right-align, empty/blank = no column header text)
```

### String manipulation

```wcs
MSTR &&a,&&b=<1><~3>%&data%                          // field 1, fields 3-through-end
MSTR &&last=<-1>%&data%                               // last field
MSTR -delims:. &&a,&&b,&&c,&&d=<1*>%&ip%             // split by custom delimiter
// MSTR command-level flags (prefix MSTR itself — control delimiter behavior):
MSTR$ a,b,c=<1*>%&data%                               // $ flag: treat consecutive spaces as single delimiter
MSTR* a,b,c=<1*>%&data%                               // * flag: TAB is the field separator
// Note: command-level * (TAB delimiter) vs segment-spec <N*> (fields N through end) are unrelated.
// MSTR$ and MSTR* are prefixes on the MSTR command; <N*> is a field-range specifier.
// Real-world example: parse TAB-delimited command output (e.g., DISKPART, WMIC)
EXEC* &raw=!wmic.exe logicaldisk get DeviceID,Size,FreeSpace /format:csv
FORX *NL &raw,&&line,{ MSTR* &&node,&&dev,&&size,&&free=<1><2><3><4>%&line% }
SED &&r=0,pattern,replacement,%&source%              // replace ALL occurrences
SED &&r=1,find,replace,%&source%                     // replace FIRST only
SED &&ext=-1,.*\.,,%&filename%                        // get file extension (negative = from end)
LPOS &&pos=needle,,%&haystack%                        // find first (case-insensitive)
RPOS &&pos=needle,,%&haystack%                        // find last
RSTR &&pad=3,000%num%                                 // zero-pad to 3 digits
```

### Dynamic variable reference (pseudo-array)

```wcs
SET Arr.1.1=row1col1
SET Arr.1.2=row1col2
SET~ &&val=Arr.%&row%.%&col%                          // indirect read
^SET &Drv[%&hd%]=%%&Drv[%&hd%]%%%%&D%                // append with delayed expansion
```

## $8 TRAPS & GOTCHAS

1. **CALC spacing**: In practice, `CALC &J=1+2` works without space. When the right side starts with a variable like `%&I%`, use a space (`CALC &J= %&I%+1`) to avoid parse ambiguity. The help.txt specifically documents: a space is required after the minus sign (e.g., `3 - 2`).
2. **Comment markers**: `//` and `;` are comment markers. They must be preceded by a space when used at end of line. `//comment` at line start may not be recognized. Line-end comments need ` space //` format.
3. **SET is ENVI &**: `SET var=val` is semantically `ENVI &var=val`. With ForceLocal=1, both create local PE variables. Without ForceLocal=1, `ENVI var=val` creates an environment variable but `SET var=val` still creates a PE variable.
4. **FIND vs IFEX**: Both can do string AND numeric comparison. `FIND $` = string comparison (default). `FIND |` = numeric comparison. `IFEX $` = numeric comparison. Don't over-think this — just be explicit with the prefix. The `|` in IFEX is only for compound OR conditions like `IFEX [| cond1 | cond2 ]`.
5. **Drive letter colon**: `FDRV`, `FORM`, `FIND C:\=?` all need `:` suffix. `FDRV &d=C:` is correct.
6. **Paths with spaces**: `LOAD "C:\Program Files\a.ini"` — require quotes. Variables: `LOAD %CurDir%\a.ini` may work without quotes.
7. **File encoding**: Chinese/GBK scripts need `#code=936T950` as line 1 AND must be saved as ANSI/GBK, not UTF-8. For English-only scripts, this is unnecessary.
8. **Comments ON/OFF**: Command-line mode: comments OFF by default. Script (LOAD) mode: comments ON. Use `COME 0` / `COME 1` to toggle.
9. **`{` positioning**: File-level and function-level `{` must start at column 1. Inside TEAM/LOOP/IFEX, `{` starts a command group.
10. **Line continuation**: `\` as the first non-space character on a line merges that line with the previous one (continuation).
11. **`_SUB` on separate lines**: Cannot define `_SUB` inside FIND/IFEX/TEAM commands.
12. **Dashed names in CALL**: `FIND-JPG` as a command name is illegal (dash starts suffix). Use `CALL FIND-JPG` explicitly.
13. **Empty string check**: `FIND $=%var%,` — when the left side of `$` is empty and right side is a variable, this tests "is not empty". `FIND $%var%=,` tests "is empty".
14. **Literal %**: Use `%%` to represent a literal `%` in strings.
15. **Exit codes**: PECMD.exe exit code = last command's error code. 0=success, other=failure.
16. **Thread safety**: Threads receive a COPY of the parent's PE variables at creation time. Use `&::` variables for true inter-thread sharing. Never share environment variables across threads.
17. **Chinese variable names**: Real PECMD code overwhelmingly uses Chinese variable names. This is the de facto standard in the PECMD community. Use Chinese names when writing scripts for this ecosystem, though English names are also fully supported.
18. **OnShutdown.wcs auto-execution**: PECMD automatically runs `%SystemRoot%\System32\OnShutdown.wcs` (if it exists) before system shutdown/reboot/logoff, passing the operation code as `%1` (shutdown, reboot, poweroff, logoff, suspend, hibernate).
19. **&&__RET convention**: The standard function return variable. When using `ENVI-ret` to return values, the caller can introspect with `&&__RET`.
20. **WM_NOTIFY / WM_COMMAND subfields**: After receiving a WM_NOTIFY message (via `_msg#` mapping), the subfields `%&__NMHDR.idFrom%`, `%&__NMHDR.code%`, `%&__NMHDR.hwndFrom%` are auto-parsed. Similarly, WM_COMMAND provides `%&__wParam.wID%` and `%&__wParam.wNotifyCode%`.
21. **FIND expansion rule**: In FIND, bare identifiers (no `%` wrappers) are treated as literal strings. Always use `FIND $%&var%=value` for PE variables, not `FIND $&var=value`.

## $9 WHEN TO READ REFERENCE FILES

- **Need a specific command's full syntax with all flags?** → Read `references/commands-full.md`
- **Need to see how a pattern works in detail, with real-world code?** → Read `references/codebook.md`
- **Building a WinPE startup/init script?** → Read `references/pe-startup.md` for PECMD.INI structure and boot flow
- **Building a GUI application with PECMD windows and controls?** → Read `references/pecmd-gui.md`

## $10 BUILT-IN ENVIRONMENT VARIABLES

| Variable | Scope | Meaning |
|----------|-------|---------|
| `%CurDir%` / `%&CurDir%` | PE/Env | Current script directory |
| `%CurFile%` / `%&CurFile%` | PE/Env | Current script full path |
| `%MyName%` | Env | PECMD.EXE full path |
| `%Desktop%` | Env | Desktop directory (use `%&Desktop%` for PE version) |
| `%SystemRoot%` | Env | Windows directory |
| `%&bX64%` | PE | 3=64bit PECMD, 1=32bit on 64, 0=32bit on 32 |
| `%&PECMDVER%` | PE | PECMD version string |
| `%&__WinID%` | PE | Current window HWND |
| `%&__LastWinID%` | PE | Last created window HWND |
| `%&__PID%` | PE | Current process ID |
| `%&__LastPID%` | PE | Last created process/thread PID |
| `%&__TID%` | PE | Current thread ID |
| `%&__LastTID%` | PE | Last created thread ID |
| `%&&__MAIN__%` | PE | 1 when running as main script (0 when IMPORTed) |
| `%RANDOM%` | Env | Random 63-bit integer (changes each read) |
| `%&&ERROR%` / `%&ERROR%` | PE | Last command error code |
| `%&&ERRORLEVEL%` | PE | Exit code of last EXEC-waited program |
| `%&__OldDir%` | PE | Directory before LOAD or at startup |
| `%&_CD%` / `%_CD%` | PE | Real-time current working directory |
| `%&__PPID%` | PE | Parent process ID |
| `%&__HINST%` | PE | Process module handle |
| `%&SYSCODEPAGE%` | PE | System language codepage number |
| `%&PeExe%` | PE | 1=normal EXE, 0=built-in script, -1=initialization |
| `%&__THIS%` | PE | Unique cookie identifying current PE call stack |
| `%&&__RET%` | PE | Convention: function return variable |
| `%&__NMHDR.idFrom%` | PE | WM_NOTIFY: control ID |
| `%&__NMHDR.code%` | PE | WM_NOTIFY: notification code |
| `%&__NMHDR.hwndFrom%` | PE | WM_NOTIFY: sender HWND |
| `%&__wParam.wID%` | PE | WM_COMMAND: control ID |
| `%&__wParam.wNotifyCode%` | PE | WM_COMMAND: notification code |
| `%PECMDBUILD%` | Env | PECMD build date |

## $11 OUTPUT CONVENTIONS

When writing PECMD scripts and tools, follow these conventions:

1. Use `#code=936T950` on line 1 for Chinese scripts, then `ENVI^ EnviMode=1`, then `ENVI^ ForceLocal=1`
2. Default to Chinese variable names — this is the de facto standard in the PECMD community
3. Use PE variables (`&varname`) everywhere unless you specifically need environment variables
4. Use `TEAM` chaining extensively for compact initialization sequences
5. Break complex operations into `_SUB` functions with clear names
6. For GUIs, save initial control positions with `ENVI @control.POS=?` and restore on resize events
7. Always handle the WM_SIZE message (0x0005) for resizable windows to recalculate layout
8. When calling external programs, prefer `EXEC =!"%MyNAME%" ...` for sub-PECMD or `EXEC* &out=!cmd.exe /c ...` for output capture
9. Thread safety: use `&::` class PE variables for inter-thread communication
10. For production code, wrap mutual-exclusion operations in `{ LOCK #pecmd ... }` blocks
11. Window messages on controls use `_` prefix (`_0x0201`); window-level messages omit `_` (`0x0010`)
12. Use `SED` for string manipulation and `MSTR` for field extraction from structured command output
