# Lab 0: Linux 内核调试

!!! danger "DDL"

    本实验文档尚未正式发布。

## 实验简介

欢迎来到《操作系统》课程实验。按照培养方案，你已经完成了《数字逻辑设计》《计算机组成》《计算机体系结构》等硬件课程的学习，了解了 RISC-V 指令集架构和计算机系统的基本组成。现在，我们将进入软件部分，动手实现一个运行在 RISC-V 架构下的简易操作系统内核。

在 Lab 0 中，我们将学习下列内容，完成环境搭建：

- **使用 Git 管理源代码：**为了方便助教查看同学们的代码修改，本教学班统一使用 [ZJU Git](https://git.zju.edu.cn/) 进行代码管理。请同学们注册 ZJU Git 账号，使用指定的仓库进行实验、提交代码。
- **使用 Docker 容器：**为了避免同学们在不同操作系统和环境下遇到各种各样的问题，我们将实验环境打包成 [Docker](https://www.docker.com/) 容器镜像，免去同学们自行搭建环境的麻烦。
- **使用 QEMU 模拟器：**同学们使用的一般是 x86-64 或 Arm 架构的设备，而本课程实现的内核使用 RISC-V 汇编指令和 C 语言编写。我们将使用 [QEMU](https://www.qemu.org/)、[Spike](https://github.com/riscv-software-src/riscv-isa-sim) 等虚拟机、模拟器来模拟 RISC-V 架构的计算机系统，由它们提供对应的 CPU、内存、外设等环境，供我们运行和调试内核。
- **使用交叉编译工具链**：特定架构上的编译器等工具一般只生成对应架构的二进制文件，生成其他架构的二进制文件的过程称为交叉编译。[RISC-V GNU Compiler Toolchain](https://github.com/riscv-collab/riscv-gnu-toolchain) 支持在 x86-64 或 Arm 架构的机器上编译、调试 RISC-V 架构的程序。
- **使用 GDB 调试器：**在内核开发过程中，调试是必不可少的环节。我们将使用 [GDB](https://www.gnu.org/software/gdb/) 来调试内核代码，定位和修复问题。

接下来的实验默认同学们：

- 拥有 Linux/macOS 环境，具备 Linux 基础操作能力
- 了解 Git、Makefile 的基本用法

## 实验要求

本实验非常简单，无需写代码，无需提交报告。

后续实验的文档会提供尽可能详细的指导，但你仍有很多问题需要回到原始文档、标准规范中仔细求证。**为了锻炼同学们阅读文档解决问题的能力，我们适当删减了本实验的指导，而是给出了一些原始文档链接，要求同学们寻找答案。**

验收时，请你现场演示使用 QEMU 启动内核，并使用 GDB 打断点、查看各类信息。

!!! question "考点"

    本文档中标记为考点的方块是希望同学们能掌握的知识点，会在验收时提问。

## 实验步骤

### 使用 Git 管理源代码

你需要先在 ZJU Git 上注册账号。如果你没有使用过 ZJU Git，请参考 [ZJU Git 101](https://www.pages.zjusct.io/git101/)。

我们在 ZJU Git 上为每个同学创建了私有的代码仓库。实验过程将涉及 Git 的进阶用法，下图展示了完整的工作流和相应命令：

![git.drawio](lab0.assets/git.drawio)

```bash
git clone git@zju.edu.cn:os/2025/jijiangming/<你的学号>.git
git remote add upstream https://git.zju.edu.cn/os/code.git
git commit
git push
git checkout -b <new-lab>
git fetch upstream
git merge upstream/<new-lab>
```

### 使用 Docker 容器

如果你尚未安装 Docker：

- Linux 和 WSL 环境请参考 [Docker CE | ZJU Mirror](https://mirrors.zju.edu.cn/docs/docker-ce/)
- macOS 推荐使用 Docker Desktop，也可以用 Homebrew 安装

实验代码库根目录下的 `Makefile` 将相关 Docker 命令封装成了 Makefile 目标。你可以：

- 运行 `make` 创建并启动容器
- ++ctrl+d++ 退出并关闭容器，此时你在容器内的更改会被保存，下次 `make` 进入容器时可以继续使用
- 如果你不小心搞坏了容器内的环境，运行 `make clean` 来删除容器，重新 `make` 运行一个新的容器

```text
$ make
docker compose create
docker compose start
[+] Running 1/1
 ✔ Container zju-os  Started               0.3s
docker compose exec -it zju-os /usr/bin/fish
Welcome to fish, the friendly interactive shell
Type help for instructions on how to use fish
root@zju-os /zju-os#
```

!!! tip

    代码库会被挂载到容器内的 `/zju-os/code` 目录下。**这意味着宿主机和容器共享代码库的文件，容器内对代码的修改会直接反映到宿主机上，反之亦然。**文件保存在宿主机，所以不会因为容器被删除而丢失。

!!! info "更多资料"

    - 如果你不了解 Docker：[What is a Container? | Docker](https://www.docker.com/resources/what-container)
    - `Dockerfile` 描述了容器镜像是如何构建的：[Dockerfile reference | Docker Docs](https://docs.docker.com/reference/dockerfile/)
    - `compose.yml` 描述了容器的运行参数：[Compose file reference | Docker Docs](https://docs.docker.com/reference/compose-file/)
    - 容器内默认使用 `fish`，这是一个比 `bash` 更友好的 shell，提供开箱即用的历史纪录、自动补全显示等功能：[fish shell](https://fishshell.com/)

### 使用交叉工具链




### 编译内核

容器内的 `/zju-os/linux-source-*` 是预先放置好的 Linux 内核源码：

```text
root@zju-os /zju-os# ls
code/  linux-source-6.16/
```

在容器中编译内核的基本命令如下：

```text
root@zju-os /zju-os# cd linux-source-6.16
root@zju-os /z/linux-source-6.16# make defconfig
root@zju-os /z/linux-source-6.16# make -j$(nproc)
  ...
  BUILD   arch/x86/boot/bzImage
Kernel: arch/x86/boot/bzImage is ready  (#1)
root@zju-os /z/linux-source-6.16# make distclean
```

如果你看到了 `Kernel: ... is ready` 这一行，说明你成功构建出了**当前架构**的内核。

!!! question "考点"

    请你运行 `make help`，了解上面运行的 `defconfig`、`distclean` 等 target 的含义。此外：

    - 如何开启构建过程的详细输出？当构建失败时，你很可能需要查看详细的编译命令。

!!! info "更多资料"

    - 内核发行说明：[kernel.org/doc/Documentation/admin-guide/README.rst](https://www.kernel.org/doc/Documentation/admin-guide/README.rst)
    - Debian 发行版内核手册：[Chapter 4. Common kernel-related tasks](https://www.debian.org/doc/manuals/debian-kernel-handbook/ch-common-tasks.html)
    - Linux 内核的构建系统十分复杂：[Linux Kernel Makefiles — The Linux Kernel documentation](https://docs.kernel.org/kbuild/makefiles.html)

### 交叉编译内核

请阅读这篇简明的文档 [Embedded Handbook/General/Cross-compiling the kernel - Gentoo wiki](https://wiki.gentoo.org/wiki/Embedded_Handbook/General/Cross-compiling_the_kernel)，了解内核交叉编译的步骤。

!!! question "考点"

    - 使用哪两个变量来指定目标架构？这两个变量的值在哪里找？
    - 如何在命令行中为 `make` 指定变量的值？

如果看到 `Kernel: arch/riscv/boot/Image is ready` 这一行，说明成功构建出了 RISC-V 架构的内核。使用 `file` 命令来验证它是否为 RISC-V 架构的内核：

```shell
root@zju-os /z/linux-source-6.16# file arch/riscv/boot/Image
arch/riscv/boot/Image: Linux kernel RISC-V boot executable Image, little-endian
root@zju-os /z/linux-source-6.16# file vmlinux
vmlinux: ELF 64-bit LSB executable, UCB RISC-V, RVC, soft-float ABI, version 1 (SYSV), statically linked, BuildID[sha1]=657ad614b9ebd9723a2d5ee0a25505df3a43b918, not stripped
```

### 使用 QEMU 运行内核

回到代码仓库，使用下面的命令启动 QEMU 运行构建好的内核，你应当能在其中运行一些简单的命令：

```shell
root@zju-os /zju-os/code# make qemu
```

退出 QEMU 的方法为：使用 <kbd>Ctrl+A</kbd>（mac 上为 <kbd>control+A</kbd>），**松开**后再按下 <kbd>X</kbd> 键即可退出 QEMU。

!!! tip

    仅有一个内核镜像是无法启动 Linux 的，你还需要一个根文件系统（root filesystem），其中包含了 Linux 启动后需要的各种文件和命令，包括与你交互的 Shell 程序。仓库根目录下的 `rootfs.img` 就是一个已经构建好的根文件系统镜像。

!!! info "更多资料"

    - QEMU 命令行各个选项的作用：[QEMU User Documentation — QEMU documentation](https://www.qemu.org/docs/master/system/qemu-manpage.html)
    - 实验使用的 `rootfs.img` 制作方法：[FOSDEM 2019 - Buildroot for RISC-V](https://archive.fosdem.org/2019/schedule/event/riscvbuildroot/attachments/slides/3040/export/events/attachments/riscvbuildroot/slides/3040/FOSDEM_2019_Buildroot_RISCV.pdf)

### 系统启动过程

### GDB 调试内核

现在需要打开两个终端，一个运行 QEMU，另一个运行 GDB 进行调试。你有几种方案：

- 在容器内使用 [tmux](https://github.com/tmux/tmux) 等终端复用工具。可参考 [Tmux 使用教程 - 阮一峰的网络日志](https://www.ruanyifeng.com/blog/2019/10/tmux.html) 或 [Home · tmux/tmux Wiki](https://github.com/tmux/tmux/wiki)。
- 在宿主机上打开两个终端窗口，均执行 `make` 进入容器。

分别在两个终端运行 `make qemu-debug` 和 `make gdb`，你会看到 QEMU 启动后停在了第一条指令处：

```text
```

接下来，请你任选资料阅读：

- [GDB Command Reference - Index page](https://visualgdb.com/gdbreference/commands/)
- [Debugging with GDB](https://sourceware.org/gdb/current/onlinedocs/gdb)
- [100个gdb小技巧](https://wizardforcel.gitbooks.io/100-gdb-tips/content/)
- [gdb 相关使用备忘 - 鹤翔万里的笔记本](https://note.tonycrane.cc/cs/tools/gdb/)

了解下列命令的含义，并进行实操：

```text
layout asm
start [Arguments]
break [Function Name]
break *[Address]
break [...] if [Condition]
continue [Repeat count]
stepi [Repeat count]
backtrace
finish
info register [Register name]
print [Expression]
x /[Length][Format] [Address expression]
quit
```

一些信息：

- Linux 内核启动的函数名为 `start_kernel`，你可以在该函数处打断点。

!!! question "考点"

    - 使用 GDB 显示
    - **断点：**设置断点、查看断点、删除断点
    - **调试：**单指令执行、逐过程执行、结束当前函数、继续执行
    - **查看：**汇编代码、函数调用栈、变量、寄存器值、内存中的内容

!!! info "更多资料"

    - GDB 插件：[gdb-dashboard](https://github.com/cyrus-and/gdb-dashboard)
