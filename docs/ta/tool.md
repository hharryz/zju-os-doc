# 助教工具手册

这是助教工具仓库 [ZJU-OS/tool](https://github.com/ZJU-OS/tool) 的文档。

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
