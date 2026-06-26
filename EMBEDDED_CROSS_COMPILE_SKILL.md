# 嵌入式跨平台交叉编译 Skill（环境解耦 · 二进制对齐）

> 适用于 RK3568 等 **Embedded Linux 目标机** 的用户态 C/C++ 开发。  
> 典型环境：**双网卡宿主机** — 一卡联网装工具链，一卡直连开发板做 rsync/scp。

---

## 1. 何时启用

| 场景 | 是否走本 Skill |
|------|----------------|
| RK3568 / ARM 板子用户态程序 | ✅ |
| 交叉编译 + Docker 构建环境 | ✅ |
| 双网卡 + 直连开发板 + rsync 同步 | ✅ |
| 依赖 glibc 的动态链接可执行文件 | ✅ |
| 裸机 / FreeRTOS / 内核驱动 | ❌ |
| 可完全静态链接的小工具（musl） | ⚠️ 见 §9 |

用户触发词：`交叉编译`、`Docker 环境`、`RK3568 构建`、`部署到板子`、`ldd not found`、`sync_sysroot`。

---

## 2. 核心原则

**环境解耦与二进制对齐** — 编译器来自宿主机 apt，运行时库来自目标机 rsync，二者不得混用。

| 原则 | 说明 |
|------|------|
| 工具链与运行库分离 | 宿主机 apt **仅**安装 `gcc-aarch64-linux-gnu` 等 **x86→ARM 交叉编译器**；ARM64 运行库 **禁止**通过宿主机/Docker apt 拉取 |
| 二进制同源 | Sysroot 必须来自**实际运行目标板**，保证 glibc、libstdc++、厂商 .so 与板上完全一致 |
| 分层构建 | 联网阶段装工具链 → 离线阶段 rsync sysroot → 镜像 COPY 植入 |
| 闭环验证 | 产物上板 `ldd` 无 `not found` 后方可宣称环境就绪 |

---

## 3. 网络架构（双网卡 + 直连）

```text
┌─────────────────────────────────────────────────────────────┐
│  宿主机 / 虚拟机                                              │
│  ┌──────────────┐          ┌──────────────┐                 │
│  │ eth0 / 外网   │  联网   │ eth1 / 板卡网 │  直连           │
│  │ (apt 工具链)  │ ──────► │ (rsync/scp)  │ ──────► 开发板   │
│  └──────────────┘          └──────────────┘                 │
└─────────────────────────────────────────────────────────────┘
```

| 网卡 | 用途 | 允许操作 |
|------|------|----------|
| 外网网卡 | 第一阶段联网 | `apt` 安装 **宿主机架构** 的交叉工具链、`cmake`、`rsync` |
| 板卡直连网卡 | 第二、五阶段 | `rsync`、`scp`、`ssh` 与开发板通信 |

**硬性规则**：

- **禁止**在宿主机或 Dockerfile 中通过 `apt install libxxx:arm64`、`qemu-user-static` 等方式拉取 ARM64 运行库 — 实测常因镜像源、架构混装、glibc 版本漂移导致交叉编译失败或板上 `ldd` 报错。
- 板卡 IP 走直连网段（如 `192.168.1.x`），rsync/scp 命令必须显式指定 `-e "ssh -b <板卡网卡IP>"` 或路由到直连接口，避免流量误走外网网卡。

---

## 4. 强制 Pipeline（三阶段 + 验证）

```text
┌──────────────┐    ┌──────────────────┐    ┌──────────────┐    ┌──────────────┐
│ 0. Grill     │───►│ 1. 联网装工具链   │───►│ 2. rsync     │───►│ 3. Docker    │
│ 需求共识      │    │ (apt + 国内源)   │    │ sync_sysroot │    │ COPY sysroot │
└──────────────┘    └──────────────────┘    └──────────────┘    └──────────────┘
                                                                        │
                    ┌───────────────────────────────────────────────────┘
                    ▼
              ┌──────────────┐    ┌──────────────┐
              │ 4. CMake 构建 │───►│ 5. 上板 ldd  │
              │ FIND_ROOT    │    │ 闭环验证      │
              └──────────────┘    └──────────────┘
```

### 阶段 0 — Grill（需求共识）

写 Dockerfile 或 `sync_sysroot.sh` 之前，必须确认：

- [ ] 目标板型号与系统镜像（如 RK3568 + Debian 11 / Ubuntu 20.04）
- [ ] 目标架构（`aarch64` / `armhf`）
- [ ] 板卡直连 IP 与 SSH 用户
- [ ] 宿主机板卡网卡名或绑定 IP
- [ ] 链接方式（动态 glibc / 静态 / 部分静态）
- [ ] 是否依赖厂商专有库（`librknn`、`librga`、`libmali` 等 — 随 sysroot 一并同步）
- [ ] 部署路径与运行用户

未共识时：**拒绝生成 Dockerfile**，引导用户完成 Grill 或回答上述清单。

---

### 阶段 1 — 联网模式：仅安装交叉工具链

**仅在虚拟机/宿主机可联网时执行。** 只装 **宿主机架构** 的编译工具，不装任何 `:arm64` 包。

#### 1.1 注入国内 apt 镜像源（必须在 apt 之前）

```bash
# 示例：Ubuntu 20.04 替换为阿里云源（按实际发行版调整 sed 模式）
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
sudo sed -i 's|http://archive.ubuntu.com|http://mirrors.aliyun.com|g' /etc/apt/sources.list
sudo sed -i 's|http://security.ubuntu.com|http://mirrors.aliyun.com|g' /etc/apt/sources.list
sudo apt-get update
```

Debian 用户替换 `/etc/apt/sources.list` 中 `deb.debian.org` 为国内镜像（清华、阿里云、中科大等）。**Agent 必须根据用户发行版给出对应 sed 命令，不得跳过此步。**

#### 1.2 安装工具链（禁止 ARM64 运行库）

```bash
sudo apt-get install -y --no-install-recommends \
    build-essential \
    cmake \
    rsync \
    gcc-aarch64-linux-gnu \
    g++-aarch64-linux-gnu \
    binutils-aarch64-linux-gnu \
    pkg-config
```

**禁止安装**：`libstdc++6:arm64`、`libc6:arm64`、`libxxx-dev:arm64` 及任何带 `:arm64` 后缀的运行时/开发包。

---

### 阶段 2 — 离线同步模式：`scripts/sync_sysroot.sh`

通过 rsync 将开发板运行时库同步至宿主机 `./sysroot`，作为 Docker 构建与 CMake 的**唯一**库来源。

#### 2.1 Sysroot 封装规则

| 允许同步 | 禁止同步 |
|----------|----------|
| `/lib/`、`/lib/aarch64-linux-gnu/` 下的 `.so*`、动态链接器、符号链接 | 全量 rootfs（`/`、`/bin`、`/etc`、`/var` 等） |
| `/usr/lib/`、`/usr/lib/aarch64-linux-gnu/` 下的 `.so*`、`.a`（按需）、符号链接 | 可执行文件、配置文件、内核模块 |
| `/usr/include/` 下项目所需头文件（按需，可 `--include` 过滤） | `/usr/share/doc`、locale、man |

#### 2.2 脚本模板（在目标项目中创建 `scripts/sync_sysroot.sh`）

```bash
#!/usr/bin/env bash
set -euo pipefail

# ── 配置（按 Grill 共识填写）────────────────────────────
BOARD_USER="${BOARD_USER:-root}"
BOARD_IP="${BOARD_IP:-192.168.1.100}"      # 板卡直连 IP
BOARD_SSH_BIND="${BOARD_SSH_BIND:-}"         # 可选：宿主机板卡网卡绑定 IP，如 192.168.1.10
SYSROOT_DIR="${SYSROOT_DIR:-$(cd "$(dirname "$0")/.." && pwd)/sysroot}"
RSYNC_SSH="ssh -o StrictHostKeyChecking=accept-new"
[[ -n "$BOARD_SSH_BIND" ]] && RSYNC_SSH="$RSYNC_SSH -b $BOARD_SSH_BIND"

RSYNC=(rsync -avz --copy-links --delete \
    --include='*/' \
    --include='*.so' --include='*.so.*' \
    --include='ld-linux-*' --include='ld-*.so.*' \
    --exclude='*')

REMOTE="${BOARD_USER}@${BOARD_IP}"

sync_path() {
    local remote_path="$1"
    local local_path="$2"
    mkdir -p "$local_path"
    "${RSYNC[@]}" -e "$RSYNC_SSH" "${REMOTE}:${remote_path}/" "${local_path}/"
    echo "✓ synced ${remote_path} → ${local_path}"
}

echo "=== Sync sysroot from ${REMOTE} ==="
sync_path /lib/aarch64-linux-gnu  "${SYSROOT_DIR}/lib/aarch64-linux-gnu"
sync_path /lib                      "${SYSROOT_DIR}/lib"
sync_path /usr/lib/aarch64-linux-gnu  "${SYSROOT_DIR}/usr/lib/aarch64-linux-gnu"
sync_path /usr/lib                    "${SYSROOT_DIR}/usr/lib"

# 按需追加头文件（示例）
# sync_path /usr/include/rknn       "${SYSROOT_DIR}/usr/include/rknn"

echo "=== Sysroot ready: ${SYSROOT_DIR} ==="
find "${SYSROOT_DIR}" -name '*.so*' | wc -l | xargs -I{} echo "  shared objects: {}"
```

#### 2.3 单库补齐（ldd 报错后的唯一修复路径）

`ldd` 出现 `not found` 时，**不得**在 Dockerfile 中 `apt install`，而应：

1. 在板上定位库：`find /usr /lib -name 'libXXX.so*' 2>/dev/null`
2. 追加精准 rsync（示例）：

```bash
rsync -avz -e "ssh -b <板卡网卡IP>" \
    root@<board-ip>:/usr/lib/aarch64-linux-gnu/libmissing.so* \
    ./sysroot/usr/lib/aarch64-linux-gnu/
```

3. 记录到 `docs/target-sysroot-manifest.md`（库名、板上路径、同步日期）
4. 重新构建 Docker 镜像并再上板 `ldd`

---

### 阶段 3 — 镜像集成：Dockerfile

**严禁**在 Dockerfile 中 `apt install` ARM64 运行库。Sysroot 通过 `COPY` 植入。

```dockerfile
# syntax=docker/dockerfile:1
# Sysroot 来源：scripts/sync_sysroot.sh @ <日期>
# 板型：RK3568 / <发行版>
FROM ubuntu:20.04 AS builder

ENV DEBIAN_FRONTEND=noninteractive

# 仅安装宿主机架构交叉工具链 — 禁止 :arm64 包
RUN sed -i 's|http://archive.ubuntu.com|http://mirrors.aliyun.com|g' /etc/apt/sources.list && \
    sed -i 's|http://security.ubuntu.com|http://mirrors.aliyun.com|g' /etc/apt/sources.list && \
    apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    cmake \
    gcc-aarch64-linux-gnu \
    g++-aarch64-linux-gnu \
    binutils-aarch64-linux-gnu \
    pkg-config \
    && rm -rf /var/lib/apt/lists/*

# 植入板端同步的 sysroot — 唯一运行时库来源
COPY ./sysroot /usr/aarch64-linux-gnu

ENV CMAKE_SYSROOT=/usr/aarch64-linux-gnu

WORKDIR /src
COPY . .

RUN cmake -S . -B build \
    -DCMAKE_TOOLCHAIN_FILE=cmake/toolchain-aarch64.cmake \
    -DCMAKE_FIND_ROOT_PATH=/usr/aarch64-linux-gnu \
    -DCMAKE_BUILD_TYPE=Release \
    && cmake --build build
```

#### 3.1 环境变量（Dockerfile 必须显式定义）

```dockerfile
ENV CMAKE_SYSROOT=/usr/aarch64-linux-gnu
```

#### 3.2 CMake toolchain 要点（`cmake/toolchain-aarch64.cmake`）

```cmake
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)

set(CMAKE_C_COMPILER aarch64-linux-gnu-gcc)
set(CMAKE_CXX_COMPILER aarch64-linux-gnu-g++)

set(CMAKE_SYSROOT /usr/aarch64-linux-gnu)
set(CMAKE_FIND_ROOT_PATH /usr/aarch64-linux-gnu)

set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)
```

**构建时必须显式传入**：

```bash
cmake -S . -B build \
  -DCMAKE_TOOLCHAIN_FILE=cmake/toolchain-aarch64.cmake \
  -DCMAKE_FIND_ROOT_PATH=/usr/aarch64-linux-gnu
```

`CMAKE_FIND_ROOT_PATH` 确保编译器与链接器**只**在 sysroot 内查找库与头文件，屏蔽宿主机 `/usr/lib/x86_64-linux-gnu` 干扰。

---

### 阶段 4 — 构建

```bash
# 宿主机直接构建（sysroot 已同步到 ./sysroot）
cmake -S . -B build-arm \
  -DCMAKE_TOOLCHAIN_FILE=cmake/toolchain-aarch64.cmake \
  -DCMAKE_SYSROOT="${PWD}/sysroot" \
  -DCMAKE_FIND_ROOT_PATH="${PWD}/sysroot" \
  -DCMAKE_BUILD_TYPE=Release
cmake --build build-arm

# 或 Docker 构建（推荐可复现）
docker build -t project-arm-builder .
docker create --name arm-build project-arm-builder
docker cp arm-build:/src/build/<your_executable> ./build-arm/
docker rm arm-build
```

---

### 阶段 5 — 闭环验证

```bash
# 1. 传输至目标机（走板卡直连网卡）
scp -o BindAddress=<板卡网卡IP> build-arm/<your_executable> root@<board-ip>:/tmp/

# 2. 板上 ldd 校验 — 必须无 not found
ssh -b <板卡网卡IP> root@<board-ip> "ldd /tmp/<your_executable>"

# 3. （可选）实际运行
ssh -b <板卡网卡IP> root@<board-ip> "/tmp/<your_executable> --help"
```

**Agent 完成条件**：用户粘贴 `ldd` 输出且无 `not found`，方可宣称环境就绪。

若仍有 `not found` → 回到 **§2.3 单库补齐**，禁止尝试 Dockerfile apt 修复。

---

## 5. Agent 硬性规则

| # | 规则 | 行为 |
|---|------|------|
| 1 | 工具链/运行库分离 | apt 只装交叉编译器；ARM64 库只来自 rsync sysroot |
| 2 | 禁止 Dockerfile apt 装 ARM64 库 | 不得 `apt install lib*:arm64` 或同架构运行库替代 sysroot |
| 3 | 国内源前置 | 任何 apt 操作前必须先 sed 替换国内镜像源 |
| 4 | Sysroot 精准同步 | 禁止全量 rootfs；只同步 `.so*`、链接器、符号链接、按需头文件 |
| 5 | 环境变量显式 | Dockerfile 必须 `ENV CMAKE_SYSROOT=...`；CMake 必须 `-DCMAKE_FIND_ROOT_PATH=...` |
| 6 | ldd 驱动补齐 | `not found` → `sync_sysroot.sh` 单库 rsync，记录 manifest |
| 7 | 先共识后生成 | 无板卡 IP/架构/网卡信息 → 拒绝 Dockerfile，引导 Grill |
| 8 | 与 TDD 衔接 | 环境走本 Skill；业务代码走 TDD |

---

## 6. 环境指纹（验证用，非 Dockerfile 前置阻塞）

Sysroot 来自板端后，Dockerfile 可生成；但仍建议采集指纹用于 **glibc 版本存档** 与 **ldd 对比**：

```bash
uname -a
cat /etc/os-release
ldd --version | head -1
file /tmp/<your_executable>
readelf -l /tmp/<your_executable> | grep -i interpreter
ldd /tmp/<your_executable>
```

建议在 `docs/target-fingerprint.md` 存档；`docs/target-sysroot-manifest.md` 记录每次 rsync 与单库补齐。

---

## 7. 常见问题诊断

| 症状 | 可能原因 | 修复 |
|------|----------|------|
| Docker 构建找错库路径 | 未设 `CMAKE_FIND_ROOT_PATH` | 显式 `-DCMAKE_FIND_ROOT_PATH=/usr/aarch64-linux-gnu` |
| 链接阶段引用 x86_64 库 | `FIND_ROOT_PATH_MODE` 未设 ONLY | 检查 toolchain 四项 MODE |
| `GLIBC_x.xx not found` | sysroot 与板端不一致 | 重新 `sync_sysroot.sh` 全量同步 lib 目录 |
| `libstdc++.so.6: GLIBCXX_x.xx not found` | sysroot 中 libstdc++ 过旧 | 从板上 re-sync `/usr/lib/aarch64-linux-gnu/libstdc++*` |
| `not found` 厂商库 | sysroot 未含 vendor .so | §2.3 单库 rsync |
| apt 装 `:arm64` 后仍 ldd 失败 | 库版本/路径与板端漂移 | **放弃 apt 方案**，改 rsync |
| rsync 连不上板子 | 流量走外网网卡 | `ssh -b <板卡网卡IP>` / 检查直连路由 |
| 容器内编译通过、板上 segfault | ABI/`-march` 不一致 | 对比 `file`、`readelf -A` |

---

## 8. RK3568 补充

- **NPU/RGA/Mali 类库**：不在标准 apt 中，随 sysroot 从板端同步或 §2.3 单库补齐
- **I2C/GPIO 用户态**：`libi2c.so`、`libgpiod.so` 等在 `/usr/lib/aarch64-linux-gnu/`，随 sysroot 同步即可
- **板厂 prebuilt toolchain**：若与板上 glibc 一致且已含匹配 sysroot，可替代自建，但仍须 `ldd` 验证
- **交叉编译主机测试**：业务逻辑用 Fake 硬件跑 `ctest`；集成测试须上板

---

## 9. 简化路径：静态链接（可选）

无厂商动态库依赖时可 Grill 评估 **musl 静态链接**：

- 优点：不依赖 sysroot/glibc 匹配
- 缺点：体积大；厂商 .so 无法静态
- 仅在用户明确同意且 `ldd` 证明无动态依赖时适用

---

## 10. 文件清单

| 文件 | 用途 |
|------|------|
| `EMBEDDED_CROSS_COMPILE_SKILL.md` | 本 Skill |
| `PROMPT_CROSS_COMPILE.md` | `/cross-env` 提示词模板 |
| `scripts/sync_sysroot.sh` | 板端 → 宿主机 sysroot 同步（目标项目中创建） |
| `sysroot/` | rsync 产物（gitignore，不入库） |
| `docs/target-fingerprint.md` | 环境指纹存档 |
| `docs/target-sysroot-manifest.md` | sysroot 同步与单库补齐记录 |
| `cmake/toolchain-aarch64.cmake` | 交叉编译 toolchain |
| `Dockerfile` | 共识 + sysroot 就绪后生成 |

---

## 11. 与 C++ 工程工作流的关系

```text
嵌入式新功能：
  Grill（含网卡/板卡 IP 共识）
    → [若需新构建环境] 本 Skill 阶段 1–5
    → TDD 垂直切片（主机 Fake 测试 + 板上集成验证）
    → scp 部署 + 板上 ldd/运行确认
```
