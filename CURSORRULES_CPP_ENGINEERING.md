# C/C++ 工程行为规则

> 将下方内容复制到你的 `.cursorrules` 或 `.cursor/rules/` 规则文件中。

---

## 项目类型

这是一个 **C/C++ 项目**。构建系统为 **CMake**（或 Makefile legacy）。测试框架为 **Google Test / Catch2 / CTest**。

**严禁**生成或依赖以下内容：`package.json`、`node_modules`、`.ts`/`.tsx` 文件、`npm`/`yarn`/`pnpm` 命令。

---

## 默认工作模式

当用户指令**模糊**（如「帮我加个传感器模块」「修一下这个 bug」「实现 XXX 功能」）且未明确指定流程时，按以下顺序引导：

### 1. 判断任务类型

| 用户意图 | 默认流程 |
|----------|----------|
| 新功能 / 新模块 / 接口设计 | 先 **Grill（需求盘问）**，再 **TDD** |
| 明确的小改动（单行修复、typo、注释） | 直接改，但改完必须编译 + 跑相关测试 |
| Bug 报告 / 测试失败 | **诊断流程**（先建立失败反馈回路，再修复） |
| 用户说 `/grill` 或「盘问一下需求」 | 进入 Grill 模式 |
| 用户说 `/tdd` 或「测试驱动」 | 进入 TDD 模式（若需求未共识，先 Grill） |
| 用户说 `/cross-env` 或请求 Docker/交叉编译环境 | 进入 **嵌入式交叉编译 Skill**（见下文） |
| RK3568 等板子部署、scp、ldd 问题 | 嵌入式交叉编译 Skill 或 Bug 诊断 |

### 2. Grill 模式（需求盘问）

- **一次只问一个问题**，等用户回答后再问下一个。
- 每个问题给出**推荐答案**。
- 能通过读代码库回答的问题，**先读代码库**，不要问用户。
- 沿设计决策树逐分支推进；先解决依赖决策。
- 关注：行为边界、公共接口（头文件）、错误处理、资源所有权、线程/ISR 约束、硬件依赖、CMake 影响、测试策略。
- Grill 结束时输出**共识摘要**：接口草案、行为清单、CMake 变更范围。
- 获得用户确认后才进入实现。

### 3. TDD 模式（测试驱动开发）

- 使用**垂直切片**：一个测试 → 最小实现 → 通过 → 下一个。禁止先写完全部测试。
- 测试只通过**公共头文件 API** 验证**可观察行为**，不测试实现细节。
- 每个周期：
  1. **先更新 `CMakeLists.txt`**（新源文件、新测试、新依赖、include 路径）
  2. 写失败测试（RED）→ 运行 `ctest` 并展示失败输出
  3. 写最小实现（GREEN）→ 运行 `ctest` 并展示通过输出
  4. 全部 GREEN 后重构，每步重构后重跑 `ctest`
- 实现完成后对变更的 `.c`/`.cpp`/`.h` 运行 `clang-tidy` 或 `cppcheck`，展示结果。

---

## CMakeLists.txt 强制规则

**在创建或修改任何头文件（公共 API）或源文件之前，必须先更新 `CMakeLists.txt`。**

必须更新 CMake 的情况：

- 新增/删除 `.cpp`、`.c`、测试文件
- 新增/变更公共头文件或 include 目录
- 新增/变更 target 间链接关系
- 新增外部依赖（`find_package`、`FetchContent`、`target_link_libraries`）

正确顺序：

```text
接口设计 → 更新 CMakeLists.txt → 确认 cmake 配置通过 → 写测试 → 写实现 → 编译 → ctest
```

若跳过 CMake 更新直接写源文件，视为违反工程规则。

---

## 构建与验证命令

每次 substantive 代码 变更后必须执行并汇报结果：

```text
cmake --build build
cd build && ctest --output-on-failure
```

单个测试调试：

```text
cd build && ctest -R <TestName> --output-on-failure
```

静态分析（变更文件）：

```text
clang-tidy -p build <changed.cpp>
cppcheck --enable=warning,style --std=c++17 <changed.cpp>
```

若 `compile_commands.json` 不存在：

```text
cmake -S . -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
```

---

## 测试原则

- 测试名描述 **WHAT**（行为），不描述 **HOW**（实现）。
- 仅在**系统边界**使用 mock/fake：硬件总线、网络、文件系统、时钟。不 mock 项目内部模块。
- 优先依赖注入（构造函数/工厂传入接口）以提高可测性。
- 不在 RED 状态重构；先 GREEN 再重构。

---

## Bug 修复原则

1. 先建立**确定性、快速、可复现**的失败反馈（失败测试优先）。
2. 在有一个能变红的命令之前，不要猜测原因。
3. 最小修复 → 原测试变绿 → 完整 `ctest` → 静态分析。

---

## 代码风格

- 匹配项目现有风格（命名、缩进、头文件 guard、命名空间）。
- 最小 diff；不改动无关代码。
- 公共 API 放在 `include/`，实现放在 `src/`（按项目现有布局）。
- 嵌入式注意：ISR 安全、内存分配约束、错误码 vs 异常 按项目惯例。

---

## 嵌入式交叉编译（RK3568 等）

针对 Embedded Linux 目标机，遵循 **环境解耦与二进制对齐** 流程（详见 `EMBEDDED_CROSS_COMPILE_SKILL.md`）。典型环境：双网卡宿主机 — 外网装工具链，直连网卡 rsync/scp 开发板。

### 触发条件

用户请求：Docker 构建环境、交叉编译、部署到 RK3568/ARM 板、解决 `ldd not found`、`sync_sysroot`。

### 强制 Pipeline

```text
Grill 共识 → ① 联网 apt 装交叉工具链(国内源) → ② rsync sync_sysroot → ③ Docker COPY sysroot → CMake 构建 → 上板 ldd 验证
```

### Agent 硬性规则

1. **工具链与运行库分离**：apt **仅**安装 `gcc-aarch64-linux-gnu` 等宿主机架构交叉编译器；**禁止** apt 安装 `:arm64` 运行库或试图用 apt 替代 sysroot。
2. **国内源前置**：任何 apt 操作前必须先 sed 替换国内镜像源。
3. **Sysroot 来自板端**：通过 `scripts/sync_sysroot.sh` 将板子 `/lib/`、`/usr/lib/` 同步至 `./sysroot`；禁止全量 rootfs 同步。
4. **Dockerfile 只 COPY**：`COPY ./sysroot /usr/aarch64-linux-gnu`；必须 `ENV CMAKE_SYSROOT=/usr/aarch64-linux-gnu`；严禁 Dockerfile 内 apt 装 ARM64 库。
5. **CMake 隔离宿主机**：构建时必须 `-DCMAKE_FIND_ROOT_PATH=/usr/aarch64-linux-gnu`，toolchain 设 `FIND_ROOT_PATH_MODE_*` 为 ONLY/NEVER。
6. **ldd 驱动补齐**：`not found` 必须回溯到 `sync_sysroot.sh` 单库 rsync，**不得**尝试 Dockerfile apt 修复。记录到 `docs/target-sysroot-manifest.md`。
7. **双网卡路由**：rsync/scp/ssh 走板卡直连网卡（`ssh -b <板卡网卡IP>`），禁止误走外网。
8. **闭环验证**：产物 scp 上板，`ldd` 无 `not found` 后方可宣称环境就绪。

### 与 TDD 的分工

- 环境搭建：本 Skill（三阶段 Pipeline + ldd 闭环）
- 业务代码：TDD 垂直切片（主机 Fake 测试 + 板上集成验证）

---

## 禁止事项

- 不生成 npm/Node/TypeScript 相关文件或命令
- 不生成可执行脚本文件（`.sh`/`.py`/`.ps1`），除非用户明确要求
- 不在用户未要求时 git commit
- 不跳过编译和测试直接宣称完成
- 不在需求未共识时直接写大段实现
- **不在无板卡 IP/架构/网卡共识时生成 Dockerfile；禁止用 apt 替代 rsync sysroot**

---

## 完成汇报格式

任务完成时汇报：

```markdown
## 变更摘要
- （文件级变更说明）

## CMake 变更
- （target/源文件/依赖变更）

## 验证结果
- 编译：（成功/失败 + 关键输出）
- ctest：（通过数/总数）
- 静态分析：（告警数或「无新增告警」）

## 残留风险 / 待确认
- （如有）
```
