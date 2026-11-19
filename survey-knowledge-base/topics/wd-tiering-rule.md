# WD / Solidigm：KV 分层规则与"性能影响不大"的含义

> 依据：WD × Solidigm 技术简报 *KV Cache Offload Across Storage Tiers*（September 2026 / Revision 02，4 页）。原文：<https://documents.westerndigital.com/content/dam/doc-library/en_us/assets/public/western-digital/collateral/tech-brief/technical-brief-kv-cache-offload-across-storage-tiers-wd.pdf>。完整章节见 [12-wd.md](../vendors/12-wd.md)。

## 1. 简报给出的是架构级分层规则，不是逐块放置算法

简报第 1 页："This study's conclusion is a tiering rule, and the measurements size it." 逐块的放置、淘汰与召回由 LMCache 0.4.6 执行，简报未公开其配置（chunk 大小、淘汰策略、写入顺序）。

| 层级 | 放什么 | 作用 |
|---|---|---|
| GPU HBM | 正在使用的 KV | 推理计算 |
| 缓冲层（30 GiB 主机 DRAM；DRAM 不够时用 QLC） | GPU 刚驱逐的 KV + "hot recall set"（很快会被召回的 KV） | 吸收写入、服务热召回 |
| 容量层（HDD 阵列或 QLC） | 缓冲层放不下的较冷 KV | 便宜容量 |

两条规划规则（第 4 页原文）：

1. **"Never put a synchronous write path from a GPU eviction stream onto spindles."**
2. **"Size the buffer to the eviction rate and the hot recall set rather than to the capacity tier behind it."**

## 2. 为什么：瓶颈是写入吸收，不是读取

| 指标 | 数值 |
|---|---|
| 每 token KV（70B，BF16） | 320 KiB |
| 驱逐流（8 卡主机，并发 ≥ 32） | 约 0.7 GB/s，持续 |
| 24 块 HDD 单独顺序读 | 6.34 GB/s |
| 24 块 HDD 并发多流读 | 2.2–2.9 GB/s |
| 单次持久化 1 MiB 写入 | 约 13 ms |
| 持久化写入上限（理想提交粒度） | 约 692 MB/s |

无缓冲（B-2）、并发 32：同步写 HDD 在测量窗口内只写入 10–18 GiB，命中率 19.0%（上限 71.7%），"The cache simply never contained the data"；异步写回 HDD 吸收 89 GiB、命中率 64.3%，但并发 64 时读写争用使平均 TTFT 达 33.58 s（QLC 为 16.02 s）；QLC 吸收 84.6 GiB 并保持 71.7%。

## 3. "性能影响不大"体现在哪里

B-1（30 GiB DRAM 缓冲在前）：

| 并发 | Decode | 平均 TTFT | 命中率 |
|---|---|---|---|
| 32 | 92.0–92.4 tokens/s | 4.13–4.17 s | 全部 71.7% |
| 64 | 90.6–91.1 tokens/s | 13.23–13.35 s | — |

- 三种后端（QLC、HDD async、HDD sync）之间最大差异 0.5 tokens/s、0.11 s TTFT。
- 本地 PCIe Gen5 NVMe 对照（A-1，并发 32）：91.9–92.5 tokens/s、4.17–4.20 s，与 HDD 结果重叠；跨批次漂移 1.3 tokens/s、0.10 s。
- 方法：每单元冷启动、逐字节相同回放、3 次重复、95% 置信区间、144 次运行零失败；结论为 "statistically indistinguishable"。

**前提**：第 4 页写明这些测试点的召回全部由 DRAM 服务。HDD 只承担写入与容量，不在读取关键路径上；结论不等于 HDD 冷读与 SSD 同速。

| 并发 64、无缓冲 | 命中率 | Decode | 平均 TTFT | P95 TTFT |
|---|---|---|---|---|
| QLC | 71.7% | 75.8 | 16.02 s | 22.4 s |
| HDD async | 61.9% | 50.6 | 33.58 s | 59.2 s |
| HDD sync | 17.2% | 35.2 | 41.36 s | 82.7 s |

## 4. 简报未覆盖的问题

- 逐块的准入、淘汰、晋升策略；"hot recall set" 如何识别。
- 缓冲层定容公式（只给出两个输入量，30 GiB 仅对这些测试点足够）。
- 闲置数小时后的 HDD 冷召回（每单元都是冷启动、约 10K token × 4 轮的短周期对话）。
- "QLC 缓冲 + HDD" 组合未实测，属设计建议。
