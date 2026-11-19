# DeepInfer：HDD 接入路线与改动范围

> 推理引擎 · HDD 可行性分析

**加一层 HDD 会带来什么、打破什么。**可能的收益有两点：一是更便宜的容量，可用来把 TTL 延长到 7 天以上，或承接 SSD 层淘汰的冷条目；二是机械盘没有闪存那样的擦写寿命约束，可以分担 write-back 写流。但这两点在官方数据中都很有限。官方称现有 TTL 已接近最大可达命中率，且 TTL 从 7 天缩到 3 天只损失 0.5%–1%，把保留期继续拉长的边际收益更小 [DS2]。代价则落在读路径上。推荐配置相当于每块 SSD 承担约 2 GB/s 读带宽（40 GB/s ÷ 20 块，本文估算）；单块机械盘即使顺序读也只有数百 MB/s 量级，而 KV 读取是细粒度的。一个 128-token 块约 111 KiB，若按 10+2 纠删切到 10 个数据节点，每个节点每次只读 10 KiB 量级（本文估算，假设一个块单独构成一个条带；kvcache-store 的实际文件与条带布局未公开），HDD 的寻道与旋转延迟会占主导。“任意 10 个返回即可解码”的尾延迟设计也是以 SSD 的低延迟为前提的。写侧可参照 WD / Solidigm 报告：其 HDD 阵列上一次持久化的 1 MiB 写入约需 13 ms；在没有 DRAM 前置层时，写入跟不上会直接拉低命中率 [VL18]。

### 冷层原型的读写路径

建议先在 SSD 淘汰时旁路写入 HDD，将模型版本、前缀键、有效长度和校验信息与冷层对象关联。对象写完并校验后才发布索引。读取时先查 SSD，对 HDD 命中设置等待上限，超时转入重算。冷层回收时先撤销索引，再释放盘上空间。实现这些步骤需要独立的目录、后台复制、限速与故障恢复逻辑；它们是拟议改动。

## 本章资料

- **[DS2]** [DeepSeek 官方 GitHub · DeepInfer doc/inference_deployment_qa.md《DeepSeek V4.1 Flash Inference Deployment Q&A》（v0.2，2026-11-03；KV cache storage、3FS、负载均衡、SWA bounded replay，采集于 2026-11-18）](https://github.com/deepseek-ai/DeepInfer/blob/a679bb86ce36e883a98a97733492b686e5b9a08e/doc/inference_deployment_qa.md)
- **[VL18]** [WD × Solidigm · KV Cache Offload Across Storage Tiers，September 2026 / Revision 02](https://documents.westerndigital.com/content/dam/doc-library/en_us/assets/public/western-digital/collateral/tech-brief/technical-brief-kv-cache-offload-across-storage-tiers-wd.pdf)
