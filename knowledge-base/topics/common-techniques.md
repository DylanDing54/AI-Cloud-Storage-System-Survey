# 各方案的共性技术点

> 跨章节归纳：CSP 托管 API、推理引擎、存储厂商与内存软件。属于综合分析，不是任何一家的原文。

### 1. 以前缀为单位复用，按块 / 页管理

- **托管 API**：AWS、Azure、Google、阿里百炼都只复用请求开头完全相同的连续前缀，并设有最低长度门槛（如 1024 tokens）。见 [AWS](../vendors/01-aws.md)、[Azure](../vendors/03-microsoft.md)、[Google](../vendors/04-google.md)、[阿里云](../vendors/05-alibaba.md)。
- **推理引擎**：LMCache 按 chunk、SGLang 按 page、Mooncake / FlexKV 按 block 管理，用 token 序列的链式哈希或 RadixTree 做索引。
- **共同约束**：一块 KV 只有在它前面所有块都命中时才有用，所以淘汰和下沉从前缀尾部（叶子）开始。

### 2. 多级存储：HBM → DRAM → SSD → 远端池 / 对象存储 → HDD

| 方案 | 层级 |
|---|---|
| [FlexKV](../vendors/06-tencent.md) | GPU → CPU → SSD → 远端扩展 |
| [Mooncake](../vendors/09-moonshot.md) | DRAM + SSD 组成分布式池 |
| [Dynamo KVBM](../vendors/10-nvidia.md) | HBM → DRAM → 共享存储（SSD 缓存 + HDD） |
| [WD / Solidigm](../vendors/12-wd.md) | DRAM 缓冲 + HDD / QLC 容量层 |
| [AMD MEXT](../vendors/13-amd.md)、[Meta TMO](../vendors/15-meta.md) | DRAM ↔ NVMe flash（通用内存页，非 KV 专用，思路相同） |

### 3. 高层做缓冲：吸收写入并服务热数据

| 方案 | 缓冲层 | 作用 |
|---|---|---|
| WD | 30 GiB 主机 DRAM | 吸收约 0.7 GB/s 的 GPU 驱逐流 |
| Seagate | SSD write-back（Open CAS） | 新 KV 先写 SSD，后台写回 HDD |
| Ceph / NFS async | 服务器内存或 SSD | 写回缓冲 |

共识：慢介质不能直接承接 GPU 的同步写入流，前面必须有一层快介质。

### 4. 异步传输，与计算重叠

LMCache 异步 save / load 并按层流水；FlexKV 异步 get / put；Mooncake Transfer Engine 走 RDMA；NVIDIA NIXL 走 GDS / RDMA。目的都是把 I/O 藏在计算后面。

### 5. 预取：在需要之前把数据提上来

- AMD MEXT：AI 模型预测即将访问的页，提前搬回 DRAM。
- SGLang HiCache：timeout / best_effort / wait_complete 预取策略。
- Seagate：会话恢复时 HDD → SSD → GPU；Dynamo prefetch hint 列为未来工作。
- Mooncake：调度器选实例时计入远端 KV 的传输代价。

### 6. "读回还是重算"的权衡与兜底

所有引擎都保留重算路径：未命中、超时或部分命中时，加载已有前缀，其余部分重新 prefill。只有读回比重算快时下沉才有意义：

> 读回时间 = 排队 + 寻道 + KV 大小 ÷ 有效带宽　＜　重新 prefill 的时间

### 7. 生命周期：TTL 与淘汰

托管 API 用 TTL（AWS 5 分钟且命中重置；Google 默认 60 分钟；Azure 扩展保留最长 24 小时）；引擎层用 LRU + 引用计数，正在使用的块不被淘汰。

### 8. 跨实例共享与 PD 分离

Mooncake Store、FastDeploy Global Cache Pooling、LMCache P2P、Dynamo 都支持多实例共享 KV 池，prefill / decode 分离后 KV 通过共享池或 RDMA 传递；元数据（块在哪里）与数据本体分开传递（NATS、GMS 等）。

### 9. 反馈驱动的自适应控制

Meta TMO 用 PSI 压力调节卸载量；AMD MEXT 用预测命中率实时切换模型；WD 的结论是按驱逐速率和热召回集给缓冲层定容。

### 一句话总结

以前缀块为单位，在 HBM → DRAM → SSD → HDD 的多级结构中，高层做写缓冲和热数据层，低层提供便宜容量；用异步传输和预取把 I/O 藏在计算后面，并以"读回比重算快"为底线，读不及时就重算。
