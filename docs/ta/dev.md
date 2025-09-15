# 助教开发手册

本课程的开发在 GitHub 上进行，ZJU Git 定期手工同步。

## 仓库列表

- （公开）实验代码发布仓库 [ZJU-OS/code](https://github.com/ZJU-OS/code)

    用于发布实验代码。设置了严格的分支保护、合并、推送策略，确保代码规范安全。

- （公开）实验代码中转仓库 [ZJU-OS/code-transit](https://github.com/ZJU-OS/code-transit)

    因为 GitHub 不支持 Public 仓库的 Private Fork，**为了走 PR 流程，确保参与的助教知道发布的代码发生了什么变更**，我们创建了一个中转仓库。助教需要将准备发布的代码推送到中转仓库，走 PR 流程后再合并到发布仓库。

- （私有）完整代码仓库 [ZJU-OS/code-private](https://github.com/ZJU-OS/code-private)

    `main` 分支存放实现了所有实验功能的代码，持续进行整体代码的维护和改进。

## 发布流程

下面从每学期初开始介绍流程。示意图中红色的节点是携带全部代码的提交，不应当公开。绿色节点是安全的，可以公开的提交。

![workflow.drawio](workflow.drawio)

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
    git push -u code-private fa25-release
    ```

- 创建空的中转仓库和发布仓库，推送第一个基础分支 `lab0`（仅此时不需要走 PR）：

    ```shell
    git remote add code-transit git@github.com:/ZJU-OS/code-transit.git
    git remote add code git@github.com:/ZJU-OS/code.git
    git push code-transit lab0
    git push code lab0
    ```

- `code-private/main` 进行其他开发工作
- 发布 `labN`：
    - 以 `fa25-release` 为基础（此时已经发布了 `lab(N-1)`）,从 `private/main` 选择需要的代码进行合并，挖空不需要的代码，达到 `labN` 的进度
    - 创建 `labN` 分支并推送到学生仓库

    ```shell
    git checkout fa25-release
    # 以当前工作目录为基础，比较 main 分支新增哪些代码
    git diff -R --compact-summary main --
    # 从 code-private/main 合并代码到 code-private/fa25-release
    # 可以用 git merge 或 git checkout
    git merge --squash --no-commit main
    git checkout private/main -- <需要的文件>
    # 手工选择需要合并的文件，挖空学生完成的代码
    # 可以分多次提交，更加清晰地添加文件
    git commit -am "labN: release"
    git checkout -b labN
    git push upstream labN
    ```

如果今后参与开发的助教增多，为了防止代码不慎泄露，建议：

- 大部分助教仅在 `private` 进行单仓库的开发、测试、合并到发布分支等任务
- 推送到 `upstream` 的操作由专人执行
- `upstream` 仓库设置分支保护策略，避免 `main` 分支被误推送到其中
