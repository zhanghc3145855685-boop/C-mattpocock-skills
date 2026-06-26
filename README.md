# C/C++ 纯逻辑工程工作流指南

> 本文档将 mattpocock skills 库中的 `/grill`、`/tdd`、`/implement`、`/diagnosing-bugs` 等核心工程思维链，重构为**完全脱离 Node.js/TypeScript 运行环境**的静态指令。  
> 不包含任何可执行脚本；所有「自动化」均转化为 AI 或开发者应执行的人类可读步骤。
> 如需生成脚本，可直接声明或修改prompt。

---

## 1. 工作流总览

```text
需求模糊 ──► /grill（需求盘问）──► 共识设计 ──► /tdd（垂直切片）──► 实现 ──► 验证 ──► 重构
                │                                    │
                └── 产出：行为清单、接口草案、术语表 ──┘
```

| 阶段 | 目标 | 核心产出 |
|------|------|----------|
| Grill | 在写代码前消除歧义 | 行为清单、公共接口草案、边界条件、CMake 影响范围 |
| TDD | 用测试驱动实现 | 一个行为 → 一个失败测试 → 最小实现 → 通过 |
| 验证 | 确认工程健康 | 编译通过、ctest 全绿、静态分析无新增告警 |
| 重构 | 在 GREEN 后改善结构 | 提取重复、加深模块、不破坏测试 |

---

## 2. 工具链映射（TS/Node → C/C++）

| 原 TS/Node 生态 | C/C++ 替代 | AI 应执行的指令 |
|-----------------|------------|-----------------|
| `tsc --noEmit` | `cmake --build build` 或 `make` | 每次修改源文件后，必须先编译并展示编译错误 |
| `eslint` | `clang-tidy` / `cppcheck` | 实现完成后对变更文件运行静态分析，展示告警 |
| `npm test` / `jest` | `ctest` / Google Test / Catch2 | 每个 TDD 周期后运行对应测试目标；结束时运行完整测试套件 |
| `package.json` scripts | CMake 自定义 target | 通过 `cmake --build build --target <name>` 调用 |
| TypeScript 接口 | C/C++ 头文件（`.h`/`.hpp`） | 公共 API 只通过头文件暴露；测试只调用公共 API |
| jest.mock | 系统边界 Fake/Mock（gmock 或手写 stub） | 仅在外部依赖（硬件、网络、文件系统）处 mock |

### 2.1 构建系统约定

- **首选 CMake**；legacy 项目可用 Makefile，但新代码优先 CMake。
- 构建目录约定：`build/`（不提交到 git）。
- 标准命令序列：

```text
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build
cd build && ctest --output-on-failure
```

### 2.2 测试框架约定

按项目已有框架为准；若无，推荐优先级：

1. **Google Test + Google Mock**（`GTest::gtest`, `GTest::gmock`）
2. **Catch2 v3**
3. **CTest + 自定义可执行文件**（最简）

测试命名：`TEST(SuiteName, BehaviorDescription)` 或 `TEST_CASE("behavior description")`。

### 2.3 静态分析约定

实现完成后，对**本次变更的 `.c`/`.cpp`/`.h` 文件**执行：

```text
# clang-tidy（需 compile_commands.json）
clang-tidy -p build path/to/changed_file.cpp

# cppcheck
cppcheck --enable=warning,style,performance --std=c++17 path/to/changed_file.cpp
```

若项目未配置 `compile_commands.json`，先生成：

```text
cmake -S . -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
```

---

## 3. CMakeLists.txt 强制规则

**任何涉及源文件或公共接口的变更，必须先更新 CMakeLists.txt，再写实现。**

### 3.1 触发条件（必须更新 CMake）

| 操作 | CMake 变更 |
|------|------------|
| 新增 `.cpp`/`.c` 源文件 | 加入对应 `add_executable` 或 `add_library` 的源文件列表 |
| 新增仅头文件的模块（有对应 `.cpp`） | 同上 |
| 新增公共头文件目录 | 更新 `target_include_directories` |
| 新增/修改公共 API（头文件符号） | 检查依赖该 target 的其他 target 是否需 `target_link_libraries` |
| 新增测试文件 | 添加 test executable + `add_test` 或 `gtest_discover_tests` |
| 新增外部依赖 | 添加 `find_package` / `FetchContent` / `target_link_libraries` |
| 新增可执行工具/脚本 target | 添加新的 `add_executable` 及 `install` 规则（如需要） |

### 3.2 AI 执行顺序（接口变更时）

```text
1. 说明将要变更的公共接口（头文件签名）
2. 更新 CMakeLists.txt（源文件、include、link）
3. 运行 cmake -S . -B build 确认配置通过
4. 编写/更新测试
5. 编写/更新实现
6. 编译 → 测试 → 静态分析
```

### 3.3 CMake 测试注册示例（Google Test）

AI 在添加测试时应确保 CMake 包含类似结构（按项目风格调整，非强制复制）：

```cmake
enable_testing()
add_executable(my_module_tests tests/my_module_test.cpp)
target_link_libraries(my_module_tests PRIVATE my_module GTest::gtest_main)
include(GoogleTest)
gtest_discover_tests(my_module_tests)
```

---

## 4. `/grill` — 需求盘问工作流

### 4.1 哲学

Grill 不是聊天，是**设计审查**。目标是在写任何代码之前，通过逐条追问，把模糊需求变成可测试的行为清单和接口草案。

### 4.2 核心规则

1. **一次只问一个问题**，等用户回答后再继续。
2. 每个问题附带**推荐答案**（基于代码库现状或嵌入式/C++ 最佳实践）。
3. 若问题可通过阅读代码库回答，**先读代码库**，不要问用户已知信息。
4. 沿设计树逐分支深入，**先解决依赖决策，再往下走**。
5. 不使用「水平切片」：不要一次抛出 10 个问题。

### 4.3 追问维度（C/C++ 嵌入式适配）

按任务选取相关维度，不必全问：

| 维度 | 示例问题 |
|------|----------|
| 行为边界 | 「传感器读失败时，重试几次？超时多少 ms？」 |
| 公共接口 | 「对外暴露 C API 还是 C++ class？头文件放哪？」 |
| 错误处理 | 「返回错误码、std::optional，还是抛异常？」 |
| 资源所有权 | 「谁分配/释放 buffer？RAII 还是调用方负责？」 |
| 线程/并发 | 「是否在 ISR 中调用？需要 mutex 吗？」 |
| 硬件依赖 | 「直接寄存器操作还是 Linux 驱动/ioctl？」 |
| 构建影响 | 「新模块是 static lib 还是 shared lib？哪些 target 链接它？」 |
| 测试策略 | 「如何在主机上测？需要 hardware fake 吗？」 |
| 非功能需求 | 「内存上限？实时性？日志格式？」 |

### 4.4 Grill 结束标准

当以下条目全部清晰时，Grill 阶段结束：

- [ ] 公共接口草案（头文件签名级别）
- [ ] 按优先级排列的行为清单（可转化为测试名）
- [ ] 明确的错误/边界处理策略
- [ ] CMake 变更范围（新增 target、源文件、依赖）
- [ ] 用户确认可以进入 TDD

### 4.5 Grill 产出模板

Grill 结束时，AI 应输出如下摘要：

```markdown
## Grill 共识摘要

### 公共接口（头文件）
- `include/my_module/sensor_reader.h`
  - `bool sensor_read(SensorData* out, uint32_t timeout_ms);`

### 行为清单（按优先级）
1. 有效 I2C 地址时返回传感器 ID
2. 超时返回 false 且不修改 out
3. ...

### CMake 变更
- 新增 `sensor_reader.cpp` → `add_library(sensor_reader ...)`
- 新增 `tests/sensor_reader_test.cpp` → gtest target

### 未决事项
- （无 / 列出待确认项）

### 下一步
用户确认后，进入 /tdd 垂直切片。
```

---

## 5. `/tdd` — 测试驱动开发工作流

### 5.1 哲学

- 测试验证**可观察行为**，不验证实现细节。
- 测试通过**公共接口**（头文件声明的 API）调用，不测试 `static`/匿名命名空间/internal 函数。
- 好测试像规格说明：`SensorReader_ReturnsIdWhenDevicePresent` 描述能力，而非 `SensorReader_CallsI2cRead`。

### 5.2 反模式：水平切片

**禁止**先写完全部测试再写全部实现：

```text
错误：test1, test2, test3 → impl1, impl2, impl3
正确：test1 → impl1 → test2 → impl2 → ...
```

### 5.3 四阶段流程

#### 阶段 1 — Planning（规划）

写任何代码前：

- [ ] 阅读项目 `CONTEXT.md`（若存在）以匹配领域术语
- [ ] 与用户确认公共接口变更
- [ ] 与用户确认要测的行为（优先级排序）
- [ ] 识别「深模块」机会：小接口、大实现
- [ ] 列出行为清单，获得用户批准
- [ ] **先更新 CMakeLists.txt**

问用户：「公共接口应是什么样？哪些行为最重要、需要先测？」

#### 阶段 2 — Tracer Bullet（追踪弹）

写一个测试，验证一条端到端路径：

```text
RED:   写第一个行为测试 → 编译可能失败或测试失败 → 展示失败输出
GREEN: 写最小实现使测试通过 → 展示 ctest 通过输出
```

#### 阶段 3 — Incremental Loop（增量循环）

对每个剩余行为重复：

```text
RED  → 写一个测试 → 运行 ctest → 展示失败
GREEN → 最小代码通过 → 运行 ctest → 展示通过
```

规则：

- 一次一个测试
- 只写够通过当前测试的代码
- 不预测未来测试
- 不添加推测性功能

#### 阶段 4 — Refactor（重构）

所有测试 GREEN 后：

- [ ] 提取重复代码
- [ ] 加深模块（复杂度藏在接口后）
- [ ] 检查现有代码是否被新代码揭示的问题
- [ ] **每步重构后重新运行 ctest**
- [ ] 对变更文件运行 clang-tidy / cppcheck

**禁止在 RED 状态重构。** 先 GREEN，再重构。

### 5.4 好测试 vs 坏测试（C++ 示例）

**好测试** — 通过公共 API 验证行为：

```cpp
TEST(SensorReader, ReturnsIdWhenDevicePresent) {
    FakeI2cBus bus;
    bus.SetDeviceAt(0x68, {0x68, 0x39});  // WHO_AM_I
    SensorReader reader(&bus);
    EXPECT_EQ(reader.ReadDeviceId(), 0x39);
}
```

**坏测试** — 绑定实现细节：

```cpp
TEST(SensorReader, CallsI2cReadExactlyOnce) {  // 坏：测 HOW 不是 WHAT
    MockI2c mock;
    EXPECT_CALL(mock, Read(_, _, _)).Times(1);
    SensorReader reader(&mock);
    reader.ReadDeviceId();
}
```

### 5.5 Mock 边界（C/C++）

**仅在系统边界 mock/fake：**

- 硬件总线（I2C/SPI/GPIO）
- 网络 socket
- 文件系统
- 系统时钟 / 随机数
- 外部 C 库

**不要 mock：**

- 你控制的内部类/模块
- 同一库内的协作者

依赖注入方式：构造函数或工厂函数传入接口（纯虚类 / function pointer / C 函数指针表）。

### 5.6 每个 TDD 周期的检查清单

```text
[ ] 测试描述行为（WHAT），非实现（HOW）
[ ] 测试只使用公共头文件 API
[ ] 内部重构不会破坏此测试
[ ] CMakeLists.txt 已更新
[ ] 代码是过当前测试的最小实现
[ ] 无推测性功能
[ ] ctest 已运行并展示结果
```

---

## 6. 完整实现流程（Implement）

当用户给出 PRD、Issue 或 Grill 共识摘要时：

```text
1. 若无 Grill 共识 → 先 /grill
2. 更新 CMakeLists.txt
3. /tdd 垂直切片实现
4. 编译：cmake --build build
5. 测试：cd build && ctest --output-on-failure
6. 静态分析：clang-tidy / cppcheck 对变更文件
7. 汇报：变更摘要 + 测试结果 + 残留风险
```

**不要**在用户未要求时 git commit。

---

## 7. Bug 诊断流程（Diagnosing Bugs）

当用户报告 bug 或测试失败时：

### 阶段 1 — 建立反馈回路

优先构造**可重复的失败信号**：

1. 失败的单元/集成测试（首选）
2. 最小可执行程序 + 固定输入
3. 带断言的 shell 命令（`./app --fixture x; echo $?`）
4. 捕获并重放的硬件 trace/log

**在有一个「确定性、快速、可自动运行、能变红」的命令之前，不要开始猜原因。**

### 阶段 2 — 复现 + 最小化

- 确认失败与用户描述一致
- 逐步削减输入/步骤，保留最小仍失败的场景

### 阶段 3 — 假设 + 插桩

- 提出单一假设
- 添加日志/断言验证
- 运行反馈回路

### 阶段 4 — 修复

- 最小修复
- 原 failing test 变绿
- 不引入新失败

### 阶段 5 — 回归

- 完整 ctest
- 静态分析变更文件

---

## 8. 代码设计原则（Deep Modules）

设计接口时遵循：

- **深模块**：小接口 + 复杂实现藏在后面
- **接口即测试面**：测试与调用者走同一 seam
- **依赖注入**：外部依赖从外部传入，不在内部 `new`/打开设备
- **返回值优于隐式副作用**：优先返回结果 struct/optional，少改全局状态
- **删除测试**：若删除模块后复杂度分散到所有调用方，说明模块太浅

---

## 9. 项目文档约定（可选但推荐）

| 文件 | 用途 |
|------|------|
| `CONTEXT.md` | 领域术语表（无实现细节） |
| `docs/adr/NNNN-*.md` | 架构决策记录（难逆转、有取舍时使用） |
| `CMakeLists.txt` | 构建真相源 |
| `tests/` | 测试代码 |

---

## 10. 日常命令速查

| 目的 | 命令 |
|------|------|
| 配置 | `cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug -DCMAKE_EXPORT_COMPILE_COMMANDS=ON` |
| 编译 | `cmake --build build` |
| 全量测试 | `cd build && ctest --output-on-failure` |
| 单个测试 | `cd build && ctest -R TestName --output-on-failure` |
| 或直接 | `./build/my_module_tests --gtest_filter=Suite.Test` |
| clang-tidy | `clang-tidy -p build src/changed.cpp` |
| cppcheck | `cppcheck --enable=warning,style --std=c++17 src/changed.cpp` |

---

## 11. 与原始 Skills 的对应关系

| 原 Skill | 本文档章节 |
|----------|------------|
| `grilling` / `grill-me` | §4 `/grill` |
| `grill-with-docs` | §4 + §9（Grill 时更新 CONTEXT.md/ADR） |
| `tdd` | §5 `/tdd` |
| `implement` | §6 |
| `diagnosing-bugs` | §7 |
| `codebase-design` | §8 |
| `domain-modeling` | §9 |

---

## 12. 文件清单

本目录包含：

| 文件 | 用途 |
|------|------|
| `C++_ENGINEERING_GUIDE.md` | 本指南（详细操作说明） |
| `CURSORRULES_CPP_ENGINEERING.md` | 复制到 `.cursorrules` 的系统指令 |
| `PROMPT_GRILL.md` | `/grill` 日常提示词模板 |
| `PROMPT_TDD.md` | `/tdd` 日常提示词模板 |
