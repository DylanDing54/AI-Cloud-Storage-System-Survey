# KV Cache Management Strategies of CSPs — Survey

A survey of how cloud service providers (CSPs), inference engines and storage vendors manage the LLM KV cache across memory and storage tiers (HBM → DRAM → SSD → remote pool → HDD), followed by a feasibility analysis of using HDD as the bottom tier.

**Read online:** https://dylanding54.github.io/KV-Cache-Survey/report/

## Repository layout

| Folder | Contents |
|---|---|
| [`report/`](report/) | The full report as a self-contained HTML page ([read online](https://dylanding54.github.io/KV-Cache-Survey/report/)) and a PDF edition with a bookmark outline ([`KV_Cache_CSP_Survey_2026-11-18.pdf`](report/KV_Cache_CSP_Survey_2026-11-18.pdf)). Last updated 2026-11-18. |
| [`knowledge-base/`](knowledge-base/) | The same material as Markdown: one file per chapter, topic notes, extracted figures and a consolidated source index. |

## Coverage

- **CSPs and managed APIs:** AWS, Microsoft Azure, Google Cloud, Alibaba Cloud, Tencent Cloud, Baidu, CoreWeave
- **Inference systems and KV cache software:** LMCache / Tensormesh, Mooncake (Moonshot AI), NVIDIA Dynamo, SGLang HiCache
- **Storage and memory vendors:** Seagate / SK hynix, Western Digital / Solidigm, AMD MEXT, Huawei, Meta TMO
- **Feasibility analysis:** HDD integration routes, required changes, long-term KV retention value, validation criteria

## Notes

- The report and knowledge base are written in Chinese.

## Copyright

Copyright © 2026 Yifan Ding. All rights reserved.

- **Original content.** The report text, analysis, tables, topic notes and page design in this repository are the original work of the author. No license is granted to copy, redistribute, modify or use them commercially without prior written permission. Short quotations for commentary or research are welcome with attribution and a link to this repository.
- **Third-party material.** Quoted passages, figures, screenshots and data from vendor documentation, white papers, blogs and academic papers remain the property of their respective copyright holders. They are reproduced in limited excerpts for research, commentary and citation purposes, each with a link to the original source. No ownership is claimed.
- **Trademarks.** All product names, company names and logos are trademarks or registered trademarks of their respective owners. Their use is for identification only and does not imply endorsement.
- **No affiliation.** This is an independent survey. It is not an official publication of, and is not endorsed by, any vendor mentioned.
- **No warranty.** Content is provided "as is" for informational purposes, without warranty of accuracy or completeness.
- **Removal requests.** If you are a rights holder and want material removed, please open an issue in this repository.

