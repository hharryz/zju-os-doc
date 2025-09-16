# OS 实验导读

欢迎来到《操作系统》课程实验。按照培养方案，你已经完成了《数字逻辑设计》《计算机组成》《计算机体系结构》等硬件课程的学习，了解了 RISC-V 指令集架构和计算机系统的基本组成。现在，我们将进入软件部分，动手实现一个运行在 RISC-V 架构下的简易操作系统内核。

OS 实验主要考察工程实践能力，核心要点是**按照 RISC-V 指令集规范，以 Linux 为参考**完成自己的操作系统内核。这需要你**具备阅读大型项目的英文文档和源代码，理解相关内容并动手实现**的能力。左侧导航栏中列出的参考手册和源码仓库将伴随你的整个实验过程。

后续的实验文档会指引同学们去阅读相应章节，但不会复述标准中的内容，避免产生歧义、过时等情况造成误导。

!!! tips "我们会在每个文档指引下放置一个折叠的「要点」，强烈建议同学们在「自行阅读文档后」再对照总结进行自测。如果你实在不愿意或看不懂文档，可以直接看总结，遇到问题时再回去查文档。"

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

详见 [Linux source code layout — The Linux Kernel documentation](https://linux-kernel-labs.github.io/refs/heads/master/lectures/intro.html#linux-source-code-layout)。你可以打开左侧的 Linux 源码链接，边读边看。

课程仓库目录的组织结构与 Linux 源码保持一致。

- `arch`：特定架构的代码，实现启动、异常处理等核心功能，都受到指令集架构的约束
- `arch/riscv/kernel/head.S`：内核最开始执行的代码
- `lib`：内核无法使用标准库，`printk` 等辅助函数实现在这里

其中 [`arch/riscv`](https://github.com/torvalds/linux/tree/master/arch/riscv) 目录最为重要，课程实验大部分的代码工作都将发生在这里。

## 使用 Git 管理源代码

为了方便助教查看同学们的代码修改、实现自动化评测，本教学班统一使用 [ZJU Git](https://git.zju.edu.cn/) 进行代码管理。同学们需要使用指定的仓库提交代码、通过评测。对于未使用过 ZJU Git 的同学，请前往[登录界面](https://git.zju.edu.cn/users/sign_in)创建账号。同时 [ZJU Git 101](https://www.pages.zjusct.io/git101/) 提供了一些使用帮助。

Git 有两种链接：`git@...`（SSH）和 `https://...`（HTTPS）。我们推荐使用 SSH 方式，避免每次推送代码时都需要输入用户名和密码。这需要同学们在 ZJU Git 上添加 SSH 公钥，具体步骤见 [设置 SSH 访问 - ZJU Git 101](https://www.pages.zjusct.io/git101/prep/setup_ssh.html)。

!!! info "FAQ"

    有同学会问：Git 的 `user.name` 和 `user.email` 要设置成什么？其实设成什么都没关系，因为仓库的地址确定，我们会去对应的仓库看你的提交。

    Git 平台（如 GitHub、GitLab、ZJU Git）会根据你的提交中的 `user.email` 来关联到你的账号，从而显示头像等信息。如果你想让提交显示正确的头像，可以将 `user.email` 设置成你在 ZJU Git 上绑定的邮箱。

课程初期名单没有确定，Lab0 也没有代码工作，所以尚未为各位同学创建仓库。请同学们暂时使用发布仓库（`git@git.zju.edu.cn:os/code.git`）完成 Lab0：

```shell
git clone git@git.zju.edu.cn:os/code.git
cd code
```

并在助教通知仓库创建完成后使用下面的命令切换到自己的私有仓库：

```console
$ git remote -v
origin  git@git.zju.edu.cn:os/code.git (fetch)
origin  git@git.zju.edu.cn:os/code.git (push)
$ git remote rename origin upstream
$ git remote add origin git@git.zju.edu.cn:os/2025/jijiangming/<你的学号>.git
$ git fetch origin
```

然后，检查是否有分支仍然使用 `upstream` 作为上游。如果有，则将其上游改为 `origin`：

```console
$ git branch -vv
* lab0      abc123 [upstream/lab0]  Work on lab0
  main      def456 [origin/main] Update README
$ git branch --set-upstream-to=origin/lab0 lab0
```

实验过程将涉及 Git 的多分支开发等进阶用法，下图展示了完整的工作流和相应命令：

![git.drawio](intro.assets/git.drawio)

```shell
# 切换到对应的 lab 分支
git checkout <labN>
# ...进行你的开发工作
git commit
# 推送到你的私有仓库
git push
# 创建下一个实验的分支
git checkout -b <labN+1>
# 从上游合并下一个实验的代码
git fetch upstream
git merge upstream/<labN+1>
# 解决冲突
git commit
# ... 进行你的开发工作
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

VSCode 自带的 IntelliSense 解析时容易受到系统头文件的影响，产生一堆报错，在内核编程等特殊场景下表现不佳。我们推荐使用如下工具：

- [clangd](https://clangd.llvm.org/)：一个基于 LLVM 的 C/C++ 语言服务器，提供智能补全、语法检查等功能。它直接使用编译命令生成的编译数据库 `compile_commands.json`，能准确地解析代码关联。
- [bear](https://github.com/rizsotto/Bear)：一个生成 `compile_commands.json` 的工具，能将 Makefile 生成的编译命令转换为 JSON 格式，供 clangd 使用。

相关工具已经内置在课程使用的容器环境中，如果你后续使用 DevContainer 开发（见 Lab0）将会自动启用。感兴趣的同学可以自行阅读这些工具的文档。

## 实验与考试的关联

考试重在原理而不会考察技术细节。下面这些 Lab 的整体流程希望你通过实验牢牢掌握，其他内容用过就忘也没关系：

- Lab3 Sv32 虚拟内存系统
- Lab4 用户态与系统调用
- Lab5 缺页异常与 Fork

!!! example

    24 秋冬《操作系统》考试的最后一道大题是：手工完成 Sv32 虚拟内存到物理内存的翻译流程。如果你认真实现了 Lab3，那么这道题非常简单。
