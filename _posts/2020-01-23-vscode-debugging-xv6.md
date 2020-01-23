---
title: "使用 VS Code 调试 Xv6"
tags: tutorials
---

[Xv6](https://pdos.csail.mit.edu/6.828/2019/xv6.html) 是MIT 开发的一个教学使用的类 Unix 操作系统，[开源](https://github.com/mit-pdos/xv6-public)并且配有“[课本](https://pdos.csail.mit.edu/6.828/2019/xv6/book-riscv-rev0.pdf)”。相比大而全的教科书，它更加精简，只包含部分核心的操作系统思想和概念；又更加具体，完全可以从源码层面去看如何简单地实现进程、页表、锁这些基础设施，很适合在离开学校之后拿来补课。

除了课本和注释，阅读和理解一份源码最有效的方式就是把它跑起来、调试起来，下面会说明怎么用在 macOS 下运行和使用 VS Code 调试 Xv6 内核。

### 如何运行 Xv6

课程官网或一些博客都能找到关于运行 Xv6 的介绍，这里就只简单写一下步骤。

1. 把[代码](https://github.com/mit-pdos/xv6-public.git) 下载到 Mac 上；
2. 安装 Xcode 命令行工具和 Homebrew（如果还没有的话）；
3. 按照[官网的指引](https://pdos.csail.mit.edu/6.828/2019/tools.html)安装运行 Xv6 依赖的 qemu、riscv-tools；
4. 这时，`cd` 到代码目录，`make qemu` 应该就可以编译和运行了。

<!--more-->

![run xv6](/assets/xv6/run-xv6.png)

### 如何调试 Xv6

课本第二章章末练习中介绍到的 `make qemu-gdb` 就是用于调试的命令，提供启动时等待 debugger attach 的能力，在另一个命令行窗口中运行 `gdb` 时，gdb 就会 attach 到等待运行的 Xv6 上。

新版本的 macOS 没有自带 gdb，所以需要先安装

```shell
brew install gdb
```

但这时由于 riscv-gnu-toolchain 和 gdb link 了同一个文件 `jit-reader.h`，会导致 gdb 安装中断，可根据 brew 的提示运行

```shell
brew link --overwrite gdb
```

这时再运行 `make qemu-gdb`，应该就会看到命令行进入等待 gdb attach 的状态，在其他窗口中运行 `gdb`，就可以按练习中的方式设置断点和其他调试了。

### 如何使用 VS Code 调试 Xv6

如果用顺手的 IDE 取代命令行的 gdb 来做 attach ，调试会方便得多。

为了方便编译，先添加 `.vscode/task.json`，添加后可以通过 command+shift+B 在 VS Code 的 terminal 中运行 `make qemu-gdb`

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "compile and run xv6 in debug mode",
            "command": "bash",
            "args": [
                "-c",
                "make && make qemu-gdb"
            ],
            "presentation": {
                "echo": true,
                "reveal": "always",
                "focus": true,
                "panel": "new",
                "showReuseMessage": true,
                "clear": true
            },
            "group": {
                "kind": "build",
                "isDefault": true
            }
        }
    ]
}
```

再配置 `.vscode/launch.json` 文件如下，就可以在 VS Code 的 debug 面板中看到相应的运行选项。需要注意的是，这个配置只负责 attach，点击 run 之前需要先运行上面的 task 或手动在命令行跑 `make qemu-gdb`，并把 `miDebuggerServerAddress` 改成 terminal 中最后输出的端口号

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Remote Debug Xv6",
            "type": "cppdbg",
            "request": "launch",
            "miDebuggerServerAddress": ":25502",
            "program": "${workspaceRoot}/kernel/kernel",
            "cwd":".",
            "MIMode": "gdb",
            "externalConsole": true,
            "serverLaunchTimeout": 3000,
            "stopAtEntry": true,
            "logging": {
                "engineLogging": true
            },
        }
    ]
}
```

在 `kernel/main.c` 中打个断点，点 run 就会看到 Xv6 的运行被中断了。🎉

![debug-xv6](/assets/xv6/debug-xv6.png)

### One more thing

再对其他函数做进一步的调试会发现，运行时代码里的很多符号被优化掉了，原因是 Makefile 没有对调试模式的编译选项做额外处理，由于是出于学习目的跑 Xv6，为了方便直接把两种运行模式的编译优化都关掉

```shell
# Makefile line 57
CFLAGS = -Wall -Werror -O -fno-omit-frame-pointer -ggdb
```

这时候再编译会先后报几个问题，根据错误信息一一处理掉即可
```shell
# Makefile line 60
CFLAGS += -ffreestanding -fno-common -mno-relax
```
```c
// user/usertests.c line 2123
static const struct test {
```
```c
// user/usertests.c line 2186
for (int i = 0; (tests+i)->s != 0; i++) {
  if((n == 0) || strcmp((tests+i)->s, n) == 0) {
    if(!run((tests+i)->f, (tests+i)->s))
      fail = 1;
  }
}
```
可参考[这个提交](https://github.com/lihaoyuan/xv6-riscv/commit/b2930ce6bed375f252bb73ce7069ecbf96100345)。