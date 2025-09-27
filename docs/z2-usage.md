# 附录 2：构建、运行和评测

## Makefile

课程仓库根目录下的 `Makefile` 将常用操作包装为 Make 目标，方便在命令行中运行。它会检测当前环境是否存在 `docker` 命令，如果存在认为是宿主机，否则认为是在容器内运行。不同环境下的目标有所不同：

- 在宿主机上
    - （默认目标）`all`：创建、启动并进入容器
    - `clean`：删除容器
    - `update`：删除并更新容器镜像
- 在容器内
    - `all`：清理并构建内核
    - `clean`：清理构建产物
    - `run`：使用 QEMU 运行内核
    - `debug`：使用 QEMU 运行内核（调试）
    - `gdb`：启动 GDB 并连接到 QEMU（与 `debug` 配合使用）
    - `judge`：运行下文介绍的评测

## 评测框架

评测框架作为 Git Submodule 放置在课程代码仓库的 `autograder` 目录下。运行方式为：

```console
uv --project autograder run autograder [--lab labN]
```

[uv](https://github.com/astral-sh/uv) 是一个流行的 Python 包管理器，评测框架使用它管理环境并运行。

不给定参数的情况下，评测框架会检查当前的 Git 分支名并运行对应的实验评测（例如 `lab1*` 分支会运行 lab1 的评测）。也可以通过 `--lab` 参数指定要运行的实验。
