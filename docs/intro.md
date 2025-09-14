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

!!! tip

    如果在课程实验过程中遇到了困难，请积极寻求助教、AI、同学的帮助。

    如果你发现课程文档、代码有值得进一步完善的地方，请向我们反馈，也**欢迎在 GitHub 上向对应仓库提交 PR**。

接下来为同学们导读 RISC-V 架构规范、Linux 内核源码，介绍本课程使用 Git 管理源代码的规范。

## RISC-V 架构规范

RISC-V 非特权级手册中的内容想必同学们已经在硬件课程中吃透了：

- Chapter 2. RV32I Base Integer Instruction Set
- Chapter 3. RV32E and RV64E Base Integer Instruction Sets
- Chapter 4. RV64I Base Integer Instruction Set
- Chapter 6. "Zicsr", Extension for Control and Status Register
 (CSR) Instructions

RISC-V 因其开源、模块化的特点，拥有丰富的扩展指令集。**其中 Zicsr 扩展主要用于操作特权级寄存器，但在非特权级也有用法**，因此没有放置在特权级手册中。其余章节描述的扩展指令对我们来说并不重要。

翻开 RISC-V 特权级手册，第一页有一幅图展示了 RISC-V 体系结构的抽象层次。对应到课程中如下图所示：

![riscv-stack.drawio](intro.assets/riscv-stack.drawio)

其中白色方框是具体实现，黑色方框是抽象接口、标准规范。只要符合接口规范，各个组件就能协同工作，组成完整的系统。在《计算机组成》《计算机体系结构》课程中，我们在 RISC-V 指令集规范的约束下，设计并实现了 CPU。在《操作系统》课程中，我们的任务是在 Linux 内核的接口约束下，向下对接 SBI 规范和 RISC-V 指令集，实现操作系统。

实验涉及 RISC-V 特权级手册中的下列内容：

- Chapter 2. Control and Status Registers (CSRs)
- Chapter 3. Machine-Level ISA
    - M-Mode 的 Trap 委托**（Lab1）**
    - 特权指令**（Lab1、4）**
- Chapter 12. Supervisor-Level ISA
    - S-Mode 的中断、异常处理**（Lab1）**，上下文切换**（Lab2）**
    - Sv32 虚拟内存系统**（Lab3）**
- 其余扩展对我们来说并不重要。

你可以通过下列笔记快速入门：

- [RISC-V ISA - 鹤翔万里的笔记本](https://note.tonycrane.cc/cs/pl/riscv/)

但笔记就像 Cache，虽然快但总会 Miss，**实验过程中一定要常看 RISC-V 标准**。

## Linux 内核源码和课程仓库结构

详见 [Linux source code layout — The Linux Kernel documentation](https://linux-kernel-labs.github.io/refs/heads/master/lectures/intro.html#linux-source-code-layout)。你可以打开本文开头提到的 Linux 源码链接，边读边看。

课程仓库目录的组织结构与 Linux 源码保持一致。

- `arch`：特定架构的代码，实现启动、异常处理等核心功能，都受到指令集架构的约束
- `arch/riscv/kernel/head.S`：内核最开始执行的代码
- `lib`：内核无法使用标准库，`printk` 等辅助函数实现在这里

其中 [`arch/riscv`](https://github.com/torvalds/linux/tree/master/arch/riscv) 目录最为重要，课程实验大部分的代码工作都将发生在这里。

## 使用 Git 管理源代码

为了方便助教查看同学们的代码修改、实现自动化评测，本教学班统一使用 [ZJU Git](https://git.zju.edu.cn/) 进行代码管理。同学们需要使用指定的仓库提交代码、通过评测。

- 对于已经在 ZJU Git 上注册过的同学，助教已经创建好仓库并将你加入到对应的项目中。
- 对于未使用过 ZJU Git 的同学，请前往[登录界面](https://git.zju.edu.cn/users/sign_in)创建账号，并**等待助教批量创建仓库**。[ZJU Git 101](https://www.pages.zjusct.io/git101/) 提供了一些使用帮助。

实验过程将涉及 Git 的进阶用法，下图展示了完整的工作流和相应命令：

![git.drawio](intro.assets/git.drawio)

```bash
git clone git@git.zju.edu.cn:os/2025/jijiangming/<你的学号>.git
git remote add upstream https://git.zju.edu.cn/os/code.git
git commit
git push
git checkout -b <new-lab>
git fetch upstream
git merge upstream/<new-lab>
```

其中 `git merge` 步骤可能遇到冲突。你需要理解冲突部分的代码的作用，并决定保留、合并或是手动整理代码。

我们推荐使用 VSCode 内置的 Git 面板进行相关操作。此外，你还可以安装 [Git Graph](https://marketplace.visualstudio.com/items?itemName=mhutchie.git-graph) 插件来可视化地查看分支、提交等信息。

请注意仓库的分支策略：

- **发布仓库：** `git@github.com:ZJU-OS/code.git`
    - `lab0`、`lab1`、... 分支：各个 Lab 的代码，位于同一个[线性历史（linear history）](https://stackoverflow.com/questions/20348629/what-are-the-advantages-of-keeping-linear-history-in-git)上
- **学生仓库：** `git.zju.edu.cn:os/2025/jijiangming/<你的学号>.git`
    - `lab0`、`lab1`、... 分支：学生提交的代码这些分支设置了保护策略，DDL 截止后无法再向对应分支提交代码，成绩评定以对应分支的代码为准
    - `lab0-makeup`、`lab1-makeup`、... 分支：补交分支，DDL 截止后你可以继续向这些分支提交代码，助教会在期末进行统一评定

!!! info "更多资料"

    - Pro Git 是一本优秀的 Git 教程：英文版 [Git](https://git-scm.com/book/en/)、中文版 [Pro Git 中文版（第二版）](https://www.progit.cn/)

## 工具使用

如果你需要语法分析等辅助工具，建议配置 [clangd](https://clangd.llvm.org/)，它比 VSCode 自带的 IntelliSense 好用很多。IntelliSense 解析时容易受到系统头文件的影响，而 clangd 直接使用编译命令生成的编译数据库 `compile_commands.json`，能准确地解析代码关联。

该工具已经内置在课程使用的容器环境中。我们推荐的开发流程如下：

- 在容器外完成 Git 操作。因为进入容器后以 root 身份运行，使用 Git 命令可能产生权限问题
- 运行 `make` 启动容器
- 在 VSCode 中 ++ctrl+shift+P++，输入 `att run` 会看到 `Dev Containers: Attach to Running Container...` 选项，选择它进入容器选择界面，选择 `zju-os-code` 容器，将会新建一个窗口连接到容器中
- 通过该方式连接容器后，VSCode 会自动识别 `.devcontainer` 目录下的配置，安装推荐的插件
- 运行 `make` 完成编译，会自动生成 `compile_commands.json` 文件，clangd 会自动识别该文件并提供语法分析、补全等功能
- Enjoy it

## 实验与考试的关联

考试重在原理而不会考察技术细节。下面这些 Lab 的整体流程希望你通过实验牢牢掌握，其他内容用过就忘也没关系：

- Lab3 Sv32 虚拟内存系统
- Lab4 用户态与系统调用
- Lab5 缺页异常与 Fork

!!! example

    24 秋冬《操作系统》考试的最后一道大题是：手工完成 Sv32 虚拟内存到物理内存的翻译流程。如果你认真实现了 Lab3，那么这道题非常简单。
