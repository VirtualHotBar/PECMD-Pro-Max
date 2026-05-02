# GUI 控件/窗口/绘制 — 代码配方

## 7. GUI Patterns

### Complete window template [CHINESE]

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


### Dynamic control creation & deletion

```wcs
// Create controls programmatically from a command string in a variable
ENVI &&cmd=LABE -vcenter -trans Lbl%&i%,L%x%T%y%W%w%H%h%,%&text%,,0x000000,14
%&cmd%                                              // execute the command to create the control

// Delete controls dynamically
ENVI @Lbl%A.*del=                                   // delete label A
ENVI @Edit%B.*del=                                  // delete edit field B
// This is essential for dynamic GUIs that rebuild control sets
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

---

## 32. SWIN Nested Windows (Tab Pages)

Complete property-page pattern: define each sub-window as a `_SUB`, embed them with `SWIN` in the parent, use `TABS.SEL` to switch visible page on tab click.

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

## 33. Dynamic Row Creation & Batch Deletion

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

## 36. TABL Scrollbar Control via LVM Messages

Use `SENDMSG` to send list-view messages for scroll control. Messages apply to the underlying SysListView32 control.

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

Note: `SENDMSG` sends to the window that last received focus / the foreground window. In a GUI context, you may need to focus the table first or use `ENVI @Table1.SENDMSG` (PECMD 2012 extension). For cross-process or precise control, use `CALL $ user32.dll,SendMessageW`.

---

## 37. TABL In-Row Sorting (Bubble Sort)

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

Key points:
- Double-percent `%%&Row[%&i%]%%` dereference: first `%%` evaluates to `%`, then `%&Row[3]%` reads the variable.
- `FIND $str1>str2` does lexicographic "greater" comparison (compare two strings, true if first > second). Use `|` prefix instead for numeric compare.
- Remove non-digits with `SED` for numeric extraction before numeric compare.

---

## 38. Custom Title Bar Window (Frameless + Manual Caption)

Borderless window with fake title bar built from LABE controls. Handles minimize, close, hover color effects, and window dragging via `WM_NCHITTEST`.

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

## 64. TREE Control (Hierarchical Node View)

### Create tree with icons and node hierarchy

```wcs
// Node data format: \parent_index:icon_index:label text
// Child delimiters: 0x0B = start children, 0x0C = end children, 0x09 = separator
TREE Tree1,L10T10W300H300,%&DATA%,0x10000127

SET &MUI_NODE_DATA=\0:0:Root1\x0B\0:0:Child1.1\x09\0:0:Child1.2\x0C\1:1:Root2\x0B\1:1:Child2.1\x0C

// Expand / Collapse nodes
ENVI @Tree1.Expand=1                 // expand node 1
ENVI @Tree1.Expand=2.1;0x0001       // collapse (TVE_COLLAPSE)
ENVI @Tree1.Expand=4;0x4002         // expand partial (TVE_EXPANDPARTIALX)

// Select a node
ENVI @Tree1.Sel=2.2                 // select node 2.2
ENVI @Tree1.Sel=?*2;&&node1         // query selected node path
ENVI @Tree1.Sel=?@&&hnode           // query selected node handle

// Checkbox state
ENVI @Tree1.Check=3.1;2             // set indeterminate (0=unchecked, 1=checked, 2=indeterminate)
ENVI @Tree1.Check=?*;&&state        // query checkbox state
```

### Handle TVN_ITEMCHANGEDW (checkbox change notification)

```wcs
CALC -base=16 #&&TVN_ITEMCHANGEDW=0x100000000-419
ENVI @this.MSG=NOTIFY#%&Tree1_ID%#%&&TVN_ITEMCHANGEDW%::&&wp,&&lp, CALL OnItemChanged %&&wp% %&&lp%

_SUB OnItemChanged
    // parse NMTREEVIEW struct for changed item
_END
```

---

## 67. GDI Painting (WM_PAINT Drawing)

### Register GDI functions as aliases

```wcs
ENVI^ Alias -opt Rectangle=CALL $--qd# --ret:* Gdi32,Rectangle,*dummy,
ENVI^ Alias -opt Ellipse=CALL $--qd# --ret:* Gdi32,Ellipse,*dummy,
ENVI^ Alias -opt Polyline=CALL $--qd# --ret:* Gdi32.dll,Polyline,*dummy,
```

### Window with paint callback and animation

```wcs
_SUB CanvasWin,W260H320,Canvas Demo,
    ENVI @this.Paint=OnPaint               // set WM_PAINT handler
    SET &aw=2
    SET &w=10
    TIME &Timer1,50, ENVI @this.InvalidateRect=;;;230;
_END

_SUB OnPaint                               // %1 = HDC handle
    CALC #L=%x0% - %w% - %L0%
    CALC #T=%y0% - %h%
    CALC #R=%x0% + %w% - %L0%
    CALC #B=%y0% + %h%
    Rectangle %1,%T%,%L%,%B%,%R%
    Ellipse %1,%L%,%T%,%R%,%B%
    CALC #w=%&w% + %&aw%
    IFEX $%&w%>100, TEAM SET aw=-2|CALC #w=%&w% + %&aw%!
    IFEX $%&w%<0, TEAM SET aw=2|CALC #w=%&w% + %&aw%
_END
```

### Polyline with POINT array

```wcs
SET$ &Pt= 0x0064 0x0000  0x0000 0x0000  0x00C8 0x0000  0x0064 0x00C8  *200 0
ENVI-addr &&PtAddr=&Pt

_SUB OnPaint
    Polyline %1,%&PtAddr%,4
_END
```

---

## 68. PBAR / SPIN Controls (Progress Bar & Up-Down)

### Progress bar with color and text

```wcs
PBAR PBAR1,L22T13W200H16,20               // initial value = 20%
ENVI @PBAR1.color=0xFF                     // foreground (red)
ENVI @PBAR1.bkcolor=0xFF00                 // background (green)

// Update progress with text overlay
ENVI @PBAR1=%&p%;%&K%s  %&p%%%            // value;text

// Advanced: percent display with colored text
ENVI @PBAR1.percent=%&p%C:0xFF00:0xCFFF:0xFF:%&K%s  %&p%%%
```

### SPIN control bound to EDIT

```wcs
EDIT EDIT1,L28T14W158H29,0,,
SPIN SPIN1,L192T13W22H30,EDIT1,&&npos:&&button:&&old,
    ENVI @LABE3= SPIN1 [%&&npos%] [%&&button%] [%&&old%], 0xA0

// Query value and range
ENVI @SPIN1.VAL=?&&POS:&&FROM:&&TO

// Set value and range
ENVI @SPIN1.VAL=%&POS%:-20:5               // current, min=-20, max=5
```

---

## 69. WM_DROPFILES Drag-and-Drop

### Enable file drop on window or control

```wcs
SET &WM_DROPFILES=0x0233

// Register drop handler on EDIT control (style 0x4 = accept files)
EDIT|- EDIT1,L10T10W400H200,,0x004
ENVI @EDIT1.MSG=%&WM_DROPFILES%::&&wp,&&lp, CALL OnDrop %&wp% %&lp%

// Register drop handler on window
ENVI @this.MSG=%&WM_DROPFILES%::&&wp,&&lp, CALL OnDrop %&wp% %&lp%
```

### Extract dropped file paths

```wcs
_SUB OnDrop
    ENVI ?&&firstFile,&&allFiles=DROPFILE,%1    // %1 = wParam
    MESS Dropped: %&&allFiles%
    ENVI @EDIT1=%&&allFiles%
_END
```

---

## 70. RICHEDIT Rich Text Formatting

### Create rich text edit control

```wcs
// -rich flag enables rich text mode on EDIT/MEMO
EDIT|- -rich RichEdit1,L10T10W400H300,Default text,,0x200
MEMO-+ -rich &&RichBox,L10T10W400H300,,0x200
```

### Color and format specific text ranges

```wcs
// Format: [:fontsize[:fontname:]BITUL;][color[#bgcolor]][;start_pos[;end_pos]]
// B=Bold, I=Italic, U=Underline, T=Strikeout, L=Link
ENVI @RichEdit1.COLOR=:20:Consolas:BI;0xFF;0;3      // Bold+Italic, red, pos 0-3
ENVI @RichEdit1.COLOR=:12;0xFF00;3;6                  // green, pos 3-6
ENVI @RichEdit1.COLOR=:9;0xFF0000;6;9                 // blue, pos 6-9
ENVI @RichEdit1.COLOR=:10;0xFF00FF;2:;4:              // magenta, line 2 to line 4
```

### Programmatic text replacement

```wcs
SET &EM_SETSEL=0x00B1
SET &EM_REPLACESEL=0x00C2
ENVI @RichEdit1.SENDMSG=%&EM_SETSEL%,startPos,endPos
ENVI @RichEdit1.SENDMSG=%&EM_REPLACESEL%,0,$newText
```

---

## 71. IMAG Advanced (GIF Animation & Dynamic Update)

### Animated GIF display

```wcs
IMAG IMAG1,L10T10W200H150,animation.gif,EXEC calc.exe     // click runs calc
ENVI @IMAG1.delay=2000                                     // set frame delay to 2s
```

### Dynamic image update at runtime

```wcs
// Update: update=w:h[:x:y:border_color:border_width][;filename]
ENVI @IMAG1.update=32:32;shell32.dll#52                    // replace with icon #52
ENVI @IMAG1.update=64:64::;*newimage.png                   // * = new image
ENVI @IMAG1.update=32:32::;?overlay.png                    // ? = overlay on existing

// Source rectangle: <X:Y:W;H>filename
ENVI @IMAG1.update=64:64::<0:0:32;32>source.bmp            // crop region
```

### IMAG as interactive image button

```wcs
IMAG ImgBtn,L10T10W64H64,#1000,CALL OnImageClick          // resource icon as button
CHEK -scale:(51*96/12)<123:51>:bg.png ImgChk,L100T100W123H53,,CALL OnCheck
RADI -scale:(51*96/12)<123:51>:bg.png ImgRad,L100T200W123H53,,CALL OnRadio
```

---

### 52. ScrollBar via GetScrollInfo API

```wcs
SET &SIF_RANGE=0x0001
SET &SIF_PAGE=0x0002
SET &SIF_POS=0x0004
SET &SIF_TRACKPOS=0x0010
SET &SIF_ALL=0x0017

SET &SCROLLINFO.SIZE=28  // cbSize(4)+fMask(4)+nMin(4)+nMax(4)+nPage(4)+nPos(4)+nTrackPos(4)
ENVI$ &&si=*28 0
SET-long &&si=28:0           // cbSize=28
SET-long &&si=%SIF_ALL%:4    // fMask

CALL $--qd --ret:&bret user32.dll,GetScrollInfo,#%&TBID%,#0,*&&si  // SB_HORZ=0
SET?int &&si=&&nPos:20        // current scroll position

// Horizontal column position:
CALC &&col=ceil(%&nPos% / %&ColumnWidth%) + 1

// Scroll to specific position via message:
SET @@sendmsg=%&TBID%;%&lvm_scroll%;%&Pos%;0
// Or scroll to row: SET @@sendmsg=%&TBID%;%&lvm_ensurevisible%;%&index%;0
```

---

## 72. TABS Cross-Page Control Access

### Access sibling page controls from a child function

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

Key: `-` navigates UP the execution stack (not window hierarchy). Each `-` = one level.
Names navigate DOWN through the window-control tree.

---

