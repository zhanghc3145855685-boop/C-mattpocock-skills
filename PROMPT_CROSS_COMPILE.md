# `/cross-env` 提示词模板

> 用于 RK3568 等嵌入式目标的 **环境解耦 · 二进制对齐** 交叉编译环境搭建。  
> 典型环境：双网卡 + 直连开发板 + rsync sysroot。  
> 复制模板到 Cursor，替换 `【...】` 后发送。

---

## 模板 A — 从零搭建 Docker 交叉编译环境

```markdown
/cross-env

## 目标
为 【RK3568 / 其他 ARM 板】搭建可复现的 Docker 交叉编译环境（工具链 apt + 板端 rsync sysroot）。

## 项目信息
- 仓库路径：【@项目路径】
- 目标架构：【aarch64 / armhf】
- 可执行文件目标名：【如 sensor_daemon】
- 板卡直连 IP：【192.168.1.100】
- 宿主机板卡网卡绑定 IP：【192.168.1.10 / 留空】
- 已知厂商依赖：【如 librknn / 无】

## 指令
严格按 EMBEDDED_CROSS_COMPILE_SKILL.md 执行：

1. 若需求未共识，先 Grill（一次一问）：板型、镜像、网卡、板卡 IP、链接方式、部署路径
2. **禁止 apt 安装 :arm64 运行库**；ARM64 库只来自 rsync sysroot
3. 阶段 1：给出国内源 sed + apt 装交叉工具链命令
4. 阶段 2：生成/完善 `scripts/sync_sysroot.sh`，引导我执行 rsync
5. 阶段 3：sysroot 就绪后生成 Dockerfile（COPY sysroot + ENV CMAKE_SYSROOT）+ toolchain
6. 闭环：构建产物 scp 上板 ldd 验证；not found → 单库 rsync 补齐

现在开始。若缺板卡信息，先 Grill。
```

---

## 模板 B — Sysroot 已同步，直接生成 Dockerfile

```markdown
/cross-env --sysroot-ready

## 板卡信息
- 板型：【RK3568 + Ubuntu 20.04】
- 板卡 IP：【】
- sysroot 路径：【./sysroot】
- 已同步库数量：【find sysroot -name '*.so*' | wc -l 输出】

## 环境指纹（可选，用于存档）
### uname -a
【粘贴】

### /etc/os-release
【粘贴】

### ldd --version
【粘贴】

## 指令
1. 确认 sysroot 结构是否完整（lib、usr/lib、aarch64-linux-gnu）
2. 生成 Dockerfile（禁止 apt ARM64 库，COPY sysroot，ENV CMAKE_SYSROOT）
3. 生成/检查 cmake/toolchain-aarch64.cmake（FIND_ROOT_PATH + MODE）
4. 给出 docker build 与 scp + ldd 验证命令
```

---

## 模板 C — ldd 报错修复

```markdown
/cross-env --fix-ldd

## 问题
板上 ldd 报错：【粘贴完整 ldd 输出，含 not found 行】

### 构建环境
- 编译方式：【本机交叉 / Docker】
- sysroot 同步日期：【】
- 是否曾尝试 apt 装 :arm64 包：【是/否】

## 指令
按 EMBEDDED_CROSS_COMPILE_SKILL.md §7 诊断：
1. 判断缺库类型（未同步 / 厂商库 / glibc 漂移）
2. 给出板上 find 定位 + 单库 rsync 补齐命令（禁止 Dockerfile apt 修复）
3. 说明修复后需重新 docker build 并 scp + ldd 验证
```

---

## 模板 D — 专有库 / 单库补齐

```markdown
/cross-env --vendor-sync

## 缺失库
ldd 显示 not found：【库名，如 librknnrt.so】

## 目标机信息
- 板卡 IP：【】
- 宿主机板卡网卡 IP：【】
- 板上已知路径：【若已知】

## 指令
1. 指导我在板上查找：`find /usr /lib -name "【库名】*"`
2. 给出精准 rsync 单库命令（禁止全量 rootfs）
3. 说明重新 docker build 后 COPY 生效路径
4. 更新 docs/target-sysroot-manifest.md 记录模板
```

---

## 模板 E — 双网卡环境确认

```markdown
/cross-env --network

## 网络拓扑
- 外网网卡：【eth0 / 可联网】
- 板卡直连网卡：【eth1 / IP 192.168.1.10】
- 开发板 IP：【192.168.1.100】
- SSH 用户：【root】

## 指令
1. 确认 rsync/scp 走直连网卡的 ssh -b 参数
2. 给出 sync_sysroot.sh 中 BOARD_SSH_BIND 配置
3. 测试连通性命令（ping + ssh + rsync dry-run）
```

---

## 模板 F — 极简触发

```markdown
/cross-env 【一句话，如：给 RK3568 sensor_daemon 搭 Docker 交叉编译环境】

按环境解耦三阶段流程（apt 工具链 → rsync sysroot → Docker COPY）。禁止 apt 装 ARM64 库。
```

---

## 与 Grill / TDD 的衔接

环境就绪后：

```markdown
交叉编译环境已验证（ldd 无 not found）。请按之前的 Grill 共识进入 /tdd，从行为 #1 开始。
```
