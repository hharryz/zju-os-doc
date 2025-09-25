# 评测框架

评测框架使用了下列开源项目：

- [pygdbmi · PyPI](https://pypi.org/project/pygdbmi/)：与 GDB 交互
- [QEMU Python Tooling](https://github.com/qemu/qemu/tree/master/python)

    QEMU 的 Python API 并没有独立出来作为一个库发布到 PyPI 上，而是和 QEMU 的源代码放在一起。因此我们在 Docker 镜像构建时用 APT 获取当前发行版的 QEMU 源码，并在 `pyptoject.toml` 中指定。

    !!! danger "注意更新"

        QEMU Python Tooling 并不保证稳定性。每年课程更新重构 Docker 镜像时，都需要检查一下评测框架是否还能用、QEMU Python API 是否有 Breaking Change。

    我们使用了其中的：

    - `qemu.machine`：创建和管理虚拟机
    - `qemu.qmp`：使用 QEMU Machine Protocol (QMP) 与虚拟机交互

    建议阅读以下资料了解如何使用这些 API：

    - [Functional testing with Python — QEMU documentation](https://www.qemu.org/docs/master/devel/testing/functional.html)
    - [`tests/functional/riscv64/test_opensbi.py`](https://github.com/qemu/qemu/blob/master/tests/functional/riscv64/test_opensbi.py)

    其他信息：

    - QEMU 开发者 John Snow 曾经为 QEMU Python API 写过一些文档，见 [QEMU Python Library Documentation](https://people.redhat.com/~jsnow/sphinx/html/index.html)，但上次更新已是 2021 年。
    - John Snow 将 `qemu.qmp` 分离出来打包，但 PyPI 上次更新已是 2023 年，也不知道为什么不顺便把 `qemu.machine` 之类的打包出来。

        文档见 [qemu.qmp: QEMU Monitor Protocol Library — QEMU Monitor Protocol Library 0.0.4.dev28+gc08fb82b3 documentation](https://qemu.readthedocs.io/projects/python-qemu-qmp/en/latest/main.html)，PyPI 见 [qemu.qmp · PyPI](https://pypi.org/project/qemu.qmp/)，源码仓见 [QEMU / python-qemu-qmp · GitLab](https://gitlab.com/qemu-project/python-qemu-qmp)。

- [unittest — Unit testing framework — Python documentation](https://docs.python.org/3/library/unittest.html)：Python 标准库的单元测试框架

## QEMU

```text
-display none -vga none
-chardev socket,id=mon,fd=6
-mon chardev=mon,mode=control
-machine virt
-chardev socket,id=console,fd=14
-serial chardev:console
```

- `-mon`：给 QEMU Monitor 用的
- `-serial`：给 OpenSBI 用的串口
- 这两个参数组合起来起到和 `-nographic` 类似的作用
