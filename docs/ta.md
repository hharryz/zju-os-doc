# 助教文档

本文档面向助教，学生不需要阅读。

## 开发流程

本课程的开发在 GitHub 上进行，ZJU Git 定期手工同步。

除了公开的学生代码仓库 [ZJU-OS/code](https://github.com/ZJU-OS/code) 外，还有一个私有仓库 [ZJU-OS/code-private](https://github.com/ZJU-OS/code-private)，其中 `main` 分支存放实现了所有实验功能的代码，持续进行整体代码的维护和改进。

下面从每学期初开始介绍流程。示意图中红色的节点是携带全部代码的提交，不应当公开。绿色节点是安全的，可以公开的提交。

![workflow.drawio](ta.assets/workflow.drawio)

- 创建本学期的发布分支 `fa25-release`：挖空 `main` 的代码，创建不带历史记录的分支推送到学生仓库

    ```shell
    git clone git@github.com:/ZJU-OS/code-private.git
    cd code-private
    git remote rename origin private
    git checkout -b fa25-release-clean main
    # 挖空代码
    git commit -am "fa25: init"
    git checkout --orphan fa25-release
    git commit -am "fa25: init"
    git push -u private fa25-release
    ```

- 创建空的学生仓库，假设为 `code-fa25`：

    ```shell
    git remote add upstream git@github.com:/ZJU-OS/code-fa25.git
    ```

- 开发工作在 `private/main` 进行
- 发布 `labN`：
    - 以 `fa25-release` 为基础（此时已经发布了 `lab(N-1)`）,从 `private/main` 选择需要的代码进行合并，挖空不需要的代码，达到 `labN` 的进度
    - 创建 `labN` 分支并推送到学生仓库

    ```shell
    git checkout fa25-release
    git merge --squash --no-commit private/main
    # 此时添加了 private/main 中的完整更改，但即使没有冲突也不会提交
    # 手工选择需要合并的文件，挖空学生完成的代码
    # 可以分多次提交，更加清晰地添加文件
    git commit -am "labN: release"
    git checkout -b labN
    git push upstream labN
    ```
