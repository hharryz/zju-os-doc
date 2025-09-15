# 助教文档

本文档面向助教，学生不需要阅读。

- 助教代码仓库（非公开）见 [ZJU-OS/code-private](https://github.com/ZJU-OS/code-private)。
- 助教工具仓库见 [ZJU-OS/tool](https://github.com/ZJU-OS/tool)。

## 助教仓库开发流程

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
    # 以当前工作目录为基础，比较 main 分支新增哪些代码
    git diff -R --compact-summary main --
    # 从主干分支合并代码，可以用 git merge 或 git checkout
    git merge --squash --no-commit private/main
    # 此时添加了 private/main 中的完整更改，但即使没有冲突也不会提交
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

## 多平台镜像构建

课程需要支持使用 amd64 和 arm64 平台的学生完成实验，因此使用 [Multi-arch build and images, the simple way | Docker](https://www.docker.com/blog/multi-arch-build-and-images-the-simple-way/) 构建多平台镜像，流程如下：

- 分别在 arm64 和 amd64 平台上执行

    ```shell
    make build-image
    make push-image
    ```

- 然后在其中一个地方执行

    ```shell
    make push-multi
    ```

    拉取在另一个平台上构建的镜像，使用 `docker manifest` 合成 `latest` 描述文件并推送。

相比用 Tag 区分不同架构，这么折腾一番可以让 Docker 自动匹配到对应架构，镜像名不用指定标签。

理论上来说，使用 Docker buildx 的 [Multi-platform builds](https://docs.docker.com/build/building/multi-platform/) 就能在一个平台上完成所有构建，更加简洁。但是：

- 笔者没有配置成功
- 跨架构运行实在是太慢了

但 Manifest 是 Docker 的实验性功能，有时候创建 `latest` 并不覆盖，挺奇怪的。
