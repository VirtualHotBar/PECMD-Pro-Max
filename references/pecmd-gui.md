# PECMD2012 GUI Window System Reference

Complete reference for PECMD's window/control system. Covers window definition, all 22 control types, message mapping, lifecycle, and advanced GUI patterns.

---

## 1. Window Definition

### `_SUB` Structure for Windows

```wcs
_SUB WinName,<shape>,<title>,[closeCmd],[icon],[style],[mask],[flags]
    // control definitions
_END
```

| Field | Syntax | Description |
|-------|--------|-------------|
| **shape** | `L<left>T<top>W<width>H<height>` | Window position and size. Omit `L`/`T` to auto-center. `W`/`H` required. |
| **title** | `"Window Title"` | Title bar text. Empty string `""` for no title. |
| **closeCmd** | `KILL \` or any command | Executed when window closes (X button or `KILL`). Omit for default close behavior. |
| **icon** | `path.ico` or `exe#index` | Window icon. File path or EXE/DLL resource reference. |
| **style** | `[#][$]number` or `,#` | Window style number. `#` = hex, `$` = decimal. `,#` = hidden window. |
| **mask** | `[color][*][w:h]bitmap` | Shaped/irregular window region from bitmap. `color` = transparency key. |
| **flags** | `-flag1 -flag2 ...` | Window behavior flags (see table below). |

### All Window Flags

| Flag | Effect |
|------|--------|
| `-top` | Always on top (TOPMOST) |
| `-nocap` | No title bar / caption |
| `-nosysmenu` | No system menu (no icon in title bar) |
| `-nofix` | Window position not fixed; allows default OS placement |
| `-trap` | Close button does NOT exit; fires closeCmd instead |
| `-size` | Resizable window (sizing border) |
| `-maxb` | Enable Maximize button |
| `-minb` | Enable Minimize button |
| `-disminb` | Disable (grey out) Minimize button |
| `-discloseb` | Disable (grey out) Close button |
| `-nfocus` | Window cannot receive keyboard focus |
| `-ntab` | Window excluded from Alt+Tab list (tool window) |
| `-disaltmv` | Disable Alt+arrow-key window move |
| `-forcenomin` | Prevent window from being minimized |
| `-scalef` | XP-style DPI font scaling |
| `-scale[:DPI]` | Win8+ Per-Monitor DPI scaling. `-scale:144` for explicit DPI. |
| `-nxp` | Disable XP visual styles (flat classic look) |
| `-csize` | Shape dimensions specify **client area** (excluding title bar/borders) |
| `-na` | Do not activate (no focus steal on creation) |
| `-nb` | No border |

### Style: Transparency and Hidden Windows

```wcs
_SUB Win,L20T20W300H200,Title,,,#0x80000000       // transparent window (alpha 0x80)
_SUB Win,L20T20W300H200,Title,,,$0x80C80000       // transparent window (decimal style)
_SUB Win,L20T20W300H200,Title,,,#                  // hidden window (,# = create hidden)
```

### Mask: Shaped / Irregular Windows

```wcs
_SUB Win,L20T20W300H200,Title,,,0xFFFFFF*mask.bmp // color-keyed shaped window
_SUB Win,L20T20W300H200,Title,,,*300:200:mask.bmp  // sized bitmap mask
_SUB Win,L20T20W300H200,Title,,,0x00FF00**mask.bmp // colored mask (green=transparent)
```

- `color`: Transparency color for bitmap-sourced windows (RGB hex: `0xFF0000` = red, `0x000000` = black)
- `*`: Separator between color and size/source
- `w:h:bitmap`: Explicit dimensions with bitmap file
- `*bitmap`: Auto-size bitmap; double-`*` = colored mask mode

---

## 2. Control Types Overview

PECMD exposes 22 control types via dedicated commands. Each creates a specific Windows common control.

| # | Command | Windows Control | Purpose |
|---|---------|-----------------|---------|
| 1 | `ITEM` | Button | Clickable button with text/image, check states, default action |
| 2 | `EDIT` | Edit Box | Single/multi-line text input, password mode, rich text |
| 3 | `MEMO` | Multi-line Edit | Large text area with built-in scrollbars |
| 4 | `CHEK` | Check Box | Binary/tristate on-off toggle |
| 5 | `RADI` | Radio Button | Mutually exclusive group selection |
| 6 | `LIST` | Combo Box | Dropdown list or editable combo, text+image items |
| 7 | `LABE` | Static Text | Display text, multi-color, images, clickable links |
| 8 | `IMAG` | Picture Control | BMP/JPG/GIF/AVI/ICO display, icon extraction from EXE/DLL |
| 9 | `PBAR` | Progress Bar | Visual progress indicator with text overlay |
| 10 | `SLID` | Slider (Trackbar) | Range slider with configurable min/max |
| 11 | `SPIN` | Spinner (Up-Down) | Numeric spinner with buddy edit control |
| 12 | `GROU` | Group Box | Visual grouping frame |
| 13 | `TABL` | ListView (Report) | Multi-column data grid with checkboxes, icons, sorting |
| 14 | `TABS` | Tab Control | Property page tabs for sub-window embedding |
| 15 | `SWIN` | (Custom) | Sub-window/pane container for embedding other windows |
| 16 | `DTIM` | Date-Time Picker | Date/time selection with configurable display format |
| 17 | `IPAD` | IP Address Control | IP address entry with dot-separated octets |
| 18 | `TIME` | Timer | Periodic callback timer (not a visual control) |
| 19 | `HKEY` | Global Hotkey | System-wide keyboard shortcut registration |
| 20 | `MENU` | Menu | Popup context menu or window menu bar |
| 21 | `TIPS` | Tray Icon | System tray notification icon with tooltip and callbacks |
| 22 | `SCRN` | Screen Capture | Screenshot / region capture (non-visual) |

---

## 3. Common Control Operations

### `ENVI @` Control Manipulation — All Controls

All controls support these operations via `ENVI @<Name>.<Operation>`:

#### Text and Enable/Visible

```wcs
ENVI @CtrlName=New Text                                // set control text
ENVI @CtrlName=%&newText%                              // set text from variable
ENVI @CtrlName.Enable=0                                // disable (grey out)
ENVI @CtrlName.Enable=1                                // enable
ENVI @CtrlName.Visible=0                               // hide
ENVI @CtrlName.Visible=1                               // show
ENVI @CtrlName.Visible=*4                              // minimize window (window only)
```

#### Position and Size

```wcs
ENVI @CtrlName.POS=left:top:width:height               // move and resize
ENVI @CtrlName.POS=?;&L:&T:&W:&H                       // query position into variables
ENVI @CtrlName.POS=?;&L;1;&T;1;&W;1;&H;1               // query, semicolon-delimited output
ENVI @CtrlName.POS=?%&WinName%;&L:&T:&W:&H             // query window position
```

#### Appearance

```wcs
ENVI @CtrlName.Font=12:Tahoma                           // set font size and face
ENVI @CtrlName.Font=12;Tahoma                          // semicolon separator also works
ENVI @CtrlName.Font=12:Microsoft YaHei:Bold             // with bold style
ENVI @CtrlName.Font=:                                    // reset to default font
ENVI @CtrlName.bkcolor=0xFF0000                         // set background color (BGR hex)
ENVI @CtrlName.bkcolor=-2                               // transparent background
ENVI @CtrlName.trans=1                                   // translucent (layered window only)
ENVI @CtrlName.Cursor=32649                             // hand pointer cursor (IDC_HAND)
ENVI @CtrlName.Cursor=32514                             // normal arrow (IDC_ARROW)
ENVI @CtrlName.Cursor=32515                             // I-beam (IDC_IBEAM)
```

#### Style Modification

```wcs
ENVI @CtrlName.Style=+0x1000                            // add window style bit
ENVI @CtrlName.Style=-0x1000                            // remove window style bit
ENVI @CtrlName.Style=?;&styleVar                         // query current style
ENVI @CtrlName.ExStyle=+0x80                            // add extended style
ENVI @CtrlName.ExStyle=-0x80                            // remove extended style
```

#### Destruction and Invalidation

```wcs
ENVI @CtrlName.*del=                                     // destroy control (remove from window)
ENVI @CtrlName.InvalidateRect=                          // force full redraw
ENVI @CtrlName.InvalidateRect=10:20:100:50               // redraw region (l:t:w:h)
```

#### Multiple Controls Simultaneously

```wcs
ENVI @MultipleCtrl.VAL=%&data%                          // set VAL on controls matching pattern
ENVI @MultipleCtrl.Enable=0                             // disable all matching controls
ENVI @MultipleCtrl.Visible=0                            // hide all matching controls
// MultipleCtrl is a wildcard: @Btn* matches Btn1, Btn2, BtnSave, etc.
```

#### Disable vs Hide vs Grey Out

```wcs
ENVI @Ctrl.Enable=0       // greyed out, visible but non-interactive
ENVI @Ctrl.Visible=0      // hidden, takes no layout space
ENVI @Ctrl.Enable=1       // fully interactive
```

---

## 4. Detailed Per-Control Reference

### 4.1 ITEM — Button

```wcs
ITEM [-font:N] [-def] [-right] [-round] [-na] Name,LxTyWwHh,Text,[Command],[State],[Style]
```

| Flag | Purpose |
|------|---------|
| `-def` | Default button (Enter triggers it), black border |
| `-right` | Right-aligned button text |
| `-round` | Rounded corners (only on themed XP+) |
| `-na` | Do not activate / no focus |
| `-font:N` | Font size N |

**State values:**
| Value | Meaning |
|-------|---------|
| `0` | Normal push button |
| `1` | Checked (toggle button, stays down) |
| `2` | 3-state semi-checked (grey check) |

**Image button (embedded IMAG):**
```wcs
IMAG Img1,L10T10W32H32,myicon.ico
ITEM Btn1,L20T10W100H32,Imag1Click me!,CALL OnClick
// Embedded image: control name of an IMAG used as button face
```

**Command:** Executed on click. Use `CALL FuncName [args]` for function calls.

```wcs
ENVI @Btn1.Check=1          // set toggle button to checked state
ENVI @Btn1.Check=?;&state   // query state
```

---

### 4.2 EDIT — Edit Box

```wcs
EDIT [-vcenter] [-rich] [-wantTAB] [-pwd|-ipwd] Name,LxTyWwHh,[InitText],[Cmd],[Style],[FontSize]
```

| Flag | Purpose |
|------|---------|
| `-vcenter` | Vertically center text (single-line only) |
| `-rich` | Rich Edit control (supports formatting) |
| `-wantTAB` | Tab key stays in edit (doesn't navigate to next control) |
| `-pwd` | Password mode: displays `*` for each character |
| `-ipwd` | Invisible password: no visual feedback at all |
| (none) | Standard single-line edit |

**Scroll bar flags in Style:**
```wcs
EDIT Ed,L10T30W200H200,,,0x0020          // VSCROLL
EDIT Ed,L10T30W200H200,,,0x0010          // HSCROLL (horizontal scroll)
EDIT Ed,L10T30W200H200,,,0x0030          // both VSCROLL + HSCROLL
```

**Read-only:** Add `0x0800` (ES_READONLY) to Style.

**Auto line-wrap:** Add `0x0004` (ES_AUTOHSCROLL off) for multi-line wrapping without horizontal scroll.

**Text operations:**
```wcs
ENVI @Ed.QUERY=;&textVar                 // get all text
ENVI @Ed=New Text                         // set all text (replaces)
ENVI @Ed.SET=Text to set                 // set text (equivalent to =)
ENVI @Ed.SEL=start:end                    // select text range (0-based)
ENVI @Ed.SEL=?;&start;&end               // query selection range
```

---

### 4.3 LABE — Label (Static Text)

```wcs
LABE [-vcenter] [-left|-center|-right] [-trans] [-3D] [-ncmd] Name,Shape,Text,[Command],[Color],[FontSize]
```

| Flag | Purpose |
|------|---------|
| `-vcenter` | Vertically center text |
| `-left` | Left-aligned (default) |
| `-center` | Center-aligned |
| `-right` | Right-aligned |
| `-trans` | Transparent background (parent shows through) |
| `-3D` | Sunken 3D border look |
| `-ncmd` | Label receives click command (makes it clickable) |

**Multi-color text support:**
```wcs
ENVI @MyLabel.Color=1:0xFF0000;3:0x00FF00   // word 1 = red, word 3 = green
ENVI @MyLabel.Color=0x0000FF                 // all text blue
// Color format: word_position:0xBBGGRR;word_position:0xBBGGRR
// word_position is 1-based by space-delimited tokens
```

**Image-embedded labels:**
```wcs
LABE ImgLbl,L10T10W200H32,#0x0100|image.bmp,Text here,,0x0000FF
// Prefix text with [image source] to embed image alongside text
```

**Clickable labels:**
```wcs
LABE ClickLbl,L10T50W100H20,Click Me,CALL OnLabelClick   // command makes it clickable
```

**Hyperlink/web link labels:**
```wcs
LABE LinkLbl,L10T80W200H20,Visit Site,-href:https://example.com
LABE LinkLbl,L10T110W200H20,Visit Site,-href:https://example.com|-hovertip:Open in browser
```

---

### 4.4 CHEK — Checkbox

```wcs
CHEK [-right] [-scale] Name,LxTyWwHh,Text,[EventCmd],[State],[Style],[FontSize]
```

| Flag | Purpose |
|------|---------|
| `-right` | Check mark on right side of text |
| `-scale` | Scale with DPI |

**State values:**
| Value | Meaning |
|-------|---------|
| `0` | Unchecked |
| `1` | Checked |
| `2` | Indeterminate / "don't care" (requires BS_3STATE style: `0x0005`) |

**Operations:**
```wcs
ENVI @Chk.Check=1                  // check
ENVI @Chk.Check=0                  // uncheck
ENVI @Chk.Check=?;&var             // query state (semicolon before var name)
ENVI @Chk.Check=?;0;&var           // query, 0=use semicolon mode
```

---

### 4.5 RADI — Radio Button

```wcs
RADI [-right] [-scale] [-center] Name,LxTyWwHh,Text,[EventCmd],[State],[GroupID],[FontSize]
```

Radio buttons are mutually exclusive within the same **group**. The group is determined by the **integer group ID** embedded in the shape:
```wcs
RADI R1,L10T10W100H20:1,Option A,CALL OnSel,1           // Group 1
RADI R2,L10T30W100H20:1,Option B,,0                     // Group 1 (same group ID)
RADI R3,L10T60W100H20:1,Option C,,0                     // Group 1

RADI R4,L10T90W100H20:2,Option X,CALL OnSel2,1           // Group 2 (different group)
RADI R5,L10T110W100H20:2,Option Y,,0                     // Group 2
```

The group ID is appended after Hheight: `H20:1` = group 1. The first radio in a group typically has `State=1` (selected by default).

**Query:**
```wcs
ENVI @R1.Check=?;&selectedState        // 1 if this radio is selected, 0 otherwise
// To find WHICH radio is selected in a group, query each radio's .Check
```

---

### 4.6 LIST — Dropdown / Combo Box

```wcs
LIST [-h] [-edt] Name,LxTyWwHh,item1|item2|item3,[EventCmd],[Style],[FontSize]
```

| Flag | Purpose |
|------|---------|
| `-h` | Expand/drop-down height in pixels |
| `-edt` | Editable combo box (user can type custom text) |

**Initial items:** Pipe-delimited list: `"Apple|Banana|Cherry|Date"`

**Operations:**
```wcs
ENVI @List.VAL=                                    // clear all items
ENVI @List.ADD=New Item                            // add one item to end
ENVI @List.ADDSEL=New Item                         // add item and select it
ENVI @List.DEL=Item Text                           // delete item by text
ENVI @List.DEL=:3                                   // delete item by 1-based index
ENVI @List.isel=3                                   // select 1-based index
ENVI @List.Sel=3                                    // select 1-based index (same)
ENVI @List.Sel=3;0                                  // deselect
ENVI @List.Sel=?;&selectedIndex                     // get 1-based selected index
ENVI @List.QUERY=;&allItems                         // get all items (newline-delimited)
ENVI @List.Val=?*;&itemCount                        // get item count
ENVI @List.Val=?;itemText                           // get selected item text

// Access via % variable:
%List.isel%                                         // selected index (1-based)
%List.Sel%                                          // same as above
```

**List with images:**
```wcs
LIST ImgLst,L10T10W150H200,
ENVI @ImgLst.ADD=.ico#5|Item with icon             // icon index from resource
```

**Editable combo:**
```wcs
LIST -edt EdtList,L10T10W150H20,Item1|Item2,,,12   // user can type custom text
ENVI @EdtList=User typed text                       // get/set the current text
```

---

### 4.7 IMAG — Image / Picture Display

```wcs
IMAG [-scale] Name,LxTyWwHh,ImageSource,[EventCmd],[Style],[Alpha]
```

**Supported formats:** `.bmp`, `.jpg`, `.jpeg`, `.gif` (animated), `.avi` (animated), `.ico`

**Icon extraction from EXE/DLL:**
```wcs
IMAG Ico,L10T10W32H32,C:\Windows\System32\shell32.dll#23    // icon index 23
IMAG Ico,L10T10W32H32,%SystemRoot%\explorer.exe#0           // first icon
```

**Dynamic image update:**
```wcs
ENVI @Img=NewImage.jpg                              // change image at runtime
ENVI @Img=shell32.dll#42                            // change to different icon
ENVI @Img=                                          // clear image
```

**Background modes (via Style):**
```wcs
IMAG Img,L10T10W200H200,bg.jpg,,0x0000             // stretch to fit
IMAG Img,L10T10W200H200,bg.jpg,,0x0001             // center, no resize
IMAG Img,L10T10W200H200,bg.jpg,,0x0002             // tile repeated
IMAG Img,L10T10W200H200,bg.jpg,,0x0004             // proportional stretch
IMAG Img,L10T10W200H200,bg.jpg,,0x0008             // clip to fit
```

**IMAGE with EMBED syntax:**
```wcs
IMAG Img,L10T10W32H32,#0                             // placeholder, embed at runtime
ENVI @Img=*newimage.jpg                              // replace with new image
```

---

### 4.8 PBAR — Progress Bar

```wcs
PBAR Name,LxTyWwHh,[InitValue],[Style],[Color]
```

**Operations:**
```wcs
ENVI @PBar.Value=50                                 // set to 50%
ENVI @PBar.Value=?;&percent                          // query current value
ENVI @PBar.text=Processing...                        // text on progress bar
ENVI @PBar.bkcolor=0x00FF00                          // bar color (green)
ENVI @PBar.bkcolor=0xFFFFFF                          // background color (white)
```

**Range:** Default 0–100. Can be changed via Windows message.

---

### 4.9 TABL — Table / Data Grid (MOST DETAILED)

```wcs
TABL [-font:N] Name,LxTyWwHh,[HeaderString],[Flags],[Style],[FontSize]
```

**Header format:**
```
HeaderString = "col1_text:col1_width col2_text:col2_width ..."
```
Width prefixes:
| Prefix | Meaning |
|--------|---------|
| `=` | Center-aligned column |
| `+` | Right-aligned column |
| `*` | Checkbox column header |
| (none) | Left-aligned (default) |
| (blank name) | No header text for this column |

```wcs
TABL Tbl,L10T10W400H200,=100:Name +80:Size =120:Date *30,0x40
// Column 1: "Name", width 100, centered
// Column 2: "Size", width 80, right-aligned
// Column 3: "Date", width 120, centered
// Column 4: (empty), width 30, checkbox
```

**Complete Status Flag Table:**

| Flag (Hex) | Flag (Dec) | Name | Description |
|------------|------------|------|-------------|
| `0x10` | 16 | Checkboxes | Each row has a checkbox |
| `0x20` | 32 | Per-row color | Custom row background colors supported |
| `0x40` | 64 | Icons | Icon per row via `.ico#id` |
| `0x80` | 128 | Full row select | Entire row highlights on selection |
| `0x100` | 256 | No column resize | Fixed column widths |
| `0x200` | 512 | Single selection | Only one row selectable at a time (default) |
| `0x400` | 1024 | Multi-selection | Multiple rows selectable (Ctrl+Click) |
| `0x800` | 2048 | Grid lines | Visible grid lines between cells |
| `0x1000` | 4096 | No header | Hide column headers |
| `0x2000` | 8192 | Editable | Cells can be edited in-place |
| `0x4000` | 16384 | Owner draw | Custom drawing via callback |
| `0x8000` | 32768 | Drag-drop | Rows can be reordered by dragging |
| `0x10000` | 65536 | Sort headers | Click header to sort column |
| `0x40000` | 262144 | No mouse select | Disable mouse-based row selection |

```wcs
// Common flag combinations:
TABL Tbl,L10T10W400H200,Col1:100 Col2:80,0x40    // icons only
TABL Tbl,L10T10W400H200,Col1:100 Col2:80,0x50    // checkboxes + icons
TABL Tbl,L10T10W400H200,Col1:100 Col2:80,0x2C0   // single-select + full-row-select + icons
TABL Tbl,L10T10W400H200,Col1:100 Col2:80,0x10C30 // sort-headers + grid + multi-select + color + checkboxes
```

**Data Operations:**

```wcs
// --- Set data ---
ENVI @Tbl.Val=1*;%&allData%                         // bulk-set ALL rows from variable
// allData format: row1col1\trow1col2\nrow2col1\trow2col2\n...
// \t = TAB between columns, \n = newline between rows

ENVI @Tbl.Val=%row%;col1%&TAB%col2%&TAB%col3       // set single row (1-based)
ENVI @Tbl.Val=%row%;*                                // delete single row

// --- Get data ---
ENVI @Tbl.Val=?%row%.%col%;&cellValue               // get cell (semicolon before var)
ENVI @Tbl.Val=?*;&rowCount                           // get total row count
ENVI @Tbl.Val=?%row%;&fullRowData                   // get entire row (TAB-delimited)

// --- Clear ---
ENVI @Tbl.Val=-*                                     // clear ALL rows

// --- Selection ---
ENVI @Tbl.Sel=%row%                                  // select row (highlight)
ENVI @Tbl.Sel=%row%;0                                 // deselect row
ENVI @Tbl.Sel=?;&selectedRow                          // get selected row number
ENVI @Tbl.Sel=?*;&allSelected                         // get all selected rows
ENVI @Tbl.Sel=?*;&count                               // get selected row count

// --- Check-state ---
ENVI @Tbl.Check=%row%;1                               // check row checkbox
ENVI @Tbl.Check=%row%;0                               // uncheck row checkbox
ENVI @Tbl.Check=?%row%;&checkState                    // query row checkbox (1/0)

// --- Row color ---
ENVI @Tbl.Color=%row%;0xFF0000                        // set row text color (BGR)
ENVI @Tbl.Color=%row%;0xFF0000;0xFFFFFF              // text color;background color

// --- Row icon ---
ENVI @Tbl.Val=%row%;.ico#5;col1%&TAB%col2           // set row with icon #5

// --- Position / Scroll ---
ENVI @Tbl.UPOS=?;&rowIndex                           // get top visible row index

// --- Percent operations (for PBAR-style columns) ---
ENVI @Tbl.Percent=?%row%;&percentVal                  // query percent column
```

---

### 4.10 TABS — Tab Control / Property Pages

```wcs
TABS Name,LxTyWwHh,Page1|Page2|Page3,[EventCmd]
```

TABS embeds **sub-windows** (SWIN) for each page:

```wcs
TABS Tabs1,L10T10W400H300,General|Advanced|About,CALL OnTabChange

SWIN Swin1,L20T40W380H260,Page1Win         // positioned inside the tab area
SWIN Swin2,L20T40W380H260,Page2Win         // occupies same space
SWIN Swin3,L20T40W380H260,Page3Win

// Page switching:
ENVI @Tabs1.SEL=2                            // switch to page 2 (1-based)
ENVI @Tabs1.SEL=?;&currentPage               // query current page

// In the tab change handler:
_SUB OnTabChange
    ENVI @Tabs1.SEL=?;&sel
    ENVI @Swin1.Visible=%&sel%=1?1:0         // show/hide corresponding SWIN
    ENVI @Swin2.Visible=%&sel%=2?1:0
_END
```

---

### 4.11 SWIN — Sub-Window / Pane Embedding

```wcs
SWIN Name,LxTyWwHh,SubWinDef,[flags]
```

`SubWinDef` is the name of a `_SUB` defining the embedded window. SWIN creates a child window and embeds the `_SUB`'s controls inside it.

```wcs
_SUB Page1
    LABE Lbl,L10T10W200H20,This is Page 1
    ITEM Btn,L10T40W80H28,Click,CALL OnClick1
_END

SWIN SwinPg1,L20T40W380H260,Page1     // embed Page1 as child window
```

**Multi-panel management:**
```wcs
// Create multiple SWINs, show only the active one
SWIN Panel1,L20T40W380H260,Page1Win
SWIN Panel2,L20T40W380H260,Page2Win
SWIN Panel3,L20T40W380H260,Page3Win

ENVI @Panel2.Visible=0                 // hide unused panels
ENVI @Panel3.Visible=0

// Switch panel:
_SUB SwitchToPanel2
    ENVI @Panel1.Visible=0
    ENVI @Panel2.Visible=1
_END
```

---

### 4.12 SLID — Slider / Trackbar

```wcs
SLID [-range:min:max] Name,LxTyWwHh,[InitVal],[EventCmd],[Style]
```

```wcs
SLID -range:0:100 Sld,L10T10W200H30,50,CALL OnSlide
SLID -range:1:10 Sld,L10T10W200H30,5                // init at 5, range 1-10
SLID -range:-20:80 Sld,L10T50W200H30,0               // negative min supported
```

**Operations:**
```wcs
ENVI @Sld.VAL=75                                     // set slider position
ENVI @Sld.VAL=?;&value                                // query slider value
```

---

### 4.13 SPIN — Spinner / Up-Down Control

```wcs
SPIN [-range:min:max] [buddyEditName] Name,LxTyWwHh,[InitVal],[EventCmd],[Style]
```

SPIN automatically pairs with a buddy EDIT control for numeric input:

```wcs
EDIT Edit1,L10T10W60H20,0
SPIN -range:0:100 Spin1,L70T10W16H20,0,CALL OnSpin   // buddy = Edit1 (auto-detect)

// Explicit buddy:
SPIN -range:0:255 Spin2,L10T40W16H20,0,,,Edit2
```

**Operations:**
```wcs
ENVI @Spin1.VAL=50                                    // set spinner value
ENVI @Spin1.VAL=?;&value                              // query value
```

---

### 4.14 DTIM — Date-Time Picker

```wcs
DTIM [-type:N] Name,LxTyWwHh,[InitDateTime],[EventCmd],[Style]
```

| Type | Display |
|------|---------|
| `-type:0` | Date and Time (default) |
| `-type:1` | Date only (short format) |
| `-type:2` | Time only |
| `-type:3` | Date + Time + Checkbox |

```wcs
DTIM -type:1 Dt1,L10T10W120H22,,CALL OnDateChange     // date only
DTIM -type:2 Dt2,L10T40W80H22                          // time only
DTIM -type:0 Dt3,L10T70W180H22                          // date + time
```

**Operations:**
```wcs
ENVI @Dt1.VAL=2025-01-15                                // set date (YYYY-MM-DD)
ENVI @Dt1.VAL=?;&dateValue                              // query date string
ENVI @Dt2.VAL=14:30:00                                   // set time
```

---

### 4.15 IPAD — IP Address Control

```wcs
IPAD Name,LxTyWwHh,[InitIP],[EventCmd],[Style]
```

```wcs
IPAD Ip1,L10T10W140H22,192.168.1.1,CALL OnIPChange
IPAD Ip2,L10T40W140H22,,CALL OnIPChange                 // defaults to 0.0.0.0
```

**Operations:**
```wcs
ENVI @Ip1.VAL=10.0.0.1                                  // set IP
ENVI @Ip1.VAL=?;&ipString                               // query: "192.168.1.100"
```

---

### 4.16 TIME — Timer

```wcs
TIME TimerName,interval,[EventCmd]
```

`TimerName` must be **prefixed with `Timer`** (e.g., `Timer1`, `TimerMain`).

```wcs
TIME Timer1,1000,CALL OnTick                            // fire every 1000ms
TIME Timer1,500,                                        // every 500ms, no command (must query)
TIME Timer2,*500,CALL OnTick                            // * = auto-recycle (restart after fire)
TIME Timer3,0,CALL OnFire                               // 0 = fire immediately once
```

**Timer flags via `-t:N`:**
| Flag | Purpose |
|------|---------|
| `-t:1` | One-shot timer (fires once, then stops) |

**Operations:**
```wcs
ENVI @Timer1=500                                        // change interval
ENVI @Timer1=0                                           // pause / stop
ENVI @Timer1=-del                                        // destroy timer
ENVI @Timer1=-1                                          // restart with current interval

// Multi-timer:
TIME TimerPulse,100,CALL Heartbeat
TIME TimerLong,5000,CALL PeriodicCheck
TIME TimerOnce,0,CALL DelayedInit
```

---

### 4.17 TIPS — System Tray Icon

```wcs
TIPS* IconName,tooltipText,[iconFile],[LeftClickCmd],[RightClickCmd],[Flags]
```

The `*` suffix creates an entry in the tray notification area.

```wcs
TIPS* MyApp,My Application,shell32.dll#13,CALL OnTrayLClick,CALL OnTrayRClick
TIPS* MyApp,My Application,%CurDir%\myicon.ico,CALL OnLClick,CALL OnRClick
```

**Left-click handler:**
```wcs
_SUB OnTrayLClick
    ENVI @MyWin.Visible=1                                // show/restore window
_END
```

**Right-click handler — typically a popup menu:**
```wcs
_SUB OnTrayRClick
    CALL @--popmenu TrayMenu
_END

_SUB TrayMenu
    MENU Show,Show Window,ENVI @MyWin.Visible=1
    MENU Hide,Hide Window,ENVI @MyWin.Visible=0
    MENU -
    MENU Exit,Exit,KILL \
_END
```

**Bubble / balloon notification:**
```wcs
ENVI @MyApp.MSG=+0x0400:CALL OnTrayNotify              // on tray notification
```

// Use TrayNotify through Shell_NotifyIcon

**Tooltip update:**
```wcs
ENVI @MyApp.tip=New Status: Processing...               // update tooltip text
```

**Multi-icon support:**
```wcs
TIPS* App1,App 1 Title,icon1.ico
TIPS* App2,App 2 Title,icon2.ico                         // multiple tray icons
```

**Removal:**
```wcs
ENVI @MyApp.DEL=                                         // remove tray icon
TIPS.DEL=MyApp                                            // alternative syntax
```

---

### 4.18 HKEY — Hotkey Registration

```wcs
HKEY [modifiers+][#]keycode,[eventCmd],[args]
```

| Prefix | Scope |
|--------|-------|
| `$` | Program-level hotkey (system-wide, registered via RegisterHotKey) |
| `*` | Window-level hotkey (only active when window has focus) |
| (none) | Default = window-level |

**Modifier keys:**
```wcs
HKEY Ctrl+#0x41,CALL OnHotKeyA                           // Ctrl+A
HKEY Ctrl+Shift+#0x42,CALL OnHotKeyB                     // Ctrl+Shift+B
HKEY Ctrl+Alt+#0x43,CALL OnHotKeyC                       // Ctrl+Alt+C
HKEY Ctrl+Shift+Alt+#0x44,CALL OnHotKeyD                 // Ctrl+Shift+Alt+D

// Virtual key codes can be numeric:
HKEY #0x0D,CALL OnEnter                                  // Enter key (VK_RETURN = 0x0D)
HKEY #0x1B,CALL OnEscape                                 // Escape (VK_ESCAPE = 0x1B)

// Win key modifier:
HKEY Win+#0x45,CALL OnWinE                               // Win+E
HKEY Win+Ctrl+#0x46,CALL OnWinCtrlF                      // Win+Ctrl+F
```

**Deletion:**
```wcs
HKEY Ctrl+#0x41,--del                                     // delete specific hotkey
HKEY --del                                               // delete ALL hotkeys
HKEY --del:0x41                                           // delete by key code
```

---

### 4.19 GROU — Group Box

```wcs
GROU [-center] Name,LxTyWwHh,[Text],[Style]
```

Visual grouping frame only — does not enforce mutual exclusion (use RADI groups for that).

```wcs
GROU Grp1,L10T10W200H100,Settings
GROU Grp2,L10T120W200H80,Options
GROU Grp1,L10T10W200H100,,0x0007                          // no text, sunken style
```

---

### 4.20 MEMO — Multi-line Text Box

```wcs
MEMO [-vcenter] [-rich] Name,LxTyWwHh,[InitText],[EventCmd],[Style],[FontSize]
```

Multi-line edit with built-in scroll support. Same operations as EDIT.

```wcs
MEMO Mem,L10T10W300H200,Initial text here,,0x0030        // VSCROLL + HSCROLL
ENVI @Mem=New multi-line\ntext                             // set with NL for newlines
```

---

### 4.21 MENU — Popup Menu / Window Menu Bar

#### Popup Menu

```wcs
_SUB MyPopupMenu
    MENU Open,Open File...,CALL OnOpen
    MENU Save,Save,CALL OnSave
    MENU -                                                // separator line
    MENU Exit,Exit,KILL \
_END

// Show at mouse position:
ENVI @Ctrl.MSG=_%&::WM_RBUTTONDOWN%: CALL @--popmenu MyPopupMenu

// Show at specific coordinates:
CALL @--popmenu MyPopupMenu 100:200                        // (x:y)
CALL @--popmenu MyPopupMenu 100:200:4                      // align right (1=left,2=up,4=right,8=down)
```

#### Window Menu Bar

```wcs
_SUB WinName,L10T10W400H300,Title,,,#,, -bar               // -bar = has menu bar
    MENU File,Open,CALL OnFileOpen
    MENU -sub:File FileOpen,Open,CALL OnFileOpen           // nested under File
    MENU -sub:File FileSave,Save,CALL OnFileSave
    MENU -
    MENU -sub:File FileExit,Exit,KILL \
    MENU Help,About,CALL OnAbout
_END
```

**Cascading sub-menus:**
```wcs
MENU -sub:ParentMenuItem ChildItem,Child Text,Command
MENU -sub:ParentMenuItem -                                 // separator in submenu
```

---

### 4.22 SCRN — Screen Capture

```wcs
SCRN [-cap] [-gui] Name,[filePath],[shape],[flags]
```

Captures screen or region to a file.

```wcs
SCRN -cap Scr,shot.bmp,L0T0W1920H1080                     // capture region
SCRN -cap Scr,shot.jpg                                    // capture full screen
SCRN -gui Scr,,                                            // interactive selection mode
```

---

## 5. Message Mapping & Event Handling

### `ENVI @Name.MSG=` — Message Handlers

```wcs
ENVI @ControlName.MSG=_msgId:command                      // control notification
ENVI @ControlName.MSG=msgId:command                       // direct window message
ENVI @WinName.MSG=msgId:command                           // window-level message
ENVI @this.MSG=msgId:command                              // "this" = current window
```

### Prefix Conventions

| Prefix | Meaning | Execution |
|--------|---------|-----------|
| `_` | Control notification (eg `_0x004E` = `_WM_NOTIFY`) | **Post** system handler. Use `_` for all WM_COMMAND/WM_NOTIFY subtypes. |
| (none) | Direct window message | Handler runs, then message is **discarded** (not passed to system) |
| `$` | Replace system handler | Handler runs **instead of** default window procedure |
| `*` | Chain handler | Handler runs **then** message is passed to default window procedure |
| `+` | Post handler | Handler is **posted** (asynchronous, runs after current processing) |

```wcs
ENVI @Btn1.MSG=_0x0201: CALL OnLeftClick                   // _ = control notification (WM_LBUTTONDOWN)
ENVI @Win.MSG=0x0010: CALL OnClose                          // WM_CLOSE, discard after handler
ENVI @Win.MSG=$0x0111: CALL OnCommand                        // replace system WM_COMMAND handler
ENVI @Win.MSG=*0x0005: CALL OnResize                         // chain: handle then pass to system
ENVI @Btn1.MSG=+0x0201: CALL OnClickDelayed                  // post: async handler
```

### Post / Send Messages

```wcs
ENVI @Ctrl.POSTMSG=#1                                       // post custom message #1
ENVI @Ctrl.POSTMSG=#2;wParam                                // post with wParam
ENVI @Ctrl.SENDMSG=#3;wParam;lParam                         // synchronous send
ENVI @@SENDMSG=wid;msg#;wParam;lParam                        // cross-process send
ENVI @@POSTMSG=wid;msg#;wParam;lParam                        // cross-process post
```

### Common Win32 Message IDs

```wcs
SET &::WM_CREATE=0x0001
SET &::WM_DESTROY=0x0002
SET &::WM_SIZE=0x0005
SET &::WM_CLOSE=0x0010
SET &::WM_KEYDOWN=0x0100
SET &::WM_COMMAND=0x0111
SET &::WM_SYSCOMMAND=0x0112
SET &::WM_NOTIFY=0x004E
SET &::WM_LBUTTONDOWN=0x0201
SET &::WM_LBUTTONUP=0x0202
SET &::WM_LBUTTONDBLCLK=0x0203
SET &::WM_RBUTTONDOWN=0x0204
SET &::WM_RBUTTONUP=0x0205
SET &::WM_MBUTTONDOWN=0x0207
SET &::WM_MBUTTONUP=0x0208
SET &::WM_MOUSEMOVE=0x0200
SET &::WM_MOUSEHOVER=0x02A1
SET &::WM_MOUSELEAVE=0x02A3
SET &::WM_MOUSEENTER=0x1000
SET &::WM_DROPFILES=0x0233
SET &::WM_DEVICECHANGE=0x0219
SET &::WM_SETCURSOR=0x0020
SET &::WM_CTLCOLOREDIT=0x0133
SET &::WM_CTLCOLORSTATIC=0x0138
SET &::WM_CTLCOLORBTN=0x0135
SET &::WM_CTLCOLORLISTBOX=0x0134
SET &::WM_VSCROLL=0x0115
SET &::WM_HSCROLL=0x0116
SET &::WM_TRAYNOTIFY=1109
```

### WM_COMMAND Subfield Access

When handling WM_COMMAND (`0x0111`):

```wcs
%&__wParam.wID%              // Control ID (the numeric ID of the control that sent the message)
%&__wParam.wNotifyCode%      // Notification code (BN_CLICKED=0, EN_CHANGE=0x300, LBN_SELCHANGE=1, etc.)
```

```wcs
ENVI @Win.MSG=_0x0111: CALL OnWM_COMMAND

_SUB OnWM_COMMAND
    FIND $%&__wParam.wNotifyCode%=0,                          // BN_CLICKED
    {
        FIND #%&__wParam.wID%=1001, CALL OnBtn1Click
        FIND #%&__wParam.wID%=1002, CALL OnBtn2Click
    }
_END
```

### WM_NOTIFY Subfield Access

When handling WM_NOTIFY (`0x004E`):

```wcs
%&__NMHDR.idFrom%            // Control ID that sent the notification
%&__NMHDR.code%              // Notification code (LVN_ITEMCHANGED=-100, NM_CLICK=-2, etc.)
%&__NMHDR.hwndFrom%          // HWND of the control
```

```wcs
ENVI @Tbl.MSG=_0x004E: CALL OnTableNotify

_SUB OnTableNotify
    FIND $%&__NMHDR.code%=-100,                                 // LVN_ITEMCHANGED
    {
        ENVI @Tbl.Sel=?;&row
        MESS Row %&row% selected/changed
    }
    FIND $%&__NMHDR.code%=-3,                                   // NM_DBLCLK
    {
        ENVI @Tbl.Sel=?;&row
        MESS Double-clicked row %&row%
    }
_END
```

---

## 6. Window Instantiation & Lifecycle

### CALL @ Variants

| Variant | Behavior |
|---------|----------|
| `CALL @WinName` | **Modal** — blocks caller until window closes |
| `CALL @*WinName` | **Parallel** — both caller and window run simultaneously |
| `CALL @-WinName` | **Background** — window runs; caller continues WITHOUT entering message loop |
| `CALL @~WinName` | **Background non-blocking** — fully decoupled; caller continues immediately |
| `CALL @+WinName` | **Abandoned child** — program can exit without waiting for this window |
| `CALL @^WinName` | **Parallel with parent priority** — like `@*` but parent doesn't block child's message loop |
| `CALL @WinName` (called twice) | If window exists, brings it to foreground |

### Class / Instance Pattern with `this`

```wcs
_SUB MyDialog
    ENVI @this.MSG=0x0010: CALL OnClose                       // handle WM_CLOSE
    ENVI @this.POS=?;&wL:&wT:&wW:&wH                          // query own position
    ENVI @this.Visible=0                                       // hide self
    ENVI @this.Visible=1                                       // show self
    ENVI @this.font=12:Microsoft YaHei                         // set window font
_END
```

`%&__WinID%` contains the current window's HWND.
`%&__LastWinID%` contains the last-created window's HWND.

### Window Destruction

```wcs
CALL @--WinName             // destroy the window environment (closes window)
KILL \                      // close current window (equivalent to X button)
KILL WinName                // close named window
KILL PidOrHwnd              // kill process or window by ID
```

### Window Enumeration

```wcs
FIND --wid*@[parentWID] &list,[titleFilter]                   // enumerate all windows
FIND --wid*@ &list,MyWindow                                   // find windows with "MyWindow" in title
FIND --wid* &list                                             // enumerate all top-level windows
// Output format: HWND\tTitle\tClass\nHWND\tTitle\tClass...
```

---

## 7. Complete GUI Example

```wcs
#code=936T950
ENVI^ EnviMode=1
ENVI^ ForceLocal=1
SET$ &NL=0d 0a
SET$ &TAB=09

// ── Window Message IDs ──
SET &::WM_LBUTTONDOWN=0x0201
SET &::WM_RBUTTONDOWN=0x0204

// ── Main Window ──
_SUB MainWin,L20T20W500H420,PECMD GUI Demo,KILL \,shell32.dll#1,, -size
    GROU Grp1,L10T10W230H130,Basic Controls
    LABE LblName,L20T35W50H20,Name:,,,8
    EDIT EdName,L75T33W150H20,,,0,10
    ITEM BtnBrowse,L230T33W50H20,...,CALL OnBrowse,0,8

    LABE LblType,L20T65W50H20,Type:,,,8
    LIST LstType,L75T63W150H100,File|Folder|Drive,,0,10
    ENVI @LstType.Sel=1
    ENVI @LstType.ADD=Custom

    CHEK ChkEnable,L20T95W120H20,Enable feature,CALL OnCheck,1
    CHEK ChkOpt,L160T95W100H20,Option,0

    GROU Grp2,L10T150W230H80,Radio Group
    RADI Rad1,L20T170W100H20:1,Small,,1                                       // Group 1, selected
    RADI Rad2,L20T190W100H20:1,Medium,,0
    RADI Rad3,L140T170W100H20:1,Large,,0

    ITEM BtnDo,L20T240W80H28,Do It,CALL OnDoIt
    ITEM BtnClear,L110T240W80H28,Clear,CALL OnClear
    PBAR PBar,L20T280W200H20,0

    // ── Table ──
    TABL Tbl,L260T10W225H200,=90:Name +60:Size =65:Date,0x820
    ENVI @Tbl.Val=1*;FileA.txt%&TAB%12 KB%&TAB%2025-01-01%&NL%FileB.exe%&TAB%256 KB%&TAB%2025-01-15

    // ── Timer ──
    TIME Timer1,1000,CALL OnTimer

    // ── Bottom buttons ──
    ITEM BtnOK,L340T380W70H28,OK,CALL OnOK,1,10
    ITEM BtnCancel,L420T380W70H28,Cancel,KILL \,0,10
_END

// ── Event Handlers ──
_SUB OnBrowse
    BROW &&path,&,Select a file...,*|*.txt|*.exe|All|*.*|
    FIND $%&path%<>, ENVI @EdName=%&path%
_END

_SUB OnCheck
    ENVI @ChkEnable.Check=?;&&st
    ENVI @ChkOpt.Enable=%&st%
_END

_SUB OnDoIt
    ENVI @PBar.Value=0
    ENVI @Timer1=50                                          // speed up for demo
    SET &::progress=0
    ENVI @EdName.Enable=0
_END

_SUB OnClear
    ENVI @Timer1=0
    ENVI @PBar.Value=0
    ENVI @EdName.Enable=1
_END

_SUB OnTimer
    CALC &::progress=%&::progress% + 2
    ENVI @PBar.Value=%&::progress%
    FIND |%&::progress%>=100, TEAM ENVI @Timer1=0| ENVI @EdName.Enable=1
_END

_SUB OnOK
    MESS Operation completed.@Information
    KILL \
_END

// ── Entry Point ──
CALL @MainWin
```

This example demonstrates:
- Window with resizing (`-size`)
- Group boxes (GROU)
- Labels (LABE), Edit boxes (EDIT), Browse button
- Dropdown list (LIST) with items
- Checkboxes (CHEK) with enable/disable logic
- Radio button group (RADI)
- Progress bar (PBAR)
- Data table (TABL) with grid lines and multi-select
- Timer (TIME) for periodic updates
- OK/Cancel buttons with event handlers
- Full message flow through CALL handlers
