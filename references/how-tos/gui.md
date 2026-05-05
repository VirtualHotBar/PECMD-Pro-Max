# GUI 控件/窗口/绘制 — 写法示例

> **版本兼容性：** 部分示例使用 DLL 调用（`CALL $--qd`）。DLL 调用需要 PECMD2012 v1.88+ 完整版支持（32-bit 和 64-bit 行为一致）。
> 如果 DLL 调用不工作，可使用 `FIND --wid*@`、`EXEC*` 等内置命令替代。

## 7. GUI 模式

### 完整窗口模板

```wcs
#code=65001
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

### DPI 感知窗口与手动布局

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

### TABL 数据操作（综合）

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

### LIST 下拉操作

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

### 右键弹出上下文菜单

```wcs
ENVI @Table1.MSG=_%&::WM_RBUTTONDOWN%: CALL @--popmenu MyMenu

_SUB MyMenu
    MENU MnuOpenLoc,打开文件位置,TEAM ENVI @Table1.Sel=?&s| CALL FileDir
    MENU -
    MENU 复制,复制到剪贴板,CALL Clipboard
    MENU -
    MENU 结束进程,结束进程,CALL KillProc
_END
```

### SWIN 子窗口（选项卡页）

```wcs
SWIN :Page1,L10T40W380H200
SWIN :Page2,L10T40W380H200
ENVI @Page2.Visible=0                    // hidden initially

_SUB Page1
    ENVI @this.bkcolor=0xFFFFFF
    LABE ,L10T10W200H20,This is page 1
_END
// Switch: ENVI @Page1.Visible=0 | ENVI @Page2.Visible=1
```

### 动态标签创建（变量中的命令）

```wcs
ENVI &&cmdString=LABE -vcenter -trans Lbl%&i%,L%&x%T%&y%W%&w%H%&h%,%&text%,CALL OnClick %&i%,0x000000,14
%&cmdString%                                      // execute the dynamically-built command
```

### 带关联控件的复选框

```wcs
CHEK Check1,L20T50W200H20,Auto refresh,CALL OnCheck,1

_SUB OnCheck
    IFEX $%Check1.Check%=1, ENVI @Timer1=1000! ENVI @Timer1=0
_END
```


### 动态控件创建与删除

```wcs
// Create controls programmatically from a command string in a variable
SET &x=100 & &y=50 & &w=200 & &h=30
ENVI &&cmd=LABE -vcenter -trans Lbl%&i%,L%x%T%y%W%w%H%h%,%&text%,,0x000000,14
%&cmd%                                              // execute the command to create the control

// Delete controls dynamically
ENVI @Lbl%A.*del=                                   // delete label A
ENVI @Edit%B.*del=                                  // delete edit field B
// This is essential for dynamic GUIs that rebuild control sets
```
---

## 26. 窗口样式标志与窗口隐藏

```wcs
// Common window creation flags
_SUB MyWin,W400H300,Title,,,,, -trap -nocap -ntab -nfocus
//   -trap: close button doesn't exit (window survives close)
//   -nocap: no title bar (frameless window)
//   -ntab: no Tab key navigation (no keyboard focus cycling)
//   -nfocus: no keyboard focus on creation
//   -nosysmenu: no system menu (no icon, no min/max/close)
//   -top: always on top (TOPMOST)
//   -maxb: enable maximize button
//   -minb: enable minimize button

// Cross-process window hide/show
ENVI @@Visible=%&WID%:0                           // 不可见
ENVI @@Visible=%&WID%:1                           // 可见
ENVI @@Visible=%&WID%:2                           // 正常显示
ENVI @@Visible=%&WID%:3                           // 最大化
ENVI @@Visible=%&WID%:*4                          // 最小化（* = 第2种方案）
ENVI @@Visible=%&WID%:5                           // 恢复
ENVI @@Visible=?%&WID%:&&state                    // query visibility state
```

---

## 27. @this. 自引用与窗口属性

`@this.` 引用当前窗口自身，无需知道窗口 ID。SDK 示例广泛使用。

```wcs
_SUB MyWin,W400H300,Test,
    // --- 属性设置（@this = 当前窗口）---
    ENVI @this.Font=12:Microsoft YaHei
    ENVI @this.bkcolor=0xF0F0F0
    ENVI @this.Cursor=32649                         // 手型光标 (IDC_HAND)
    ENVI @this.trans=1*                             // 背景透明（*=透明色模式）
    ENVI @this.trans=0x2                            // 完全透明
    ENVI @this.style=0x00040000:0x00C00000          // 去掉WS_SIZEBOX,加上WS_CAPTION
    ENVI @this.nxp=                                  // 禁用 XP 视觉样式
    ENVI @this.Paint=OnPaint                        // WM_PAINT 回调
    ENVI @this.HitTest=20                           // 顶部20像素可拖动（高=20）
    ENVI @this.HitTest=-20                          // 半透明穿透拖动

    // --- 控件属性 ---
    ENVI @Edit1.ReadOnly=1                          // EDIT 只读
    ENVI @Edit1.LINE=10                             // EDIT/MEMO 滚动到第10行
    ENVI @this.MouseCapture=1                       // 捕获鼠标（0=释放）

    // --- 动态命令 ---
    ENVI @this.cmd=CALL MyHandler                   // 设置窗口响应命令
    ENVI @this.cmd?=&currentCmd                     // 查询当前命令（需先 ENVI^ QueryCmd=1）

    // --- 跨进程查询 ---
    ENVI @@Pos=?%&WID%:&L:&T:&W:&H                 // 查询位置
    ENVI @@Pos=?%&WID%:&L:&T:&W:&H:&SX:&SY::&Z    // 扩展：含屏幕坐标和层叠序

    // --- 完整 POS 设置语法 ---
    // ENVI @@POS=wid:L:T:W:H:Z:trans:front:activate
    // Z 值: 1=底部, 2=移除置顶, 3=置顶, 4=始终置顶(钉住)
    // trans: 0-255 透明度
    // front: 1=前台激活
    // activate: 0=不改变焦点
    // @前缀: 绝对屏幕坐标; 无@: 客户区坐标
    ENVI @@POS=%&WID%:100:100:400:300:3:$200       // 置顶+半透明（$=0-255格式透明度）
    ENVI @this.POS=100:100:400:300                  // 设置位置（相对）
    ENVI @this.POS=?;&L;&T;&W;&H                   // 查询位置（分号分隔）
    ENVI @@IsWindow=?%&WID%:&valid                  // 检查窗口是否有效
    ENVI @@Enable=?%&WID%:&enabled                  // 查询启用状态

    // --- 跨进程消息发送 ---
    ENVI @@SENDMSG=%&hwnd%:#0x0010;0;0             // 同步发送 WM_CLOSE
    ENVI @@POSTMSG=%&hwnd%:#1;0;0                  // 异步投递自定义消息 #1
    ENVI @this.MSG=0x1000: CALL OnMouseEnter       // WM_MOUSEENTER (PECMD 自定义 0x1000)
```

---

## 28. 缺少的控件类型（DTIM/IPAD/SLID/SBAR/GROU）

### DTIM — 日期时间选择器

```wcs
DTIM DTIM1,L10T10W200H25,初始值,命令,类型
// 初始值: "2008;5;12" 等年;月;日格式（分号分隔）
// 查询值（最多5个：年月日周标志 或 时分秒标志）:
ENVI @DTIM1.VAL=?&&year;&&month;&&day;&&weekday;&&flag
// 设置值（3字段：年;月;日 或 时;分;秒）:
ENVI @DTIM1.VAL=2024;1;15
// 鼠标悬停/离开消息:
ENVI @DTIM1.MSG=0x1000: CALL OnMouseEnter   // WM_MOUSEENTER
ENVI @DTIM1.MSG=0x1001: CALL OnMouseLeave   // WM_MOUSELEAVE
```

### IPAD — IP 地址编辑器

```wcs
IPAD IPAD1,L10T10W200H25,,命令,状态
// 查询 IP:
ENVI @IPAD1.VAL=?&&ip1;&&ip2;&&ip3;&&ip4        // 无前缀点：分别返回4段
ENVI @IPAD1.VAL=?.FullIP                         // 带前缀点：返回完整IP字符串
// 设置 IP:
ENVI @IPAD1.VAL=192.168.1.100                    // 点分格式
// 设置焦点:
ENVI @IPAD1.VAL=#3                               // #前缀+段号，聚焦到第3段
// 设置范围:
ENVI @IPAD1.VAL=#3:0:255                         // #段号:最小:最大（段号1-4）
```

### SLID — 滑块控件

```wcs
SLID [-right] [-left] [*] SLID1,形状[,值信息,命令,状态]
// 值信息: [起始值][:终到值][:初值][:页大小]，默认 0:100:0
SLID SLID1,L10T10W200H30,0:100:50:1,CALL OnSlide,0
// 查询/设置值:
ENVI @SLID1.VAL=?&&val                           // 查询当前值
ENVI @SLID1.VAL=75                               // 设置值
```

### SBAR — 滚动条控件

```wcs
SBAR SBAR1,L10T10W200H20,,命令,状态
// 查询/设置值:
ENVI @SBAR1.VAL=?&&pos:&&min:&&max:&&page         // 查询位置、范围和页大小
ENVI @SBAR1.VAL=50:0:100:20                      // 当前值:起始值:终到值:页大小
// 启用/禁用和可见性:
ENVI @SBAR1.Enable=0                             // 禁用
ENVI @SBAR1.Visible=0                            // 隐藏
```

### GROU — 分组面板

```wcs
GROU [-right] [-center] [*] GROU1,形状,[标题],[状态],[前景色#背景色],[字体]
GROU GROU1,L10T10W380H200,设置选项,,0x000000#0xF0F0F0,12
// 状态 ±16 = 不可见（正负均可）
GROU GROU1,L10T10W380H200,高级,,-0x10             // 隐藏的分组
// -right: 标题右对齐, -center: 标题居中
```

---

## 32. SWIN 嵌套窗口（选项卡页）

完整的属性页模式：将每个子窗口定义为 `_SUB`，在父窗口中使用 `SWIN` 嵌入，通过 `TABS.SEL` 在选项卡点击时切换可见页面。

```wcs
#code=65001
ENVI^ EnviMode=1
ENVI^ ForceLocal=1
SET$ &NL=0d 0a

CALL @主窗口

_SUB 主窗口,W500H380,Tabbed Settings,,#1,,
    ENVI @this.Font=12:Microsoft YaHei
    TEAM ENVI &&nTab=0| ENVI &&TabSel=1

    // --- TABS control (page buttons) ---
    TABS TabMain,L10T10W480H40

    // --- SWIN containers for each page ---
    SWIN :PageGeneral,L10T55W480H290
    SWIN :PageNetwork,L10T55W480H290,,0x10         // hidden initially
    SWIN :PageAdvanced,L10T55W480H290,,0x10        // hidden initially

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
    IFEX $%ChkAutoRun.Check%=1, MESS AutoRun: ON
    MESS IP=%&EdtIP%:%&EdtPort%@Info#OK
    KILL \
_END
```

关键点：
- `_SUB` 类定义可在文件中任意位置（PECMD 解析时解析），但 SWIN 实例需引用已定义的类名。
- 使用 `0x10` 标志初始隐藏页面。
- `TABS.SEL=<n>` 选择选项卡；`TABS.SEL=?` 查询当前选择。
- 通过 `ENVI @PageName.Visible=0` / `=1` 切换可见性。

---

## 33. 动态行创建与批量删除

运行时通过变量展开的命令字符串构建控件。在循环中批量删除控件组。

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

关键点：
- `ENVI @CtrlName.*del=` 销毁控件并释放其资源。
- 变量中的命令字符串（`%&cmd%`）是使用动态计算控件名的唯一方法。
- 批量删除循环必须注意在控件已预索引时不要跳过索引。

---

## 36. TABL 通过 LVM 消息控制滚动条

使用 `SENDMSG` 发送列表视图消息以控制滚动。消息作用于底层的 SysListView32 控件。

```wcs
#code=65001
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

注意：`SENDMSG` 发送到最后获得焦点/前台窗口。在 GUI 上下文中，可能需要先聚焦表格或使用 `ENVI @Table1.SENDMSG`（PECMD 2012 扩展）。跨进程或精确控制时使用 `CALL $ user32.dll,SendMessageW`。

---

## 37. TABL 行内排序（冒泡排序）

将所有行读入内存，按目标列冒泡排序，重写表格。

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
                SET &Row[%&i%]=%%&Row[%&j%]%%
                SET &Row[%&j%]=%&tmp%
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

关键点：
- 双百分号 `%%&Row[%&i%]%%` 解引用：第一个 `%%` 求值为 `%`，然后 `%&Row[3]%` 读取变量。
- `FIND $str1>str2` 进行字典序"大于"比较（比较两个字符串，第一个 > 第二个时为真）。数值比较请改用 `|` 前缀。
- 数值比较前用 `SED` 移除非数字字符以提取数值。

---

## 38. 自定义标题栏窗口（无边框 + 手动标题）

无边框窗口，使用 LABE 控件构建仿标题栏。处理最小化、关闭、悬停颜色效果以及通过 `WM_NCHITTEST` 实现的窗口拖动。

```wcs
#code=65001
ENVI^ EnviMode=1
ENVI^ ForceLocal=1

SET &::WM_LBUTTONDOWN=0x0201
SET &::WM_LBUTTONUP=0x0202
SET &::WM_MOUSEMOVE=0x0200
SET &::WM_NCHITTEST=0x0084
SET &::HTCAPTION=2

CALL @CustomWin

_SUB CustomWin,W500H350,My Custom Tool,,#1,, -trap -nocap
    ENVI @this.Font=12:Microsoft YaHei
    ENVI @this.bkcolor=0xF0F0F0

    // --- Fake title bar background ---
    LABE -center TitleBar,L0T0W500H32,,,0xFFFFFF#0x2D2D30

    // --- Fake icon ---
    LABE -center -vcenter LblIcon,L8T4W24H24, ,0xFFFFFF#0x2D2D30#0xFFFFFF#0x3D3D40
    ENVI @LblIcon.MSG=%&::WM_LBUTTONDOWN%: CALL @--popmenu SysMenu

    // --- Title text ---
    LABE -center -vcenter LblTitle,L36T4W360H24,My Custom Tool,,0xFFFFFF#0x2D2D30

    // --- Minimize button ---
    LABE -center -vcenter BtnMin,L412T4W36H24,_,CALL OnMin,0xCCCCCC#0x2D2D30#0xFFFFFF#0x3D3D40

    // --- Close button ---
    LABE -center -vcenter BtnClose,L452T4W40H24,X,CALL OnClose,0xCCCCCC#0x2D2D30#0xFFFFFF#0xE81123

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
    ENVI @@Visible=%&__WinID%:4                      // PECMD @@Visible 常量: 0=隐藏, 1=显示, 2=正常, 3=最大化, 4=最小化, 5=恢复
_END

_SUB OnClose
    KILL \
_END

_SUB OnAbout
    MESS Custom Title Bar Demo v1.0@About#OK
_END
```

关键点：
- `-nocap` 移除系统标题栏；`-trap` 防止关闭按钮自动退出。
- 在 LABE 控件上通过 `WM_NCHITTEST` 返回 `HTCAPTION` 使其可拖动。
- 4段颜色格式 `文本色#背景色#悬停文本色#悬停背景色` 启用悬停效果。
- 按钮字形使用 ASCII 字符（`_` = 最小化、`X` = 关闭），或直接写入 Unicode 字符（UTF-8 编码）。

---

## 64. TREE 控件（层级节点视图）

### 创建带图标和节点层级的树形控件

```wcs
// TREE 格式: TREE [名称],<形状>,[图片数据],[节点数据],[状态]
// 节点数据格式: \图标索引:选择图标索引:文本，0x09分隔节点，0x0B开始子节点，0x0C结束子节点
// 图片数据格式: 表头.[:图标宽:图标高]图标1%TAB%图标2...

// 用 SET$ 构建含控制字符的节点数据
SET$ &TAB=09
SET$ &CHILDBEGIN=0b
SET$ &CHILDEND=0c
SET &MUI_NODE_DATA=\0:0:Root1%&CHILDBEGIN%\0:0:Child1.1%&TAB%\0:0:Child1.2%&CHILDEND%\1:1:Root2%&CHILDBEGIN%\1:1:Child2.1%&CHILDEND%

// 图片数据（可选，不需要图片时留空）
SET &IMAGELIST=icons.16:16%&TAB%icon1.ico%&TAB%icon2.ico

TREE Tree1,L10T10W300H300,%&IMAGELIST%,%&MUI_NODE_DATA%,0x10000127

// Expand / Collapse nodes（格式: 节点;值，值: 1=折叠 2=展开 3=切换）
ENVI @Tree1.Expand=1;2               // expand node 1
ENVI @Tree1.Expand=2.1;1             // collapse node 2.1
ENVI @Tree1.Expand=4;3               // toggle node 4

// Select a node
ENVI @Tree1.Sel=2.2                 // select node 2.2
ENVI @Tree1.Sel=?*2;&&node1         // query selected node path
ENVI @Tree1.Sel=?@&&hnode           // query selected node handle

// Checkbox state
ENVI @Tree1.Check=3.1;2             // toggle (0=unchecked, 1=checked, 2=toggle/乒乓)
ENVI @Tree1.Check=?*;&&state        // query checkbox state
```

### 处理 TVN_ITEMCHANGEDW（复选框变更通知）

```wcs
ENVI @Tree1.ID=?;&&Tree1_ID                        // 获取控件 ID
CALC -base=16 #&&TVN_ITEMCHANGEDW=0x100000000-419
ENVI @this.MSG=NOTIFY#%&Tree1_ID%#%&&TVN_ITEMCHANGEDW%::&&wp,&&lp, CALL OnItemChanged %&&wp% %&&lp%

_SUB OnItemChanged
    // parse NMTREEVIEW struct for changed item
_END
```

---

## 65. TIPS/TIPS* 系统托盘与 TABL 高级操作

### 系统托盘（TIPS/TIPS*）

```wcs
// 创建全局托盘图标（与任务栏共享）
TIPS 托盘名,提示文本,图标路径,命令
TIPS* 托盘名,提示文本,图标路径,命令           // * = 窗口私有托盘图标

// 在窗口 _SUB 中使用
_SUB MyWin,W400H300,Tray App,,#1,,
    TIPS* MyTray,My App,shell32.dll#43,        // 创建私有托盘图标
_END

// 更新托盘提示文本
TIPS* MyTray,新提示文本,,

// 最小化到托盘
_SUB OnMin
    ENVI @this.Visible=0                       // 隐藏窗口
    // 点击托盘图标时恢复：
_END

_SUB DoMenu
    IFEX $%2=%&WM_RBUTTONDOWN%, CALL @--popmenu TrayMenu
    IFEX $%2=%&WM_LBUTTONDOWN%, ENVI @this.Visible=1
_END

_SUB TrayMenu
    MENU 显示窗口,Show,ENVI @this.Visible=1
    MENU -
    MENU 退出,Exit,KILL \
_END
```

### TABL 行内着色与高级操作

```wcs
// 创建带颜色的表格（4 色格式：背景#文字背景#默认文字#选中行）
TABL -color:0x00F000#0x808000#0xF0E0FF#0x80 Table1,L10T10W500H300,%&Title%,,0x10040

// 单元格着色（格式：行.列;;颜色）
ENVI @Table1.Color=3.2;;0xFF                         // 第3行第2列，红色（BGR）
// 行着色（* 前缀）
ENVI @Table1.Color=*5;;0xFFFF00                       // 第5行整行，黄色
// 查询鼠标下单元格
ENVI @Table1.Sel=?.&&Row;&&Col                        // 获取鼠标所在的行.列

// 行内进度条（Percent）
ENVI @Table1.Percent=3.2;75                           // 第3行第2列显示75%进度条
ENVI @Table1.Percent=3.2;50;C:::0xFF0000:Loading      // 红色进度条+文字
// C=居中, R=右对齐, L=左对齐, F=填充, K=块模式

// 行级使能/禁用（~ 前缀）
ENVI @Table1.Enable=~3;0                               // 禁用第3行
ENVI @Table1.Enable=~?*;&disabledRows                  // 查询禁用行

// TABS 动态修改标签
ENVI @TabMain.Title1=新标题                             // 修改第1个标签的标题
ENVI @TabMain.Tip1=新提示                               // 修改第1个标签的提示
ENVI @TabMain.SEL=?&&sel                                // 查询当前选中的标签索引
```

---

## 67. GDI 绘制（WM_PAINT 绘图）

### 将 GDI 函数注册为别名

```wcs
ENVI^ Alias -opt Rectangle=CALL $--qd# --ret:* Gdi32.dll,Rectangle,*dummy,
ENVI^ Alias -opt Ellipse=CALL $--qd# --ret:* Gdi32.dll,Ellipse,*dummy,
ENVI^ Alias -opt Polyline=CALL $--qd# --ret:* Gdi32.dll,Polyline,*dummy,
```

### 带绘制回调和动画的窗口

```wcs
_SUB CanvasWin,W260H320,Canvas Demo,
    ENVI @this.Paint=OnPaint               // set WM_PAINT handler
    SET &aw=2
    SET &w=10
    TIME &Timer1,50, ENVI @this.InvalidateRect=;;;230;
_END

_SUB OnPaint                               // %1=HDC, %2=width, %3=height
    SET &w0=115 & &h0=115                           // 中心点
    CALC #L=%&w0% - %&w%
    CALC #T=%&h0% - %&w%
    CALC #R=%&w0% + %&w%
    CALC #B=%&h0% + %&w%
    Rectangle %1,%L%,%T%,%R%,%B%
    Ellipse %1,%L%,%T%,%R%,%B%
    CALC #&w=%&w% + %&aw%
    IFEX $%&w%>100, TEAM SET &aw=-2| CALC #&w=%&w% + %&aw%!
    IFEX $%&w%<0, TEAM SET &aw=2| CALC #&w=%&w% + %&aw%
_END
```

### 使用 POINT 数组绘制折线

```wcs
SET$ &Pt= 0x0064 0x0000  0x0000 0x0000  0x00C8 0x0000  0x0064 0x00C8  *200 0
ENVI-addr &&PtAddr=&Pt

_SUB OnPaint
    Polyline %1,%&PtAddr%,4
_END
```

---

## 68. PBAR / SPIN 控件（进度条与微调器）

### 带颜色和文字覆盖的进度条

```wcs
PBAR PBAR1,L22T13W200H16,20               // initial value = 20%
ENVI @PBAR1.color=0xFF                     // text color (BGR red)

// Update progress with text overlay (help.txt: 进度[;[#颜色:]文本])
ENVI @PBAR1=50;#00FF00:Processing...       // 50%, green text
ENVI @PBAR1=%&p%;%&K%s  %&p%%%            // value;text
```

### 绑定到 EDIT 的 SPIN 微调控件

```wcs
EDIT EDIT1,L28T14W158H29,0,,
SPIN SPIN1,L192T13W22H30,EDIT1:-20:5,&&npos:&&button:&&old,
    ENVI @LABE3= SPIN1 [%&&npos%] [%&&button%] [%&&old%], 0xA0

// Query value and range
ENVI @SPIN1.VAL=?&&POS:&&FROM:&&TO

// Set value and range
ENVI @SPIN1.VAL=%&POS%:-20:5               // current, min=-20, max=5
```

---

## 69. WM_DROPFILES 拖放

### 在窗口或控件上启用文件拖放

```wcs
SET &WM_DROPFILES=0x0233

// Register drop handler on EDIT control (type 0x100 = accept dropped file)
EDIT|- EDIT1,L10T10W400H200,,0x100
ENVI @EDIT1.MSG=%&WM_DROPFILES%::&&wp,&&lp, CALL OnDrop %&wp% %&lp%

// Register drop handler on window
ENVI @this.MSG=%&WM_DROPFILES%::&&wp,&&lp, CALL OnDrop %&wp% %&lp%
```

### 提取拖放的文件路径

```wcs
_SUB OnDrop
    ENVI ?&&firstFile,&&allFiles=DROPFILE,%1    // %1 = wParam
    MESS Dropped: %&&allFiles%
    ENVI @EDIT1=%&&allFiles%
_END
```

---

## 70. RICHEDIT 富文本格式化

### 创建富文本编辑控件

```wcs
// -rich flag enables rich text mode on EDIT/MEMO
EDIT|- -rich RichEdit1,L10T10W400H300,Default text,,0x200
MEMO-+ -rich &&RichBox,L10T10W400H300,,0x200
```

### 对特定文本范围着色和格式化

```wcs
// Format: [:fontsize[:fontname:]BITUL;][color[#bgcolor]][;start_pos[;end_pos]]
// B=Bold, I=Italic, U=Underline, T=Strikeout, L=Link
ENVI @RichEdit1.COLOR=:20:Consolas:BI;0xFF;0;3      // Bold+Italic, red, pos 0-3
ENVI @RichEdit1.COLOR=:12;0xFF00;3;6                  // green, pos 3-6
ENVI @RichEdit1.COLOR=:9;0xFF0000;6;9                 // red (BGR), pos 6-9
ENVI @RichEdit1.COLOR=:10;0xFF00FF;2:;4:              // magenta, line 2 to line 4
```

### 编程式文本替换

```wcs
SET &EM_SETSEL=0x00B1
SET &EM_REPLACESEL=0x00C2
ENVI @RichEdit1.SENDMSG=%&EM_SETSEL%,startPos,endPos
ENVI @RichEdit1.SENDMSG=%&EM_REPLACESEL%,0,$newText
```

---

## 71. IMAG 高级（GIF 动画与动态更新）

### GIF 动画显示

```wcs
IMAG IMAG1,L10T10W200H150,animation.gif,EXEC calc.exe     // click runs calc
ENVI @IMAG1.delay=2000                                     // set frame delay to 2s
```

### 运行时动态更新图像

```wcs
// Update: update=w:h[:x:y:border_color:border_width][;filename]
ENVI @IMAG1.update=32:32;shell32.dll#52                    // replace with icon #52
ENVI @IMAG1.update=64:64::;*newimage.png                   // * = new image
ENVI @IMAG1.update=32:32::;?overlay.png                    // ? = overlay on existing

// Source rectangle: <X:Y:W;H>filename
ENVI @IMAG1.update=64:64::<0:0:32;32>source.bmp            // crop region
```

### IMAG 作为交互式图像按钮

```wcs
IMAG ImgBtn,L10T10W64H64,#1000,CALL OnImageClick          // resource icon as button
CHEK -scale:(51*96/12)<123:51>:bg.png ImgChk,L100T100W123H53,,CALL OnCheck
RADI -scale:(51*96/12)<123:51>:bg.png ImgRad,L100T200W123H53,,CALL OnRadio
```

---

### 52. 通过 GetScrollInfo API 控制滚动条

```wcs
SET &SIF_RANGE=0x0001
SET &SIF_PAGE=0x0002
SET &SIF_POS=0x0004
SET &SIF_TRACKPOS=0x0010
SET &SIF_ALL=0x0017

SET &SCROLLINFO.SIZE=28  // cbSize(4)+fMask(4)+nMin(4)+nMax(4)+nPage(4)+nPos(4)+nTrackPos(4)
ENVI$ &&si=*28 0
SET-long &&si=28:0           // cbSize=28
SET-long &&si=%&SIF_ALL%:4    // fMask

CALL $--qd --ret:&bret user32.dll,GetScrollInfo,#%&TBID%,#0,*&si  // SB_HORZ=0
SET?int &&si=&&nPos:20        // current scroll position

// Horizontal column position:
CALC &&col=ceil(%&nPos% / %&ColumnWidth%) + 1

// Scroll to specific position via message:
SET @@sendmsg=%&TBID%;%&lvm_scroll%;%&Pos%;0
// Or scroll to row: SET @@sendmsg=%&TBID%;%&lvm_ensurevisible%;%&index%;0
```

---

## 72. TABS 跨页控件访问

### 从子函数访问同级页面控件

```wcs
_SUB Page1,W289H249,P1,,,#
    LIST L01,L18T20W240H20,,,,0x100
_END
_SUB Page2,W289H249,P2,,,#
    LIST L02,L18T20W240H20,,,,0x100
_END
_SUB WIN3,W350H333,Tab Switch,
    TABS TABS1,L21T4W300H188,Page1:Name1:Title1:tip1;Page2:Name2:Title2:tip2
    ITEM ITEM2,L218T272W96H30,Close,KILL \
_END

// From a child function (e.g., called by Page1's event handler):
_SUB ADD2LIST
    ENVI &&PARENT=-:-:                               // two levels up in execution stack
    ENVI @%&PARENT%Name1:L01.VAL=%&ToList%           // access Page1's LIST
    ENVI @%&PARENT%Name2:L02.VAL=%&ToList%           // access Page2's LIST
_END

// Shortcut from parent window (WIN3 itself):
// ENVI @Name1:L01.VAL=%&ToList%   // no "-" needed when directly in parent
```

关键点：`-` 沿执行栈向上导航（非窗口层级）。每个 `-` = 一级。
名称沿窗口-控件树向下导航。

---

