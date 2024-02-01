---
layout: post
keywords: chatgpt
description: chatgpt
title: "ChatGPT API 赚钱吗?"
categories: [Transformer]
tags: [Transformer]
---

* content
{:toc}

# 源起

一条推特讨论了 ChatGPT API 的收入和成本：

<center><img src="/image/chatgpt/chagpt_making_money.jpeg"></center>

收入很明确，[官网](https://openai.com/pricing)定价，童叟无欺，$0.002 / 1K tokens。




<center><img src="/image/chatgpt/chagpt_pricing.png"></center>

成本需要估算下，作者只考虑了计算成本。1K tokens 需要多少计算量，这些计算量需要花多少钱？

# 成本

## 计算量

一般认为 ChatGPT 和 GPT3 同样有 175B 参数，每个 token 的推理计算量约为 2x175B = 350B。详细讨论见[huggingface](https://discuss.huggingface.co/t/understanding-flops-per-token-estimates-from-openais-scaling-laws/23133)和[Paper](https://arxiv.org/pdf/2001.08361.pdf)。

## 计算成本

350B 计算量的成本是多少？

目前，NVIDIA A100是AWS最具成本效益的GPU选择，每个 [A100](https://www.nvidia.com/content/dam/en-zz/Solutions/Data-Center/a100/pdf/nvidia-a100-datasheet-us-nvidia-1758950-r4-web.pdf) 提供峰值 312 TFLOPS（万亿次浮点数/秒）FP16/FP32 混合精度吞吐量。

<center><img src="/image/chatgpt/nvidia-a100-datasheet.jpeg"></center>

A00 计算效率：按利用率 50%（该计算偏高）算，156 T/S。

A00 每秒能处理多少 ChatGPT token：156T/350B = 446

A00 每小时能处理多少 ChatGPT token：446 * 3600 = 1605600

A100 的成本：参考 lambdalabs 定价，8x NVIDIA A100 费用为 $12.00 / hr，那么 1 个 A100 费用为 $1.5 / hr

OpenAI 收入：每 1K tokens 0.002 美元，每小时 1605600 tokens，那么每小时收入 $3.2

收入 $3.2 / hr，成本 $1.5 / hr，利润率 100%。当然，这里只考虑推理的计算成本，其实最大的成本是训练阶段，训练阶段有多少成本就无从考据了，但一定成本不菲。

# 参考资料

- [Is OpenAI making money off their ChatGPT API?](https://twitter.com/debarghya_das/status/1631485296754987014)
- [Understanding FLOPs-per-token estimates from OpenAI’s scaling laws](https://discuss.huggingface.co/t/understanding-flops-per-token-estimates-from-openais-scaling-laws/23133)
- [The Economics of Large Language Models](https://sunyan.substack.com/p/the-economics-of-large-language-models)
- [ChatGPT背后的经济账](https://mp.weixin.qq.com/s/aAg1ptEkQ6ahdjs-3s_g3A)