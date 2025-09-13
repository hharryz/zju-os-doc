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

接下来为同学们导读 RISC-V 架构规范、Linux 内核源码，介绍课程实验的分数评定方法、报告要求。

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

但笔记就像 Cache，虽然快但总会 Miss，实验过程中一定要常看 RISC-V 标准。

## Linux 内核源码和课程仓库结构

详见 [Linux source code layout — The Linux Kernel documentation](https://linux-kernel-labs.github.io/refs/heads/master/lectures/intro.html#linux-source-code-layout)。你可以打开本文开头提到的 Linux 源码链接，边读边看。

课程仓库目录的组织结构与 Linux 源码保持一致。

- `arch`：特定架构的代码，实现启动、异常处理等核心功能，都受到指令集架构的约束
- `arch/riscv/kernel/head.S`：内核最开始执行的代码
- `lib`：内核无法使用标准库，`printk` 等辅助函数实现在这里

其中 [`arch/riscv`](https://github.com/torvalds/linux/tree/master/arch/riscv) 目录最为重要，课程实验大部分的代码工作都将发生在这里。

## 使用 Git 管理源代码

为了方便助教查看同学们的代码修改，本教学班统一使用 [ZJU Git](https://git.zju.edu.cn/) 进行代码管理。请同学们注册 ZJU Git 账号，使用指定的仓库进行实验、提交代码。如果你没有使用过 ZJU Git，请参考 [ZJU Git 101](https://www.pages.zjusct.io/git101/)。

我们在 ZJU Git 上为每个同学创建了私有的代码仓库。实验过程将涉及 Git 的进阶用法，下图展示了完整的工作流和相应命令：

![git.drawio](intro.assets/git.drawio)

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

请注意仓库的分支策略：

- **助教仓库：**
    - `main` 分支：持续开发，你可以向这个分支提交 bug 修复
    - `lab0`、`lab1`、... 分支：当 `main` 分支的开发进度足够完成下一个 Lab 时，我们会创建并冻结相应的分支，代表该 Lab 发布，不会再有变更
- **学生仓库：**
    - `lab0`、`lab1`、... 分支：这些分支设置了保护策略，DDL 截止后无法再向对应分支提交代码，成绩评定以对应分支的代码为准
    - `lab0-makeup`、`lab1-makeup`、... 分支：补交分支，DDL 截止后你可以继续向这些分支提交代码，助教会在期末进行统一评定

!!! info "更多资料"

    - Pro Git 是一本优秀的 Git 教程：英文版 [Git](https://git-scm.com/book/en/)、中文版 [Pro Git 中文版（第二版）](https://www.progit.cn/)

## 工具使用

如果你需要语法分析等辅助工具，建议配置 [clangd](https://clangd.llvm.org/)，它比 VSCode 自带的 IntelliSense 好用很多。该工具已经内置在课程使用的容器环境中。

鼓励同学们使用 AI **辅助**编程，良好的使用方式有：

- AI 辅助，我主导编写：我自己写大部分代码，只在遇到 bug 或不懂的地方才请 AI 帮忙。
- 用 AI 学习编程概念：我主要用 AI 来学习语法、概念、算法原理，而不是直接写项目。
- 和 AI 协同调试：我让 AI 帮我分析报错信息，提供解决思路，自己来修改代码。
- 代码优化和重构：我会让 AI 帮我检查已有代码的性能、可读性，并提出优化建议。
- 生成文档或注释：我让 AI 帮我为代码写注释、生成文档、补充使用说明。

## 实验要求和分数评定

| 实验 | 内容 | 分数占比 | 代码、报告提交 DDL |
| ---- | ---- | ---- | ---- |
| Lab0 | Linux 内核调试 | 5% | （仅验收）暂定第二周(2025-09-23) |
| Lab1 | 引导与时钟中断 | 15% | 第五周 |
| Lab2 | 内核线程调度 | 15% | 第八周 |
| Lab3 | Sv32 虚拟内存 | 15% | 第十一周 |
| Lab4 | 用户模式与系统调用 | 20% | 第十四周 |
| Lab5 | 缺页异常与 Fork | 30% | 第十六周 |
| BonusA | VFS 与 FAT32 文件系统 | 10% | 第十六周 |
| BonusB | 多核系统与调度算法 | 10% | 第十六周 |

- BonusA 和 Bonus B 任选其一
- Lab3-5 允许两人自由组队完成，组队后两人分数相同，请同时到场验收
    - 每个 Lab 允许和不同的人组队/独立完成，以验收时登记 Lab 的合作情况为准

| 部分 | 占比 | 基本分 | 满分 |
| ---- | ---- | ---- | ---- |
| 代码 | 40% | 通过正确性测试和查重 | 编译无 Error 和 Warning |
| 报告 | 50% | **格式：**提交 PDF 文件<br>**内容：**是人写的且人能读懂 | **格式：**推荐使用 LaTeX、Typst、MarkDown 等标记语言编写<br>**内容：**展示出分析实验任务、思考实现方法的过程 |
| 验收 | 10% | 能讲清楚代码在干什么 | （经过提示）能正确回答助教的问题 |

- **推荐作业：**如果你认为自己的实现有独到之处，可以在报告中说明，或许会被选为推荐作业
- **实验报告建议：**
    - 不用写实验目标、实验环境、实验结果等内容，前者实验文档已经描述清楚，后者环境已经统一，结果也已在测试中体现
    - 欢迎在报告中吐槽实验文档、代码，帮助我们改进实验
    - 尽可能精简，比如不要整片贴代码，展示关键语句即可

## 实验与考试的关联

考试重在原理而不会考察技术细节。下面这些 Lab 的整体流程希望你通过实验牢牢掌握，其他内容用过就忘也没关系：

- Lab3 Sv32 虚拟内存系统
- Lab4 用户态与系统调用
- Lab5 缺页异常与 Fork

!!! example

    24 秋冬《操作系统》考试的最后一道大题是：手工完成 Sv32 虚拟内存到物理内存的翻译流程。如果你认真实现了 Lab3，那么这道题非常简单。
