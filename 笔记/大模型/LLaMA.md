## 整体介绍
Llama系列由Meta AI推出，已经迭代到llama4 。Llama 4在2025年4月正式发布，并推出了三个不同规模的模型版本，前两个都比deepseek V3/R1小一些：
1. Llama 4 Scout: 激活参数170亿（17B），总参数109B，16个专家，支持高达1000万（10M）token的超长上下文窗口。可在INT4模式下在单个H100 GPU上部署。
2. Llama 4 Maverick: 激活参数170亿（17B），总参数400B，128个专家，性能进一步提升。专家数少于DeepSeek V3的256，但在某些基准测试上与DeepSeek V3相当。上下文长度为 256K。
3. Llama 4 Behemoth: 仍在训练中，激活参数288B，16个专家，总参数2万亿，目标是超越当前的闭源顶尖模型如Gemini 2.5 Pro和GPT-4.5。

## metaCLIP
![LLaVA结构|500](file-20251110114536870.png)

