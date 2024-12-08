---
title: 在 Windows 11 中基于 DOSBox 仿真器使用 Debug 调试 `8086` 汇编程序
date: 2024-12-08 12:45:11
tags:
- Assembly
- DOSBox
- Debug
- Windows 11
- 8086
---

## 环境准备

### 安装 `DOSBox` 仿真器

[`DOSBox`](https://www.dosbox.com/information.php?page=0) 是一个 `DOS` 仿真器。它可以在现代操作系统，如 `Windows 11` 中运行 `DOS` 程序，还可以模拟 `8086` 处理器，因此可以用来调试 `8086` 汇编程序。`DOSBox` 基于 [`SDL-library`](https://www.libsdl.org/) 跨平台开发库开发，通过 `OpenGL` 和 `Direct3D` 提供对鼠标、键盘、音频、图像等硬件的底层访问，因此可以运行在 `Windows`、`Linux` 和 `macOS` 等操作系统上。

可根据自己使用的开发操作系统，[下载对应的 `DOSBox` 安装包](https://www.dosbox.com/download.php?main=1)，安装完成后即可使用。

### 安装 `Debug` 调试器

`Debug` 是 `DOS` 系统自带的调试工具，可以用来调试 `8086` 汇编程序，`Windows 10` 默认已经移除。在 `DOSBox` 中，可以直接使用 `Debug` 命令来启动调试工具。

下载 `debug.exe` 后，将其放到本地目录下，如 `D:\workspace\github.com\yobol\assembly-study`。使用管理员身份打开 `DOSBox` 安装目录下的 `DOSBox 0.74-3 Options.bat` 文件，在文件末尾添加以下内容：

```
mount c D:\workspace\github.com\yobol\assembly-study
c:
```

打开 `DOSBox`，执行 `debug` 命令即可启动 `Debug` 调试器。

### 安装 `VSCode` - `MASM/TASM` 插件

[MASM/TASM](https://gitee.com/dosasm/masm-tasm/tree/main/masm-tasm) 是 `VSCode` 的一个[插件](https://marketplace.visualstudio.com/items?itemName=xsro.masm-tasm)，可实现对 `DOSBox` 等汇编工具的快速调用。

注：`MASM（Microsoft Macro Assembler）` 是微软公司开发的宏汇编器，而 `TASM（Turbo Assembler）` 是 `Borland` 公司开发的宏汇编器。`MASM` 主要用于微软的操作系统和开发环境中，而 `TASM` 则更为适用于 `DOS` 环境下的编程。现在，`TASM` 已经成为 `MASM` 的兼容版本，用户可以互换使用。

添加如下配置：

```json
{
    "masmtasm.ASM.mode": "workspace",
    "masmtasm.ASM.emulator": "jsdos",
    "masmtasm.ASM.assembler": "TASM",
    "masmtasm.ASM.actions": {
        "TASM": {
            "baseBundle": "<built-in>/TASM.jsdos",
            "before": [
                "set PATH=C:\\TASM"
            ],
            "run": [
                "TASM ${file}",
                "TLINK ${filename}",
                ">${filename}"
            ],
            "debug": [
                "TASM /zi ${file}",
                "TLINK /v/3 ${filename}.obj",
                "copy C:\\TASM\\TDC2.TD TDCONFIG.TD",
                "TD -cTDCONFIG.TD ${filename}.exe"
            ]
        }
    }
}
```

确保使用插件默认集成的 `jsdos`，并且会将当前 `workspace` 作为 `D:` 挂载进去。如果想使用 `Debug` 工具，因为默认不会使用本地 `DOSBox` 配置，所以需要确保 `debug.exe` 放置在当前 `workspace` 目录下。

## 常用命令

### `Debug` 命令

- `r`：显示/修改 CPU 寄存器的内容，如 `r ax`
- `d`：显示内存中的内容，如 `d 段地址:偏移地址`、`d 段地址:偏移地址 结尾偏移地址`
- `e`：修改内存中的内容，如 `e 段地址:偏移地址`（空格切换下一字节）、`e 段地址:偏移地址 数据 数据 ...`
- `u`：将内存中机器指令翻译成汇编指令（反汇编）
- `g`：运行程序
- `t`：单步执行
- `a`：以汇编指令的格式向内存中写入一条机器指令
- `p`：执行到下一个断点
- `q`：退出