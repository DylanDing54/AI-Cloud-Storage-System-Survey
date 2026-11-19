# Moonshot AI / Kimi

> 服务平台 · 开源项目 · FAST'25 最佳论文 · 官方文档

Mooncake 是 Moonshot AI 旗下 Kimi 的推理服务平台，后以 kvcache-ai/Mooncake 仓库开源。官方 README 将其概括为“以 KV Cache 为中心的分离式架构”：把 prefill 与 decode 拆成两个集群，同时把 GPU 集群里闲置的 CPU、DRAM、SSD 资源聚合成分离式 KV Cache 池。 [M] 它不是传统意义上的 IaaS 云厂商产品，而是一家大模型服务商把自身生产架构写成论文、拆成组件开源的案例。论文最早以技术报告形式于 2024-06 发布在 arXiv，扩展版发表于 FAST 2025（第 23 届 USENIX 文件与存储技术会议），README 记载其于 2025-02-25 获得该会议最佳论文奖。 [MP][MC1][M]

### 定位：用更多存储换更少计算

FAST'25 论文的标题是“Trading More Storage for Less Computation”。作者先给出复用前缀 KV 的收益条件：若请求长度为 n、命中前缀长度为 p，复用能省掉约 l×(ap²d+bpd²) 的 prefill 计算，但需要把 p×l×(2×d/gqa)×s 字节的 KV 搬进 GPU HBM；只有当加载带宽 B 与 GPU 算力 G 之比超过一个与 d、p 相关的阈值时，复用才对 TTFT 有利。按论文表 1 的 LLaMA3-70B 参数，在 8×A800 上、前缀长 8192 时，所需的最低 B 约为 6 GB/s，换成 8×H800 则升到 19 GB/s；由于各传输阶段不可能完全重叠，实际要求更高。 [MC1]

第二个论据是容量。LLaMA3-70B 每个 token 的 KV 约 320 KB，单机即使预留约 1 TB DRAM 也只能放约 300 万 token；在论文的三类负载中，3M token 的本地缓存在多数情况下达不到理论最大命中率的 50%，而约 5000 万 token 的容量才接近理论最大值，这至少需要汇聚 20 个节点的 DRAM。这就是 Mooncake 坚持“全局”而非“本地”缓存的原因。 [MC1]

![FAST'25 论文图 9：缓存容量（token）与前缀命中率的关系；虚线为单机约 3M token 的本地缓存容量（PDF 第 12 页渲染）](../figures/moonshot-1.png)

*FAST'25 论文图 9：缓存容量（token）与前缀命中率的关系；虚线为单机约 3M token 的本地缓存容量（PDF 第 12 页渲染）* — [原始来源](https://www.usenix.org/system/files/fast25-qin.pdf)

### 论文架构：Conductor、Prefill/Decode 池与 Mooncake Store

论文图 2 把系统分成三部分：左侧的全局调度器 Conductor（含 cache-aware prefill 调度、KV Cache 均衡调度、负载均衡 decode 调度）；右侧的 Prefill 池与 Decoding 池，每个实例的 GPU/VRAM 中保留分页 KV Cache；以及位于两池之间、由各节点 CPU/DRAM/SSD 组成并经 RDMA 互联的分布式 KV Cache 池，即 Mooncake Store。Prefill 阶段的目标是在 TTFT SLO、MFU 下限和“KV Cache 小于 DRAM”的约束下最大化缓存复用；Decode 阶段的目标是在 TBT SLO 和“KV Cache 小于 VRAM”约束下最大化吞吐。 [MC1]

![FAST'25 论文图 2：Mooncake 架构——Conductor、Prefill/Decoding 池与基于 CPU/DRAM/SSD 的分布式 KV Cache 池（PDF 第 3 页渲染）](../figures/moonshot-2.png)

*FAST'25 论文图 2：Mooncake 架构——Conductor、Prefill/Decoding 池与基于 CPU/DRAM/SSD 的分布式 KV Cache 池（PDF 第 3 页渲染）* — [原始来源](https://www.usenix.org/system/files/fast25-qin.pdf)

一次请求的典型流程分四步：（1）KV Cache 复用：被选中的 prefill 节点拿到原始输入、可复用前缀块的 key 和为整个请求分配的块 key，从远端 CPU 内存把前缀 KV 加载进 GPU；（2）增量 prefill：计算未命中的部分，并把新生成的增量 KV 写回 CPU 内存，未命中 token 超过阈值（通常大于 1000 token）时分块流水执行；（3）KV Cache 传输：与增量 prefill 重叠，逐层把 KV 流式送到目标 decode 节点的 CPU 内存；（4）Decoding：KV 全部到达后，请求以 continuous batching 方式加入下一批。 [MC1]

对于长上下文，论文没有采用跨节点张量并行或序列并行，而是提出分块流水并行（CPP）：每 X 个 prefill 节点组成一个流水组，同一请求的不同 chunk 由不同节点同时处理，只在流水级边界通信，从而减少与 KV 传输争抢网络。 [MC1]

### 关键机制：块管理、淘汰、传输与调度

**块与淘汰。**论文中，Mooncake Store 以分页块保存所有 KV，块大小按模型规模和最佳网络传输粒度确定，通常为 16–512 token（评测设为 256）；每个块的 hash key 由自身内容和前缀共同决定，用于去重；同一 key 可在不同节点有多个副本以缓解热点。缓存池满时采用 LRU 淘汰，但正在被请求访问的块不淘汰。 [MC1] 开源版的淘汰机制更细：Master 在空间使用达到高水位（默认 90%）或 PutStart 分配失败时触发淘汰，按比例（默认 5%）回收对象，并借助按租约截止时间分桶的索引优先淘汰租约已过期的对象；硬 pin 对象和活跃租约受保护，软 pin 可配置。 [MC3]

**传输引擎。**Transfer Engine 对上提供 Segment（可远程访问的地址空间，可在 DRAM 或加速器显存中；NVMe-oF 存储作为持久 Segment 暴露）和 BatchTransfer（批量异步读写）两类抽象；控制面（etcd/Redis/HTTP 元数据或 P2P 握手）只交换位置信息，不承载数据。 [MC2] 论文披露的实现要点包括：按 NUMA/PCIe 拓扑生成“首选/备选”网卡矩阵，单次请求按 16 KB 切片分散到多张 RDMA 网卡，端点池用 SIEVE 算法淘汰连接，链路故障时自动换路重传。 [MC1] 官方协议文档列出 tcp、rdma、efa 为主要传输，nvmeof、nvlink、hip、cxl、ascend 等为高级选项；检测不到 RDMA 网卡时自动回落到 TCP。 [MC6]

**KV 感知调度。**论文算法 1 对每个 prefill 实例估算 TTFT＝远端 KV 传输时间＋排队时间＋剩余 prefill 执行时间（执行时间由离线数据拟合的多项式回归模型预测），选 TTFT 最小者；若最佳远端前缀比本地前缀长出一定倍数（kvcache_balancing_threshold，目前人工设定），就先把 KV 拉到目标实例，从而自动复制热点块。若 TTFT 或 TBT 预计无法满足 SLO，Conductor 直接返回 HTTP 429。 [MC1] arXiv 技术报告还讨论了过载场景下的“早拒绝”及“基于预测的早拒绝”：在 prefill 开始前就同时评估 decode 池负载，并预测 prefill 完成后的 decode 负载，以避免 prefill 与 decode 负载反相振荡。 [MP]

### 开源组件与 SSD / 对象存储下沉

![Mooncake 官方 README 组件总览原图（Transfer Engine、Mooncake Store、EP/PG 等，范围比本文关注的 KV 管理更广）](../figures/moonshot-3.png)

*Mooncake 官方 README 组件总览原图（Transfer Engine、Mooncake Store、EP/PG 等，范围比本文关注的 KV 管理更广）* — [原始来源](https://github.com/kvcache-ai/Mooncake)

开源仓库的组件与论文并非一一对应。按官方架构文档：Transfer Engine 是嵌入数据所属进程的传输库，可单独用于 PD 传输；Mooncake Store 由 Master Service（控制面，负责对象目录、分配与淘汰、副本、租约、租户配额，不转发数据）和 Client（直接经 Transfer Engine 搬运切片）组成；开源版的 Conductor 是供网关/路由器查询的全局前缀索引服务，订阅 vLLM、SGLang 与 Store master 的 KV 事件，按 device、cpu_local、cpu_share、disk 四个层级记录前缀所在位置，并不等同于论文中完整的调度器。 [MC2][MC7] 另有 P2P Store 用于检查点等对象的点对点分发，文档称其已用于 Moonshot AI 的检查点传输服务。 [MC10]

SSD 下沉由 Store 的 real client 后台子系统完成，对应用透明：心跳线程定期向 Master 领取待从内存淘汰的对象，读出后写入本地 SSD，成功后 Master 为对象增加一个 LOCAL_DISK 副本；Get 找不到内存副本时自动回落到 SSD——持有该盘的客户端把数据读入预注册的 O_DIRECT 对齐暂存区，再由请求方经 Transfer Engine（RDMA 或 TCP）拉取。默认的 BucketStorageBackend 把对象攒成桶后写盘，单桶上限默认 256 MB 或 500 个 key；盘上淘汰支持 FIFO/LRU 和高低水位，另有逻辑容量与物理容量两种上限。 [MC4] 可选的“存在即预取”功能会在 is_exist 查询时尽力把仅在 SSD 上的对象提升回 DRAM，默认关闭。 [MC8] 同一 FileStorage 路径还可接 NVMe KV 命名空间以及阿里云 OSS、S3 兼容对象存储（含 MinIO、Ceph RGW 等）。 [MC9]

生态集成是开源 Mooncake 被广泛引用的主要原因。README 的集成目录显示：SGLang 同时使用 Transfer Engine 做 PD/EPD 传输，并把 Mooncake Store 作为 HiCache 的外部存储层，在设备、主机、远端三级上扩展 RadixAttention；vLLM 通过 MooncakeConnector 做 PD 间 KV 传输，通过 MooncakeStoreConnector 让多个实例基于块哈希前缀共享集群级 KV 池；LMCache 将 Mooncake Store 作为远端存储及多进程模式下的 L2 适配器；TensorRT-LLM、LMDeploy、vLLM-Ascend、FastDeploy、FlexKV、NIXL、Dynamo、llm-d 等也各自接入了 Transfer Engine 或 Store。README 同时提醒，具体能力取决于上游版本与配置，部署时应以各项目的链接文档为准。 [M] 换言之，开源 Mooncake 更像一套可被多种推理引擎复用的“KV 传输 + 共享池”基础设施，而调度策略多由上游引擎或路由层自行实现。

### 公开性能数据与条件

Mooncake 不同材料中的数字口径差异很大，引用时必须带上条件：

| 数字 | 来源 | 条件 |
|---|---|---|
| 多处理 115%（A800）/ 107%（H800）请求 | FAST'25 摘要与第 5 节 | Kimi 历史统计，对比此前基于 vLLM 的自有系统；非受控实验 |
| 有效请求容量 +59%～+498% | FAST'25 图 1 | 真实对话 trace，16 台 8×A800，LLaMA3-70B 结构的 dummy 模型；TTFT 阈值 30 s，TBT 阈值 100/200/300 ms 分别对应 +498%/+157%/+59% |
| 吞吐提升最高 525% | arXiv v1 摘要与 8.1.2 节 | 模拟数据（16k–128k 输入、50% 前缀缓存），Mooncake-[3P+1D]/[2P+2D] 对比 vLLM-[4M]，范围 50%～525%；LLaMA2-70B 结构 dummy 模型 |
| 多处理约 75% 请求 | arXiv v1 8.1.3 节 | 23,000 条真实请求重放，10P+10D 对比 20 个 vLLM 实例；TTFT≤30 s、TBT≤0.1 s；vLLM 仅约 57% 请求满足 TBT |
| 全局缓存命中率最高为本地缓存的 2.36×，prefill 计算时间最多降 48% | FAST'25 图 10 | 10 个 prefill 节点、每节点 3M token 容量、输出长度限制为 1 |
| 平均 TTFT 3.07 s（全局缓存感知）vs 3.58 s（本地）vs 5.27 s（负载均衡）vs 19.65 s（随机） | FAST'25 图 5 | 16 台 8×A800，对话 trace |
| 87 GB/s 与 190 GB/s，约为 TCP 的 2.4× 与 4.6× | FAST'25 5.4.1 节、README | 传输 40 GB（约 LLaMA3-70B 128k token 的 KV），4×200 Gbps 与 8×400 Gbps RoCE，并发 64，最小粒度 128 KB |
| 128k 输入时 prefill 时间降 92%，计入开销后 TTFT 降 86% | FAST'25 5.4.3 节 | 前缀缓存比例 95% 对比 0% |

论文还建议网络总带宽不低于 100 Gbps：在合成负载下低于该值时 TTFT 急剧上升、实际传输时间显著偏离理论值；P/D 节点比约 1:1 时有效请求容量最高。 [MC1] 开源项目给出的 SSD 下沉基准（单台 8×A100-40GB、双 200 Gb/s IB 网卡、5 块 NVMe SSD 组 RAID0，Qwen3-8B，SGLang 多轮对话基准、每轮输入 4096、输出 1 token）显示：开启 SSD 后平均 TTFT 比纯 GPU 降 57%、比不带 SSD 的 Mooncake 降 34%，输入吞吐约为纯 GPU 的 2.4 倍；第 7 轮起 80 GB 内存池装不下时，不带 SSD 的命中率从 83% 跌到 36%，带 SSD 的仍保持 84% 以上。 [MC5] README 另记载 Mooncake 支撑 Kimi K2 在 128 张 H200 上以 PD 分离部署，达到 224k token/s 的 prefill 和 288k token/s 的 decode 吞吐。 [M]

![Mooncake 官方文档 SSD Offload Benchmark 总体结果：GPU only、+L2、+Mooncake、+Mooncake+SSD 的平均 TTFT 与输入吞吐](../figures/moonshot-4.png)

*Mooncake 官方文档 SSD Offload Benchmark 总体结果：GPU only、+L2、+Mooncake、+Mooncake+SSD 的平均 TTFT 与输入吞吐* — [原始来源](https://kvcache-ai.github.io/Mooncake/performance/mooncake/ssd-offload-benchmark-results.html)

与官方基准一致的最小配置示例（Master 与 Client 开启下沉，SGLang 以 Mooncake 作 HiCache 存储后端）：

```
mooncake_master -enable_offload=true -http_metadata_server_port=8081
MOONCAKE_OFFLOAD_FILE_STORAGE_PATH=/mnt/data/file_storage \
MOONCAKE_OFFLOAD_LOCAL_BUFFER_SIZE_BYTES=21474836480 \
mooncake_client --global_segment_size=80GB --protocol=rdma ...
python3 -m sglang.launch_server ... --enable-hierarchical-cache \
  --hicache-storage-backend mooncake
```

[MC5]

### 与 SSD / HDD 多级存储的关系（本文分析）

Mooncake 的价值在于把“KV 放在哪一层”和“请求调度到哪里”放在一起决策：Conductor 的代价模型显式计入传输时间，Store 用副本和淘汰决定热数据留在 DRAM，SSD 作为 DRAM 之下的 LOCAL_DISK 副本层。若再往下加 HDD，有几点可直接从公开资料推出：第一，论文的收益条件给出了每台机器需要 6–19 GB/s 级别的有效加载带宽，这一量级远高于单块机械盘，HDD 不适合放在请求关键路径上直接被读；第二，现有 SSD 后端按 256 MB 级的桶聚合写入，写入模式偏向大块顺序，这对容量型介质友好，但桶内对象零散失效会造成空间回收滞后，文档也承认逻辑计数会落后于实际占用；第三，S3 兼容后端可接 Ceph RGW 等对象存储，理论上可把 HDD 集群作为最底层，但官方材料没有任何 HDD 介质的测试，也没有 SSD→HDD 的分层迁移策略。因此较稳妥的定位是：HDD 只承接冷前缀的长期保存，并依赖预取在请求到达前把数据提升到 SSD 或 DRAM。

### 资料边界

论文的受控实验均使用 dummy 模型和重放 trace，115%/107% 来自线上历史统计，二者不能互换。开源组件与 Kimi 生产内部实现的对应关系没有完整披露，开源 Conductor 也不是论文调度器的完整实现。SSD 基准为单机、小模型、单一负载，没有公开 HDD 或对象存储层的性能数据。

## 本章资料

- **[M]** [Mooncake](https://github.com/kvcache-ai/Mooncake)
- **[MC1]** [USENIX FAST'25 · Qin et al. · Mooncake: Trading More Storage for Less Computation — A KVCache-centric Architecture for Serving LLM Chatbot，2025-02（会议论文 PDF）](https://www.usenix.org/system/files/fast25-qin.pdf)
- **[MC2]** [Mooncake 官方文档 · Mooncake Architecture（采集于 2026-11-18）](https://kvcache-ai.github.io/Mooncake/design/architecture.html)
- **[MC3]** [Mooncake 官方文档 · Mooncake Store 设计（Eviction Policy 等，采集于 2026-11-18）](https://kvcache-ai.github.io/Mooncake/design/store/mooncake-store.html)
- **[MC4]** [Mooncake 官方文档 · SSD Offload Design（采集于 2026-11-18）](https://kvcache-ai.github.io/Mooncake/design/store/ssd-offload.html)
- **[MC5]** [Mooncake 官方文档 · Mooncake SSD Offload Benchmark（采集于 2026-11-18）](https://kvcache-ai.github.io/Mooncake/performance/mooncake/ssd-offload-benchmark-results.html)
- **[MC6]** [Mooncake 官方文档 · Supported Protocols（采集于 2026-11-18）](https://kvcache-ai.github.io/Mooncake/getting_started/supported-protocols.html)
- **[MC7]** [Mooncake 官方文档 · Conductor Architecture（采集于 2026-11-18）](https://kvcache-ai.github.io/Mooncake/design/conductor/conductor-architecture-design.html)
- **[MC8]** [Mooncake 官方文档 · SSD Prefetch-on-Exist（采集于 2026-11-18）](https://kvcache-ai.github.io/Mooncake/design/store/ssd-prefetch.html)
- **[MC9]** [Mooncake 官方文档 · Object Storage Offload（采集于 2026-11-18）](https://kvcache-ai.github.io/Mooncake/deployment/object-storage-offload.html)
- **[MC10]** [Mooncake 官方文档 · P2P Store（采集于 2026-11-18）](https://kvcache-ai.github.io/Mooncake/design/p2p-store.html)
- **[MP]** [Qin et al. · Mooncake: Kimi’s KVCache-centric Architecture for LLM Serving，2024，v1](https://arxiv.org/html/2407.00079v1)
