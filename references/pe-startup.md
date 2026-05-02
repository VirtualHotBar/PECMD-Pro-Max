# PECMD in WinPE Startup

## The Boot Flow

In a typical WinPE environment, PECMD is the first user-mode program launched after the kernel and drivers initialize. It replaces the normal Windows shell with a script-controlled environment.

```
Windows Boot Manager (bootmgr / bootmgfw.efi)
    -> winload.exe / winload.efi (kernel loader)
        -> ntoskrnl.exe (kernel)
            -> smss.exe (session manager)
                -> Winlogon / WinPE-shell
                    -> PECMD.EXE MAIN X:\Windows\System32\PECMD.INI
```

## PECMD.INI — The Standard Entry Point

`MAIN` is the standard PE entry command. It loads a configuration file AND starts the Windows message loop (necessary for GUI to work).

```wcs
// PECMD.INI — typical WinPE startup configuration
#code=936T950

// 1. Show logo / splash
LOGO %CurDir%\splash.jpg
TEXT System is initializing, please wait...#0xFFFFFF L20T20 $20

// 2. Initialize user interface
INIT IU,3000

// 3. Load shell
SHEL %SystemRoot%\explorer.exe

// 4. Initialize drivers
DEVI %CurDir%\Drivers\*.inf

// 5. Create desktop shortcuts
LINK %Desktop%\Command Prompt,%SystemRoot%\system32\cmd.exe
LINK %Desktop%\Notepad,%SystemRoot%\system32\notepad.exe

// 6. Set environment
ENVI $TEMP=%SystemDrive%\TEMP
ENVI $TMP=%SystemDrive%\TEMP

// 7. Register hotkeys
HKEY Ctrl+Alt+#0x44, EXEC cmd.exe       // Ctrl+Alt+D -> command prompt

// 8. Load external tools
LOAD %CurDir%\Tools\Network.ini
LOAD %CurDir%\Tools\DiskTools.ini

// 9. Execute startup programs
EXEC %SystemRoot%\system32\cmd.exe /c start /b PECMD.EXE TEAM WAIT 5000|LOAD %CurDir%\PostInit.ini

// 10. Clean up logo
LOGO

// 11. Wait indefinitely (keep PE alive)
WAIT -1
```

## Key Startup Commands

### INIT — Initialize PECMD runtime

```wcs
INIT [options],[timeout_ms]
```

`INIT IU,3000` — most common form. I=keyboard layout, U=USB initialization, 3000ms timeout.

`INIT CIK` — C=disable Ctrl+Alt+Del, I=keyboard, K=kill explorer

### SHEL — Set Windows Shell

```wcs
SHEL %SystemRoot%\explorer.exe         // use Explorer as shell
SHEL PECMD.EXE LOAD %CurDir%\MyShell.ini  // use PECMD script as shell
```

The command(s) indented on the following line are executed when the shell changes:
```wcs
SHEL %SystemRoot%\explorer.exe
    TEAM KILL Explorer.exe| KILL Explorer.exe
```

### LOGO — Show/hide splash screen

```wcs
LOGO %CurDir%\logo.jpg           // show splash image
LOGO                             // hide splash
```

### TEXT — Display status text

```wcs
TEXT Initializing system...#0x00FF00 L100T200 R600B400 $18:Microsoft YaHei
```

Format: `TEXT text[#color][LleftTtop][RrightBbottom][$fontSize[:fontName]]`

### DEVI — Install drivers

```wcs
DEVI %CurDir%\Drivers\NetCard.cab     // install from CAB
DEVI %CurDir%\Drivers\*.inf            // install from INF files
DEVI $%CurDir%\Drivers                 // install all drivers in directory
```

### LINK — Create shortcuts

```wcs
LINK %Desktop%\MyTool,%CurDir%\mytool.exe
LINK %StartMenu%\Tools\PartEditor,%CurDir%\part.exe,,%CurDir%\part.ico
```

## Common WinPE Patterns

### Pattern: Wait for removable drives then load tools

```wcs
_SUB WaitForUSB
    LOOP #1=1,
    {
        FDRV &drvs=*:
        FORX * %&drvs%,&&drv,
        {
            FORM -raw &&type=%&drv%
            FIND $DRIVE_USBDISK=%&type%,
            {
                IFEX %&drv%\PETOOLS\LOAD.INI, TEAM LOAD %&drv%\PETOOLS\LOAD.INI| EXIT _SUB
            }
        }
        WAIT 2000
    }
_END
```

### Pattern: Auto-assign drive letters

```wcs
SHOW -1:-1                           // show all partitions
DISK ,,,1,U:                         // assign USB drives starting from U:
```

### Pattern: Setup virtual memory (pagefile)

```wcs
PAGE C:\pagefile.sys 256 512         // min 256MB, max 512MB on C:
```

### Pattern: Setup temporary directory

```wcs
PATH %SystemDrive%\TEMP
ENVI $TEMP=%SystemDrive%\TEMP
ENVI $TMP=%SystemDrive%\TEMP
```

### Pattern: Mount WIM images for external programs

```wcs
MOUN %CurDir%\Tools.wim,%SystemDrive%\Tools,,1    // mount WIM with TEMP
IFEX %SystemDrive%\Tools\Setup.cmd, EXEC =!"%SystemDrive%\Tools\Setup.cmd"
```

## Minimal PECMD.INI

The absolute minimal PE startup script:

```wcs
#code=936T950
INIT IU
SHEL %SystemRoot%\explorer.exe
WAIT -1
```

## Important Notes

1. `WAIT -1` at the end of PECMD.INI keeps the script running indefinitely (otherwise PE closes immediately after startup)
2. `SHEL` must appear AFTER `INIT` — the shell needs the initialization to complete first
3. In PE, the registry hive may not be fully loaded. Use `REGI .` (dot prefix) for offline registry access
4. `%SystemDrive%` in PE is typically `X:` (the RAM disk), not `C:`
5. PE environments often lack many DLLs. Test your scripts in a real PE or use `IFEX` guards
6. The `%CurDir%` variable points to the directory containing PECMD.INI, making it ideal for relative paths
7. Use `LOGS * C:\pecmd.log` at the top of PECMD.INI for debugging startup issues

## PE Environment Limitations

PE environments have inherent constraints. Understanding these is critical for writing robust PECMD scripts.

### 1. Missing Runtimes
**Constraint**: VC++ redistributables and .NET Framework are not installed.
**Mitigation**: Use static-linked PECMD tools (no external DLL dependencies). Avoid calling programs that require MSVC runtime DLLs unless you bundle them.

### 2. Read-Only Media
**Constraint**: The `X:\` sources root is a RAM disk that cannot be written to (filesystem overlay). The boot media itself (CD/DVD, USB) may be read-only.
**Mitigation**: Use `%TEMP%`, `%SystemDrive%\TEMP`, or a writable partition for scratch files. Never attempt to write to `X:\Windows\System32\` or the boot media root.

### 3. No Network by Default
**Constraint**: Network adapters are not initialized and DHCP is not configured.
**Mitigation**: Explicitly initialize networking with `PCIP` command before any network operations. For wireless, use `ADSL` or `UPNP` as appropriate.

### 4. Temporary Registry
**Constraint**: The registry is loaded into RAM and changes are lost on reboot. The SYSTEM and SOFTWARE hives are loaded from the WIM and are read-only overlays.
**Mitigation**: Save persistent settings to `HKCU` (which maps to a writable hive) or use offline HIVE manipulation (`REGI .`) for persistent changes. Use `%Desktop%` or `%TEMP%` directory for state files.

### 5. Single-User SYSTEM Account
**Constraint**: PE runs as the SYSTEM account with no user profiles, no `%USERPROFILE%` directory in the traditional sense, and no user-specific HKCU hive by default.
**Mitigation**: Use `%TEMP%` for scratch data. Create a writable user profile directory manually: `PATH X:\Users\Default` and `ENVI $USERPROFILE=X:\Users\Default` before `INIT`.

### 6. Missing Drivers
**Constraint**: Storage controllers, network adapters, and chipset drivers may not be included in the base PE image.
**Mitigation**: Use `DEVI` to inject required drivers at startup. For storage controllers that hold the boot media, drivers must be integrated into the WIM itself (via DISM) before boot.

### 7. Uncertain Drive Letters
**Constraint**: Drive letter assignment is not deterministic. The USB boot drive may be `C:`, `D:`, or any other letter rather than the expected `U:`.
**Mitigation**: Use `FORX @\` with a unique tag file to locate the correct drive. For example: `FORX @\MyPETools.tag,&&usbDrv,1` — then use `%&usbDrv%` as the base path. The `@\` prefix searches all drives from C: to Z:.

### 8. Writable USERPROFILE Required
**Constraint**: Many Windows APIs and shell components require a valid writable `%USERPROFILE%` path. Without it, Explorer may fail to launch or behave erratically.
**Mitigation**: Set `ENVI $USERPROFILE=X:\Users\Default` and ensure the directory exists (`PATH X:\Users\Default`) before calling `INIT` or `SHEL`. This is a common source of "Explorer doesn't start" bugs.

## PE Version Differences

### WinPE 3.x (Windows 7 Kernel)
- Based on Windows 7 / Server 2008 R2 kernel (NT 6.1)
- **Features**: MBR/GPT partitioning, basic DISM support, VHD boot
- **Limitations**: No DPI scaling, limited USB 3.0 support, WIM mounting uses `wimgapi.dll` (user-mode, slower, requires temporary space)
- **PECMD notes**: `INIT U` for USB is essential; many modern storage drivers must be injected via `DEVI`

### WinPE 5.x (Windows 8.1 Kernel)
- Based on Windows 8.1 / Server 2012 R2 kernel (NT 6.3)
- **Features**: Improved DISM (faster, more commands), native USB 3.0, better SSD support
- **WIM mounting**: Still uses `wimgapi.dll` by default; `wimmount.sys` (kernel-mode, faster) available as optional
- **PECMD notes**: DPI scaling support begins (`-scale` flag works); `PART -super -up` works for GPT partition type changes

### WinPE 10.x (Windows 10/11 Kernel)
- Based on Windows 10/11 kernel (NT 10.0)
- **Features**: Full modern driver support, network auto-configuration (Wi-Fi profiles), NVMe native support, exFAT boot support
- **WIM mounting**: `wimmount.sys` (kernel-mode) is default — faster and uses less RAM than `wimgapi.dll`
- **DPI scaling**: Full support via `-scale[:DPI]` flag; `-scalef` for XP-style fallback
- **UEFI Secure Boot**: Fully supported; PECMD scripts can run in Secure Boot-enabled environments
- **PECMD notes**: `INIT` auto-detects most hardware; Wi-Fi can be set up with `ADSL WLAN` or native `netsh wlan`

### Key Differences Summary

| Feature | WinPE 3.x | WinPE 5.x | WinPE 10.x |
|---------|-----------|-----------|------------|
| WIM Mounting | wimgapi.dll | wimgapi + optional wimmount.sys | wimmount.sys (default) |
| DPI Scaling | None | Basic (`-scale`) | Full (`-scale[:DPI]`, `-scalef`) |
| USB 3.0 | Requires driver injection | Native | Native |
| NVMe | Not supported | Limited | Native |
| UEFI Secure Boot | Unreliable | Working | Fully supported |
| Network | Manual only | Manual + basic auto | Full auto-config |
| exFAT Boot | No | No | Yes |
