---
name: pecmd-pro-max
version: 1.2.3
description: |
  PECMD2012 WinPE 脚本编程 — 轻量级 Windows GUI、系统工具、启动/初始化
  脚本、自动化。适用于 .wcs/.wci/.wce 文件、磁盘分区、批处理转 PECMD、
  系统信息收集、PE/预安装环境工具、PECMD 代码调试。
  参考文件：references/commands-full.md、references/pecmd-gui.md、references/pe-startup.md、
  references/how-tos/storage.md、references/how-tos/system.md、references/how-tos/gui.md、references/how-tos/net.md。
compatibility: 需要 PECMD2012 v1.88+ 解释器。脚本运行于 PECMD/WinCMD 环境，非 cmd.exe。
---

# PECMD Pro Max

你是 PECMD 专家程序员。PECMD 是从 XCMD V2.2 演进而来的 WinPE 命令解释器与脚本语言——可将其视为 Windows PE 系统管理和轻量级 GUI 工具的领域特定语言。

## $1 心智模型

PECMD 有两种执行模式：
- **命令行模式**：`PECMD.EXE ENVI $PPPoE=OK` — 单条命令，注释默认关闭
- **脚本模式**：`PECMD.EXE LOAD C:\PECMD.INI` — 多命令文件，注释默认开启

PECMD 脚本是一组平铺的顶层语句。从上到下顺序执行。`_SUB` 块在解析时声明，仅在 CALL 时运行。所有 GUI 由定义窗口函数的 `_SUB` 块创建（通过 `CALL @窗口名` 调用）。

关键运行时事实：
- `PECMD.EXE MAIN 路径\PECMD.INI` — 标准 WinPE 入口，执行 INI 并启动消息循环
- 可在 PECMD 内运行 PECMD：`EXEC =!"%MyNAME%" <命令>` — 用于隔离的子操作
- `EXIT FILE` 终止整个脚本；`EXIT _SUB` 从当前函数返回
- 循环体内 `EXIT -` 继续下一次迭代；`EXIT LOOP` / `EXIT FORX` 跳出循环
- 脚本文件常用 `.wcs` 扩展名。中文脚本首行加 `#code=65001` 声明 UTF-8 编码。若首行以 `#!` 开头，编码指令放在第二行。

## $2 变量系统

PECMD 有**三层**变量体系。搞错这一点是绝大多数错误的根源。

| 层级 | 语法 | 设置方式 | 作用域 |
|------|------|----------|--------|
| 环境变量 | `%var%` | `ENVI var=val` 或 `ENVI $var=val` | 进程级，与子进程共享 |
| PE 变量（局部） | `%&var%` | `ENVI &var=val` 或 `SET var=val` | 当前 `_SUB` 或 `{}` 块 |
| PE 类/全局变量 | `%&::var%` | `ENVI &::var=val` 或 `SET &::var=val` | 跨函数、跨线程、文件级 |

**关键规则：**
1. `ENVI^ ForceLocal=1` 放在文件顶部——强制 `ENVI` 和 `SET` 默认创建局部 PE 变量。**务必始终使用**。
2. `ENVI^ EnviMode=1` — 空变量引用返回空字符串而非报错。务必始终使用。
3. `SET` **始终**等价于 `ENVI &`（SET 总是创建 PE 变量，不受 ForceLocal 影响）。
4. `%Desktop%` 是**环境变量**版本；`%&Desktop%` 是**PE 变量**版本。
5. 多线程代码中务必使用 PE 变量（`&var`）——环境变量跨线程共享会竞态。跨线程通信用 `&::` 类变量。
6. `SET~ &&dest=Source.Key` — `~` 运算符执行**间接解引用**：将右侧展开为变量名，再读取该变量的值。伪数组必备。
7. `^SET` / `^ENVI`（带 `^` 前缀）— 将变量展开推迟到执行时。循环中动态变量名必备。
8. `ENVI-ret %~1=%var%` — 将值设置到*名称*存储在 `%~1` 中的变量（引用返回）。
9. `SET-def var=value` — 仅在变量未定义时设置（安全默认值）。

### 标准文件头

```wcs
#code=65001
ENVI^ EnviMode=1
ENVI^ ForceLocal=1
SET$ &NL=0d 0a
SET$ &TAB=09
```

### 十六进制数据与原始内存

```wcs
SET$ &NL=0d 0a                  // 十六进制转宽字符串（Unicode）
SET$# &buf=*4096 0              // 十六进制转原始字节（二进制缓冲区）
ENVI$ &data=*1M 30 0d 0a        // 可变长度十六进制分配
```

完整 ENVI^ 控制命令、二进制缓冲区操作（SET-cmp、SET-tom、SET-copy、SET?int 等）和函数参数引用，参见 [commands-full.md](references/commands-full.md)。

## $3 代码组织

### 函数与 CALL 变体

```wcs
_SUB 函数名 [*]                   // * = this-call（在调用者栈中运行）
    SET &param1=%~1                 // %~1 剥离外层引号
    ENVI-ret %~3=%&result%          // 通过引用参数返回值
_END
```

```
CALL 函数名 [参数]                 // 调用函数
CALL *函数名 [参数]                // this-call（调用者栈）
CALL @窗口名 [参数]                // 创建/显示窗口（模态，阻塞）
CALL @*窗口名 [参数]               // 并行窗口
CALL @-窗口名 [参数]               // 后台窗口
CALL @~窗口名 [参数]               // 后台，完全非阻塞
CALL @^窗口名 [参数]               // 并行，父窗口不阻塞子窗口
CALL @+窗口名 [参数]               // 弃养子窗口
CALL @--窗口名                     // 销毁 Win 环境
CALL @--popmenu 窗口名 [x.y]       // 在指定位置弹出菜单
```

### 窗口（GUI）

```wcs
_SUB 窗口名,L200T100W400H300,窗口标题,[关闭命令],[图标],[样式],[遮罩],[标志]
_END
```

窗口形状：`L<左>T<上>W<宽>H<高>`。省略 L/T 则居中。
常用标志：`-trap`（关闭按钮不退出）、`-nocap`（无标题栏）、`-nosysmenu`、`-top`、`-size`、`-maxb`、`-minb`、`-disminb`、`-discloseb`、`-nfocus`、`-ntab`、`-forcenomin`、`-nofix`、`-nb`、`-scalef`、`-scale[:DPI]`、`-nxp`、`-csize`、`-na`

### IMPORT 与代码块

```wcs
IMPORT 路径\库文件.wcs          // 将可复用函数加载到当前脚本
_ENDFILE-IMPORT                // 此行以下内容在 IMPORT 时被丢弃

{                               // 带独立 PE 变量栈的代码块
    SET temp=仅此处有效
}
```

文件级和函数级的 `{` 必须从第 1 列开始。嵌套 `_SUB` 用点号访问：`类名.子函数名`。

## $4 流程控制

### FIND — 字符串比较（默认区分大小写）

```wcs
FIND $%var%=hello, 命令               // 相等
FIND $%var%<>hello, 命令              // 不等
FIND $=%var%, 命令                    // "为空" 测试
FIND *=var, 命令                      // 惯用法："为空"
FIND *<>var, 命令                     // 惯用法："非空"
FIND |%a%>%b%, 命令                   // | 前缀 = 数值比较
FIND [ $A & $B ], 命令                // 复合 AND
FIND [| $A | $B ], 命令               // 复合 OR
FIND --pid &var,                     // 获取进程/CPU 信息
FIND --pid*@ &var,                   // 进程列表（用于 TABL）
FIND --wid*@ &var,标题过滤           // 窗口列表
```

### IFEX — 文件测试 / 数值比较

```wcs
IFEX C:\boot.ini, 命令                // 文件/目录存在
IFEX C:\boot.ini,! 命令               // 不存在
IFEX x:\, 命令                        // 盘符存在且有文件系统
IFEX $%val%>=5, 命令                  // $=数值比较
IFEX [ 条件1 & 条件2 ], 命令          // AND 复合条件
IFEX [| 条件1 | 条件2 ], 命令         // OR 复合条件
IFEX MEMU=?,&var                     // 查询可用内存
IFEX C:\=?,&可用空间                  // 查询磁盘可用空间
```

### LOOP / FORX / TEAM / LOCK

```wcs
LOOP #%I%<=10, { MESS %I% | CALC &I=%I% + 1 }

FORX * %&list%,&&item, { MESS %&item% }              // * = 空格分隔
FORX *NL &多行变量,&&line, { ... }                    // *NL = 换行分隔
FORX /S C:\Windows\*.exe,&&file,0 { ... }            // 文件系统枚举
FORX @\Windows,&&winDir,1 { ... }                    // 搜索所有盘符

TEAM SET &a=1| SET &b=2| CALC &c=%&a% + %&b%         // 多命令链

{ LOCK #pecmd                                      // 原子作用域
    LOCK --exist #MyLock,&&ret                      // 检查锁是否存在
}
LOCK #MyLock,&&ret2                                 // 创建/获取命名锁
```

完整 EXIT 变体、LAMBDA 语法和 `FIND --class:` 参见 [commands-full.md](references/commands-full.md)。

## $5 GUI 编程

### 控件类型

| 控件 | 命令 | 控件 | 命令 |
|------|------|------|------|
| 按钮 | `ITEM` | 标签 | `LABE` |
| 编辑框 | `EDIT` | 复选框 | `CHEK` |
| 单选框 | `RADI` | 下拉列表 | `LIST` |
| 表格 | `TABL` | 进度条 | `PBAR` |
| 分组框 | `GROU` | 图片 | `IMAG` |
| 多行文本 | `MEMO` | 子窗口 | `SWIN` |
| 定时器 | `TIME` | 托盘图标 | `TIPS*` |
| 选项卡 | `TABS` | 滑块 | `SLID` |
| 微调器 | `SPIN` | 树形视图 | `TREE` |

### 消息映射

```wcs
ENVI @控件.MSG=_%&WM_LBUTTONDOWN%: 命令        // _ = 后置系统处理器（控件通知）
ENVI @窗口.MSG=0x0010: CALL OnClose             // 无 _ = 窗口级消息（WM_CLOSE）
ENVI @控件.POSTMSG=#1                            // 投递自定义消息 #1
ENVI @控件.SENDMSG=消息号;wParam;lParam          // 同步发送
```

MSG 上的 `_` 前缀至关重要：控件通知（WM_COMMAND/WM_NOTIFY 子类型）用 `_`，窗口直接消息省略 `_`。

### 控件操作

```wcs
ENVI @控件名=新文本                              // 设置文本
ENVI @控件名.Enable=0                            // 禁用（1=启用）
ENVI @控件名.Visible=0                           // 隐藏（1=显示）
ENVI @控件名.POS=左:上:宽:高                     // 移动/调整大小
ENVI @控件名.POS=?;&L:&T:&W:&H                  // 查询位置
ENVI @控件名.Val=?行.列;&var                     // 获取 TABL 单元格
ENVI @控件名.Val=?*;&count                       // 获取 TABL 行数
ENVI @控件名.*del=                               // 销毁控件
```

完整控件语法、全部 22 种控件类型、ENVI @ 属性参考、窗口生命周期和消息映射——参见 [pecmd-gui.md](references/pecmd-gui.md)。

GUI 写法示例（动态控件、选项卡页、自定义标题栏、GDI 绘图、拖放等）——参见 [how-tos/gui.md](references/how-tos/gui.md)。

## $6 DLL 调用

```wcs
CALL $--qd --ret:&&ret DLL路径,函数名,[#]参数1,[#]参数2,...
CALL $--qd --bool --ret:&&ret DLL,函数,...        // --bool：函数返回 BOOL
CALL $--qd --cd --ret:&&ret DLL,函数,...          // --cd：先切到 DLL 所在目录
CALL $--qd --ret:&&ret ,-LoadLibrary,[^]DLL路径   // 加载 DLL（^ = 自动释放）
CALL $--qd --ret:&&ret ,-FreeLibrary,*hDll        // 释放已加载的 DLL
CALL $--qd --ret:&&ret ,-GetProcAddress,*hDll,函数名
```

DLL 参数：`#N`=整数，`$s`=宽字符串，`*buf`=缓冲区指针，`=s`=原始字符串。
按参数类型覆盖：`--qd#`（全整数）、`--qd*`（全 PE 变量）、`--qd$`（全字符串）。
平台检测：`IFEX #%&::bX64%=3, SET &PtrSz=8! SET &PtrSz=4`。

完整 DLL 调用参考、内存缓冲区操作（SET-long、SET?int、ENVI-addr 等）和 CALL $ 扩展类型系统，参见 [commands-full.md](references/commands-full.md)。

## $7 常用范式

### 磁盘与分区

```wcs
FDRV &盘符列表=*:                                        // 枚举所有盘符
FDRV *vol &卷标,&文件系统=C:                             // 获取卷标和文件系统
PART list disk,&&全部磁盘                                 // 枚举所有物理磁盘
PART list disk %&dsk%,&&磁盘信息                          // 获取磁盘信息
PART list part %&dsk%,&&分区列表                          // 列出分区号
PART -hextp -phy# list part %&dsk%#%&pt%,&&分区信息       // 详细分区信息
SHOW * %&dsk%#%&pt%,%&盘符%                               // 分配盘符
SHOW *- %&dsk%#%&pt%,                                     // 移除盘符
```

### 注册表与文件 I/O

```wcs
REGI $HKLM\SOFTWARE\App\Key,&var                  // 读 REG_SZ
REGI #HKLM\SOFTWARE\App\Count,&var                // 读 REG_DWORD
REGI $HKLM\SOFTWARE\App\Key=值                     // 写 REG_SZ
READ %路径%,**,&内容                               // 读取整个文件
WRIT %路径%,$0,第一行                              // 写入文件
GETF# %路径%,0#*,&原始数据                         // 按原始字节读取文件
```

### 执行与捕获

```wcs
EXEC* &输出=!cmd.exe /c dir /b                            // 捕获全部输出
EXEC =!"%MyNAME%" TEAM WAIT 1000|LOAD other.ini           // 运行子 PECMD
EXEC* &out=*IPCONFIG                                      // PECMD 内部命令
EXEC* -exe:#101 &out=*embedded.exe                        // 从 PECMD 资源运行 EXE
```

### 单实例互斥体

```wcs
{ LOCK #pecmd
    LOCK --exist #MyAppLock,&&exists
}
IFEX $1=%&exists%,
{
    REGI $HKCU\Software\MyApp\WID,&&wid
    IFEX $%&wid%>0, TEAM ENVI @@Visable=%&wid%:2| ENVI @@POS=%&wid%:::::::1
    EXIT FILE
}
LOCK #MyAppLock,&ret2
```

### 字符串操作

```wcs
MSTR &&a,&&b=<1><~3>%&data%                              // 字段1、字段3到尾
MSTR &&last=<-1>%&data%                                   // 最后字段
MSTR -delims:. &&a,&&b,&&c,&&d=<1*>%&ip%                 // 按自定义分隔符拆分
SED &&r=0,模式,替换,%&source%                             // 替换所有匹配
LPOS &&pos=needle,,%&haystack%                            // 查找首次出现（不区分大小写）
```

含错误处理和边界情况处理的完整展开版本，参见 [how-tos/storage.md](references/how-tos/storage.md)（磁盘/注册表/文件）、[how-tos/system.md](references/how-tos/system.md)（进程/线程/定时器）、[how-tos/net.md](references/how-tos/net.md)（网络/COM）。

## $8 陷阱与注意事项

1. **CALC 空格**：`CALC &J=1+2` 可以正常执行。右侧以 `%&I%` 等变量开头时，用空格分隔（`CALC &J= %&I%+1`）。PECMD 要求减号后必须有空格（`3 - 2`）。
2. **注释标记**：`//` 和 `;` 在行尾时必须前面有空格。行首的 `//comment` 可能不被识别。
3. **SET 就是 ENVI &**：`SET var=val` 语义上等价于 `ENVI &var=val`。启用 ForceLocal=1 后，两者都创建局部 PE 变量。
4. **FIND 与 IFEX**：`FIND $` = 字符串比较。`FIND |` = 数值比较。`IFEX $` = 数值比较。明确使用前缀，不要想当然。
5. **盘符冒号**：`FDRV`、`FORM`、`FIND C:\=?` 都需要 `:` 后缀。
6. **带空格的路径**：`LOAD "C:\Program Files\a.ini"` 需要引号。
7. **文件编码**：中文脚本需要首行 `#code=65001` 且文件必须保存为 UTF-8。
8. **`{` 位置**：文件级和函数级 `{` 必须从第 1 列开始。在 TEAM/LOOP/IFEX 内部，`{` 启动命令组。
9. **行续接**：行首第一个非空格字符为 `\` 时，将该行合并到上一行。
10. **`_SUB` 不能内联**：不能在 FIND/IFEX/TEAM 命令内部定义 `_SUB`。
11. **空字符串检测**：`FIND $%var%=,` 测试"为空"。`FIND *=var,` 测试"非空"。
12. **字面 %**：字符串中表示字面 `%` 用 `%%`。
13. **线程安全**：线程接收父级 PE 变量的**副本**。真正跨线程共享用 `&::` 变量。
14. **中文变量名**：PECMD 社区的事实标准。为这个生态系统编写脚本时使用中文名称。
15. **OnShutdown.wcs**：PECMD 在系统关机/重启/注销前自动运行 `%SystemRoot%\System32\OnShutdown.wcs`。
16. **`^` 预解释**：`^COMMAND` 将变量展开推迟到执行时（循环中必备）。`^^COMMAND` 预解释两次。
17. **MSG 上的 `_` 前缀**：控件通知用 `_msg#`；窗口级消息省略 `_`。搞错这一点是非常常见的错误。
18. **THREAD\* 与 THREAD**：只有持久（窗口）栈中的 THREAD* 共享 PE 变量。在 `{}` 块中，两者都复制。
19. **FIND 展开规则**：FIND 中的裸标识符被视为字面字符串。始终使用 `FIND $%&var%=值` 引用 PE 变量。
20. **`@@Visable` 与 `@Visible`**：跨进程用 `ENVI @@Visable=窗口ID:值`。进程内用 `ENVI @控件.Visible=0|1`。

## $9 参考文件

当需要超越上述快速参考的详细信息时，查阅以下权威文件：

| 文件 | 何时查阅 |
|------|----------|
| `references/commands-full.md` | 命令语法、全部标志、参数细节 |
| `references/pecmd-gui.md` | GUI 控件、消息、窗口生命周期、ENVI @ 属性 |
| `references/pe-startup.md` | WinPE 启动脚本、PECMD.INI 结构、启动阶段 |
| `references/how-tos/storage.md` | 磁盘、分区、文件、注册表、设备 写法示例 |
| `references/how-tos/system.md` | 进程、线程、定时器、系统信息、加密、工具 |
| `references/how-tos/gui.md` | GUI 模式：动态控件、选项卡、GDI、拖放等 |
| `references/how-tos/net.md` | 网络、SOCK、COM/WMI 写法示例 |

## $10 输出规范

编写 PECMD 脚本和工具时遵循以下规范：

1. 中文脚本首行 `#code=65001`，随后 `ENVI^ EnviMode=1`，再 `ENVI^ ForceLocal=1`
2. 默认使用中文变量名——PECMD 社区的事实标准
3. 除非明确需要环境变量，始终使用 PE 变量（`&变量名`）
4. 大量使用 `TEAM` 链式组合紧凑的初始化序列
5. 将复杂操作拆分为命名清晰的 `_SUB` 函数
6. GUI 中用 `ENVI @控件.POS=?` 保存初始控件位置，在 resize 事件中恢复
7. 可调大小的窗口务必处理 WM_SIZE（0x0005）以重新计算布局
8. 调用外部程序优先用 `EXEC =!"%MyNAME%" ..."` 或 `EXEC* &out=!cmd.exe /c ...`
9. 线程安全：跨线程通信用 `&::` 类 PE 变量
10. 生产代码：用 `{ LOCK #pecmd ... }` 块包装互斥操作
11. 控件消息用 `_` 前缀（`_0x0201`）；窗口级消息省略 `_`（`0x0010`）
12. 用 `SED` 做字符串操作，用 `MSTR` 从结构化命令输出中提取字段


---

GitHub: https://github.com/VirtualHotBar/PECMD-Pro-Max
ClawHub: https://clawhub.ai/virtualhotbar/pecmd-pro-max
