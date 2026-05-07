##### 5.2.5. Sandbox Infrastructure for Agentic AI

To meet the diverse execution demands of agentic AI during post-training and evaluation, we build a production-grade sandbox platform, DeepSeek Elastic Compute (DSec). DSec comprises three Rust components — the API gateway (Apiserver), per-host agent (Edge), and the cluster monitor (Watcher) — that are interconnected by a custom RPC protocol and scale horizontally atop the 3FS distributed filesystem (DeepSeek-AI, 2025). In production, a single DSec cluster manages hundreds of thousands of concurrent sandbox instances.

The design of DSec is motivated by four observations: (1) agentic workloads are highly heterogeneous, spanning lightweight function calls to full software-engineering pipelines with diverse OS and security requirements; (2) environment images are numerous and large, yet must load quickly and support iterative customization; (3) high-density deployment demands efficient CPU and memory utilization; (4) sandbox lifecycles must coordinate with GPU training schedules, including preemption and checkpoint-based resumption. Based on these observations, we elaborate on the four core designs of DSec individually in the following.

Four Execution Substrates Behind One Unified Interface. DSec exposes a single Python SDK (libdsec) that abstracts four execution substrates. Function Call dispatches stateless invocations to a pre-warmed container pool, eliminating cold-start overhead. Container is fully Docker-compatible and leverages EROFS (Gao et al., 2019) on-demand loading for efficient image assembly. microVM, built on Firecracker (Agache et al., 2020), adds VM-level isolation for security-sensitive, high-density deployments. fullVM, built on QEMU (Bellard, 2005), supports arbitrary guest operating systems. All four share a common API surface — command execution, file transfer, and TTY access — and switching between them requires only a parameter change.

Fast Image Loading via Layered Storage. DSec reconciles fast startup with a large and growing corpus of environment images through layered, on-demand loading. For containers, base images and filesystem commits are stored as 3FS-backed readonly EROFS layers mounted directly into overlay lowerdirs. We keep file metadata readily available on the local disk at mount time;

meanwhile, data blocks are fetched from 3FS upon request. For microVMs, DSec uses the overlaybd (Li et al., 2020) disk format: the read-only base layer resides on 3FS for cross-instance sharing, while writes go to a local copy-on-write layer. Such snapshots are chainable, facilitating efficient versioning and millisecond-scale resumption.

Density Optimizations Under Massive Concurrency. To accommodate hundreds of thousands of sandboxes per cluster, DSec tackles two resource bottlenecks. First, it mitigates duplicate page-cache footprints in virtualized environments and applies memory reclamation to enable safe overcommitment. Second, it alleviates spinlock contention in the container runtime and therefore, reduces per-sandbox CPU overhead, significantly increasing per-host packing density.

Trajectory Logging and Preemption-Safe Resumption. DSec maintains a globally ordered trajectory log for each sandbox, persistently recording every command invocation and its results. The trajectory serves three purposes: (1) client fast-forwarding — when a training task is preempted, sandbox resources are retained nonetheless; upon resumption, DSec replays cached results for previously completed commands, accelerating task recovery whilst also preventing errors from re-execution of non-idempotent operations; (2) fine-grained provenance — the origin and corresponding outcomes of each state change are traceable; (3) deterministic replay — any historical session can be faithfully reproduced from its trajectory.


### 5.2.5. 面向智能体 AI 的沙箱基础设施

为了满足智能体 AI 在后训练和评估阶段多样化的执行需求，我们构建了一个生产级的沙箱平台——**DeepSeek Elastic Compute (DSec)**。DSec 由三个用 Rust 编写的组件构成——API 网关 (Apiserver)、每主机代理 (Edge) 和集群监控器 (Watcher)——它们通过自定义的 RPC 协议互联，并构建在 3FS 分布式文件系统之上，支持水平扩展。在生产环境中，单个 DSec 集群能够管理数十万个并发沙箱实例。

DSec 的设计源于四个观察：(1) 智能体的工作负载具有高度异构性，涵盖了从轻量级函数调用到包含多样化操作系统和安全要求的全套软件工程流水线；(2) 环境镜像数量庞大且体积巨大，但必须能够快速加载并支持迭代定制；(3) 高密度部署需要高效的 CPU 和内存利用率；(4) 沙箱的生命周期必须与 GPU 训练调度相协调，包括任务抢占和基于检查点的恢复。基于这些观察，我们将在下文中分别详述 DSec 的四个核心设计。

#### 统一接口背后的四种执行底座
DSec 通过一个统一的 Python SDK (`libdsec`) 抽象了四种执行底座：
1.  **Function Call（函数调用）**：将无状态调用分发给预热的容器池，消除了冷启动的开销。
2.  **Container（容器）**：完全兼容 Docker，并利用 EROFS 按需加载技术实现高效的镜像组装。
3.  **microVM（微虚拟机）**：基于 Firecracker 构建，为安全敏感且需高密度部署的场景增加了虚拟机级别的隔离。
4.  **fullVM（全虚拟机）**：基于 QEMU 构建，支持任意的客户操作系统。
所有这四种模式共享通用的 API 界面——命令执行、文件传输和 TTY 访问——并且只需更改参数即可在它们之间切换。

#### 通过分层存储实现快速镜像加载
DSec 通过分层按需加载，在快速启动与庞大且不断增长的环境镜像库之间取得了平衡。
*   **对于容器**：基础镜像和文件系统提交被存储为基于 3FS 的只读 EROFS 层，直接挂载到 overlay 的 lowerdirs 中。我们在挂载时将文件元数据保留在本地磁盘上；同时，数据块则按需从 3FS 获取。
*   **对于微虚拟机**：DSec 使用 overlaybd 磁盘格式：只读基础层驻留在 3FS 上以供跨实例共享，而写入操作则进入本地的写时复制层。这种快照支持链式结构，便于高效的版本控制和毫秒级的恢复。

#### 大规模并发下的密度优化
为了在每个集群中容纳数十万个沙箱，DSec 解决了两个资源瓶颈。
*   首先，它减少了虚拟化环境中的重复页缓存占用，并应用内存回收机制以实现安全的内存超卖。
*   其次，它缓解了容器运行时中的自旋锁争用，从而降低了每个沙箱的 CPU 开销，显著提高了每主机的部署密度。

#### 轨迹日志记录与防抢占恢复
DSec 为每个沙箱维护一个全局有序的轨迹日志，持久化记录每一个命令调用及其结果。该轨迹有三个用途：
1.  **客户端快进**：当训练任务被抢占时，沙箱资源会被保留；恢复时，DSec 会重放之前已完成命令的缓存结果，这不仅加速了任务恢复，还防止了因重新执行非幂等操作而产生的错误；
2.  **细粒度溯源**：每个状态变更的来源和相应结果都是可追踪的；
3.  **确定性重放**：任何历史会话都可以从其轨迹中被忠实地复现。