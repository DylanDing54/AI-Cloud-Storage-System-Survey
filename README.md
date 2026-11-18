# KV Cache Management Strategies of CSPs — Survey

A survey of how cloud service providers (CSPs), inference engines and storage vendors manage the LLM KV cache across memory and storage tiers (HBM → DRAM → SSD → remote pool → HDD), followed by a feasibility analysis of using HDD as the bottom tier.

**Read online:** https://dylanding54.github.io/KV-Cache-Survey/report/

## Repository layout

| Folder | Contents |
|---|---|
| [`report/`](report/) | The report website in a docs-site layout: a [home page](https://dylanding54.github.io/KV-Cache-Survey/report/) with quick links, and tabs for [Part 1 · CSP and vendor survey](https://dylanding54.github.io/KV-Cache-Survey/report/part1.html), [Part 2 · AI inference engines](https://dylanding54.github.io/KV-Cache-Survey/report/part2.html) and [Part 3 · Big-data I/O management](https://dylanding54.github.io/KV-Cache-Survey/report/part3.html). Also includes a PDF edition with a table of contents and bookmark outline ([`KV_Cache_CSP_Survey.pdf`](report/KV_Cache_CSP_Survey.pdf)). Last updated 2026-11-18. |
| [`knowledge-base/`](knowledge-base/) | The same material as Markdown: one file per chapter, topic notes, extracted figures and a consolidated source index. |

## Coverage

- **Part 1 · CSP and vendor survey:** AWS, Microsoft Azure, Google Cloud, Alibaba Cloud, Tencent Cloud, Baidu, CoreWeave; LMCache / Tensormesh, Mooncake (Moonshot AI), NVIDIA Dynamo; Seagate / SK hynix, Western Digital / Solidigm, AMD MEXT, Huawei, Meta TMO
- **Part 2 · AI inference engines:** SGLang, vLLM and DeepInfer (DeepSeek) — how each implements KV cache management (with block diagrams drawn from pinned source commits) and whether HDD can serve as the bottom tier; enterprise research on long-term KV retention
- **Part 3 · Big-data I/O management:** to be added
- Each part ends with a consolidated list of reference links.

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

