# 附录 2：评测框架说明

评测框架作为 Git Submodule 放置在课程代码仓库的 `autograder` 目录下。运行方式为：

```console
uv --project autograder run autograder [--lab labN]
```

[uv](https://github.com/astral-sh/uv) 是一个流行的 Python 包管理器，评测框架使用它管理环境并运行。

不给定参数的情况下，评测框架会检查当前的 Git 分支名并运行对应的实验评测（例如 `lab1*` 分支会运行 lab1 的评测）。也可以通过 `--lab` 参数指定要运行的实验。
