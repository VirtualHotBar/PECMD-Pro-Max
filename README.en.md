# PECMD Pro Max

> AI coding assistant skill for PECMD2012 scripting — write correct WinPE tools, GUIs, and boot scripts on the first try.

[简体中文](./README.md) | English

[![Version](https://img.shields.io/badge/version-1.5.1-blue)](./SKILL.md)
[![License](https://img.shields.io/badge/license-NonCopyRight-yellow)](./LICENSE)
[![PECMD](https://img.shields.io/badge/PECMD-v1.88+-green)](https://pecmd.net)

## What is this?

PECMD Pro Max is a **Claude Code skill** that teaches AI coding assistants to write correct PECMD2012 scripts. PECMD is the WinPE Commander — a scripting language and command interpreter used for Windows PE boot scripts, lightweight GUI system tools, disk utilities, and pre-install environment automation. Its variable scope rules, hex memory model, and window system are notoriously tricky. This skill encodes 10+ years of PECMD scripting knowledge so your AI assistant can write production-ready `.wcs` / `.wci` / `.wce` files without the usual trial-and-error.

## File structure

```
pecmd-pro-max/
├── README.md                        ← Simplified Chinese
├── README.en.md                     ← You're reading it
├── SKILL.md                         ← Main skill file — mental model, variable system, critical rules, traps (561 lines)
└── references/
    ├── commands-full.md             ← Complete 110+ command reference (1943 lines)
    ├── pecmd-gui.md                 ← Comprehensive GUI controls & window system reference (2178 lines)
    ├── pe-startup.md                ← WinPE boot flow, environment limitations, PE version differences
    └── how-tos/
        ├── storage.md               ← Disk/partition/file/registry/device how-tos
        ├── system.md                ← Process/thread/system/utility/callback how-tos
        ├── gui.md                   ← GUI controls/window/drawing how-tos
        └── net.md                   ← Network/SOCK/COM/WMI how-tos
```

## Key features

- **Three-tier variable system explained** — environment vs PE-local vs PE-global, indirect dereference, deferral, return-by-reference, hex/raw-buffer allocation, binary compare/convert
- **110+ command reference** — every PECMD command with full syntax, return values, gotchas (incl. BROW, TREE, LAMBDA, SBAR, IPAD, BLOCK, DTIM, SLID, SPIN)
- **Domain-organized code how-tos** — disk/partition, process/thread, GUI controls/drawing, network/COM, in separate files for on-demand loading
- **Complete GUI reference** — all 25 controls & sub-windows (`LABE`, `EDIT`, `CHEK`, `RADI`, `IMAG`, `TABL`, `LIST`, `MEMO`, `TREE`...), ENVI @ properties full reference, message system, window management
- **WinPE boot flow** — from `winpeshl.exe` → `PECMD MAIN PECMD.INI` through to `EXPLORER.EXE`, with `WinXShell` integration
- **50 built-in variables table** — path/shell vars, process/thread vars, window/GUI vars, script/runtime vars, command-result vars
- **17 ENVI runtime control commands** — EnviMode, ForceLocal, Alias, WndProc, memvar, LoadPlugin, DisX64, zero, Arg, Clipboard, EnviBroad, __arg, LoadEnvi, HelpColor, DeskTopFresh(@@), TaskIcoMenu(@@), EXPORTLOCAL
- **ForceLocal + EnviMode defaults** — ensures every generated script uses the safest variable scoping by default
- **Code organization rules** — `_SUB` declaration at parse time vs runtime, `SET^` Win32 callback binding, `CALL @` variants cheat sheet
- **ENVI @ control properties reference** — universal, EDIT/ITEM/window-specific, TABL 30+ operations, TREE 10+ operations, cross-process control

## Quick start

```wcs
#code=65001
ENVI^ EnviMode=1
ENVI^ ForceLocal=1
SET$ &NL=0d 0a

_SUB MainWin,L200T100W400H200,Hello World,-trap
    LABE Lbl1,L20T30W350H24,Welcome to PECMD
    ITEM Btn1,L150T120W80H28,Close,CALL CloseMe
_END

_SUB CloseMe
    KILL \
_END

CALL @MainWin
```

## Installation

Copy the `pecmd-pro-max/` folder into your AI coding assistant's skills directory:

- **Claude Code**: `~/.claude/skills/pecmd-pro-max/`
- **OpenCode**: `~/.config/opencode/skills/pecmd-pro-max/`

## Requirements

- [Claude Code](https://claude.ai), [OpenCode](https://opencode.ai), or any compatible AI coding tool that supports agent skills
- [PECMD2012](https://pecmd.net) v1.88+ (the interpreter that runs the generated scripts)

## License

**NonCopyRight** — this skill is free, open source, and unrestricted. Use it, modify it, ship products with it. No attribution required. No restrictions apply.

---

Source repository: [github.com/VirtualHotBar/PECMD-Pro-Max](https://github.com/VirtualHotBar/PECMD-Pro-Max)

ClawHub: [clawhub.ai/virtualhotbar/pecmd-pro-max](https://clawhub.ai/virtualhotbar/pecmd-pro-max)