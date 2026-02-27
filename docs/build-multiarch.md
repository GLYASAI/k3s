# K3s 多架构镜像本地构建方案

在单台 amd64 机器上，通过交叉编译构建 `linux/amd64` + `linux/arm64` 两架构镜像并合并为 manifest list。

## 前提条件

- Linux amd64 宿主机（也可以是 macOS 上的 Linux VM）
- Docker 已安装并启动
- 代理配置（如需访问 GitHub 等外网资源）
- QEMU binfmt_misc 已注册（构建 arm64 镜像时必须，见下方说明）

---

## 一、代理配置

### 1. Docker 构建阶段代理（`docker build` 时使用）

编辑 `~/.docker/config.json`，Docker 会自动将代理作为 build-arg 注入所有镜像构建：

```json
{
  "proxies": {
    "default": {
      "httpProxy": "http://172.16.1.135:3128",
      "httpsProxy": "http://172.16.1.135:3128",
      "noProxy": "localhost,127.0.0.1"
    }
  }
}
```

### 2. 运行时代理（Dapper 容器内 `scripts/download`、`scripts/build` 使用）

在宿主机 export，Dapper 会通过 `DAPPER_ENV` 自动透传进构建容器：

```bash
export http_proxy=http://172.16.1.135:3128
export https_proxy=http://172.16.1.135:3128
export no_proxy=localhost,127.0.0.1
```

---

## 二、对源码的修改

### 修改 1：`Dockerfile.dapper`

**Alpine 3.22 已移除 `aarch64-linux-musl-cross` 包**，改为从 musl.cc 下载。同时新增 aarch64 所需的 zlib 和 libseccomp 交叉编译步骤，新增 `GOPROXY` 环境变量，扩展 `DAPPER_ENV` 以透传 `ARCH`、代理变量和 Go 相关变量。

关键变更：

```dockerfile
# apk 新增 gperf（libseccomp ./configure 依赖）
RUN apk -U --no-cache add ... gperf

# amd64 宿主机：从 musl.cc 下载 aarch64 交叉编译工具链
# （Alpine 3.22 已移除 aarch64-linux-musl-cross 包）
RUN if [ "$(go env GOARCH)" = "amd64" ]; then \
    apk -U --no-cache add mingw-w64-gcc && \
    curl -fsSL https://musl.cc/aarch64-linux-musl-cross.tgz | tar -xz -C /usr/local && \
    ln -sf /usr/local/aarch64-linux-musl-cross/bin/aarch64-linux-musl-gcc /usr/local/bin/aarch64-linux-musl-gcc && \
    ...; \
    fi

# 交叉编译 zlib 1.3.2，安装到 aarch64 sysroot
# （musl.cc 工具链只包含 musl libc，-lz 需单独构建）
ENV ZLIB_VERSION=1.3.2
RUN if [ "$(go env GOARCH)" = "amd64" ]; then \
    SYSROOT=/usr/local/aarch64-linux-musl-cross/aarch64-linux-musl && \
    curl -fsSL https://zlib.net/zlib-${ZLIB_VERSION}.tar.gz | tar -xz -C /tmp && \
    cd /tmp/zlib-${ZLIB_VERSION} && \
    CC=aarch64-linux-musl-gcc CFLAGS="-fPIC" \
    ./configure --prefix=${SYSROOT} --static && \
    make -j$(nproc) && make install; \
    fi

# 交叉编译 libseccomp 2.5.5，安装到 aarch64 sysroot
# （runc 使用 seccomp 构建 tag，需要头文件和静态库）
ENV LIBSECCOMP_VERSION=2.5.5
RUN if [ "$(go env GOARCH)" = "amd64" ]; then \
    SYSROOT=/usr/local/aarch64-linux-musl-cross/aarch64-linux-musl && \
    curl -fsSL https://github.com/seccomp/libseccomp/releases/download/v${LIBSECCOMP_VERSION}/libseccomp-${LIBSECCOMP_VERSION}.tar.gz | tar -xz -C /tmp && \
    cd /tmp/libseccomp-${LIBSECCOMP_VERSION} && \
    ./configure --host=aarch64-linux-musl --prefix=${SYSROOT} \
        --disable-shared --enable-static --enable-python=no \
        CC=aarch64-linux-musl-gcc && \
    make -j$(nproc) && make install; \
    fi

# 为容器内所有 go 命令设置 GOPROXY（proxy.golang.org 在 CN 网络不可达）
ENV GOPROXY=https://goproxy.cn,direct

# DAPPER_ENV：新增 ARCH、代理变量、Go 相关变量
DAPPER_ENV="... ARCH DEBUG http_proxy https_proxy no_proxy HTTP_PROXY HTTPS_PROXY NO_PROXY GOPROXY GONOSUMCHECK GONOSUMDB GONOPROXY"
```

### 修改 2：`scripts/build`

为 amd64 宿主机交叉编译 arm64 时，自动设置 CGO 交叉编译器及 sysroot 路径：

```bash
if [ ${ARCH} = arm64 ] && [ "$(uname -m)" = "x86_64" ]; then
    export GOARCH="arm64"
    export CC="aarch64-linux-musl-gcc"
    export CXX="aarch64-linux-musl-g++"
    # 交叉编译时移除 libsqlite3 tag，使用 go-sqlite3 内置的 amalgamation
    # 避免链接宿主机 x86_64 的系统 sqlite3 头文件
    TAGS="${TAGS/libsqlite3 /}"
    # 指向 Dockerfile.dapper 中交叉编译安装的 zlib 和 libseccomp 头文件/静态库
    AARCH64_SYSROOT="/usr/local/aarch64-linux-musl-cross/aarch64-linux-musl"
    export CGO_CFLAGS="-I${AARCH64_SYSROOT}/include"
    export CGO_LDFLAGS="-L${AARCH64_SYSROOT}/lib"
fi
```

### 修改 3：`scripts/version.sh`

在顶部新增 GOPROXY 设置：

```bash
export GOPROXY=${GOPROXY:-https://goproxy.cn,direct}
```

> `http_proxy` 等代理变量通过 `DAPPER_ENV` 从宿主机透传，无需在脚本内单独设置。

### 修改 4：`scripts/image_scan.sh`

`mirror.gcr.io/aquasec/trivy-db` 已下线，改用官方位置：

```bash
trivy --quiet image --severity ${SEVERITIES} \
    --db-repository ghcr.io/aquasecurity/trivy-db \
    ...
```

---

## 三、构建步骤

### 第一步：安装 Dapper

```bash
curl -sL https://releases.rancher.com/dapper/v0.6.0/dapper-Linux-x86_64 -o /usr/local/bin/dapper
chmod +x /usr/local/bin/dapper
dapper -v   # 验证安装
```

### 第二步：构建 amd64 镜像

```bash
# 清理旧产物，确保干净环境
rm -f .dapper  # 强制重建 Dapper 镜像（因为 Dockerfile.dapper 已修改）
rm -rf bin build/data build/src dist

# 下载 amd64 外部依赖（runc、containerd、k3s-root 等）
mkdir -p build/data
ARCH=amd64 make download

# 编译 + 打包镜像（跳过 lint 和 airgap）
ARCH=amd64 REPO=registry.sensetime.com/diamond SKIP_VALIDATE=true SKIP_AIRGAP=true make

# 此时产出镜像 tag 为：registry.sensetime.com/diamond/k3s:<version>-amd64
# 推送到镜像仓库
docker push registry.sensetime.com/diamond/k3s:<version>-amd64
```

> `<version>` 由 `scripts/version.sh` 自动从 git tag 或 commit 生成，例如 `v1.32.2-k3s1`。
> 可用 `ARCH=amd64 . ./scripts/version.sh && echo $VERSION_TAG$SUFFIX` 预先查看。

### 第三步：注册 QEMU binfmt_misc

`package/Dockerfile` 的最终打包阶段需要在容器内执行 arm64 的 `/bin/sh`（`FROM scratch` + `RUN` 指令）。amd64 宿主机无法直接运行 arm64 二进制，需要先注册 QEMU 用户态模拟器：

```bash
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
```

> **注意**：此命令效果在宿主机重启后失效，重启后需重新执行。
> 可加入 `/etc/rc.local` 或 systemd service 实现开机自动注册。

验证注册成功：

```bash
ls /proc/sys/fs/binfmt_misc/qemu-aarch64
# 输出文件存在即表示注册成功
```

### 第四步：构建 arm64 镜像（交叉编译）

```bash
# 清理 amd64 产物，两次构建的 build/data 不能混用
rm -rf bin build/data build/src dist

# 下载 arm64 外部依赖（k3s-root-arm64.tar 等）
mkdir -p build/data
ARCH=arm64 make download

# 交叉编译 + 打包镜像
ARCH=arm64 REPO=registry.sensetime.com/diamond SKIP_VALIDATE=true SKIP_AIRGAP=true make

# 产出镜像 tag：registry.sensetime.com/diamond/k3s:<version>-arm64
docker push registry.sensetime.com/diamond/k3s:<version>-arm64
```

### 第五步：合并为多架构 manifest

```bash
export TAG=<version>   # 例如 v1.32.2-k3s1
export REPO=registry.sensetime.com/diamond

docker manifest create ${REPO}/k3s:${TAG} \
  --amend ${REPO}/k3s:${TAG}-amd64 \
  --amend ${REPO}/k3s:${TAG}-arm64

docker manifest annotate ${REPO}/k3s:${TAG} \
  ${REPO}/k3s:${TAG}-amd64 --arch amd64 --os linux

docker manifest annotate ${REPO}/k3s:${TAG} \
  ${REPO}/k3s:${TAG}-arm64 --arch arm64 --os linux

docker manifest push ${REPO}/k3s:${TAG}

# 验证
docker manifest inspect ${REPO}/k3s:${TAG}
```

---

## 四、构建流程概览

```
make download (ARCH=amd64)
  └── scripts/download
        ├── git clone runc / containerd
        ├── curl k3s-root-amd64.tar
        └── curl helm charts

make (ARCH=amd64)
  └── Dapper 容器（Dockerfile.dapper）
        └── scripts/ci
              ├── scripts/download   （已完成，跳过）
              ├── scripts/validate   （SKIP_VALIDATE=true 跳过）
              ├── scripts/build      → go build + CGO → bin/k3s
              └── scripts/package
                    ├── scripts/package-cli    → dist/artifacts/k3s
                    └── scripts/package-image  → docker build package/Dockerfile
                                                  → myrepo/k3s:<ver>-amd64

重复上述步骤（ARCH=arm64，CC=aarch64-linux-musl-gcc）

docker manifest create / push
  → myrepo/k3s:<ver>  （包含 amd64 + arm64）
```

---

## 五、常见问题

### arm64 构建报 `exec format error`

`package/Dockerfile` 打包阶段需要在容器内执行 arm64 二进制，未注册 QEMU 时报此错误。执行以下命令注册后重试：

```bash
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
```

### `make download` 下载失败

确认 `http_proxy` / `https_proxy` 已 export，且 Docker config.json 代理已配置。
运行 `rm -f .dapper` 强制重建 Dapper 镜像后重试。

### arm64 构建报 `cannot find -lseccomp`

**此问题已在 `Dockerfile.dapper` 中解决**：构建镜像时会自动交叉编译 libseccomp 2.5.5 并安装到 aarch64 sysroot（`/usr/local/aarch64-linux-musl-cross/aarch64-linux-musl/`），`scripts/build` 通过 `CGO_LDFLAGS` 指向该路径。

若仍报此错误，原因通常是 Dapper 镜像未重新构建。运行以下命令强制重建后重试：

```bash
rm -f .dapper
ARCH=arm64 REPO=<your-repo> SKIP_VALIDATE=true SKIP_AIRGAP=true make
```

### 版本号查看

```bash
# 在 Dapper 容器外查看（需要 git tag）
NO_DAPPER=1 . ./scripts/version.sh && echo "VERSION=$VERSION  TAG=$VERSION_TAG"
```
