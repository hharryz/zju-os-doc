# FAQ

本文档记录实验过程中同学们遇到的常见问题及解决办法。

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
