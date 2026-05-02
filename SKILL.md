---
name: pecmd-pro-max
version: 1.2.1
description: |
  PECMD2012 scripting for WinPE — lightweight Windows GUIs, system
  tools, boot/init scripts, and automation. Use for .wcs/.wci/.wce files,
  disk partitioning, batch-to-PECMD conversion, system info collectors,
  PE/pre-install environment utilities, and any PECMD code debugging.
  References: commands-full.md, pecmd-gui.md, pe-startup.md,
  recipes/storage.md, recipes/system.md, recipes/gui.md, recipes/net.md.
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
- `PECMD.EXE MAIN path\to\PECMD.INI` — the standard WinPE entry point, executes the INI and starts the message loop
- Run PECMD inside itself: `EXEC =!"%MyNAME%" <command>` — useful for isolated sub-operations
- `EXIT FILE` terminates the entire script; `EXIT _SUB` returns from the current function
- `EXIT -` in a loop body continues to the next iteration; `EXIT LOOP` / `EXIT FORX` breaks out
- Script files use `.wcs` extension. Start with `#code=65001` on line 1 for UTF-8 encoded files. If line 1 starts with `#!`, the encoding directive goes on line 2.

## $2 VARIABLE SYSTEM — THE MOST CRITICAL SECTION

PECMD has a **three-tier** variable system. Getting this wrong causes the majority of bugs.

| Tier | Syntax | Set with | Scope |
|------|--------|----------|-------|
| Environment | `%var%` | `ENVI var=val` or `ENVI $var=val` | Process-level, shared with child processes |
| PE variables (local) | `%&var%` | `ENVI &var=val` or `SET var=val` | Current `_SUB` or `{}` block |
| PE class/global | `%&::var%` | `ENVI &::var=val` or `SET &::var=val` | Cross-function, cross-thread, file-level |

**Critical rules:**
1. `ENVI^ ForceLocal=1` at top of file — forces `ENVI` and `SET` to default to local PE variables. Use this ALWAYS.
2. `ENVI^ EnviMode=1` — empty-variable references return empty string instead of error. Always use this.
3. `SET` is **always** equivalent to `ENVI &` (SET always creates PE variables, regardless of ForceLocal).
4. `%Desktop%` is the **environment variable** version; `%&Desktop%` is the **PE variable** version.
5. In multi-threaded code, ALWAYS use PE variables (`&var`) — environment variables are shared across threads and will race. Use `&::` class variables for inter-thread communication.
6. `SET~ &&dest=Source.Key` — the `~` operator performs **indirect dereferencing**: expands the right side to a variable name, then reads that variable's value. Essential for pseudo-arrays.
7. `^SET` / `^ENVI` (with `^` prefix) — defers variable expansion to execution time. Essential for dynamic variable names in loops.
8. `ENVI-ret %~1=%var%` — sets the variable whose *name* is in `%~1` (return-by-reference).
9. `SET-def var=value` — only sets if the variable is not already defined (safe default).

### Standard file header

```wcs
#code=65001
ENVI^ EnviMode=1
ENVI^ ForceLocal=1
SET$ &NL=0d 0a
SET$ &TAB=09
```

### Hex data and raw memory

```wcs
SET$ &NL=0d 0a                  // hex to wide string (Unicode)
SET$# &buf=*4096 0              // hex to raw bytes (binary buffer)
ENVI$ &data=*1M 30 0d 0a        // variable length hex allocation
```

For full ENVI^ control commands, binary buffer operations (SET-cmp, SET-tom, SET-copy, SET?int, etc.), and function parameter references, see [commands-full.md](references/commands-full.md).

## $3 CODE ORGANIZATION

### Functions and CALL variants

```wcs
_SUB FunctionName [*]              // * = this-call (runs in caller's stack)
    SET &param1=%~1                 // %~1 strips outer quotes
    ENVI-ret %~3=%&result%          // return value via reference parameter
_END
```

```
CALL FuncName [args]               // call function
CALL *FuncName [args]              // this-call (caller's stack)
CALL @WinName [args]              // create/show window (modal, blocks)
CALL @*WinName [args]             // parallel window
CALL @-WinName [args]             // background window
CALL @~WinName [args]             // background, fully non-blocking
CALL @^WinName [args]             // parallel, parent doesn't block child
CALL @+WinName [args]             // abandoned child
CALL @--WinName                   // destroy Win environment
CALL @--popmenu WinName [x.y]     // popup menu at position
```

### Windows (GUI)

```wcs
_SUB WindowName,L200T100W400H300,Window Title,[close-command],[icon],[style],[mask],[flags]
_END
```

Window shape: `L<left>T<top>W<width>H<height>`. Omit L/T for centered.
Common flags: `-trap` (close button doesn't exit), `-nocap` (no title bar), `-nosysmenu`, `-top`, `-size`, `-maxb`, `-minb`, `-disminb`, `-discloseb`, `-nfocus`, `-ntab`, `-forcenomin`, `-nofix`, `-nb`, `-scalef`, `-scale[:DPI]`, `-nxp`, `-csize`, `-na`

### IMPORT and code blocks

```wcs
IMPORT path\to\library.wcs     // loads reusable functions into current script
_ENDFILE-IMPORT                // everything below discarded when IMPORTed

{                               // block with its own PE variable stack
    SET temp=only here
}
```

File-level and function-level `{` must start at column 1. Nest `_SUB` with dot-notation: `ClassName.SubFunc`.

## $4 FLOW CONTROL

### FIND — string comparison (case-sensitive by default)

```wcs
FIND $%var%=hello, command           // equal
FIND $%var%<>hello, command          // not equal
FIND $=%var%, command               // "is empty" test
FIND *=var, command                  // IDIOM: "is empty"
FIND *<>var, command                 // IDIOM: "is NOT empty"
FIND |%a%>%b%, command              // | prefix = numeric comparison
FIND [ $A & $B ], command           // compound AND
FIND [| $A | $B ], command          // compound OR
FIND --pid &var,                    // get process/CPU info
FIND --pid*@ &var,                  // process list for TABL
FIND --wid*@ &var,titleFilter       // window list
```

### IFEX — file test / numeric comparison

```wcs
IFEX C:\boot.ini, command           // file/directory exists
IFEX C:\boot.ini,! command          // NOT exists
IFEX x:\, command                   // drive letter exists AND has filesystem
IFEX $%val%>=5, command             // $=numeric comparison
IFEX [ cond1 & cond2 ], command     // AND compound condition
IFEX [| cond1 | cond2 ], command    // OR compound condition
IFEX MEMU=?,&var                    // query free memory
IFEX C:\=?,&FreeSpace               // query free disk space
```

### LOOP / FORX / TEAM / LOCK

```wcs
LOOP #%I%<=10, { MESS %I% | CALC &I=%I% + 1 }

FORX * %&list%,&&item, { MESS %&item% }              // * = space-delimited
FORX *NL &multiLineVar,&&line, { ... }                // *NL = newline-delimited
FORX /S C:\Windows\*.exe,&&file,0 { ... }             // file system enum
FORX @\Windows,&&winDir,1 { ... }                     // search ALL drives

TEAM SET &a=1| SET &b=2| CALC &c=%&a% + %&b%          // multi-cmd chain

{ LOCK #pecmd                                     // atomic scope
    LOCK --exist #MyLock,&&ret                     // check if lock exists
}
LOCK #MyLock,&&ret2                               // create/acquire named lock
```

For full EXIT variants, LAMBDA syntax, and `FIND --class:`, see [commands-full.md](references/commands-full.md).

## $5 GUI PROGRAMMING

### Control types

| Control | Command | Control | Command |
|---------|---------|---------|---------|
| Button | `ITEM` | Label | `LABE` |
| Edit | `EDIT` | CheckBox | `CHEK` |
| Radio | `RADI` | List | `LIST` |
| Table | `TABL` | Progress | `PBAR` |
| Group | `GROU` | Image | `IMAG` |
| Memo | `MEMO` | Sub-window | `SWIN` |
| Timer | `TIME` | Tray Icon | `TIPS*` |
| Tabs | `TABS` | Slider | `SLID` |
| Spin | `SPIN` | Tree | `TREE` |

### Message mapping

```wcs
ENVI @Control.MSG=_%&WM_LBUTTONDOWN%: command     // _ = post-system handler (control notifications)
ENVI @Window.MSG=0x0010: CALL OnClose              // no _ = window-level messages (WM_CLOSE)
ENVI @Control.POSTMSG=#1                           // post custom message #1
ENVI @Control.SENDMSG=msg#;wParam;lParam           // synchronous send
```

The `_` prefix on MSG is critical: use `_` for control notifications (WM_COMMAND/WM_NOTIFY subtypes), omit `_` for direct window messages.

### Control manipulation

```wcs
ENVI @ControlName=New Text                       // set text
ENVI @ControlName.Enable=0                       // disable (1=enable)
ENVI @ControlName.Visible=0                      // hide (1=show)
ENVI @ControlName.POS=left:top:width:height      // move/size
ENVI @ControlName.POS=?;&L:&T:&W:&H             // query position
ENVI @ControlName.Val=?row.col;&var              // get TABL cell
ENVI @ControlName.Val=?*;&count                  // get TABL row count
ENVI @ControlName.*del=                          // destroy control
```

For full control syntax, all 22 control types, ENVI @ property reference, window lifecycle, and message mapping — see [pecmd-gui.md](references/pecmd-gui.md).

For GUI code recipes (dynamic controls, tab pages, custom titlebar, GDI, drag-drop, etc.) — see [recipes/gui.md](references/recipes/gui.md).

## $6 DLL CALLING

```wcs
CALL $--qd --ret:&&ret DLLPath,FunctionName,[#]Param1,[#]Param2,...
CALL $--qd --bool --ret:&&ret DLL,Func,...       // --bool: function returns BOOL
CALL $--qd --cd --ret:&&ret DLL,Func,...         // --cd: chdir to DLL dir first
CALL $--qd --ret:&&ret ,-LoadLibrary,[^]DLLPath  // load DLL (^ = self-releasing)
CALL $--qd --ret:&&ret ,-FreeLibrary,*hDll       // free loaded DLL
CALL $--qd --ret:&&ret ,-GetProcAddress,*hDll,FuncName
```

DLL params: `#N`=integer, `$s`=wide string, `*buf`=buffer pointer, `=s`=raw string.
Type override per-param: `--qd#` (all int), `--qd*` (all PE var), `--qd$` (all string).
Platform detection: `IFEX #%&::bX64%=3, SET &PtrSz=8! SET &PtrSz=4`.

For full DLL calling reference, memory buffer operations (SET-long, SET?int, ENVI-addr, etc.), and the CALL $ extended type system, see [commands-full.md](references/commands-full.md).

## $7 COMMON IDIOMS

### Disk & partition

```wcs
FDRV &drives=*:                                          // enumerate all drive letters
FDRV *vol &label,&fs=C:                                  // get volume label + filesystem
PART list disk,&&disks                                    // enumerate all disks
PART list disk %&dsk%,&&info                              // get disk info
PART list part %&dsk%,&&parts                             // list partition numbers
PART -hextp -phy# list part %&dsk%#%&pt%,&&info           // detailed partition info
SHOW * %&dsk%#%&pt%,%&drv%                                // assign drive letter
SHOW *- %&dsk%#%&pt%,                                     // remove drive letter
```

### Registry & file I/O

```wcs
REGI $HKLM\SOFTWARE\App\Key,&var                 // read REG_SZ
REGI #HKLM\SOFTWARE\App\Count,&var               // read REG_DWORD
REGI $HKLM\SOFTWARE\App\Key=value                 // write REG_SZ
READ %path%,**,&content                           // read entire file
WRIT %path%,$0,first line                         // write to file
GETF# %path%,0#*,&raw                             // read file as raw bytes
```

### Execute & capture

```wcs
EXEC* &output=!cmd.exe /c dir /b                        // capture all output
EXEC =!"%MyNAME%" TEAM WAIT 1000|LOAD other.ini         // run sub-PECMD
EXEC* &out=*IPCONFIG                                   // internal PECMD command
EXEC* -exe:#101 &out=*embedded.exe                     // EXE from PECMD resources
```

### Single-instance mutex

```wcs
{ LOCK #pecmd
    LOCK --exist #MyAppLock,&&exists
}
IFEX $1=%&exists%,
{
    REGI $HKCU\Software\MyApp\WID,&&wid
    IFEX $%&wid%>0, TEAM ENVI @@Visible=%&wid%:2| ENVI @@POS=%&wid%:::::::1
    EXIT FILE
}
LOCK #MyAppLock,&ret2
```

### String manipulation

```wcs
MSTR &&a,&&b=<1><~3>%&data%                          // field 1, fields 3-through-end
MSTR &&last=<-1>%&data%                               // last field
MSTR -delims:. &&a,&&b,&&c,&&d=<1*>%&ip%             // split by custom delimiter
SED &&r=0,pattern,replacement,%&source%              // replace ALL occurrences
LPOS &&pos=needle,,%&haystack%                        // find first (case-insensitive)
```

For expanded versions with error handling and edge cases, see [recipes/storage.md](references/recipes/storage.md) (disk/reg/file), [recipes/system.md](references/recipes/system.md) (process/thread/timer), [recipes/net.md](references/recipes/net.md) (network/COM).

## $8 TRAPS & GOTCHAS

1. **CALC spacing**: `CALC &J=1+2` works. When the right side starts with a variable like `%&I%`, use a space (`CALC &J= %&I%+1`). PECMD requires a space after minus sign (`3 - 2`).
2. **Comment markers**: `//` and `;` must be preceded by a space at end of line. `//comment` at line start may not be recognized.
3. **SET is ENVI &**: `SET var=val` is semantically `ENVI &var=val`. With ForceLocal=1, both create local PE variables.
4. **FIND vs IFEX**: `FIND $` = string comparison. `FIND |` = numeric comparison. `IFEX $` = numeric comparison. Be explicit with the prefix.
5. **Drive letter colon**: `FDRV`, `FORM`, `FIND C:\=?` all need `:` suffix.
6. **Paths with spaces**: `LOAD "C:\Program Files\a.ini"` requires quotes.
7. **File encoding**: Chinese scripts need `#code=65001` on line 1 AND must be saved as UTF-8.
8. **`{` positioning**: File-level and function-level `{` must start at column 1. Inside TEAM/LOOP/IFEX, `{` starts a command group.
9. **Line continuation**: `\` as first non-space character merges that line with the previous one.
10. **`_SUB` on separate lines**: Cannot define `_SUB` inside FIND/IFEX/TEAM commands.
11. **Empty string check**: `FIND $%var%=,` tests "is empty". `FIND *=var,` tests "is not empty".
12. **Literal %**: Use `%%` to represent a literal `%` in strings.
13. **Thread safety**: Threads receive a COPY of parent's PE variables. Use `&::` variables for true inter-thread sharing.
14. **Chinese variable names**: The de facto standard in the PECMD community. Use Chinese names when writing scripts for this ecosystem.
15. **OnShutdown.wcs**: PECMD auto-runs `%SystemRoot%\System32\OnShutdown.wcs` before system shutdown/reboot/logoff.
16. **`^` pre-interpretation**: `^COMMAND` defers variable expansion to execution time (essential in loops). `^^COMMAND` pre-interprets twice.
17. **`_` prefix on MSG**: Control notifications use `_msg#`; window-level messages omit `_`. Getting this wrong is a very common bug.
18. **THREAD* vs THREAD**: Only THREAD* in a persistent (window) stack shares PE variables. In `{}` blocks, both copy.
19. **FIND expansion rule**: Bare identifiers in FIND are literal strings. Always use `FIND $%&var%=value` for PE variables.
20. **`@@Visable` vs `@Visible`**: Cross-process uses `ENVI @@Visable=WinID:value`. In-process uses `ENVI @Control.Visible=0|1`.

## $9 REFERENCE FILES

When you need details beyond the quick reference above, consult the authoritative files:

| File | When to consult |
|------|-----------------|
| `references/commands-full.md` | Command syntax, all flags, parameter details |
| `references/pecmd-gui.md` | GUI controls, messages, window lifecycle, ENVI @ properties |
| `references/pe-startup.md` | WinPE startup scripts, PECMD.INI structure, boot phases |
| `references/recipes/storage.md` | Disk, partition, file, registry, device code recipes |
| `references/recipes/system.md` | Process, thread, timer, system info, encryption, utilities |
| `references/recipes/gui.md` | GUI patterns: dynamic controls, tabs, GDI, drag-drop, etc. |
| `references/recipes/net.md` | Network, SOCK, COM/WMI code recipes |

## $10 OUTPUT CONVENTIONS

When writing PECMD scripts and tools:

1. Use `#code=65001` on line 1 for Chinese scripts, then `ENVI^ EnviMode=1`, then `ENVI^ ForceLocal=1`
2. Default to Chinese variable names — the de facto standard in the PECMD community
3. Use PE variables (`&varname`) everywhere unless you specifically need environment variables
4. Use `TEAM` chaining extensively for compact initialization sequences
5. Break complex operations into `_SUB` functions with clear names
6. For GUIs, save initial control positions with `ENVI @control.POS=?` and restore on resize
7. Always handle WM_SIZE (0x0005) for resizable windows to recalculate layout
8. When calling external programs, prefer `EXEC =!"%MyNAME%" ...` or `EXEC* &out=!cmd.exe /c ...`
9. Thread safety: use `&::` class PE variables for inter-thread communication
10. Production code: wrap mutual-exclusion operations in `{ LOCK #pecmd ... }` blocks
11. Window messages on controls use `_` prefix (`_0x0201`); window-level messages omit `_` (`0x0010`)
12. Use `SED` for string manipulation and `MSTR` for field extraction from structured command output


---

GitHub: https://github.com/VirtualHotBar/PECMD-Pro-Max
ClawHub: https://clawhub.ai/virtualhotbar/pecmd-pro-max
