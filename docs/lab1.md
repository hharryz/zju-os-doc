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

完成这些，你就能够启动自己的内核，并打印 `Hello, kernel!` 了。

- 我们的下一个目标是为 Lab2 **线程调度**做准备：

    操作系统是**事件驱动**的，意味着系统的执行流是由各种事件触发的，例如 I/O 完成、进程创建、用户输入等。时钟中断是其中最基础、最重要的事件源之一。**如果没有时钟中断，操作系统将无法感知时间的流逝，无法在合适的时机进行进程切换、调度、超时处理或实现延时等待。**通过周期性触发的时钟中断，内核可以被动地转变为主动：不再依赖进程自愿交出 CPU，而是能在中断到来时**打断当前执行流**，检查是否有更高优先级的任务或等待超时的事件需要处理，从而保证多任务系统的公平性和响应性。

    那么有如下问题：

    - RISC-V 是如何设计中断与异常的？怎么开启时钟中断？这需要你学习**RISC-V 特权级、中断与异常、CSR 寄存器**。
    - 时钟中断可能在任何时候发生。应该如何实现中断处理程序，保存和恢复现场，从而保证内核和用户程序的正确执行？这需要你学习**RISC-V 中断异常委派、中断处理**，然后补全 `arch/riscv/kernel/entry.S`。

完成这些，你就能做 Lab2 了。

## Part 1：启动工作

### OpenSBI 调试

<在内核第一条指令处设置断点，查看当前环境状态>

### RISC-V 调用约定

同学们上次熟悉 RISC-V 汇编与 C 语言的关系，应该是在《计算机组成》课程中。上《计算机体系结构》的时候，同学们只烧二进制 Bit 文件，应该已经把相关内容忘得差不多了。

<figure markdown="span">
    ![co.webp](lab1.assets/co.webp)
    <figcaption>
    《计算机组成》课程 PPT
    </figcaption>
</figure>

使用 GCC 编译器编译一个简单的程序，得到的汇编如下（请点击 Edit on  Compiler Explorer 放大查看）：

<iframe width="800px" height="200px" src="https://godbolt.org/e#g:!((g:!((g:!((h:codeEditor,i:(filename:'1',fontScale:14,fontUsePx:'0',j:1,lang:c%2B%2B,selection:(endColumn:2,endLineNumber:6,positionColumn:2,positionLineNumber:6,selectionStartColumn:2,selectionStartLineNumber:6,startColumn:2,startLineNumber:6),source:'int+func(int+a)%7Breturn+a%3B%7D%0A%0Aint+main()+%7B%0A++int+r+%3D+func(10)%3B%0A++return+r+%2B+1%3B%0A%7D'),l:'5',n:'0',o:'C%2B%2B+source+%231',t:'0')),k:50,l:'4',n:'0',o:'',s:0,t:'0'),(g:!((h:compiler,i:(compiler:rv64-gcctrunk,filters:(b:'0',binary:'1',binaryObject:'1',commentOnly:'0',debugCalls:'1',demangle:'0',directives:'0',execute:'1',intel:'0',libraryCode:'0',trim:'1',verboseDemangling:'0'),flagsViewOpen:'1',fontScale:14,fontUsePx:'0',j:1,lang:c%2B%2B,libs:!(),options:'',overrides:!(),selection:(endColumn:27,endLineNumber:7,positionColumn:27,positionLineNumber:7,selectionStartColumn:27,selectionStartLineNumber:7,startColumn:27,startLineNumber:7),source:1),l:'5',n:'0',o:'+RISC-V+(64-bits)+gcc+(trunk)+(Editor+%231)',t:'0')),k:50,l:'4',n:'0',o:'',s:0,t:'0')),l:'2',n:'0',o:'',t:'0')),version:4"></iframe>

请回忆你在《计算机组成》课程中学习的相关内容，回答下面的问题。如果有想不来的地方，请阅读 [RISC-V ELF psABI Document](spec/riscv-abi.pdf) 的以下章节：

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

### 链接器脚本与内核内存布局

在 C 语言编程课上，我们了解过编译器的基本流程：预处理、编译、汇编、链接。具体命令例：

```shell
riscv64-linux-gnu-gcc -S main.c -o main.S
riscv64-linux-gnu-as main.S -o main.o
riscv64-linux-gnu-gcc -S func.c -o func.S
riscv64-linux-gnu-as func.S -o func.o
riscv64-linux-gnu-ld main.o func.o -o main
```

链接过程合并多个目标文件，并**重定位**它们的数据，**链接**各类符号（函数名、变量名等）。链接器 `ld` 通过**链接脚本**来控制最终的生成的文件，默认使用内置的脚本生成 ELF 格式的可执行文件，你可以使用下面的命令查看这个脚本：

```shell
riscv64-linux-gnu-ld --verbose
```

请阅读 [3 Linker Scripts - LD](https://sourceware.org/binutils/docs/ld.html#Scripts) 的以下章节，学习基本的链接器脚本语法：

- 3.1 [Basic Linker Script Concepts](https://sourceware.org/binutils/docs/ld.html#Basic-Linker-Script-Concepts)
- 3.3 [Simple Linker Script Example](https://sourceware.org/binutils/docs/ld.html#Simple-Linker-Script-Example)

??? note "要点"

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

现在，我们来简单对比一下 Linux 内核链接脚本和 `ld` 内置链接脚本的区别：

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

- **起始位置**：ELF 程序的在 `0x10000 + SIZEOF_HEADERS`，而内核是 `0xffffffff80000000`。学习虚拟内存后你会理解这些地址受[操作系统内存布局](https://docs.kernel.org/arch/riscv/vm-layout.html)影响。
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

至于具体怎么做并不重要，感兴趣的同学可以自行结合 `ld` 文档，阅读实验代码和内核中的 `arch/riscv/kernel/vmlinux.lds`。

### Task 1：为 `start_kernel()` 准备运行环境

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

### C 内联汇编

### Task 2：使用 SBI 实现 `printk()`

请阅读 RISC-V Supervisor Binary Interface Specification 的以下章节：

- Chapter 12. Debug Console Extension (EID #0x4442434E "DBCN")

在 `arch/riscv/sbi.c` 中使用内联汇编实现 `sbi_debug_console_write`。

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
            当从特权级 y 陷入到更高的特权级 x 时，xPIE、xIE 和 xPP 会如何变化？
             - xRET 指令返回时，特权级和中断使能位会如何恢复？xPP 会设置为什么？

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
