# FAQ

本文档记录实验过程中同学们遇到的常见问题及解决办法，会频繁更新。如果你遇到问题，可以先来本文档找找是不是已经存在。

## Lab1

### Clangd 出现问题

- 现象：

    VSCode Clangd 插件报错，提示找不到头文件或语法解析错误等，但你确认该错误不应存在。

- 原因：

    这往往是因为 Clangd 没有重建索引。

- 解决办法：

    按 ++ctrl+shift+p++，输入 `clangd`，点击 `clangd: Restart language server` 重载语法解析器。

## Lab 0

### 无法连接到 Docker 守护进程

- 现象：

    ```text
    Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
    ```

- 原因：

    Docker 的工作原理是客户端-服务器模式。在终端中执行的是 Docker CLI 客户端，它需要连接到服务器即 Docker Daemon（守护进程）来完成实际的工作，比如拉取、启动容器等。错误信息表明 Docker Daemon 没有启动。

    该问题常见于在 WSL 中初次安装 Docker Engine 的场景，此时 Docker Daemon 可能尚未自动启动。

- 解决办法：

    - Linux：

        在现代 Linux 发行版中，一般使用 [systemd](https://www.freedesktop.org/wiki/Software/systemd/) 来管理系统服务。其中，`systemctl` 用于管理服务，`journalctl` 用于管理日志。

        ```shell
        # 检查 Docker 服务状态
        sudo systemctl status docker
        # 如果输出不是 active (running)，则启动 Docker 服务
        sudo systemctl enable --now docker
        # 如果输出是其他状态（如 failed），可以将日志信息喂给 AI 寻求帮助
        sudo journalctl -u docker --no-pager | tail -n 50
        ```

    - WSL：

        如果你运行的是较老版本的 WSL，则 systemd 可能不可用。推荐你按照 [Use systemd to manage Linux services with WSL | Microsoft Learn](https://learn.microsoft.com/en-us/windows/wsl/systemd) 的指引升级 WSL 版本，启用 systemd。

        如果你不想升级，重启 WSL 应该也能解决问题。在 Windows 终端中执行：

        ```powershell
        wsl --shutdown
        ```

        然后重新打开 WSL。

    - macOS：

        确保你启动了 [Docker Desktop for Mac](https://www.docker.com/products/docker-desktop)。

### 拉取容器时 ZJU Git EOF

- 现象：

    ```text
    Error response from daemon: Get "https://git.zju.edu.cn:5050/v2/": EOF
    ```

- 原因：

    网络问题，Docker Daemon 无法连接到 ZJU Git。这往往是由于代理设置不正确导致的。

    使用 Windows Docker Desktop 的同学反映，即使自己关闭了 Clash，代理设置也会残留在 Docker Desktop 中，运行 `docker info` 可以看到下面的内容：

    ```text
    HTTP Proxy: ...
    HTTPS Proxy: ...
    ```

- 解决办法：

    - Windows Docker Desktop 用户：在 Docker Desktop 设置界面进入 `Resources` -> `Proxies`，打开 `Manual proxy configuration`，将所有代理设置留空，然后点击 `Apply`，重启 Docker Desktop。

### Git not a valid repository name

- 现象：

    ```console
    git clone git@git.zju.edu.cn:...
    fatal: remote error:
      ... is not a valid repository name
    Visit https://support.github.com/ for help
    ```

- 原因：

    从报错可以看到是 GitHub 返回的错误，但我们的仓库在 ZJU Git。这一般是 Git 配置错误导致的，运行 `git config --list` 检查是否有错误的配置。

    目前发现有错误配置如下：

    ```text
    core.sshcommand=ssh -T -p 443 -o Hostname=ssh.github.com
    ```

- 解决办法：

    删除相关错误配置：

    ```console
    git config --global --unset core.sshcommand
    ```
