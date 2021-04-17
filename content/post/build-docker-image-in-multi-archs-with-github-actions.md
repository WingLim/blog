---
title: "使用GiHub Actions构建多架构Docker镜像"
date: 2021-04-17T15:00:52+08:00
slug: build-docker-image-in-multi-archs-with-github-actions
tags:
- GitHub
- Docker Image
categories:
---

Docker为我们提供了一个一键式部署代码环境的方式，透过Docker Image我们可以不关心当前的操作系统的环境，只需要拉取镜像下来即可获得一致的运行环境。

但仍存在一个问题，如果你直接使用 `docker build` ，构建出来的镜像是基于你当前机器的CPU架构。当然，你也可以将代码和 `Dockerfile` 拉到虚拟机中进行构建，但这会花费很多时间。

通过 GitHub Actions，我们可以在 `push` 代码的时候，自动进行多架构的镜像构建，不再需要手动构建并推送到 Docker Hub 等镜像仓库中。

下面是一个示例：

这是我在 [kea-dhcp4](https://github.com/WingLim/kea-dhcp4) 中使用的 GitHub Actions 构建脚本的一部分

要推送到 DockerHub，则需要在 repo 的 sercrets 中设置 DockerHub 的用户名及 TOKEN

TOKEN 的生成参考：[Managing access tokens](https://docs.docker.com/docker-hub/access-tokens/)

要推送到 GitHub 的镜像仓库，即 ghcr.io 中则需要设置 PAT(Personal Access Token)

CR_PAT 的生成参考：[Migrating to GitHub Container Registry for Docker images](https://docs.github.com/en/packages/guides/migrating-to-github-container-registry-for-docker-images)

```yaml
name: build

on:
  push:
    branches: [ main ]
    paths-ignore:
      - "README.md"

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      -
        name: Checkout
        uses: actions/checkout@v2
      -
        name: Set up QEMU
        uses: docker/setup-qemu-action@v1
      -
        name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1
      -
        name: Login to DockerHub
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      -
        name: Login to GitHub Container Registry
        uses: docker/login-action@v1 
        # 在 sercrets 中设置 CR_PAT
        # 
        with:
          registry: ghcr.io
          username: ${{ github.repository_owner }}
          password: ${{ secrets.CR_PAT }}
      -
        name: Build kea-dhcp4
        uses: docker/build-push-action@v2
        with:
          context: .
          file: ./Dockerfile
          platforms: linux/amd64,linux/arm64,linux/arm
          push: true
          tags: |
            winglim/kea-dhcp4:latest
            ghcr.io/winglim/kea-dhcp4:latest
```

可以支持的`platforms`如下

- linux/amd64
- linux/arm64
- linux/riscv64
- linux/ppc64le
- linux/s390x
- linux/386
- linux/arm/v7
- linux/arm/v6

需要注意的是：在进行跨平台构建时，如果使用了基础镜像，需要确保对应的基础镜像也有对应的架构，否则会构建失败。

