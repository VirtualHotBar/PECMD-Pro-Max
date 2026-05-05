---
name: pecmd-pro-max
version: 1.5.5
description: |
  PECMD2012 WinPE 脚本编程 — 轻量级 Windows GUI、系统工具、启动/初始化
  脚本、自动化。适用于 .wcs/.wci/.wce 文件、磁盘分区、批处理转 PECMD、
  系统信息收集、PE/预安装环境工具、PECMD 代码调试。
  参考文件：references/commands-full.md、references/pecmd-gui.md、references/pe-startup.md、
  references/how-tos/storage.md、references/how-tos/system.md、references/how-tos/gui.md、references/how-tos/net.md、
  references/troubleshooting.md。
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
- 循环体内 `EXIT CONTINUE` 继续下一次迭代；`EXIT -` 跳到当前 `{}` 块尾部；`EXIT LOOP` / `EXIT FORX` / `EXIT BREAK` 跳出循环；`EXIT ToWin` 中止函数返回窗口消息循环
- 脚本文件常用 `.wcs` 扩展名（也支持 `.wce`、`.wci`、`.wcx`、`.ini`、`.inf`、`.txt`、`.log` 等）。中文脚本首行加 `#code=65001` 声明 UTF-8 编码。若首行以 `#!` 开头，编码指令放在第二行。

**标准 I/O（stdin/stdout/stderr）：**
- `READ -,n,&var` — 从 stdin 读取一行
- `WRIT -,$+0,text` — 写入 stdout
- `WRIT --,$+0,text` — 写入 stderr
- `LOGS * CONOUT$` — 输出到控制台（调试终端输出）

## $2 变量系统

PECMD 有**四种访问前缀**对应三种存储层级（加上就近查找机制）。搞错这一点是绝大多数错误的根源。

| 层级 | 语法 | 设置方式 | 作用域 |
|------|------|----------|--------|
| 环境变量 | `%var%` | `ENVI var=val` 或 `ENVI $var=val` | 进程级，与子进程共享 |
| PE 全局变量 | `%&::var%` | `ENVI &::var=val` 或 `SET &::var=val` | 跨函数、跨线程、文件级，永不消失 |
| PE 局部变量 | `%&&var%` | `SET &&var=val` 或 `ENVI &&var=val` | 当前 `_SUB`/`{}` 块，退出自动销毁 |
| PE 变量引用 | `%&var%` | `ENVI &var=val` 或 `SET var=val` | 就近解析：从内向外查找，找到谁就是谁；找不到则创建为 `&&var` |

**`&a` 解析顺序**（反向匹配，从内向外）：
1. 先找当前范围的 `&&a` → 找到就用它
2. 再往外层找 `&::a`（全局）→ 找到就用它
3. 都没找到 → 当作 `&&a` 在当前范围新建

**关键规则：**
1. `ENVI^ ForceLocal=1` 放在文件顶部——强制 `ENVI` 和 `SET` 默认创建局部 PE 变量。**务必始终使用**（默认 `ForceLocal=0`，变量按就近查找链解析）。
2. `ENVI^ EnviMode=1` — 空变量引用返回空字符串。务必始终使用（默认 `EnviMode=0`，兼容 4.0 模式：空变量不解释为字面文本，自动多轮次非顺序解释）。
3. `SET` **始终**等价于 `ENVI &`（SET 总是创建 PE 变量而非环境变量）。ForceLocal=1 同时影响 SET 和 ENVI 的作用域（默认局部）。
4. `%Desktop%` 是**环境变量**版本；`%&Desktop%` 是**PE 变量**版本。
5. 多线程代码中务必使用 PE 变量（`&var`）——环境变量跨线程共享会竞态。跨线程通信用 `&::` 类变量。
6. `SET~ &&dest=Source.Key` — `~` 运算符执行**间接解引用**：将右侧展开为变量名，再读取该变量的值。伪数组必备。
7. `^` 前缀（`^SET`、`^ENVI`、`^CALC`、`^IFEX` 等）— 预先解释本命令（help.txt: "命令前若干个^表示预先解释本命令几次"）。循环中动态变量名必备。`^^` 预解释两次。
8. `ENVI-ret %~1=%var%` — 将值设置到*名称*存储在 `%~1` 中的变量（引用返回）。
9. `SET-def var=value` — 仅在变量未定义时设置（安全默认值）。

### ENVI^ / ENVI @@ 运行时控制命令

| 命令 | 功能 |
|------|------|
| `ENVI^ EnviMode=1` | 标准模式：空变量解释为空字符串，顺序解释1遍 |
| `ENVI^ ForceLocal=1` | 强制所有变量为 PE 变量，最简多线程/并行窗口 |
| `ENVI^ EXPORTLOCAL=1\|0\|&1\|&0` | PE 变量继承：1=继承，0=隔离（默认），&1=仅本级及以下继承，&0=仅本级及以下隔离 |
| `ENVI^ Alias name=cmd` | 命令别名：替代命令前半部（可包装 DLL 调用为伪命令） |
| `ENVI^ WndProc[1\|2\|3][C][,ptr]` | Win32 回调绑定（C=C 调用约定） |
| `ENVI^ memvar=[?返回名,][:字节数:]偏移,值` | 修改/查询 PECMD 内存变量 |
| `ENVI^ LoadPlugin=basename` | 加载插件 |
| `ENVI^ DisX64=1[,Old]` | 禁用/恢复 WOW64 文件系统重定向 |
| `ENVI^ zero=0\|1` | 私密模式：内存用完清零 |
| `ENVI^ Arg=*[字符集]字符串` | 词语分断 |
| `ENVI^ Clipboard=text\|?=var` | 读写剪贴板 |
| `ENVI^ EnviBroad=0\|1\|-` | 环境变量广播开关 |
| `ENVI^ __arg=0\|1` | 兼容模式：启用 `&&__arg` 参数表 |
| `ENVI^ LoadEnvi [路径\|-] [变量名]` | 从注册表刷新环境变量 |
| `ENVI^ HelpColor=[*高度] [前景色][#背景色]` | HELP 显示颜色 |
| `ENVI @@DeskTopFresh=[clearicon][;][1\|2\|4\|8\|16][;[-+]path]` | 桌面刷新 |
| `ENVI @@TaskIcoMenu=0\|1\|2` | 托盘菜单切换（0=关闭, 1=开启, 2=切换） |

> 完整 ENVI^ 控制命令、二进制缓冲区操作（SET-cmp、SET-tom、SET-copy、SET?int 等）和函数参数引用，参见 [commands-full.md](references/commands-full.md)。

### ENVI-env / SET-env — 临时绕过 ForceLocal

`ENVI-env` 和 `SET-env` 后缀临时绕过 ForceLocal=1 限制，直接操作环境变量。常用于线程内读取父窗口上下文的环境变量（如 `__WinID`）：

```wcs
ENVI-env hwnd0=%&__WinID%           // 从环境变量读取 __WinID（绕过 ForceLocal）
SET-env MyName=%MyName%             // 临时设为环境变量
```

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

## $3 代码组织

### 函数与 CALL 变体

```wcs
_SUB 函数名 [*]                   // * = this-call（在调用者栈中运行）
    SET &param1=%~1                 // %~1 剥离外层引号
    ENVI-ret %~3=%&result%          // 通过引用参数返回值
    // 函数参数：%0=函数名, %1~%n=各参数, %#=参数个数, %*=全部参数, %~=去掉引号
_END
```

```
CALL 函数名 [参数]                 // 调用函数
CALL *函数名 [参数]                // this-call（调用者栈）
CALL @窗口名 [参数]                // 创建/显示窗口（模态，阻塞）
CALL @*窗口名 [参数]               // 并行窗口（可同时操作，但关闭前阻塞后续命令）
CALL @-窗口名 [参数]               // 后台窗口
CALL @~窗口名 [参数]               // 后台，完全非阻塞
CALL @~~窗口名 [参数]              // 后台快速调用（非阻塞）
CALL @^窗口名 [参数]               // 并行，父窗口不阻塞子窗口
CALL @+窗口名 [参数]               // 弃养子窗口
CALL @--                           // 销毁 Win 环境（无参数）
CALL @--popmenu 窗口名 [x.y[:对齐]] // 在指定位置弹出菜单（对齐：对齐方式）
CALL --mem &变量名 [*] [参数]      // 执行内存中的动态函数代码
```

### 窗口（GUI）

```wcs
_SUB 窗口名,L200T100W400H300,窗口标题,[关闭命令],[图标],[样式],[遮罩] [-标志1 -标志2 ...]
_END
```

窗口形状：`L<左>T<上>W<宽>H<高>`。省略 L/T 则居中。
窗口样式：`-` = 无标题栏，`#` = 无边框，数字 1-99 = 透明度，`:`透明色。
常用标志：`-trap`（关闭按钮不退出）、`-nocap`（无标题栏）、`-nosysmenu`、`-top`、`-size`、`-maxb`、`-disminb`、`-discloseb`、`-nfocus`、`-ntab`、`-forcenomin`、`-scalef`、`-scale[:DPI]`、`-nxp`、`-csize`、`-na`、`-layer`（支持渐透明）、`-disaltmv`（禁用 ALT 拖动）

### IMPORT 与代码块

```wcs
IMPORT 路径\库文件.wcs          // 将可复用函数加载到当前脚本
_ENDFILE-IMPORT                // 此行以下内容在 IMPORT 时被丢弃

{                               // 带独立 PE 变量栈的代码块
    SET temp=仅此处有效
}
{*                              // this-call 代码块（在调用者栈中运行）
    LOCK .ole                   // COM 初始化等需要在当前作用域执行的操作
}
```

文件级和函数级的 `{` 必须从第 1 列开始。嵌套 `_SUB` 用双冒号访问：`类名::子函数名`。点号用于数据成员访问（如 `类名.属性`）。

## $4 流程控制

### FIND — 字符串比较（默认不区分大小写，`*c` 后缀区分）

```wcs
FIND $%var%=hello, 命令               // 相等（不区分大小写）
FIND $%var%*c=hello, 命令             // 相等（区分大小写，*c 后缀）
FIND $%var%<>hello, 命令              // 不等
FIND $%var%=, 命令                    // "为空" 测试（惯用法）
FIND *=var, 命令                      // 惯用法："为空"
FIND *<>var, 命令                     // 惯用法："非空"
FIND |%a%>%b%, 命令                   // | 前缀 = 浮点数比较
FIND #%a%>%b%, 命令                   // # 前缀 = 整数比较（INT64）
FIND [ $A & $B ], 命令                // 复合 AND
FIND [ $A | $B ], 命令                // 复合 OR（| 分隔）
FIND --pid &var,                     // 返回 5 值：空闲时间 总时间 CPU个数 1秒时钟数 一时钟100ns数
FIND --pid*@ &var,                   // 进程列表（用于 TABL）
FIND --class:Shell_TrayWnd --wid*@ &var  // 按类名过滤窗口列表
FIND --forpid:PID --wid*@ &var       // 按进程 ID 过滤窗口
FIND --sub --wid*@ &var              // 递归子窗口
FIND --menu &var,窗口ID              // 查询窗口的 MENU 句柄
FIND --menu#Index &var,MenuID        // 按索引查询子 MENU
```

### IFEX — 文件测试 / 数值比较

```wcs
IFEX C:\boot.ini, 命令                // 文件/目录存在
IFEX C:\boot.ini,! 命令               // 不存在
IFEX x:\, 命令                        // 盘符存在且有文件系统
IFEX $%val%>=5, 命令                  // $=浮点数比较（#=整数比较，|=字符串比较）
IFEX [ 条件1 & 条件2 ], 命令          // AND 复合条件
IFEX [ 条件1 | 条件2 ], 命令          // OR 复合条件（| 分隔）
IFEX MEMU=?,&var                     // 查询可用内存
IFEX C:\=?,&可用空间                  // 查询磁盘可用空间
```

### LOOP / FORX / TEAM / LOCK

```wcs
SET &&I=0
LOOP #%&I%<=10, { MESS %&I% | CALC &I=%&I% + 1 }

FORX * %&list%,&&item, { MESS %&item% }              // * = 空格分隔
FORX *NL &多行变量,&&line, { ... }                    // *NL = 换行分隔
FORX *NL:| &数据,&&item, { ... }                     // *NL:分隔符 = 自定义分隔符
FORX *L 0 2 10,&&val, { ... }                        // *L = 数值循环（起始 步长 结束）
FORX /S C:\Windows\*.exe,&&file,0 { ... }            // 文件系统枚举
FORX @\Windows,&&winDir,1 { ... }                    // @=仅搜索目录，\=所有盘符

// ⚠ FORX 循环变量引用规则：
// 定义用 &&item，引用值用 %&item%（单&），**不是** %&&item%
// 示例：FORX *NL &list,&&line, { SET &val=%&line% }  // ✅ 正确
//       FORX *NL &list,&&line, { SET &val=%&&line% }  // ❌ 错误

TEAM SET &a=1| SET &b=2| CALC &c=%&a% + %&b%         // 多命令链

// 线程
THREAD* CALL WorkerFunc                             // * = 持久栈（共享变量，保持父子关系）
THREAD CALL WorkerFunc                              // 独立模式（复制变量到子线程）
THREAD& CALL WorkerFunc                             // & = 强制 PE 变量模式（最简多线程）
THREAD+ CALL WorkerFunc                             // + = 抛弃式线程（不等待退出）
THREAD# CALL WorkerFunc                             // # = 代理模式（线程结束则退出）
THREAD $ CALL WorkerFunc                            // $ = 预先解释命令组
THREAD -link CALL WorkerFunc                        // 保持父子关系（类似 *）
THREAD -wait CALL WorkerFunc                        // 等待完成（阻塞消息循环）
THREAD -waitx CALL WorkerFunc                       // 等待完成（不阻塞消息循环）
THREAD -here CALL WorkerFunc                        // 当前栈的孩子（可修改父栈临时PE变量）
THREAD -waitp CALL WorkerFunc                       // 进程结束前等待该线程
THREAD -tid:&tid CALL WorkerFunc                    // 获取线程 ID
THREAD --st:128K CALL WorkerFunc                    // 设置栈大小

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
| 定时器 | `TIME` | 托盘/气泡 | `TIPS` / `TIPS*` |
| 选项卡 | `TABS` | 滑块 | `SLID` |
| 微调器 | `SPIN` | 树形视图 | `TREE` |
| 滚动条 | `SBAR` | 热键 | `HKEY` |
| 日期时间 | `DTIM` | IP 地址 | `IPAD` |
| 菜单项 | `MENU` | 屏幕捕捉 | `SCRN` |
| 浏览窗口 | `BROW` | | |

### 消息映射

```wcs
ENVI @控件.MSG=_%&WM_LBUTTONDOWN%: 命令        // _ = 后置系统处理器（控件通知）
ENVI @窗口.MSG=0x0010: CALL OnClose             // 无 _ = 窗口级消息（WM_CLOSE）
ENVI @控件.MSG=_%&msg%::&&wp,&&lp, CALL Handler // 捕获 wParam/lParam
ENVI @控件.POSTMSG=#1                            // 投递自定义消息 #1
ENVI @控件.SENDMSG=消息号;wParam;lParam          // 同步发送
ENVI @控件.MSG=$0x0201: 命令                     // $ = 屏蔽系统响应（可返回结果码）
ENVI @控件.MSG=*msg: 命令                        // * = 捕鼠器 B（鼠标捕获模式）
ENVI @控件.MSG=+msg: 命令                        // + = 超级捕获
```

MSG 前缀说明：无前缀=直接处理后丢弃（不传递给系统），`_`=系统响应后执行（控件通知），`$`=屏蔽系统响应（可返回结果码），`*`=捕鼠器 B（鼠标捕获），`+`=超级捕获。消息号 `*del` 删除映射。`#1`~`#N` 为 PECMD 自定义消息。

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

完整控件语法、全部 25 种控件类型、ENVI @ 属性参考、窗口生命周期和消息映射——参见 [pecmd-gui.md](references/pecmd-gui.md)。

GUI 写法示例（动态控件、选项卡页、自定义标题栏、GDI 绘图、拖放等）——参见 [how-tos/gui.md](references/how-tos/gui.md)。

## $6 DLL 调用

> **实测验证（32-bit / 64-bit 行为一致）：** 以下内容在 PECMD 32-bit (PECMDx86) 和 64-bit (PECMD) 上均经实测确认，行为完全相同。DLL 调用可正常工作，但有重要限制。

### 调用约定

DLL 函数**默认按 PASCAL/stdcall** 调用约定。Windows API（WINAPI 标记）不需要 `--c`。使用 `--c` 切换到 CDECL。最多 20 个参数（C 调用 100 个）。

### 正确语法（逗号分隔 + 类型前缀 + --qd）

```wcs
// ✅ 正确格式：逗号分隔，整数 #，字符串 $，建议加 --qd
CALL $ --qd --ret:&&r user32.dll,GetSystemMetrics,#0             // 整数参数
CALL $ --qd --ret:&&r user32.dll,FindWindowW,$Progman,#0         // 字符串+整数
CALL $ --qd --ret:&&r kernel32.dll,lstrlenW,$Hello World          // 字符串参数
CALL $ --ret:&&hDll ,-LoadLibrary,^user32.dll                     // 加载 DLL（^ = 自动释放）
CALL $ --ret:&&r *%&hDll%,GetSystemMetrics,#0                     // 通过句柄调用
CALL $ ,-FreeLibrary,*%&hDll%                                      // 释放 DLL
CALL $ --bool --ret:&&ret DLL,函数,...                             // --bool：函数返回 BOOL
CALL $ --cd --ret:&&ret DLL,函数,...                               // --cd：执行前切换到目标目录

// Win32 回调函数绑定（用于 EnumResourceNames 等 API）
ENVI^ WndProc1C,WndProc1Addr               // C 调用约定回调
SET^ WndProc1,WndProc1Addr                  // PASCAL/stdcall 回调
// PECMD 自动查找 _SUB OnWndProc1 并包装为 Win32 回调
```

### 关键规则

1. 参数**必须**用逗号分隔——点语法（`Func.#Param`）完全不工作
2. 整数参数**必须**有 `#` 前缀——无前缀返回 0
3. 字符串**必须**用 `$` 前缀
4. **务必加 `--qd`**——否则字符串多传一个 null 终止符，导致 API 结果错误：

| 字符串 | 无 `--qd` | 有 `--qd` | 说明 |
|--------|-----------|-----------|------|
| `$A` | lstrlenW → 2 | lstrlenW → 1 | 无 --qd 含 null |
| `$Test` | lstrlenW → 5 | lstrlenW → 4 | 无 --qd 含 null |
| `$Progman` | FindWindowW → 0 | FindWindowW → 66062 | 无 --qd 找不到窗口 |

### ❌ 不可用的语法

```wcs
// ❌ 点语法完全不工作
CALL $--qd --ret:&&r user32.dll,GetSystemMetrics.#0              // 返回空

// ❌ 整数无前缀返回 0
CALL $ --ret:&&r user32.dll,GetSystemMetrics,0                   // 返回 0

// ⚠ GetProcAddress：无 --qd 时常返回 0x0，加 --qd 后可工作
CALL $ --ret:&&p ,-GetProcAddress,*%&hDll%,$GetSystemMetrics     // 可能返回 0x0
CALL $--qd --ret:&&p ,-GetProcAddress,*%&hDll%,GetSystemMetrics  // 加 --qd 更可靠

// ⚠ 缓冲区输出：无 --qd 时数据可能不回传，加 --qd 后可工作
SET$ &buf=*256 0
CALL $--qd --ret:&&r kernel32.dll,GetWindowsDirectoryW,*&buf,#256  // --qd 模式

// ❌ 无 --qd 时字符串含 null 终止符，FindWindowW 等失败
CALL $ --ret:&&r user32.dll,FindWindowW,$Progman,#0             // 返回 0
```

### 可靠工作的场景

| 场景 | 语法 | 状态 |
|------|------|------|
| 无参整数返回 | `CALL $ --ret:&&r kernel32.dll,GetTickCount` | ✅ |
| 整数参数 | `CALL $ --qd --ret:&&r user32.dll,GetSystemMetrics,#0` | ✅ |
| 字符串输入 | `CALL $ --qd --ret:&&r user32.dll,FindWindowW,$Progman,#0` | ✅ |
| 变量作参数 | `CALL $ --qd --ret:&&r kernel32.dll,lstrlenW,$%&str%` | ✅ |
| LoadLibrary | `CALL $ --ret:&&h ,-LoadLibrary,^user32.dll` | ✅ |
| 通过句柄调用 | `CALL $ --ret:&&r *%&h%,GetSystemMetrics,#0` | ✅ |
| 缓冲区输出 | 加 `--qd` 后可回传到 PE 变量 | ⚠ |
| GetProcAddress | 加 `--qd` 后可工作 | ⚠ |
| 点语法 | Func.#Param 不解析 | ❌ |

### 32-bit vs 64-bit 区别

| 项目 | PECMDx86 (32-bit) | PECMD (64-bit) |
|------|-------------------|----------------|
| 文件标识 | PE32, i386 | PE32+, x86-64 |
| `bX64` 值 | `0`（32 位系统）或 `1`（64 位系统） | `3` |
| 指针大小 | 4 字节（< 4GB） | 8 字节（> 4GB） |
| DLL 调用行为 | 与 64-bit 完全相同 | 与 32-bit 完全相同 |
| LoadLibrary 句柄 | `0x761A0000` | `0x7FFC34B00000` |
| 代码差异 | 无 | 无 |

`bX64` 值含义：`0`=WIN32（32 位系统原生），`1`=WIN64（64 位系统运行 32 位 PECMD），`3`=PECMD64（64 位 PECMD）。

### 参数类型前缀

| 前缀 | 类型 | 描述 | 示例 |
|------|------|-------------|------|
| `#` | integer | 按整数传递 | `#0`、`#256`、`#%&hwnd%` |
| `<` | INT64 | 64 位整数 | `<0x100000000` |
| `$` | string | 按 Unicode 字符串传递（配合 `--qd` 不含 null） | `$CabinetWClass` |
| `@` | ANSI | 按 ANSI 字符串传递 | `@text` |
| `*` | buffer | 传递 PE 变量地址（`*&buf` 形式可回传输出） | `*&buf` |
| `=` | raw | 按原始数据传递 | |
| `~` | deref | 间接解引用（展开变量名再读值） | |

### 缓冲区操作（PE 变量内部）

```wcs
SET$# &buf=*256 0                     // 分配 256 字节原始缓冲区
SET$ &strBuf=*256 0                   // 分配 256 个宽字符
SET-long &buf=value:offset            // 写入 32 位整数到缓冲区偏移
SET?int &buf=&&var:offset             // 从缓冲区偏移读取 32 位整数
```

### DLL 输出数据的替代方案

DLL 缓冲区输出可使用 `*&buf` 形式（PE 变量地址传递，输出回传）。获取系统信息也可用内置命令：
- `FIND --wid*@ &var` — 枚举窗口（返回 HWND、类名、标题）
- `FIND --pid*@ &var` — 枚举进程
- `REGI` — 注册表读取系统信息
- `EXEC* &out=!cmd.exe /c ...` — 捕获外部命令输出
- `IFEX MEMU=?,&var` — 查询可用内存
- `IFEX C:\=?,&var` — 查询磁盘空间

### 平台检测

```wcs
IFEX #%&::bX64%=3, SET &PtrSz=8! SET &PtrSz=4  // 3=PECMD64, 1=WIN64(32位PECMD在64位系统), 0=WIN32
```

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
WRIT %路径%,+0,新行                                // 追加一行
GETF# %路径%,0#*,&原始数据                         // 按原始字节读取文件
```

### 执行与捕获

```wcs
EXEC* &输出=!cmd.exe /c dir /b                            // 捕获全部输出
EXEC =!"%MyNAME%" TEAM WAIT 1000|LOAD other.ini           // 运行子 PECMD
EXEC* &out=!cmd.exe /c ipconfig                           // 捕获外部命令输出
EXEC* --exe:#101 &out=*embedded.exe                       // 从 PECMD 资源运行 EXE
```

### 单实例互斥体

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

### 字符串操作

```wcs
MSTR &&a,&&b=<1><3->%&data%                              // 字段1、字段3到末尾（- = 到末尾）
MSTR &&last=<-1>%&data%                                   // 最后字段（负索引）
MSTR &&q=<~5>%&data%                                      // 字段5并剥离外层引号（~前缀）
MSTR -delims:. &&a,&&b,&&c,&&d=<1><2><3><4>%&ip%         // 按自定义分隔符拆分
MSTR* &&a,&&b=<1><2>%&data%                               // *前缀 = TAB 分隔
MSTR$ &&a,&&b=<1><2>%&data%                               // $前缀 = 空格分隔（连续空格视为单个）
SED &&r=0,模式,替换,%&source%                             // 替换所有匹配（正则模式）
SED &&r=1,模式,替换,%&source%                             // 替换第一个匹配
SED &&r=-1,模式,替换,%&source%                            // 替换最后一个匹配（负数=从末尾）
SED -t &&r=0,ABCD,abcd,%&source%                          // -t = 字符集翻译（A→a, B→b, 逐字符映射）
SED -ni &&r=0,模式,替换,%&source%                         // -ni = 不区分大小写
SED &&ext=-1,.*\.,,  ,%&filename%                          // 获取扩展名（.*\. 匹配到最后一个句号）
LPOS &&pos=needle,,%&haystack%                            // 查找首次出现（不区分大小写）
```

> **MSTR 注意事项：** 默认分隔符为空格。`-delims:09` 指定制表符（十六进制 09）。字段索引从 1 开始。`<N>` = 单个字段，`<N->` = 字段N到末尾，`<N*>` = 仅字段N（单字段），`<~N>` = 剥离引号，`<-N>` = 从末尾反向索引。`MSTR*` = TAB分隔，`MSTR$` = 空格分隔（连续空格视为单个）。如果 MSTR 返回空，先用 `WRIT` 打印源变量确认格式，或改用 `FORX *` 按空格逐字段拆分作为替代。
>
> **SED 注意事项：** SED 默认使用正则模式。`.` 匹配任意字符，非字面量句号。匹配字面量句号用 `\.`。使用标志字符 `*` 可切换为字面量模式（不解释正则）。count=0 替换全部匹配，count=N 替换前 N 个，count=-N 替换后 N 个。`0:0` 与 `0` 等价。`-t` 标志启用字符集翻译，`-ni` 标志不区分大小写。模式中的空格作为参数分隔符处理时可能引起问题——用变量间接传递含空格的模式。
>
> **LPOS 注意事项：** 参数顺序为 `LPOS &&pos=needle,,haystack`（查找目标在前，源字符串在后）。返回首次出现的位置索引。加 `,1,` 可区分大小写。`RPOS` 从末尾查找。

含错误处理和边界情况处理的完整展开版本，参见 [how-tos/storage.md](references/how-tos/storage.md)（磁盘/注册表/文件）、[how-tos/system.md](references/how-tos/system.md)（进程/线程/定时器）、[how-tos/net.md](references/how-tos/net.md)（网络/COM）。

### CALC 计算与数学函数

```wcs
CALC &结果=%&a% + %&b%                               // 基本算术：+ - * / % ^
CALC #&结果=%&a% & %&b%                               // # = 整数模式；位运算：& | @
CALC -base=16 #&hex=shl(0x07,16)|0x20                 // 十六进制输出 + 位移
CALC &sz=%&bytes%/1G#3                                // 字节转 GB，3 位小数
CALC -err=0 &r=%&a% / %&b%                            // 出错时返回默认值 0
```

**数学函数（34个）：**

| 类别 | 函数 |
|------|------|
| 三角 | `sin cos tan ctg arcsin arccos arctan arcctg deg rad` |
| 代数 | `abs sqrt ln lg log exp pow pow10 hypot`（⚠ `log()` 需两参数 `log(底数,值)`，单参数返回 0；用 `lg()` 代替 log₁₀） |
| 取整 | `floor ceil round int frac` |
| 位运算 | `shl shr xor not lnot` |
| 其他 | `div mod rand max min` |

常量：`e`、`pi`。尺寸后缀：`K`=1024、`M`=1024²、`G`=1024³、`T`=1024⁴、`S`=512。
多表达式用 `;` 分隔，子变量 `$subName=expr`。

## $8 陷阱与注意事项

1. **CALC 空格**：`CALC &J=1+2` 可以正常执行。右侧以 `%&I%` 等变量开头时，用空格分隔（`CALC &J= %&I%+1`）。PECMD 要求减号后必须有空格（`3 - 2`）。
2. **注释标记**：`//`、`;`、`` ` ``（反引号）均为有效注释符。行尾注释前必须有空格（如 `SET &a=1 // 注释`）。help.txt: "注释符前有一个空字符，该空字符算注释"——所有位置的注释符前都建议加空格。
3. **SET 就是 ENVI &**：`SET var=val` 语义上等价于 `ENVI &var=val`。启用 ForceLocal=1 后，两者都创建局部 PE 变量。
4. **FIND 与 IFEX 前缀交换**：FIND 中 `$` = 字符串比较、`|` = 浮点比较；IFEX 中 `$` = 浮点比较、`|` = 字符串比较。`#` 在两者中均为整数比较。IFEX 中 `|` 在 `[]` 复合条件内为 OR 逻辑运算符。
5. **盘符冒号**：`FDRV`、`FORM`、`FIND C:\=?` 都需要 `:` 后缀。
6. **带空格的路径**：`LOAD "C:\Program Files\a.ini"` 需要引号。
7. **文件编码**：中文脚本首行声明编码（`#code=65001` = UTF-8，`#code=936` = GBK）且文件编码须与声明一致。SDK 示例普遍使用 GBK（936）。
8. **`{` 位置**：文件级和函数级 `{` 必须从第 1 列开始。在 TEAM/LOOP/IFEX 内部，`{` 启动命令组。
9. **行续接**：行首第一个非空格字符为 `\` 时，将该行合并到上一行。
10. **`_SUB`、`WRIT`、`LOOP` 不能内联**：不能在 FIND/IFEX/TEAM 命令内部定义 `_SUB`。`WRIT` 和 `LOOP` 也必须位于单独一行，不能嵌套在 FIND/IFEX/TEAM 内。
11. **空字符串检测**：`FIND $%var%=,` 测试"为空"。`FIND *=var,` 也测试"为空"（惯用法）。`FIND *<>var,` 测试"非空"。
12. **字面 %**：字符串中表示字面 `%` 用 `%%`。
13. **线程安全**：线程创建时，**共享点之后**的 PE 变量复制到子线程。持久栈上下文（窗口、`*` 函数）中的变量是**共享**的（非复制）；非持久上下文（普通函数、`{}` 块）中的变量自动复制一份。真正跨线程通信用 `&::` 变量。
14. **中文变量名**：PECMD 社区的事实标准。为这个生态系统编写脚本时使用中文名称。
15. **OnShutdown.wcs**：PECMD 在关机/重启/注销前自动运行 `%SystemRoot%\System32\OnShutdown.wcs`，格式为 `OnShutdown.wcs <操作码> [脚本参数表]`。操作码：`shutdown`=关机、`reboot`=重启、`logout`=注销、`suspend`=挂起、`hiber`=休眠、`poweroff`=关电、`lock`=锁定计算机、`unknown`=未知。关机菜单支持的子集：`shutdown`、`reboot`、`logout`、`poweroff`、`unknown`。
16. **`^` 预解释**：`^COMMAND` 将变量展开推迟到执行时（循环中必备）。`^^COMMAND` 预解释两次。
17. **MSG 上的 `_` 前缀**：控件通知用 `_msg#`；窗口级消息省略 `_`。搞错这一点是非常常见的错误。
18. **THREAD\* 与 THREAD**：只有持久（窗口）栈中的 THREAD* 共享 PE 变量。在 `{}` 块中，两者都复制。
19. **FIND 展开规则**：FIND 中的裸标识符被视为字面字符串。始终使用 `FIND $%&var%=值` 引用 PE 变量。
20. **`@@Visible` 与 `@Visible`**：跨进程用 `ENVI @@Visible=窗口ID:值`。进程内用 `ENVI @控件.Visible=0|1`。
21. **FORX 变量引用**：`FORX ... &&item, { }` 定义用 `&&item`，循环体内引用值用 `%&item%`（单&）。用 `%&&item%` 会找不到变量。
22. **FIND --wid\*@ 输出格式**：`序号 窗口ID 控件ID 父窗口ID 线程ID 进程ID 类型 标题`，字段用 **TAB** 分隔。配合 `FORX *NL` 按行遍历、`MSTR*` 按 TAB 分字段提取（field 2=窗口ID, field 7=类型, field 8=标题）。
23. **DLL 调用不可用时**：优先用 `FIND --wid*@`（窗口枚举）、`FIND --pid*@`（进程枚举）、`REGI`（注册表）、`EXEC*`（外部命令）替代。
24. **`Visible` 与 `Visable`**：两者均可工作（PECMD 接受两种拼写）。文档统一用 `Visible`，但 SDK 示例普遍用 `Visable`。
25. **FORX *NL: 自定义分隔符**：`FORX *NL:| &data,&&item,` 用 `|` 作为行分隔符（默认为换行）。
26. **ENVI^ ALIAS 包装 DLL**：`ENVI^ ALIAS GetTick=CALL $ --ret:&&r kernel32.dll,GetTickCount` 使 `GetTick` 可作为伪命令直接调用。

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
| `references/troubleshooting.md` | 常见错误排查、DLL 调用失败、变量引用问题 |

## $10 输出规范

编写 PECMD 脚本和工具时遵循以下规范：

1. 中文脚本首行 `#code=65001`，随后 `ENVI^ EnviMode=1`，再 `ENVI^ ForceLocal=1`
2. 默认使用中文变量名——PECMD 社区的事实标准
3. 除非明确需要环境变量，始终使用 PE 变量（`&变量名`）
4. 大量使用 `TEAM` 链式组合紧凑的初始化序列
5. 将复杂操作拆分为命名清晰的 `_SUB` 函数
6. GUI 中用 `ENVI @控件.POS=?` 保存初始控件位置，在 resize 事件中恢复
7. 可调大小的窗口务必处理 WM_SIZE（0x0005）以重新计算布局
8. 调用外部程序优先用 `EXEC =!"%MyNAME%" ...` 或 `EXEC* &out=!cmd.exe /c ...`
9. 线程安全：跨线程通信用 `&::` 类 PE 变量
10. 生产代码：用 `{ LOCK #pecmd ... }` 块包装互斥操作
11. 控件消息用 `_` 前缀（`_0x0201`）；窗口级消息省略 `_`（`0x0010`）
12. 用 `SED` 做正则替换（始终正则模式，`.` 匹配任意字符，`\.` 匹配字面量句号），用 `MSTR` 从结构化输出中提取字段——如果 MSTR 不工作，改用 `FORX *` 逐字段拆分


---

GitHub: https://github.com/VirtualHotBar/PECMD-Pro-Max
ClawHub: https://clawhub.ai/virtualhotbar/pecmd-pro-max
