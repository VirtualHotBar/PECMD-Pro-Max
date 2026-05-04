# PECMD2012 GUI 窗口系统参考

PECMD 窗口/控件系统的完整参考。涵盖窗口定义、全部 25 种控件类型、消息映射、生命周期以及高级 GUI 模式。

---

## 1. 窗口定义

### `_SUB` 窗口结构

```wcs
_SUB WinName,<shape>,<title>,[closeCmd],[icon],[style],[mask] [-flag1 -flag2 ...]
    // control definitions
_END
```

| Field | 语法 | 说明 |
|-------|--------|-------------|
| **shape** | `L<left>T<top>W<width>H<height>` | 窗口位置和大小。省略 `L`/`T` 可自动居中。`W`/`H` 为必填。 |
| **title** | `"窗口标题"` | 标题栏文字。空字符串 `""` 表示无标题。 |
| **closeCmd** | `KILL \` 或其他命令 | 窗口关闭时（X 按钮或 `KILL`）执行的命令。省略则使用默认关闭行为。 |
| **icon** | `path.ico` 或 `exe#index` | 窗口图标。文件路径或 EXE/DLL 资源引用。 |
| **style** | `[#][$]number` 或 `,#` | 窗口样式编号。`#` = 十六进制，`$` = 十进制。`,#` = 隐藏窗口。 |
| **mask** | `[color][*][w:h]bitmap` | 基于位图的异形/不规则窗口区域。`color` = 透明色键。 |
| **flags** | `-flag1 -flag2 ...` | 窗口行为标志（参见下表）。 |

### 全部窗口标志

| Flag | 效果 |
|------|--------|
| `-top` | 始终置顶 (TOPMOST) |
| `-nocap` | 无标题栏/标题 |
| `-nosysmenu` | 无系统菜单（标题栏无图标） |
| `-trap` | 关闭按钮不退出；改为触发 closeCmd |
| `-size` | 可调整大小的窗口（可调边框） |
| `-maxb` | 启用最大化按钮 |
| `-minb` | 启用最小化按钮 |
| `-disminb` | 禁用（灰掉）最小化按钮 |
| `-discloseb` | 禁用（灰掉）关闭按钮 |
| `-nfocus` | 窗口无法获得键盘焦点 |
| `-ntab` | 窗口排除在 Alt+Tab 列表之外（工具窗口） |
| `-disaltmv` | 禁用 Alt+方向键 移动窗口 |
| `-forcenomin` | 阻止窗口被最小化 |
| `-scalef` | XP 风格的 DPI 字体缩放 |
| `-scale[:DPI]` | Win8+ 逐显示器 DPI 缩放。`-scale:144` 指定显式 DPI。 |
| `-nxp` | 禁用 XP 视觉样式（经典扁平外观） |
| `-csize` | shape 尺寸指定**客户区**（不包括标题栏/边框） |
| `-na` | 不激活（创建时不抢夺焦点） |
| `-layer` | 支持渐变透明（可配合 `SetLayeredWindowAttributes`） |

### 样式：透明与隐藏窗口

```wcs
_SUB Win,L20T20W300H200,Title,,,#0x80000000       // 透明窗口 (alpha 0x80)
_SUB Win,L20T20W300H200,Title,,,$0x80C80000       // 透明窗口 (十进制样式)
_SUB Win,L20T20W300H200,Title,,,#                  // 隐藏窗口 (,# = 创建时隐藏)
```

### 遮罩：异形/不规则窗口

```wcs
_SUB Win,L20T20W300H200,Title,,,0xFFFFFF*mask.bmp // 颜色键异形窗口
_SUB Win,L20T20W300H200,Title,,,*300:200:mask.bmp  // 指定尺寸的位图遮罩
_SUB Win,L20T20W300H200,Title,,,0x00FF00**mask.bmp // 彩色遮罩（绿色=透明）
```

- `color`: 位图源窗口的透明色（RGB 十六进制：`0xFF0000` = 红色，`0x000000` = 黑色）
- `*`: 颜色与尺寸/源之间的分隔符
- `w:h:bitmap`: 显式尺寸与位图文件
- `*bitmap`: 自适配尺寸位图；双 `*` = 彩色遮罩模式

---

## 2. 控件类型概览

PECMD 通过专用命令提供 25 种控件类型。每种控件创建一个特定的 Windows 通用控件。

| # | 命令 | Windows 控件 | 用途 |
|---|---------|-----------------|---------|
| 1 | `ITEM` | 按钮 (Button) | 带文字/图片的可点击按钮，支持选中状态和默认操作 |
| 2 | `EDIT` | 编辑框 (Edit Box) | 单行/多行文本输入，密码模式，富文本 |
| 3 | `MEMO` | 多行编辑框 (Multi-line Edit) | 带内置滚动条的大文本区域 |
| 4 | `CHEK` | 复选框 (Check Box) | 二值/三态开关切换 |
| 5 | `RADI` | 单选按钮 (Radio Button) | 互斥的组内选择 |
| 6 | `LIST` | 组合框 (Combo Box) | 下拉列表或可编辑组合框，支持文字+图片项 |
| 7 | `LABE` | 静态文本 (Static Text) | 显示文本、多色彩、图片、可点击链接 |
| 8 | `IMAG` | 图片控件 (Picture Control) | BMP/JPG/GIF/AVI/ICO 显示，从 EXE/DLL 提取图标 |
| 9 | `PBAR` | 进度条 (Progress Bar) | 带文字覆盖层的可视化进度指示器 |
| 10 | `SLID` | 滑块 (Slider/Trackbar) | 可配置最小/最大值的范围滑块 |
| 11 | `SPIN` | 微调器 (Spinner/Up-Down) | 与编辑框配对使用的数字微调控件 |
| 12 | `GROU` | 分组框 (Group Box) | 视觉分组框架 |
| 13 | `TABL` | 列表视图 (ListView/Report) | 多列数据网格，支持复选框、图标、排序 |
| 14 | `TABS` | 选项卡控件 (Tab Control) | 属性页选项卡，用于嵌入了窗口 |
| 15 | `SWIN` | （自定义） | 子窗口/面板容器，用于嵌入其他窗口 |
| 16 | `DTIM` | 日期时间选择器 (Date-Time Picker) | 日期/时间选择，显示格式可配置 |
| 17 | `IPAD` | IP 地址控件 (IP Address Control) | 点分八位格式的 IP 地址输入 |
| 18 | `TIME` | 定时器 (Timer) | 周期性回调定时器（非可视化控件） |
| 19 | `HKEY` | 全局热键 (Global Hotkey) | 系统级键盘快捷键注册 |
| 20 | `MENU` | 菜单 (Menu) | 弹出式上下文菜单或窗口菜单栏 |
| 21 | `TIPS`/`TIPS*` | 托盘图标 (Tray Icon) | 系统托盘通知图标（`TIPS*` = 绑定到特定窗口） |
| 22 | `TREE` | 树形视图 (Tree View) | 可展开/折叠的层次节点列表，支持复选框 |
| 23 | `SBAR` | 滚动条 (Scroll Bar) | 独立滚动条控件，可绑定到其他控件 |
| 24 | `SCRN` | 屏幕捕捉 (Screen Capture) | 截图/区域捕获（非可视化） |
| 25 | `BROW` | 浏览对话框 (Browse Dialog) | 文件/目录选择对话框 |

---

## 3. 通用控件操作

### `ENVI @` 控件操作 — 全部控件

所有控件均通过 `ENVI @<Name>.<Operation>` 支持以下操作：

#### 文本与启用/可见

```wcs
ENVI @CtrlName=New Text                                // 设置控件文本
ENVI @CtrlName=%&newText%                              // 从变量设置文本
ENVI @CtrlName.Enable=0                                // 禁用（灰掉）
ENVI @CtrlName.Enable=1                                // 启用
ENVI @CtrlName.Visible=0                               // 隐藏
ENVI @CtrlName.Visible=1                               // 显示
ENVI @CtrlName.Visible=*4                              // 最小化窗口（仅窗口）
```

#### 位置与大小

```wcs
ENVI @CtrlName.POS=left:top:width:height               // 移动并调整大小
ENVI @CtrlName.POS=?;&L:&T:&W:&H                       // 查询位置到变量
ENVI @CtrlName.POS=?;&L;1;&T;1;&W;1;&H;1               // 查询，分号分隔输出
ENVI @CtrlName.POS=?%&WinName%;&L:&T:&W:&H             // 查询窗口位置
```

#### 外观

```wcs
ENVI @CtrlName.Font=12:Tahoma                           // 设置字体大小与字体名
ENVI @CtrlName.Font=12;Tahoma                          // 分号分隔符同样有效
ENVI @CtrlName.Font=12:Microsoft YaHei:Bold             // 带粗体样式
ENVI @CtrlName.Font=:                                    // 重置为默认字体
ENVI @CtrlName.bkcolor=0xFF0000                         // 设置背景颜色（BGR 十六进制）
ENVI @CtrlName.bkcolor=-2                               // 透明背景
ENVI @CtrlName.trans=1                                   // 半透明（仅分层窗口）
ENVI @CtrlName.Cursor=32649                             // 手形指针光标 (IDC_HAND)
ENVI @CtrlName.Cursor=32514                             // 普通箭头 (IDC_ARROW)
ENVI @CtrlName.Cursor=32515                             // I 形光标 (IDC_IBEAM)
```

#### 样式修改

```wcs
ENVI @CtrlName.Style=+0x1000                            // 添加窗口样式位
ENVI @CtrlName.Style=-0x1000                            // 移除窗口样式位
ENVI @CtrlName.Style=?;&styleVar                         // 查询当前样式
ENVI @CtrlName.ExStyle=+0x80                            // 添加扩展样式
ENVI @CtrlName.ExStyle=-0x80                            // 移除扩展样式
```

#### 销毁与失效

```wcs
ENVI @CtrlName.*del=                                     // 销毁控件（从窗口移除）
ENVI @CtrlName.InvalidateRect=                          // 强制完整重绘
ENVI @CtrlName.InvalidateRect=10:20:100:50               // 重绘区域 (l:t:w:h)
ENVI @CtrlName.InvalidateRect=<L:T:R:B>                  // 通过 LTRB 坐标重绘
ENVI @CtrlName.InvalidateRect=#WID                       // 通过窗口句柄使其失效
ENVI @CtrlName.InvalidateRect=@SubNAME                   // 通过子窗口名称使其失效
```

#### 高级控件属性

```wcs
ENVI @CtrlName.cmd=command                                // 动态命令绑定
ENVI @CtrlName.cmd=?var                                   // 查询命令（需要 QueryCmd=1）
ENVI @CtrlName.nxp=                                       // 禁用 XP 视觉样式
ENVI @CtrlName.trans=1[*]                                 // 0x1=背景透明，0x2=完全透明，*=透明色
ENVI @CtrlName.percent=[%][R|L|C|V|E|F][:bg:prog:text]  // 在任意控件上显示进度背景
ENVI @CtrlName.MouseCapture=1|0                          // 鼠标捕获模式 A
ENVI @CtrlName.MouseCapture=#1|#0                        // 鼠标捕获模式 B（通过句柄）
```

#### EDIT 专用

```wcs
ENVI @Ed.ReadOnly=0|1                                     // 0=可编辑，1=只读
ENVI @Ed.LINE=0|1|-1|:N                                   // 滚动到行（0/1=顶部，-1=底部，:N=相对）
```

#### ITEM/BUTTON 专用

```wcs
ENVI @Btn.color=0xRRGGBB                                  // 设置文本颜色
```

#### 窗口级属性

```wcs
ENVI @Wnd.Paint=callbackFunc                              // 画布回调（参数：HDC、宽度、高度）
ENVI @Wnd.style=[@*]remove[:add]                          // 修改窗口样式（@*=跨进程）
ENVI @Wnd.HitTest=[-]height[:w:x:y]                       // 拖动命中测试（0=取消，-=半透明）
ENVI @Wnd.Font=size[:name[style]]                         // 设置窗口字体
ENVI @Wnd.trans=0|1|2[*]                                 // 0x1=背景透明，0x2=完全透明，*=透明色
```

#### 跨进程操作（使用 WID）

```wcs
ENVI @@Enable=?WID:varName                                // 查询跨进程启用状态
ENVI @@IsWindow=?WID:varName                              // 检查 WID 是否为有效窗口
ENVI @@style=%WID%:[@*]remove:add                        // 跨进程样式更改
ENVI @@percent=WID:...                                     // 跨进程进度
ENVI @@<win|mess|help|login>.font=                         // 设置系统对话框字体
```

#### 同时操作多个控件

```wcs
ENVI @MultipleCtrl.VAL=%&data%                          // 对匹配模式的控件设置 VAL
ENVI @MultipleCtrl.Enable=0                             // 禁用所有匹配的控件
ENVI @MultipleCtrl.Visible=0                            // 隐藏所有匹配的控件
// MultipleCtrl 为通配符：@Btn* 匹配 Btn1、Btn2、BtnSave 等
```

#### 禁用 vs 隐藏 vs 灰掉

```wcs
ENVI @Ctrl.Enable=0       // 灰掉，可见但不可交互
ENVI @Ctrl.Visible=0      // 隐藏，不占用布局空间
ENVI @Ctrl.Enable=1       // 完全可交互
```

---

## 4. 控件详细参考

### 4.1 ITEM — 按钮

```wcs
ITEM [-na] [-b[:父窗句柄]] [-right] [-left] [-def] [-font:字体大小:字体名及修饰] [*] Name,LxTyWwHh,[标题],[事件],[图标],[状态]
```

| Flag | 用途 |
|------|---------|
| `-def` | 默认按钮（回车键触发），黑色边框 |
| `-right` | 按钮文字右对齐 |
| `-round` | 圆角（仅限有主题的 XP+） |
| `-na` | 不激活/不获取焦点 |
| `-font:N` | 字体大小 N |

**状态值：**
| Value | 含义 |
|-------|---------|
| `0` | 可用（默认） |
| 负数 | 灰色禁用 |
| `4` | 多行文本 |
| `0x10` | 不可见 |

**图片按钮（嵌入 IMAG）：**
```wcs
IMAG Img1,L10T10W32H32,myicon.ico
ITEM Btn1,L20T10W100H32,Imag1Click me!,CALL OnClick
// 嵌入图片：将 IMAG 的控件名用作按钮面板
```

**Command：** 点击时执行。使用 `CALL FuncName [args]` 进行函数调用。

```wcs
ENVI @Btn1.Check=1          // 设置切换按钮为选中状态
ENVI @Btn1.Check=?;&state   // 查询状态
```

---

### 4.2 EDIT — 编辑框

```wcs
EDIT[-|+.*=] [-right] [-center] [-vcenter[:缩小量]] [-rich] [-3D] [*] <名称>,<形状>,[内容],[事件],[类型],[颜色],[字体]
```

| Flag | 用途 |
|------|---------|
| `-` | 前缀标志：水平滚动条 |
| `\|` | 前缀标志：垂直滚动条 |
| `+` | 前缀标志：无边框 |
| `.` | 前缀标志：不转换 `\n`（否则自动转换） |
| `*` | 前缀标志：预解释事件（变量展开） |
| `=` | 前缀标志：编辑框内容是文件名（加载文件内容） |
| `-right` | 文本右对齐 |
| `-center` | 文本居中 |
| `-vcenter[:缩小量]` | 垂直居中文本（仅单行）；可指定缩小量 |
| `-rich` | 富文本编辑控件（支持格式化、颜色） |
| `-3D` | 3D 轮廓外观 |
| `[*]` | 退出代码块/函数时自动回收 |
| 类型值 `1` | 密码输入框（每个字符显示 `*`） |

前缀标志必须紧跟 `EDIT` 命令后，无空格，无顺序之分。

**滚动条与前缀标志：**
```wcs
EDIT- Ed,L10T30W200H200          // 水平滚动条
EDIT| Ed,L10T30W200H200          // 垂直滚动条
EDIT-| Ed,L10T30W200H200         // 双向滚动条
EDIT+ Ed,L10T30W200H200          // 无边框
EDIT= Ed,L10T30W200H200          // 内容来自文件
```

**类型值（累加）：**
| Value | 含义 |
|-------|---------|
| 0 | 正常编辑框 |
| 1 | 密码输入框 |
| 2 | 禁用（灰色） |
| 3 | 只读（兼容） |
| 4 | 多行 |
| 8 | 只读 |
| 0x10 | 不可见 |
| 0x20 | 可编辑并支持自动换行 |
| 0x40 | 跳到末尾 |
| 0x100 | 接受一个拖入文件名 |
| 0x200 | 接受所有拖入文件名，多行 |
| 0x400 | 数字 |
| 0x800 | 跳到行尾 |
| 0x1000 | 按字符换行（兼容） |

**颜色：** `文字颜色#背景颜色[#活动文字颜色#活动背景颜色]`，省略时为默认颜色。

**字体：** `字体大小[~^][:字体名]`。`~` 反缩放，`^` 内核大小。字体名可附带修饰：`[**BbUuIiSs#Weight#Width#CharSet#Quality#...]`。

**文本操作：**
```wcs
ENVI @Ed.QUERY=;&textVar                 // 获取全部文本
ENVI @Ed=New Text                         // 设置全部文本（替换）
ENVI @Ed.SET=Text to set                 // 设置文本（等同于 =）
ENVI @Ed.SEL=start:end                    // 选择文本范围（从 0 开始）
ENVI @Ed.SEL=?;&start;&end               // 查询选择范围
```

---

### 4.3 LABE — 标签（静态文本）

```wcs
LABE[-|+.*>] [-right -center -left -trans -nf -w -vcenter -ncmd -3D -mod] [*] <名称>,<形状>,[文字],[*][命令],[颜色集合],[字体大小:[字体名]]
```

| Flag | 用途 |
|------|---------|
| `-` | 前缀标志：水平滚动条（只能看不能动） |
| `\|` | 前缀标志：垂直滚动条 |
| `+` | 前缀标志：带边框 |
| `.` | 前缀标志：不转换 `\n` |
| `>` | 前缀标志：ENVI @ 转换 `\n` |
| `*` | 前缀标志：预解释事件（变量展开） |
| `-right` | 右对齐 |
| `-center` | 居中对齐 |
| `-left` | 左对齐（默认） |
| `-trans` | 透明背景（透出父窗口） |
| `-nf` | 不闪（no flash） |
| `-w[x]` | 自动换行（`-wx` 可断词） |
| `-vcenter` | 垂直居中文本（仅单行） |
| `-ncmd` | 禁用命令特性（使其不可点击/静态） |
| `-3D` | 3D 轮廓外观 |
| `-mod` | 凸起/凸面外观（raised/convex） |
| `[*]` | 退出代码块/函数时自动回收 |

前缀标志必须紧跟 `LABE` 命令后，无空格，无顺序之分。

**多色彩文本支持：**
```wcs
ENVI @MyLabel.Color=1:0xFF0000;3:0x00FF00   // 第 1 个词 = 红色，第 3 个词 = 绿色
ENVI @MyLabel.Color=0x0000FF                 // 全部文本蓝色
// 颜色格式：词位置:0xBBGGRR;词位置:0xBBGGRR
// 词位置从 1 开始，按空格分隔的标记计数
```

**嵌入图片的标签：**
```wcs
LABE ImgLbl,L10T10W200H32,#0x0100|image.bmp,Text here,,0x0000FF
// 在文本前加上 [图片源] 以在文本旁嵌入图片
```

**可点击标签：**
```wcs
LABE ClickLbl,L10T50W100H20,Click Me,CALL OnLabelClick   // 命令使其可点击
```

**可点击网页链接标签：**
```wcs
LABE LinkLbl,L10T80W200H20,Visit Site,EXEC $http://example.com
// 第4个参数为 EXEC 命令时，鼠标悬停显示手形光标，点击执行命令
```

---

### 4.4 CHEK — 复选框

```wcs
CHEK [-right -center -scale[:[H_Dpi][[<sW;sH>]:图片]]] [*] <名称>,<形状>,[标题,事件,状态]
```

| Flag | 用途 |
|------|---------|
| `-right` | 勾选标记在文字右侧 |
| `-center` | 居中对齐 |
| `-scale[:...]` | 随 DPI 缩放；可指定 `H_Dpi`、`sW;sH` 尺寸和图片 |
| `[*]` | 退出代码块/函数时自动回收 |

**状态值：**
| Value | 含义 |
|-------|---------|
| `0` | 未勾选 |
| `1` | 已勾选 |
| `2` | 未勾选（同 0，help.txt: "0，2或-2为没有钩选"） |

**操作：**
```wcs
ENVI @Chk.Check=1                  // 勾选
ENVI @Chk.Check=0                  // 取消勾选
ENVI @Chk.Check=?;&var             // 查询状态（分号在变量名前）
ENVI @Chk.Check=?;0;&var           // 查询，0=使用分号模式
```

---

### 4.5 RADI — 单选按钮

```wcs
RADI [-right] [-scale] [-center] Name,LxTyWwHh,[标题],[事件],[状态],[组ID]
```

单选按钮在同一**组**内互斥。**组 ID 是第 6 个参数**（非 shape 的一部分）：
```wcs
RADI R1,L10T10W100H20,Option A,CALL OnSel,1,1           // 组 1，初始选中
RADI R2,L10T30W100H20,Option B,,0,1                     // 组 1
RADI R3,L10T60W100H20,Option C,,0,1                     // 组 1

RADI R4,L10T90W100H20,Option X,CALL OnSel2,1,2           // 组 2
RADI R5,L10T110W100H20,Option Y,,0,2                     // 组 2
```

组 ID 默认为 0。组中的第一个单选按钮通常设 `State=1`（默认选中）。

**查询：**
```wcs
ENVI @R1.Check=?;&selectedState        // 若此单选按钮被选中则为 1，否则为 0
// 要查找组中哪个单选按钮被选中，需逐一查询各单选按钮的 .Check
```

---

### 4.6 LIST — 下拉列表/组合框

```wcs
LIST [-h] Name,LxTyWwHh,item1|item2|item3,[EventCmd],[默认选中条目],[状态]
```

| Flag | 用途 |
|------|---------|
| `-h` | 展开/下拉高度（像素） |

**状态值（累加）：**
| Value | 含义 |
|-------|---------|
| `0x4` | 可编辑列表（用户可输入自定义文本） |
| `0x10` | 不可见 |
| `0x100` | 自动垂直滚动条 |
| `0x200` | 简单列表 |
| `0x400` | 自动排序 |
| `0x800` | 大写 |
| `0x1000` | 小写 |
| `0x2000` | 编辑不触发命令 |

**初始项目：** 管道符分隔的列表：`"Apple|Banana|Cherry|Date"`

**操作：**
```wcs
ENVI @List.VAL=                                    // 清空所有项目
ENVI @List.ADD=New Item                            // 在末尾添加一个项目
ENVI @List.ADDSEL=New Item                         // 添加项目并选中
ENVI @List.DEL=Item Text                           // 按文本删除项目（help.txt: DEL=被删除的条目）
ENVI @List.isel=3                                   // 按索引选中（从 1 开始）
ENVI @List.Sel=Item Text                            // 按文本选中
ENVI @List.Sel=3;0                                  // 取消选中
ENVI @List.Sel=?;&selectedIndex                     // 获取从 1 开始的选中索引
ENVI @List.QUERY=;&allItems                         // 获取所有项目（换行符分隔）
ENVI @List.Val=?*;&itemCount                        // 获取项目数量
ENVI @List.Val=?;itemText                           // 获取选中项目文本

// 通过 % 变量访问：
%List.isel%                                         // 选中索引（从 1 开始）
%List.Sel%                                          // 同上
```

**带图片的列表：**
```wcs
LIST ImgLst,L10T10W150H200,
ENVI @ImgLst.ADD=.ico#5|Item with icon             // 从资源中获取图标索引
```

**可编辑组合框：**
```wcs
LIST EdtList,L10T10W150H20,Item1|Item2,,0,0x4      // 0x4 = 可编辑组合框
ENVI @EdtList=User typed text                       // 获取/设置当前文本
```

---

### 4.7 IMAG — 图片/图像显示

```wcs
IMAG [-gui|-size|-real|-sel|-bupdate] [*] <框名>,[形状],[资源],[命令],[边框颜色],[边框线宽]
```

| Flag | 用途 |
|------|---------|
| `-gui` | GUI 模式 |
| `-size` | 尺寸模式 |
| `-real` | 实际尺寸模式 |
| `-sel` | 选择模式 |
| `-bupdate` | 强制为图片文件浏览模式 |
| `-smooth` | 光滑显示 |
| `-tab` | TAB 键切换 |
| `[*]` | 退出代码块/函数时自动回收 |

**支持的格式：** `.bmp`、`.jpg`、`.jpeg`、`.gif`（动画）、`.avi`（动画）、`.ico`

**参数：**
- **边框颜色：** 由正常颜色和活动颜色组成，`#` 分隔，如 `0x00FFFF#0xFF0000`。省略采用系统默认颜色。
- **边框线宽：** 像素大小。`-16` 不可见。执行命令省略或无效时活动颜色无效。

**从 EXE/DLL 提取图标：**
```wcs
IMAG Ico,L10T10W32H32,C:\Windows\System32\shell32.dll#23    // 图标索引 23
IMAG Ico,L10T10W32H32,%SystemRoot%\explorer.exe#0           // 第一个图标
```

**动态图像更新：**
```wcs
ENVI @Img=NewImage.jpg                              // 运行时更换图片
ENVI @Img=shell32.dll#42                            // 更换为不同图标
ENVI @Img=                                          // 清除图像
ENVI @Img.update=32:32:100:50::;shell32.dll#52      // 更换图标（update 语法）
```

---

### 4.8 PBAR — 进度条

```wcs
PBAR [*] [-smooth] Name,LxTyWwHh,[InitPercent]
```

**操作：**
```wcs
ENVI @PBar=50                                       // 设为 50%
ENVI @PBar=50;#00FF00Processing...                  // 设为 50%，绿色文字
ENVI @PBar.color=0xFF0000                           // 文字颜色（BGR 红色）
ENVI @PBar.percent=-smooth                          // 切换平滑模式
ENVI @PBar.Visible=0                                // 隐藏（-1 也隐藏）
```

**范围：** 1–100（help.txt: "浮点数(1~100)"）。0 为默认初始状态。

---

### 4.9 TABL — 表格/数据网格（最详细）

```wcs
TABL [-font:N] Name,LxTyWwHh,[HeaderString],[Flags],[Style],[FontSize]
```

**表头格式：**
```
HeaderString = "col1_text:col1_width col2_text:col2_width ..."
```
宽度前缀：
| 前缀 | 含义 |
|--------|---------|
| `*` | 左对齐（默认） |
| `=` | 右对齐列 |
| `+` | 居中对齐列 |
| (无) | 左对齐（默认） |
| (空名称) | 此列无表头文本 |

```wcs
TABL Tbl,L10T10W400H200,=100:Name +80:Size =120:Date *30,0x40
// 第 1 列："Name"，宽度 100，右对齐
// 第 2 列："Size"，宽度 80，居中
// 第 3 列："Date"，宽度 120，右对齐
// 第 4 列：（空），宽度 30，复选框
```

**完整状态标志表（权威来源：help.txt 3276-3281）：**

| Flag (十六进制) | Flag (十进制) | 名称 | 说明 |
|------------|------------|------|-------------|
| `0x10` | 16 | 不可见 | 隐藏表格 |
| `0x40` | 64 | 有边框 | 显示边框 |
| `0x80` | 128 | 无水平滚动条 | 隐藏水平滚动条 |
| `0x100` | 256 | 无垂直滚动条 | 隐藏垂直滚动条 |
| `0x200` | 512 | 无网格线 | 隐藏单元格网格线 |
| `0x400` | 1024 | 带打勾器 | 每行有复选框（勾选） |
| `0x800` | 2048 | 拖拉标题调整列顺序 | 可拖拽表头重排列序 |
| `0x2000` | 8192 | 无标题 | 隐藏列标题行 |
| `0x4000` | 16384 | 禁止调整宽度 | 固定列宽 |
| `0x10000` | 65536 | 仅单行选择 | 只能选择(加亮)一行 |
| `0x40000` | 262144 | 可双击选单元 | 双击选中单元格 |
| `0x80000` | 524288 | 禁用行选择 | 禁止鼠标行选择 |
| `0x100000` | 1048576 | 勾选行着色 | 勾选的行显示背景色 |
| `0x200000` | 2097152 | 第一列可含图片 | 图片列（第一列） |
| `0x400000` | 4194304 | 所有列可含图片 | 图片列（全部列） |
| `0x800000` | 8388608 | 画横线 | 绘制水平线 |
| `0x1000000` | 16777216 | 画竖线 | 绘制垂直线 |
| `0x4000000` | 67108864 | TAB 切换 | 支持 TAB 键切换焦点 |
| `0x8000000` | 134217728 | 选择图片 | 图片选择模式 |

注意：选择（高亮行）和勾选（打勾器）是两套独立方案！

```wcs
// 常用标志组合：
TABL Tbl,L10T10W400H200,Col1:100 Col2:80,0x400      // 带打勾器
TABL Tbl,L10T10W400H200,Col1:100 Col2:80,0x180400   // 勾选行着色 + 禁用行选择 + 打勾器
TABL Tbl,L10T10W400H200,Col1:100 Col2:80,0xC0000    // 双击选单元 + 禁用行选择
TABL Tbl,L10T10W400H200,Col1:100 Col2:80,0x10000    // 仅单行选择（单击行着色）
```

**数据操作：**

```wcs
// --- 设置数据 ---
ENVI @Tbl.Val=1*;%&allData%                         // 从变量批量设置所有行
// allData 格式：row1col1\trow1col2\nrow2col1\trow2col2\n...
// \t = 列之间的 TAB，\n = 行之间的换行

ENVI @Tbl.Val=%row%;col1%&TAB%col2%&TAB%col3       // 设置单行（从 1 开始）
ENVI @Tbl.Val=%row%;*                                // 删除单行

// --- 获取数据 ---
ENVI @Tbl.Val=?%row%.%col%;&cellValue               // 获取单元格（分号在变量前）
ENVI @Tbl.Val=?*;&rowCount                           // 获取总行数
ENVI @Tbl.Val=?*;&rowCount;&colCount                 // 获取行数和列数
ENVI @Tbl.Val=?%row%;&fullRowData                   // 获取整行（TAB 分隔）
ENVI @Tbl.Val=?*.*;&allData                         // 获取全部数据（行间换行）

// --- 清除 ---
ENVI @Tbl.Val=-*                                     // 清空所有行

// --- 删除行 ---
ENVI @Tbl.Val=-%row%                                 // 删除指定行
ENVI @Tbl.Val=-*%row%                                // 删除该行到尾部
ENVI @Tbl.Val=-%row%#%count%                         // 删除多行

// --- 列操作 ---
ENVI @Tbl.Val=.-%col%                                // 删除指定列
ENVI @Tbl.Val=+;#0xFF0000#0xFFFFFF=80:NewCol         // 增加列（颜色#背景色=宽度:标题）

// --- 选择（行） ---
ENVI @Tbl.Sel=%row%                                  // 选中行（高亮）
ENVI @Tbl.Sel=%row%;0                                 // 取消选中行
ENVI @Tbl.Sel=%row%;2                                 // 乒乓选择
ENVI @Tbl.Sel=?;&selectedRow                          // 获取选中行号
ENVI @Tbl.Sel=?*;&allSelected                         // 获取所有选中行（空格分隔）

// --- 选择（单元格） ---
ENVI @Tbl.Sel=+%row%;%col%                           // 设置当前选择单元位置（等同双击）
ENVI @Tbl.Sel=?+;&cellRow;&cellCol                   // 查询当前选择单元位置
ENVI @Tbl.Sel=?.;&mouseRow;&mouseCol                 // 查询鼠标下单元位置

// --- 勾选状态 ---
ENVI @Tbl.Check=%row%;1                               // 勾选行复选框
ENVI @Tbl.Check=%row%;0                               // 取消勾选行复选框
ENVI @Tbl.Check=%row%;2                               // 乒乓勾选
ENVI @Tbl.Check=?%row%;&checkState                    // 查询行复选框 (1/0)
ENVI @Tbl.Check=?*;&allChecked                        // 获取所有勾选行（空格分隔）

// --- 行使能 ---
ENVI @Tbl.Enable=~%row%;0                             // 禁用行（灰色）
ENVI @Tbl.Enable=~%row%;1                             // 启用行
ENVI @Tbl.Enable=~%row%;2                             // 乒乓使能
ENVI @Tbl.Enable=~-%row%                              // 禁用且取消选择
ENVI @Tbl.Enable=~?%row%;&enableState                 // 查询行使能状态
ENVI @Tbl.Enable=~?*;&allDisabled                     // 获取所有禁用行

// --- 行颜色 ---
ENVI @Tbl.Color=%row%;0xFF0000                        // 设置行文本颜色 (BGR)
ENVI @Tbl.Color=%row%;0xFF0000;0xFFFFFF              // 文本颜色;背景颜色
ENVI @Tbl.Color=%row%.%col%;0xFF0000                 // 设置单元格颜色
ENVI @Tbl.Color=%col%;0xFF0000                        // 设置列颜色
ENVI @Tbl.Color=*%row%;0xFF0000                       // 设置行颜色（*前缀）

// --- 行图标 ---
ENVI @Tbl.Val=%row%;.ico#5;col1%&TAB%col2           // 设置带图标 #5 的行

// --- 位置 / 滚动 ---
ENVI @Tbl.UPOS=?;&rowIndex                           // 获取顶部可见行索引
ENVI @Tbl.UPOS=?*@%row%;L;T;R;B                     // 查询行矩形位置
ENVI @Tbl.UPOS=?*%row%.%col%;L;T;R;B                // 查询单元格位置

// --- 百分比操作（用于类似 PBAR 的列） ---
ENVI @Tbl.Percent=?%row%;&percentVal                  // 查询百分比列
ENVI @Tbl.Percent=%row%.%col%;50                      // 设置百分比
```

---

### 4.10 TABS — 选项卡控件/属性页

```wcs
TABS *Name,LxTyWwHh,ClassName1[:InstName1][:Title1][:Tip1];ClassName2[:InstName2][:Title2][:Tip2];...,[Status]
```

- **页面分隔符是分号 `;`**（不是 `|`）
- 每页格式：`ClassName[:InstanceName][:Title][:Tip]`
- ClassName 是 `_SUB` 定义的窗口类名；InstanceName 用于引用该页内的控件
- 第 4 个参数是**状态标志**（非事件回调）：负值=灰色不可用、`0x10`=不可见、`4`=多行、`0x40`=有边框、`0x80`=竖排
- 名称前的 `*` 表示退出代码块或函数时自动回收
- TABS 自动为每页创建 SWIN，无需手动声明

```wcs
// 定义页面窗口类
_SUB PageGeneral,W380H260,General
    LABE Lbl1,L10T10W360H24,常规设置
_END

_SUB PageAdvanced,W380H260,Advanced
    LABE Lbl2,L10T10W360H24,高级设置
_END

_SUB PageAbout,W380H260,About
    LABE Lbl3,L10T10W360H24,关于
_END

// 创建选项卡（注意分号分隔页面）
TABS TabMain,L10T10W400H300,PageGeneral:Gen:常规:提示1;PageAdvanced:Adv:高级:提示2;PageAbout:Abt:关于:提示3

// 页面切换：
ENVI @TabMain.Select=2                            // 切换到第 2 页（从 1 开始）
ENVI @TabMain.Select=?;&currentPage               // 查询当前页面

// 通过 InstanceName 访问页面内控件
ENVI @Gen:Lbl1=更新文本                             // 访问 InstanceName "Gen" 内的 "Lbl1"
```

---

### 4.11 SWIN — 子窗口/面板嵌入

```wcs
SWIN [*] [画框名]:类名:[实例名],<形状>,[内部位置],[状态]
```

- 画框名（FrameName）：可选，框架控件名
- 类名（ClassName）：`_SUB` 定义的窗口类名
- 实例名（InstanceName）：可选，用于引用该实例
- `*` 表示退出代码块或函数时自动回收
- 状态：负数=灰色禁用，`0x10`=不可见，`0x40`=有边框，`0x80`=水平滚动条，`0x100`=垂直滚动条，`0x200`=标题栏

```wcs
_SUB Page1
    LABE Lbl,L10T10W200H20,This is Page 1
    ITEM Btn,L10T40W80H28,Click,CALL OnClick1
_END

// 简单形式（画框名:类名）：
SWIN SwinPg1:Page1,L20T40W380H260     // 将 Page1 作为子窗口嵌入

// 完整形式（画框名:类名:实例名）：
SWIN Frame1:Page1:PgInst,L20T40W380H260    // 区分框架、类和实例
```

**多面板管理：**
```wcs
// 创建多个 SWIN，只显示当前活动的那一个
SWIN Panel1:Page1Win,L20T40W380H260
SWIN Panel2:Page2Win,L20T40W380H260
SWIN Panel3:Page3Win,L20T40W380H260

ENVI @Panel2.Visible=0                 // 隐藏不用的面板
ENVI @Panel3.Visible=0

// 切换面板：
_SUB SwitchToPanel2
    ENVI @Panel1.Visible=0
    ENVI @Panel2.Visible=1
_END
```

---

### 4.12 SLID — 滑块/轨道条

```wcs
SLID [-right] [-left] [*] Name,LxTyWwHh,[起始值:终到值:初值:页大小],[事件命令],[状态]
```

```wcs
SLID Sld,L10T10W200H30,0:100:50:10,CALL OnSlide       // 初始 50，范围 0-100，页步长=10
SLID Sld,L10T10W200H30,1:10:5                          // 初始 5，范围 1-10
SLID Sld,L10T50W200H30,-20:80:0                         // 支持负的最小值
```

**操作：**
```wcs
ENVI @Sld.VAL=75                                     // 设置滑块位置
ENVI @Sld.VAL=?;&value                                // 查询滑块值
```

---

### 4.13 SPIN — 微调器/上下控件

```wcs
SPIN [-right] [-left] [*] <名称>,<形状>[,值信息][,命令参数名][,命令][,状态]
```

**值信息：** `[伙伴EDIT名][:起始值][:终到值][:初值]`。默认 `0:100:0`。EDIT 名优先于自动结伴。

**命令参数名：** `[新值名][:按钮名][:旧值名]`。按钮名返回 0/1，对应下按钮/上按钮。

SPIN 自动与一个配对 EDIT 控件组合用于数字输入：

```wcs
EDIT Edit1,L10T10W60H20,0
SPIN Spin1,L70T10W16H20,Edit1:0:100:0,CALL OnSpin

// 显式配对：
EDIT Edit2,L10T40W60H20,0
SPIN Spin2,L70T40W16H20,Edit2:0:255:0

// 自动结伴前一控件（状态 0x80）：
EDIT Edit3,L10T70W60H20,0
SPIN Spin3,L70T70W16H20,,,0x80                      // 自动结伴 Edit3
```

**状态值：**
| Value | 含义 |
|-------|---------|
| <0 | 灰色禁用状态 |
| 0x10 | 不可见 |
| 0x20 | 回绕（到达最大值后循环到最小值） |
| 0x40 | 水平方向 |
| 0x80 | 自动结伴前一控件 |

**操作：**
```wcs
ENVI @Spin1.VAL=50                                    // 设置微调器值
ENVI @Spin1.VAL=?;&value                              // 查询值
ENVI @Spin1.VAL=cur:start:end                         // 设置值信息
ENVI @Spin1.VAL=?curVar:startVar:endVar               // 查询值信息
```

---

### 4.14 DTIM — 日期时间选择器

```wcs
DTIM Name,LxTyWwHh,[InitDateTime],[EventCmd],[Style]
```

类型通过 Style 参数中的位标志设置：

| 位 | 十六进制 | 说明 |
|-----|-----|-------------|
| — | 0x00 | 短日期格式（默认） |
| 0x20 | 0x20 | 长日期格式 |
| 0x40 | 0x40 | 时间格式 |
| 0x80 | 0x80 | 短世纪日期格式 |
| 0x100 | 0x100 | 上/下键调整 |
| 0x200 | 0x200 | 复选框选择器 |
| 0x10 | 0x10 | 不可见 |
| <0 | (负数) | 灰掉（禁用） |

```wcs
DTIM Dt1,L10T10W120H22,,CALL OnDateChange,0x20         // 长日期格式
DTIM Dt2,L10T40W80H22,,,0x40                             // 时间格式
DTIM Dt3,L10T70W180H22                                    // 默认：短日期
DTIM Dt4,L10T100W180H22,,,0x60                            // 长日期 + 时间 (0x20|0x40)
```

**操作：**
```wcs
ENVI @Dt1.VAL=2025;1;15                                 // 设置日期 (年;月;日 分号分隔)
ENVI @Dt1.VAL=?&&yr;&&mo;&&dy;&&wf;&&tf                 // 查询：年/月/日/星期标志/时间标志
ENVI @Dt2.VAL=14;30;0                                    // 设置时间 (时;分;秒)
```

---

### 4.15 IPAD — IP 地址控件

```wcs
IPAD Name,LxTyWwHh,[InitIP],[EventCmd],[Style]
```

```wcs
IPAD Ip1,L10T10W140H22,192.168.1.1,CALL OnIPChange
IPAD Ip2,L10T40W140H22,,CALL OnIPChange                 // 默认为 0.0.0.0
```

**操作：**
```wcs
ENVI @Ip1.VAL=10.0.0.1                                  // 设置 IP
ENVI @Ip1.VAL=?;&ipString                               // 查询："192.168.1.100"
```

---

### 4.16 TIME — 定时器

```wcs
TIME TimerName,interval,[EventCmd]
```

`TimerName` 应具有描述性名称（社区惯例使用 `Timer` 前缀，如 `Timer1`、`TimerMain`，但 help.txt 无强制要求）。

```wcs
TIME Timer1,1000,CALL OnTick                            // 每 1000ms 触发
TIME Timer1,500,                                        // 每 500ms，无命令（必须主动查询）
TIME *Timer2,500,CALL OnTick                            // * = 退出代码块/函数时自动回收
TIME Timer3,0,CALL OnFire                               // 0 = 立即触发一次
```

**定时器标志 `-t:N`：**
| Flag | 用途 |
|------|---------|
| `-t:1` | 单次定时器（触发一次后停止） |

**操作：**
```wcs
ENVI @Timer1=500                                        // 更改间隔
ENVI @Timer1=0                                           // 暂停/停止
ENVI @Timer1=-del                                        // 销毁定时器

// 多定时器：
TIME TimerPulse,100,CALL Heartbeat
TIME TimerLong,5000,CALL PeriodicCheck
TIME TimerOnce,0,CALL DelayedInit
```

---

### 4.17 TIPS* — 系统托盘图标

```wcs
TIPS* [标题],[内容],[寿命],[图标样式ID],[托盘图标],[#WID]
```

| 参数 | 说明 |
|-------|-------------|
| `标题` | 气泡提示框标题（最大 64 字符） |
| `Content` | 工具提示文本/托盘标签（最大 256 字符，`\n` 换行） |
| `timeout` | 气泡存活时间（毫秒）（默认 10 秒，0=永久） |
| `iconStyleID` | 0=无，1=信息图标，2=警告图标，3=错误图标，4+=托盘图标 |
| `trayIcon` | 图标文件路径或 `#resID`（如 `shell32.dll#94`） |
| `#WID` | 关联的窗口句柄 |

点击处理通过 `WM_TRAYNOTIFY` (1109) 消息映射完成——没有点击处理参数：

```wcs
SET &WM_TRAYNOTIFY=1109
SET &WM_LBUTTONDOWN=0x0201
SET &WM_RBUTTONDOWN=0x0204

CALL @WinMain
_SUB WinMain,#
    ENVI @this.MSG=_%&WM_TRAYNOTIFY%::&&wp,&&lp,CALL DoTrayClick %&wp% %&lp%
    TIPS* WinMain,My Tool,,,shell32.dll#94
_END

_SUB DoTrayClick
    IFEX $%2=%&WM_RBUTTONDOWN%, CALL @--popmenu TrayMenu    // %2=lParam=鼠标消息（WM_TRAYNOTIFY 中 lParam 携带鼠标消息码）
_END

_SUB TrayMenu
    MENU Show,Show Window,ENVI @MyWin.Visible=1
    MENU Hide,Hide Window,ENVI @MyWin.Visible=0
    MENU -
    MENU Exit,Exit,KILL \
_END
```

**气泡通知：**
```wcs
TIPS MyTitle,Hello World\nLine 2,5000,1                     // 信息图标，5 秒
TIPS* WinMain,Status update,,2,#1                            // 警告图标，资源图标
```

**清除：**
```wcs
TIPS -                                                       // 清除气泡
TIPS *                                                       // 清除所有托盘 + 气泡
```

---

### 4.18 HKEY — 热键注册

```wcs
HKEY[$*] [辅助键+]<按键字母|#虚拟按键代码>,<热键命令>
```

| 后缀 | 作用域 |
|--------|-------|
| `HKEY$` | 程序级热键（程序运行时均有效，默认），不可重用 |
| `HKEY*` | 窗口级热键（仅本窗口活动时有效） |
| `HKEY` | 默认 = 程序级（等同 `HKEY$`） |

**修饰键：**
```wcs
HKEY Ctrl+#0x41,CALL OnHotKeyA                           // Ctrl+A
HKEY Ctrl+Shift+#0x42,CALL OnHotKeyB                     // Ctrl+Shift+B
HKEY Ctrl+Alt+#0x43,CALL OnHotKeyC                       // Ctrl+Alt+C
HKEY Ctrl+Shift+Alt+#0x44,CALL OnHotKeyD                 // Ctrl+Shift+Alt+D

// 虚拟键码可以是数字：
HKEY #0x0D,CALL OnEnter                                  // Enter key (VK_RETURN = 0x0D)
HKEY #0x1B,CALL OnEscape                                 // Escape (VK_ESCAPE = 0x1B)

// Win 键修饰符：
HKEY Win+#0x45,CALL OnWinE                               // Win+E
HKEY Win+Ctrl+#0x46,CALL OnWinCtrlF                      // Win+Ctrl+F
```

**删除：**
```wcs
HKEY Ctrl+#0x41,--del                                     // 删除特定热键
HKEY --del                                               // 删除全部热键
HKEY --del:0x41                                           // 按键码删除
```

---

### 4.19 GROU — 分组框

```wcs
GROU [-center] Name,LxTyWwHh,[Text],[Style]
```

仅为视觉分组框架——不强制互斥（请使用 RADI 组实现互斥）。

```wcs
GROU Grp1,L10T10W200H100,Settings
GROU Grp2,L10T120W200H80,Options
GROU Grp1,L10T10W200H100,,0x0007                          // no text, sunken style
```

---

### 4.20 MEMO — 多行文本框

```wcs
MEMO[-|+.] [-right] [-center] [-vcenter[:缩小量]] [-rich] [*] Name,LxTyWwHh,[内容],[目标文件名],[类型],[颜色],[字体]
```

带内置滚动支持的多行编辑框。

**注意：MEMO 的前缀标志含义与 EDIT 相反！**
- EDIT：`-` = 添加水平滚动条，`|` = 添加垂直滚动条
- MEMO：`-` = 移除水平滚动条，`|` = 移除垂直滚动条

```wcs
MEMO Mem,L10T10W300H200,Initial text here,,0x0030        // VSCROLL + HSCROLL
ENVI @Mem=New multi-line\ntext                             // set with NL for newlines
```

---

### 4.21 MENU — 弹出菜单 / 窗口菜单栏

#### 弹出菜单

```wcs
_SUB MyPopupMenu
    MENU Open,Open File...,CALL OnOpen
    MENU Save,Save,CALL OnSave
    MENU -                                                // 分隔线
    MENU Exit,Exit,KILL \
_END

// 在鼠标位置显示：
ENVI @Ctrl.MSG=_%&::WM_RBUTTONDOWN%: CALL @--popmenu MyPopupMenu

// 在指定坐标显示：
CALL @--popmenu MyPopupMenu 100:200                        // (x:y)
CALL @--popmenu MyPopupMenu 100:200:4                      // 右对齐 (1=左,2=上,4=右,8=下)
```

#### 窗口菜单栏

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

**级联子菜单：**
```wcs
MENU -sub:ParentMenuItem ChildItem,Child Text,Command
MENU -sub:ParentMenuItem -                                 // 子菜单中的分隔线
```

---

### 4.22 TREE — 树形视图控件

```wcs
TREE [-font:... -color:...] [*] [Name],LxTyWwHh,[ImageSource],[Data],[Status]
```

带可展开/折叠节点的层次树。`*` = 自动回收。必须位于 `_SUB` 窗口内。

**状态标志：**

| Flag | 说明 |
|------|-------------|
| `0x1` | HASBUTTONS — 显示 +/- 按钮 |
| `0x2` | HASLINES — 显示节点间连线 |
| `0x4` | LINESATROOT — 连线连接到根节点 |
| `0x8` | EDITLABELS — 用户可编辑节点标签 |
| `0x10` | DISABLEDRAGDROP — 禁止拖放 |
| `0x20` | SHOWSELALWAYS — 始终显示选中项 |
| `0x40` | RTLREADING — 从右到左阅读 |
| `0x80` | NOTOOLTIPS — 禁用工具提示 |
| `0x100` | CHECKBOXES — 每个节点有复选框 |
| `0x400` | SINGLEEXPAND — 单击展开 |
| `0x800` | INFOTIP — 悬停显示信息提示 |
| `0x1000` | FULLROWSELECT — 整行高亮 |
| `0x2000` | NOSCROLL — 无滚动条 |
| `0x4200` | TRACKSELECT — 热追踪 |
| `0x4000` | NONEVENHEIGHT — 可变行高度 |
| `0x8000` | NOHSCROLL — 无水平滚动条 |
| `0x1000000` | 可见 |

**数据格式：** `<iconIdx:selIconIdx>Text`，节点以 `0x09` (TAB) 分隔，子节点起始 `0x0b`，子节点结束 `0x0c`。

```wcs
// 用 SET$ 构建含 0x0B/0x0C 控制字符的节点数据
SET$ &TAB=09
SET$ &CHILDBEGIN=0b
SET$ &CHILDEND=0c
SET &DATA=<0:1>Root%&TAB%<1:2>Child1%&CHILDBEGIN%<2:3>Grandchild%&CHILDEND%<1:2>Child2
TREE Tr1,L10T10W300H200,shell32.dll,%&DATA%,0x100
```

**操作：**
```wcs
// Selection
ENVI @Tr1.Sel=nodeChain[;[*~#]val]    // set selection (*=multi, ~=show, #=focus)
ENVI @Tr1.Sel=?[.][@*]var[;posName]   // query (.=mouse pos, @=handle, *=multi)
ENVI @Tr1.Sel=?.;&&rowVar;&&colVar    // query node at mouse position

// Data
ENVI @Tr1.Val=[>+][node][*[*]$][#];val // set node (>insert, +append, *multi, #trim)
ENVI @Tr1.Val=?*[+$][node];var        // query (+children, $without children)
ENVI @Tr1.Val=?**[+$~[~]-#][node];var // get all node data

// Checkbox
ENVI @Tr1.Check=node;0|1|2             // set/get checkbox (2=toggle)

// State
ENVI @Tr1.Enable=~node;val             // gray state (0=grayed, 1=normal, 2=toggle)

// Expand/Collapse
ENVI @Tr1.Expand=[?]node;val           // 0x0001=collapse, 0x0002=expand, 0x0003=toggle
                                       // 0x4002=expand partial, 0x8000=collapse+reset

// Position
ENVI @Tr1.UPos=?[#]node;L;T;R;B      // query node rectangle

// Handle mapping
ENVI @Tr1.hID=[~]nodeChain|*hID;var   // node chain ↔ tree item handle
```

---

### 4.23 SBAR — 独立滚动条

```wcs
SBAR [-left|-right|-color:barColor:thumbColor:[*]bindTarget] [*] Name,LxTyWwHh,[ValueInfo],[EventCmd],[Status]
```

独立滚动条控件。`*` = 自动回收。必须位于 `_SUB` 窗口内。

**值信息：** `[initialValue][:endValue][:initValue][:pageSize]`，默认 `0:100:0`。

**Status：** 负数=禁用，`0x10`=不可见，`0x40`=水平。

**绑定目标：** `-color:fg:bg:*TargetName` 将滚动条附加到控件以便滚动。

**操作：**
```wcs
ENVI @Sbar.VAL=[cur][:start][:end][:pageSize]  // set value info
ENVI @Sbar.VAL=?[curVar][:startVar][:endVar]   // query
```

---

### 4.24 SCRN — 屏幕信息与捕获

**屏幕信息查询：**
```wcs
SCRN [W,H,X,Y,任务栏位置,DpiX,DpiY,放X,放Y]              // 查询屏幕尺寸等信息
SCRN -win [W,H,...]                                       // 活动窗口尺寸
SCRN -taskbar [W,H,...]                                   // 任务栏区域
SCRN -desk [W,H,...]                                      // 桌面区域
SCRN -display:名 [W,H,...]                                // 指定显示器
```

**屏幕捕获：**
```wcs
SCRN -cap [:格式:][文件名],[#窗口ID|<x:y:R:B>[;源文件]]    // 截图到文件
SCRN -capgui [:格式:][文件名],[#窗口ID|<x:y:R:B>]          // GUI 交互截图
```

```wcs
SCRN ScrW,ScrH                                            // 获取屏幕宽高
SCRN -cap shot.bmp                                        // 捕获全屏
SCRN -cap shot.bmp,#0x12345                               // 捕获指定窗口
SCRN -cap :image/png:shot.png,<0:0:1920:1080>             // 捕获指定区域
```

---

## 5. 消息映射与事件处理

### `ENVI @Name.MSG=` — 消息处理

```wcs
ENVI @ControlName.MSG=_msgId:command                      // 控件通知
ENVI @ControlName.MSG=msgId:command                       // 直接窗口消息
ENVI @WinName.MSG=msgId:command                           // 窗口级消息
ENVI @this.MSG=msgId:command                              // "this" = 当前窗口
```

### 前缀约定

| 前缀 | 含义 | 执行方式 |
|--------|---------|-----------|
| `_` | 控件通知（例如 `_0x004E` = `_WM_NOTIFY`） | **后于**系统处理。对所有 WM_COMMAND/WM_NOTIFY 子类型使用 `_`。 |
| (无) | 直接窗口消息 | 处理函数运行后，消息被**丢弃**（不传递给系统） |
| `$` | 替换系统处理 | 处理函数**代替**默认窗口过程运行 |
| `*` | 捕鼠器 B | 鼠标捕获模式（MouseCapture B） |
| `+` | 超级捕获 | 超级捕获模式 |

```wcs
ENVI @Btn1.MSG=_0x0201: CALL OnLeftClick                   // _ = 控件通知 (WM_LBUTTONDOWN)
ENVI @Win.MSG=0x0010: CALL OnClose                          // WM_CLOSE，处理函数运行后丢弃
ENVI @Win.MSG=$0x0111: CALL OnCommand                        // 替换系统 WM_COMMAND 处理函数
ENVI @Win.MSG=*0x0005: CALL OnResize                         // 捕鼠器 B（鼠标捕获）
ENVI @Btn1.MSG=+0x0201: CALL OnClickDelayed                  // 超级捕获
```

### 投递/发送消息

```wcs
ENVI @Ctrl.POSTMSG=#1                                       // 投递自定义消息 #1
ENVI @Ctrl.POSTMSG=#2;wParam                                // 投递并携带 wParam
ENVI @Ctrl.SENDMSG=#3;wParam;lParam                         // 同步发送
ENVI @@SENDMSG=wid;msg#;wParam;lParam                        // 跨进程发送
ENVI @@POSTMSG=wid;msg#;wParam;lParam                        // 跨进程投递
```

### 常用 Win32 消息 ID

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
SET &::WM_MOUSEENTER=0x1000                 // PECMD 内部消息（非 Win32 标准）
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

### WM_COMMAND 子字段访问

处理 WM_COMMAND (`0x0111`) 时：

```wcs
%&__wParam.wID%              // 控件 ID（发送消息的控件的数字 ID）
%&__wParam.wNotifyCode%      // 通知代码（BN_CLICKED=0, EN_CHANGE=0x300, LBN_SELCHANGE=1 等）
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

**EN_CHANGE 通知模式（输入框实时验证）：**

```wcs
// 方式1：通过 _COMMAND# 组合消息 ID 直接匹配
ENVI &::EN_CHANGE=0x0300
ENVI @Edit1.ID=?;&editID
CALC -base=16 #&::edit1_CHANGE=%&::EN_CHANGE% * 0x10000 + %&editID%
ENVI @Win.MSG=_COMMAND#%&::edit1_CHANGE%: CALL OnEdit1Changed

// 方式2：在 OnWM_COMMAND 内检查 wNotifyCode
_SUB OnWM_COMMAND
    FIND $%&__wParam.wNotifyCode%=0x300,  // EN_CHANGE
    {
        FIND #%&__wParam.wID%=%&editID%, CALL OnEdit1Changed
    }
_END
```

常用通知代码：BN_CLICKED=0, EN_CHANGE=0x300, EN_SETFOCUS=0x100, EN_KILLFOCUS=0x200, LBN_SELCHANGE=1, CBN_SELCHANGE=1。

### WM_NOTIFY 子字段访问

处理 WM_NOTIFY (`0x004E`) 时：

```wcs
%&__NMHDR.idFrom%            // 发送通知的控件 ID
%&__NMHDR.code%              // 通知代码（LVN_ITEMCHANGED=-100, NM_CLICK=-2 等）
%&__NMHDR.hwndFrom%          // 控件的 HWND
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

## 6. 窗口实例化与生命周期

### CALL @ 变体

| 变体 | 行为 |
|---------|----------|
| `CALL @WinName` | **模态** — 阻塞调用方直到窗口关闭 |
| `CALL @*WinName` | **并行** — 调用方与窗口同时运行 |
| `CALL @-WinName` | **后台** — 窗口运行；调用方继续但不进入消息循环 |
| `CALL @~WinName` | **后台非阻塞** — 费时操作也不阻塞 |
| `CALL @~~WinName` | **后台快速** — 同 `@~` 但更快（减少消息循环开销） |
| `CALL @+WinName` | **放弃的子窗口** — 程序可以退出而不等待此窗口 |
| `CALL @^WinName` | **并行且父窗口优先** — 类似 `@*` 但父窗口不阻塞子窗口的消息循环 |
| `CALL @WinName`（调用两次） | 如果窗口存在，将其置于前台 |

### 使用 `this` 的类/实例模式

```wcs
_SUB MyDialog
    ENVI @this.MSG=0x0010: CALL OnClose                       // handle WM_CLOSE
    ENVI @this.POS=?;&wL:&wT:&wW:&wH                          // query own position
    ENVI @this.Visible=0                                       // hide self
    ENVI @this.Visible=1                                       // show self
    ENVI @this.font=12:Microsoft YaHei                         // set window font
_END
```

`%&__WinID%` 包含当前窗口的 HWND。
`%&__LastWinID%` 包含最后创建的窗口的 HWND。

### 窗口销毁

```wcs
CALL @--WinName             // destroy the window environment (closes window)
KILL \                      // close current window (equivalent to X button)
KILL WinName                // close named window
KILL PidOrHwnd              // kill process or window by ID
```

### 窗口枚举

```wcs
FIND --wid*@[parentWID] &list,[titleFilter]                   // enumerate all windows
FIND --wid*@ &list,MyWindow                                   // find windows with "MyWindow" in title
FIND --wid* &list                                             // enumerate all top-level windows
// Output format (TAB-separated): 序号 窗口ID 控件ID 父窗口ID 线程ID 进程ID 类型 标题
```

---

## 7. 完整 GUI 示例

```wcs
#code=65001
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
    RADI Rad1,L20T170W100H20,Small,,1,1                                       // Group 1, selected
    RADI Rad2,L20T190W100H20,Medium,,0,1
    RADI Rad3,L140T170W100H20,Large,,0,1

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

本示例演示了：
- 可调整大小的窗口（`-size`）
- 分组框（GROU）
- 标签（LABE）、编辑框（EDIT）、浏览按钮
- 下拉列表（LIST）含选项
- 复选框（CHEK）含启用/禁用逻辑
- 单选按钮组（RADI）
- 进度条（PBAR）
- 数据表格（TABL）含网格线和多选
- 定时器（TIME）用于周期性更新
- 确定/取消按钮含事件处理器
- 通过 CALL 处理器的完整消息流


---

## 8. GUI 写法示例

有关实用 GUI 模式（动态控件、选项卡页面、自定义标题栏、GDI 绘图、拖放等），请参见 [how-tos/gui.md](how-tos/gui.md)。

---

## 9. 高级技巧

### 9.1 窗口到托盘的生命周期

```wcs
_SUB OnSize                                         // WM_SIZE handler
    IFEX $%&SIZE_MINIMIZED%=%1, OnHide              // minimize → hide window
_END

_SUB OnHide
    ENVI @@Visible=%&WID%:0                         // hide without closing
    TIPS* ,Status message,,,tray_icon.dll#1
_END

_SUB OnSwitch                                       // toggle visibility
    ENVI @@Visible=?%&WID%:&&zzView
    FIND |%&zzView%=0, ENVI @@Visible=%&WID%:1
    ! ENVI @@Visible=%&WID%:0
_END
```

### 9.2 SendMessage：ListView EnsureVisible

滚动 TABL 确保指定行可见：

```wcs
SET &lvm_first=0x1000
CALC #&lvm_ensurevisible=(%&lvm_first% + 19)

_SUB 滚动到行
    ^CALC #&inx=%要选中的行% - 1                       // 0-based index
    SET @TABL.sendmsg=%&lvm_ensurevisible%;%&inx%;0
_END
```

### 9.3 窗口自动定位到屏幕边缘

将窗口定位到屏幕右边缘（用于 WiFi 工具）：

```wcs
SCRN &ScrW,&ScrH                                       // get screen size
CALC #ScrH=%&ScrH% - %&B_TRIM%                       // subtract taskbar height

CALC &WinL=%&ScrW% - %&WinW%                          // right-aligned
CALC &WinT=160
CALC &WinH=%&ScrH% - 5

_SUB Win,L%WinL%T%WinT%W%WinW%H%WinH%,Title
_END
```

### 9.4 任务栏高度检测

```wcs
ENVI &B_TRIM=40                                       // fallback
FIND --wid*@ &&all_win
FORX *NL &&all_win,&&tray_win,
{
    MSTR &&win_type=<7>%&tray_win%
    FIND $%&win_type%=Shell_TrayWnd,
        TEAM MSTR &taskbar_wid=<2>%&tray_win%|
             ENVI @@POS=?%&taskbar_wid%::::&B_TRIM
}

SCRN &ScrW,&ScrH
CALC #ScrH=%&ScrH% - %&B_TRIM%                       // usable screen height
```

### 9.5 网卡枚举

遍历所有有效网卡（跳过虚拟适配器）：

```wcs
_SUB GetNetStatus
    ENVI NIC=0
    ENVI &msg_all=
    ENVI &net_icon=pnidui.dll#0                       // default icon
    LOOP #1=1,
    {
        PCIP ?* IP,MASK,GW,DNS,%NIC%?NAME,MAC,LINK,...,TYPE
        LSTR &&valid_name=1,%NAME%
        FIND $%&valid_name%={,!EXIT LOOP              // NAME starts with { → invalid
        LSTR &&virtual_mac=11,%MAC%
        FIND $%&virtual_mac%=00-50-56-C0,!            // skip VMware adapters
        {
            FIND #%STATUS%=2, ENVI &&msg1=%LINK%\n%IP%
            ! ENVI &&msg1=%LINK%\n未连接
            ENVI &msg_all=%&msg_all%%&msg1%\n\n
        }
        CALC NIC=%NIC%+1
    }
_END
```

### 9.6 磁盘/分区信息

```wcs
// Enumerate all drive letters:
FDRV AllDrive=                                        // available drive letters
FDRV AllDrive=*:                                      // existing volumes

// Check partition type (ESP GUID = C12A7328-F81F-11D2-BA4B-00A0C93EC93B):
PART -phy# LIST drv Z:,&&V
MSTR id=5,36,%&V%                                      // extract GUID (bytes 5-40)
MSTR EFIPF=-4,2,%&V%                                   // extract partition number
FIND $%id%=C12A7328-F81F-11D2-BA4B-00A0C93EC93B,
    MESS This is the ESP partition

// Mount ESP with mountvol:
EXEC!=!mountvol.exe Z: /S                              // mount ESP as Z:
SUBJ -Z:                                                // dismount
```

### 9.7 隐藏任务栏

```wcs
// 隐藏任务栏：
FIND --class:Shell_TrayWnd --wid*@ &任务栏
MSTR* id=<2>%&任务栏%
ENVI @@visible=%id%:0                                   // hide
// ENVI @@visible=%id%:1                                // 显示
```

---

## 10. API 调用集成

直接从 PECMD 脚本调用 Win32 API 以进行深层系统访问。

> **版本兼容性警告：** DLL 调用需要 PECMD2012 v1.88+ 完整版支持。32-bit 和 64-bit 行为完全一致。
> 如果 DLL 调用返回空，使用内置命令替代：`FIND --wid*@`（窗口）、`FIND --pid*@`（进程）、`REGI`（注册表）、`EXEC*`（外部命令）。

### 10.1 DLL 函数调用语法

```wcs
CALL $--qd --ret:&&ReturnVar DLL_Path,FunctionName,[param1],[param2],...
```

**常用标志：**

| 标志 | 用途 |
|------|---------|
| `--qd` | 启用类型前缀系统（qualified mode），允许 `#`/`$`/`*` 等前缀指定参数类型 |
| `--bool` | 函数返回 BOOL 类型 |
| `--ret:var` | 保存返回值到变量 |
| `--cd` | 调用前切换到 DLL 目录 |

**参数类型：**

| 前缀 | 类型 | 示例 |
|--------|------|---------|
| `#N` | 整数 | `#0`、`#%&handle%` |
| `$string` | 宽字符串（UTF-16） | `$一些文字` |
| `@string` | ANSI 字符串 | `@text` |
| `*buffer` | 缓冲区指针 | `*&buf` |

### 10.2 缓冲区操作

```wcs
// Allocate zero-filled buffer
SET$# &buf=*4096 0                        // 4096 bytes of zeros

// Write integer to buffer at offset
SET-long &buf=value:offset                // write DWORD
SET-ptr &buf=value:offset                 // write pointer-sized value

// Read from buffer
SET?int &buf=&&var:offset                 // read DWORD
SET?longlong &buf=&&var:offset            // read QWORD
SET?ptr &buf=&&var:offset                 // read pointer
SET?short &buf=&&var:offset              // read WORD
SET?byte &buf=&&var:offset               // read BYTE

// Extract string from buffer
SET-make &str=&buf@offset;length          // copy null-terminated string

// Memory copy
SET-copy &dest=&src;srcOff;len;destOff    // copy between buffers
```

### 10.3 SCROLLINFO 结构体模式

滚动条控件的常用模式（7 个 DWORD = 28 字节）：

```wcs
// SCROLLINFO: cbSize, fMask, nMin, nMax, nPage, nPos, nTrackPos
// Offsets:       0       4      8     12    16     20       24

SET$# &lpsi=*28 0                         // allocate
SET-long &lpsi=%&sif_all%:4               // fMask at offset 4
SET-long &lpsi=%&nMax%:12                 // nMax at offset 12

CALL $--qd --bool --ret:&&bret User32.dll,SetScrollInfo,
    #%&hwnd%,#%&sb_vert%,*&lpsi,#1

// Read back:
SET?int &lpsi=&&nPos:20                   // nPos at offset 20
```

### 10.4 GetIfTable（网络流量）

两次调用模式：获取所需大小 → 分配 → 获取数据：

```wcs
// 1st call: get size
CALL $--qd --ret:&&ret Iphlpapi.dll,GetIfTable,*0,*&dwSize,#0

// Allocate
SET$# &pIfTable=*%&dwSize% 0

// 2nd call: get data
CALL $--qd --ret:&&ret Iphlpapi.dll,GetIfTable,*&pIfTable,*&dwSize,#1

// Parse MIB_IFROW (each entry = 860 bytes starting at offset 4)
// dwInOctets at offset 552, dwOutOctets at offset 576
SET?int &pIfTable=&&dwIn:(4 + %i% * 860 + 552)
SET?int &pIfTable=&&dwOut:(4 + %i% * 860 + 576)

// Calculate speed (difference per second × 8 = bps)
CALC &&down_bps=(%&dwIn% - %&lastIn%) * 8
```

### 10.5 DeviceIoControl（磁盘控制）

```wcs
// Open physical drive
CALL $--qd --ret:&&h Kernel32.dll,CreateFileW,
    $\\.\PhysicalDrive0,              // device path
    #0xC0000000,                      // GENERIC_READ | GENERIC_WRITE
    #3,                               // FILE_SHARE_READ | FILE_SHARE_WRITE
    #0,                               // no security
    #3,                               // OPEN_EXISTING
    #128,                             // FILE_ATTRIBUTE_NORMAL
    #0                                // no template

// IOCTL_STORAGE_QUERY_PROPERTY
CALL $--qd --ret:&&ret Kernel32.dll,DeviceIoControl,
    #%&h%,                            // handle
    #0x2D1400,                        // IOCTL code
    *&lpInBuffer,                     // input buffer
    #%&inSize%,                       // input size
    *&lpOutBuffer,                    // output buffer
    #%&outSize%,                      // output size
    *&lpBytesReturned,                // bytes returned
    #0                                // no overlapped

CALL $--qd --bool Kernel32.dll,CloseHandle,#%&h%
```

### 10.6 SetupAPI（设备枚举）

```wcs
// GUID for disk drives: {53f56307-b6bf-11d0-94af-0000c09ef10b}
// Layout (little-endian): [4B Data1][2B Data2][2B Data3][8B Data4]
SET$# &guid=*16 0
SET-long &guid=0x53F56307:0       // Data1 — bytes 0-3
SET-long &guid=0x11D0B6BF:4       // Data2(2B) + Data3(2B) as one DWORD — bytes 4-7
SET-long &guid=0x0000AF94:8       // Data4[0..3]: 0x94 0xAF 0x00 0x00 — bytes 8-11
SET-long &guid=0x0BF19EC0:12      // Data4[4..7]: 0xC0 0x9E 0xF1 0x0B — bytes 12-15

// Get device list
CALL $--qd --ret:&&h Setupapi.dll,SetupDiGetClassDevsW,
    *&guid,#0,#0,#18                 // DIGCF_PRESENT | DIGCF_DEVICEINTERFACE

// Enumerate
ENVI &&i=0
LOOP #1=1,
{
    CALL $--qd --bool --ret:&&ret Setupapi.dll,SetupDiEnumDeviceInterfaces,
        #%&h%,#0,*&guid,#%&i%,*&data
    FIND $%&ret%=0, EXIT LOOP

    // Get detail
    CALL $--qd --ret:&&ret Setupapi.dll,SetupDiGetDeviceInterfaceDetailW,
        #%&h%,*&data,#0,#0,*&size,#0
    // ... allocate and call again ...
    CALC &&i=%&i%+1
}

CALL $--qd --bool Setupapi.dll,SetupDiDestroyDeviceInfoList,#%&h%
```

### 10.7 EnumResourceNames 回调模式

```wcs
// Bind callback function
SET^ OnEnumProc,&&callbackAddr

// Load DLL and enumerate
CALL $--qd --ret:&&hMod Kernel32.dll,LoadLibraryExW,
    $%file%,#0,#0x22                // LOAD_LIBRARY_AS_DATAFILE

CALL $--qd --bool Kernel32.dll,EnumResourceNamesW,
    #%&hMod%,#14,                    // RT_GROUP_ICON = 14
    #%&callbackAddr%,#0

CALL $--qd --bool Kernel32.dll,FreeLibrary,#%&hMod%

// Unbind
SET^ OnEnumProc=0

// Callback function (receives: hModule, lpszType, lpszName, lParam)
_SUB OnEnumProc
    // Extract resource ID
    ^CALC #&&id=%~3
    // Return 1 to continue enumeration
    EXIT _SUB 1
_END
```

### 10.8 COM / GUID 操作

```wcs
// Create GUID (16-byte buffer)
SET$# &guid=*16 0
CALL $--qd --ret:&&ret Ole32.dll,CoCreateGuid,*&guid

// GUID → string (allocated by COM, must free with CoTaskMemFree)
SET$# &lplpsz=*8 0
CALL $--qd --ret:&&ret Ole32.dll,StringFromCLSID,*&guid,*&lplpsz
// pString now at address stored in lplpsz; CoTaskMemFree to clean up

// String → GUID
SET$# &pclsid=*16 0
CALL $--qd --ret:&&ret Ole32.dll,CLSIDFromString,
    ${GUID-string},*&pclsid

// Free COM-allocated memory
CALL $--qd Ole32.dll,CoTaskMemFree,*&lplpsz

// DPI query (using desktop DC)
CALL $--qd --ret:&&hdc User32.dll,GetDC,#0          // desktop HWND
CALL $--ret:&&LogPx Gdi32.dll,GetDeviceCaps,#%&hdc%,#88  // LOGPIXELSX
CALL $--qd User32.dll,ReleaseDC,#0,#%&hdc%
CALC &DPI=%&LogPx% / 96
```


---

## 11. 系统与磁盘操作

有关磁盘/分区/文件/注册表操作，请参见 [commands-full.md](commands-full.md)。

有关代码示例，请参见 [how-tos/storage.md](how-tos/storage.md) 和 [how-tos/system.md](how-tos/system.md)。

---

## 12. 补充技巧

### 12.1 执行锁模式

防止操作进行中重入：

```wcs
_SUB StartOperation
    // Lock
    ENVI &&RUNNING=1
    ENVI @BtnExec.Enable=0
    ENVI @Tabs1.Enable=0
    TIME Timer1,500,CALL UpdateProgress

    // ... long operation ...

    // Unlock
    ENVI @Timer1=0
    ENVI @BtnExec.Enable=1
    ENVI @Tabs1.Enable=1
    ENVI &&RUNNING=0
_END
```

### 12.2 THREAD* 后台工作

```wcs
_SUB MainWin,W300H200,Title
    // Register completion handler
    ENVI @this.MSG=#1: CALL OnTaskDone
    // Launch background thread
    THREAD* CALL LongTask
_END

// Long-running task in background
_SUB LongTask
    // ... heavy work ...
    // Notify main window when done
    ENVI @MainWin.POSTMSG=#1
_END

_SUB OnTaskDone
    MESS Task complete!
_END
```

### 12.3 ENVI-ret 返回值模式

将变量名作为参数传递，并通过 `ENVI-ret` 设置：

```wcs
// Calling code:
CALL GetDriveInfo &&result

// Called function:
_SUB GetDriveInfo
    // ... compute ...
    ENVI-ret %~1=最终结果值
_END
```

### 12.4 SET^ 回调绑定

将 `_SUB` 函数绑定为 Win32 回调（用于 EnumResourceNames、EnumWindows 等）：

```wcs
// 1. Bind: creates a callable address from a _SUB
SET^ OnMyCallback,&&addr

// 2. Use in API call
CALL $--qd --bool Kernel32.dll,SomeEnumFunction,
    #%&h%,#%&addr%,#0

// 3. Unbind when done
SET^ OnMyCallback=0

// 4. The callback function:
_SUB OnMyCallback
    // Parameters from Win32:
    //   %1 = first arg, %2 = second arg, etc.
    // Return non-zero to continue, 0 to stop
    ENVI @@RET=1                         // set return value
_END
```

### 12.5 滚动条像素控制（SendMessage）

```wcs
CALC &&LVM_SCROLL=0x1000 + 20
CALC &&LVM_ENSUREVISIBLE=0x1000 + 19

// Scroll horizontally to pixel position
SET @@sendmsg=%&hwnd%;%&LVM_SCROLL%;%pixels%;0

// Ensure row is visible
CALC #&&idx=%targetRow% - 1              // 0-based index
SET @@sendmsg=%&hwnd%;%&LVM_ENSUREVISIBLE%;%&idx%;0
```

### 12.6 DPI 感知窗口

```wcs
_SUB MainWin,L0T0W500H400,Title,,,,#,-ntab
    // Set DPI awareness
    CALL $--qd --bool --ret:&&r User32.dll,
        SetProcessDpiAwarenessContext,#-4   // DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE

    // Query DPI
    CALL $--ret:&&hdc User32.dll,GetDC,#%&__WinID%
    CALL $--ret:&&dpi Gdi32.dll,GetDeviceCaps,#%&hdc%,#88
    CALC &&scale=%&dpi% / 96

    // Scale coordinates
    CALC &&w=500 * %&scale%
    CALC &&h=400 * %&scale%
    ENVI @this.POS=:0::%&w%:%&h%          // resize window

    // Set DPI scaled font
    ENVI @this.Font=%&dpi%:Microsoft YaHei
_END
```

### 12.7 定时器倒计时 + 自动恢复

来自分辨率调节工具：

```wcs
TEAM ENVI &&COUNT=0|ENVI &&SECONDS=20

_SUB MainWin,W340H270,分辨率调节,,shell32.dll#43,,
    PBAR PBar1,L18T15W100H13,1
    LABE LabelK,L130T15W180H12,20秒后恢复

    // ... resolution radio buttons ...

    ITEM Button1,L30T150W125H40,应用更改,CALL ApplyRes
    TIME TimerFill,200,CALL OnFill           // progress bar fill
    TIME TimerCount,1000,CALL OnCount        // countdown
_END

_SUB OnFill
    CALC &&COUNT=%&COUNT%+1
    ENVI @PBar1=%&COUNT%
_END

_SUB OnCount
    CALC &&SECONDS=%&SECONDS% - 1
    ENVI @LabelK=%&SECONDS%秒后恢复
    IFEX $%&SECONDS%<1,
    {
        DISP                                    // auto-revert
        ENVI @TimerCount=0
    }
_END
```

---

## 附录 A：控件样式标志速查表

| 控件 | 样式（十六进制） | 含义 | 来源 |
|---------|-------------|---------|--------|
| **EDIT** | `0x0800` | 只读 (ES_READONLY) | DISMGUI |
| **EDIT** | `0x18` | 只读 + 边框 | 打开方式 |
| **EDIT** | `0x224` | 密码 + 凹陷边框 | WiFi_NEW |
| **LIST** | `0x10` | 下拉（不可编辑） | 打开方式 |
| **SWIN** | `0x100` | 自动垂直滚动条 | 打开方式 |
| **TABL** | `0x416280` | 网格 + 整行选择 + 排序表头 + 多选 | 打开方式-TABL |
| **TABL** | `0x10010` | 图标 + 单选 | WiFi_NEW |
| **TABL** | `0x820` | 网格线 + 整行选择（实测组合值） | demo |
| **TABL** | `0x2000` | 无标题行 | help.txt |
| **TABL** | `0x10000` | 单行选择（只能选一行） | help.txt |
| **LABE** | (默认) | 可点击（接收命令） | 打开方式 |
| **LABE** | `-ncmd` | 静态，不可点击 | 打开方式 |

---

## 附录 D：文件头标准模板

```wcs
#code=65001                                         // UTF-8 encoding
ENVI^ EnviMode=1                                    // modern mode
ENVI^ ForceLocal=1                                   // forced local vars
SET$ &NL=0d 0a                                       // newline
SET$ &TAB=09                                         // tab

// ── Window Message Constants ──
SET &::WM_LBUTTONDOWN=0x0201
SET &::WM_RBUTTONDOWN=0x0204
SET &::WM_LBUTTONUP=0x0202
SET &::WM_LBUTTONDBLCLK=0x0203
SET &::WM_MOUSEMOVE=0x0200
SET &::WM_MOUSEHOVER=0x02A1
SET &::WM_MOUSELEAVE=0x02A3
SET &::WM_SIZE=0x0005
SET &::WM_CLOSE=0x0010
SET &::WM_DROPFILES=0x0233
SET &::WM_TRAYNOTIFY=1109

// ── Global Flags ──
SET &::FLAG_RUNNING=0

// ── Entry Point ──
CALL @MainWindow
EXIT FILE

// ══════════════════════════════════════════
//  Window Definitions
// ══════════════════════════════════════════

_SUB MainWindow,W500H400,Title
    // controls ...
_END

// ══════════════════════════════════════════
//  Event Handlers
// ══════════════════════════════════════════

// ══════════════════════════════════════════
//  Helper Functions
// ══════════════════════════════════════════
```

---

## 附录 E：最佳实践总结

### 变量管理

| 作用域 | 模式 | 用法 |
|-------|---------|-------|
| 全局常量 | `SET &::NAME=value` | 跨线程、跨文件，永不消失 |
| 层叠引用 | `SET &var=value` | 就近解析：从内向外找，碰到谁就是谁 |
| 局部（推荐） | `SET &&var=value` 或 `ENVI &&var=value` | 函数局部，自动清理 |
| 参数 | `%~1`、`%~2` | 使用 `%~N` 安全访问 |
| 返回值 | `ENVI-ret %~N=%value%` | 调用方：`CALL func &&result` |
| 线程通信 | `&::` 前缀 | THREAD* 协调用的全局变量 |

> **`&var` 解析顺序**（反向匹配）：先找当前范围的 `&&var`，再往外找全局 `&::var`，都没找到就新建为 `&&var`。

### 安全与错误处理

- **管理员检查**：`SET ?adminMODE=isadmin` → `IFEX $%adminMODE%<>1, MESS ... | EXIT`
- **破坏性操作前确认**：`MESS 确定要格式化？ #YN $N` + `FIND $%YESNO%=NO, EXIT`
- **磁盘号验证**：`CALC -err=-1 #disk=(%n%)+0` — 若为负数则无效
- **锁定运行标志**：使用 `ENVI @Btn.Enable=0` + 标志变量防止双重执行
- **临时文件清理**：操作后始终清理临时文件

### UI 性能

- **批量表格数据**：使用 `ENVI @Tbl.Val=1*;%data%` 而非逐行 `ADD`
- **后台加载**：`THREAD* CALL LoadData` + 完成后 `POSTMSG`
- **销毁动态控件**：重建前先 `ENVI @Ctrl.*del=` 每个旧实例
- **定时器清理**：不再需要时 `ENVI @TimerName=0`

### 布局约定（来自实际代码）

- 分组框：x = 边距(9)，y = 16，宽度 = `(总宽度 - 2*边距)/N` - 间隙
- 分组内标签：x = 分组_x + 12，y = 分组_y + 27
- 编辑框：x = 标签_x + 70，与标签同 y
- 浏览按钮：分组内右对齐（分组_x + 分组_宽 - 60）
- 底部按钮：右对齐，y = 窗口高度 - 40

---

## 附录 F：ENVI @ 操作速查表

| 操作 | 用途 | 示例 |
|-----------|---------|---------|
| `ENVI @Ctrl=Text` | 设置文本 | `ENVI @Label1=Hello` |
| `ENVI @Ctrl.Enable=0` | 禁用 | `ENVI @Btn1.Enable=0` |
| `ENVI @Ctrl.Visible=0` | 隐藏 | `ENVI @Panel1.Visible=0` |
| `ENVI @Ctrl.Visible=1` | 显示 | `ENVI @Panel1.Visible=1` |
| `ENVI @Ctrl.POS=L:T:W:H` | 移动/调整大小 | `ENVI @Btn1.POS=10:20:80:28` |
| `ENVI @Ctrl.POS=?L:T:W:H` | 查询位置 | `ENVI @this.POS=?;&l;&t;&w;&h` |
| `ENVI @Ctrl.bkcolor=0xRRGGBB` | 背景色 | `ENVI @Lbl1.bkcolor=0xf0f0f0` |
| `ENVI @Ctrl.Font=12:微软雅黑` | 字体 | `ENVI @this.Font=10:Tahoma` |
| `ENVI @Ctrl.Check=1` | 复选框选中 | `ENVI @Chk1.Check=1` |
| `ENVI @Ctrl.Check=?&v` | 查询复选框 | `ENVI @Chk1.Check=?;&st` |
| `ENVI @Ctrl.Val=1*;%data%` | 批量设置表格 | `ENVI @Tbl.Val=1*;%&rows%` |
| `ENVI @Ctrl.Val=?*;&n` | 获取行数 | `ENVI @Tbl.Val=?*;&cnt` |
| `ENVI @Ctrl.Sel=%n%` | 选择行 | `ENVI @Tbl.Sel=3` |
| `ENVI @Ctrl.Sel=?&r` | 获取选择 | `ENVI @Tbl.Sel=?;&row` |
| `ENVI @Ctrl.ADD=Item` | 添加到列表 | `ENVI @List1.ADD=New Item` |
| `ENVI @Ctrl.DEL=text` | 按文本删除 | `ENVI @List1.DEL=Item Text` |
| `ENVI @Ctrl.*del=` | 销毁控件 | `ENVI @Labe5A.*del=` |
| `ENVI @Ctrl.MSG=msg:Cmd` | 消息映射 | `ENVI @Btn1.MSG=_0x0201:CALL Fn` |
| `ENVI @Ctrl.POSTMSG=#N` | 投递消息 | `ENVI @Win.POSTMSG=#1` |
| `ENVI @Ctrl.SENDMSG=#N;w;l` | 发送消息 | `ENVI @Tbl.SENDMSG=#0x1000+19;%i%;0` |
| `ENVI @Ctrl.Style=+0x1000` | 添加样式 | `ENVI @Ed1.Style=+0x0800` |
| `ENVI @Ctrl.Style=-0x1000` | 移除样式 | `ENVI @Ed1.Style=-0x0800` |
| `ENVI @this.HitTest=31` | 拖动窗口 | `ENVI @this.HitTest=31` |
| `ENVI @Ctrl.id=?&var` | 获取 HWND | `ENVI @Panel1.id=?&hwnd` |
| `ENVI @Ctrl.InvalidateRect=` | 强制重绘 | `ENVI @Tbl.InvalidateRect=` |
| `ENVI @@Visible=WID:0` | 跨进程隐藏 | `ENVI @@Visible=%&wid%:0` |
| `ENVI @@POS=WID:L:T:W:H` | 跨进程移动 | `ENVI @@POS=%&wid%:0:0:300:200` |
| `ENVI @@style=WID:*remove:add` | 跨进程样式 | `ENVI @@style=%&wid%:*:0x00800000` |
| `SET @@sendmsg=WID;msg;w;l` | 跨进程消息 | `SET @@sendmsg=%&hwnd%;0x0111;%id%;0` |
