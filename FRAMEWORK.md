# CubeSandbox 代码框架总结

> 面向 AI Agent 的即时、并发、安全、轻量级沙箱服务。基于 RustVMM + KVM，单沙箱冷启动 <60ms、内存开销 <5MB，兼容 E2B SDK。 本文档基于对仓库源码/文档/OpenAPI 契约/构建系统的通读整理，用于快速理解整体架构与各组件职责。

---

## 1. 定位与核心能力

CubeSandbox 是一个为 AI Agent 打造的沙箱运行时平台，每个沙箱是一台独立内核的 KVM MicroVM（硬件级隔离），而非共享内核的容器。

核心特性：

- **超快启动 / 高密度**：资源池化 + 快照克隆，冷启动均值 <60ms，单实例开销 <5MB，单节点可跑数千沙箱。
- **硬件级隔离**：每个沙箱独立 Guest OS 内核，KVM + eBPF 双重边界，安全运行不受信任的 LLM 生成代码。
- **E2B SDK 兼容**：改一个环境变量即可从 E2B Cloud 无缝迁移，业务代码零改动。
- **AutoPause / AutoResume**：空闲沙箱自动快照挂起（零资源占用），下次请求透明恢复（百毫秒级）。
- **Snapshot / Clone / Rollback**：百毫秒级检查点，可回滚到任意状态或分叉出多个探索分支（由 CubeCoW Copy-on-Write 引擎驱动）。
- **网络安全**：eBPF 内核级沙箱间隔离与出向过滤；L7 安全代理（CubeEgress）做域名过滤 + 凭据注入（密钥不进入沙箱）。
- **Volume 框架 / 模板系统 / Web 控制台 / 多种部署方式**（裸金属、K8s、腾讯云 Terraform）。

---

## 2. 整体架构

```
                         AI Agent (E2B SDK / cubesandbox SDK)
                                       │
              ┌────────────────────────┼────────────────────────┐
              │ 控制面 (Control Plane)                           │
              │                                                  │
        ┌─────▼─────┐   ┌──────────┐   ┌──────────┐             │
        │  CubeAPI  │   │  CubeOps │   │  WebUI   │  :12088     │
        │(Rust/Axum)│   │  (Go)    │◄──┤ (nginx)  │             │
        │ E2B兼容API│   │ 运维/Admin│   └──────────┘             │
        └─────┬─────┘   └────┬─────┘                            │
              │              │         共享 MySQL (via CubeDB)   │
              └──────┬───────┘                                  │
                     ▼                                          │
              ┌──────────────┐    Redis (生命周期事件流)          │
              │  CubeMaster  │◄──►┌──────────────────────┐      │
              │ (Go, 集群编排)│    │cube-lifecycle-manager│      │
              └──────┬───────┘    └──────────────────────┘      │
              └──────┼─────────────────────────────────────────┘
                     │ 调度分发
      ┌──────────────┼───────────────────────────────┐
      │ 数据面 / 计算节点 (Data Plane / Node)          │
      │              ▼                                 │
      │        ┌──────────┐   ┌─────────────┐          │
      │        │ Cubelet  │   │network-agent│          │
      │        │(节点本地  │   │ (节点网络编排)│          │
      │        │ 调度)     │   └──────┬──────┘          │
      │        └────┬─────┘          │                 │
      │             │           ┌────▼─────┐           │
      │      ┌──────▼──────┐    │  CubeVS  │ eBPF虚拟交换│
      │      │  CubeShim   │    └──────────┘           │
      │      │(containerd  │    ┌──────────┐           │
      │      │  shim v2)   │    │CubeEgress│ L7出向网关 │
      │      └──────┬──────┘    └──────────┘           │
      │             │ ttrpc/vsock                       │
      │      ┌──────▼────────────────────┐             │
      │      │  MicroVM (KVM)             │             │
      │      │  ┌──────────────────────┐  │             │
      │      │  │ CubeHypervisor       │  │             │
      │      │  │  (RustVMM/KVM VMM)   │  │             │
      │      │  │  ┌────────────────┐  │  │             │
      │      │  │  │ cube-agent PID1│  │  │             │
      │      │  │  │  container工作负载│ │  │             │
      │      │  │  └────────────────┘  │  │             │
      │      │  └──────────────────────┘  │             │
      │      └────────────────────────────┘             │
      │      CubeCoW (CoW存储引擎) ── 快照/克隆            │
      │      CubeProxy (反向代理, 路由到具体沙箱)          │
      └────────────────────────────────────────────────┘
```

### 请求链路（举例：创建 / 删除沙箱）

1. Agent 通过 E2B SDK → **CubeAPI**（或 WebUI → **CubeOps**）。
2. → **CubeMaster** 做集群资源调度，选定节点、分发到 **Cubelet**。
3. Cubelet → **CubeShim**（containerd shim v2）→ 拉起 **CubeHypervisor** 的 KVM MicroVM。
4. MicroVM 内 **cube-agent** 作为 PID 1 启动，通过 ttrpc/vsock 接管容器生命周期。
5. 网络由 **network-agent + CubeVS**（eBPF）打通，出向流量经 **CubeEgress** 审计过滤。
6. 数据面访问经 **CubeProxy** 路由到具体沙箱实例。

---

## 3. 组件清单

### 3.1 控制面（Control Plane）

| 组件                         | 语言/技术       | 职责                                                                                                                              |
| -------------------------- | ----------- | ------------------------------------------------------------------------------------------------------------------------------- |
| **CubeAPI**                | Rust / Axum | 高并发无状态 REST 网关，**E2B 协议兼容**。改 URL 即可迁移。无 DB。默认监听 `0.0.0.0:3000`。支持 callback / simple-key 两种鉴权。                                  |
| **CubeMaster**             | Go          | 集群编排器，接收 API 请求分发到各 Cubelet，管理资源调度与集群状态。默认 `:8089`。                                                                             |
| **CubeOps**                | Go          | 运维/Admin 后端 + WebUI 的 SDK 代理。有状态，与 CubeMaster 共享 MySQL。监听 `:3010`。提供 JWT 鉴权、AgentHub、集群监控、Store 元数据、SDK 代理（直连 CubeMaster REST）。 |
| **CubeDB**                 | Go          | CubeMaster 与 CubeOps 共享的 DB 迁移 + DAO 包。封装 goose，带**内容指纹防篡改**和**集群级会话锁**。                                                        |
| **cube-lifecycle-manager** | Go          | AutoPause 协调器（跑在控制节点）。消费 CubeMaster 经 Redis stream 发布的生命周期事件，发现所有 CubeProxy 副本并广播状态，用 Redis `SETNX` 解决跨副本竞争。                    |

### 3.2 数据面 / 计算节点（Data Plane）

| 组件                                 | 语言/技术                  | 职责                                                                                                                                                                                                                                  |
| ---------------------------------- | ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Cubelet**                        | Go                     | 计算节点本地调度组件，管理节点上所有沙箱实例的完整生命周期。集成 cubecow（cgo 静态链接 `libcubecow.a`）。提供 gRPC API（`api/services/cubebox/v1/cubebox.proto`，含 CreateSandbox / Exec / Snapshot / Commit 等）。                                                                |
| **CubeShim**                       | Rust                   | containerd Shim v2 实现（`containerd-shim-cube-rs`），桥接 containerd 与 Cube 沙箱 VM 生命周期。子组件：`shim/`（shim 进程）、`cube-runtime`（Cubelet 调用的快照/恢复 CLI）、`protoc/`（构建期 protobuf 生成，源自 Kata Containers）。                                           |
| **CubeHypervisor** (`hypervisor/`) | Rust                   | 基于 **Cloud Hypervisor / RustVMM** 定制的 KVM MicroVM 监视器。管理 CPU/内存/PCI 热插拔、迁移。子 crate：`vmm/`、`pci/`、`qcow/`、`vhdx/`、`arch/`、`tpm/` 等。                                                                                                  |
| **agent** (`cube-agent`)           | Rust (musl 静态)         | **VM 内 Guest Agent**，作为 PID 1（`/sbin/init`）运行。派生自 Kata Containers agent。通过 vsock 上的 ttrpc 暴露 API，接受 shim 驱动，用 `rustjail/` 做 OCI 容器运行。子模块：`src/`（main/rpc/sandbox/mount/network）、`rustjail/`、`cube/`、`libs/`（protocols/oci/logging）。 |
| **cubecow**                        | Rust (库 crate + C FFI) | Copy-On-Write 存储引擎，基于 xfs-reflink（FICLONE ioctl）实现 O(1) 快照/克隆。通过 `Engine` trait 抽象后端，当前仅 `ReflinkEngine`。产出 `libcubecow.a/.so` + `cubecow.h`，供 Cubelet cgo 调用。                                                                      |

### 3.3 网络（CubeNet 及相关）

| 组件                            | 语言/技术                   | 职责                                                                                                                                              |
| ----------------------------- | ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| **CubeVS** (`CubeNet/cubevs`) | Go + eBPF (C)           | 基于 eBPF 的虚拟交换机，提供内核级网络隔离与安全策略。eBPF 程序在 `CubeNet/src/*.bpf.c`（localgw / mvmtap / nodenic），Go 侧做加载与策略管理（netpolicy / dnspolicy / snat / tap / tc）。 |
| **network-agent**             | Go                      | 开源版新增的节点本地网络组件，承接从 Cubelet 迁出的本地网络编排，复用 cubevs 执行能力。对 Cubelet 提供 `Ensure / Release / Reconcile / Health` 接口，含本地 TAP 创建、HostPort 用户态代理、状态落盘恢复。   |
| **CubeEgress**                | OpenResty (nginx + Lua) | L7 出向安全网关：域名过滤、凭据注入、访问审计。Lua 脚本：`admin/audit/policy/redactor`。与 CubeVS 内核策略配合，流量无法绕过检查。                                                         |
| **CubeProxy**                 | OpenResty (nginx + Lua) | 反向代理，兼容 E2B 协议，把请求路由到具体沙箱实例。Lua：`log_phase / utils`。数据面探针 `:8082/admin/healthz`。                                                                |

### 3.4 SDK / 客户端 / 工具

| 组件                        | 语言         | 说明                                                                 |
| ------------------------- | ---------- | ------------------------------------------------------------------ |
| **sdk/go**                | Go         | Go SDK，对齐 Python SDK 面：沙箱生命周期、代码执行、命令、PTY、文件系统操作、快照/克隆/回滚、L7 出向策略。 |
| **sdk/node**              | TypeScript | Node SDK（commands/filesystem/policy/models）。                       |
| **cubelog**               | Go         | 结构化日志库（buffer_pool / context / entry / metric），被 Go 组件复用。          |
| **CubeOps CLI / cubeops** | Go         | 运维二进制。                                                             |
| **examples/cube-bench**   | Go         | 性能压测 TUI 工具。                                                       |

---

## 4. 沙箱生命周期（Lifecycle）

状态机：

```
   create()   ┌─────────┐  timeout & on_timeout=pause  ┌────────┐
  ───────────►│ running │ ────────────────────────────►│ paused │
              │         │◄── connect()/auto_resume 请求  │        │
              └──┬───┬──┘                               └───┬────┘
       kill()    │   │ timeout & on_timeout=kill            │ kill()
                 │   └──────────────┐                       │
                 ▼                  ▼                       ▼
                              ┌────────────┐
                              │ terminated │
                              └────────────┘
```

状态：`running` / `pausing`（瞬态）/ `paused`（零 CPU/内存开销，快照持久化）/ `resuming`（瞬态）/ `terminated`（不可恢复）。

两个关键设置：

- **`timeout`**：空闲多少秒后触发超时（Cube 用秒；E2B 用毫秒）。`-1`=永不超时，`0`=立即回收，省略=用服务端默认（`CubeMaster/conf.yaml` 的 `default_timeout_insec`，仓库默认 `-1` 即不回收）。
- **`on_timeout`**：`kill`（默认，销毁）或 `pause`（快照挂起）。

**AutoPause / AutoResume**：`lifecycle.on_timeout="pause"` + `auto_resume=True`，由 `cube-lifecycle-manager` 经 Redis 事件流协调。恢复对调用方透明，每次成功 resume 重置 timeout。

**暂停资源释放**：节点级调节旋钮 `host.quota.paused_resource_release_ratio`（`Cubelet/config/config.toml`，`[0,1]`，默认 `0`）——0 表示暂停仍占满配额（恢复永不失败），1 表示完全释放（恢复尽力而为，可能被拒 409）。

---

## 5. 多 Agent 并行 Rollout

CubeSandbox 从平台架构到示例工具都为**大规模并行 rollout** 设计——每个 rollout 是一台独立的 KVM MicroVM，硬件级隔离、互不干扰，天然适合 RL 的 batch / group 采样。

### 5.1 平台层能力

- **秒级冷启动 + 高密度**：冷启动均值 <60ms、单实例开销 <5MB，单节点可跑数千沙箱——为 batch rollout 提供吞吐基础。
- **无状态 API + 集群调度**：`CubeAPI` 高并发无状态，`CubeMaster` 做集群级资源调度，把沙箱分发到多节点 `Cubelet`，并发可横向扩展到整个集群。
- **Snapshot / Clone / Fork**：CubeCoW 基于 xfs-reflink 的 O(1) 克隆，可从同一检查点分叉出多个探索分支，用于 RL 的并行探索。
- **AutoPause / AutoResume**：空闲沙箱自动挂起释放资源、请求到来时透明恢复，提升在飞沙箱密度。

### 5.2 示例层：`run-concurrent.py` 并行运行器

`examples/mini-rl-training/scripts/run-concurrent.py` 提供开箱即用的多 Agent 并行 rollout 工具：

| 能力 | 说明 |
|------|------|
| **线程池并发** | `ThreadPoolExecutor(max_workers=workers)`，每 worker 独立驱动一个 agent × 沙箱 |
| **任务矩阵展开** | `model × instance × repeat`，一次运行可覆盖多模型、多题目、多采样 |
| **多 Agent / 多模型** | `-m` 支持逗号分隔多模型或 `tokenhub` 全量，多个 Policy 同时 rollout |
| **同题多采样** | `--repeat N` 对同一题目并行跑多次，正是 GRPO/PPO 所需的 group rollout |
| **批量预创建** | `--pre-create` + `batch_create_sandboxes` 先并行拉起所有沙箱再启动 agent，压缩总耗时 |
| **连接池复用** | 共享 `ApiClient` keep-alive 池，降低高并发下每请求的 TCP/TLS 开销 |
| **纯压测模式** | `benchmark` 子命令与 `--sandbox-only` 专测高并发沙箱创建/销毁吞吐 |

对应 PRD 的 RL 愿景（5.3 节）：**Batch 采样——并行创建多个沙箱，同一题目多次尝试**，收集 (state, action, reward) 轨迹供 GRPO/PPO 更新 Policy。

### 5.3 边界与注意事项

- 并发上限受**节点资源配额**约束（`Cubelet` 的 `host.quota`）；AutoPause 的 `paused_resource_release_ratio` 决定暂停沙箱是否仍占配额，进而影响可同时在飞的沙箱数。
- 示例运行器是**单机线程池**驱动多沙箱；跨多节点的更大规模并发依赖 `CubeMaster` 集群调度分发（平台支持，示例脚本本身单机驱动）。
- LLM 推理（Policy）运行在沙箱外的 GPU 侧，与沙箱完全解耦；沙箱只承担 CPU 侧的 Environment 执行与 reward 计算。

---

## 6. 对外 API（E2B 兼容）

CubeAPI 提供 E2B 兼容的沙箱 REST API（`openapi.yml` 描述的是 dashboard 面，前缀 `/cubeapi/v1`）。

E2B 兼容 SDK 接口（`CubeAPI/README.md`）：

| Method     | Path                                                       | 说明                     | 状态                   |
| ---------- | ---------------------------------------------------------- | ---------------------- | -------------------- |
| GET        | `/health`                                                  | 健康检查（免鉴权）              | ✅                    |
| GET        | `/sandboxes` · `/v2/sandboxes`                             | 列出沙箱（v2 支持状态/元数据过滤）    | ✅                    |
| POST       | `/sandboxes`                                               | 创建沙箱                   | ✅                    |
| GET/DELETE | `/sandboxes/:id`                                           | 详情 / 销毁                | ✅                    |
| POST       | `/sandboxes/:id/pause` `/resume` `/connect`                | 暂停 / 恢复(废弃) / 连接(自动恢复) | ✅                    |
| GET        | `/sandboxes/:id/logs`、`/timeout`、`/metrics`、`/snapshots` 等 | 日志/超时/指标/快照            | ❌（未注册或依赖 CubeMaster） |

Cube 扩展：`metadata.host-mount` 挂载宿主目录；内置 Chromium 浏览器沙箱（CDP + Playwright）。

Dashboard API（`openapi.yml`）：`/cluster/overview`、`/cluster/versions`、`/nodes`、`/templates`（含 aliases / compat / builds）、`/v2/sandboxes/*/logs` 等。鉴权：`X-API-Key` 或 Bearer JWT。

---

## 7. 构建系统

根 `Makefile` 用统一的 Docker builder 镜像（`cube-sandbox-builder:ubuntu2004`）做可复现构建。

- 5 个顶层 Rust 工程（各自 Cargo workspace）：`CubeAPI` / `CubeShim` / `agent` / `cubecow` / `hypervisor`。
- 产物二进制（`_output/bin/`）：`agent` `cubeapi` `cubelet` `cubemaster` `cubeops` `cubevsmapdump` `network-agent` `shim`。
- 关键 target：`make all`（全量构建）、`make <组件>`、`make <组件>-test`、`make cubecow-sdk`（构建静态库供 Cubelet cgo 链接）、`make guest-kernel`（构建 Guest 内核 vmlinux）、`make manual-release`（打包手动更新 tarball）、`make web-*`（WebUI）、`make fmt`（各组件格式化）。
- 版本注入：`CUBE_VERSION` / `CUBE_COMMIT` / `CUBE_BUILD_TIME` 三元组。
- ARM64：`TARGET_ARCH` 自动识别，支持 x86_64 ↔ aarch64 交叉编译。

---

## 8. 部署方式

- **PVM · 云 VM**（推荐）：普通云主机，无需裸金属或嵌套虚拟化。
- **裸金属**（Bare Metal）：需 x86_64 Linux + KVM。
- **Kubernetes**：Helm 部署控制面 + 计算节点（preview）。
- **腾讯云 Terraform**：一键生产集群。
- **dev-env**：无 KVM 时用一次性 OpenCloudOS 9 QEMU VM 体验（性能差，不推荐生产）。
- **one-click**（`deploy/one-click/`）：离线发布包，单机 `install.sh` 一键安装，systemd 管理（如 `cube-sandbox-cubemaster.service` / `cube-sandbox-cubeops.service`）。

装完访问 Web 控制台：`http://<控制节点IP>:12088`。

---

## 9. 目录速查

```
CubeSandbox/
├── CubeAPI/          Rust/Axum E2B 兼容 API 网关
├── CubeMaster/       Go 集群编排器
├── CubeOps/          Go 运维/Admin API + WebUI SDK 代理
├── CubeDB/           Go 共享 DB 迁移 + DAO（goose 封装）
├── Cubelet/          Go 节点本地调度（cubebox.proto gRPC，cgo 链 cubecow）
├── CubeShim/         Rust containerd shim v2 + cube-runtime
├── hypervisor/       Rust KVM MicroVM VMM（Cloud Hypervisor 定制）
├── agent/            Rust VM 内 Guest Agent（cube-agent, PID1, 源自 Kata）
├── cubecow/          Rust CoW 存储引擎（xfs-reflink/FICLONE, C FFI）
├── CubeNet/          eBPF 虚拟交换 CubeVS（cubevs Go + src/*.bpf.c）
├── network-agent/    Go 节点本地网络编排
├── CubeEgress/       OpenResty L7 出向安全网关（Lua）
├── CubeProxy/        OpenResty 反向代理（Lua）
├── cubelog/          Go 结构化日志库
├── sdk/{go,node}/    客户端 SDK
├── deploy/           部署（one-click / pvm）
├── dev-env/          QEMU 开发环境脚本
├── docker/           builder / cube-base 镜像
├── docs/             VitePress 文档站（guide / architecture / changelog / blog）
├── examples/         示例（cube-bench 压测、ivshmem）
├── scripts/          构建/发布脚本
├── openapi.yml       CubeAPI dashboard OpenAPI 契约
└── Makefile          统一 Docker builder 构建入口
```

---

## 10. 关键技术栈

- **虚拟化**：KVM + RustVMM（Cloud Hypervisor 定制）
- **容器运行时集成**：containerd Shim v2（Rust）+ ttrpc/vsock + rustjail（OCI）
- **存储/快照**：xfs-reflink FICLONE Copy-on-Write（cubecow）
- **网络**：eBPF（CubeVS）+ OpenResty/Lua（CubeEgress / CubeProxy）
- **控制面**：Rust/Axum（CubeAPI）+ Go（CubeMaster/CubeOps）+ MySQL + Redis
- **协议兼容**：E2B SDK
- **上游致谢**：Cloud Hypervisor、Kata Containers、virtiofsd、containerd-shim-rs、ttrpc-rust（Apache-2.0）

---

_说明：本文档为代码框架级别的高层总结，各组件的详细接口/配置请参阅对应目录下的 README 与 `docs/` 文档站。_

```
```
