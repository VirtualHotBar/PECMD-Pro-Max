# PECMD 命令参考

按类别组织的完整命令参考。所有命令不区分大小写。
语法约定：`<required>` 为必填，`[optional]` 为可选，`|` 表示多选一。

---

## 脚本结构

### _END
结束一个 `_SUB` 函数或代码块。
```
_END
```
必须单独一行。每个 `_SUB` 需要一个对应的 `_END`。

### _ENDFILE
脚本文件结束。该行之后的代码不会被加载。
```
_ENDFILE[-IMPORT]
```
`-IMPORT`：仅在文件被 IMPORT 时生效。

### _SUB — 定义函数 / 类 / 窗口
```
_SUB FuncName [*]                   // 函数 (* = this-call，使用调用者栈)
_SUB FuncName,*,,destroyCmd         // 带析构命令的函数
_SUB WinName,<shape>,[title],[closeCmd],[icon],[style],[mask] [-flag1 -flag2 ...]  // 窗口
```
窗口形状：`LleftTtopWwidthHheight`。省略 L/T 为居中。
窗口标志：
`-top`（始终置顶），`-nocap`（无标题栏），`-nosysmenu`（无系统菜单），
`-trap`（关闭时不退出），`-size`（可调整大小），`-maxb`（启用最大化），
`-minb`（启用最小化），`-disminb`（禁用最小化按钮），
`-discloseb`（禁用关闭按钮），`-nfocus`（不接受键盘焦点），
`-ntab`（无 Tab 键导航），`-disaltmv`（禁用 ALT 拖动），
`-forcenomin`（阻止最小化），`-scalef`（XP 风格 DPI 缩放），
`-scale[:DPI]`（Win8+ DPI 缩放），`-nxp`（无 XP 视觉样式），
`-csize`（尺寸=客户区），`-na`（创建时不激活）
样式：`[#][$]`数字 表示透明度，`,#` 表示隐藏窗口。
遮罩：`[color][*][w:h]bmpname` 用于异形窗口。

### CALL — 调用函数/窗口/DLL
```
CALL FuncName [args...]               // 调用函数
CALL *FuncName [args...]              // this-call（使用调用者栈）
CALL @WinName [args...]              // 创建/显示窗口（模态，会阻塞）
CALL @*WinName [args...]             // 并行窗口
CALL @-WinName [args...]             // 后台窗口
CALL @~WinName [args...]             // 后台，非阻塞
CALL @+WinName [args...]             // 抛弃式子窗口
CALL @^WinName [args...]             // 并行，父窗口不阻塞子窗口
CALL @--popmenu WinName [x.y[:align]] // 弹出菜单
CALL @--WinName                      // 销毁 Win 环境
CALL @WinName                        // 初始化 Win 环境

// DLL 调用（缓冲区输出用 `*&buf` 形式传递，见下方说明；需要 PECMD2012 v1.88+ 完整版）
CALL $[? --cd --nrcd --c --[[i]v]ret:[~@]retVar] DLL|*hDll,Func,[#]p1,[#]p2...
CALL $--ret:retVar [--cd],[--nrcd],-LoadLibrary,[^]DLLpath     // 加载 DLL（^=自动释放）
CALL $--ret:retVar [&&memVar],-LoadLibrary,*[file]#resID[|type] // 从内存加载
CALL $--ret:retVar ,-GetProcAddress,*hDll,FuncName             // 获取函数地址
CALL $[--ret:retVar] ,-FreeLibrary,*hDll                        // 释放 DLL
CALL $--win [--qd@ --cd --nrcd --ret:retVar] DLL,Func,cmdLine   // rundll32
CALL $--cpl CPLpath                                             // 控制面板
CALL $--ret:var ,-LoadLibrary,^<DLLpath                         // COM DLL 加载
```
DLL 标志：`--cd`=切换目录，`--nrcd`=不恢复目录，`--c`=C 调用约定（默认 PASCAL/stdcall），
`--bool`=BOOL 返回，`--ret:*`=通过指针返回，`--m`=内存中，
`--1`=剩余全部作为一个参数，`--co`=注册 DLL（默认），`--nco`=不注册 DLL。
`--qd`=启用类型前缀系统（qualified mode），允许用 `#`/`$`/`*` 等前缀为每个参数指定类型。

**类型前缀系统（--qd）：** 每个参数的类型覆盖，使用 `--qd:类型1,类型2,...`
| 前缀 | 类型 | 描述 |
|--------|------|-------------|
| `#` | integer | 按整数传递（数字的默认方式） |
| `<` | INT64 | 64位整数 |
| `*` | PE variable | 按 PE 变量指针传递 |
| `$` | string | 按字符串传递（Unicode） |
| `=` | raw | 按原始数据传递 |
| `>` | VARIANT | 按 VARIANT 传递 |
| `@` | narrow | ANSI 窄字符串 |
| `~` | UTF8 | UTF-8 字符串 |

附加标志：
```
--sret           // 返回符号数量
--16             // 以十六进制返回
--iret:retVar    // 以 INT 返回
--vret:[~@]var   // 返回 VARIANT（~=剥离，@=原始）
--arg:~.table    // 参数备选格式（~=去除引号）
.vFun:index      // 虚函数索引
.vFun:[?]name    // IDispatch 函数名（[propget] 前缀已移除）
--get / --put    // COM 属性 获取/设置
?                // 查询 DLL 函数地址（存入返回变量）
^<               // COM DLL 加载前缀
^                // 变量超出作用域时自动释放
```

**实测验证的调用格式（PECMD2012 v1.88+, 32-bit / 64-bit 行为一致）：**
- 正确格式：`CALL $ --qd --ret:&&r DLL,Func,#intParam,$strParam`（逗号分隔，`#` 整数，`$` 字符串）
- `--qd` 影响字符串传递方式（无 --qd 时字符串多含 null 终止符，建议始终加 `--qd`）
- 点语法（`Func.#param`）不工作，整数无 `#` 前缀返回 0
- **缓冲区输出**：`*&buf` 形式传递 PE 变量地址，DLL 写入可回传；`SET$#` 原始缓冲区 + `*` 不加 `&` 可能不回传
- `GetProcAddress` 始终返回 0x0（实现问题），建议直接用函数名调用
- 整数返回和字符串输入参数正常工作，32-bit 和 64-bit 行为完全一致

地址调用：DLL 路径=`#`，函数=原始地址。函数名加 `*` 前缀 = 取地址。参数加 `&` 前缀 = 组合变量地址。
内置：`-DllRegisterServer` / `-DllUnregisterServer`。
DLL 架构必须与 PECMD 进程（x86/x64）匹配。

### EXIT — 终止
```
EXIT FILE      // 终止整个脚本
EXIT WIN       // 退出当前窗口（销毁窗口环境）
EXIT _SUB      // 退出当前函数
EXIT LOOP      // 跳出循环（同 EXIT BREAK）
EXIT FORX      // 跳出 FORX 循环（同 EXIT BREAK）
EXIT CONTINUE  // 继续下一次迭代（LOOP/FORX）
EXIT BLOCK     // 跳到当前 {} 块尾部
EXIT -         // 同 EXIT BLOCK
EXIT _SIB      // 跳过当前 FORX 迭代的剩余兄弟命令，进入下一迭代
EXIT ToWin     // 中止函数执行，立即返回窗口消息循环（推荐使用）
```

### IMPORT — 包含库文件
```
IMPORT path\to\library.wcs
```
从另一个文件导入函数。被导入文件中 `_ENDFILE-IMPORT` 会排除尾部内容。

### LAMBDA — 匿名代码块
```
[]参数列表{ 函数体 }
```
`[]` 分隔参数列表，`{}` 分隔函数体。在文件级立即执行（非可调用函数）。
LAMBDA 拥有独立栈，退出时 PE 变量、锁、控件、HKEY 自动销毁。
`_SUB` 本质上是 LAMBDA；LAMBDA 的独特之处在于可访问调用者栈。
在命令群组（TEAM）内的 LAMBDA 函数体被解释为变量字符串，`%` 必须写为 `%%`。
```
// 示例：文件级内联块
[]P1 P2{ WRIT -,$+0,%P1% %P2% }

// 在 _SUB 内使用
_SUB MyFunc
    []%~1%{ MESS Hello %1! }
_END
```

### LOAD — 执行脚本文件
```
LOAD path\to\script.ini [args]
LOAD #101 [args]                           // 从 EXE 资源中执行内置脚本
LOAD --mem &var [args]                     // 执行变量中存储的代码
LOAD --Local --EnviMode path.ini           // 以 ForceLocal=1 + EnviMode=1 模式运行
LOAD --ncd path.ini                        // 不转移当前目录和环境变量（多线程安全）
LOAD --logs:[*]logFile path.ini            // 日志输出重定向（* = 同时输出到 stdout）
LOAD - path.ini                            // 不转移持久栈
LOAD -* path.ini                           // 不转移持久栈和执行栈
LOAD -del path.ini                         // 加载后删除该脚本
LOAD path\*.ini *FuncName args             // 调用脚本中的指定函数
```
加载优先级：PEI > #number > WCI/WCS/WCE/WCZ > EXE/COM/NTR/NTE/BAT/CMD > DLL。

### THREAD / THRD — 创建线程
```
THREAD[*][&][+][$][#] [-exp] [-wait[x][-here]] [-tid:var] [--st:stackSize] command
```
`*` = 立即执行，`&` = 强制 PE 变量模式，`$` = 预解释，`+` = 抛弃式线程，
`#` = 代理/线程模式（线程结束后代理退出），前缀 `&*+$` 无固定顺序。
`-wait` = 等待完成，`-tid:var` = 获取线程 ID

附加标志：
`-link` = 维护父子窗口连接，
`-waitp` = 线程前等待进程结束，
`-here` = 当前栈子线程（修改执行栈），
`-htid:var` = 获取线程句柄

---

## 变量与数据

### ENVI / SET — 设置/查询变量
```
ENVI (SET) [&][$][@]VarName=Value
ENVI^ EnviMode=1|ForceLocal=1|FORCELOCAL=1|LoadEnvi [...]
ENVI $var=val              // 设置环境变量
ENVI &var=val  (alias: SET var=val)   // 设置局部 PE 变量
SET &::var=val             // 设置类/全局 PE 变量
ENVI-def var=val  (alias: SET-def var=val)  // 仅当变量尚未定义时设置
ENVI-ret[level] %~1=%val%  // 按引用返回（默认 level=1）
ENVI~ &&Dst=Source.Key     // ~ = 间接展开
```
控件操作：
```
ENVI @Ctrl=Text                   // 设置控件文本
ENVI @Ctrl.Enable=0|1             // 禁用/启用
ENVI @Ctrl.Visible=0|1|*4        // 隐藏/显示/最小化
ENVI @Ctrl.POS=l:t:w:h            // 移动/调整大小
ENVI @Ctrl.POS=?&L:&T:&W:&H     // 查询位置
ENVI @Ctrl.Check=0|1|2|-1|-2       // 复选框状态（1/-1=选中，0/2/-2=未选，<0=灰色，±16=不可见）
ENVI @Ctrl.Val=data               // 设置内容
ENVI @Ctrl.Val=?row.col;&var      // 获取单元格（分号分隔）
ENVI @Ctrl.Val=?*;&count          // 获取行数
ENVI @Ctrl.Val=-*                 // 清除所有行
ENVI @Ctrl.Val=1*;%&data%         // 从变量批量设置
ENVI @Ctrl.Sel=idx|idx;0          // 选中/取消选中
ENVI @Ctrl.MSG=_msgId:cmd         // 消息映射（控件通知使用 _）
ENVI @Ctrl.POSTMSG=#msg;wp;lp     // 异步发送消息
ENVI @Ctrl.SENDMSG=#msg;wp;lp     // 同步发送消息
ENVI @Ctrl.*del=                  // 销毁控件
ENVI @Ctrl.ID=?&hwnd               // 获取控件 HWND
ENVI @Ctrl.Font=size:name
ENVI @Ctrl.bkcolor=0xRRGGBB
ENVI @Ctrl.Cursor=32649           // 手型光标
ENVI @@POS=wid:l:t:w:h:layer:trans:front:activate  // 设置窗口位置
ENVI @@POS=?wid:&L:&T:&W:&H:&SX:&SY::&Z  // 查询（含屏幕坐标和层叠序）
ENVI @@Visible=wid:0|1|*4        // 跨进程可见性（0=SW_HIDE, 1=SW_SHOW, 3=SW_MAXIMIZE, 4=SW_MINIMIZE 等）
ENVI @@Enable=wid:[#]0|1         // 跨进程禁用/启用（#=子线程）
ENVI @@Visible=?wid:&var         // 查询跨进程可见状态
ENVI @@Enable=?wid:&var          // 查询跨进程可用状态
ENVI @@IsWindow=?wid:&var        // 查询是否为有效窗口
ENVI @@SENDMSG=hwnd:#msg;wp;lp  // 跨进程同步发送消息（如 SendMessage）
ENVI @@POSTMSG=hwnd:#msg;wp;lp  // 跨进程异步投递消息（如 PostMessage）
ENVI @Win.Paint=funcName         // 画布回调（参数：HDC 宽 高）
ENVI @Win.HitTest=[-]h[:w:x:y]  // 拖动敏感区域（高=0取消，-=半透明穿透）
ENVI @Win.trans=0|1|0x2[*]      // 背景透明（0x1=透明模式，0x2=完全透明，*=透明色）
ENVI @Win.style=[@*]remove[:add] // 窗口风格操作（@=直接，*=扩展）
ENVI @Win.InvalidateRect=[左:上:右:下]|#WID|@SubName[~扩量][:擦除]  // 刷新区域
ENVI @Win.cmd[?]=var|cmd         // 动态设定/查询响应命令（?需 ENVI^ QueryCmd=1）
ENVI @Win.nxp=                   // 禁用 XP 视觉样式
ENVI^ Clipboard=text             // 写入剪贴板
ENVI^ Clipboard?=var             // 读取剪贴板到变量
ENVI^ EXPORTLOCAL=1|0|&1|&0      // PE 变量继承：1=继承，0=隔离（默认），&1=仅本级及以下继承，&0=仅本级及以下隔离
ENVI^ DisX64=1[,Old]             // 禁用 WOW64 文件系统重定向（Old=保存原状态用于恢复）
ENVI^ Arg=*                      // 将单词拆分为参数
ENVI @@DeskTopFresh=[clearicon][;][1|2|4|8|16][;[-+]path]  // 桌面刷新
ENVI @@TaskIcoMenu=0|1|2         // 托盘菜单切换
ENVI^ HelpColor=[*cmdHeight] [fgColor][#bgColor]  // HELP 显示颜色
ENVI^ Alias name=cmd              // 命令别名：替代命令前半部
ENVI^ Alias -opt name=cmd         // -opt 优化模式
ENVI^ WndProc[1|2|3][C][,ptr]    // Win32 回调绑定（C=C 调用约定）
ENVI^ memvar=[?返回名,][:字节数:]偏移,值  // 修改/查询 PECMD 内存变量
ENVI^ LoadPlugin=basename         // 加载插件
ENVI^ zero=0|1                    // 私密模式：内存用完清零
ENVI^ EnviBroad=0|1|-             // 环境变量广播开关：1=开启，0=关闭，-=后台
ENVI^ __arg=0|1                   // 兼容模式：启用 &&__arg 参数表
ENVI^ LoadEnvi [路径|-] [变量名]  // 从注册表刷新环境变量
ENVI^ QueryCmd=1                   // 启用 @控件.cmd?=变量名 动态命令查询
```

### ENVI ? — 系统查询
```
ENVI ?返回名=PPID,进程号           // 查询父进程 ID
ENVI ?返回名=ISADMIN               // 是否管理员（1/0）
ENVI ?返回名=ispe                  // 是否 PE 环境
ENVI ?字符串名,数字名=PEBIT,[path] // 查询位数（32/64）
ENVI ?返回名=WinVer[+][;...]       // Windows 版本信息
ENVI ?[$.]返名[,属性名]=FVAR,varName[;GUID]  // EFI 固件变量
ENVI ?[单个名],[全部名]=DROPFILE,wParam  // 拖放文件信息
ENVI ?文件版本名[,产品版本名][,2]=FVER,文件路径  // 文件版本查询（2=文件自身版本）
ENVI @@Cur=?[X名][;Y名]           // 鼠标位置查询
ENVI @@EATEKEYS=组合键1 ...        // 按键拦截
ENVI @@RMENU=变量名;文件名         // 获取文件右键菜单（多行，空行为分隔符）
```

### CALC — 计算/求值
```
CALC[-u|-txt|-cb-[Lfr:Lto[:Rfr:step]]] [-gui] [-base=[u]2|8|10|16|N] [-err=defaultValue] [#][变量=]表达式[#[#][小数位][E|F|G]]
```
`#` 前缀 = 整数模式。支持：`+ - * / % ^`，位运算 `& | @`，比较 `= <> > >= < <=`，
逻辑 `&& ||`。函数（共34个）：`abs sin cos tan ctg sqrt ln lg log pow exp pow10`，
`floor ceil round int frac div mod rand shl shr xor not lnot`，
`arcsin arccos arctan arcctg deg rad hypot max min`。
`lnot` = 逻辑非（`!a`），`not` = 按位非（`~a`）。常量：`e`，`pi`。
⚠ `log()` 需两参数 `log(底数,值)`，单参数返回 0；用 `lg()` 代替 log₁₀，`ln()` 为自然对数。
尺寸后缀：`K`=1024，`M`=1024^2，`G`=1024^3，`T`=1024^4，`S`=512。
结果加 `#` = 整数，结果加 `$` = 双精度（INT64/float）。
`-base=[u]2|8|10|16|N` — 输出进制（`u` = 无符号）。`-gui` — 图形界面计算器。
`-err=defaultValue` — 出错时返回默认值。
`-u` — 无符号输出修饰符（直接跟在 CALC 后面，无空格）。
`-txt` — 文本模式（直接跟在 CALC 后面）。
`-cb` — 剪贴板模式（直接跟在 CALC 后面）。
`-[Lfr:Lto[:Rfr:step]]` — 向量操作模式（`Lfr:Lto` 为左侧行从/到范围，`Rfr` 为右侧起始，`step` 为步长）。
`CALC -base=16 #&hex=shl(0x07,16)|0x20` — 十六进制位运算。
`CALC &sz=%&bytes%/1G#3` — 字节转 GB，3 位小数。
多个表达式：用 `;` 或换行分隔。子变量：`#subName` = 整数子变量，`$subName` = 浮点子变量。

### CODE — 编码转换
```
CODE -srcFmt,srcFile,-dstFmt,dstFile
CODE **-GBK,&src,**-UNI,&dst
```
格式：`-ANSI`，`-UNICODE`，`-UTF8`，`-UTF7`，`-GBK`，`-BIG5`，`-UNICODEB`，`-BOM`

### SET$ / ENVI$ — 从十六进制创建字符串
```
SET$ &Var=0d 0a                   // 变量 = CR+LF（Unicode 宽字符串）
SET$# &Buf=*4096 0                 // 分配零填充的原始字节缓冲区
ENVI$ &data=*1M 30 0d 0a          // 可变长度分配
```

### SET-def — 仅当未定义时设置
```
SET-def Var=DefaultValue
```

### SET-copy — 原始字节复制
```
SET-copy Dst=&src;srcOff;len;dstOff
```

### SET-long / SET-short / SET-ptr — 写入类型化值
```
SET-long &buf=value:offset         // 在缓冲区偏移处写入 32 位整数
SET-short &buf=value:offset        // 16 位
SET-ptr &buf=value:offset           // 指针大小
```

### SET?int / SET?longlong / SET?char — 读取类型化值
```
SET?int &buf=&&Var:offset
SET?longlong &buf=&&Var:offset
SET?char &buf=&&Var:offset
SET?short &buf=&&Var:offset              // 读取 SHORT
SET?ptr &buf=&&Var:offset                // 读取指针大小值
```

### ENVI-* — 内存基元（二进制数据操作）
```
ENVI-mkfixdummy &&var=源变量@偏移       // 创建固定虚拟变量（内存不可变）
ENVI-mkdummy &var=地址;字节数            // 从地址创建虚拟变量
ENVI-addr &&ptr=源变量                   // 获取变量内存地址和长度
ENVI-long &buf=value:offset              // 写入 LONG (32位)
ENVI?long &var=源变量:offset             // 读取 LONG
ENVI?short &var=源变量:offset            // 读取 SHORT
ENVI?char &var=源变量:offset             // 读取 CHAR
ENVI?int64 &var=源变量:offset            // 读取 INT64
ENVI?ptr &var=源变量:offset              // 读取指针
```

### struct — C 风格结构体定义
```
struct 结构体名
{
    __virtual 返回类型 STDMETHODCALLTYPE 方法名(参数列表);
    成员类型 成员名;
    ...
};
typedef 类型别名 原类型;
```
配合 `SET-*结构体 变量.成员=值` 和 `SET?*结构体 变量.成员=&目标` 读写。

### SET-make / ENVI-make — 从缓冲区取子串
```
SET-make &&Str=&buf@offset;$length           // $ 固定长度
SET-make &&Str=&buf@offset;(%expr%*2)        // ;(...) 计算表达式作为长度（⚠语法待确认）
```

### SET< / ENVI< — 追加到变量
```
SET< Var=text to append
```

### ENVI-addr — 获取缓冲区地址
```
ENVI-addr &&ptr=&buf                // 获取地址（单返回值）
ENVI-addr ;&len=&buf                // 获取地址和字节长度（双返回值，分号分隔）
```

### ENVI-mkdummy — 虚拟指针/长度描述符
```
ENVI-mkdummy &&Name=&buf@offset;length
```

### SET-cmp — 二进制比较
```
SET-cmp dst=src;srcOff;len;dstOff;[S|s|I|i]
```
S/I = 宽字符，s/i = 窄字符，I/i = 不区分大小写。

### SET-tom / SET-tow — 编码转换
```
SET-tom dst=src       // UNICODE 转多字节（如 GBK）
SET-tow dst=src       // 多字节转 UNICODE
```

### SET-swap — 交换变量内容
```
SET-swap var1=var2
```

### SET-zero — 清除变量内存
```
SET-zero var=[value][@offset][;count]
```
`$` 前缀用于宽字符模式。

### ENVI-ex — 检查变量是否存在
```
ENVI-ex retVar=varName
```

### ENVI-tom — 字符串转指针
```
ENVI-tom &&dst=&src    // 将字符串转换为内存指针
```

### SET-mkfixdummy — 固定虚拟变量（内存不可变）
```
SET-mkfixdummy PE变量名=[地址][;[*][字节数]]
```
同 `SET-mkdummy`，但所引用的内存不会变动（fixed dummy）。

### SET-addr — 获取地址和长度
```
SET-addr [地址名][;长度名]=源PE变量名
```
返回源 PE 变量的内存地址和字节长度到指定变量。

### SET-*结构体 / SET?*结构体 — 结构体读写
```
SET-*结构体 PE变量名.成员=数值[:[~]附加总偏移字节数]         // 写入结构体成员
SET?*结构体[:0[@]s] 源PE变量名.成员=变量名[:[~]附加总偏移字节数]  // 读取结构体成员
```
`0`=补零，`@`=去掉 0x 前缀，`s`=带符号。`~`=偏移以类型大小为单位。
配合 `struct` 块定义的 C 风格结构体使用。

### ENVI 后缀变体 — 临时模式切换
```
ENVI -env ...                     // 临时取消 forceLocal（必须为第一个后缀）
ENVI -std ...                     // 临时设置 EnviMode=1（必须为第一个后缀）
ENVI -raw ...                     // 等号右侧不解释
ENVI -get[N] ...                  // 回溯 N 级（默认 1）获取 PE 变量
ENVI -ret[N] ...                  // 回溯 N 级（默认 1）操作 PE 变量名
SET-env ...                       // 等价于 ENVI -env &...（SET 也可用 -env 后缀）
```
`-env` 和 `-std` 便于操作环境变量；`-get` 用于函数传入 PE 变量名时获取；`-ret` 用于函数返回时操作。
`SET-env` 临时取消 ForceLocal，用于 SET 命令需要读写环境变量的场景。

### 析构函数语法
```
ENVI ~析构函数~变量名=初始值       // 定义时指定析构函数，退出变量定义范围时自动调用
// 调用形式：析构函数 变量引用 变量值
_SUB 函数名,*,,析构命令            // _SUB 也支持析构命令（返回前自动执行）
```

---

## 流程控制

### FIND — 字符串比较
```
FIND $str1=str2, command        // 等于（不区分大小写，*c 后缀区分）
FIND $str1<>str2, command       // 不等于
FIND $=%var%, command           // 变量为空
FIND $%var%=, command           // 变量为空（同上，变量在左侧）
FIND *=var, command              // 惯用法：变量为空（主要源模式）
FIND *<>var, command             // 惯用法：变量非空
FIND $str1=str2,! cmd1! cmd2   // if else（用 ! 分隔）
FIND $str1=str2,!! command      // 仅 else
FIND |num1>num2, command        // 数值比较
FIND $'%var%'='', command       // 安全的空值检查（单引号保护）
FIND [$][A & B], command         // 复合 AND（条件之间用 &）
FIND [$][A | B], command         // 复合 OR（条件之间用 |）
FIND --pid &var,ProcessName            // 获取进程 PID
FIND --pid &var                        // 返回 5 个值：空闲时间 总时间 CPU个数 1秒时钟数 一时钟100ns数
FIND --pid*@[.ext|#parentPID] &var,    // 进程列表（可选：扩展名过滤或父进程 PID）
FIND [--user] --pid*@[.ext|#parentPID] &var,prog[|用户名]  // 按用户名过滤进程
FIND [--sub][--forpid:PID|--fortid:TID] --wid*@[parentWID] &var,[title]   // 窗口列表（* = 标题前缀匹配）
                                         // --sub=递归子窗口，--forpid=按进程过滤，--fortid=按线程过滤
FIND --wid#ParentWID &var,ControlID    // 查询控件的窗口 ID
FIND --class:ClassName --wid*@ &var    // 按窗口类名过滤窗口列表
FIND --menu &var,WindowID              // 查询窗口的 MENU 句柄
FIND --menu#Index &var,MenuID          // 按索引查询子 MENU
FIND $!=%var%,                         // 与字面量 "!" 比较（特殊：$ 后跟比较操作符）
FIND C:\=?,&var                        // 查询磁盘总空间（字节）
FIND MEMB<比较符>数值, command            // 字节级内存比较（单位：字节）
FIND R:\<比较符>数值, command             // 磁盘空间比较（R: 为盘符，单位 MB）
FIND R:\<比较符>*数值, command            // 磁盘空间比较（* = 字节单位）
```

**FIND --pid 输出字段（@=列表模式）：**
`进程ID  父进程ID  内存K  CPU使用时间(100ns)  总时间  [用户]  文件名  命令行`
一行一条，以 TAB 间隔。0 进程的 "CPU 使用时间" 为系统 "空闲时间"。

### IFEX — 文件测试 / 数值比较 / 系统查询
> **注意**：IFEX 的 `$`/`|` 前缀与 FIND **相反**。IFEX: `$`=数值比较，`|`=字符串比较。
```
IFEX path\|file, command         // 文件存在
IFEX path\|file,! command        // 不存在
IFEX x:\, command                // 盘符存在且有文件系统
IFEX $num1>=num2, command        // 数值比较
IFEX #num1=#num2, command        // 强制整数
IFEX [ cond1 & cond2 ], command  // AND 复合（条件间用 &、| 或 @）
IFEX [ cond1 | cond2 ], command  // OR 复合
IFEX [ cond1 @ cond2 ], command  // XOR 复合
IFEX MEMU=?,&var                 // 查询可用内存（单位 MB）
IFEX MEMA=?,&var                 // 查询总内存（单位 MB）
IFEX MEMBU=?,&var                // 查询可用内存（单位字节）
IFEX MEMBA=?,&var                // 查询总内存（单位字节）
IFEX drv:\=?,&var               // 查询磁盘可用空间
IFEX KEY=?                       // 等待按键
```

> **复合条件类型前缀：** `[前的$` 或 `[前的|` 表示后续所有条件默认为 `$`（数值）或 `|`（字符串）比较，可省略逐个标注。

### LOOP — While 循环
```
LOOP [#]condition,
{
    // 循环体
}
// BREAK：EXIT LOOP 或 EXIT BREAK
// CONTINUE：EXIT CONTINUE
```

### FORX — 迭代
```
FORX * list,&&item,                              // 空格分隔迭代
FORX *NL &multiLine,&&line,                      // 换行分隔
FORX *NL:| &data,&&item,                         // 自定义分隔符（|为分隔符）
FORX *v &a &b &c,&&name,                          // 迭代变量名
FORX /S[:depth] path\*.ext,&&name,0               // 文件枚举（0=文件，1=目录）
FORX /S:3 /O:N path\*.ext,&&f,0                  // 最大深度 3，按名称排序
FORX /S /O:-N path\*.ext,&&f,0                   // 深度不限，反向排序
FORX /S /size:0:1048576:512 path\*.ext,&&f,0     // 大小 0-1MB，512 对齐（已分配大小）
FORX /S /size*:0:1048576:512 path\*.ext,&&f,0    // 同上（* = 实际文件大小，非已分配大小）
FORX @\Windows,&&dir,1                            // 搜索目录根
FORX !\*.ext,&&f,0                                // 反向目录顺序
FORX @\*.ext,&&d,1                                // 仅目录（@ 前缀）
FORX *ab \*.ext,&&f,0                             // 排除 A/B 可移动驱动器
FORX *cur \*.ext,&&f,0                            // 搜索时优先当前驱动器
FORX *qu[~] \*.ext,&&name,0                       // 支持路径中的引号
FORX *off \*.ext,&&name,0                         // 仅返回变化部分
FORX *bf \*.ext,&&name,0                          // 广度优先目录搜索
FORX *L start step end,&&val,                     // 数值循环：FORX *L 0 2 10,&&val,
FORX . \*.ext,&&f,0                               // ; 可替代 , 作为分隔符
FORX : \*.ext,&&f,0                               // : 可替代 , 作为分隔符
```

### TEAM — 多命令
```
TEAM cmd1 | cmd2 | cmd3 ...
```
嵌套分隔符：`|`（第1层），`||`（第2层），`|||`（第3层）。

### LOCK — 临界区 / 互斥锁
```
LOCK #lockName,&retVar           // 创建/获取命名锁
LOCK --exist #lockName,&retVar   // 检查锁是否存在（1=是，0=否）
{ LOCK #pecmd ... }              // 用花括号包裹以创建原子作用域
```

---

## 文件 I/O

### READ — 读取文件
```
READ[-UNICODE|-UNICODEB|-UTF8|-GBK|-BIG5|-ANSI|-<codepage>] [*fix] path,*r,&var     // 原始（不转换行尾符）
READ path,*,&var      // UNIX LF -> 本地
READ path,**,&var     // DOS CRLF -> 本地
READ -,-1,&count,&var // 获取行数
READ -,lineNo,&line,&var  // 读取指定行（lineNo=-1 获取行数）
READ -,10,&line,&var  // 从 stdin 读取最多 10 字节
READ -*[?],lineNo,&line,&var  // 从变量读取（? = 测试/查询编码）
```
编码缩写：`-UNI` = `-UNICODE`，`-UNIBE` = `-UNICODEB`。
`*[?]` 变量名后缀：用于测试编码（`?` 探测变量内容的编码格式）。

### WRIT — 写入文件
```
WRIT[-UNICODE|-UNICODEB|-UTF8|-GBK|-BIG5|-ANSI|-<codepage>] [*fix] [*-nl] [*v] [*fv] [*c] [*nobom]
    path,[$][+|-]lineID,text
```
编码标志（在文件名之前）：`-UNICODE`=带 BOM 的 UTF-16LE，`-UNICODEB`=UTF-16BE，`-UTF8`=带 BOM 的 UTF-8，`-GBK`，`-BIG5`，`-ANSI`，`-<code_number>`=指定代码页。当现有文件有 BOM 时，BOM 优先。
缩写：`-UNI` = `-UNICODE`，`-UNIBE` = `-UNICODEB`。

星号前缀修饰符：`*fix`=单独的 CR 视为换行，`*-nl`=不添加尾部换行，`*v`=写入变量，`*fv`=FileData 是变量名，`*c`=先清空文件，`*nobom`=写入时不添加 BOM。

位置：`$`=展开环境变量，`+`=插入新行，`-`=删除行，纯数字=替换行。`0` = 最后一行。

**标准 I/O（特殊文件名）：**
| 文件名 | 含义 |
|--------|------|
| `-` | stdin（READ）/ stdout（WRIT）|
| `--` | stderr |
| `CONOUT$` | 调试终端 |

```
READ -,10,&line,&var     // 从 stdin 读取一行
WRIT -,$+0,Hello         // 写入 stdout
WRIT --,$+0,Error        // 写入 stderr
WRIT C:\BOOT.INI,+0,text              // 追加新行
WRIT path,$0,a=%var%                  // 替换最后一行，展开变量
WRIT -,$+0,result                     // 写入 stdout
WRIT path,$-3,                        // 删除第 3 行
```

### GETF — 二进制文件读取
```
GETF# path,offset#size,&var       // 从偏移读取原始字节，共 size 字节
GETF# path,0#*,&var               // 读取整个文件
GETF -bin &src,offset#size,&hexOut // 读取为十六进制字符串输出
```

### PUTF — 二进制文件写入
```
PUTF -dd -len=0 path,0,zero       // 创建/截断
PUTF path,offset,#&data            // 在偏移处写入
PUTF -dd -bs=1M src,0,dst,0,-len=size  // 磁盘到磁盘复制
```

### FILE — 文件/目录操作
```
FILE src -> dst               // 复制
FILE -force src -> dst        // 强制覆盖
FILE src                      // 删除
FILE -r dir                   // 递归删除
FILE -md dir                  // 创建目录
FILE -simpleprogress src -> dst  // 带进度条
```

### DIR — 列出目录
```
DIR &var /s /b path            // 获取目录列表到变量
```

### FDIR / FEXT / FNAM / NAME — 路径各部分
```
FDIR &var=fullPath              // 目录部分
FEXT &var=fullPath              // 扩展名（如 "EXE"）
FNAM &var=fullPath              // 带扩展名的文件名
NAME &var=fullPath              // 不带扩展名的文件名
```

### SIZE — 文件大小
```
SIZE &var=filePath
```

### HASH — 计算哈希
```
HASH filePath,&var,MD5|SHA1|SHA256|CRC32
HASH $string,&var,SHA1          // 对字符串内容计算哈希
HASH *PEvarName,&var,SHA256     // 对 PE 变量内容直接计算哈希（`*` 可省略如果变量有 `&` 前缀）
```
默认算法：MD5。不指定变量时，在消息框中显示结果并复制到剪贴板。

### MDIR — 创建目录
```
MDIR dirPath
```

### FLNK — 符号链接/硬链接
```
FLNK targetPath,sourcePath             // 硬链接（默认，类型=0）
FLNK targetPath,sourcePath,1           // 符号链接（类型=1）
FLNK -j targetPath,sourcePath          // 目录联接
FLNK targetPath,                       // 删除链接（源为空）
```

---

## 磁盘与分区

### PART — 分区管理（全面）
```
// 列出/信息操作
PART list disk,&var                         // 列出所有磁盘号
PART list disk N,&var                       // 磁盘信息（大小、柱面、磁头、介质类型、签名、总线、类型、可移动）
PART list part N,&var                       // 列出磁盘 N 上的分区号
PART -hextp list part N#M,&var              // 分区信息（十六进制类型 0xNN）
PART -hextp -phy list part N#M,&var         // 分区信息，含物理编号（1-4 主分区，5-N 逻辑分区）
PART -hextp -phy# list part N#M,&var        // 分区信息 + 追加物理#字段
PART -fill list part N#M,&var               // 空盘符用 * 占位
PART -devid list disk N,&var                // 设备路径/ID（如 \\.\PHYSICALDRIVE0）
PART -devidx list disk N,&var               // 型号 + 序列号
PART -devidn list disk N,&var               // 仅名称
PART -devida list disk N,&var               // 完整：产品号 + 序列号 + 版本 + 设备类型 + 可移动介质 + 命令队列 + 供应商ID + 产品修订版
PART -iv=N list disk N,&var                 // 查询磁盘信息的第 N 个子字段
PART -raw list disk N,&var                  // 原始磁盘信息（设备路径、介质 GUID、卷名）
PART list drv D:,&var                       // 盘符 → 磁盘号 分区号 类型 总线 驱动器 介质
PART -raw list drv D:,&var                  // 原始盘符信息：设备号 分区号 盘符类型 总线 盘符 媒体类型
PART list volume volumeName,&var            // 卷信息
PART -drv list volume N,&var                // 按驱动器号列出卷
PART -phy list volume N,&var                // 卷信息（含物理设备号/分区号）
PART -report[:retvar][diskNum]              // 显示/列出报告（忽略其他参数）
PART -floppy list disk N,&var               // 列出软盘设备
PART [-cdrom|-floppy] list parent <devOrDrv>,&var  // 列出父设备
PART [-cdrom] list dep <devOrDrv>,&var      // 列出依赖/源文件名
PART [-devid[x|n|a]] list cdrom [N],&var    // 列出 CDROM 设备

// 修改操作
PART -super -up -xup N#M type [attr]        // 设置分区类型+属性（-super 和 -up 均需指定）
PART -super -up -axup N#M type [attr]       // 可移动磁盘的增强 xupdate
PART -super -up -swap:M N#P                 // 交换物理分区号
PART -super -up N#M a|A|-a|-A type start len// 创建（a=活动，A=扩展，-a=非活动）
PART -super -up del N#M                     // 删除分区
PART -super -up N#M a|A|-a|-A               // 切换现有分区的活动标志
PART -super -up -fs0 N#M init               // 初始化为原始状态（无文件系统）
PART -super -up -force N#M ...              // 强制执行危险操作
PART update N                               // 从系统刷新磁盘信息
PART hupdate[f] N                           // 硬盘刷新（f=强制重新编号）
PART -ahup -up N#M ...                      // 可移动磁盘重新编号的额外硬更新

// MBR/PBR 操作
PART /mbr[=nt6|=win|=nt5|=dos|=file] N      // 重写磁盘 N 上的 MBR
PART /pbr[=nt6|=win|=nt5|=dos|=file] N#M    // 重写分区上的 PBR
PART -img=[*offs*len*]file|disk[/mbr|/pbr]  // 对镜像文件而非物理磁盘操作

// GPT 操作
PART -gpt init N                            // 初始化为 GPT
PART -super -up -gpt N#M a type start len guid attr name  // 创建 GPT 分区
PART -super -up -gpt -fs0 -mbr init N       // 初始化 GPT+MBR 混合，原始文件系统
PART -gpt -cmp N                            // 压缩 GPT 表（从1开始编号，连续排列）
PART fix N                                  // 修复 GPT：纠正校验和、标志、分区计数

GPT 属性：
| 值 | 含义 |
|---|---|
| `0x1000000000000000` | 只读 |
| `0x2000000000000000` | 影子 |
| `0x4000000000000000` | 隐藏 |
| `0x8000000000000000` | 无盘符 |
| `0x0000000000000001` | 计算机必须的分区 |

// 智能盘符控制
PART -lock[:\\\\.\D:] N                     // 锁定盘符（阻止自动分配）
PART -locku[:\\\\.\D:] N                    // 解锁
PART -lock *                                // 锁定所有卷
PART -dvol N#M,&volGUID                     // 动态校正的 VolumeGUID
PART -mount-                                // 不显示未分配分区的标签
PART -fill                                  // 填充空盘符槽位

// 实用工具
PART -gui                                   // 启动 GUI 分区管理器
PART -usb                                   // 仅 USB 模式
PART -admin                                 // 高级模式（危险）
PART -align[=size]                          // 对齐（默认或指定值）
PART -CHS=C:H:S                             // 覆盖柱面/磁头/扇区几何参数
PART -clear                                 // 强制清除分区内有效信息（不可恢复）
```

PART MBR 输出字段：`分区号 类型(hex) 激活 起始(字节) 长度(字节) 隐藏扇区 结束(字节) 物理# 盘符`
PART GPT 输出字段：`分区号 GUID 属性 起始(字节) 长度(字节) 结束(字节) 物理# 盘符`

### MOUN — WIM/VHD/UDM 挂载
```
// WIM 挂载
MOUN [-svr] WimPath,MountDir,[ImageID],[TempDir]  // 只读挂载 WIM（ID=1 可省略）
MOUN -u MountDir                                   // 卸载 WIM
MOUN -query VarName[=rw],MountDir,WimPath          // 查询挂载状态（=rw 仅返回 RW 标志）

// VHD 挂载
MOUN-vhd [-c[x]|-d|-u|-r|-s:sectsz] VHDPath,MountDir|Size|ParentVHD,[ID],[RetVar][,PEVar]
// -c 创建 VHD（-cx 创建 VHDX），-r 只读，-d 动态，-iso:ISO 模式
MOUN-vhd -query [-r] VHDPath,BufPEVar[,SizeVar][,cmd]  // GET_VIRTUAL_DISK_INFO 查询

// UDM 挂载子命令
MOUN-udm [-ud|-uh|-muh[g]] [-u+] [-udfs] [-udm-] [-w] [-m] [-mall] [-mhide[1]]
    [-findboot[Only]] [-CurDrv[R][+]] [-udmid:pt#] [-udmask:掩码]
    [-udimg:文件] [-check[-]] [-ret:返名] 设备名 [盘符表]   // 主挂载命令
MOUN-udm listudm -ret:返名 设备名 [UD通配符]         // 列出 UDM 分区
MOUN-udm listud -ret:返名 [-udmask:掩码] 设备名 [通配符]  // 详细 UD 文件列表
MOUN-udm findboot -ret:返名                           // 查找启动设备
MOUN-udm findudm [-img] [-norm] -ret:返名 盘符        // 查找对应 UDM
MOUN-udm setboot -ret:返名 启动菜单 [UDM盘符|#udm号] [类型] [自动加载盘符]
MOUN-udm sync "盘符列表"                               // 刷新数据到存储体
MOUN-udm mapsub [-check] [-r] 文件名 盘符             // 只读 UDm 盘的可写加载
MOUN-udm ud2fs [-efi] 设备名 [bClr=1] [bMkNew=1] [FS=FAT]  // UD 扩展区转文件系统
MOUN-udm OnlyApp [-noauto]                             // 检测并执行一键恢复
MOUN-udm Server [-FreshDriver] [-quit] [-safe]         // UDM 自动挂载服务
MOUN-udm setbootcfg 启动菜单 "值" ["标签头"]           // 设置启动配置
MOUN-udm getbootcfg [-x[+|a]] 返回名 标签头           // 获取启动配置
```

UDM 掩码（`-udmask`）：`0x20000`=UD 扩展区，`0x40000`=仅 UD 扩展区，`0x80000`=检查 UD 扩展区，`0xA0001`=组合。

### SHOW — 显示/隐藏分区
```
SHOW -1:-1                                // 显示所有分区
SHOW * hd:part,driveLetter                 // 分配盘符（hd=磁盘，part=分区）
SHOW *- hd:part,                           // 移除盘符
SHOW & hd:part,driveLetter                 // 本地模式分配
SHOW =1 * hd:part,driveLetter              // 已加载则跳过
SHOW -check * hd:part,driveLetter          // 无有效文件系统则跳过
SHOW * F:,driveLetter                      // 固定磁盘
SHOW * U:,driveLetter                      // USB 磁盘
SHOW * #physicalPart,driveLetter            // 物理分区号
SHOW * -1,driveLetter                       // 所有未分配盘符的分区
SHOW * hd:part,ChineseChar                  // 分配中文字符盘符
SHOW * hd:part,letter,WaitMs               // 分配并等待设备就绪（等待毫秒数）
```

> **help.txt 标准语法**: `SHOW [=1] [-SKIP=类型] [-check] [-skiptp:tp1;tp2] [-skippt:hd1:lpt1;hd1:lpt2] [-from:盘符[表]] [*&-] [磁盘分区],[盘符[表]],[等待时间],[起始盘符[表]]`

### SUBJ — 挂载/卸载
```
SUBJ D:,\Device\Harddisk0\Partition1       // 挂载
SUBJ -D:                                   // 卸载
```

### FDRV — 驱动器枚举
```
FDRV &var=*:                               // 所有有卷的盘符
FDRV *idle &var=*:                         // 空闲（未分配）盘符
FDRV *vol &label,&fs=D:                   // 获取卷标和文件系统
FDRV *rsort &var=*:                        // 反向排序

// 返回格式变体
FDRV &var=                                 // 返回 C:|D:|E:|F:...（管道分隔）
FDRV &var=*                                // 返回 C D E F...（空格分隔，无冒号）
FDRV &var=*:                               // 返回 C: D: E: F:...（空格分隔，带冒号）
FDRV &var=?                                // 返回所有 DOS 设备名

// 卷标操作
FDRV -vol [卷标名],[文件系统名],[序列号名],[最大文件名长度名],[标志名],[UUID名]=驱动器名
FDRV -setvol 驱动器名=卷标                 // 设置卷标

// 排序与过滤
FDRV -ab &var=*:                           // 排除 A/B 软盘驱动器
FDRV -idle[c][:盘符集] &var               // 空闲盘符（-idlec 排除 A/B）
FDRV -link? 返名,所有名,终极名=符号名      // 返回链接对象
```

### FORM — 驱动器类型（全面）
```
// 基本文件系统查询
FORM &var=D:                               // 文件系统类型字符串（如 "NTFS"、"FAT32"、"CDFS"）
FORM -raw &var=D:                          // 驱动器类型常量（见下表）
FORM TYPE,&var,BUS=D:                      // 总线类型字符串（如 "USB"、"SATA"、"SCSI"、"NVMe"）
FORM -raw &type,&bus,&drvType=&dsk,<drive>  // 全面：一次调用获取所有类型信息

// 驱动器类型常量（FORM -raw 返回）
```
| 常量 | 值 | 描述 |
|---|---|---|
| DRIVE_UNKNOWN | 0 | 未知驱动器类型 |
| DRIVE_NO_ROOT_DIR | 1 | 无效/未挂载 |
| DRIVE_REMOVABLE | 2 | 可移动介质（U盘、软盘） |
| DRIVE_FIXED | 3 | 固定磁盘（HDD、SSD） |
| DRIVE_REMOTE | 4 | 网络/映射驱动器 |
| DRIVE_CDROM | 5 | 光盘（CD/DVD/BD） |
| DRIVE_RAMDISK | 6 | RAM 磁盘 |
| DRIVE_CDROMUSB | 7 | USB 光盘 |
| DRIVE_USBFLASH | 8+ | USB 闪存驱动器 |
| DRIVE_USBDISK | 9+ | USB 磁盘 |
| FUNCTION_ERROR | -1 | API 错误 |

### 总线类型常量（FORM -raw &busType,&bus,drvType=&id,D:）
| 常量 | 值 | 描述 |
|---|---|---|
| BusTypeUnknown | 0x00 | 未知 |
| BusTypeScsi | 0x01 | SCSI |
| BusTypeAtapi | 0x02 | ATAPI |
| BusTypeAta | 0x03 | ATA |
| BusType1394 | 0x04 | 1394 (FireWire) |
| BusTypeSsa | 0x05 | SSA |
| BusTypeFibre | 0x06 | 光纤通道 |
| BusTypeUsb | 0x07 | USB |
| BusTypeRAID | 0x08 | RAID |
| BusTypeiScsi | 0x09 | iSCSI |
| BusTypeSas | 0x0A | SAS |
| BusTypeSata | 0x0B | SATA |
| BusTypeSd | 0x0C | SD |
| BusTypeMmc | 0x0D | MMC |
| BusTypeVirtual | 0x0E | 虚拟 |
| BusTypeFileBackedVirtual | 0x0F | 文件支持虚拟 |
| BusTypeSpaces | 0x10 | 存储空间 |
| BusTypeNvme | 0x11 | NVMe |
| BusTypeSCM | 0x12 | SCM |
| BusTypeUfs | 0x13 | UFS |
| BusTypeMax | 0x14 | |
| BusTypeMaxReserved | 0x7F | 保留最大值 |

### DFMT — 格式化
```
DFMT d:,NTFS,label,quick
DFMT d:,FAT32,,quick
```

### EJEC — 弹出
```
EJEC D:                       // 弹出光驱 D:
EJEC * D:                     // 弹出可移动 USB 磁盘 D:
EJEC C-                       // 关闭所有光驱托盘
EJEC U-                       // 弹出所有 USB 磁盘
EJEC C- X:                    // 关闭 X: 的托盘
EJEC U- X:                    // 弹出 USB 磁盘 X:
EJEC C- HDD#1                 // 关闭磁盘 1 的托盘
EJEC U- HDD#1                 // 弹出磁盘 1 上的 USB 磁盘
```

### DISK — 磁盘操作
```
DISK [varName],[diskNum],[partNum],function,[USBDriveLetters][,options]
  // function 1=分配，2=释放，3=重新分配，22=第一个主分区
  // USBDriveLetters：例如 "UW" 表示 USB 从 W: 开始分配（盘符表）
```
| 功能 | 描述 |
|---|---|
| 1 | 分配盘符 |
| 2 | 释放盘符 |
| 3 | 重新分配（重排 + 分配） |
| 22 | 第一个主分区分配 |
| **varName 特殊形式：** | |
| `&drvLetter,diskNum,partNum` | 获取指定分区的盘符 |
| `uAllPart,diskNum,partNum` | 分配 USB → 新分区盘符 |
| `Vol:volLabel,diskNum,partNum` | 按卷标查找 |
| `Part:partName,diskNum,partNum` | 按分区名查找 |
| `\Windows\|\WinXP\|\WinNT\|` | 跨驱动器搜索系统目录 |
| **选项（0x**）：** ||
| 0x1 | 仅重排已有盘符的分区 |
| 0x2 | 验证分区有效性 |
| 0x4 | 跳过 0xEE/0xEF 分区 |
| 0x10 | 也包含隐藏分区 |
| 0x20 | 也包含 CDROM |
| 0x40 | 限制盘符表 |
| **标志：** ||
| `-check` | 已加载则跳过 |
| `-skiptp:tp1;tp2` | 跳过分区类型 |
| `-skippt:hd:pt` | 跳过指定 disk:partition |
| `-from:D:` | 从 D: 开始分配盘符 |
| `-from:UW` | USB 盘符表 "UW" |
| `-cdrom` | 包含 CDROM |

---

## 挂载 (WIM / VHD / UDM)

### WIM 挂载
```
MOUN[-svr] [!] wimFile,mountDir,[imageID],[tempDir]    // 挂载（读写）
MOUN[-svr] -w [!] wimFile,mountDir,[imageID],[tempDir]  // 挂载为可写
MOUN[-svr] -m [!] wimFile,mountDir,[imageID],[tempDir]  // 挂载为只读（无 -w）
MOUN -u mountDir                                         // 卸载
MOUN -query &var                                        // 查询已挂载镜像
MOUN[-svr] -u [!] wimFile,mountDir,[imageID],[tempDir]   // 卸载并提交
MOUN[-svr] -rw [!] wimFile,mountDir,...                  // 挂载为读写（别名）
```
选项：`-dll WIMDLLpath:` 指定 wimgapi.dll 位置。

### VHD/VHDX 挂载
```
MOUN-vhd -c[x] file.vhd,size                            // 创建（x=先扩展到指定大小）
MOUN-vhd -c[x] -d file.vhd,size                         // 创建动态（稀疏）
MOUN-vhd -c[x] -s:512 file.vhd,size                     // 扇区大小覆盖
MOUN-vhd -r file.vhd,mountDir                           // 挂载为只读
MOUN-vhd -d file.vhd,mountDir                           // 挂载动态 VHD
MOUN-vhd -u mountDir                                    // 卸载
MOUN-vhd -iso file.iso,mountDir                         // 挂载 ISO
MOUN-vhd -query file.vhd,&var                          // 查询 VHD 信息
```
PECMD 私有 PE 变量：当变量超出作用域时自动卸载。使用 `PEvar` 作为第 3 个参数。

### UDM（超深度挂载）— 隐藏分区挂载
```
MOUN-udm [flags] \\\\.PhysicalDriveN                     // 挂载全部或指定
MOUN-udm -findboot -ret:&retVar                         // 查找并挂载引导设备
MOUN-udm -u mountDir                                    // 卸载
```
| 标志 | 描述 |
|---|---|
| `-ud` | UD 分区 |
| `-uh` | UD 高端 |
| `-muh` | 挂载 UD 高端 |
| `-u+` | U+ 分区 |
| `-udfs` | UD 文件系统 |
| `-udm-` | 禁用 UDM |
| `-mall` | 挂载全部（不仅是隐藏的） |
| `-mhide` | 仅挂载隐藏分区 |
| `-mhide1` | 仅挂载隐藏分区（变体 1） |
| `-onlys` | 仅挂载特定系统类型 |
| `-findboot` | 自动查找引导设备 |
| `-ret:` | 返回设备路径到变量 |
| `-CheckFile[+]:path` | 按文件存在验证 |
| `-CheckVol[R]` | 按卷标验证 |
| `-CheckUuid[R]` | 按 UUID 验证 |
| `-CheckPtType` | 按分区类型验证 |
| `-check[-]` | 仅挂载有效文件系统分区 |
| `-tag[+]:name` | 标签标识用于匹配 |
| `-opts:`/`-opt:` | 挂载选项（分隔或合并） |
| `-nbrd[-]` | 不广播盘符 |
| `-ainf:var` | 存储分区表缓冲区到变量 |
| `-udmid:pt#physicalNum` | 按物理分区号软挂载（默认只读） |
| `-udmdev:device` | 指定引导设备和 UDM |

---

## 系统

### MAIN — WinPE 入口点
```
MAIN path\to\PECMD.INI
```
启动桌面，挂钩 Ctrl+Alt+Del，运行配置文件，并进入消息循环。这是标准的 PE 引导入口命令。

### INIT — 初始化
```
INIT [options],[timeout],[USB起始盘符]
```
选项：`C`=将光驱盘符写入环境变量，`I`=安装托盘图标菜单，`K`=立即安装低级键盘钩子，`U`=USB 移动硬盘即插即用
常用：`INIT IU,3000`（USB 起始盘符默认为 U）

### SHEL — 设置 Windows 外壳
```
SHEL %SystemRoot%\explorer.exe
SHEL PECMD.EXE LOAD MyShell.ini
    cmd_on_shell_change
```
缩进的行在外壳切换时执行。

### DISP — 显示设置
```
DISP W1024H768B32F60                             // 宽，高，色深，刷新率
DISP                                             // 自动检测最佳模式
DISP =N W1024H768B32F60                         // 目标显示器 N（从 0 开始）
DISP W1024H768B32F60 T15                         // 应用并设 15 秒超时（自动恢复）
DISP W1024H768B32F60 P                           // 设为主显示器
DISP W1024H768B32F60 O0                          // 方向（0=默认，1=90°, 2=180°, 3=270°）
DISP -confirm W1024H768B32F60                    // 确认提示
DISP -nwb W1024H768B32F60                        // 不等待广播
DISP -delay W1024H768B32F60                      // 仅写注册表（不应用），等待广播
DISP @X0:Y0:X1:Y1:...                            // 多显示器位置（矩阵）
DISP S0x84                                        // 多显示器模式：0x81=单屏，0x82=克隆，0x84=扩展，0x88=双屏
DISP ?[?*] [=N] &var                             // 查询当前（*=所有可能）模式
DISP -reset                                       // 重置为默认值
DISP -bright[?]:value/&var                        // 亮度控制
DISP -ori [?] &var                                // 查询方向
DISP -guis                                        // 图形界面
DISP -sort[-r|-n]                                 // 排序模式（r=反向，n=按名称）
```

### PAGE — 虚拟内存
```
PAGE [*force] C:\pagefile.sys 256 512   // 最小 256MB，最大 512MB（*force 强制创建）
```

### RAMD — RAM 磁盘（ImDisk）
```
RAMD ImDisk,L100,FAT32,C:,MyRam      // 创建
RAMD ImDisk* -D -m G:                 // 移除
```

### SERV — 服务管理
```
SERV [-wait] servicename                     // 启动（-wait=等待完成）
SERV [-wait] !servicename                    // 停止（! 前缀，-wait=等待完成）
SERV [?返回名] servicename                   // 查询状态
SERV -create [?返回名] name,path,type,start[,error,dep,user,pass,display,group,tag]
SERV -delete [-stop-] [?返回名] name         // 删除（-stop-=删除前自动停止）
```
启动类型：`-boot`，`-system`，`-auto`，`-demand`，`-disabled`，`-delayed-auto`

### HOTK — 系统级热键
```
HOTK [--delall] [?[.]返回名] Ctrl+Alt+#0x41,command  // 注册（全局，系统级）
HOTK Ctrl+Shift+Alt+Win+#0x42,command       // 多修饰键
HOTK #0x0D,--del                            // 按键码取消注册
HOTK --del:keyname                          // 按名称取消注册
HOTK --delall                               // 取消所有已注册热键
HOTK ?返回名 #0x41                          // 查询指定键的热键名
```
修饰键：`Ctrl`，`Alt`，`Shift`，`Win`。用 `+` 组合。
虚拟键码使用 `#` 前缀（十进制或十六进制：`#0x41`）。

### HKEY — 窗口/程序级热键
```
HKEY #0x41,command                          // 默认 = 窗口级（此窗口有焦点时响应）
HKEY *#0x41,command                         // * = 窗口激活时响应（不同窗口可重用，个数不限）
HKEY $#0x41,command                         // $ = 程序级全局（此 PECMD 实例的任何窗口）
HKEY Ctrl+Shift+#0x42,command               // 多修饰键
HKEY #0x0D,--del                            // 按键码取消注册
HKEY --del:keyname                          // 按名称取消注册
```

### DATE — 日期/时间变量和子变量
```
DATE &var                                  // 获取当前日期（yyyy mm dd HH MM SS ms weekday 格式）
DATE &var yyyy-mm-dd-HH-MM-SS-ms-wd        // 设置系统日期/时间（可部分设置）
DATE -h &var                               // 高精度计时器（微秒）
DATE -r &var                               // 同步 + 读取高精度计时器
DATE -space0 &var                          // 空格分隔，0 填充
DATE -space &var                           // 空格分隔（默认紧凑）
DATE -bsys &var                            // 输出系统时间
DATE -utc:UTCtime &var                     // 从 UTC 时间转换
DATE -gmt:GMTtime &var                     // 从 GMT 时间转换
DATE -local:LOCALtime &var                 // 从本地时间转换
DATE -sys:internTime &var                  // 国际/UTC 时间
DATE -us &var                              // 微秒（4 位小数）
```
子项（使用 `MSTR` 或直接 `%&var:item%` 语法）：
| 子项 | 含义 |
|---|---|
| `y` / `year` | 年（4位） |
| `mon` / `month` | 月（2位） |
| `d` / `day` | 日（2位） |
| `w` / `weekday` | 星期几（1=周一..7=周日） |
| `h` / `hour` | 小时（24小时制，2位） |
| `min` / `minute` | 分钟（2位） |
| `s` / `second` | 秒（2位） |
| `ms` / `msec` | 毫秒（3位） |
| `ws[1]` | 年内第几周（[1]=周日为周末边界） |
| `ds` / `daysofyear` | 年内第几天（1-366） |
| `Freq` / `frequency` | 计数器频率 |
| `Counter` / `counter` | 硬件计时器计数器值 |
| `gmt` | GMT 秒数（自 1970-01-01 起） |
| `uptime` / `uptime_ms` | 自开机以来的毫秒数 |
| `utc` | 自 1601-01-01 至今的 100ns 单位数 |
| `uptimens` | 自开机以来的纳秒数 |

### TEMP — 临时文件/目录管理
```
TEMP [[@]Delete|[$]Setting] [初始目录][,变量名]           // 查询/设置临时目录
TEMP @[$]Setting 新临时目录,[变量名]                       // 静默设置
TEMP [*del] [*tmpl:[前部]*[尾部]] *tmpdir [,]变量名        // 生成唯一临时目录
TEMP [*del] [*tmpl:...]*tmpfile [,]变量名[,目录变量名]      // 生成唯一临时文件
```
`@` = 静默模式。`*del` = 退出时自动删除。`*tmpl:` = 自定义名称模板（`*` = 随机部分）。

### RUNS — 运行注册表项
```
RUNS prog,Name                       // 添加到 HKLM\...\Run
RUNS -d Name                         // 删除条目
```

### PATH — 设置搜索路径
```
PATH C:\Tools;%PATH%                 // 设置 PATH 环境变量
PATH %CurDir%\Tools                  // 追加到现有值
```

### RECY — 设置回收站容量
```
RECY D:,10                           // NT5.x: D: 盘回收站最大 10%
RECY C:,2048                         // NT6.x: C: 盘回收站最大 2048MB
RECY D:,0                            // 禁用 D: 盘回收站
RECY *:,0                            // 禁用所有分区回收站
```

### USER — 设置所有者信息
```
USER 用户名,公司名                    // 设置"我的电脑"属性值
```

### HOME — 设置 IE 主页 / 锁定主页 / 禁用注册表编辑器
```
HOME http://example.com              // 设置 IE 主页
HOME http://example.com,1            // 锁定主页（禁止修改）
HOME http://example.com,1,1          // 锁定主页 + 禁用注册表编辑器
```

---

## 音频与显示

### SCRN — 截图 / 捕获屏幕
```
SCRN &w,&h                             // 获取屏幕宽度和高度
SCRN -win &w,&h                        // 最大化窗口尺寸
SCRN -desk &w,&h                       // 桌面分辨率（不考虑 DPI 缩放）
SCRN -cur &x,&y                        // 获取光标位置
SCRN -cap scrn.bmp,&wid                // 捕获全屏为 BMP，返回窗口 ID
SCRN -cap scrn.bmp,&wid,WxH            // 按指定分辨率捕获
SCRN -cap scrn.bmp,&wid,WxH,x,y        // 捕获 (x,y) 处大小为 WxH 的区域
SCRN -cap -capwid:WID scrn.bmp,&wid    // 捕获指定窗口
SCRN -cap -cur scrn.bmp,&wid           // 捕获时包含光标
SCRN -cap scrn.jpg,&wid,0,0,0,80       // 捕获为 JPG（质量 80）
SCRN -cap :image/png:screenshot.png,0  // 捕获为 PNG（格式前缀）
SCRN -cap :image/bmp:file.bmp,0        // 捕获为 BMP（格式前缀）
SCRN -cap file.bmp,#WindowID           // 按句柄捕获指定窗口
SCRN -cap file.bmp,<x:y:R:B>           // 捕获矩形区域
```
捕获格式：`.bmp`，`.jpg`，`.png`（按扩展名或 `:image/format:` 前缀）。
捕获目标：`0`=全屏，`#WindowID`=指定窗口，`<x:y:R:B>`=矩形区域。
扩展尺寸参数：`SCRN -taskbar W,H,X,Y,TaskBarPos,DpiX,DpiY,ScaleX,ScaleY` 用于 DPI 感知信息。

### FONT — 注册/注销字体
```
FONT fontPath                           // 注册字体文件
FONT \Windows                           // 从所有分区的 Windows\Fonts 注册字体
FONT - fontPath                         // 注销字体（- 前缀）
FONT -p:retName:fontName:lang fontRes   // 私有字体（不注册系统）
```

### WALL — 设置桌面壁纸
```
WALL imagePath                           // 设置壁纸（BMP、JPG、PNG、GIF）
WALL %SystemRoot%\Web\Wallpaper\img.jpg  // 绝对路径
WALL ""                                  // 清除壁纸（纯色）
```

### SITE — 文件属性查询/设置
```
// 查询
SITE ?filePath                           // 在消息框中显示属性
SITE &var,filePath                       // 获取属性到变量（如 "A--RHS-"）
SITE ?-attr,&var=&attr                   // 查询原始属性标志

// 设置
SITE +R,filePath                         // 设置只读
SITE +H,filePath                         // 设置隐藏
SITE +S,filePath                         // 设置系统
SITE +A,filePath                         // 设置存档
SITE -R,filePath                         // 移除只读
SITE +R+H+S+A,filePath                   // 组合多个
SITE +R-H,filePath                       // 设置只读并移除隐藏
SITE -R-H-S-A,filePath                   // 清除所有属性
SITE +R+H,dirPath\*                      // 应用于目录中所有文件（路径末尾加反斜杠）

// 编码变量（安全）
SITE ?-all,VAR=variable                  // 编码变量（PECMD 风格）
SITE ?-sys,VAR=variable                  // 以系统标志编码
SITE ?H:hWnd,variable1[,variable2]       // 复制到剪贴板

// 文件版本查询
SITE ?fileVerVar[,prodVerVar]=FVER,filePath  // 查询文件版本（如 "1.2.3.4"）

// 文件时间查询
SITE ?[-local -ws -link] [[*]creationVar,[*]writeVar,[*]accessVar]=FTIME,filePath
  // * 前缀 → 返回 UTC 时间整数（可直接比较）
  // 无 * → 返回 "yyyy mm dd HH MM SS us weekday"（固定宽度字段）
  // -local → 本地时间（默认：UTC）
  // -ws → 追加年内周数；-ws1 → 周日为周末边界
  // -link → 跟随符号链接

// 文件属性查询
SITE ?[attrVar][,hidVar][,roVar][,sysVar][,fullVar]=FATTR,filePath

// 更新文件时间戳
SITE *touch[:[cr][*local:|*local0:|*sys:|*sys0:|*utc:]time],<file>[,retVar]
```
查询结果中的属性标志：`R`=只读，`H`=隐藏，`S`=系统，`A`=存档，
`N`=普通，`D`=目录，`C`=压缩，`E`=加密，`T`=临时，`O`=脱机。

---

## 驱动与设备

### DEVI — 设备驱动安装
```
// 从 CAB/INF/文件夹安装
DEVI [$]<CAB文件>[,匹配级别[,解压目录]]         // 从 CAB 安装
DEVI [*nocheck] <INF文件>[,DevClass]           // 从 INF 安装
DEVI [*rescan] <含有INF的目录>[,DevClass]       // 从目录安装
DEVI $<INF文件>,[安装节],[操作码]               // 高级安装
DEVI *extract <CAB>[,匹配级别],解压目录          // 仅解压

// 列出设备
DEVI listdev:var [*devclass:Class] [*ALL] [*listdev=i|c|+]
DEVI listdev:var *many *devid:PCI\VEN_14E4*     // 按筛选条件列出

// 控制设备
DEVI *enable:[h|c|+:]devID                      // 启用设备
DEVI *disable:[h|c|+:]devID                     // 禁用设备
DEVI *remove:[h|c|+:]devID                      // 移除设备
DEVI *restart:[h|c|+:]devID                     // 重启设备
DEVI *status:retVar:[h|c|+:]devID               // 查询状态
DEVI *update:hardwareID:INF                      // 更新驱动
DEVI *install:hardwareID:INF                     // 安装驱动

// 其他
DEVI *rescan[:Fun]                               // 重新扫描设备
DEVI buildcache:[-a:arch] dir                    // 构建驱动缓存
```
高级标志：`*dummy`=测试模式，`*7pe[-]`=强制 DrvLoad，`*inner`=强制不使用 DrvLoad，`*drvload/*devcon`=优先级选择，`*retid:var`=返回安装的设备 ID，`*auto`=自动转换 INF，`*sys:`=复制到系统目录，`*cab`=强制 CAB 类型，`*comp+`=匹配兼容 ID，`*ret:retVar`=返回报告，`*IdCah:PeVar`=重用 ID 缓冲区，`*infcache:`=加速缓存，`*optsys[:val]`=系统工具优先级，`*num:count`=计数限制，`*disverify/*autodisverify`=签名检查控制，`*sub/*self`=搜索模式，`*showdev:`=显示设备信息，`*norescan`=跳过重新扫描。
Listdev 选项：`*comp[+]`=兼容 ID，`*hwid`=硬件 ID，`*inst`=实例 ID，`*many`=多行，`*rescan`=先重新扫描。

### FBWF — FBWF 缓存控制
```
FBWF [Ppercent] [Lmin] [Hmax] [Fremain]
```
所有值单位为 MB。示例：`FBWF P50 L200 H300` — 内存的 50%，最小 200MB，最大 300MB。

---

## 注册表

### REGI — 读写注册表
```
// 读取（所有类型前缀）
REGI $HKLM\SOFTWARE\Key\Val,&var          // REG_SZ
REGI #HKLM\SOFTWARE\Key\Val,&var          // REG_DWORD
REGI @HKLM\SOFTWARE\Key\Val,&var          // REG_BINARY
REGI *HKLM\SOFTWARE\Key\Val,&var          // REG_MULTI_SZ
REGI **HKLM\SOFTWARE\Key\Val,&var         // REG_MULTI_SZ（特殊）
REGI *$HKLM\SOFTWARE\Key\Val,&var         // 多行 REG_MULTI_SZ
REGI ~HKLM\SOFTWARE\Key\Val,&var          // REG_EXPAND_SZ
REGI ~~HKLM\SOFTWARE\Key\Val,&var         // REG_EXPAND_SZ（~~ 读取并重新解释注册表数据中的环境变量）
REGI +HKLM\SOFTWARE\Key\Val,&var          // REG_QWORD
REGI ^HKLM\SOFTWARE\Key\Val,&var          // REG_LINK
REGI bHKLM\SOFTWARE\Key\Val,&var          // REG_QWORD_BIG_ENDIAN
REGI uHKLM\SOFTWARE\Key\Val,&var          // REG_MUI_SZ
REGI nHKLM\SOFTWARE\Key\Val,&var          // REG_NONE
REGI .HKLM\SOFTWARE\Key\Val,&var          // 离线注册表（离线 Windows/system）
REGI HKCU\Software\Key\,&&keys            // 枚举子键（换行分隔）

// 写入
REGI $HKLM\SOFTWARE\Key\Val=string         // 写入 REG_SZ
REGI #HKLM\SOFTWARE\Key\Val=#0x100        // 写入 REG_DWORD（十六进制）
REGI $HKLM\SOFTWARE\Key\Val=               // 删除值
REGI HKCU\abc=""                           // 写入空字符串（"" 表示空值写入）

// 高级操作
REGI --ak HKCU\Software\Key\,&all             // 枚举所有子键 (k=keys)
REGI --av HKCU\Software\Key\,&all             // 枚举所有值 (v=values)
REGI .?\HKLM\SOFTWARE\Key\Val,&type           // 查询值类型（点+问号）
REGI --16 ...                                 // 十六进制数据输入
REGI --su path\val=value                      // 以 SYSTEM 身份运行（提升权限）— 用于 32 位在 64 位系统上
REGI --init path\val,&var                     // 读取失败时返回空字符串
REGI --name path\val,&var                     // 数据变量名模式
REGI --k path\key\                            // 仅创建键（不设置值）
REGI --byte path\val,&var                     // 字节流模式
REGI --v[-] path\val,&var                     // 不保存更改（读取快照）
REGI --qk path\val,&var                       // 快速模式
REGI --r10 path\val,&var                      // 输出十进制（用于 DWORD）
REGI --t:NUM path\val,&var                    // 按数字指定任意注册表类型
REGI --0[:N] path\key\                        // 清除键：1=清除默认值，2=删除子键，4=删除值（组合：5=1+4）

// 查询存在性（找不到返回 ERROR）
REGI ?HKLM\SOFTWARE\Key\,&&VT
FIND $%&VT%=ERROR, MESS Key not found! MESS Key exists
REGI ?HKLM\SOFTWARE\Key\Val,&&VT           // 检查值是否存在
REGI ?HKLM\SOFTWARE\Key\,&&VT               // 检查键是否存在
FIND $%&VT%=NI, MESS Data not set!          // NI = 键存在但无数据
```

### HIVE — 加载/卸载离线注册表配置单元（全面）
> help.txt 文档化标志：`-u`（加载到 HKU）、`-super`/`-super_r`（强制权限）、`-quick`（不添加权限）、`*`（导出到文件）。
> 以下 `-tmp`、`-restore`、`ACL` 为未文档化扩展，实测可用但可能因版本而异。
```
// 将离线配置单元挂载到 HKLM（或 HKU）下的挂载点
HIVE E:\Windows\System32\config\SOFTWARE,HKLM\PE-SYS     // 加载离线 SOFTWARE 配置单元
HIVE E:\Windows\System32\config\SYSTEM,HKLM\PE-SYS       // 加载离线 SYSTEM 配置单元
HIVE E:\Users\Default\NTUSER.DAT,HKU\PE-DEF             // 加载离线用户配置单元
HIVE HKLM\PE-SYS,                                        // 卸载（路径为空，相同挂载点）

// [未文档化] 加载时保留安全描述符（保留 ACL）
HIVE E:\...\SOFTWARE,HKLM\PE-SYS,ACL                    // 带安全/ACL 加载

// [未文档化] 作为临时配置单元加载（卸载时丢弃更改）
HIVE -tmp E:\...\SOFTWARE,HKLM\PE-TMP                   // 使用临时配置单元（只读意图）

// [未文档化] 加载并在卸载时还原（将更改保存回配置单元文件）
HIVE -restore E:\...\SOFTWARE,HKLM\PE-SYS               // 卸载时还原（写回）
HIVE -restore HKLM\PE-SYS,                               // 卸载并还原（保存更改）
```
挂载点语法：`HKLM\PE-SYS` = 将配置单元挂载到 `HKEY_LOCAL_MACHINE\PE-SYS`。
挂载后，使用 `REGI .HKLM\PE-SYS\...` 访问（点前缀表示离线注册表）。
要卸载，提供相同的挂载点并留空路径（或使用 `-restore` 前缀来保存更改）。

---

## GUI 控件

所有 GUI 控件文档参见 [pecmd-gui.md](pecmd-gui.md)。

---

## 网络

### ADSL — 宽带/WiFi
```
// 拨号（PPPoE）
ADSL userEncoded,passEncoded,[retries],[name|*|retVar]     // 拨号
ADSL start[+] userEncoded,passEncoded,[retries],[retVar]    // 开始（连接）
ADSL stop,connectionName                                    // 挂断
ADSL list[on],connectionName                                // 列出连接

// WiFi (ADSL-wlan)
ADSL-wlan SSID|&profileVar,password,encType,[index]        // 连接（加密默认=WPA2PSK AES）
ADSL-wlan -start SSID|&profileVar,password,encType,[index]  // 显式开始
ADSL-wlan index,,list,&&result                              // 列出 WiFi 配置文件
ADSL-wlan index,,query[all],&&result                        // 查询详情（序号 guid State Desc）
ADSL-wlan index,,scan,&&result                              // 扫描网络
ADSL-wlan index,,-list,&&result                             // 网络广播扫描
// 结果格式（list）：SSID SignalQuality Flags BssType NumBssid bConnectable ...
// 结果格式（query[all]）：index guid State Description
// Flags & 1 = 当前已连接
```

### PCIP — IP 配置
```
PCIP 192.168.1.100,255.255.255.0,192.168.1.1,[DNS1],[DNS2]  // 固定IP
PCIP -,-,-,-,,                                                // DHCP（动态IP）
```

### 其他网络
```
NTPC time.server.com                                // 时间同步（默认仅同步）
NTPC -q ,现在时间                                     // 同步并查询（-qo 仅查询）
SITE ftp://user:pass@server/path,local,get|put        // FTP（下载/上传）
UPNP add|del TCP|UDP,port,internalIP                  // 端口转发
```

---

## 外部执行

### EXEC — 执行程序（全面）
```
EXEC [=][!][@][^][&][*] [flags] program [args]
```
基本：`=`=等待完成，`!`=隐藏运行，`@`=不等待+隐藏，`^`=不等待，`*`=不等待

附加标志（最常用）：
```
-err+                  // 捕获 stderr 到同一输出
-err                   // 单独捕获 stderr
-cmd:::Callback        // 实时输出行回调
-wd:path               // 设置工作目录
-pid:var               // 获取进程 PID
-su[acde]              // 以 SYSTEM 身份运行
-doc:mode              // 打开/编辑/打印/属性 文档
-min|-max|-show|-hide  // 窗口状态
-user:name -passwd:pwd // 以指定用户身份运行
-timeout:ms[:code]     // 超时
-waiti                 // 等待 UI 初始化
-raw                   // 捕获原始数据（不重新编码）
-nowin                 // CREATE_NO_WINDOW
-incmd                  // 在新的 PECMD 实例中运行命令（无消息循环）
-hook                   // 修改进程关机代码
-no64                   // PECMD32：不释放 X64 文件系统限制
-ex1                    // 继承父进程 PE 变量为环境变量
-nfb                    // 禁用等待光标
-hpid:var               // 获取进程句柄（非 PID）
-exe:filename           // 执行指定文件名（可执行非标准后缀文件如 .tmp）
--exe:#resID            // 执行内嵌资源程序（#=资源 ID，无*=内存执行，有*=临时文件执行）
--exe:[*[*]][?.ext:][cab:]传递  // 内嵌程序高级语法

// 附加 EXEC 标志：
-clone:var            // 克隆 PECMD 运行脚本变量
-mem                  // 幽灵进程（内存中执行）
-io                   // 接管子进程 I/O
-code:<enc>           // 指定源编码
-REALTIME|-HIGH|-ABOVENORMAL|-NORMAL|-BELOWNORMAL|-LOW|-IDLE  // 进程优先级
-shel:"auto_cmd"      // 在 SHEL 模式下执行
-svrsys|-svrusr|-svr- // 服务到桌面执行
/InstallService /name // 安装为 Windows 服务
/RemoveService name   // 卸载服务
-poprmenu|-runrmenu   // 弹出/执行文件右键菜单
-runs                 // 写入注册表自动运行（= 前导: HKLM\...\Run, 否则 HKCR\...\Run）
// 服务模式附加标志（配合 /InstallService）：
--wait|--nowait       // 等待/不等待进程结束（默认等待）
--idle ms             // 空闲 ms 毫秒后执行命令
--idlewait            // 有输入时不终止已运行程序
--killwin             // 监控并关闭 MESS-svr 窗口
--nojob               // 不杀子进程
--gui-                // 无界面交互
--desk[-]:[不]切换显示 // 桌面切换控制
--delayservice ms     // 服务启动延迟毫秒数
--hide                // 隐藏进程窗口
```

### EXEC* — 捕获输出
```
EXEC* [*1|*N|*-] [-catch] [-cmd:::Callback] [-err+] [&]outputVar=program [args]
```
`*1`=仅第一行，`*N`=合并行，`*-`=去除尾部换行，`*$`=首行+变量展开，`*^`=首行+推迟展开
`NAME+=` — 追加模式（在 EXEC* 中，NAME+= 将输出追加到变量而非覆盖）
EXEC| [flags] program [args]                        // 管道模式（支持 > >> < 2> >& 重定向）

### SOCK — Windows 套接字 / IPC
```
SOCK [*] Name[;ProFamily][;ProType][;ProID]        // 创建套接字（*=自动回收）
SOCK --file [*] Name;[we][-rwd];FileName           // 文件句柄
SOCK --shm [*] Name;[w];ShareName;Length[;...]     // 共享内存（w=可写）
SOCK --event [*] Name;ShareName[;Init;ManualReset] // 事件对象
SOCK --sem [*] Name;ShareName[;InitCount;MaxCount] // 信号量
SOCK --mutex [*] Name;ShareName[;InitLocked]       // 互斥锁
SOCK --pipe [*] Name;ShareName[;Timeout;BufSz;Mode] // 命名管道（0x1=立即，0x2=客户端，0x4=服务端）
SOCK --unknown Name[,InitialValue]                 // COM IUnknown 指针（*=自动释放）
SOCK --BSTR[vt] Name[,[*][InitialValue][,FromString]] // COM BSTR 字符串
SOCK --gethostbyname[*|#] IPName;HostName          // DNS 解析
SOCK --BST                                         // 加载 BSTR DLL
SOCK --mailslot [*] Name;ShareName[;IsServer;Timeout] // 邮件槽
```

套接字操作（全部通过 `ENVI @Name.operation=`）：
```
ENVI @Name.connect=[ErrVar];IP;Port               // TCP 连接
ENVI @Name.bind=[ErrVar];IP;Port                  // 绑定（服务端）
ENVI @Name.listen=[ErrVar][;Backlog]              // 监听（默认 backlog=7）
ENVI @Name.accept=[ErrVar];[ListenFD][;IPVar][;PortVar]  // 接受连接
ENVI @Name.write=[ErrVar];[LenVar];[DataVar];[BytesToSend[@Offset]][;Flags][;IP][;Port]  // 发送
ENVI @Name.read=[ErrVar];[LenVar];[DataVar];[*][BytesToRecv[@Offset]][;Flags][;IPVar][;PortVar]  // 接收（*=多次读取）
ENVI @Name.close=[ErrVar]                         // 关闭
ENVI @Name.shutdown=[ErrVar][;Mode]               // 0=接收，1=发送，2=两者
ENVI @Name.sock=[ErrVar][;ProFamily;ProType;ProID]  // 重新创建套接字
ENVI @Name.fd=fdVarName                           // 获取文件描述符
ENVI @Name.mem=memVarName                         // 获取共享内存地址
ENVI @Name.setsockopt=[ErrVar];[Level];Item;DataVar[;DataLen]  // 设置套接字选项
ENVI @Name.select=[[*]ErrVar];MsTimeout;[[Ret:]fd1:fd2:...]  // 多路复用（*=API 错误）
ENVI @Name.getname=[ErrVar];[0/1];[IPVar][;PortVar]  // 0=本地，1=对端
ENVI @Name.wait=[ErrVar][;Timeout][;[*]handle2:...]  // 等待（事件/信号量/互斥锁）
ENVI @Name.setevent=[ErrVar][;1][;OldValueVar]    // 信号事件（0=清除）
```

管道/邮件槽操作：`.read` / `.write` / `.connect`

### PINT — 固定到任务栏/开始菜单
```
PINT %Desktop%\Name.lnk,TaskBand         // 固定到任务栏
PINT %Desktop%\Name.lnk,StartMenu        // 固定到开始菜单
```

### LINK — 创建快捷方式
```
LINK %Desktop%\Name.lnk,target,[args],[icon],[iconIdx],[workDir]
LINK [?]Name.lnk                          // 查询快捷方式信息
```

---

## 附加控件

附加 GUI 控件文档参见 [pecmd-gui.md](pecmd-gui.md)。

---

## 字符串操作

### MSTR — 多字符串提取
```
MSTR &a,&b=<1><3>%&data%              // 提取字段 1 和 3
MSTR &rest=<5->%&data%                 // 字段 5 到末尾（`-` = 到末尾）
MSTR &single=<5*>%&data%               // 仅字段 5（`*` = 单字段）
MSTR &restq=<~5>%&data%                 // 字段 5 并剥离外层引号（~ = 去除引号）
MSTR &s=pos,len,%&str%                 // 指定位置的子串
MSTR &last=<-1>%&data%                 // 最后一个字段（负索引）
MSTR -delims:. &a,&b,&c=<1><2><3>%&ip% // 按自定义分隔符拆分
MSTR* &a,&b=<1><2>%&data%               // TAB 分隔（命令前缀 *）
MSTR$ &a,&b=<1><2>%&data%               // 空格分隔，连续空格视为单个
MSTR -xq &a=<1>%&data%                  // 字符串内可含转义引号 (\")
MSTR -rq &a=<1>%&data%                  // 剥离双引号
MSTR -rq1 &a=<1>%&data%                 // 剥离单个双引号（含不配对的）
MSTR -term &a,<1>%&data%               // 保留后置分隔符
MSTR -term2 &a,<1>%&data%              // 保留前置和后置分隔符
MSTR -trim &a=  hello  %&data%         // 去除前后空白
MSTR -trimp &a=  hello  %&data%        // 预先处理后去除空白
MSTR -trimleft &a=  hello  %&data%     // 仅去除左侧空白
MSTR -trimright &a=  hello  %&data%    // 仅去除右侧空白
MSTR &count=<#>%&data%                  // `串号#` 返回总字段数（# 替代数字）
```
`-xq`：字段内可含 `\"` 转义引号。`-rq`/`-rq1`：剥除外层引号。`-term[2]`：保留分隔符。
`-trim[p][left|right]`：`$` 前缀时仅匹配空格，不含 TAB。`-trimp`：预先处理模式。
`-left`：保留字符串最开头的空白（TAB 方式默认开启）。
`<#>`：用 `#` 替代数字索引，返回总字段数。

### SED — 字符串/正则替换
```
SED [flags] &r=count,pattern,replacement,%&source%
SED &r=0,pat,rep,%&s%                  // 替换全部（count=0 全部替换）
SED &r=1,pat,rep,%&s%                  // 替换第一个
SED &r=-1,pat,rep,%&s%                 // 替换最后一个匹配（负数=从末尾）
SED &r=0:0,pat,rep,%&s%                // 正则模式（等同 count=0）
SED -t &r=0,Hel,Wor,%&s%              // -t = 字符集翻译（H→W, e→o, l→r，逐字符映射）
SED -ts &r=0,[old],[new],%&s%          // -ts = 字符串集翻译（0x0A 分隔）
SED -ni &r=0,pat,rep,%&s%              // -ni = 不区分大小写
SED -ex &r=0,0x00 0x00,0x0D 0x0A,%&hex%  // -ex = 二进制十六进制模式（hex byte pattern）
```
⚠ **SED 默认使用正则模式。** `.` 匹配任意字符，非字面量句号。匹配字面量句号用 `\.`。使用标志字符 `*` 可切换为字面量模式（不解释正则）：`SED &r=*0,.,X,%&s%` 中 `.` 匹配字面量句号。
count：`0`=替换全部匹配，`N`=替换前N个匹配，`-N`=替换后N个匹配。
`0:0` 与 `0` 等价（均替换全部）。`.*\.` 匹配到最后一个句号及之前内容。

标志（flags）：
- `-t` — 字符集翻译模式（一对一字符映射，如 Unix `tr`，无方括号）
- `-ts[1]` — 字符串集翻译（0x0A 分隔各子串）
- `-ni` — 不区分大小写
- `\u:` — 转换为大写，`\l:` — 转换为小写
- `-x[:group]` — 二进制对象十六进制模式
- `-ex` — 扩展模式
- 查找模式（`count=?`）：`SED -ni &r=?:1,pattern,,%&s%` 查找位置
- `-Lf:Lt:Rf:Stp` — 向量操作模式（`Lf:Lt` 为从/到向量，`Rf` 为替换源，`Stp` 为步长）
- `-h[~]?` — 非起始匹配时的前导串（`~` = 不含前导串本身）
- `-e[~]?` — 非结尾匹配时的后缀串（`~` = 不含后缀串本身）
- `-f` — 严格模式：不匹配则丢弃
- `-many` 或变量名前缀 `*` — 返回多个匹配位置
- 标志字符：`*` = 字面量模式（不解释正则），`_` = 占位

### LPOS — 左起查找位置（全面）
```
LPOS[*] [-case] &pos=needle,[count],%&haystack%       // 查找字符位置（默认不区分大小写）
LPOS* [-case] &pos=substring,[count],%&haystack%      // 查找子串位置（* = 子串模式）
LPOS** [&pos] [-qu] [-delims:C] [-case] =substring,[count],%&haystack%  // 返回子串编号
LPOS*** [-qu] [-delims:C] [-case] &count,&pos=substring,,%&haystack%    // 返回计数+位置
LPOS**# [-delims:C] =substring,[count],%&haystack%    // TAB 分隔模式（# = TAB 分隔）
LPOS**$ [-delims:C] =substring,[count],%&haystack%    // 空格分隔模式（$ = 空格分隔）
```
`LPOS` — 查找字符位置（默认不区分大小写）。
`LPOS*` — 返回所有匹配位置。
`LPOS**` — 返回匹配计数（第几个匹配）。
`LPOS***` — 返回计数和位置。
`-case` — 区分大小写（默认不区分）。
`-qu` — 去除引号。
`-delims:C` — 自定义分隔符（支持 `\n\r\t\v\f\b`）。
count < 1 时返回最右边位置。返回 0 表示未找到。

### 其他字符串操作
```
LSTR &left=N,%&str%                     // 前 N 个字符
RSTR &right=N,%&str%                    // 后 N 个字符
SSTR [-case] &pos=needle,count,%&str%    // LPOS* 别名——查找子字符串位置（非子串提取）
RPOS &pos=needle,[1],%&haystack%        // 查找最后一个（行为与 LPOS 类似，从右起）
STRL &len=%&str%                        // 字符串长度
RAND &var                               // 随机 63 位整数
```

---

## 其他命令

### DTIM — 日期/时间选择器（GUI 控件）
```
DTIM [-right] [*] Name,LxTyWwHh,[初始值],[事件],[类型]
// 类型：0x20=长日期，0x40=时间，0x80=短世纪，0x100=上下键，0x200=带勾选器
// 初始值格式：年;月;日（如 2008;5;12）
```
> 日期/时间选择器控件，必须位于 `_SUB` 窗口内。查询：`ENVI @Name.VAL=?&&Y;&&M;&&D;&&DOW;&&Extra`（5 个字段：年、月、日、星期、附加）。
> 设置：`ENVI @Name.VAL=年;月;日`。

### BASE — Base64
```
BASE string,&var                         // PECMD 变体（用于 ADSL 安全）
BASE* string,&var                        // 标准 base64
BASE* -u string,&var                     // 标准解码
```

### CMPS — 压缩
```
CMPS -m source.wcs,dest.wcz             // 压缩（加密）
CMPS -m -u source.wcz,dest.wcs           // 解压
CMPS -f -m source.wcs,dest.wcz           // 压缩（不加密，-m 在 -f 之后）
CMPS -bin source.exe,dest.wcz            // 压缩二进制（非脚本）
CMPS -src[:flags] source.wcs,dest.wcz    // 源码压缩及清理标志：
  // -src:1 = 移除注释行
  // -src:2 = 转换行尾符
  // -src:4 = 压缩空行
  // -src:8 = 移除行内注释
  // 组合：-src:15 = 以上全部
CMPS -utf8 source.wcs,dest.wcz           // 编码为 UTF-8
```

### WAIT — 暂停 / 按键等待
```
WAIT ms                                    // 暂停毫秒数
WAIT -1                                    // 永久等待
WAIT -cont [-timeout],[&var]              // 非阻塞按键等待
WAIT *pid|*tid                             // 等待进程/线程完成
WAIT **                                     // 等待祖父进程
WAIT =tid                                   // 等待指定线程 ID
WAIT -del file1 [-del file2]               // 等待后删除文件（含重试）
WAIT -delms:N                               // 删除重试间隔（毫秒）
WAIT -scanall|scan:key,&var                // 获取键盘扫描状态表
WAIT -sys[0] [switch] -cmd                 // 系统代理执行
WAIT -sys[0]cmd                            // 系统直接代理执行
WAIT -thread                                // 等待所有子线程
WAIT $handle                                // 等待句柄
WAIT -freemem                               // 释放内存
WAIT -pad                                   // 区分小键盘按键
WAIT -ncd                                   // 等待期间不切换目录
WAIT &&PressKey.Hex                         // 十六进制键码子变量
WAIT time1 time2                            // time1>0&<1：*100000 = 待处理消息数
```

### KILL — 终止
```
KILL process.exe                           // 按名称
KILL *12345                                // 按 PID
KILL \                                     // 当前脚本的窗口
KILL \WinName                              // 指定窗口标题
KILL process.exe|username                   // 按名称 + 所有者
KILL \[windowTitle]                        // 按窗口标题
KILL @[windowName]                         // 按窗口类名
KILL @@windowID                            // 按窗口 ID
KILL **tid                                 // 按线程 ID（异步终止）
KILL *&hpid                                // 按进程句柄
KILL **&htid                               // 按线程句柄（异步）
KILL -force process.exe                    // 强制终止
KILL -explorer process.exe                 // 阻止 explorer 自动重启
KILL -gui                                  // 进程管理器 GUI
KILL -tree process.exe                     // 终止进程树
KILL -svr2                                 // 用于 MESS-svr2
KILL -exitcode:NUM process.exe             // 设置退出代码
KILL ** process.exe                        // 强制同步终止
```

### LOGS — 调试日志 / 控制台输出
```
LOGS * C:\log.txt                       // 开始记录日志到文件
LOGS * CONOUT$                          // 输出到控制台（调试终端）
LOGS --t=1 --ln=1                       // 包含时间戳和行号
LOGS                                    // 停止记录
```
- `CONOUT$` — 输出到控制台/调试终端（替代 ECHO）
- `--thread` — 本线程日志，`@` — 公共日志，`3` — 两者
- `--p` — 执行前打印，`--2` — 前后都打印
- `--ln=1/0` — 显示/隐藏行号，**logs_ln** 环境变量可替代

### COME / NOTE — 注释开关
```
COME 0 / NOTE OFF                       // 禁用注释
COME 1 / NOTE ON                        // 启用注释
```

### HELP
```
HELP                                    // 显示完整帮助
HELP commandName                        // 显示指定命令帮助
HELP -hlpdoc=page                       // 指定帮助页
HELP ~bookmark                          // 跳转到指定主题
HELP *Height                            // 命令行模式（Height<=-100000 无命令栏）
HELP 0xFG#0xBG                          // 设置前景色和背景色
```

### SEND — 键盘/鼠标模拟
```
SEND [--ext] [--s] [-gui [-m] [-nfocus] [-right|-left|-top]] <key[_|^]>[;key2;...]
SEND -m flag;dx;dy[;dat;extdat]         // 鼠标事件模拟
```
按键后缀：`_` = 仅按下，`^` = 仅释放，无后缀 = 按下+释放。
支持 VK_ 常量名字符串（VK_ 可省略）、直接字母（A-Z, 0-9）、十六进制键码（`#0x0D` = Enter）。
支持 `,` `;` `:` `/` 和空格作为分隔符。

附加标志：
`--ext` — 扩展键模式。
`--s` — 静默模式（不产生蜂鸣等声音）。
`-nfocus` — 不改变焦点（仅在 `-gui` 模式下有效）。
`-right|-left|-top` — 方向标志（仅在 `-gui` 模式下有效）。

鼠标标志（`-m flag`）：
| 标志 | 描述 |
|---|---|
| `0x0001` | 鼠标移动 |
| `0x0002` / `0x0004` | 左键 按下/释放 |
| `0x0008` / `0x0010` | 右键 按下/释放 |
| `0x0020` / `0x0040` | 中键 按下/释放 |
| `0x0080` / `0x0100` | X 按钮 按下/释放 |
| `0x0800` | 滚轮滚动（dat=120 向上一格） |
| `0x8000` | 绝对坐标（否则为相对坐标） |
| `0x4000` | 映射到整个虚拟桌面 |
| `0x100000` | 触摸屏 |

```
SEND #0x0D                              // 发送 Enter
SEND A;B;C                              // 依次发送 A B C
SEND -gui #0x1B                         // GUI 模式发送 Escape
SEND VK_NUMLOCK                         // 发送 NumLock（VK_ 名称直接使用）
SEND -m 0x0001;100;200                  // 鼠标移动到 (100,200)
SEND -m 0x0002;0;0                      // 左键按下
SEND -m 0x0004;0;0                      // 左键释放
SEND -m 0x0800;0;0;120                  // 鼠标滚轮（120=向上一格）
SEND -m 0x8080;0;0                      // X 按钮按下（绝对坐标）
SEND -m 0x100000;0;0                    // 触摸屏事件
```

### BROW — 浏览文件/目录对话框
```
BROW [-fix] <变量名[;flgnm]>,[[*|&]初始路径],[提示文字],[扩展名],[标志][,hookfun[,参数]]
```
`-fix` — 破解系统目录屏蔽（不一定成功）。
前导符：无 = 打开文件对话框，`*` = 浏览目录对话框，`&` = 保存文件对话框。
扩展名支持多选串：`说明1|*.后缀1|说明2|*.后缀2|`。
标志：`0x10`=有编辑框，`0x200`=无新建文件夹按钮/多选，`0x4000`=混合选择文件和目录，
`0x1000`=文件必须存在，`0x80000`=浏览器风格，`0x2`=覆盖警告，`0x1`=勾选只读。

### MESS — 消息框
```
MESS [flags] [文字内容][@标题][#类型[*自动关闭ms][$默认选择]]
```
消息类型：`OK`（默认）、`YN`、`YNC`、`OKC`、`RETRY`、`ABORT`。
默认选择：`$Y`、`$N`、`$C`、`$O`、`$R` 等。负数不显示计时。

调用模式：
`MESS` — 标准阻塞模式。
`MESS*` 或 `MESS-bin` — 并行调用（父窗口可同时操作）。
`MESS-` 或 `MESS-bg` — 后台调用（继续执行后续命令）。
`MESS~` — 快速后台调用（费时操作也不阻塞）。
`MESS.` 或 `-raw` — 不转换 `\n`（原始模式）。

图标（`+icon` 后数字）：0=惊叹，1=警告，2=信息/3=星号，4=问题，5=停止/6=错误，7=招手停。
`>=32` 或前带 `*` 为自定义 IconGroup 号。
`-svr` / `-svr2` — 登录前可显示窗口。`-min` / `-max` — 最小化/最大化。
`-size` — 可调大小。`-close` — 无关闭按钮。`-top` — 最顶端。
`-txt` — 静态文本。`-cb` — 复制到剪贴板。

返回值保存在 `%YESNO%` / `%&YESNO%` 中：YES/NO/OK/CANCEL/RETRY/IGNORE/ABORT。

### HIDE — 隐藏 PECMD 进程
```
HIDE
```
隐藏 PECMD.EXE 进程，防止被其它程序或人为误杀。
不能在命令行中使用，只能在配置文件中使用。
`SHEL` 命令必须在 `HOTK` 和 `HIDE` 命令之后。只有通过 `SHEL` 加载 Shell 时，`HIDE` 才能生效。

### UPNP — 端口转发 / BartPE 功能
```
UPNP [$]<参数>
```
执行 BartPE.EXE 的功能（内嵌，无需单独文件）。
前导 `$` — 显示 BartPE.EXE 的执行界面。
参数为 BartPE.EXE 的命令行参数。
仅支持 NT5.x 系列 PE，阻塞模式执行。

### TEXT — 显示文字
```
TEXT[.] [文字][#颜色][L左][T上][R右][B下][=+-][$字体大小[:字体名]][*]
```
在登录画面或桌面窗口显示文字。`\n` 换行，后缀 `.` 为原始模式（不转换换行）。
文字为空则清除最近定义的矩形区内的文字。默认颜色为白色。
`#LTRB=+-` 必须大写，顺序不能变。`=右对齐` `+水平居中` `-垂直居中`。
`$字体大小[:字体名]` 设置字体。结尾 `*` 表示显示新文字前不清除原来已显示的文字。
默认字体大小为 16（相当于宋体小 5 号）。

### NUMK — 小键盘数字锁定
```
NUMK <数值>
```
控制小数字键盘的 NumLock 开关状态。`0` = 关闭，`1` = 开启。
比 `SEND VK_NUMLOCK` 更准确：当 NumLock 已开时，SEND 再发一次反而会关掉。

### TIME — 定时器控制
```
TIME TimerName,interval,command          // 周期性定时器
TIME -t:1 TimerName,interval,command     // 单次定时器
ENVI @TimerName=0                        // 停止
ENVI @TimerName=interval;count           // 运行 N 次
ENVI @TimerName=-del                     // 销毁
```

### SHUT — 关机/重启/注销
```
SHUT [-force] [E|O数字|C|R|L|H|S|K|SHUTDOWN|-] [--] [脚本参数表]
```
无参数=关机，`R`=重启，`L`=注销，`H`=休眠，`S`=挂起，`K`=锁定，`-force`=快速关机。
`E`=弹出光驱后等待10秒，`O数字`=弹出光驱后等待指定毫秒，`C`=关闭光驱。
`SHUTDOWN -s -r -f --f -t 秒数`=另类关机方式。
`-- [scriptFile]`：关机时运行指定脚本。
关机前自动执行 `%SystemRoot%\System32\OnShutdown.wcs`（操作码: shutdown reboot logout suspend hiber poweroff unknown lock）。

示例：`SHUT R`（重启）、`SHUT -force R`（强制重启）、`SHUT O5`（弹出光驱等5秒）、`SHUT K`（锁定）。

### LOGO — 启动画面
```
LOGO [文件路径[,透明色]] [-] [-top] [-enable] [-wait] [-trans:N]
```
支持 BMP/JPG/PNG/GIF（需 GDI+）。无参数 = 渐隐淡出隐藏。
标志：`-`（快速退出），`-top`（置顶），`-enable`（ESC 退出），`-wait`（等待动画结束），`-trans:N`（透明度 0-255）。

---

## 附录：虚拟键码

与 `HKEY`、`HOTK`、`WAIT -cont` 和 `SEND` 配合使用的常用 Windows 虚拟键码：

| 键名 | 十进制 | 十六进制 | 描述 |
|---|---|---|---|
| VK_LBUTTON | 1 | 0x01 | Left mouse button |
| VK_RBUTTON | 2 | 0x02 | Right mouse button |
| VK_CANCEL | 3 | 0x03 | Ctrl+Break |
| VK_MBUTTON | 4 | 0x04 | Middle mouse button |
| VK_XBUTTON1 | 5 | 0x05 | Mouse X button 1 |
| VK_XBUTTON2 | 6 | 0x06 | Mouse X button 2 |
| VK_CLEAR | 12 | 0x0C | Numpad 5 (Num Lock off) |
| VK_BACK | 8 | 0x08 | Backspace |
| VK_TAB | 9 | 0x09 | Tab |
| VK_RETURN | 13 | 0x0D | Enter |
| VK_SHIFT | 16 | 0x10 | Shift |
| VK_CONTROL | 17 | 0x11 | Ctrl |
| VK_MENU | 18 | 0x12 | Alt |
| VK_PAUSE | 19 | 0x13 | Pause/Break |
| VK_CAPITAL | 20 | 0x14 | Caps Lock |
| VK_KANA | 21 | 0x15 | IME Kana/Hangul mode |
| VK_JUNJA | 23 | 0x17 | IME Junja mode |
| VK_FINAL | 24 | 0x18 | IME Final mode |
| VK_HANJA | 25 | 0x19 | IME Hanja/Kanji mode |
| VK_CONVERT | 28 | 0x1C | IME Convert |
| VK_NONCONVERT | 29 | 0x1D | IME Non-Convert |
| VK_ESCAPE | 27 | 0x1B | Esc |
| VK_SPACE | 32 | 0x20 | Spacebar |
| VK_PRIOR | 33 | 0x21 | Page Up |
| VK_NEXT | 34 | 0x22 | Page Down |
| VK_END | 35 | 0x23 | End |
| VK_HOME | 36 | 0x24 | Home |
| VK_LEFT | 37 | 0x25 | Left Arrow |
| VK_UP | 38 | 0x26 | Up Arrow |
| VK_RIGHT | 39 | 0x27 | Right Arrow |
| VK_DOWN | 40 | 0x28 | Down Arrow |
| VK_SELECT | 41 | 0x29 | Select key |
| VK_PRINT | 42 | 0x2A | Print key |
| VK_EXECUTE | 43 | 0x2B | Execute key |
| VK_SNAPSHOT | 44 | 0x2C | Print Screen |
| VK_INSERT | 45 | 0x2D | Insert |
| VK_DELETE | 46 | 0x2E | Delete |
| VK_HELP | 47 | 0x2F | Help key |
| VK_0 | 48 | 0x30 | 0 |
| VK_1 | 49 | 0x31 | 1 |
| VK_2 | 50 | 0x32 | 2 |
| VK_3 | 51 | 0x33 | 3 |
| VK_4 | 52 | 0x34 | 4 |
| VK_5 | 53 | 0x35 | 5 |
| VK_6 | 54 | 0x36 | 6 |
| VK_7 | 55 | 0x37 | 7 |
| VK_8 | 56 | 0x38 | 8 |
| VK_9 | 57 | 0x39 | 9 |
| VK_A | 65 | 0x41 | A |
| VK_B | 66 | 0x42 | B |
| VK_C | 67 | 0x43 | C |
| VK_D | 68 | 0x44 | D |
| VK_E | 69 | 0x45 | E |
| VK_F | 70 | 0x46 | F |
| VK_G | 71 | 0x47 | G |
| VK_H | 72 | 0x48 | H |
| VK_I | 73 | 0x49 | I |
| VK_J | 74 | 0x4A | J |
| VK_K | 75 | 0x4B | K |
| VK_L | 76 | 0x4C | L |
| VK_M | 77 | 0x4D | M |
| VK_N | 78 | 0x4E | N |
| VK_O | 79 | 0x4F | O |
| VK_P | 80 | 0x50 | P |
| VK_Q | 81 | 0x51 | Q |
| VK_R | 82 | 0x52 | R |
| VK_S | 83 | 0x53 | S |
| VK_T | 84 | 0x54 | T |
| VK_U | 85 | 0x55 | U |
| VK_V | 86 | 0x56 | V |
| VK_W | 87 | 0x57 | W |
| VK_X | 88 | 0x58 | X |
| VK_Y | 89 | 0x59 | Y |
| VK_Z | 90 | 0x5A | Z |
| VK_LWIN | 91 | 0x5B | Left Windows key |
| VK_RWIN | 92 | 0x5C | Right Windows key |
| VK_APPS | 93 | 0x5D | Application/Menu key |
| VK_NUMPAD0 | 96 | 0x60 | Numpad 0 |
| VK_NUMPAD1 | 97 | 0x61 | Numpad 1 |
| VK_NUMPAD2 | 98 | 0x62 | Numpad 2 |
| VK_NUMPAD3 | 99 | 0x63 | Numpad 3 |
| VK_NUMPAD4 | 100 | 0x64 | Numpad 4 |
| VK_NUMPAD5 | 101 | 0x65 | Numpad 5 |
| VK_NUMPAD6 | 102 | 0x66 | Numpad 6 |
| VK_NUMPAD7 | 103 | 0x67 | Numpad 7 |
| VK_NUMPAD8 | 104 | 0x68 | Numpad 8 |
| VK_NUMPAD9 | 105 | 0x69 | Numpad 9 |
| VK_MULTIPLY | 106 | 0x6A | Numpad * |
| VK_ADD | 107 | 0x6B | Numpad + |
| VK_SUBTRACT | 109 | 0x6D | Numpad - |
| VK_DECIMAL | 110 | 0x6E | Numpad . |
| VK_DIVIDE | 111 | 0x6F | Numpad / |
| VK_F1 | 112 | 0x70 | F1 |
| VK_F2 | 113 | 0x71 | F2 |
| VK_F3 | 114 | 0x72 | F3 |
| VK_F4 | 115 | 0x73 | F4 |
| VK_F5 | 116 | 0x74 | F5 |
| VK_F6 | 117 | 0x75 | F6 |
| VK_F7 | 118 | 0x76 | F7 |
| VK_F8 | 119 | 0x77 | F8 |
| VK_F9 | 120 | 0x78 | F9 |
| VK_F10 | 121 | 0x79 | F10 |
| VK_F11 | 122 | 0x7A | F11 |
| VK_F12 | 123 | 0x7B | F12 |
| VK_F13 | 124 | 0x7C | F13 |
| VK_F14 | 125 | 0x7D | F14 |
| VK_F15 | 126 | 0x7E | F15 |
| VK_F16 | 127 | 0x7F | F16 |
| VK_F17 | 128 | 0x80 | F17 |
| VK_F18 | 129 | 0x81 | F18 |
| VK_F19 | 130 | 0x82 | F19 |
| VK_F20 | 131 | 0x83 | F20 |
| VK_F21 | 132 | 0x84 | F21 |
| VK_F22 | 133 | 0x85 | F22 |
| VK_F23 | 134 | 0x86 | F23 |
| VK_F24 | 135 | 0x87 | F24 |
| VK_NUMLOCK | 144 | 0x90 | Num Lock |
| VK_SCROLL | 145 | 0x91 | Scroll Lock |
| VK_LSHIFT | 160 | 0xA0 | Left Shift |
| VK_RSHIFT | 161 | 0xA1 | Right Shift |
| VK_LCONTROL | 162 | 0xA2 | Left Ctrl |
| VK_RCONTROL | 163 | 0xA3 | Right Ctrl |
| VK_LMENU | 164 | 0xA4 | Left Alt (also VK_LALT) |
| VK_RMENU | 165 | 0xA5 | Right Alt (also VK_RALT) |
| VK_BROWSER_BACK | 166 | 0xA6 | Browser Back |
| VK_BROWSER_FORWARD | 167 | 0xA7 | Browser Forward |
| VK_VOLUME_MUTE | 173 | 0xAD | Volume Mute |
| VK_VOLUME_DOWN | 174 | 0xAE | Volume Down |
| VK_VOLUME_UP | 175 | 0xAF | Volume Up |
| VK_MEDIA_NEXT_TRACK | 176 | 0xB0 | Media Next Track |
| VK_MEDIA_PREV_TRACK | 177 | 0xB1 | Media Previous Track |
| VK_MEDIA_STOP | 178 | 0xB2 | Media Stop |
| VK_MEDIA_PLAY_PAUSE | 179 | 0xB3 | Media Play/Pause |
| VK_OEM_1 | 186 | 0xBA | ;: key (US) |
| VK_OEM_PLUS | 187 | 0xBB | =+ key (US) |
| VK_OEM_COMMA | 188 | 0xBC | , key |
| VK_OEM_MINUS | 189 | 0xBD | -_ key (US) |
| VK_OEM_PERIOD | 190 | 0xBE | . key |
| VK_OEM_2 | 191 | 0xBF | /? key |
| VK_OEM_3 | 192 | 0xC0 | `~ key |
| VK_OEM_4 | 219 | 0xDB | [{ key |
| VK_OEM_5 | 220 | 0xDC | \| key |
| VK_OEM_6 | 221 | 0xDD | ]} key |
| VK_OEM_7 | 222 | 0xDE | '" key |

与 PECMD 命令配合使用示例：
```
HKEY Alt+#0x41,TEAM MESS Pressed Alt+A!
HOTK Ctrl+Win+#0x53,MYFUNC         // Ctrl+Win+S
WAIT -cont -3000,&key               // 等待 3 秒按键，将码存入 &key
SEND #0x0D                          // 发送 Enter 键
```
