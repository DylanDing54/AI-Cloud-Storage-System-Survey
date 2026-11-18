# KV Cache Management Strategies of CSPs — Survey

A survey of how cloud service providers (CSPs), inference engines and storage vendors manage the LLM KV cache across memory and storage tiers (HBM → DRAM → SSD → remote pool → HDD), followed by a feasibility analysis of using HDD as the bottom tier.

**Read online:** https://dylanding54.github.io/KV-Cache-Survey/

## Repository layout

| Folder | Contents |
|---|---|
| [`report/`](report/) | The full report as a single self-contained HTML file (figures embedded). Published at [/report/](https://dylanding54.github.io/KV-Cache-Survey/report/). |
| [`knowledge-base/`](knowledge-base/) | The same material as Markdown: one file per chapter, topic notes, extracted figures and a consolidated source index. |

## Coverage

- **CSPs and managed APIs:** AWS, Microsoft Azure, Google Cloud, Alibaba Cloud, Tencent Cloud, Baidu, CoreWeave
- **Inference systems and KV cache software:** LMCache / Tensormesh, Mooncake (Moonshot AI), NVIDIA Dynamo, SGLang HiCache
- **Storage and memory vendors:** Seagate / SK hynix, Western Digital / Solidigm, AMD MEXT, Huawei, Meta TMO
- **Feasibility analysis:** HDD integration routes, required changes, long-term KV retention value, validation criteria

## Notes

- The report and knowledge base are written in Chinese.
- Sources were collected from official documentation, engineering blogs, repositories, white papers and papers between 2026-11-06 and 2026-11-18. Product parameters and prices may have changed since.
- Copyright of quoted material and figures remains with the original authors and organizations. This is an independent survey, not an official publication of any vendor.
