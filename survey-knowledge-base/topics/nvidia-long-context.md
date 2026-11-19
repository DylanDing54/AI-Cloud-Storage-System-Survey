# NVIDIA *You don't need long context*：完整 KV 是否值得长期保存

> 依据：Belcak et al.（NVIDIA），*You don't need long context*，arXiv:2611.05306v1，2026-11-03。<https://arxiv.org/abs/2611.05306v1>。报告对应章节见 [07-context-feasibility.md](../inference-engines/07-context-feasibility.md)。

## 1. 出发点：越靠后的 token 越贵

图 1（H100 按需 $3.85/GPU·h 折算，生成 100 万 token）：上下文 256K 相对 16K 的成本——GLM 5.2 为 3.6 倍、Kimi K2.6 为 5.6 倍、Qwen 3 Coder Next 为 7 倍。图 2（DGX Spark）：多个模型在长上下文下单 token 时延约为短上下文的 2 倍，显存占用也更大。

## 2. 三种上下文管理方式

| 方式 | 做法 |
|---|---|
| FHA（Full-History Accumulation） | 基线：每步动作与观测都追加进同一对话，从不删除 |
| EoCC（End-of-Conversation Compaction） | 超过阈值时让模型生成有界的状态摘要，开新对话继续 |
| SAC（Single-Action Compaction） | 每个对话只执行一个动作，更新紧凑的语义状态后丢弃对话 |

长文档输入不直接放入上下文，而是存成文件，由 agent 用 `sed` / `head` 等工具按需读取，把长输入任务转化为长时程任务。

## 3. 主要结果

- **EoCC（长时程任务）**：22 个阈值配置中，每个模型最多 5 个比基线低 2 个百分点以上；估算成本下降 Opus 4.5 为 2.2–8.8 倍、GPT 5.6 Sol 为 2.4–5.0 倍。
- **EoCC + 按需读取（长输入任务）**：准确率多数持平或提升；成本下降 Opus 4.5 为 6.5–54.6 倍、GPT 5.6 Sol 为 4.4–233.3 倍。
- **SAC（表 2）**：

| 模型 | 基准 | FHA 基线 | SAC | 峰值上下文中位数（K，FHA → SAC） |
|---|---|---|---|---|
| Claude Opus 4.5 | SWE-bench Verified | 70.0% | 70.2% | 32 → 8 |
| Claude Opus 4.5 | Terminal-Bench 2.1 | 47.2% | 51.8% | 27 → 9 |
| GPT 5.6 Sol | SWE-bench Verified | 84.4% | 82.4% | 18 → 7 |
| GPT 5.6 Sol | Terminal-Bench 2.1 | 87.6% | 84.3% | 17 → 8 |

峰值上下文最大值从 130–200K 降到 15–37K。

- **反例**：Qwen 3 Coder Next 在 SWE-bench Verified 上用 EoCC（16K / 12K）时准确率从 53.0% 降到 3.6%，结论不能外推到所有模型。
- **展望**：提出 Semantic State Space Model（SSSM）——训练模型在多轮之间只传递可读的自然语言状态。

## 4. 对 HDD 长期保存 KV 的含义（分析）

1. 保存完整 KV 只省掉重复 prefill，省不掉长上下文 decode 变慢、变贵的成本。
2. 语义压缩会改变后续上下文和 KV 前缀，旧的完整 KV 多数无法再按前缀直接复用，长期保存价值下降；需分别测量任务质量、未来复用率与全生命周期成本。
3. 图 1 的"最高约 7 倍"是特定上下文长度下的 decode 成本口径，不是 HDD 读取成本或账单降幅；压缩有质量风险，也不能据此认定旧 KV 都可丢弃。
