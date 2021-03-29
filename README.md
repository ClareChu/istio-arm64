# istio-arm64

这个教程主要 学习如何编译istio arm64的版本

## 首先我们先编译istio operator

在编译istio-operator依赖base和distroless这两个镜像 所以在这个之前我们编译出这两个镜像

### 编译 istio base基础镜像

因为ubuntu ubuntu:bionic 是可以支持arm64 和amd的 所以这个base 是可以不需要变更的

```bash

$ docker build -t clarechu/istio-base:release-1.7 . -f Dockerfile.base

$ docker push clarechu/istio-base:release-1.7

```

### 编译 istio distroless

因为 distroless 编译的时候需要 static-debian 所以我tag了一下到自己的仓库 所以需要更改一个Dockerfile `Dockerfile.distroless`

```bash
gcr.io/distroless/static-debian10@sha256:34cb7dca899f9f40f576509977cd43b093cf13eeec2ce6333f9b9b2db331f937 ----> clarechu/static-debian

# docker build distroless
$  docker build -t clarechu/istio-distroless:release-1.7 . -f Dockerfile.distroless
# docker push
$ docker push clarechu/istio-distroless:release-1.7

```

## build 二进制文件

编译所有的包 在istio中所有的包都是在镜像中编译完成的 但是我在找包的时候并没有看到有arm64的镜像，所以我准备使用amd64的服务器去编译arm64的包。


不过需要改以下几个地方

1. 改变uname

在common/scripts/setup_env.sh

```bash
$ LOCAL_ARCH=$(uname -m)
改成
$ LOCAL_ARCH=aarch64

```

2. 更改编译代理

在common/scripts/run.sh

```bash
$     -e IN_BUILD_CONTAINER=1 \
#下面加上
  -e GOPROXY=https://goproxy.io \
```

3. 在项目的根目录 编译

```bash
IMG=clarechu/build-tools:release-1.7 make build
```
