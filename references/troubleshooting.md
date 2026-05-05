# PECMD 故障排除指南

基于实际调试经验整理的常见问题与解决方案。

---

## 1. DLL 调用返回空或结果异常

**症状：** `CALL $--qd --ret:&&ret user32.dll,...` 返回空值，或缓冲区输出参数为空。

**已知问题（实测确认，32-bit / 64-bit 行为一致）：**

| 问题 | 原因 | 修复 |
|------|------|------|
| 点语法返回空 | `Func.#Param` 格式不解析 | 改用逗号分隔：`Func,#Param` |
| 整数参数返回 0 | 整数缺少 `#` 前缀 | 加 `#` 前缀：`#0`、`#%&var%` |
| 字符串长度多 1 | 无 `--qd` 时含 null 终止符 | 加 `--qd` 标志 |
| 缓冲区输出为空 | `*` 前缀不加 `&` 时输出不回传 | 改用 `*&buf` 形式传递 PE 变量地址 |
| `*` 前缀传 SET$# 崩溃 | 原始字节缓冲区指针无效 | 改用 `SET$#` + `*&buf` 配合使用 |
| GetProcAddress 返回 0 | PECMD 实现问题 | 直接用函数名调用 |

**排查步骤：**

```wcs
// 步骤1：确认整数返回 API 工作（用标准输出，不用 MESS）
CALL $ --ret:&&r kernel32.dll,GetTickCount
WRIT -,$+0,GetTickCount: [%&r%]
FIND $%&r%=,
{
    WRIT -,$+0,ERROR: DLL calls not working
}

// 步骤2：确认字符串参数
CALL $ --qd --ret:&&r user32.dll,FindWindowW,$Progman,#0
WRIT -,$+0,Progman HWND: [%&r%]
```

**替代方案（推荐）：**
| 需求 | 替代命令 | 说明 |
|----------|----------|------|
| 枚举窗口 | `FIND --wid*@ &var` | 返回 HWND、类名、标题 |
| 枚举进程 | `FIND --pid*@ &var` | 返回 PID、路径等 |
| 注册表读写 | `REGI` | 读写系统配置 |
| 外部命令 | `EXEC* &out=!cmd /c ...` | 捕获命令输出 |
| 磁盘空间 | `IFEX C:\=?,&var` | 查询可用空间 |
| 内存查询 | `IFEX MEMU=?,&var` | 查询可用内存 |

---

## 2. FORX 循环变量引用失败

**症状：** 循环体内变量为空或未找到。

**原因：** 使用了 `%&&item%`（双&）引用循环变量。

**正确写法：**

```wcs
// ✅ 正确：定义用 &&item，引用用 %&item%（单&）
FORX *NL &list,&&line,
{*
    SET &value=%&line%
    WRIT -,$+0,Value: %&value%
}

// ❌ 错误：不要用 %&&item%
FORX *NL &list,&&line,
{*
    SET &value=%&&line%       // 变量未找到！
}
```

**规则总结：**
- `FORX ... &&item, { }` — `&&` 声明为局部 PE 变量
- `%&item%` — 循环体内用单&引用值
- `%&&item%` — **不会**工作，会报变量未找到

---

## 3. MSTR 字段提取返回空

**症状：** `MSTR &&field,<N>%&data%` 返回空值。

**原因：**
- 源数据的分隔符与 MSTR 默认分隔符不匹配
- 字段索引超出数据范围
- 源变量为空
- 使用了 `<N*>` 但期望 "N到末尾"（实际 `*` 仅返回单字段，用 `-` 代替）

**MSTR 范围语法（实测确认）：**
| 语法 | 含义 | 示例（"a b c d e"） |
|------|------|---------------------|
| `<N>` | 单字段 | `<3>` → "c" |
| `<N*>` | 字段 N 到末尾（同 `<N->`） | `<3*>` → "c d e" |
| `<N->` | 字段 N 到末尾 | `<3->` → "c d e" |
| `<-N>` | 从末尾反向索引 | `<-1>` → "e" |
| `<~N>` | 剥离引号 | `<~1>` → 去掉外层引号 |

**排查步骤：**

```wcs
// 步骤1：打印源变量确认有数据
WRIT -,$+0,Source data: [%&data%]

// 步骤2：尝试提取第1个字段
MSTR &&f1,<1>%&data%
WRIT -,$+0,Field 1: [%&f1%]

// 步骤3：如果默认空格分隔不行，指定分隔符
MSTR -delims:09 &&f1,<1>%&data%       // 09 = TAB
MSTR -delims:0a &&f1,<1>%&data%       // 0A = 换行
```

**替代方案 — 用 FORX 逐字段拆分：**

```wcs
SET &fIdx=0
FORX * %&data%,&&field,
{*
    CALC &fIdx=%&fIdx% + 1
    // 手动提取第 N 个字段
    IFEX $%&fIdx%=1, SET &field1=%&field%
    IFEX $%&fIdx%=2, SET &field2=%&field%
    IFEX $%&fIdx%=3, SET &field3=%&field%
}
```

---

## 4. LPOS 查找位置不正确

**症状：** `LPOS` 返回 0 或预期外的值。

**原因：** 参数顺序错误。

**正确语法：** `LPOS &&pos=needle,,haystack`（查找目标在前，源字符串在后）

```wcs
// ✅ 正确
LPOS &&pos=CabinetWClass,,%&line%        // 在 line 中查找 CabinetWClass

// ❌ 错误（参数反了）
LPOS &&pos=%&line%,,CabinetWClass        // 不会报错但结果不对
```

---

## 5. FIND --wid\*@ 输出解析

**症状：** 不知道如何从窗口列表中提取窗口信息。

**输出格式（TAB 分隔）：** `序号 窗口ID 控件ID 父窗口ID 线程ID 进程ID 类型 标题`

**正确解析方法（用 MSTR* 按 TAB 分割）：**

```wcs
FIND --wid*@ &allWin,
FORX *NL &allWin,&&line,
{*
    MSTR* &&idx,&&hwnd,&&ctlid,&&parent,&&tid,&&pid,&&type,&&title=<1><2><3><4><5><6><7><8>%&line%
    WRIT -,$+0,HWND=%&hwnd%  Type=%&type%  Title=%&title%
}
```

---

## 6. SED 替换行为异常

**症状：** `SED &r=0,pattern,replacement,%&source%` 返回意外结果。

**关键事实（实测确认）：**
- SED 默认使用正则模式。`.` 匹配任意字符，非字面量句号。使用标志字符 `*` 可切换为字面量模式。
- 匹配字面量句号：正则模式用 `\.`，或用 `*` 标志切到字面量模式直接写 `.`。
- count=0 替换全部匹配，count=-N 替换后 N 个匹配。

```wcs
// ✅ 替换字面量句号
SED &result=0,\.,X,%&source%              // 所有 "." → "X"

// ❌ 错误：. 匹配任意字符，整个字符串被替换
SED &result=0,.,X,%&source%               // 每个字符 → "X"

// ✅ 提取扩展名
SED &ext=-1,.*\.,,  ,%&filename%          // 获取最后一个 "." 后的内容

// ✅ 简单文本替换（正则中普通字符仍为字面匹配）
SED &result=0,hello,world,%&source%       // "hello" → "world"
```

**count 含义：**
| count | 行为 |
|-------|------|
| `0` | 替换全部匹配 |
| `N` | 替换前 N 个匹配 |
| `-N` | 替换后 N 个匹配 |
| 默认 | 替换第 1 个匹配（help.txt: "替换次数默认1"） |

---

## 7. SET 变量追加问题

**症状：** `SET<` 或 `SET &list=%&list% %&val%` 行为异常。

**推荐写法：**

```wcs
// ✅ 可靠的变量追加方式
SET &expWinList=%&expWinList% %&hwnd%

// ⚠ SET< 可能在某些版本不工作，不推荐使用
```

---

## 8. 空字符串检测

**症状：** 不确定如何判断变量是否为空。

```wcs
// 方法1：FIND $ 比较
FIND $%&var%=,
{
    WRIT -,$+0,变量为空
}

// 方法2：FIND * 惯用法
FIND *=var,                      // 变量为空时执行后续命令
FIND *<>var,                     // 变量非空时执行后续命令
```

---

## 9. 注释不被识别

**症状：** `//comment` 被当作命令执行。

**原因：** `//` 和 `;` 在行尾时前面必须有空格。行首的 `//` 如果紧跟非空格字符可能不被识别。

```wcs
// ✅ 正确（行首有空格或独立一行）
SET &a=1 // 这是注释

// ❌ 可能有问题
//这个注释可能不被识别
```

---

## 10. 变量展开时机问题

**症状：** 循环中变量值不随迭代变化。

**原因：** PECMD 可能在循环开始前就展开了变量。

**解决方案：** 使用 `^` 前缀推迟展开到执行时。

```wcs
// ❌ 变量可能在循环开始前全部展开
LOOP #%I%<=5, { MESS %I% | CALC &I=%I% + 1 }

// ✅ 使用 ^ 前缀推迟展开
LOOP #%I%<=5, { ^MESS %&I% | CALC &I=%I% + 1 }
```

---

## 11. CALC log() 需要两个参数

**症状：** `CALC &r=log(100)` 返回 0 而非 2。

**原因：** `log()` 是双参数函数 `log(底数, 值)`，只传 1 个参数时返回 0。

**解决方案：** 使用 `log(底数, 值)` 或用 `lg()` / `ln()` 替代。

```wcs
// ❌ log() 只传 1 个参数 → 返回 0
CALC &r=log(100)                          // 0

// ✅ log() 双参数形式
CALC &r=log(10,100)                       // 2
CALC &r=log(2,8)                          // 3

// ✅ lg() 返回 log base 10（更简洁）
CALC &r=lg(100)                           // 2
CALC &r=lg(1000)                          // 3

// ✅ ln() 返回自然对数
CALC &r=ln(100)                           // 4.605...
```

---

## 12. WRIT/LOOP 嵌套在 FIND/IFEX/TEAM 中失败

**症状：** `WRIT` 或 `LOOP` 命令在 `FIND`/`IFEX`/`TEAM` 内不执行或行为异常。

**原因：** `WRIT` 和 `LOOP` 必须位于单独一行，不能嵌套在 `FIND`、`IFEX`、`TEAM` 命令内部。与 `_SUB` 有相同的限制。

```wcs
// ❌ 错误：WRIT 嵌套在 TEAM 内
TEAM FIND $%&a%=hello, WRIT -,$+0,found

// ✅ 正确：用 ! 分隔的命令链
FIND $%&a%=hello, TEAM WRIT -,$+0,found
```

---

## 13. 线程中 PE 变量行为不一致

**症状：** 线程中修改的变量有时影响父线程，有时不影响。

**原因：** PE 变量的复制/共享取决于创建线程的上下文：
- **持久栈上下文**（窗口 `_SUB`、`*` 函数）：变量**共享**——子线程修改直接影响父
- **非持久上下文**（普通函数、`{}` 块）：变量**复制**——子线程修改不影响父

```wcs
// 共享场景：窗口内创建线程
_SUB MainWindow
    SET &sharedVar=hello
    THREAD* CALL Worker      // sharedVar 是共享的
_END

// 复制场景：普通函数内创建线程
_SUB MyFunc
    SET &localVar=hello
    THREAD* CALL Worker      // localVar 是复制的
_END

// 安全跨线程通信用全局 PE 变量
SET &::threadResult=
THREAD* CALL Worker
// ...稍后读取 %&::threadResult%
```

---

## 调试技巧总结

| 技巧 | 说明 |
|------|------|
| `WRIT -,$+0,text` | 输出到 stdout，最可靠的调试方式 |
| `WRIT C:\debug.log,$+0,text` | 输出到日志文件 |
| 简化测试 | 先测试最小代码片段，确认基础功能正常 |
| 打印变量值 | 对每个关键变量使用 WRIT 输出确认值 |
| 版本检测 | 先测试 DLL 调用是否可用，再决定使用哪种方案 |
| 字段索引 | 用 `FORX *` + 计数器逐字段确认分隔和索引正确 |

---

*基于 explorer_tree_probe.wcs 移植任务的调试经验整理，经 PECMD2012 32-bit 和 64-bit 交叉验证*
