# 磁盘/分区/文件/注册表/设备 — 写法示例

> **版本兼容性：** 部分示例使用 DLL 调用（`CALL $--qd`）。DLL 调用需要 PECMD2012 v1.88+ 完整版支持（32-bit 和 64-bit 行为一致）。
> 如果 DLL 调用不工作，可使用 `PART`、`FDRV`、`REGI`、`EXEC*` 等内置命令替代。

## 1. 磁盘枚举与信息

### 列出所有物理磁盘及详细信息

```wcs
PART list disk,&&全部磁盘
FORX * %&全部磁盘%,&&磁盘,
{
    PART list disk %&磁盘%,&&磁盘信息
    MSTR &&sz,&&bus,&&mbr,&&sign=<2><7><8><9>%&磁盘信息% // ⚠ 字段索引因 PART 输出格式而异
    MESS Disk#%&磁盘%: Size=%&sz% bytes, Bus=%&bus%, Type=%&mbr%
}
```

### 列出磁盘所有分区（兼容 GPT + MBR）

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

### 盘符与磁盘号对应关系

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

### 获取卷标和文件系统（带自动挂载回退）

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

## 2. 通过 Win32 API 枚举设备

### 通过 SetupAPI 枚举物理磁盘

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

### 获取磁盘性能计数器（IOCTL_DISK_PERFORMANCE）

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

## 4. 文件与配置操作

### 读取整个文件（文本 / 二进制）

```wcs
READ C:\config.ini,**,&content      // ** = DOS CRLF -> native, *r = raw, * = LF only
GETF# C:\data.bin,0#*,&raw          // binary read
```

### 解析 INI 风格配置文件到变量

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

### 写入文件

```wcs
WRIT C:\output.txt,1,First line           // 1 = 替换第1行
WRIT C:\output.txt,+0,Second line        // +0 = 在末尾追加新行
WRIT C:\output.txt,$0,a=%var%            // $ = 展开环境变量，0 = 替换最后一行
PUTF -dd -len=0 C:\file.bin,0,zero        // create/truncate
PUTF C:\file.bin,%offset%,#%&data%        // write at offset
```

### 获取文件时间戳和大小

```wcs
SIZE &&sz=C:\file.txt
MESS Size: %&sz% bytes
```

---

## 5. 注册表操作

### 读/写（所有类型）

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
REGI --ak HKCU\Software\,&&keys              // enumerate subkeys (--ak)
REGI --av HKCU\Software\MyApp\,&vals         // enumerate all values (--av)
```

---

## 20. PART 操作（完整工具包模式）

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

## 24. 跨盘符目录搜索（FORX @）

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

## 25. 离线注册表（完整增删改查）

PE 系统部署的关键技术。使用完整的 offreg.dll API 对离线 Windows 注册表配置单元进行创建/读取/写入/枚举操作。

```wcs
// --- Open offline hive ---
SET &hHive=
CALL $--qd --ret:&&ret offreg.dll,OROpenHive,$%&HiveFile%,*&hHive
IFEX $%&ret%<>0, TEAM MESS Failed to open hive@错误#OK| EXIT

// Alternative: ORLoadHive (loads with NT version context)
CALL $--qd --ret:&bret offreg.dll,ORLoadHive,$%&hivepath%,*&hHive

// --- Open or create key ---
SET &hKey=
CALL $--qd --ret:&&ret offreg.dll,OROpenKey,#%&hHive%,$%&SubKey%,*&hKey
IFEX #%&ret%<>0,
{
    CALL $--qd --ret:&&ret offreg.dll,ORCreateKey,#%&hHive%,$%&SubKey%,#0,$,#0,*&hKey,#0
}

// --- Read REG_SZ (type 1, two-call buffer) ---
SET$# &pdwType=*4 0
SET$# &pcbData=*%&PtrSz% 0
SET-long &pcbData=8192:0
CALL $--qd --ret:&&ret offreg.dll,ORGetValue,#%&hKey%,#0,#0,$%&Value%,#1,#0,*&pcbData
SET?int &pcbData=&&cbData:0
SET$# &Data=*%&cbData% 0
CALL $--qd --ret:&&ret offreg.dll,ORGetValue,#%&hKey%,#0,#0,$%&Value%,#1,*&Data,*&pcbData

// --- Read REG_DWORD (type 4) ---
SET$# &dwData=*4 0
SET$# &lpcbData=*4 4
CALL $--qd --ret:&bret offreg.dll,ORGetValue,#%&hKey%,#0,#0,$%&Value%,#4,*&dwData,*&lpcbData
SET?int &dwData=&&dwVal:0

// --- Read REG_QWORD (type 11) ---
SET$# &qwData=*8 0
SET$# &lpcbData=*8 8
CALL $--qd --ret:&bret offreg.dll,ORGetValue,#%&hKey%,#0,#0,$%&Value%,#11,*&qwData,*&lpcbData
SET?longlong &qwData=&&qwVal:0

// --- Write value ---
CALL $--qd --ret:&&ret offreg.dll,ORSetValue,#%&hKey%,$%&Value%,#1,*&Data,#%&DataSize%

// --- Enumerate subkeys ---
SET$# &keyCount=*4 0
SET$# &maxSubKeyLen=*4 0
SET$# &valCount=*4 0
CALL $--qd --ret:&&ret offreg.dll,ORQueryInfoKey,#%&hKey%,#0,#0,*&keyCount,*&maxSubKeyLen,#0,*&valCount,#0,#0,#0,#0
SET?int &keyCount=&&nKeys:0
SET?int &maxSubKeyLen=&&maxLen:0
CALC &nameBufSz=%&maxLen%*2+2
SET$# &lpName=*%&nameBufSz% 0
SET$# &lpcchName=*4 0
SET &i=0
LOOP #%&i%<%&nKeys%,
{
    SET-long &lpcchName=%&nameBufSz%:0
    CALL $--qd --ret:&&ret offreg.dll,OREnumKey,#%&hKey%,#%&i%,*&lpName,*&lpcchName,#0,#0,#0
    SET-make &&subName=&lpName;0
    CALC &i=%&i%+1
}

// --- Enumerate values ---
CALL $--qd --ret:&&ret offreg.dll,ORQueryInfoKey,#%&hKey%,#0,#0,#0,#0,#0,*&valCount,#0,#0,#0,#0
SET?int &valCount=&&nVals:0

// --- Save and close ---
CALL $--qd --ret:&&ret offreg.dll,ORSaveHive,#%&hHive%,$%&HiveFile%,$0,$0
CALL $--qd offreg.dll,ORCloseKey,#%&hKey%
CALL $--qd offreg.dll,ORCloseHive,#%&hHive%
```

---

## 27. BROW — 文件/目录浏览对话框

```wcs
BROW &saveFile,&%Desktop%\output.iso,Save ISO file,iso           // save dialog
BROW &openFile,,Select a file,INI|*.INI|All Files|*.*|           // open dialog with filter
BROW &folder,*C:\,Select a folder                                // folder browser (* prefix)
// Additional flags: 0x10=edit box; 0x200=multi-select(文件对话框) / 无新建文件夹按钮(目录对话框)
```

---

## 28. SUBJ — 挂载/卸载盘符

```wcs
SUBJ -X:                                            // remove drive letter X:
SUBJ G:,\Device\HarddiskVolume3                     // mount volume as G:
```

---

### 40. EFI 启动项管理（FVAR UEFI NVRAM）

```wcs
// Read EFI firmware variable
ENVI ?&var=FVAR,Boot0000;{8be4df61-93ca-11d2-aa0d-00e098032b8c}

// Write EFI firmware variable
ENVI ?-v =FVAR+,BootXXXX,&&buf   // create/modify boot entry

// Get buffer byte length
ENVI-addr ;&len=&&buf

// Create view into buffer at offset (FVAR data starts at offset)
ENVI-mkdummy &&view=&&buf@8

// 16-bit WORD access for BootOrder entries
SET?short &buf=&val:offset

// Need SeSystemEnvironmentPrivilege to write
CALL $--qd --ret:&bret ntdll.dll,RtlAdjustPrivilege,#22,#1,#0,&&pEnabled

// Boot entry ID naming: Boot%IDXX% where IDXX = BootID + 0x100000
// Clean empty IDs, compact BootOrder array
```

---

### 41. SSD 检测（寻道惩罚查询）

IOCTL 计算公式：`CALC &ioctl = shl(0x2D,16) | shl(0,14) | shl(0x09,2) | 0` → 0x2D1400

```wcs
SET &STORAGE_PROPERTY_QUERY_Unknown=0
SET &StorageDeviceSeekPenaltyProperty=7
SET &STORAGE_PROPERTY_QUERY.INPUT=12  // PropertyId(4)+QueryType(4)+AdditionalParams(4)
ENVI$ &&input=*0xC 0
SET-long &&input=7:0       // PropertyId=7 (SeekPenalty)
SET-long &&input=0:4       // QueryType=0 (Standard)
SET-long &&input=0:8       // AdditionalParams=0
SET &output.SIZE=12         // DEVICE_SEEK_PENALTY_DESCRIPTOR: Version(4)+Size(4)+IncursSeekPenalty(1)+3pad
ENVI$ &&output=*0xC 0
SET$# &&dwSize=*4 0
CALL $--qd --ret:&bret kernel32.dll,DeviceIoControl,%&hdisk%,#0x2D1400,*&input,#0xC,*&output,#0xC,*&dwSize,#0
SET?int &&output=&&IncursSeekPenalty:8
// 0=SSD (no seek penalty), 1=HDD
```

---


### 42. TRIM 支持检测

相同的 IOCTL 0x2D1400，PropertyId=8（StorageDeviceTrimProperty）。

```wcs
SET &StorageDeviceTrimProperty=8
SET-long &&input=8:0       // PropertyId=8
SET-long &&input=0:4       // QueryType
SET-long &&input=0:8       // AdditionalParams
CALL $--qd --ret:&bret kernel32.dll,DeviceIoControl,%&hdisk%,#0x2D1400,*&input,#0xC,*&output,#0x20,*&dwSize,#0
// DEVICE_TRIM_DESCRIPTOR: Version(4)+Size(4)+TrimEnabled(1)
SET?int &&output=&&TrimEnabled:8
```

---

### 43. STORAGE_GET_DEVICE_NUMBER（路径 → 磁盘/分区映射）

```wcs
CALC &IOCTL_STORAGE_GET_DEVICE_NUMBER = shl(0x2D,16) | shl(0,14) | shl(0x0420,2) | 0
// 0x2D1080
SET &STORAGE_DEVICE_NUMBER.SIZE=12  // DeviceType(4)+DeviceNumber(4)+PartitionNumber(4)
ENVI$ &&output=*0xC 0
SET$# &&dwSize=*4 0
CALL $--qd --ret:&bret kernel32.dll,DeviceIoControl,%&hdisk%,#%&IOCTL_STORAGE_GET_DEVICE_NUMBER%,#0,#0,*&output,#0xC,*&dwSize,#0
SET?long &&output=&&DeviceType:0
SET?long &&output=&&DeviceNumber:4
SET?long &&output=&&PartitionNumber:8
```

通过设备路径打开：`\\.\C:` → 返回分区信息。通过 `\\.\PhysicalDrive0` 打开 → 返回磁盘信息（PartitionNumber=0）。

---


### 44. Drive Layout Information EX（完整磁盘布局）

```wcs
CALC &IOCTL_DISK_GET_DRIVE_LAYOUT_EX = shl(0x07,16) | shl(0,14) | shl(0x14,2) | 0  // 0x70050
SET &outBufSz=0x1000                               // 4096 bytes (足够容纳布局+分区表)
ENVI$ &&output=*8M 0
SET$# &&dwSize=*4 0
CALL $--qd --ret:&bret kernel32.dll,DeviceIoControl,%&hdisk%,#%&IOCTL_DISK_GET_DRIVE_LAYOUT_EX%,#0,#0,*&output,#%&outBufSz%,*&dwSize,#0

// MBR:  PartitionStyle=0, Header at offset 48: Signature(4)+CheckSum(4)
// GPT:  PartitionStyle=1, Header at offset 48: DiskId(16)+StartingUsableOffset(8)+UsableLength(8)+MaxPartitionCount(4)
// Then partition entries at offset 112, each 120 bytes (PARTITION_INFORMATION_EX)
```

完整 MBR 类型表（26 项）：0x00=空→0xEF=EFI 系统分区，包含 0x07=NTFS、0x0B/0x0C=FAT32、0x0E/0x0F=EFI FAT、0x27=Windows RE

完整 GPT 类型 GUID 表（23 项）：包含 EBD0A0A2（MS 基本数据）、C12A7328（EFI 系统）、E3C9E316（MSR）、DE94BBA4（恢复）

---


### 47. SITE fattr — 文件属性位掩码解码

```wcs
SITE ?,,,,var=fattr,"C:\Windows\notepad.exe"
CALC &isReadOnly=%&var% & 0x1
CALC &isHidden=%&var% & 0x2
CALC &isSystem=%&var% & 0x4
CALC &isArchive=%&var% & 0x20
CALC &isCompressed=%&var% & 0x800
// FILE_ATTRIBUTE_ constants: 0x1=READONLY, 0x2=HIDDEN, 0x4=SYSTEM,
//   0x10=DIRECTORY, 0x20=ARCHIVE, 0x80=NORMAL, 0x100=TEMPORARY,
//   0x200=SPARSE, 0x400=REPARSE, 0x800=COMPRESSED, 0x1000=OFFLINE,
//   0x2000=NOT_CONTENT_INDEXED, 0x4000=ENCRYPTED, 0x8000=INTEGRITY_STREAM,
//   0x10000=VIRTUAL, 0x20000=NO_SCRUB_DATA, 0x40000=EA, 0x80000=PINNED,
//   0x100000=UNPINNED, 0x80000000=DEVICE
```

---


### 56. 通用 IOCTL 代码构造

任意 IOCTL 控制码的通用计算公式：
```
IOCTL = shl(DeviceType, 16) | shl(Access, 14) | shl(Function, 2) | Method
```
其中：
- DeviceType：FILE_DEVICE_ 前缀值（例如存储设备为 0x2D）
- Access：FILE_READ_ACCESS=0、FILE_WRITE_ACCESS=1、FILE_ANY_ACCESS=0
- Function：操作专用编号
- Method：METHOD_BUFFERED=0、METHOD_IN_DIRECT=1、METHOD_OUT_DIRECT=2、METHOD_NEITHER=3

| IOCTL Constant | DeviceType | Access | Function | Method | Result |
|---|---|---|---|---|---|
| IOCTL_STORAGE_QUERY_PROPERTY | 0x2D | 0 | 0x0500 | 0 | 0x2D1400 |
| IOCTL_DISK_GET_DRIVE_GEOMETRY_EX | 0x07 | 0 | 0x28 | 0 | 0x700A0 |
| IOCTL_DISK_GET_DRIVE_LAYOUT_EX | 0x07 | 0 | 0x14 | 0 | 0x70050 |
| IOCTL_DISK_GET_PARTITION_INFO_EX | 0x07 | 0 | 0x12 | 0 | 0x70048 |
| IOCTL_STORAGE_GET_DEVICE_NUMBER | 0x2D | 0 | 0x0420 | 0 | 0x2D1080 |
| IOCTL_DISK_PERFORMANCE | 0x07 | 0 | 0x08 | 0 | 0x70020 |
| IOCTL_DISK_UPDATE_PROPERTIES | 0x07 | 0 | 0x40 | 0 | 0x70100 |

---

## 74. 注册表 键/值/数据 存在性检查

```wcs
// Check if KEY exists:
REGI ?HKCU\Software\MicrosoftXxX\,&&VT
FIND $%&VT%=ERROR, MESS KEY does not exist!  MESS KEY exists

// Check if VALUE exists (key must exist):
REGI ?HKCU\Software\Microsoft\ABC,&&VT
FIND $%&VT%=ERROR, MESS Value does not exist!  MESS Value exists

// Check if DATA exists:
REGI ?HKCU\Software\Microsoft\,&&VT
FIND $%&VT%=ERROR,! { MESS KEY does not exist }
FIND $%&VT%<>ERROR,
{
    FIND $%&VT%=NI, MESS DATA does not exist!  MESS DATA exists
}
```

---

## 75. 文件与目录检测

```wcs
FDIR --fullfile &&F=%&NAME1%
IFEX %&F%,   SET &bfile=1                           // file or directory exists
IFEX %&F%\,  SET &bfile=0                           // trailing slash = it's a directory
FIND *=NAME1, SET &bfile=0                           // empty input check
```

---

