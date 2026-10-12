Claude 是 Anthropic 开发的闭源模型系列，以 **“扩展思考”（Extended Thinking）** 和 **“Agentic Coding”** 为核心标签。与 GPT 的“内建思考”不同，Claude 将推理过程显式暴露给用户，并通过 system card 系统性地披露安全评估结果 。

### 🧬 Claude 模型迭代时间线

| 时间 | 关键模型/版本 | 核心变化与定位 |
| :--- | :--- | :--- |
| **2023.03** | **Claude 1** | 初代模型，奠定 Constitutional AI 安全对齐路线 |
| **2023.07** | **Claude 2** | 上下文扩展至 100K，首次面向公众开放 |
| **2024.03** | **Claude 3 (Opus/Sonnet/Haiku)** | 三档分级确立：Opus 旗舰、Sonnet 均衡、Haiku 极速 |
| **2024.06** | **Claude 3.5 Sonnet** | 首个在编码和 agentic 任务上显著超越前代的版本 |
| **2025.02** | **Claude 3.7 Sonnet** | **“一模型双模式”**：可产出近乎即时响应，也可展示逐步扩展思考；输出上限从 8K 提升至 64K  |
| **2025.05** | **Claude 4 (Opus/Sonnet)** | 扩展思考能力正式集成，Agent 能力大幅增强 |
| **2025.09** | **Claude Sonnet 4.5** | 被定位为“复杂 Agent 和编码的最佳模型”，支持 1M 上下文  |
| **2025.11** | **Claude Opus 4.5** | 面向编码、Agent、计算机使用、电子表格和长对话  |
| **2026.02** | **Claude Sonnet 4.6 / Opus 4.6** | Sonnet 4.6 全面升级编码、计算机使用、长上下文推理、Agent 规划；两者均支持 1M 上下文  |
| **2026.04** | **Claude Opus 4.7** | 软件工程和长时间运行编码任务改进，视觉分辨率提升  |
| **2026.06** | **Claude Sonnet 5 / Opus 5** | Sonnet 5 于 6.30 发布，Opus 5 于 7.24 发布，新增 3D 渲染能力  |
| **2026.09** | **Claude Opus 5.5 / Sonnet 5.5** | Opus 5.5 于 9.22 发布；Sonnet 5.5 于 9.29 发布，速度提升 30%+，成本降低最高 30%  |

### 💡 与开源路线的核心差异

Claude 的差异化在于 **“推理透明化 + Agent 产品矩阵”**：Extended Thinking 让用户看到模型的推理链，而 Claude Code、Cowork、Computer Use 等产品将模型能力封装为可执行复杂任务的 Agent 工具 。与 DeepSeek/Qwen 的架构创新路线不同，Claude 的迭代更侧重于 **Agentic 场景的工程化落地**和**安全评估的系统化披露**。