# `/tdd` 提示词模板

> 复制下方模板到 Cursor 对话，替换 `【...】` 占位符后发送。

---

## 模板 A — 标准 TDD（新功能）

```markdown
/tdd

## 目标
【要实现的功能，例如：实现 SensorReader，通过 I2C 读取设备 ID】

## 前置共识（若已 Grill，粘贴摘要；否则请先做简短 Grill）
- 公共接口：【头文件路径 + 关键签名】
- 优先行为：
  1. 【行为 1，将变为测试名】
  2. 【行为 2】
  3. 【行为 3】

## 测试框架
【Google Test / Catch2 / 项目默认】

## 指令
严格按 **TDD 垂直切片** 执行：

### 每个切片
1. **先更新 `CMakeLists.txt`**（源文件、测试 target、依赖）
2. RED：写一个测试 → `cmake --build build` → `ctest` → **展示失败输出**
3. GREEN：最小实现 → `ctest` → **展示通过输出**
4. 不添加推测性功能

### 全部 GREEN 后
- 重构（若有），每步重跑 `ctest`
- 对变更文件运行 `clang-tidy` 或 `cppcheck`，展示结果

### 禁止
- 禁止水平切片（先写全部测试再写全部实现）
- 禁止测试实现细节（只测公共 API 的可观察行为）
- 禁止跳过 CMake 更新

从**优先级最高的第一个行为**开始。现在开始第一个 RED。
```

---

## 模板 B — TDD 修复 Bug

```markdown
/tdd

## Bug 描述
【用户可见的故障现象】

## 复现条件
【如何触发，已知输入/环境】

## 指令
1. 先建立**失败反馈回路**（优先写 failing test 或最小复现）
2. 运行 `ctest` 并**展示失败输出**
3. 最小修复使测试变绿
4. 完整 `ctest` 确认无回归
5. 对变更文件做静态分析

不要在没有 failing test 之前猜测修复方案。
现在开始 Phase 1：建立失败测试。
```

---

## 模板 C — 继续未完成的 TDD

```markdown
/tdd --continue

## 当前进度
- 已完成测试：【列出已 GREEN 的测试名】
- 下一个行为：【下一个要测的行为】
- 相关文件：【@源文件或测试文件路径】

## 指令
继续垂直切片 TDD。先确认 CMakeLists.txt 是最新的，然后从下一个 RED 开始。
每个切片展示 ctest 输出。
```

---

## 模板 D — 嵌入式 / 硬件边界 TDD

```markdown
/tdd

## 目标
【模块描述，如：I2C 传感器读取器】

## 接口
【头文件 + 关键 API】

## 硬件 Fake 策略
- 系统边界：【如 II2cBus 纯虚接口 / C 函数指针表】
- Fake 实现位置：【如 tests/fakes/fake_i2c_bus.cpp】
- 板上集成测试：【如有，列出 target 名；否则「仅主机单元测试」]

## 指令
TDD 垂直切片，规则：
1. 先更新 CMakeLists.txt
2. 测试只通过公共 API + Fake 硬件，不 mock 内部模块
3. 每个切片：RED（展示 ctest 失败）→ GREEN（展示通过）
4. 全部 GREEN 后：重构 + clang-tidy/cppcheck

从 tracer bullet 开始：【第一个端到端行为，如「设备在线时返回 WHO_AM_I」】
```

---

## 模板 E — 极简 TDD（单行触发）

```markdown
/tdd 【一句话功能描述】

按 C++ TDD 垂直切片实现：先 CMake → RED → GREEN → 重构。
测试框架跟随项目默认。每个切片展示 ctest 输出。开始。
```

---

## TDD 周期检查清单（可附在任意模板末尾）

```markdown
## 每周期自检
- [ ] CMakeLists.txt 已更新
- [ ] 测试描述 WHAT 而非 HOW
- [ ] 测试只用公共头文件 API
- [ ] 代码是过当前测试的最小实现
- [ ] ctest 输出已展示
```

---

## 使用建议

| 场景 | 推荐模板 |
|------|----------|
| 新功能，需求已 Grill | 模板 A |
| Bug 修复 | 模板 B |
| 上次会话中断 | 模板 C |
| 驱动 / 传感器 / 硬件 | 模板 D |
| 快速启动 | 模板 E |

---

## Grill → TDD 衔接模板

Grill 确认后，可直接发送：

```markdown
Grill 共识已确认。请按共识摘要进入 /tdd，从行为 #1 开始垂直切片。
```
