---
name: Qwen
sources:
  - "[[articles/llm/qwen/Qwen（通义千问）开源模型迭代史：从Qwen到Qwen3.8的技术演进]]"
kind: family
vendor: 阿里巴巴通义实验室
open: true
tagline: 全尺寸、跨模态、广开源：从 0.5B 端侧模型到近 4000B 旗舰 MoE，覆盖文本、视觉、音频、代码、智能体的完整生态
---

## Qwen
- 发布:: 2023-08
- 技术:: [[Transformer]] 解码器结构
- 亮点:: 初代 Qwen 奠定了系列的基础架构，采用标准 Transformer 解码器结构。
- 突破:: 阿里正式进入大语言模型开源赛道，为后续快速迭代建立技术底座和开发者生态。
- 系列发布:: 2023-04（通义千问大语言模型系列发布）

## Qwen1.5
- 发布:: 2024
- 前代:: Qwen
- 亮点:: 优化训练数据规模和模型参数效率，提升多轮对话和长文本处理能力。
- 突破:: 为后续版本打下基础。
- 发布时段:: 2024年初

## Qwen2
- 发布:: 2024
- 上下文:: 128K
- 前代:: Qwen1.5
- 技术:: [[分组查询注意力GQA]]、SwiGLU 激活函数、RoPE 位置编码
- 亮点:: 支持超过 27 种语言，上下文窗口 128K tokens。
- 突破:: 使 Qwen 在中文理解和多语言场景中具备竞争力；引入 GQA 降低了 KV Cache 内存占用，为后续长上下文和高效推理铺平道路。
- 发布时段:: 2024年中
- 语言:: 27+

## Qwen2.5
- 发布:: 2024-12-19
- 参数:: 0.5B / 1.5B / 3B / 7B / 14B / 32B / 72B
- 训练数据:: 18T tokens
- 前代:: Qwen2
- 技术:: 预训练数据 7T→18T tokens、SFT + DPO（离线）+ GRPO（在线）多阶段强化学习、MoE 层替换 FFN（细粒度专家分割 + 共享专家路由）、统一 Token 词表（新增工具调用专用 token）
- 亮点:: Qwen2.5-72B-Instruct 在多项基准上超越参数规模约为其 5 倍的 Llama-3-405B-Instruct。
- 突破:: 证明数据质量与训练策略可以部分弥补参数规模的差距；生成长度从 2K 扩展到 8K，显著改善了结构化输入输出（表格、JSON）和工具使用能力。
- 论文:: arXiv:2412.15115
- MoE 变体:: Qwen2.5-Turbo / Qwen2.5-Plus
- 生成长度:: 8K
- 后训练数据:: 超100万样本
- 词表:: 151,643 个常规 token，控制 token 从 3 个扩展到 22 个

## Qwen3
- 发布:: 2025-04
- 参数:: 0.6B / 1.7B / 4B / 8B / 14B / 32B / 30B-A3B / 235B-A22B
- 激活:: 3B（30B-A3B）/ 22B（235B-A22B）
- 训练数据:: 36T tokens
- 前代:: Qwen2.5
- 技术:: 混合推理（快思考 / 慢思考同一模型）、MoE、原生支持 MCP 协议与复杂工具调用
- 亮点:: BFCL 智能体评估得分 70.8，领先 Gemini 2.5-Pro 和 OpenAI-o1；AIME25 与 LiveCodeBench 取得最优成绩。
- 突破:: 解决了「推理模型不能做 Agent」的痛点，让同一个模型既能深度推理，又能调用工具完成复杂任务。
- 论文:: arXiv:2505.09388
- 语言:: 100+
- 混合推理:: 国内首款把非推理模式与推理模式集成到同一模型

## Qwen3-Coder
- 发布:: 2025-07
- 亮点:: 编程能力登顶全球开源模型阵营；据 OpenRouter 统计，单周调用量增长 1474%，跃居编程类模型第二。
- 突破:: 代码能力与 Agent 调用能力取得重大突破。

## Qwen3-2507
- 发布:: 2025
- 上下文:: 256K（可扩展至1M）
- 前代:: Qwen3
- 基于:: Qwen3
- 技术:: Instruct-only 与 Thinking-only 模型分开发布
- 亮点:: 根据社区反馈做的专项优化版，256K 长上下文理解显著增强。
- 发布时段:: 2025年下半年

## Qwen3-Max-Thinking
- 发布:: 2026-01
- 参数:: 1T
- 训练数据:: 36T tokens
- 技术:: Test-time Scaling 多轮推理策略
- 亮点:: 总参数量超万亿，是阿里当时规模最大、能力最强的推理模型。

## Qwen3.5-Plus
- 发布:: 2026-02
- 参数:: 397B
- 激活:: 17B
- 上下文:: 262K（可扩展至1M）
- 技术:: Hybrid GatedDeltaNet 架构、原生多模态
- 亮点:: 性能媲美 Gemini 3 Pro，登顶全球最强开源模型。
- 所属代:: Qwen3.5（旗舰版）
- 语言:: 201

## Qwen3-Coder-Next
- 发布:: 2026-02
- 激活:: 3B
- 亮点:: 仅 3B 激活参数，SWE-Bench 达到 70.6%，编程能力与 Claude Sonnet 4.0 相当。
- 突破:: 在消费级硬件上就能运行达到这一水平的编程模型。

## Qwen3.8-Flash-Next
- 发布:: 2026-08
- 参数:: 125B
- 激活:: 6B
- 技术:: Gated DeltaNet + Qwen Sparse Attention（QSA）混合注意力、Gated Residual（4 条并行残差分支）、N-gram Embedding、Muon 优化器（与 AdamW 分工）
- 亮点:: 1M 上下文下 QSA 让 Prefill 和 Decode 分别最高加速 7.6× 和 4.9×；线上 90% Prefix Cache 命中率时，1M 上下文的 Prefill 吞吐是 Qwen3.7-Plus 的 8.6 倍。
- 突破:: 在不显著增加每 token 计算量的前提下扩展模型容量，同时大幅降低长上下文下的延迟。
- 定位:: Qwen4 架构的实验性预览
- N-gram Embedding 参数:: 51B（可放在 Host Memory 中异步 Prefetch）
- MTP:: 4B
- N-gram Embedding 启发来源:: [[Engram]]、Gemma 3n
- Muon 分工:: Attention、GDN、MoE Expert 等二维线性权重用 Muon；Embedding 和 Router 用 AdamW
