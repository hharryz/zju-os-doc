# OS 实验导读

欢迎来到《操作系统》课程实验。按照培养方案，你已经完成了《数字逻辑设计》《计算机组成》《计算机体系结构》等硬件课程的学习，了解了 RISC-V 指令集架构和计算机系统的基本组成。现在，我们将进入软件部分，动手实现一个运行在 RISC-V 架构下的简易操作系统内核。

OS 实验主要考察工程实践能力，核心要点是**按照 RISC-V 指令集规范，以 Linux 为参考**完成自己的操作系统内核。这需要你**具备阅读大型项目的英文文档和源代码，理解相关内容并动手实现**的能力，下列资料将陪伴你的整个实验过程：

- [RISC-V Ratified Specifications](https://riscv.org/specifications/ratified/)
    - [ISA Specifications](https://github.com/riscv/riscv-isa-manual/)
        - 非特权级手册：The RISC-V Instruction Set Manual Volume I: Unprivileged ISA
        - 特权级手册：The RISC-V Instruction Set Manual Volume II: Privileged Architecture
    - Non-ISA Hardware Specifications
        - SBI 规范：[RISC-V Supervisor Binary Interface Specification](https://github.com/riscv-non-isa/riscv-sbi-doc/)
        - 汇编手册：[RISC-V Assembly Programmer's Manual](https://github.com/riscv-non-isa/riscv-asm-manual)
- Linux 内核源码：
    - [torvalds/linux: Linux kernel source tree](https://github.com/torvalds/linux)
    - [Linux source code - Bootlin Elixir Cross Referencer](https://elixir.bootlin.com/)

后续的实验文档会指引同学们去阅读相应章节，但不会复述标准中的内容，避免产生歧义、过时等情况造成误导。

根据调查问卷，后续实验默认同学们掌握下列技能：

- 拥有 Linux/macOS 环境，具备 Linux 基础操作能力
- 了解 Git、Makefile 的基本用法

!!! tips

    如果在课程实验过程中遇到了困难，请积极寻求助教、AI、同学的帮助。

    如果你发现课程文档、代码有值得进一步完善的地方，请向我们反馈，也**欢迎在 GitHub 上向对应仓库提交 PR**。

接下来为同学们导读 RISC-V 架构规范、Linux 内核源码。

## RISC-V 软件栈

翻开 RISC-V 特权级手册，第一页有有一幅图展示了 RISC-V 体系结构的抽象层次。对应到课程中如下图所示：

![riscv-stack.drawio](intro.assets/riscv-stack.drawio)

其中白色方框是具体实现，黑色方框是抽象接口、标准规范。只要符合接口规范，各个组件就能协同工作，组成完整的系统。在《计算机组成》《计算机体系结构》课程中，我们在 RISC-V 指令集规范的约束下，设计并实现了 CPU。在《操作系统》课程中，我们的任务是在 Linux 内核的接口约束下，向下对接 SBI 规范和 RISC-V 指令集，实现操作系统。

## 系统启动过程

当你按下电脑的电源键，计算机开始按照下面的流程启动：

1. **固件加载（Firmware）**

    - 在主板 ROM 中烧录的固件（如 BIOS 或 UEFI）会首先被加载到内存中执行。
    - 固件会完成 **硬件自检**（Power-On Self Test，POST），检测 CPU、内存、外设等是否可用，并进行基础初始化。
    - 接着，它会寻找合适的启动设备（如硬盘、U 盘、网络等），然后将 **引导程序**（Bootloader）加载到内存并执行。

2. **引导程序（Bootloader）阶段**

    该阶段不是必要的，固件可以直接引导进入内核。如果有多个操作系统，则引导程序可以提供选择界面。

    - 引导程序的任务是把系统带入可以运行内核的状态。
    - 典型的引导程序如 **GRUB** 或 **U-Boot** 会加载内核镜像（Kernel Image）和 **根文件系统**（Root File System）。
    - 加载完成后，控制权会被转交给内核，开始执行内核入口点的代码。

3. **内核启动（Kernel Boot）**

    - 内核会初始化内存管理、设备驱动、进程调度等核心功能。
    - 初始化完成后，内核会启动 **第一个用户空间进程**（通常是 `/sbin/init` 或 `systemd`），进一步完成系统服务和登录界面的启动。
    - 最终，用户可以登录系统，进入正常的操作环境。

启动流程根据不同的硬件平台和固件实现会有所差异。

## Linux 内核源码和课程仓库结构

详见 [Linux source code layout — The Linux Kernel documentation](https://linux-kernel-labs.github.io/refs/heads/master/lectures/intro.html#linux-source-code-layout)。现在，你可以打开本文开头提到的 Linux 源码链接，边读边看。

课程仓库目录的组织结构与 Linux 源码保持一致。

- `arch`：特定架构的代码，实现启动、异常处理等核心功能，都受到指令集架构的约束
- `arch/riscv/kernel/head.S`：内核最开始执行的代码
- `lib`：内核无法使用标准库，`printk` 等辅助函数实现在这里

其中 [`arch/riscv`](https://github.com/torvalds/linux/tree/master/arch/riscv) 目录最为重要，课程实验大部分的代码工作都将发生在这里。

## 工具使用

如果你需要语法分析等辅助工具，建议配置 [clangd](https://clangd.llvm.org/)，它比 VSCode 自带的 IntelliSense 好用很多。该工具已经内置在课程使用的容器环境中。
