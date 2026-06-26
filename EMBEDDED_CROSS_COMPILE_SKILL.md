# 嵌入式跨平台交叉编译 Skill（观测-验证-构建）

> 适用于 RK3568 等 **Embedded Linux 目标机** 的用户态 C/C++ 开发。  
> 本 Skill 是静态指令，不含可执行脚本；Dockerfile 仅在目标机环境指纹确认后生成。

---

## 1. 何时启用

| 场景 | 是否走本 Skill |
|------|----------------|
| RK3568 / ARM 板子用户态程序 | ✅ |
| 交叉编译 + Docker 构建环境 | ✅ |
| 依赖 glibc 的动态链接可执行文件 | ✅ |
| 裸机 / FreeRTOS / 内核驱动 | ❌（无 Docker/glibc 流程） |
| 可完全静态链接的小工具（musl） | ⚠️ 简化流程，见 §8 |

用户触发词：`交叉编译`、`Docker 环境`、`RK3568 构建`、`部署到板子`、`ldd not found`。

---

## 2. 核心原则

1. **观测先于构建**：Dockerfile 必须来自目标机实测，禁止凭猜测选 Ubuntu 版本或 apt 包。
2. **需求共识先于 Dockerfile**：未明确目标板型号、部署路径、链接方式前，不写 Dockerfile。
3. **一次验证闭环**：每次环境变更后，必须重新 `scp` → 目标机 `ldd` → 确认无 `not found`。
4. **精准同步**：专有库优先官方 SDK/deb；仅对无法 apt 安装的库做 **rsync 单路径** 同步，禁止全量 rootfs 拷贝。

---

## 3. 强制 Pipeline（观测-验证-构建）

```text
┌─────────────┐    ┌──────────────┐    ┌─────────────┐    ┌──────────────┐
│ 0. Grill    │───►│ 1. 基准编译   │───►│ 2. 环境指纹  │───►│ 3. Dockerfile │
│ 需求共识     │    │ + scp 传输   │    │ 目标机采集   │    │ + 容器内重编  │
└─────────────┘    └──────────────┘    └─────────────┘    └──────────────┘
                                              │                    │
                                              └──── 4. 再验证 ◄────┘
```

### 阶段 0 — Grill（需求共识）

写 Dockerfile 或改 CMake toolchain 之前，必须确认：

- [ ] 目标板型号与系统镜像（如 RK3568 + Debian 11 / Ubuntu 20.04 rootfs）
- [ ] 目标架构（`aarch64` / `armhf`）
- [ ] 链接方式（动态 glibc / 静态 / 部分静态）
- [ ] 部署路径与运行用户（如 `/usr/local/bin`、`root` 或普通用户）
- [ ] 是否依赖厂商专有库（`librknn`、`librga`、`libmali` 等）
- [ ] 主机侧现有工具链（`aarch64-linux-gnu-g++` 版本、是否已有 SDK）

未共识时：**拒绝生成 Dockerfile**，引导用户完成 Grill 或回答上述清单。

---

### 阶段 1 — 基准编译与传输

在**开发机或虚拟机**完成首次交叉编译（允许存在潜在库冲突，目的是获取诊断样本）：

```text
# 配置（示例，按项目 toolchain 文件调整）
cmake -S . -B build-arm \
  -DCMAKE_TOOLCHAIN_FILE=cmake/toolchain-aarch64.cmake \
  -DCMAKE_BUILD_TYPE=Release
cmake --build build-arm

# 传输至目标机（替换 IP、路径、二进制名）
scp build-arm/<your_executable> user@<board-ip>:/tmp/
```

**Agent 指令**：若用户尚未编译出二进制，引导其先完成 CMake 交叉编译配置，不要跳过此步去写 Docker。

---

### 阶段 2 — 环境指纹提取（目标机执行）

用户在目标机上执行以下命令，**完整粘贴输出**给 Agent：

```bash
# 1. 系统与内核
uname -a

# 2. 发行版（用于锁定 Docker FROM 版本）
cat /etc/os-release

# 3. GLIBC 版本
ldd --version | head -1

# 4. 可执行文件架构与动态链接器
file /tmp/<your_executable>
readelf -l /tmp/<your_executable> | grep -i interpreter

# 5. 运行时依赖（核心）
ldd /tmp/<your_executable>

# 6. （可选）已有厂商库路径
ls -la /usr/lib/aarch64-linux-gnu/librk* 2>/dev/null
ls -la /usr/lib/librk* 2>/dev/null
```

**Agent 拒绝条件**：若用户未提供 **§2 中至少 `uname -a`、`/etc/os-release`、`ldd --version`、`ldd ./<executable>` 四项输出**，必须拒绝编写 Dockerfile，并输出：

```markdown
## 缺少环境指纹

请先在目标机执行 EMBEDDED_CROSS_COMPILE_SKILL.md §2 阶段 2 的命令，将输出完整粘贴回来。

在收到指纹前，我不会生成 Dockerfile 或修改交叉编译 sysroot。
```

---

### 阶段 3 — 指纹分析与 Dockerfile 生成

Agent 收到指纹后，按以下顺序分析并产出文档（先分析表，再 Dockerfile）：

#### 3.1 指纹解析表（必须先输出）

| 字段 | 从指纹中提取 | 用途 |
|------|-------------|------|
| `DISTRO` | `/etc/os-release` 的 `ID` + `VERSION_ID` | `FROM ubuntu:20.04` 等 |
| `GLIBC` | `ldd --version` | 容器 glibc 必须 ≤ 目标机（见 §3.3） |
| `ARCH` | `uname -m` / `file` | `aarch64-linux-gnu` toolchain |
| `INTERPRETER` | `readelf` 的 interpreter 路径 | 验证 sysroot 动态链接器 |
| `APT_PACKAGES` | 将 `ldd` 中 `.so` 映射到 apt 包名 | `apt-get install` 列表 |
| `NOT_FOUND` | `ldd` 中 `not found` 行 | 阻塞项，必须解决后才能 Docker 化 |
| `VENDOR_LIBS` | 非标准路径的 `.so` | rsync 或 SDK 安装 |

#### 3.2 `ldd` → `apt` 映射规则

对 `ldd` 输出中每个已解析的 `.so`：

1. 记录完整路径（如 `libpthread.so.0 => /lib/aarch64-linux-gnu/libpthread.so.0`）
2. 在**与目标同版本的 Ubuntu/Debian** 上，用 `dpkg -S <path>` 查所属包（用户可在目标机执行后粘贴）
3. 归纳去重为 `apt-get install` 列表，常见映射：

| ldd 库 | 典型 apt 包 |
|--------|------------|
| `libstdc++.so.6` | `libstdc++6` |
| `libgcc_s.so.1` | `libgcc-s1`（Debian）/ `libgcc1`（旧 Ubuntu） |
| `libpthread.so.0` | `libc6` |
| `libdl.so.2` | `libc6` |
| `libm.so.6` | `libc6` |
| `libi2c.so.0` | `libi2c0` |
| `libgpiod.so.2` | `libgpiod2` |

对 **`not found`** 库：

- 标准库名 → 补充对应 `-dev` 和运行时包，重新编译后再测
- 厂商库（`librknnrt.so`、`librga.so` 等）→ 走 §3.4，**不得**凭空写 `apt install`

#### 3.3 glibc 兼容性规则

交叉编译动态链接程序时：

- 构建环境 glibc 版本 **必须 ≤ 目标机** glibc（旧 glibc 编出的二进制可在新系统跑）
- `FROM` 应锁定指纹中 `VERSION_ID` 对应的官方镜像（如 `ubuntu:20.04`、`debian:11`）
- 禁止 `FROM ubuntu:latest`

#### 3.4 专有库处理（精准同步）

优先级：

1. **厂商 deb/SDK**（Rockchip `rknn-toolkit2`、板厂提供的 `.deb`）
2. **目标机精准 rsync**（仅库文件与头文件，非全系统）：

```text
# 在开发机执行 — 仅同步单个库及其符号链接
rsync -avz user@<board-ip>:/usr/lib/aarch64-linux-gnu/librknnrt.so* ./sysroot/vendor/lib/
rsync -avz user@<board-ip>:/usr/include/rknn/ ./sysroot/vendor/include/rknn/
```

3. 在 `docs/target-sysroot-manifest.md` 记录来源路径、版本、同步日期

**禁止**：`rsync -avz user@board:/ ./sysroot/` 全量同步 rootfs。

---

### 阶段 4 — Dockerfile 结构（推荐多阶段）

在指纹分析表经用户确认后，生成 Dockerfile。推荐结构：

```dockerfile
# syntax=docker/dockerfile:1
# 由环境指纹生成 — 请勿手动改 FROM 版本
# 指纹来源：<日期> / <板子 IP> / <镜像版本>
FROM ubuntu:20.04 AS sysroot-base

ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
    # 以下列表来自 ldd → apt 映射表
    libstdc++6 \
    <其它运行时包> \
    && rm -rf /var/lib/apt/lists/*

# 构建阶段
FROM sysroot-base AS builder

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    cmake \
    gcc-aarch64-linux-gnu \
    g++-aarch64-linux-gnu \
    <:-dev 包> \
    && rm -rf /var/lib/apt/lists/*

# 厂商库（若有）— 从精准 rsync 的 sysroot/vendor 拷贝
COPY sysroot/vendor/ /usr/aarch64-linux-gnu/

WORKDIR /src
COPY . .

# 使用 toolchain 文件，CMAKE_FIND_ROOT_PATH 指向 vendor
RUN cmake -S . -B build \
    -DCMAKE_TOOLCHAIN_FILE=cmake/toolchain-aarch64.cmake \
    -DCMAKE_BUILD_TYPE=Release \
    && cmake --build build

# 可选：输出阶段仅拷贝产物
FROM scratch AS export
COPY --from=builder /src/build/<your_executable> /
```

**CMake toolchain 要点**（`cmake/toolchain-aarch64.cmake`）：

```cmake
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)
set(CMAKE_C_COMPILER aarch64-linux-gnu-gcc)
set(CMAKE_CXX_COMPILER aarch64-linux-gnu-g++)
set(CMAKE_FIND_ROOT_PATH /usr/aarch64-linux-gnu ${CMAKE_CURRENT_LIST_DIR}/../sysroot/vendor)
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
```

---

### 阶段 5 — 闭环再验证

容器内编译产物必须再次上板验证：

```text
1. 从容器拷贝二进制：docker cp <container>:/src/build/<exe> .
2. scp 到目标机
3. 目标机再次执行：ldd ./<exe>
4. 确认：无 not found；必要时实际运行并检查退出码/日志
```

**Agent 完成条件**：用户粘贴「再验证」的 `ldd` 输出且无 `not found`，方可宣称环境就绪。

---

## 4. Agent 引导规则（摘要）

| 规则 | 行为 |
|------|------|
| 禁止凭空生成 | 无指纹 → 拒绝 Dockerfile，输出 §2 阶段 2 采集指令 |
| 先分析后生成 | 必须先输出指纹解析表，用户确认后再给 Dockerfile |
| 差异诊断 | 逐项列出 `not found`，映射到 apt 或 vendor 方案 |
| 精准同步 | 专有库只 rsync 单路径；记录到 manifest |
| 与 Grill 衔接 | 新模块涉及新 `.so` 依赖 → 重新走阶段 1–5 |
| 与 TDD 衔接 | 业务代码仍走 TDD；环境搭建走本 Skill |

---

## 5. 环境指纹存档模板

建议在项目中维护 `docs/target-fingerprint.md`（由 Agent 根据用户粘贴内容填写）：

```markdown
# 目标机环境指纹

- 采集日期：
- 板型：RK3568 / ...
- IP：
- 镜像：

## uname -a
（粘贴）

## /etc/os-release
（粘贴）

## GLIBC
（粘贴）

## ldd --version
（粘贴）

## ldd <executable>（首次）
（粘贴）

## ldd <executable>（Docker 重编后）
（粘贴）

## apt 包映射结论
| .so | apt 包 | 状态 |
|-----|--------|------|

## 厂商库 manifest
| 库名 | 目标机路径 | 同步到 | 来源 |
|------|-----------|--------|------|
```

---

## 6. 常见问题诊断

| 症状 | 可能原因 | Agent 应建议 |
|------|----------|-------------|
| `GLIBC_x.xx not found` | 构建机 glibc 太新 | 降低 `FROM` 版本或在旧容器内编译 |
| `libstdc++.so.6: version GLIBCXX_x.xx not found` | g++ 版本高于目标机 | 容器内安装匹配版本的 `libstdc++6`，或 pin g++ 版本 |
| `No such file or directory`（文件存在） | 动态链接器路径错误 | 检查 `readelf -l` interpreter 与 sysroot |
| `not found` 厂商库 | 未同步 vendor lib | rsync 单库 + `LD_LIBRARY_PATH` 或 rpath |
| 容器内编译通过、板上 segfault | ABI/编译选项不一致 | 对比 `file`、`readelf -A`、`-march` 标志 |

---

## 7. 与 RK3568 相关的补充

- **优先查阅**：板厂/瑞芯微提供的 prebuilt toolchain 或 Docker 镜像（若与指纹一致，可替代自建 Dockerfile，但仍需 `ldd` 验证）
- **NPU/RGA 类库**：几乎从不在标准 apt 中，必须走 vendor SDK 或 rsync
- **I2C/GPIO 用户态**：通常 `libi2c0`、`libgpiod2`；在目标机 `dpkg -S` 确认
- **交叉编译主机测试**：业务逻辑用 Fake 硬件跑 `ctest`；集成测试仍须上板

---

## 8. 简化路径：静态链接（可选）

若程序无厂商动态库依赖，可 Grill 时评估 **musl 静态链接**：

- 优点：不依赖目标 glibc 版本，Docker 更简单
- 缺点：体积大；某些库（glibc 特有、厂商 .so）无法静态
- 流程：仍可先 `ldd` 确认「静态」；`file` 应显示 `statically linked`

仅在用户明确同意且 `ldd` 证明无动态依赖时，可跳过 glibc 匹配，但仍建议保留指纹存档。

---

## 9. 文件清单

| 文件 | 用途 |
|------|------|
| `EMBEDDED_CROSS_COMPILE_SKILL.md` | 本 Skill（完整流程） |
| `PROMPT_CROSS_COMPILE.md` | 用户触发 `/cross-env` 的提示词模板 |
| `docs/target-fingerprint.md` | 项目内指纹存档（运行时创建） |
| `docs/target-sysroot-manifest.md` | 厂商库同步记录（运行时创建） |
| `cmake/toolchain-aarch64.cmake` | 交叉编译 toolchain |
| `Dockerfile` | 指纹确认后生成 |

---

## 10. 与 C++ 工程工作流的关系

```text
嵌入式新功能：
  Grill（含硬件/部署共识）
    → [若需新构建环境] 本 Skill 阶段 1–5
    → TDD 垂直切片（主机 Fake 测试 + 板上集成验证）
    → 部署 scp + 板上 ldd/运行确认
```
