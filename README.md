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
LOCAL_ARCH=aarch64 IMG=clarechu/build-tools:release-1.7 make build
```

现在在项目根目录上会出现 out 目录 及下面所有的包

```bash
$ ls
client       istio-cni         istio-iptables  mixc    operator         policybackend  server
envoy        istio-cni-repair  istio_is_init   mixgen  pilot-agent      release
install-cni  istioctl          logs            mixs    pilot-discovery  sdsclient
```

更改arch manifests/charts/gateways/istio-ingress/values.yaml

```yaml
  arch:
    amd64: 2
    s390x: 2
    ppc64le: 2
    # 加这一行代码
    + arm64: 2
```

```bash
$ HUB=docker.io/clarechu TAG=1.7 IMG=clarechu/build-tools:release-1.7  make gen-charts
$ HUB=docker.io/clarechu TAG=1.7 IMG=clarechu/build-tools:release-1.7 make docker.operator
```

出现了几个问题 需要改以下代码 `Makefile.core.mk`

```bash
# 306 build-linux: depend
# 307         STATIC=0 GOOS=linux GOARCH=$(GOARCH_LOCAL) LDFLAGS='-extldflags -static -s -w' common/scripts/gobuild.sh $(ISTIO_OUT)/ $(STANDARD_BINARIES)
# 308         GOOS=linux GOARCH=$(GOARCH_LOCAL) LDFLAGS=$(RELEASE_LDFLAGS) common/scripts/gobuild.sh $(ISTIO_OUT_LINUX)/ -tags=agent $(AGENT_BINARIES)

# istio 把这个地方写死了
 GOARCH=amd64 -->  GOARCH=$(GOARCH_LOCAL)
# 目录改调
$(ISTIO_OUT_LINUX) --> $(ISTIO_OUT)

export ISTIO_OUT_LINUX:=$(TARGET_OUT_LINUX)
--->
export ISTIO_OUT_LINUX:=$(TARGET_OUT)
```

tools/istio-docker.mk

```bash
docker.operator: $(ISTIO_OUT_LINUX)/operator --> docker.operator: $(ISTIO_OUT)/operator
```

### 编译 istiod

```bash
$  LOCAL_ARCH=aarch64 IMG=clarechu/build-tools:release-1.7 HUB=docker.io/clarechu TAG=1.7 make docker.pilot
```

### 编译 proxyv2

```bash
$ LOCAL_ARCH=aarch64 IMG=clarechu/build-tools:release-1.7 HUB=docker.io/clarechu TAG=1.7 make docker.proxyv2
```

出现了这个问题

```bash

standard_init_linux.go:211: exec user process caused "exec format error"
The command '/bin/sh -c chown -R istio-proxy /var/lib/istio' returned a non-zero code: 1

real	0m6.493s
user	0m0.660s
sys	0m1.529s
tools/istio-docker.mk:98: recipe for target 'docker.proxyv2' failed

```

所以注释 掉

```bash
RUN chown -R istio-proxy /var/lib/istio
COPY --from=default /usr/lib/x86_64-linux-gnu/xtables/ /usr/lib/x86_64-linux-gnu/xtables
COPY --from=default /usr/lib/x86_64-linux-gnu/ /usr/lib/x86_64-linux-gnu
```

然后重新在arm的机器上面做镜像

```dockerfile
cat >> Dockerfile << EOF
FROM clarechu/proxyv2:1.7

RUN chown -R istio-proxy /var/lib/istio
EOF

docker build -t clarechu/proxyv2:1.7 .

docker push clarechu/proxyv2:1.7
```
