# 附录 1：Spike 工具链

感兴趣的同学可以根据本文档自行使用 spike 工具链运行 RISC-V kernel。

理论上，Spike 和 QEMU 的运行结果应当相同。如果你发现运行结果不同、出错等情况，请向助教反馈并带上你的探究结果，可以获得加分。

## 工具链简介

### Spike

[Spike](https://github.com/riscv-software-src/riscv-isa-sim/) 是一个和 QEMU 类似的指令模拟器。虽然它不像 QEMU 那样支持各种指令集架构，可以模拟复杂的外设，但是在 RISC-V 的模拟和支持上，spike 是逐条指令进行模拟，且严格按照 RISC-V 的指令集手册，对于 RISC-V 程序的检查更为严谨。

!!! note "特别是在页表的实验中，spike 会严谨得多，推荐有精力的同学额外用 spike 运行进行测试"

QEMU 为了追求运行的高效，往往会将指令分块打包编译为宿主机指令高效运行；而 spike 则选择了根据 RISC-V 指令集充分模拟 RISC-V 体系结构的各种硬件，然后逐条运行指令，并修改各个模拟硬件，当然这也会导致 spike 运行速度比 qemu 慢一些。

如果同学们想要了解一些 RISC-V 机制的具体实现，直接阅读 spike 的源码也是不错的选择。

### OpenOCD

[OpenOCD](https://github.com/openocd-org/openocd/) 是一个调试的中间工具。一些 RISC-V 芯片为了方便硬件调试内部的信息会在内部集成一个 debug module，并对外暴露一个 JTAG 接口。调试者可以用 JTAG 线的输入线向 debug module 输入命令，并用输出线得到需要读取的结果。这个指令传输协议是复杂的，我们本能地希望可以有一个简单的命令工具，我们向他发送诸如 read、write 的指令，它再帮我们转换为晦涩的 DMI 指令序列，这个工具就是 OpenOCD。

Spike 直接模拟的是硬件设备，所以它也直接模拟了一个 debug module，指定 `--rbb-port` 选项之后，它就可以开放对应的模拟 JTAG 端口等待被连接调试。OpenOCD 连接这个端口然后开始调试，不过它并不知道我们的 spike 的平台类型的连接类型，因此需要我们额外提供 openocd.cfg 文件，只要运行 `openocd -f openocd.cfg`，就会自动连接 9824 端口，并准备调试我们的 spike。

它连接了 remote bitbang 的端口，然后创建了一个 tap 和 target，开启了 gdb 后重置等待调试。这样 OpenOCD 会默认开放 3333 端口给 gdb 连接，通过 `target extended-remote localhost:3333` 或者 `tar ext :3333` 就可以连接到 OpenOCD。

之后 OpenOCD 可以担当 gdb 和 spike 之间的桥梁。gdb 接收到的指令会发送给 OpenOCD，OpenOCD 将它转换为对应的 01 串发送给 spike 进行调试，反之亦然。于是我们可以使用 gdb-OpenOCD-spike 的工具链实现原来 gdb-qemu 的效果。

### OpenSBI

OpenSBI 大家在实验过程中应该都有所了解，就不介绍它本身了。这个工具链之所以需要 OpenSBI，是因为 spike 模拟的是一台裸机。我们的 kernel 不可以直接在裸机上运行，所以我们需要一个额外的 OpenSBI 的代码做前期的启动。

## 运行

运行 `make` 时添加 `SIMULATOR=spike` 选项即可：

```shell
make SIMULATOR=spike run
```

## 调试

调试的话一共需要三个终端窗口，分别执行：

1. `make SIMULATOR=spike debug`：启动 spike 并等待调试
2. `make ocd`：启动 OpenOCD 并连接 spike
3. `make gdb`：启动 gdb

后续的调试流程和 qemu 类似，不再赘述。
