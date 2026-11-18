# Knowledge Base

Markdown edition of every chapter in the report, plus the figures and a consolidated source index, for search, reuse and citation. Chapter text is in Chinese; each chapter ends with its source list.

## Part 1 · CSP and vendor survey (`vendors/`)

| # | Chapter | Scope |
|---|---|---|
| 01 | [AWS / Amazon Bedrock](vendors/01-aws.md) | Bedrock prompt caching, SageMaker HyperPod tiered KV cache, FSx/S3 offload |
| 02 | [Tensormesh / LMCache 生态](vendors/02-lmcache.md) | Open-source KV cache layer: architecture, storage backends, P2P sharing, PD disaggregation |
| 03 | [Microsoft / Azure](vendors/03-microsoft.md) | Azure OpenAI / Foundry prompt caching, retention, pricing; Splitwise; Blob offload |
| 04 | [Google Cloud / Gemini](vendors/04-google.md) | Vertex AI / Gemini context caching, GKE Inference Gateway, llm-d tiered KV, Managed Lustre |
| 05 | [阿里云 / 百炼与 DashInfer](vendors/05-alibaba.md) | Bailian context cache, DashInfer, Tair KVCache, SGLang HiCache in production |
| 06 | [腾讯云 / TACO FlexKV](vendors/06-tencent.md) | TACO FlexKV multi-tier cache (GPU/CPU/SSD/remote), TACO-LLM prefix offload |
| 07 | [百度 / PaddlePaddle FastDeploy](vendors/07-baidu.md) | FastDeploy global cache pooling, AttentionStore, Qianfan prefix cache |
| 08 | [CoreWeave / Cohere](vendors/08-coreweave.md) | Cohere x CoreWeave x LMCache case on AI Object Storage / LOTA |
| 09 | [Moonshot AI / Kimi](vendors/09-moonshot.md) | Mooncake: KVCache-centric disaggregated serving, Transfer Engine, Store, SSD offload |
| 10 | [NVIDIA / Dynamo](vendors/10-nvidia.md) | Dynamo / KVBM / NIXL in the Seagate-SK hynix hybrid storage design |
| 11 | [Seagate / SK hynix](vendors/11-seagate.md) | SSD + HDD hybrid KV cache white paper: topology, test method, results |
| 12 | [Western Digital / Solidigm](vendors/12-wd.md) | Controlled benchmark: DRAM buffer in front of HDD / QLC capacity tiers |
| 13 | [AMD / MEXT](vendors/13-amd.md) | MEXT Predictive Memory: AI-driven DRAM / NVMe page tiering (official sources) |
| 14 | [华为 / KVU](vendors/14-huawei.md) | UCM, KVU compute-storage module, OceanStor, CloudMatrix384 EMS |
| 15 | [Meta](vendors/15-meta.md) | Transparent Memory Offloading (PSI + Senpai); LLM inference parallelism |

## Part 2 · AI inference engines (`inference-engines/`)

| # | Section | Chapter | Scope |
|---|---|---|---|
| 01 | 2.1 SGLang | [SGLang：KV 缓存管理策略实现](inference-engines/01-sglang-impl.md) | SGLang KV management implementation: RadixCache, memory pools, request lifecycle, HiCache tiers (with block diagrams) |
| 02 | 2.1 SGLang | [SGLang：最底层接入 HDD](inference-engines/02-sglang.md) | Interface feasibility: HiCache L3 file backend on HDD (source review) |
| 03 | 2.1 SGLang | [HDD 接入路线与改动范围](inference-engines/03-hdd-design.md) | Engineering routes and required changes for an HDD tier |
| 04 | 2.1 SGLang | [可行性验证与判定标准](inference-engines/04-validation.md) | Validation plan and pass criteria |
| 05 | 2.2 vLLM | [vLLM：KV 卸载与接入 HDD](inference-engines/05-vllm.md) | vLLM prefix cache, KV connector API, offloading connector fs tier and LMCache disk paths to HDD (with block diagrams) |
| 06 | 2.3 DeepInfer (DeepSeek) | [DeepInfer：DeepSeek 推理系统的 KV 缓存管理与硬盘缓存](inference-engines/06-deepseek.md) | DeepSeek's inference stack: MLA KV size, 3FS on-SSD context cache, PD disaggregation, HDD relevance (with block diagrams) |
| 07 | 2.4 Enterprise research | [NVIDIA：完整 KV 是否值得长期保存](inference-engines/07-context-feasibility.md) | Enterprise research: is full long-context KV worth keeping? (NVIDIA) |

## Part 3 · Big-data I/O management (`bigdata-io/`)

| # | Chapter | Scope |
|---|---|---|
| 01 | [内容待补充](bigdata-io/01-bigdata-io.md) | Placeholder — to be added |

## Topic notes (`topics/`)

| Note | Scope |
|---|---|
| [common-techniques.md](topics/common-techniques.md) | Techniques shared across CSPs, inference engines and storage vendors |
| [wd-tiering-rule.md](topics/wd-tiering-rule.md) | WD / Solidigm tiering rule, buffer sizing, and what "no performance impact" actually means |
| [nvidia-long-context.md](topics/nvidia-long-context.md) | NVIDIA *You don't need long context* and its implications for long-term KV retention |

## Other files

- [sources.md](sources.md) — all 231 cited sources with keys and URLs
- [figures/](figures/) — figures extracted from the report (official diagrams, document screenshots, PDF page renders, block diagrams drawn from source code)
