# PECMD Pro Max

> PECMD2012 脚本编程的 AI 技能文件 —— 让 AI 编码助手一次写出正确的 WinPE 工具、GUI 和启动脚本。

简体中文 | [English](./README.en.md)

[![Version](https://img.shields.io/badge/version-1.5.1-blue)](./SKILL.md)
[![License](https://img.shields.io/badge/license-NonCopyRight-yellow)](./LICENSE)
[![PECMD](https://img.shields.io/badge/PECMD-v1.88+-green)](https://pecmd.net)

## 这是什么？

PECMD Pro Max 是一个 **AI 编码助手技能文件**，教会 AI 如何编写正确的 PECMD2012 脚本。PECMD 是 WinPE 指挥官 —— 一种用于 Windows PE 启动脚本、轻量级 GUI 系统工具、磁盘实用程序和预安装环境自动化的脚本语言与命令解释器。其变量作用域规则、十六进制内存模型和窗口系统以复杂著称。本技能编码了十多年的 PECMD 脚本知识，让 AI 助手能够一次写出生产就绪的 `.wcs` / `.wci` / `.wce` 文件，无需反复试错。

## 文件结构

```
pecmd-pro-max/
├── README.md                        ← 你正在阅读
├── README.en.md                     ← English version
├── SKILL.md                         ← 主技能文件 — 心智模型、变量系统、关键规则、陷阱（561行）
└── references/
    ├── commands-full.md             ← 110+ 条命令完整参考（1943行）
    ├── pecmd-gui.md                 ← 完整 GUI 控件与窗口系统参考（2178行）
    ├── pe-startup.md                ← WinPE 启动流程、环境限制、PE版本差异
    └── how-tos/
        ├── storage.md               ← 磁盘/分区/文件/注册表/设备 写法示例
        ├── system.md                ← 进程/线程/系统/工具/回调 写法示例
        ├── gui.md                   ← GUI 控件/窗口/绘制 写法示例
        └── net.md                   ← 网络/SOCK/COM/WMI 写法示例
```

## 核心特性

- **三层变量系统详解** —— 环境变量 vs PE-局部 vs PE-全局，间接引用、延迟展开、引用返回、十六进制/原始缓冲区分配、二进制比较/转换
- **110+ 命令参考** —— 每条 PECMD 命令包含完整语法、返回值、陷阱说明（含 BROW、TREE、LAMBDA、SBAR、IPAD、BLOCK、DTIM、SLID、SPIN）
- **按域组织的写法示例** —— 磁盘/分区、进程/线程、GUI控件/绘制、网络/COM，按主题独立文件，AI按需加载
- **完整 GUI 参考** —— 所有 25 种控件与子窗口（`LABE`、`EDIT`、`CHEK`、`RADI`、`IMAG`、`TABL`、`LIST`、`MEMO`、`TREE`...）、ENVI @ 控件属性大全、消息系统、窗口管理
- **WinPE 启动流程** —— 从 `winpeshl.exe` → `PECMD MAIN PECMD.INI` 直至 `EXPLORER.EXE`，包含 `WinXShell` 集成
- **50 个内置变量全表** —— 路径/Shell变量、进程/线程变量、窗口/GUI变量、脚本/运行时变量、命令结果变量
- **17 个 ENVI 运行时控制命令** —— EnviMode、ForceLocal、Alias、WndProc、memvar、LoadPlugin、DisX64、zero、Arg、Clipboard、EnviBroad、__arg、LoadEnvi、HelpColor、DeskTopFresh(@@)、TaskIcoMenu(@@)、EXPORTLOCAL
- **ForceLocal + EnviMode 默认值** —— 确保每个生成的脚本都使用最安全的变量作用域
- **代码组织规则** —— `_SUB` 声明时解析 vs 运行时调用、`SET^` Win32 回调绑定、`CALL @` 变体速查表
- **ENVI @ 控件属性大全** —— 通用属性、EDIT/ITEM/窗口专用、TABL 30+ 操作、TREE 10+ 操作、跨进程控制

## 快速上手

```wcs
#code=65001
ENVI^ EnviMode=1
ENVI^ ForceLocal=1
SET$ &NL=0d 0a

_SUB MainWin,L200T100W400H200,你好世界,-trap
    LABE Lbl1,L20T30W350H24,欢迎使用 PECMD
    ITEM Btn1,L150T120W80H28,关闭,CALL CloseMe
_END

_SUB CloseMe
    KILL \
_END

CALL @MainWin
```

## 安装

将 `pecmd-pro-max/` 文件夹复制到 AI 编码助手的技能目录：

- **Claude Code**: `~/.claude/skills/pecmd-pro-max/`
- **OpenCode**: `~/.config/opencode/skills/pecmd-pro-max/`

## 运行要求

- [Claude Code](https://claude.ai)、[OpenCode](https://opencode.ai) 或任何支持 agent 技能的兼容 AI 编码工具
- [PECMD2012](https://pecmd.net) v1.88+（执行生成脚本的解释器）

## 协议

**NonCopyRight** —— 本技能完全自由、开源、无限制。随意使用、修改、用于商业产品。无需署名。无任何限制。

---

源码仓库：[github.com/VirtualHotBar/PECMD-Pro-Max](https://github.com/VirtualHotBar/PECMD-Pro-Max)

ClawHub：[clawhub.ai/virtualhotbar/pecmd-pro-max](https://clawhub.ai/virtualhotbar/pecmd-pro-max)