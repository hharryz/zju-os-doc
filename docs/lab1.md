# Lab 1：内核启动与时钟中断

!!! danger "DDL"

    本实验文档正在编写中，尚未正式发布。

## 实验简介

在 Lab 1 中我们将开始构建自己的内核。梳理一下目前的思路：

- 在实验导读和 Lab0 中，我们已经了解了启动过程：

    - OpenSBI 跳转到内核第一条指令处。
    - `arch/riscv/boot.S` 是内核最开始执行的代码。查看它，你会发现它最终执行了 `call start_kernel`。这个函数位于 `arch/riscv/init/main.c`，是一个 **C 语言写的函数**。

    那么有如下问题：

    - 从 OpenSBI 跳到内核时，系统的状态（寄存器、内存等）是什么样的？这需要你动手调试，用 Lab0 学习的 QEMU Monitor 和 GDB 自行查看。
    - 从汇编代码跳到 C 函数，调用者、被调用者各需要做什么工作？回忆《计算机组成》课程，我们学习过汇编与 C 语言的关系。这需要你学习**RISC-V 调用约定**，然后补全 `arch/riscv/boot.S`。
    - 如何控制内核最开始执行的代码放置的位置？这需要你学习**链接器脚本**，然后理解 `arch/riscv/kernel/vmlinux.lds`。
    - 没有了标准库，打印字符等基本功能如何实现？这需要你学习**SBI 规范**，然后补全 `arch/riscv/kernel/sbi.c`。

完成这些，你就能够启动自己的内核，并打印 `Hello, ZJU-OS!` 了。

- 我们的下一个目标是为 Lab2 **线程调度**做准备：

    操作系统是**事件驱动**的，意味着系统的执行流是由各种事件触发的，例如 I/O 完成、进程创建、用户输入等。时钟中断是其中最基础、最重要的事件源之一。**如果没有时钟中断，操作系统将无法感知时间的流逝，无法在合适的时机进行进程切换、调度、超时处理或实现延时等待。**通过周期性触发的时钟中断，**内核可以被动地转变为主动：不再依赖进程自愿交出 CPU，而是能在中断到来时打断当前执行流**，检查是否有更高优先级的任务或等待超时的事件需要处理，从而保证多任务系统的公平性和响应性。

    那么有如下问题：

    - RISC-V 是如何设计中断与异常的？怎么开启时钟中断？这需要你学习**RISC-V 特权级、中断与异常、CSR 寄存器**。
    - 时钟中断可能在任何时候发生。应该如何实现中断处理程序，保存和恢复现场，从而保证内核和用户程序的正确执行？这需要你学习**RISC-V 中断异常委派、中断处理**，然后补全 `arch/riscv/kernel/entry.S`。

完成这些，你就能做 Lab2 了。

## Part 1：启动工作

### RISC-V 汇编与调用约定

同学们上次熟悉 RISC-V 汇编与 C 语言的关系，应该是在《计算机组成》课程中。上《计算机体系结构》的时候，同学们只烧二进制 Bit 文件，应该已经把相关内容忘得差不多了。本节我们来回顾一下。

<figure markdown="span">
    ![co.webp](lab1.assets/co.webp)
    <figcaption>
    《计算机组成》课程 PPT
    </figcaption>
</figure>

[Compiler Explorer](https://godbolt.org/) 可以方便地探索不同编译器、不同版本、不同优化选项下的汇编代码。我们用它来看看，C 与 RISC-V 汇编是如何对应的：

!!! tip "下面是一个可以交互的子页面，请点击右下角 Edit on Compiler Explorer 放大查看"

<iframe width="800px" height="200px" src="https://godbolt.org/e#g:!((g:!((g:!((h:codeEditor,i:(filename:'1',fontScale:14,fontUsePx:'0',j:1,lang:c%2B%2B,selection:(endColumn:2,endLineNumber:6,positionColumn:2,positionLineNumber:6,selectionStartColumn:2,selectionStartLineNumber:6,startColumn:2,startLineNumber:6),source:'int+func(int+a)%7Breturn+a%3B%7D%0A%0Aint+main()+%7B%0A++int+r+%3D+func(10)%3B%0A++return+r+%2B+1%3B%0A%7D'),l:'5',n:'0',o:'C%2B%2B+source+%231',t:'0')),k:50,l:'4',n:'0',o:'',s:0,t:'0'),(g:!((h:compiler,i:(compiler:rv64-gcctrunk,filters:(b:'0',binary:'1',binaryObject:'1',commentOnly:'0',debugCalls:'1',demangle:'0',directives:'0',execute:'1',intel:'0',libraryCode:'0',trim:'1',verboseDemangling:'0'),flagsViewOpen:'1',fontScale:14,fontUsePx:'0',j:1,lang:c%2B%2B,libs:!(),options:'',overrides:!(),selection:(endColumn:27,endLineNumber:7,positionColumn:27,positionLineNumber:7,selectionStartColumn:27,selectionStartLineNumber:7,startColumn:27,startLineNumber:7),source:1),l:'5',n:'0',o:'+RISC-V+(64-bits)+gcc+(trunk)+(Editor+%231)',t:'0')),k:50,l:'4',n:'0',o:'',s:0,t:'0')),l:'2',n:'0',o:'',t:'0')),version:4"></iframe>

请回忆你在《计算机组成》课程中学习的相关内容，回答下面的问题。如果有想不起来的地方，请阅读 [RISC-V ELF psABI Document](spec/riscv-abi.pdf) 的以下章节：

- 1.1. Integer Register Convention
- 2.1. Integer Calling Convention

!!! question "考点"

    - 每个函数的开头都操作了 `sp`，这是在干什么？
    - 尝试修改 C 语言代码，你会发现 `sp` 的差值总是 16 的倍数，这是为什么？
    - 调用函数前后做了什么？

??? note "要点"

    ??? note "参数寄存器与返回值"

        - RISC-V 提供 **8 个参数寄存器 a0–a7**：
            - **a0、a1**：用于返回值
            - **a2–a7**：用于传递参数，超出部分放在栈上
        - 超过寄存器数量的参数会通过**栈**传递

    ??? note "参数传递规则"

        **标量（Scalars）**

        - **≤ XLEN 位**：放在一个寄存器或栈上传递
            - 小于 XLEN 的整数会符号扩展（sign-extend）到 XLEN 位
            - 小于 XLEN 的浮点数会零扩展到 XLEN 位，高位不定义
        - **2 × XLEN 位**：用连续两个寄存器传递，高位在编号大的寄存器中
        - **> 2 × XLEN 位**：通过引用（地址）传递

        **结构体和联合体（Aggregates）**

        - **≤ XLEN 位**：单寄存器传递
        - **≤ 2 × XLEN 位**：双寄存器传递，若寄存器不足，剩余部分放栈上
        - **> 2 × XLEN 位**：按引用传递（传地址）

    ??? note "栈对齐和栈传参"

        - 栈指针（SP）必须在**函数入口时 128-bit 对齐**
        - 参数在栈上的布局按 **≥ type alignment 和 XLEN 对齐**，但不会超过栈本身的对齐要求
        - 传栈的第一个参数在 SP + 0 位置，之后依次递增

    ??? note "变参函数（Variadic Functions）"

        - 变参与命名参数传递方式相同
        - 特殊规则：
            - **2 × XLEN 位对齐**的参数必须使用**偶数号寄存器对**
            - 一旦有参数被传到栈上，之后的参数都必须放栈上

    ??? note "返回值"

        - 返回值的传递方式 = 该类型第一个命名参数的传递方式
        - 如果是按引用传递的，调用者分配空间，传地址，callee 写结果

    ??? note "结构体对齐与位域规则"

        - 小于一个 XLEN 的结构体直接放在寄存器里，按内存布局排列字段
        - 位域按小端序（little-endian）打包：
            - 如果跨越对齐边界，跳到下一个对齐边界
            - 多余位按 padding 处理

    ??? note "寄存器保存规则"

        - **s0–s11** 必须在函数调用间保持（callee-save）
        - **临时寄存器 t0–t6、a0–a7** 调用不保证保存（caller-save）

此外，除了基本的 I 指令集，我们在汇编时还经常使用一些**伪指令（pseudo-instruction）**，它们并不是 RISC-V 指令集的一部分，而是汇编器提供的方便写代码的语法糖。伪指令会被汇编器翻译成一个或多个真实的指令。

请阅读 [RISC-V 汇编手册](spec/riscv-asm.pdf) 的以下章节，学习常用的伪指令：

- Chapter 29. A listing of standard RISC-V pseudoinstructions

!!! question "考点"

    - `li`、`la`、`mv`、`j`、`ret` 等伪指令分别对应什么真实指令？
    - `call` 伪指令做了什么工作？它与 `jal` 指令有什么区别？

### 汇编器指令

你已经熟悉 RISC-V 汇编指令。但是为了写汇编代码，还要了解一些汇编器指令（assembler directive），这就像 C 语言中的预处理指令一样。

阅读 `arch/riscv/head.S`，尝试理解它的内容。如果有不懂的地方，可以查阅 `as` 汇编器手册：

- [5 Symbols](https://sourceware.org/binutils/docs/as.html#Symbols-2)：阅读 5.1-5.4 即可
- [7.3 .align [abs-expr[, abs-expr[, abs-expr]]]](https://sourceware.org/binutils/docs/as.html#Align)
- [7.43 .global symbol, .globl symbol](https://sourceware.org/binutils/docs/as.html#Global)
- [7.87 .section name - ELF Version](https://sourceware.org/binutils/docs/as.html#ELF-Version)
- [7.65 .macro](https://sourceware.org/binutils/docs/as.html#g_t_002emacro)：类似于 C 语言中的 `#define`，当你需要复用代码时很好用，比如稍后写 Trap Handler 时会用到
- [7.94 .space size [,fill]](https://sourceware.org/binutils/docs/as.html#g_t_002espace-size-_005b_002cfill_005d)

??? note "要点"

    ??? note "Symbols 基础概念"

        - **符号（Symbol）的作用**
            - 程序员用符号命名；链接器用符号做链接；调试器用符号调试。
            - 符号在目标文件中的顺序可能与声明顺序不同，这可能影响某些调试器。

        - **符号的核心功能**
            - 命名程序中的地址或数据位置。
            - 可以代表位置计数器、常量值或程序中定义的标签。
            - 支持多种类型：全局符号、本地符号、局部标签等。

    ??? note "5.1 Labels"

        - **标签的定义**
            - `symbol:` 定义标签，表示当前位置计数器的值。
            - 相同符号多次定义时，以第一次为准，后续会收到警告。

        - **本地标签（Local Labels）**
            - 形式：`N:` 定义标签，`Nb` 引用最近的前一个，`Nf` 引用最近的下一个。
            - 编译器会将本地标签转换成唯一的符号名，避免冲突。

        - **美元标签（Dollar Labels）**
            - 形式如 `55$:`，作用范围更小，遇到非本地标签就失效。

    ??? note "5.2 Giving Symbols Other Values"

        - **符号赋值**
            - 形式：`symbol = expression` 等价于 `.set`，`symbol == expression` 等价于 `.eqv`。
            - 可以把符号赋值为任意表达式结果。

    ??? note "5.3 Symbol Names"

        - **命名规则**
            - 以字母或 `._` 开头，支持 `$`，大小写敏感。
            - 不以数字开头，除非是本地标签。
            - 支持多字节字符，但可能受编译器或系统限制。

        - **本地符号**
            - ELF 默认前缀 `.L`，传统 a.out 系统前缀 `L`。
            - 本地符号通常不会出现在目标文件中，除非用 `-L` 选项。

    ??? note "5.4 The Special Dot Symbol"

        - **`.` 符号**
            - 表示当前位置计数器。
            - `melvin: .long .` → 定义 `melvin` 为其自身地址。
            - 赋值 `.=.+4` 等价于 `.org` 指令，跳过 4 字节。

    ??? note "常见汇编器指令"

        - **`.align n`**：将位置计数器对齐到 n 字节边界。
        - **`.global symbol` / `.globl symbol`**：导出符号给链接器 `ld` 使用。
        - **`.section name`**：切换到指定节，支持 ELF 特定参数。
        - **`.macro` / `.endm`**：宏定义，可生成重复代码。
        - **`.space size[,fill]`**：分配 size 字节空间，填充值为 fill（默认为 0）。

### 链接器脚本与内核内存布局

在 C 语言编程课上，我们了解过编译器的基本流程：预处理、编译、汇编、链接，`gcc` 默认帮你完成了所有流程。拆解开来流程示例如下：

```shell
# 编译生成 RISC-V 汇编代码 main.S
riscv64-linux-gnu-gcc -S main.c -o main.S
# 汇编生成 RISC-V 目标文件 main.o
riscv64-linux-gnu-as main.S -o main.o
# 编译生成 RISC-V 汇编代码 func.S
riscv64-linux-gnu-gcc -S func.c -o func.S
# 汇编生成 RISC-V 目标文件 func.o
riscv64-linux-gnu-as func.S -o func.o
# 链接 main.o 和 func.o 生成可执行文件 main
riscv64-linux-gnu-ld main.o func.o -o main
```

链接过程合并多个目标文件，并**重定位**它们的数据，**链接**各类符号（函数名、变量名等）。链接器 `ld` 通过**链接脚本**来控制最终的生成的文件，默认使用内置的脚本生成 **ELF 格式的可执行文件**，你可以使用下面的命令查看这个脚本：

```shell
riscv64-linux-gnu-ld --verbose
```

请阅读 [3 Linker Scripts - LD](https://sourceware.org/binutils/docs/ld.html#Scripts) 的以下章节，学习基本的链接器脚本语法：

- 3.1 [Basic Linker Script Concepts](https://sourceware.org/binutils/docs/ld.html#Basic-Linker-Script-Concepts)
- 3.3 [Simple Linker Script Example](https://sourceware.org/binutils/docs/ld.html#Simple-Linker-Script-Example)
- 3.5 [Assigning Values to Symbols](https://sourceware.org/binutils/docs/ld.html#Assigning-Values-to-Symbols)

??? note "要点"

    下面的要点中，**源码中的符号引用**非常重要，在后续写 C 代码时会用到。

    ??? note "Linker Script 基础概念"

        1. **目标文件（Object File）**

            - GCC 或其他编译器生成的 `.o` 文件。
            - 每个目标文件包含若干 **输入节（Input Section）**，例如 `.text`, `.data`, `.bss`。
            - 每节有名字、大小和数据内容。

        2. **输出文件（Output File）**

            - 链接器将多个输入文件合并生成单一输出文件。
            - 输出文件也有若干 **输出节（Output Section）**。
            - 输出节是由输入节组成的，它们可以被加载到内存中或只分配空间。

        3. **节的属性**

            - **Loadable（可加载）**：节内容需要加载到内存，例如代码和初始化数据。
            - **Allocatable（可分配）**：节不包含数据，但需要在内存中保留空间，例如 `.bss`。
            - **非加载非分配**：节通常用于调试信息，例如 `.debug_*`。

        4. **符号（Symbol）**

            - 每个目标文件中定义或引用的函数、全局变量都是符号。
            - 链接器会解析符号的地址，未定义符号需要在链接时找到定义。

        5. **VMA 和 LMA**

            - **VMA（Virtual Memory Address）**：程序运行时的虚拟地址。
            - **LMA（Load Memory Address）**：程序加载到内存时的地址。
            - 例如，ROM 加载到 RAM 时 LMA ≠ VMA。

        6. **位置计数器 `.`**

            - 链接器使用 `.` 来表示当前内存位置。
            - 每定义一个输出节，`.` 会自动增加节大小。

    ??? note "Linker Script 文件格式"

        - LD Script 是文本文件。
        - 支持 **注释**：

            ```ld
            /* 这是注释 */
            ```

        - 语法结构：

            - 关键字 + 参数，例如 `OUTPUT_ARCH("riscv")`
            - 符号赋值，例如 `BASE_ADDR = 0x80000000`
            - 使用 `SECTIONS` 定义输出节布局。

    ??? note "SECTIONS 指令"

        `SECTIONS` 是最核心的命令，用于描述输出文件的内存布局。示例：

        ```ld
        SECTIONS
        {
            . = 0x10000;        /* 设置位置计数器 */
            .text : { *(.text) } /* 将所有输入文件的 .text 节放入输出节 .text */
            . = 0x8000000;
            .data : { *(.data) }
            .bss : { *(.bss) }
        }
        ```

        解释：

        - `. = 0x10000`：将位置计数器设置为 0x10000。
        - `.text : { *(.text) }`：输出节 `.text` 包含所有输入节 `.text`。
        - `.data` 和 `.bss` 同理。
        - 链接器会保证输出节按需对齐，如果地址不符合要求，会在节之间插入间隙。

    ??? note "符号赋值"

        ??? note "符号赋值的基本概念"

            - 在链接脚本（linker script）中可以给符号（symbol）赋值，定义它并把它加入符号表，作用域为全局。
            - 赋值语句形式：

                ```ld
                symbol = expression;      // 定义符号值
                symbol += expression;     // 在原值基础上加
                symbol -= expression;     // 在原值基础上减
                ...
                symbol |= expression;     // 按位或
                ```

            - 第一次赋值会定义符号，后续的 +=, -= 等操作要求符号已存在。
            - `.` 表示位置计数器（location counter），只能在 `SECTIONS` 中使用。

        ??? note "三种赋值位置"

            符号赋值可以出现在：

            1. 单独作为命令
            2. `SECTIONS` 命令内部
            3. 输出段（output section）描述内部

            示例：

            ```ld
            floating_point = 0;
            SECTIONS
            {
              .text :
                {
                  *(.text)
                  _etext = .;
                }
              _bdata = (. + 3) & ~ 3;
              .data : { *(.data) }
            }
            ```

            - `floating_point = 0` → 定义为 0
            - `_etext = .` → 定义为 `.text` 末尾地址
            - `_bdata = (. + 3) & ~3` → 定义为 4 字节对齐的 `.text` 末尾地址


        ??? note "PROVIDE"

            - `PROVIDE(symbol = expression)`
            - 仅在**符号被引用且未被定义**时才定义，避免与用户代码冲突。
            - 常用于传统符号如 `etext`，用户可覆盖定义。

            示例：

            ```ld
            SECTIONS {
              .text : {
                PROVIDE(etext = .);
              }
            }
            ```

        ??? note "源码中的符号引用"

            - 链接脚本符号 ≠ C 语言变量，**没有实际内存分配**，只是一个地址。
            - 在汇编代码中直接作为地址使用：

                ```asm
                la a0, symbol_name  # 加载符号地址到 a0
                ```

            - 在 C 代码中使用时：

                ```c
                extern char start_of_ROM, end_of_ROM, start_of_FLASH;
                memcpy(&start_of_FLASH, &start_of_ROM, &end_of_ROM - &start_of_ROM);
                ```

            - 也可以直接声明为数组：

                ```c
                extern char start_of_ROM[], end_of_ROM[], start_of_FLASH[];
                memcpy(start_of_FLASH, start_of_ROM, end_of_ROM - start_of_ROM);
                ```

            - **必须取地址，不能直接当作数值使用**。

现在，我们来简单对比一下 Linux 内核链接脚本和 `ld` 内置链接脚本（ELF 文件）的区别：

<div class="grid cards" markdown>

```text title="linux/arch/riscv/kernel/vmlinux.lds"
ENTRY(_start)
SECTIONS
{
 . = ((((-1))) - 0x80000000 + 1);
 _start = .;
 . = ALIGN((1 << 12));
 . = ALIGN((1 << 21));
 . = ALIGN(4);
```

```text title="riscv64-linux-gnu-ld --verbose"
ENTRY(_start)
SECTIONS
{
  . = SEGMENT_START("text-segment", 0x10000) + SIZEOF_HEADERS;
  .dynsym         : { *(.dynsym) }
  .rela.dyn       :
    {
      *(.rela.init)
      *(.rela.text .rela.text.* .rela.gnu.linkonce.t.*)
```

</div>

关键要点：

- **起始位置**：ELF 程序为 `0x10000 + SIZEOF_HEADERS`，而内核是 `0xffffffff80000000`。学习虚拟内存后你会理解这些地址受[操作系统内存布局](https://docs.kernel.org/arch/riscv/vm-layout.html)影响。
- **对齐：**内核对齐要求十分严格，通常使用页面对齐（4KB 或 2MB 等）、缓存行对齐（64B）等，需要考虑内存分页、NUMA 架构 Cache 一致性等问题。
- **节的选择**：内核不使用[动态链接](https://lwn.net/Articles/961117/)、[重定位](https://dram.page/p/relative-relocs-explained/)，这些技术是为了应用程序设计的。因此没有 `.dynsym`、`.rela.dyn` 等节。
- **`_start` 符号**：两者都定义了 `_start` 作为程序入口，但内核的 `_start` 定义在脚本中，而应用程序的 `_start` 由[标准库定义](https://www.monperrus.net/martin/compiling-where-is-_start)（标准库需要做一些初始化工作），并不在应用程序代码中。

    你可以尝试编译一个简单的程序而不带上标准库，`ld` 就会找不到 `_start` 符号：

    ```console
    $ riscv64-linux-gnu-gcc -nostdlib test.c
    /usr/lib/gcc-cross/riscv64-linux-gnu/15/../../../../riscv64-linux-gnu/bin/ld: warning: cannot find entry symbol _start; defaulting to 000000000000030a
    ```

!!! note "Take Home Message"

    总而言之，与普通的应用程序相比，操作系统的链接需要精细控制内存布局、初始化代码段、只读/可写数据段、调试信息和特定硬件相关表格的放置。

    链接器的输入和输出都是二进制文件，受到 **ABI 规范** 的约束。

链接器脚本的具体语法并不重要，能大致看懂就行。接下来，请自行查找资料或使用 AI 辅助，理解实验代码仓中的 `arch/riscv/kernel/vmlinux.lds`，回答下面的问题：

!!! question "考点"

    - 这个链接脚本描述的就是整个内核的内存布局。它从哪里开始，有多大？
    - 其中的各个段（`.text`、`.rodata`、`.data`、`.bss`）分别存放什么数据？
    - 你认为内核的栈应该放在哪里？为什么？（提示：栈的增长方向是？）
    - `_skernel` 这些符号是什么？你要如何在汇编和 C 代码中使用它们？

!!! info "ABI 规范及其历史"

    RISC-V ABI 手册中没有 `.text`、`.rodata` 等的定义，因为它是历史传承下来的。RISC-V ABI 手册的参考文献指向了 [System V Application Binary Interface - DRAFT](https://www.sco.com/developers/gabi/latest/contents.html)，你可以在这里找到相关定义。

    操作系统有比较久远的历史沉淀，因此ABI 层的规范不像 SBI 那样唯一。感兴趣的同学可以阅读 [The UNIX System -- History and Timeline -- UNIX History](https://unix.org/what_is_unix/history_timeline.html)。

### OpenSBI 调试

本节让我们动手探究 OpenSBI 跳转到内核时，系统的状态（寄存器、内存等）是什么样的。

!!! question "考点"

    - OpenSBI 跳转到内核时，`sp` 寄存器的值是多少？

同时，注意到 OpenSBI 有如下输出。这是 OpenSBI 在通过物理内存保护（Physical Memory Protection, PMP）机制设置物理内存的权限：

```text
Domain0 Region00            : 0x0000000000100000-0x0000000000100fff M: (I,R,W) S/U: (R,W)
Domain0 Region01            : 0x0000000010000000-0x0000000010000fff M: (I,R,W) S/U: (R,W)
Domain0 Region02            : 0x0000000002000000-0x000000000200ffff M: (I,R,W) S/U: ()
Domain0 Region03            : 0x0000000080040000-0x000000008005ffff M: (R,W) S/U: ()
Domain0 Region04            : 0x0000000080000000-0x000000008003ffff M: (R,X) S/U: ()
Domain0 Region05            : 0x000000000c400000-0x000000000c5fffff M: (I,R,W) S/U: (R,W)
Domain0 Region06            : 0x000000000c000000-0x000000000c3fffff M: (I,R,W) S/U: (R,W)
Domain0 Region07            : 0x0000000000000000-0xffffffffffffffff M: () S/U: (R,W,X)
```

!!! question "考点"

    - `sp` 落在哪个区域？该区域各个特权级的权限是什么？

!!! info "更多资料"

    - Physical Memory Protection (PMP) 机制：RISC-V 特权级手册 [3.7. Physical Memory Protection](https://zju-os.github.io/doc/spec/riscv-privileged.html#pmp)
    - OpenSBI 相关源码相关源码位于 [`lib/sbi/sbi_domain.c`](https://github.com/riscv-software-src/opensbi/blob/master/lib/sbi/sbi_domain.c) 中的 `sbi_domain_init()`
    - 感兴趣的同学可以使用 QEMU Monitor 打印内存树（`info mtree`），对比了解 OpenSBI 为什么要设置这些 PMP 区域

### Task 1：为 `start_kernel()` 准备运行环境

现在你已经知道：

- OpenSBI 跳转到内核时，`sp` 指向的位置没法读写
- 内核的栈应该放在 `.bss.stack` 段中
- 汇编代码中可以引用链接脚本定义的符号

你的任务是修改 `arch/riscv/head.S`，让 `start_kernel()` 函数能够成功执行。

!!! success "完成条件"

    - GDB 断点打在 `printk()` 处，能够成功到达断点。
    - 可以通过评测框架的 `lab1-task1` 测试。

### C 内联汇编

C++ 标准支持内联汇编，但 C 标准并不支持。GCC 提供了内联汇编的扩展语法，允许在 C 代码中嵌入汇编指令。

请阅读 GCC 编译器手册中的下列章节，学习内联汇编的语法：

- [Extended Asm (Using the GNU Compiler Collection (GCC))](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html)
- [Local Register Variables (Using the GNU Compiler Collection (GCC))](https://gcc.gnu.org/onlinedocs/gcc/Local-Register-Variables.html)

??? note "要点"

    ??? note "Extended Asm"

        ??? note "基本概念"

            - **Extended asm** 允许在 C 代码中嵌入汇编指令，并可读写 C 变量、使用 C 标签跳转。
            - 语法格式：

                ```c
                asm asm-qualifiers (
                    AssemblerTemplate
                    : OutputOperands
                    [ : InputOperands
                    [ : Clobbers
                    [ : GotoLabels ]]]
                )
                ```

            - `asm` 是 GNU 扩展。若需兼容 `-ansi` 和 `-std`，应使用 `__asm__`。

        ??? note "关键限定符"

            - **volatile**
                - 防止编译器优化掉 asm 语句或移动其位置。
                - 必须用于有副作用、或结果不可预测的指令（如 `rdtsc`）。
            - **inline**
                - 告诉编译器 asm 语句可被内联。
            - **goto**
                - 允许从 asm 跳转到指定的 C 标签。

        ??? note "参数类型"

            1. **AssemblerTemplate**
                - 字符串模板，混合汇编指令和参数占位符（如 `%0`, `%1` 或符号名 `%[name]`）。

            2. **OutputOperands**
                - C 变量，被汇编代码修改。格式：`[symbol] "constraint" (variable)`
                - 约束符必须以 `=`（只写）或 `+`（读写）开头。
                - 可用 `&` 防止输出与输入重叠。

            3. **InputOperands**
                - C 表达式，被汇编代码读取。格式：`[symbol] "constraint" (expression)`
                - 约束不能以 `=` 或 `+` 开头。可用数字/符号名指定与输出共享寄存器。

            4. **Clobbers**
                - 汇编会修改的额外寄存器或状态，如 `"cc"`（标志寄存器）、`"memory"`（内存屏障）、`"redzone"`。
                - 不可与输入输出寄存器重叠，不应包含栈指针。

            5. **GotoLabels**
                - 汇编中可能跳转的 C 标签。禁止跨越 asm 语句跳转。

        ??? note "常见用法示例"

            - **基本用法**：

                ```c
                int src = 1, dst;
                asm ("mov %1, %0\n\t"
                    "add $1, %0"
                    : "=r"(dst)      // 输出
                    : "r"(src));     // 输入
                ```

            - **带 clobber**：

                ```c
                asm volatile ("movc3 %0, %1, %2"
                    : /* no outputs */
                    : "g"(from), "g"(to), "g"(count)
                    : "r0","r1","r2","r3","r4","r5","memory");
                ```

            - **符号名输入输出**：

                ```c
                uint32_t Mask = 1234, Index;
                asm ("bsfl %[aMask], %[aIndex]"
                    : [aIndex] "=r"(Index)
                    : [aMask] "r"(Mask)
                    : "cc");
                ```

        ??? note "编译器与优化注意事项"

            - 若输出未使用，编译器可能删除 asm，需加 `volatile`。
            - 输入寄存器不可在 asm 内修改，除非与输出绑定。
            - `memory` clobber 形成编译器级别内存屏障，但不能阻止 CPU 推测执行。
            - 寄存器分配避免 clobber 中的寄存器。
            - 输出约束 `+` 同时算输入和输出，影响 30 个操作数上限。

        ??? note "性能与安全"

            - 使用早期 clobber (`&`) 或绑定输出-输入避免寄存器冲突。
            - 谨慎使用 `memory`，会导致寄存器刷新，影响性能。
            - 避免直接修改栈指针寄存器。

    ??? note "Specifying Registers for Local Variables"

        ??? note "基本概念"

            - 可以在函数内定义局部寄存器变量，并将其绑定到特定寄存器：

                ```c
                register int *foo asm("r12");
                ```

            - `register` 关键字必需，不能与 `static` 一起使用。
            - 寄存器名称必须是目标平台上有效的寄存器名。

        ??? note "限制与注意事项"

            - **禁止使用 `const` 和 `volatile`**：
                - `const` 可能导致编译器用初始值替换变量，使操作数分配到不同寄存器。
                - `volatile` 行为可能与预期不符。
            - 仅在调用 **Extended asm** 指定输入/输出寄存器时才支持此功能。
            - 建议选择在函数调用时 **自动保存和恢复** 的寄存器，以避免库函数调用破坏其值。

        ??? note "用法示例"

            ```c
            register int *p1 asm("r0") = ...;
            register int *p2 asm("r1") = ...;
            register int *result asm("r0");
            asm("sysint" : "=r"(result) : "0"(p1), "r"(p2));
            ```

            - 上例中，`p1` 和 `p2` 的值在 `asm` 调用中绑定到 `r0` 和 `r1`。

        ??? note "常见问题与解决"

            - **寄存器可能被后续代码或库函数调用破坏**，包括算术运算时的临时调用。
            - 解决方法：对中间表达式使用临时变量，避免直接在寄存器变量初始化中做计算：

                ```c
                int t1 = ...;
                register int *p1 asm("r0") = ...;
                register int *p2 asm("r1") = t1;
                register int *result asm("r0");
                asm("sysint" : "=r"(result) : "0"(p1), "r"(p2));
                ```

!!! question "思考题"

    下面这段 C 内联汇编存在问题：

    <iframe width="800px" height="200px" src="https://godbolt.org/e#g:!((g:!((g:!((h:codeEditor,i:(filename:'1',fontScale:14,fontUsePx:'0',j:1,lang:___c,selection:(endColumn:1,endLineNumber:19,positionColumn:1,positionLineNumber:19,selectionStartColumn:1,selectionStartLineNumber:19,startColumn:1,startLineNumber:19),source:'%23include+%3Cstdint.h%3E%0A%0Along+wrong_function(long+a,+long+b,+long+c,+long+d)+%7B%0A++++long+out1,+out2%3B%0A++++__asm__+volatile(%0A++++++++%22mv+a0,+%25%5Ba%5D%5Cn%22%0A++++++++%22mv+a1,+%25%5Bb%5D%5Cn%22%0A++++++++%22mv+a2,+%25%5Bc%5D%5Cn%22%0A++++++++%22mv+a3,+%25%5Bd%5D%5Cn%22%0A++++++++%22ecall%5Cn%22%0A++++++++%22mv+%25%5Bout1%5D,+a0%5Cn%22%0A++++++++%22mv+%25%5Bout2%5D,+a1%5Cn%22%0A++++++++:+%5Bout1%5D+%22%3Dr%22(out1),+%5Bout2%5D+%22%3Dr%22(out2)%0A++++++++:+%5Ba%5D+%22r%22(a),+%5Bb%5D+%22r%22(b),+%5Bc%5D+%22r%22(c),+%5Bd%5D+%22r%22(d)%0A++++++++:+%22memory%22%0A++++)%3B%0A++++return+out1+%2B+out2%3B%0A%7D%0A'),l:'5',n:'0',o:'C+source+%231',t:'0')),k:35.29657581185066,l:'4',n:'0',o:'',s:0,t:'0'),(g:!((h:compiler,i:(compiler:rv64-cgcctrunk,filters:(b:'0',binary:'1',binaryObject:'1',commentOnly:'0',debugCalls:'1',demangle:'0',directives:'0',execute:'1',intel:'0',libraryCode:'0',trim:'1',verboseDemangling:'0'),flagsViewOpen:'1',fontScale:14,fontUsePx:'0',j:2,lang:___c,libs:!(),options:'-O0',overrides:!(),selection:(endColumn:1,endLineNumber:1,positionColumn:1,positionLineNumber:1,selectionStartColumn:1,selectionStartLineNumber:1,startColumn:1,startLineNumber:1),source:1),l:'5',n:'0',o:'+RISC-V+(64-bits)+gcc+(trunk)+(Editor+%231)',t:'0')),k:64.70342418814936,l:'4',n:'0',o:'',s:0,t:'0')),l:'2',n:'0',o:'',t:'0')),version:4"></iframe>

    1. 请你观察右侧汇编结果，指出问题所在，并分析产生问题的原因。
    2. 给出你修正后的代码。

### SBI 与 ECALL

请阅读 Volume I: Unprivileged ISA Specification 的以下章节：

- 2.8. Environment Call and Breakpoints

    !!! question "考点"

        - ECALL 指令的作用是什么？
        - 对于我们实现的操作系统来说，服务请求的参数传递由谁定义？

请阅读 RISC-V Supervisor Binary Interface Specification 的以下章节：

- Chapter 1. Introduction

    !!! question "考点"

        - 什么是 SBI？它为谁提供服务？

- Chapter 3. Binary Encoding 的章节导言

    !!! question "考点"

        - 在本课程中，谁是 Supervisor？谁是 SEE？
        - 如何标识一个特定的 SBI 调用？
        - SBI 调用的参数和返回值是如何传递的？
        - SBI 调用时，哪些寄存器的值不会被保存？
        - 如何判断 SBI 调用是否成功？

- Chapter 12. Debug Console Extension (EID #0x4442434E "DBCN")

    !!! question "考点"

        - Debug Console Extension 提供了什么功能？
        - `sbi_debug_console_write` 函数的参数和返回值分别是什么？

### Task 2：使用 SBI 实现 `printk()`

现在你已经知道：

- 怎么写 C 内联汇编
- 怎么使用 ECALL 指令发起 SBI 调用
- `sbi_debug_console_write` 函数的参数和返回值

你的任务是：

- 在 `arch/riscv/sbi.c` 中使用内联汇编实现 `sbi_ecall()`。
- 在 `arch/riscv/sbi.c` 中补全 `sbi_debug_console_*()` 几个函数。

!!! success "完成条件"

    - 成功看到 `Hello, ZJU-OS!`。
    - 通过评测框架的 `lab1-task2` 测试。

!!! info "更多资料"

    Linux 内核的 `printk()` 与我们的简单输出不同，它实际上是打印到内核日志缓冲区（ring buffer）中，然后由专门的内核线程 `klogd` 负责将日志输出到控制台或保存到文件中。

    感兴趣的同学可以阅读 [Linux kernel printk new implementation - huang weiliang](https://huangweiliang.github.io/2024/06/05/printk/)。

## Part 2：时钟中断及其处理

### RISC-V 特权级

请阅读 Volume I: Unprivileged ISA Specification 的以下章节：

- 1.6. Exceptions, Traps, and Interrupts 的第一段（直接贴在下面了）：

    > We use the term exception to refer to an unusual condition occurring at run time associated with an instruction in the current RISC-V hart. We use the term interrupt to refer to an external asynchronous event that may cause a RISC-V hart to experience an unexpected transfer of control. We use the term trap to refer to the transfer of control to a trap handler caused by either an exception or an interrupt.

    !!! question "考点"

        - RISC-V 中 exception、interrupt 和 trap 有何区别？
        - 举两个 Trap 的例子

请阅读 Volume II: Privileged Architecture Specification 的以下章节：

- 1.2. Privilege Levels

    !!! question "考点"

        - 特权级是用来干什么的？
        - 执行当前特权级不允许的操作会发生什么？
        - M、U、S 模式分别是为了什么设计的？

### RISC-V 中断、异常与 CSR 寄存器

请阅读 Volume II: Privileged Architecture Specification 的以下章节：

- Chapter 2. Control and Status Registers (CSRs) 的章节导言

    !!! question "考点"

        - 读取、修改、写入 CSR 的指令定义在哪个扩展？
        - S 模式的 CSR 能被 M 模式访问吗？反之呢？

- Chapter 3. Machine-Level ISA

    - 3.1.6.1. Privilege and Global Interrupt-Enable Stack in **mstatus** register

        !!! question "考点"

            - mstatus 寄存器的作用是什么？
            - xIE bit 的作用是什么？
            - 运行在 S 模式且 mstatus.SIE=0，mstatus.MIE=0 时，发生中断会进入哪个模式？
            从特权级 y 陷入到更高的特权级 x 时，xPIE、xIE 和 xPP 会如何变化？
            - xRET 指令返回时，特权级和中断使能位会如何恢复？xPP 会设置为什么？

        ??? note "要点"

            1. **全局中断使能位 (Global Interrupt-Enable Bits)**

                - `MIE` → 控制 M-mode 中断的全局使能。
                - `SIE` → 控制 S-mode 中断的全局使能（如果 S-mode 未实现，则为只读 0）。
                - 当在特权级 x 下执行时：
                    - `xIE = 1` → 该特权级的中断**全局使能**。
                    - `xIE = 0` → 该特权级的中断**全局关闭**。
                - 更高特权级的中断始终使能；更低特权级的中断始终关闭。

            2. **原子性保障**

                - `xIE` 位位于 `mstatus` 低位，可用单条 CSR 指令原子地开/关中断，确保中断处理的原子性。

            3. **两级中断与特权栈 (Two-Level Stack)**

                - **xPIE**：保存陷入前 `xIE` 的值。
                - **xPP**：保存陷入前的特权级，M-mode 是 2 位，S-mode 是 1 位。
                - 陷入时：
                    - `xPIE ← xIE`
                    - `xIE ← 0`（进入陷入时中断会被关闭）
                    - `xPP ← y`（保存之前的特权级 y）

            4. **返回指令 xRET 的行为**

                - 从陷入返回时：
                    - `xIE ← xPIE`
                    - 特权级 ← `xPP`
                    - `xPIE ← 1`
                    - `xPP ← 最低支持的特权级`（帮助检测软件管理错误）
                - 如果返回的特权级 ≠ M，`MPRV` 被清零。

            5. **陷入处理的安全性要求**

                - 陷入处理必须避免在**关键状态保存阶段**开启中断或引发异常，避免：
                    - 覆盖关键状态，导致恢复失败。
                    - 陷入处理过程中出现无限递归陷入。
                - 需要小心设计，确保异常在安全的阶段被正确处理。

    - 3.1.9. Machine Interrupt (mip and mie) Registers

        !!! question "考点"

             - mip 和 mie 寄存器的作用分别是什么？
             - 在什么条件下，中断会陷入 M 模式？
             - 列举一些中断源
             - 如果中断委派到 S 模式，它在 mip 和 mie 中的行为是什么？
             - 为什么软件中断的优先级高于定时器中断？

### Task3：开启时钟中断

### RISC-V 中断异常委派

请阅读 Volume II: Privileged Architecture Specification 的以下章节：

- Chapter 3. Machine-Level ISA

    - 3.1.7. Machine Trap-Vector Base-Address (**mtvec**) Register

        !!! question "考点"

             - mtvec 寄存器的作用是什么？
             - 为什么这个寄存器的低两位可以分配给 MODE 字段？
             - 向量中断模式下，发生中断后 PC 会被设置成多少？

    - 3.1.8. Machine Trap Delegation (**medeleg** and **mideleg**) Registers

        !!! question "考点"

             - 为什么需要委派机制？
             - medeleg 和 mideleg 寄存器的作用分别是什么？
             - 当一个 trap 被委派到 S 模式后，下面这些地方的值会如何变化？
                 - scause
                 - stval
                 - sepc
                 - mstatus.SPP
                 - mstatus.SPIE
                 - mstatus.SIE
             - 如果一个 trap 是在 M 模式下发生的，但它在 medeleg 中已被设置委派给 S 模式，会在哪里处理？
             - mideleg 中的某一位被设置后，这个中断在 M 模式下会被触发吗？

### Task4：实现中断处理程序
