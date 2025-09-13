# Lab 0: Linux 内核调试

!!! danger "DDL"

    本实验文档尚未正式发布。

## 实验简介

在 Lab 0 中，我们将学习下列内容，完成环境搭建：

- **使用 Git 管理源代码：**为了方便助教查看同学们的代码修改，本教学班统一使用 [ZJU Git](https://git.zju.edu.cn/) 进行代码管理。请同学们注册 ZJU Git 账号，使用指定的仓库进行实验、提交代码。
- **使用 Docker 容器：**为了避免同学们在不同操作系统和环境下遇到各种各样的问题，我们将实验环境打包成 [Docker](https://www.docker.com/) 容器镜像，免去同学们自行搭建环境的麻烦。
- **使用 QEMU 模拟器：**同学们使用的一般是 x86-64 或 Arm 架构的设备，而本课程实现的内核使用 RISC-V 汇编指令和 C 语言编写。我们将使用 [QEMU](https://www.qemu.org/)、[Spike](https://github.com/riscv-software-src/riscv-isa-sim) 等虚拟机、模拟器来模拟 RISC-V 架构的计算机系统，由它们提供对应的 CPU、内存、外设等环境，供我们运行和调试内核。
- **使用交叉编译工具链**：特定架构上的编译器等工具一般只生成对应架构的二进制文件，生成其他架构的二进制文件的过程称为交叉编译。[RISC-V GNU Compiler Toolchain](https://github.com/riscv-collab/riscv-gnu-toolchain) 支持在 x86-64 或 Arm 架构的机器上编译、调试 RISC-V 架构的程序。
- **使用 GDB 调试器：**在内核开发过程中，调试是必不可少的环节。我们将使用 [GDB](https://www.gnu.org/software/gdb/) 来调试内核代码，定位和修复问题。

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

其中 `git merge` 步骤可能遇到冲突。你需要理解冲突部分的代码的作用，并决定保留、合并或是手动整理代码。

我们推荐使用 VSCode 内置的 Git 面板进行相关操作。此外，你还可以安装 [Git Graph](https://marketplace.visualstudio.com/items?itemName=mhutchie.git-graph) 插件来可视化地查看分支、提交等信息。

!!! info "更多资料"

    - Pro Git 是一本优秀的 Git 教程：英文版 [Git](https://git-scm.com/book/en/)、中文版 [Pro Git 中文版（第二版）](https://www.progit.cn/)

### 使用 Docker 容器

如果你尚未安装 Docker：

- Linux 和 WSL 环境请参考 [Docker CE | ZJU Mirror](https://mirrors.zju.edu.cn/docs/docker-ce/)
- macOS 推荐使用 [Docker Desktop](https://docs.docker.com/desktop/setup/install/mac-install/) 或 [OrbStack](https://orbstack.dev/download) (可以用 [Homebrew](https://formulae.brew.sh/formula/docker) 直接安装)

实验代码库根目录下的 `Makefile` 将相关 Docker 命令封装成了 Makefile 目标。你可以：

- 运行 `make` 创建并启动容器
- ++ctrl+d++ 退出并关闭容器，此时你在容器内的更改会被保存，下次 `make` 进入容器时可以继续使用
- 如果你不小心搞坏了容器内的环境，运行 `make clean` 来删除容器，重新 `make` 运行一个新的容器

```console
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
    - `compose.yml` 描述了容器的运行参数：[Compose file reference | Docker Docs](https://docs.docker.com/reference/compose-file/)
    - `Dockerfile` 描述了容器镜像是如何构建的：[Dockerfile reference | Docker Docs](https://docs.docker.com/reference/dockerfile/)
    - 本课程的 `Dockerfile`：[tool/container/Dockerfile at main · ZJU-OS/tool](https://github.com/ZJU-OS/tool/blob/main/container/Dockerfile)
    - 容器内默认使用 `fish`，这是一个比 `bash` 更友好的 shell，提供开箱即用的历史纪录、自动补全显示等功能：[fish shell](https://fishshell.com/)
    - 如果未来你有进入 Docker 容器调试、使用其他工作目录的需求，可以在 VSCode 中使用：[Dev Containers 插件](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)

### 使用交叉编译工具链

也许你接触过下列编译器工具：

```text
gcc gdb objdump readelf as ...
```

它们对应的交叉编译工具带有格式为 `<目标架构>-<系统>-<套件名>-` 的前缀。比如 Linux 系统上用于交叉编译 RISC-V 64 架构的 GNU 工具前缀为 `riscv64-linux-gnu-`，使用方式与原版相同。以 GCC 为例：

```console
root@zju-os-code /z/code# riscv64-linux-gnu-gcc hello.c -o hello
root@zju-os-code /z/code# file hello
hello: ELF 64-bit LSB pie executable, UCB RISC-V, RVC, double-float ABI, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-riscv64-lp64d.so.1, BuildID[sha1]=963fa6ea0ba96c8e9b928927d0e0306355e326d5, for GNU/Linux 4.15.0, not stripped
```

!!! question "考点"

    用 C 写一个 Hello World 程序：

    - 生成它的 RISC-V 汇编代码
    - 将其编译为 RISC-V 可执行程序
    - 将 RISC-V 可执行程序反汇编

!!! info "更多资料"

    - GNU 二进制工具文档，介绍了实验中会用到的相关工具：[Binutils - GNU Project - Free Software Foundation](https://www.gnu.org/software/binutils/)

### 编译内核

容器内的 `/zju-os/linux-source-*` 是预先放置好的 Linux 内核源码：

```console
root@zju-os /zju-os# ls
code/  linux-source-6.16/
```

在容器中编译内核的基本命令如下：

```console
root@zju-os /zju-os# cd linux-source-6.16
root@zju-os /z/linux-source-6.16# make defconfig
root@zju-os /z/linux-source-6.16# make -j$(nproc)
  ...
  LD      vmlinux
  NM      System.map
  ...
  BUILD   arch/x86/boot/bzImage
Kernel: arch/x86/boot/bzImage is ready  (#1)
root@zju-os /z/linux-source-6.16# make distclean
```

如果你看到了 `Kernel: ... is ready` 这一行，说明你成功构建出了**当前架构**的内核。

!!! question "考点"

    - 运行 `make help`，了解上面运行的 `defconfig`、`distclean` 等 target 的含义
    - 如何开启构建过程的详细输出？当构建失败时，你很可能需要查看详细的编译命令
    - `Image` 和 `vmlinux` 是什么？有什么异同？

!!! info "更多资料"

    - 内核发行说明：[kernel.org/doc/Documentation/admin-guide/README.rst](https://www.kernel.org/doc/Documentation/admin-guide/README.rst)
    - Debian 发行版内核手册：[Chapter 4. Common kernel-related tasks](https://www.debian.org/doc/manuals/debian-kernel-handbook/ch-common-tasks.html)
    - Linux 内核的构建系统十分复杂，如果你有兴趣了解：[Linux Kernel Makefiles — The Linux Kernel documentation](https://docs.kernel.org/kbuild/makefiles.html)

### 交叉编译内核

请阅读这篇简明的文档 [Embedded Handbook/General/Cross-compiling the kernel - Gentoo wiki](https://wiki.gentoo.org/wiki/Embedded_Handbook/General/Cross-compiling_the_kernel)，了解内核交叉编译的步骤。

!!! question "考点"

    - 使用哪两个变量来指定目标架构？这两个变量的值在哪里找？
    - 如何在命令行中为 `make` 指定变量的值？

如果看到 `Kernel: arch/riscv/boot/Image is ready` 这一行，说明成功构建出了 RISC-V 架构的内核。使用 `file` 命令来验证它是否为 RISC-V 架构的内核：

```console
root@zju-os /z/linux-source-6.16# file arch/riscv/boot/Image
arch/riscv/boot/Image: Linux kernel RISC-V boot executable Image, little-endian
root@zju-os /z/linux-source-6.16# file vmlinux
vmlinux: ELF 64-bit LSB executable, UCB RISC-V, RVC, soft-float ABI, version 1 (SYSV), statically linked, BuildID[sha1]=657ad614b9ebd9723a2d5ee0a25505df3a43b918, not stripped
```

### 使用 QEMU 运行内核

回到代码仓库，使用 `make qemu` 启动 QEMU 运行构建好的内核：

```console
root@zju-os /zju-os/code# make qemu
...
Welcome to Buildroot
buildroot login:
```

buildroot 默认用户为 `root`，密码为空。你可以登录 Shell，试试这个极简的 RISC-V 系统能干些什么（它应该能联网）：

```console
buildroot login: root
# pwd
/root
# uname -a
Linux buildroot 6.16.3 #1 SMP Fri Sep 12 03:45:12 UTC 2025 riscv64 GNU/Linux
# wget http://www.baidu.com
Connecting to www.baidu.com (223.109.82.16:80)
saving to 'index.html'
index.html           100% |********************************|  2381  0:00:00 ETA
'index.html' saved
```

接下来学习 QEMU 操作：

- 现在与你交互的是 [QEMU 自带的终端复用器（Terminal Multiplexer）](https://www.qemu.org/docs/master/system/mux-chardev.html)
    - 它连接着 QEMU Monitor 和虚拟机的控制台（Console），默认情况下连接后者。
    - ++ctrl+a++ ++c++ 可以在两者间切换。
    - ++ctrl+a++ ++h++ 可以查看帮助。
    - ++ctrl+a++ ++x++ 可以退出 QEMU。
- [QEMU Monitor](https://qemu-project.gitlab.io/qemu/system/monitor.html) 可以控制、查看、调试虚拟机。

    它与下文介绍的 GDB 各有所长：QEMU Monitor 可以查看内存映射、TLB 等各类系统信息，而 GDB 专注于程序调试，主要是查看和控制代码运行。在后续实验中，如果你的代码有问题，可能导致 GDB 无法调试，而 QEMU Monitor 仍然可以使用。

    你可以在 Monitor 中运行 `help` 查看支持的命令。

    ```text
    QEMU 10.1.0 monitor - type 'help' for more information
    (qemu) help
    ...
    x /fmt addr -- virtual memory dump starting at 'addr'
    (qemu) info mem
    vaddr            paddr            size             attr
    ---------------- ---------------- ---------------- -------
    000055556ae09000 0000000081055000 0000000000001000 r-xu-a-
    (qemu) info registers

    CPU#0
    V      =   0
    pc       ffffffff80b57780
    mhartid  0000000000000000
    mstatus  0000000a000000a0
    hstatus  0000000200000000
    ```

!!! question "考点"

    使用 QEMU Monitor 进行下列操作：

    - 查看寄存器、内存树、内存映射、TLB、设备树、物理内存中的值
    - Linux 第一条指令位于物理内存 `0x80200000`，打印这条指令

!!! tip

    仅有一个内核镜像是无法运行系统的，你还需要一个根文件系统（root filesystem），其中包含了 Linux 启动后需要的各种文件，例如执行你输入的指令的 Shell 程序。容器内预置的 `/zju-os/rootfs.ext2` 就是一个已经构建好的根文件系统镜像。

!!! info "更多资料"

    - QEMU System 手册：[QEMU User Documentation — QEMU documentation](https://www.qemu.org/docs/master/system/qemu-manpage.html)
    - QEMU Monitor 手册：[QEMU Monitor Commands — QEMU documentation](https://www.qemu.org/docs/master/system/monitor.html)
    - `rootfs.ext2` 制作方法：[FOSDEM 2019 - Buildroot for RISC-V](https://archive.fosdem.org/2019/schedule/event/riscvbuildroot/attachments/slides/3040/export/events/attachments/riscvbuildroot/slides/3040/FOSDEM_2019_Buildroot_RISCV.pdf)

### QEMU 启动过程

对应到[OS 实验导读](intro.md)提及的 RISC-V 软件栈：

- **QEMU** 提供模拟的 RISC-V [**硬件环境**（CPU、内存、外设等）](https://www.qemu.org/docs/master/system/riscv/virt.html)
- **OpenSBI** 内置在 QEMU 中，作为 RISC-V 架构的**默认固件**，它运行在 **RISC-V M-Mode**
- 因为系统简单没有引导程序，固件直接启动内核，内核运行在 **RISC-V S-Mode**

整体流程如下图所示：

<figure markdown="span">
    ![boot.webp](lab0.assets/boot.webp)
    <figcaption>
    [OpenSBI Deep Dive - RISC-V International](https://riscv.org/wp-content/uploads/2024/12/13.30-RISCV_OpenSBI_Deep_Dive_v5.pdf)
    </figcaption>
</figure>

让我们结合输出信息来看

```text
...
   ____                    _____ ____ _____
  / __ \                  / ____|  _ \_   _|
 | |  | |_ __   ___ _ __ | (___ | |_) || |
 | |  | | '_ \ / _ \ '_ \ \___ \|  _ < | |
 | |__| | |_) |  __/ | | |____) | |_) || |_
  \____/| .__/ \___|_| |_|_____/|____/_____|
        | |
        |_|
Firmware Base               : 0x80000000
Domain0 Next Address        : 0x0000000080200000
Domain0 Next Arg1           : 0x0000000087e00000
Domain0 Next Mode           : S-mode
...
[    0.000000] Booting Linux on hartid 0
[    0.000000] Linux version 6.16.3 (root@zju-os-code) (riscv64-linux-gnu-gcc (Debian 15.2.0-3) 15.2.0, GNU ld (GNU Binutils for Debian) 2.45) #1 SMP Fri Sep 12 03:45:12 UTC 2025
```

- QEMU 作为 Loader
    - 将 OpenSBI 加载到内存中的 `0x80000000`
    - 将 Linux 内核加载到内存中的 `0x80200000`
- QEMU 内置的 OpenSBI 使用 `FW_DYNAMIC` 模式
    - QEMU 将 `next_addr` 等信息放在 `struct fw_dynamic_info` 结构体中，将该结构体的地址存放在 `a2` 寄存器中
    - OpenSBI 读取该信息，了解下一步要怎么做
- `Booting Linux on hartid 0` 是内核 [`start_kernel()`](https://github.com/torvalds/linux/blob/master/init/main.c#L903) 打印出的第一条日志

!!! info "更多信息"

    - QEMU RISC-V 启动实现：[qemu/hw/riscv/boot.c at master · qemu/qemu](https://github.com/qemu/qemu/blob/master/hw/riscv/boot.c)
    - OpenSBI 详解：[OpenSBI Deep Dive - RISC-V International](https://riscv.org/wp-content/uploads/2024/12/13.30-RISCV_OpenSBI_Deep_Dive_v5.pdf)

### GDB 调试内核

现在需要打开两个终端，一个运行 QEMU，另一个运行 GDB 进行调试。你可以：

- 在容器内使用 [tmux](https://github.com/tmux/tmux) 等终端复用工具。可参考 [Tmux 使用教程 - 阮一峰的网络日志](https://www.ruanyifeng.com/blog/2019/10/tmux.html) 或 [Home · tmux/tmux Wiki](https://github.com/tmux/tmux/wiki)。
- 在宿主机上打开两个终端窗口，均执行 `make` 进入容器。

在其中一个终端运行 `make qemu-debug`，会看到 QEMU 命令执行后就停住了。在另一个终端运行 `make gdb`，GDB 自动连接到 QEMU 上，但因为什么命令都没执行，GDB 显示的内容全空：

```text
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│                    [ No Source Available ]                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│                    [ No Assembly Available ]                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
remote Thread 1.1 (src) In:                     L??   PC: 0x1000
(gdb)
```

接下来，请你任选资料阅读：

- [GDB Command Reference - Index page](https://visualgdb.com/gdbreference/commands/)
- [Debugging with GDB](https://sourceware.org/gdb/current/onlinedocs/gdb)
- [100个gdb小技巧](https://wizardforcel.gitbooks.io/100-gdb-tips/content/)
- [gdb 相关使用备忘 - 鹤翔万里的笔记本](https://note.tonycrane.cc/cs/tools/gdb/)

了解下列命令的含义，并进行实操：

```text
layout [Name]
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

!!! tip "关于 gdb 脚本"

    当你运行 `make gdb` 启动调试时，它会首先执行 `gdbinit` 脚本，进行一些初始化设置，这避免了我们每次手动输入一些重复的命令。如果你想尝试更加“现代”的 gdb 调试，现在的 gdb 为我们提供了 [Python API](https://sourceware.org/gdb/current/onlinedocs/gdb.html/Python-API.html)，我们可以更加方便地实现事件回调、调试状态访问、自定义打印等功能，也可以借助 Python 生态实现更丰富的自动化流程。未来的内核调试可能越来越复杂，你可以积极探索更多更好的调试方式。

    在代码仓库中，你可以选择使用 `gdbinit.py` 替代 `gdbinit` 脚本：
    ```diff title="(diff) Makefile" linenums="33"
      gdb:
    !     gdb-multiarch -x gdbinit.py $(KERNEL_PATH)/vmlinux  <- gdbinit 改为 gdbinit.py
    ```


!!! question "考点"

    阅读 `Makefile` 和 `gdbinit`：

    - `make qemu` 和 `make qemu-debug` 有什么不同？
    - 你运行 `make gdb` 时，脚本对 GDB 做了哪些设置？

    使用 GDB 进行下列操作：

    - **断点：**设置断点、查看断点、删除断点
    - **调试：**单指令执行、逐过程执行、结束当前函数、继续执行
    - **查看：**汇编代码、函数调用栈、变量、寄存器值、内存中的内容
    - **分屏：**如何在汇编代码和交互命令行窗格之间切换

    上一节我们了解了启动的详细过程，要求你使用 GDB 进一步了解：

    - 在 OpenSBI 的起始处打断点，看看这时候 `a2` 寄存器的值是多少；进一步，查看对应内存位置的值
    - 在内核起始处打断点，看看这时候 Next Arg1 处的内存存放了什么内容

!!! info "更多资料"

    - GDB 插件：[gdb-dashboard](https://github.com/cyrus-and/gdb-dashboard)
