# `/cross-env` 提示词模板

> 用于 RK3568 等嵌入式目标的 **观测-验证-构建** 交叉编译环境搭建。  
> 复制模板到 Cursor，替换 `【...】` 后发送。

---

## 模板 A — 从零搭建 Docker 交叉编译环境

```markdown
/cross-env

## 目标
为 【RK3568 / 其他 ARM 板】搭建可复现的 Docker 交叉编译环境。

## 项目信息
- 仓库路径：【@项目路径】
- 目标架构：【aarch64 / armhf】
- 可执行文件目标名：【如 sensor_daemon】
- 已知厂商依赖：【如 librknn / 无】

## 指令
严格按 EMBEDDED_CROSS_COMPILE_SKILL.md 执行：

1. 若需求未共识，先 Grill（一次一问）：板型、镜像、链接方式、部署路径、厂商库
2. **禁止在无环境指纹时生成 Dockerfile**
3. 引导我完成：基准交叉编译 → scp 上板 → 目标机采集指纹
4. 我粘贴指纹后，先输出**指纹解析表**（DISTRO/GLIBC/apt 映射/not found/vendor 库）
5. 我确认解析表后，再生成 Dockerfile + toolchain 建议
6. 闭环：容器产物再次上板 ldd 验证

现在开始。若缺指纹，请给出阶段 2 要在目标机执行的命令清单。
```

---

## 模板 B — 已有指纹，直接分析

```markdown
/cross-env --fingerprint

## 目标机环境指纹（已采集）

### uname -a
【粘贴】

### /etc/os-release
【粘贴】

### ldd --version
【粘贴】

### file + readelf
【粘贴 file 和 readelf -l 输出】

### ldd ./<executable>
【粘贴完整 ldd 输出】

## 指令
1. 解析指纹，输出指纹解析表（apt 映射 + not found + vendor 库）
2. 给出 not found 的修复方案（apt 或 rsync 精准路径）
3. **等我确认后**再生成 Dockerfile
4. 不要跳过解析表直接写 Docker
```

---

## 模板 C — ldd 报错修复

```markdown
/cross-env --fix-ldd

## 问题
板上运行或 ldd 报错：【粘贴错误信息】

### 当前 ldd 输出
【粘贴】

### 构建环境
- 编译方式：【本机交叉 / Docker / 板端原生】
- FROM 镜像（若用 Docker）：【如 ubuntu:22.04】

## 指令
按 EMBEDDED_CROSS_COMPILE_SKILL.md §6 诊断：
1. 判断是 glibc / libstdc++ / 缺库 / vendor 库哪一类
2. 给出最小修复（改 FROM、补 apt、rsync 单库、或 CMake rpath）
3. 说明修复后需要重新执行的验证命令（scp + ldd）
```

---

## 模板 D — 专有库精准同步

```markdown
/cross-env --vendor-sync

## 缺失库
ldd 显示 not found：【库名，如 librknnrt.so】

## 目标机信息
- IP：【】
- 板上已知路径：【若已知，如 /usr/lib/librknnrt.so】

## 指令
1. 指导我在目标机查找库：`find /usr -name "【库名】*"`
2. 给出精准 rsync 命令（禁止全量 rootfs 同步）
3. 说明 Dockerfile 中 COPY 路径与 CMake FIND_ROOT_PATH 如何配置
4. 更新 target-sysroot-manifest 的记录模板
```

---

## 模板 E — 极简触发

```markdown
/cross-env 【一句话，如：给 RK3568 sensor_daemon 搭 Docker 交叉编译环境】

按观测-验证-构建流程。无指纹则先引导采集，禁止凭空写 Dockerfile。
```

---

## 与 Grill / TDD 的衔接

环境就绪后：

```markdown
交叉编译环境已验证（ldd 无 not found）。请按之前的 Grill 共识进入 /tdd，从行为 #1 开始。
```
