# 附录 2：工具使用

实验指南中仅对使用到的工具作必要的介绍，无法涵盖所有细节，而且有一些零散。

本文档按照实验流程的不同环节划分，尽可能详细地介绍相关工具的使用方法和注意事项，供基础薄弱或有兴趣探究的同学参考。

根据调查问卷，实验中默认同学们掌握下列技能：

- 拥有 Linux/macOS 环境，具备 Linux 基础操作能力
- 了解 Git、Makefile 的基本用法

## 源代码管理

### ZJU Git

本课程统一使用 [ZJU Git](https://git.zju.edu.cn/) 作为代码管理平台。同学们需要使用指定的仓库提交代码、通过评测。未使用过 ZJU Git 的同学请前往[登录界面](https://git.zju.edu.cn/users/sign_in)创建账号。同时 [ZJU Git 101](https://www.pages.zjusct.io/git101/) 提供了一些使用帮助。

### Git

基本的 Git 使用和配置不作介绍。如果你想深入学习 Git，可以阅读这本官方教程：英文版 [Git](https://git-scm.com/book/en/)、中文版 [Pro Git 中文版（第二版）](https://www.progit.cn/)。

Git 有两种连接方式：`git@...`（SSH）和 `https://...`（HTTPS）。推荐使用 SSH 方式，避免每次操作涉及远程仓库时都需要输入用户名和密码。这需要在 ZJU Git 上添加 SSH 公钥，见 [设置 SSH 访问 - ZJU Git 101](https://www.pages.zjusct.io/git101/prep/setup_ssh.html)。

实验评测框架作为 Git Submodule 放置在课程代码仓库的 `autograder` 目录下，所以同学们需要了解 Submodule 的基本使用方法，详见官方教程 [Git - Submodules](https://git-scm.com/book/en/v2/Git-Tools-Submodules)。简单来说：

- `git clone` 后还需要运行下面的命令来初始化并更新 Submodule：

    ```shell
    git submodule update --init --recursive
    ```

- Submodule 中内容作为一个独立的 Git 仓库管理，父仓库只记录 Submodule 的某个提交 ID。

!!! tip "Tip：WSL 用户请不要将代码存放在 Windows 目录下"

    建议 WSL 用户直接在 WSL 而不是 Windows 中使用 Git。

    WSL 用户应该了解：**Linux 和 Windows 的文件系统不同，文件权限、[换行符](https://stackoverflow.com/questions/1552749/difference-between-cr-lf-lf-and-cr-line-break-types)、链接、[大小写](https://stackoverflow.com/questions/33998669/windows-ntfs-and-case-sensitivity)等方面都存在区别。**

    虽然 WSL 将 Windows 目录挂载到了 `/mnt/c` 等路径下，但这只是为了方便文件互访。官方并不建议你将文件**存放**在一边，又在另一边**使用**。首先性能较低，其次代码仓库等对文件权限、链接等有要求的项目很容易出问题。

    请 WSL 用户将代码仓库克隆到 WSL 的 Linux 文件系统中（比如 `/home/<username>/` 下），以避免出现各种奇怪的问题。

!!! question "FAQ：Git 用户名和邮箱如何配置？"

    其实设成什么都没关系，因为仓库的地址确定，助教会去对应的仓库看你的提交。

    Git 平台（如 GitHub、GitLab、ZJU Git）会根据你 commit 中的邮箱来关联到你的账号，从而显示头像等信息。如果你想让 commit 显示正确的头像，可以将 `user.email` 设置成你在 ZJU Git 上绑定的邮箱。

### VSCode Git 插件

推荐使用 VSCode 内置的 Git（源代码管理面板）进行相关操作，官方文档见 [Using Git source control in VS Code](https://code.visualstudio.com/docs/sourcecontrol/overview)。

此外，你还可以安装 [Git Graph](https://marketplace.visualstudio.com/items?itemName=mhutchie.git-graph) 插件来可视化地查看分支、提交等信息，进行 Git 操作。

### 本课程的 Git 工作流

实验过程涉及两个仓库：

- **公开发布仓库：** `https://git.zju.edu.cn/os/code.git`
    - `lab0`、`lab1`、... 分支：各个 Lab 的代码，位于同一个[线性历史（linear history）](https://stackoverflow.com/questions/20348629/what-are-the-advantages-of-keeping-linear-history-in-git)上。
- **学生私有仓库：** `git@git.zju.edu.cn:os/<学期>/<教学班>/os-<你的学号>.git`
    - `lab0`、`lab1`、... 分支：学生提交的代码。这些分支设置了保护策略，禁止强制推送，DDL 截止后无法再向对应分支提交代码，成绩评定以对应分支的代码为准。
    - `lab0-makeup`、`lab1-makeup`、... 分支：补交分支，DDL 截止后你可以继续向这些分支提交代码，助教会在期末进行统一评定

下图展示了本课程的 Git 工作流和相应命令：

![git.drawio](z2.assets/git.drawio)

其中从上游合并代码的步骤可以自行选用 Merge 或 Rebase。这一步可能遇到冲突，你需要理解冲突部分的代码的作用，并决定保留、合并或是手动整理代码。

## 容器环境

### Docker

如果你想学习 Docker，可以阅读官方教程 [Docker 101 Tutorial | Docker](https://www.docker.com/101-tutorial/)。

简单来说，**容器**是一种轻量级的虚拟化技术，**Docker** 是一个容器平台。容器和虚拟机都能够将应用程序及其依赖打包在一个独立的环境中运行，与宿主机隔离开来。区别在于，虚拟机需要完整的操作系统，而容器只需要包含应用程序及其依赖，利用宿主机的内核来运行，因此更加轻量和高效。

本课程使用 Docker 容器的流程如下：

- 助教将所需的环境打包成一个**镜像（image）**，存放到 ZJUGit **Registry（存放镜像的地方）**。- 同学们从 Registry **拉取（Pull）** 镜像，然后基于该镜像创建并启动一个**容器（container）**。

这为我们带来了以下好处：

- 简单：
    - 同学们不用手动配环境了，只需要配 Docker。
    - 环境搞坏了就删掉重建一个新的，非常方便。
- 隔离：环境和宿主机隔离，不会彼此污染，不影响你做其他事情。
- 一致：所有人的环境都是从同一个镜像创建的，避免了绝大多数“在我电脑上能跑”的问题。

如果你尚未安装 Docker：

- Linux 和 WSL 环境请参考 [Docker CE | ZJU Mirror](https://mirrors.zju.edu.cn/docs/docker-ce/)
- macOS 推荐使用 [Docker Desktop](https://docs.docker.com/desktop/setup/install/mac-install/) 或 [OrbStack](https://orbstack.dev/download) (可以用 [Homebrew](https://formulae.brew.sh/formula/docker) 直接安装)

!!! tips "Tips：Windows 用户请避雷 Docker Desktop"

    Windows 用户应该在 WSL 中直接安装 Docker Engine，而不是在 Windows 上使用 Docker Desktop。

### Docker Compose

在日常开发中，如果直接使用 `docker run` 启动容器，命令会非常冗长，需要手动指定镜像、端口、挂载、环境变量等参数，既不直观，也不方便维护。**Docker Compose** 提供了一种更简单的方式：只需在一个 `compose.yml` 文件里定义好服务、镜像和配置，之后用一条命令就能启动整个开发环境。

在我们的仓库中，已经为大家准备好了 `compose.yml`，里面定义了容器的运行方式和一些必要的配置。下面是一些重要的配置点：

1. 代码挂载

    代码库会被**挂载（mount）**到容器内的 `/zju-os/code` 目录下。
    这意味着：

    - 宿主机和容器**共享同一套代码文件**，修改会实时同步。
    - 代码文件实际仍保存在宿主机上，**删除容器不会导致代码丢失**。

    这样做的好处是，大家在容器内开发时，无需担心数据持久化问题，调试体验和本地几乎一样。

2. 用户与文件权限

    容器内默认用户是 **`root`**，而宿主机上你一般是普通用户。
    当你在容器内生成文件（比如编译产物）时，这些文件在宿主机上会显示为 `root` 所有，这可能会导致你在宿主机直接编辑或删除文件时遇到**权限不足**的问题。

    解决方法：在宿主机执行以下命令，把文件的所有者改回普通用户：

    ```shell
    sudo chown -R <username>:<username> .
    ```

    请将 `<username>` 替换为你在宿主机上的用户名。

3. Git 和 SSH 配置映射

    特别地，执行一些 Git 操作也会产生文件，因此会产生这样的情况：在容器内执行 Git 操作后，在宿主机执行 Git 操作遇到 Permission Denied。所以我们将宿主机的 Git 和 SSH 配置映射到容器内，这样同学们的开发、Git 操作都在容器内进行，不用回到宿主机了。

    为此，我们在 `compose.yml` 中做了以下处理：

    - 将宿主机的 `~/.gitconfig` 和 `~/.ssh` 挂载到容器内。
    - 确保你在宿主机上已经配置好 Git 用户信息和 SSH Key。

    这样，所有的 Git 操作都可以在容器内完成，免去了来回切换的麻烦。

### DevContainer

[Development containers](https://containers.dev/) 是一个由 Microsoft 开源的项目，旨在通过 `devcontainer.json` 文件定义和配置开发容器环境。它与 Docker Compose 类似，但更专注于开发环境的配置和集成，尤其是与 VSCode 的无缝对接。

在我们的仓库中，已经为大家准备好了 `.devcontainer/devcontainer.json` 文件。它会使用上一节介绍的 `compose.yml`，并对 VSCode 配置插件等。

下面以 VSCode 为例介绍使用流程：

- VSCode 安装 [Dev Containers 插件](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)
- {==VSCode 打开实验仓库==}
- 右下角可能会出现**开发容器（Dev Container）相关的弹窗**，点击在开发容器中打开

    如果没有弹窗，则按 ++ctrl+shift+p++ 打开命令窗口，输入 `reopen` 找到 `Dev Containers: Reopen in Container` 选项，选择它

    !!! tip

        VSCode 是从当前打开的文件夹找 `.devcontainer` 的，所以请确保 VSCode 当前打开了实验仓库的根目录。

        如果选择 `Reopen in Container` 后 VSCode 弹出了选择容器之类的窗口，说明你没有打开正确的目录。请尝试关闭 VSCode 重新打开。

- VSCode 将重载窗口，启动并连接到容器，自动完成插件安装等配置步骤

    !!! tip

        实验镜像比较大（约 10G），你可以结合自己的网速评估一下需要的拉取时间，耐心等待。

        右下角应该会有弹窗表示开发容器正在启动，你可以点击查看日志了解启动进度。

## 编写代码

### Clangd

VSCode 自带的 C/C++ 插件 IntelliSense 解析时容易受到系统头文件的影响，产生一堆报错，在内核编程等特殊场景下表现不佳。我们推荐使用如下工具：

- [clangd](https://clangd.llvm.org/)：一个基于 LLVM 的 C/C++ 语言服务器，提供智能补全、语法检查等功能。它直接使用编译命令生成的编译数据库 `compile_commands.json`，能准确地解析代码关联。
- [bear](https://github.com/rizsotto/Bear)：一个生成 `compile_commands.json` 的工具，能将 Makefile 生成的编译命令转换为 JSON 格式，供 clangd 使用。

相关工具已经内置在课程使用的容器环境中，如果你后续使用 DevContainer 开发将会自动启用相关 VSCode 插件。

## 构建和运行

### Makefile

课程仓库根目录下的 `Makefile` 将常用操作包装为 Make 目标，方便运行。它会检测当前环境是否存在 `docker` 命令，如果存在认为是宿主机，否则认为是在容器内运行。不同环境下的目标有所不同：

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
    - `format`：格式化代码，请在提交代码前运行

### QEMU

已在 Lab0 介绍。

## 调试

### GDB

已在 Lab0 介绍。

### VSCode 调试器

VSCode 自带的调试器界面能够对接 GDB 等调试器，方便地进行各类操作，比如设置断点、单步执行、查看数据和汇编指令等。官方使用文档见 [Debug code with Visual Studio Code](https://code.visualstudio.com/docs/debugtest/debugging)，官方配置文档见 [Visual Studio Code debug configuration](https://code.visualstudio.com/docs/debugtest/debugging-configuration)。

**推荐同学们使用 VSCode GUI 调试，这样会比 GDB 命令更加高效**。但是，在虚拟内存等实验中，可能遇到代码实现不完善导致 GDB 无法调试（GDB 受虚拟内存影响），这时候请记得 QEMU Monitor 是你的救命稻草。

在我们的仓库中，已经为大家准备好了 `.vscode/launch.json`，里面定义了调试配置。

VSCode 的官方文档演示了调试器的基本使用，包括：启动调试、设置断点、查看数据等。下面对本课程的使用做一些补充：

- **启动调试：**在终端中 `make debug` 启动 QEMU 调试，再点击 VSCode 调试器的启动按钮，VSCode 会使用仓库中的配置文件启动 `gdb-multiarch` 连接到 QEMU 并进行调试。
- **查看汇编：**++ctrl+shift+p++ 打开命令面板，输入 `assem` 找到 `Open Disassembly View`。

## 评测

评测框架作为 Git Submodule 放置在课程代码仓库的 `autograder` 目录下。运行方式为：

```console
uv --project autograder run autograder [--lab labN]
```

[uv](https://github.com/astral-sh/uv) 是一个流行的 Python 包管理器，评测框架使用它管理环境并运行。

不给定参数的情况下，评测框架会检查当前的 Git 分支名并运行对应的实验评测（例如 `lab1*` 分支会运行 lab1 的评测）。也可以通过 `--lab` 参数指定要运行的实验。
