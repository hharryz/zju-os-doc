# FAQ

本文档记录实验过程中同学们遇到的常见问题及解决办法，会频繁更新。如果你遇到问题，可以先来本文档找找是不是已经存在。

## Lab 0

### 运行 docker 命令时 ZJU Git EOF

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
