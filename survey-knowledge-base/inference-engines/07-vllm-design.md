# vLLM：HDD 接入路线与改动范围

> 推理引擎 · HDD 可行性分析

### HDD 特有瓶颈：对象数、随机读与写入吸收

**chunk 大小与对象数。**fs 层每个文件的大小等于单个 rank 的一个 chunk。vLLM 博客给出的 16-token 物理块在新内存布局下约为 0.5–2 MB [VL5]。以 WD / Solidigm 报告中 70B 模型每 token 320 KiB KV 估算：一个 64-token 块约 20 MiB，TP=8 时平均每个 rank 的文件约 2.5 MiB（本文估算，按 KV 在各 rank 间均分） [VL18]。TP 越大，单个文件越小，文件数越多；`canonical_layout` 可以让 MLA 等副本相同的 KV 只存一份 [VL3]。调大 `blocks_per_chunk` 可以减少对象数和元数据操作，代价是查找粒度变粗 [VL3]。每个 chunk 落盘时都要经过“检查是否存在、建目录、创建临时文件、写入、rename”这一串元数据操作，对 HDD 来说是 create-heavy 的写流。WD / Solidigm 报告测得：该阵列上一次持久化的 1 MiB 写入约需 13 ms，理想提交粒度下 create-heavy 持久写的上限约为 692 MB/s，提交粒度不同，持久写吞吐可相差约 9 倍 [VL18]。

**随机读与等待。**文件按哈希前缀分散到 `<hhh>/<hh>_g<group>` 目录。一条前缀在逻辑上是连续的，落到盘上却是一组相互独立的小文件，冷恢复因此表现为随机读。promotion 的数据要先读进 CPU 主层，再由 DMA 搬到 GPU。等待期间请求持续拿到 `None`，没有超时机制把它切换回重算。HDD 读得慢，代价会直接体现在该请求的 TTFT 上。这与 SGLang HiCache 提供 timeout 预取策略不同，需要在 vLLM 之上自行实现“预计读取比 prefill 慢就放弃加载”的判断（可以借助 `max_load_tokens`）。

**写入吸收。**“存入主层即级联到所有二级层”意味着 HDD 要承接全部卸载写流。`offload_prompt_only` 默认只卸载 prompt 块，能减少一部分写入 [VL3][VL10]。fs 层只有在 store 任务成功完成后才把键标记为存在，查找时也以文件是否存在为准，因此还没写完的 chunk 一律算 MISS [VL9]。WD / Solidigm 报告（vLLM 0.22.0 + LMCache 0.4.6，NFS 远端 HDD）观察到的正是这种现象：没有 DRAM 前置层时，同步（持久）导出的 HDD 在并发 32 下外部命中率只有 19.0%，上限为 71.7%，decode 为 34.7 tokens/s。报告的解释是“chunk 在被再次查询时尚未落盘”，并强调这不是读延迟造成的。前置 30 GiB DRAM 后，各容量层在这些测试点上的结果无法区分 [VL18]。需要说明，该报告用的是 LMCache 路径，不是本章所述的 vLLM 原生 fs 层，只能作为机制上的对照。

**持久化与重启。**vLLM 原生 fs 层的写路径没有 fsync；O_DIRECT 只是绕过页缓存，不等于掉电持久。掉电后可能留下短文件，读到短文件时实现会删除该文件并按 miss 重算，这为“重算兜底”提供了一定的自愈能力 [VL9]。另一方面，`reset_cache()` 刻意不清空 FS / 网络这类持久二级层，所以 HDD 上的 KV 能跨重启保留并复用 [VL10]。代价是 fs 层没有淘汰机制：`SecondaryTierManager` 的接口约定由层自身负责“按需淘汰”，fs 实现却没有这部分逻辑，长期运行必须依赖外部的容量监控和清理 [VL10][VL9]。此外，二级层目前不支持 worker 跨多节点的引擎 [VL3]。

### 从挂载点原型到受控写回

第一步保留原生 fs 层，固定块布局，只切换 SSD 和 HDD 挂载点。第二步再改变每个 chunk 的块数，记录减少文件操作换来的收益，以及短前缀命中时多读的字节数。如果压力来自写入，先限制待写队列并观察丢写后的重算成本。只有这些手段仍无法满足尾延迟目标时，再评估合并小文件、维护偏移索引和独立容量回收的工程成本。

## 本章资料

- **[VL3]** [vLLM 官方文档 · KV Offloading Usage Guide（OffloadingConnector、TieringOffloadingSpec、fs / obj / p2p 二级层、On-Disk Layout、max_load_tokens，采集于 2026-11-18）](https://docs.vllm.ai/en/latest/features/kv_offloading_usage/)
- **[VL5]** [vLLM 官方博客 · Inside vLLM's New KV Offloading Connector: Smarter Memory Transfer for Maximizing Inference Throughput（Or Ozeri、Danny Harnik，IBM Research），2026-01-08](https://vllm.ai/blog/2026-01-08-kv-offloading-connector)
- **[VL9]** [vLLM 固定提交源码 · vllm/v1/kv_offload/tiering/fs/manager.py 与 io.py（FileSystemTierManager、batch_store_block / batch_load_block、probe_o_direct）](https://github.com/vllm-project/vllm/blob/0c80c3a6aa0201d1b4f71915f90c03fd64837cec/vllm/v1/kv_offload/tiering/fs/manager.py)
- **[VL10]** [vLLM 固定提交源码 · vllm/v1/kv_offload/tiering/manager.py（TieringOffloadingManager：级联、promotion、lookup、reset_cache）与 kv_connector/v1/offloading/scheduler.py](https://github.com/vllm-project/vllm/blob/0c80c3a6aa0201d1b4f71915f90c03fd64837cec/vllm/v1/kv_offload/tiering/manager.py)
- **[VL18]** [WD × Solidigm · KV Cache Offload Across Storage Tiers，September 2026 / Revision 02](https://documents.westerndigital.com/content/dam/doc-library/en_us/assets/public/western-digital/collateral/tech-brief/technical-brief-kv-cache-offload-across-storage-tiers-wd.pdf)
