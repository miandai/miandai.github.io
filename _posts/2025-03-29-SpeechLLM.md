---
layout: post
keywords: blog
description: 本文系统性地分析了2024年以来具有代表性的7个开源语音大模型，包括Moshi、Mini-Omni、Freeze-Omni、GLM-4-Voice、MiniCPM-o、Step-Audio和Qwen2.5 Omni（其中MiniCPM-o和Qwen2.5 Omni作为多模态模型，我们重点探讨其语音交互模块）。受限于研究范围，部分同期项目暂未纳入本次讨论。基于官方技术报告、开源代码及模型权重，我们对这些模型的架构进行了深入的技术解构，通过白盒化的分析方法，旨在帮助读者快速把握语音大模型的技术精髓，并为后续研究或工程落地提供可复用的方法论。
title: "语音大模型技术详解：主流开源方案架构对比"
categories: [LLM]
tags: [LLM]
excerpt: 本文系统性地分析了2024年以来具有代表性的7个开源语音大模型，包括Moshi、Mini-Omni、Freeze-Omni、GLM-4-Voice、MiniCPM-o、Step-Audio和Qwen2.5 Omni（其中MiniCPM-o和Qwen2.5 Omni作为多模态模型，我们重点探讨其语音交互模块）。受限于研究范围，部分同期项目暂未纳入本次讨论。基于官方技术报告、开源代码及模型权重，我们对这些模型的架构进行了深入的技术解构，通过白盒化的分析方法，旨在帮助读者快速把握语音大模型的技术精髓，并为后续研究或工程落地提供可复用的方法论。
location: 北京
author: 增益
---

# 引言
现有的语音对话系统主要依赖级联式架构：

<center>
<img src="/image/SpeechLLM/ASR + LLM + TTS 级联架构.png" alt="ASR + LLM + TTS 级联架构" width="50%" />
</center>

这种架构在应用中存在局限：
1. 误差累积：错误在系统中传播和放大。例如，ASR的识别错误会影响NLP的理解，进而影响TTS的输出，导致整体性能下降。
2. 信息丢失：由于依赖于文本作为中介，因此语音在流转过程中会丢失非文本信息，如语音特有的音色、韵律、语速等声学信息。
3. 响应延迟：多模块延迟累加，导致系统响应速度慢，影响用户体验。
4. 轮次僵化：对话基于轮次，适合文本对话，但在模拟口语对话的某些方面存在局限，如中断、重叠语音和回传（即非中断的插话，如“好的”或“我明白了”）。

本文系统性地分析了2024年以来具有代表性的7个开源语音大模型，包括**Moshi、Mini-Omni、Freeze-Omni、GLM-4-Voice、MiniCPM-o、Step-Audio和Qwen2.5 Omni**（其中MiniCPM-o和Qwen2.5 Omni作为多模态模型，我们重点探讨其语音交互模块）。受限于研究范围，部分同期项目暂未纳入本次讨论。基于官方技术报告、开源代码及模型权重，我们对这些模型的架构进行了深入的技术解构，通过白盒化的分析方法，旨在帮助读者快速把握语音大模型的技术精髓，并为后续研究或工程落地提供可复用的方法论。
# 技术原理
本节将概述语音大模型的核心技术框架，为后续深入分析各模型的具体设计与优化奠定基础。
## 模型架构
语音大模型主要组件：

<center>
<img src="/image/SpeechLLM/语音大模型架构.png" alt="语音大模型架构" width="100%" />
</center>

- Speech Tokenizer：将连续的音频波形转换成离散 token。多由语音编码器和量化器组成，编码器从原始波形中编码基本信息，量化器则将前者离散化为 token。这里的基本信息包括语义信息和声学信息，不同模型会对两者的捕捉有所侧重。也有使用未量化的、实值表示的连续特征来作为语音表征。
- Language Model：将音频 token 附加到文本 token 后面，或者直接复用 speech adapter 后的 embedding，然后和 text embedding 结合，经过多层 transformer，predict next token。
- Token-to-Speech Synthesizer (Vocoder)：将 LLM 的输出 tokens 转换回音频波形，可看作是 Speech Tokenizer 的逆过程。主要有两种方式，一种是直接合成，直接将 LM 生成的 speech tokens 转换成音频波形，另一种是增强合成，先将 speech tokens 转换成 continuous latent representation（如mel-spectrograms），再将其输入到 vocoder 转成音频波形。两种方式的选择取决于 speech tokens 包含的信息类型，如果包含声学信息较多，则适合直接生成，相反，如果包含语义信息更丰富，但缺乏精细的声学细节，特别是高频范围，则更适合先增强为富含声学的表示，如梅尔频谱图，再合成语音。
## 训练方法
语音大模型的训练过程主要分为两个阶段：预训练、指令微调。
- 预训练：这一阶段的主要目标是学习语音数据中固有的统计模式和依赖关系，使模型能够根据前面的上下文 predict next token。训练数据主要是一些大规模开源语音数据，包括 ASR、TTS、语音翻译、播客和对话的数据集。关于训练起点，有些是冷启动，从头训练模型，模型参数刚开始是随机初始化；有些是持续预训练，基于现成的 LLM 做调整，使其可处理 speech tokens。实验证明，使用交错文本和语音 token 进行预训练课显著提高模型性能。
- 指令微调：指对语音大模型进行微调，使其能够遵循特定指令来执行各种任务，这一阶段对于提高预训练模型的泛化能力并使其更适应不同的应用至关重要。因此，关键在于创建有效的 instruction-following 数据集。
## 交互模式
语音大模型的典型交互模式是，模型接收预定义的输入序列，然后生成完整的响应，但这并没有反映真实语音交互的自然流程。例如，在真实对话中，随时打断、第三者介入都是很常见的情况。基于这些观察，业界定义了两种高级语音交互技能：实时交互和静音模式。
- 实时交互：指模型能够与用户及时交互，能够被用户打断，并根据新指令做出适当回应，同时能够在用户仍在说话时生成响应。这要求模型能够同时执行语音理解和语音生成。
- 静音模式：指模型在非交互期间保持不活动或静音。当一小群用户进行讨论时，模型需要辨别何时发言，何时保持沉默。
# 模型竞技场
本章内容均基于官方技术报告、开源代码及权重。
## Moshi
Moshi 是由法国初创团队 Kyutai 开发的全双工语音对话模型，2024 年 7 月发布实验原型，10月开源推理代码和模型权重，并给出了技术报告。
### 模型架构

<center>
<img src="/image/SpeechLLM/Moshi 架构.png" alt="Moshi 架构" width="100%" />
</center>

Moshi 是一个 multi-stream 语音到语音的 Transformer 模型，通过创新架构实现全双工对话。Moshi 架构主要组件包括：
- Speech Encoder& Decoder：由Mimi负责，speech waveforms—>discrete speech token（编码），speech tokens —> speech waveforms（解码）。
- RQ-Transformer：由 Temporal Transformer 和 Depth Transformer 组成。其中 Temporal Transformer 基于文本大模型 Helium 7B 模型构建，接收来自音频编解码器的音频 token作为输入，并生成上下文信息以指导 Depth Transformer（较小规模）的预测。Depth Transformer 负责处理多流音频数据中的子序列依赖关系，并预测每个子序列的 next token。
- Multi-stream：将用户和 Moshi 的音频流分别编码为不同的 token 流，与文本 token 流（Inner Monologue）一起建模，以实现复杂的对话交互。
#### Mimi
<center>
<img src="/image/SpeechLLM/Mini 架构及训练.png" alt="Mini 架构及训练" width="50%" />
</center>
Mini 架构由 Neural encoder、Neural decoder、残差矢量量化 (RVQ)组成。
- Neural encoder & decoder：
    - Neural encoder： 24kHz 单通道波形—>多个一维卷积—>512 维采样率为 12.5Hz 的 latent representation—>8 层的 Transformer 模块，以增强对音频信号的建模能力，更好地捕捉长距离依赖关系—>latent representation
    - Neural decoder采用与 encoder对称的结构，将latent representation投影回 24kHz 音频。
- Residual Vector quantization：latent representation—>Q=8 个量化器（2048 * 256）离散化—>S×Q 个 token
    - 为解决语义与音频质量之间的冲突，Mimi 使用分开的 RVQ 结构，将语义信息蒸馏到第一个普通的 VQ 中，并在其旁边应用一个 7 级的 RVQ。RVQ 的第一层是通过训练时从自监督模型（ WavLM）蒸馏语义信息得来。上图中的蓝色部分只是在训练时用到。
Mimi 使用纯对抗训练配置，其主要特点包括：
- 放弃了传统对抗训练中的的重建损失，完全依赖对抗损失来生成逼真的音频；
- 保留了蒸馏损失，用于将语义信息从预训练的 WavLM 模型迁移到 Mimi，以确保生成音频的语言内容准确；
- 这种方法虽然降低了客观音频质量指标，但显著提升了生成音频的主观听觉体验，使其听起来更加自然。
#### RQ-Transformer

<center>
<img src="/image/SpeechLLM/RQ-Transformer.png" alt="RQ-Transformer" width="50%" />
</center>

RQ-Transformer 是用于处理长序列数据的分层 Transformer 架构，由两个子 Transformer 组成：
- Temporal Transformer（大模型，32 layers）：处理时间维度上的依赖关系，将之前的所有子序列映射到一个时间上下文向量。如上图，即是将 s-1 步的所有子序列 embedding 相加形成融合 embedding，输入 Temporal Transformer 输出一个时间上下文向量，供第 s 步的各子序列生成。
- Depth Transformer（小模型，6 layers）：处理子序列维度上的依赖关系，基于时间上下文向量和之前的子序列预测当前子序列的下一个标记。如上图，即根据 s-1 步给出的上下文信息，结合 s 步前 k 个子序列向量，预测 s 步第 k+1 个子序列向量。
#### Multi-stream
Multi-stream 功能充分利用了 RQ-Transformer 的通用架构特性来实现多流处理。
<center>
<img src="/image/SpeechLLM/Moshi 的联合序列建模.png" alt="Moshi 的联合序列建模" width="50%" />
</center>

上图中横轴是时间维度，纵轴是序列维度。
- 序列维度：分为 Moshi stream 和 User stream，User steam 包含 8 个子序列，其中第 1 个子序列即 Mimi 中的 VQ 输出的语义 token 子序列，另外 7 个是 RVQ 输出的声学 token 子序列。Moshi Stream 除了类似的 8 个子序列，还包含 1 个 Text token 子序列，被称为内心独白信息。17 个子序列一起输入到 RQ-Transformer。
- 时间维度：声学 token 的生成要比语义 token 延迟1 个时间位，语义 token 会辅助声学 token 的生成。Moshi Steam 的生成不仅依赖于 User steam，也依赖于历史 Moshi steam。
Moshi 在训练时，模型预测完整的联合序列，在推理时，模型只生成 Moshi 的输出，同时保留用户音频作为输入。Moshi 无明确的轮次边界，可以同时进行说话和倾听，当用户说话而 Moshi 保持沉默时，Moshi 的音频流解码为“自然沉默”，即接近无声的波形。文本流提供了控制 Moshi 行为的机制，使其能够根据需要实时调整。这些设计使得 Moshi 能够实现更自然、更流畅的实时对话体验。
### 训练方法
Moshi 的训练过程分为多个阶段，每个阶段都有其特定的目标和训练数据。
1. Helium 预训练：Helium 7B，从头训练，使用 2.1T tokens 的高质量英文文本数据，包括 Wikipedia、Stack Exchange、科学文章以及过滤后的 CommonCrawl 数据。目标是为模型提供强大的语言理解和推理能力。
2. Moshi 预训练：Temporal Transformer 用 Helium 7B初始化，Depth Transformer 随机初始化。使用无监督音频数据集（7 百万小时音频），采用单一音频流表示所有说话者。目标是使模型能够将音频信号转换为离散化的声学标记，并提取语义信息。
3. Moshi 后训练：先使用 PyAnnote 对无监督音频数据集做说话人分离，产生两个音频流：一个包含主说话者的音频（代表 Moshi），另一个包含剩余的音频。然后模型被训练来预测主说话者的音频和文本标记，同时处理用户的音频输入。目标是使模型获得多流能力，能够同时处理用户和系统的音频流。
4. Moshi 微调：先在 Fisher 数据集（2000 小时电话对话）上训练，使模型能够处理真实的对话场景，包括重叠语音、中断和插话等。再在合成的指令数据集（包含一般知识对话、语音指令、角色扮演情境等）上训练，使 Moshi 成为一个有用的对话助手，并确保其使用一致的声音。
### 效果评估
论文对 Moshi 做了非常翔实的效果评估和消融实验，可归结为如下几类：
- 语言理解与生成能力：Text Language Modeling、Audio Language Modeling、Spoken Question Answering
- 语音处理与生成质量：Audio Tokenization、Compressing Moshi and Impact on Speech Quality
- 实时对话能力：Quality and Statistics of Generated Dialogues、Streaming ASR and TTS
- 多模态整合能力：Ablations on Generative Modeling
具体指标和分析请参考原论文，这里不再复述。

由于官方体验需要排队，个人就在本地做了部署测试：
- 本地部署：模型不大，部署很简单，一张RTX 4090足矣，总共占不到 20G 显存
- 实测效果：回复速度确实快，有时我话都没说完，它就开始回复了。但也就是个快，回复质量差强人意，频繁出现卡顿、抢话、喋喋不休的情况。只会说英文以及知识量和推理能力的短板就不提了，毕竟 7B 的模型。
尽管 Moshi 在实测中暴露出一些不足，但作为首个实时全双工对话系统，Moshi通过创新的多流架构和 Inner Monologue 方法，实现了低延迟的语音交互，为未来的实时人机交互技术开辟了新的探索路径。
## Mini-Omni

Mini-Omni 是由清华大学 和 Inspirai 的研究人员联合开发并于2024年9月6日发布的开源语音大模型。
### 模型架构

<center>
<img src="/image/SpeechLLM/Mini-Omni 模型结构.png" alt="Mini-Omni 模型结构" width="50%" />
</center>

- Speech Encoder：speech waveforms—>mel-spectrogram—>Whisper-small encoder—>audio_feature—>whisper_adapter(two-layer MLP，为对齐 LLM embedding)—>speech embedding
- Language Model：基于 Qwen2-0.5B 模型。text embedding + speech embedding —> LLM—> text token + speech token。生成时采用了文本引导的音频生成、文本延迟并行解码、批并行解码三种解码策略，提高了模型在实时语音交互中的推理能力和效率。
- Vocoder：音频 token 通过 SNAC（Sound-to-Numeric Audio Codec）decode 为连续的语音信号。
    - SNAC decode：speech token—>quantizer.from_codes(speech_token)—>cookbook对应 embedding—>深度可分离卷积WNConv1d—>4层DecoderBlock(转置卷积上采样，加权噪声注入，残差单元增强特征)—>最终上采样—>speech waveforms
### 训练方法

<center>
<img src="/image/SpeechLLM/Mini-Omni 三阶段训练方法.png" alt="Mini-Omni 三阶段训练方法" width="50%" />
</center>

Mini-Omni模型的训练方法分为三个主要阶段，每个阶段都有其特定的目标和方法：
1. Modality Alignment：Mini-Omni的核心模型完全冻结，只有语音理解和生成两个Adapter进行梯度更新。使用来自语音识别和语音合成任务的数据来训练模型，确保模型能够有效地处理音频输入和输出。
2. LM Adaption Training：Adapter的参数被冻结，专注于训练模型的核心部分。使用来自语音识别、口语问答和文本响应任务的数据进行训练。音频输出是从文本合成的，因此重点在于模型在接收到音频输入时能够生成准确的文本响应。
3. Multi-modal Finetuning：所有模型的权重都被解冻并进行训练。使用来自语音识别、口语问答、文本响应和综合多模态交互任务的数据进行训练。确保模型在多模态交互中表现出色，同时保留其在文本处理方面的原始能力。
### 效果评估
原文只做了语音识别效果评估，并展示了两个案例。
本地部署，一张4090足矣。效果：回复速度很快，但音频质量很差，说话慢吞吞，而且明显有刺刺啦啦的噪声。这跟模型参数量和训练充分性有很大关系，毕竟语音模型只有 0.5B参数。作者可能更关注于证明思路的可行性，而不是优化到最佳性能。
## Freeze-Omni
Freeze-Omni 是由 腾讯、西北工业大学和南京大学的研究人员联合开发的语音大模型。该模型于 2024 年11月首次发布，并在相关论文和开源项目中公开。
### 模型架构
<center>
<img src="/image/SpeechLLM/Freeze-Omni 模型架构.png" alt="Freeze-Omni 模型架构" width="50%" />
</center>

- Speech Encoder：speech waveforms—>kaldi.fbank—>mel-spectrogram—>2层卷积，降采样到1/4—>24层 transformer encoder—>adapter(multi-convolution layer with 2-times downsampling)—> speech embedding( 12.5Hz)
- Language Model：speech embedding—>Qwen2-7B-Instruct—>text + hidden_state
- Speech Decoder：hidden_state—>LLM2TTSCodecAR—>ticodec.vqvae—>speech waveforms
### 训练方法
1. 语音输入建模
  - 第一阶段：使用大量 ASR 数据训练语音编码器，使其能够将语音转换为文本，类似于常规语音识别模型的训练过程。
  - 第二阶段：将训练好的语音编码器与冻结的 LLM 连接，通过适配器将语音编码器输出映射到 LLM 的嵌入空间，并使用少量问答数据进行训练。
  - 第三阶段：构建多轮问答数据集，使用多说话人 TTS 系统生成语音数据，并在每个问题前添加可训练的提示嵌入，以指导 LLM 实现语音输入到文本输出的能力。
<center>
<img src="/image/SpeechLLM/语音输入三阶段建模.png" alt="语音输入三阶段建模" width="50%" />
</center>

2. 语音输出建模
  - 第一阶段：使用单码本编解码器模型和语音数据训练，以提取语音信号中的语音标记。
  - 第二阶段：构建大量文本-语音配对数据，将文本通过 LLM 分词器转换为文本标记，再通过 LLM 嵌入层转换为语义特征，并发送到 NAR 语音解码器，AR 语音解码器以教师强制方式预测输出语音标记。
  - 第三阶段：使用多轮问答数据集，利用 LLM 生成的文本标记和隐藏状态序列，添加 NAR 前缀语音解码器来对 LLM 隐藏状态建模，并将其输出传递给 NAR 语音解码器，仅训练 NAR 前缀语音解码器参数。
<center>
<img src="/image/SpeechLLM/语音输出三阶段建模.png" alt="语音输出三阶段建模" width="50%" />
</center>

3. 双向对话设计：通过多任务学习进行块级状态预测，以确定用户是否中断对话。具体来说，使用声学 VAD（语音活动检测）模块检测流式语音的起始点，当 VAD 被触发时，语音流将被逐块发送到 Freeze-Omni，并在 LLM 的最后一层之后添加一个额外的分类层来预测不同的状态。
<center>
<img src="/image/SpeechLLM/双向对话设计.png" alt="双向对话设计" width="50%" />
</center>

### 效果评估
Freeze-Omni 在语音输入理解、语音输出生成和口语问答任务中表现出色，准确性高，延迟低，验证了其在保持 LLM 原始智能的同时，实现高效语音对话的能力。
本地部署，一张 4090 足矣。Real-Time Interactive Demo 交互体验很赞，效果不苛求了。
## GLM-4-Voice
GLM-4-Voice 是由智谱AI（Zhipu AI）开发的端到端情感语音大模型。该模型于 2024年10月25日正式发布，并同步开源。
### 模型架构
<center>
<img src="/image/SpeechLLM/GLM-4-Voice 模型架构.png" alt="GLM-4-Voice 模型架构" width="50%" />
</center>

- Speech Encoder：speech waveforms—>mel-spectrogram—>Whisper Encoder + Vector Quantization—>discrete speech token
    - Whisper Encoder：2层卷积下采样（缩小到1/2），16层 transformer encoder with block causal attention，均值池化下采样（缩小到1/4）
    - Vector Quantization：(16384, 1280)
    - Example：30秒16kHz音频—>(1, 480000)—>分帧—>(3000, 400)—>短时傅立叶变换，并取模平方—>(3000, 201)—>梅尔滤波器组，对数变换—>(3000, 128)—>Whisper Encoder—>(375, 1280)—> Vecotr Quantization—> (375, )
- Language Model：Text Token + Speech Token—>GLM-4-9B—>Text Token + Speech Token
- Speech Decoder：speech token—>Flow-Matching—>mel-spectrogram—>HiFi-GAN—>speech waveforms
<center>
<img src="/image/SpeechLLM/语音编解码架构.png" alt="语音编解码架构" width="50%" />
</center>

### 训练方法
1. Joint Speech-Text Pre-training：在 GLM-4-9B 的基座模型基础之上，通过数百万小时音频和数千亿 token 的音频文本交错数据预训练，扩展LLM的语音能力。训练数据包括语音文本交错数据、语音数据、ASR、TTS 数据。

<center>
<img src="/image/SpeechLLM/Joint Speech-Text Pre-training.png" alt="Joint Speech-Text Pre-training" width="50%" />
</center>

2. Supervised Fine-tuning：使用多轮对话式语音对话和语音风格控制的口语对话数据做微调，以构建一个类人化的语音 chatbot。

<center>
<img src="/image/SpeechLLM/Supervised Fine-tuning.png" alt="Supervised Fine-tuning" width="50%" />
</center>

### 效果评估
- Base Model Evaluation：对预训练模型的评估，主要通过两个任务：speech language modeling 和 spoken question answering。大部分任务上的效果均高于基线模型。
- Chat Model Evaluation：对于微调模型的评估，包括模型的问答能力和知识记忆能力、生成语音质量，以及生成文本和生成语音的相关性。

<center>
<img src="/image/SpeechLLM/Chat model evaluation results.png" alt="Chat model evaluation results" width="50%" />
</center>

本地部署，一张 4090 足矣。简单对话效果不错，长System Prompt指令遵循能力一般。
## MiniCPM-o

MiniCPM-o 2.6 是由面壁智能于2025年1月16日发布的多模态大模型。总共 8B 参数，单图、多图、视频理解能力均超越 GPT-4V，单图理解能力超越 GPT-4o mini、Gemini 1.5 Pro 和 Claude 3.5 Sonnet，并首次支持在 iPad 上做实时视频理解。本节重点关注语音对话部分。
特点：
1. 全模态时分复用机制来处理并行多模态流
2. speech token—> speech embedding
### 模型架构

<center>
<img src="/image/SpeechLLM/MiniCPM-o 2.6 模型架构.png" alt="MiniCPM-o 2.6 模型架构" width="50%" />
</center>

- Speech Encoding：Whisper encoder+further compress 把音频转成 25 tokens/s，并将将音频embedding投影到 LLM 的 embedding 空间，后续需跟 text token 的 embedding 融合，将文本嵌入、音频嵌入和视觉嵌入（如果有）在相应位置合并
- Speech-to-Speech Framework：基于Qwen2.5-7B-Instruct，融合后的多模态嵌入输入到 Qwen2 架构的 LLM 中，通过 LLM 的 lm_head 生成文本 token，检测生成的文本中是否包含 TTS 触发标记（如 <|tts_bos|>），如果检测到，提取后续文本作为 TTS 输入
- Speech Decoding：speech embedding + text token—>dvae—>梅尔频谱—>声码器Vocos—>音频波形（基本复用 ChatTTS 模型）。注：Vocos为ChatTTS声码器，去除了转置卷积，直接由傅立叶逆变换完成上采样，速度相比 HiFiGAN 提升一个数量级，且效果更好
- 多模态流式支持：借鉴通信领域的时分复用技术，将每个模态的信息流分割成小块（每秒一块），并将同一秒内的所有模态信息组合成一个紧凑的序列单元输入给大语言模型主干。基于这个策略，主干模型可以高效地在线处理多模态信息流。如将音频分成一秒的连续块，同时音频编码时做 causal attention。音频解码也是同样，交替生成文本和音频，同样做causal attention mask，每个新生成的 audio chunk 只能“看到”（注意）前几块 text tokens 和之前的所有 audio tokens。
### 训练方法
1. Pretraining
    a. 视觉预训练：首先，使用大规模 image-text pairs 来对齐视觉和语言模块，这一步需冻结LLM backbone，只更新 visual encoder，使模型拥有基本的图像理解和OCR能力。然后，使用image-text interleave data 训练更新 visual encoder 和 LLM backbone，使模型具备 multi-image understanding 和 multimodal in-context-learning 能力。
    b. 音频预训练：使用 audio-text pair 数据训练 projector 层，以对齐音频模态。在大规模natural speech 数据上做端到端预训练，以学习丰富的细粒度的音频知识，然后使用 user instructions 对齐模型。
    c. 整体预训练：结合来自大规模 Web videos 的 visual 和 audio 流，使模型获得和对齐不同模态的丰富知识。
2. SFT：使用视觉问答、语音理解、语音生成及流媒体视频（含音频）理解数据做全参数微调，以统一模型的视觉能力、语音理解与生成能力以及流媒体处理能力，同时增强模型的指令跟随性能。
3. RLAIF：利用强大AI的反馈来进一步提升模型的可信度。通过分治策略对不同响应进行评分，构建用于直接偏好优化（DPO）的偏好数据集，并基于这些响应和评分优化模型，通过此方法将视频幻觉（相比图像幻觉更频繁出现的问题）降了63%。
### 效果评估
评估结果见 MiniCPM-o 2.6 原文，这里不再赘述。
本地部署非常方便，一张 4090 即可。Local WebUI Demo 的交互体验非常赞。
## Step-Audio
Step-Audio 是由阶跃星辰团队于 2025年2月18日推出的号称业内首款产品级的开源语音交互模型。
### 模型架构
<center>
<img src="/image/SpeechLLM/Step-Audio 模型架构.png" alt="Step-Audio 模型架构" width="50%" />
</center>

- Speech Tokenizer：linguistic tokenization 使用 Paraformer encoder，semantic tokenization 使用 CosyVoice tokenizer
  - Paraformer：audio waveforms—>mel-spectrogram—>Paraformer.encoder（主要是 Self-Attention+FFN）—>quantize（codebook.shape=[1024, 512]）—>linguistic token（16.7 tokens/s）
  - CosyVoice：audio waveforms—>mel-spectrogram—>CosyVoice.encoder—>CosyVoice.quantizer（codebook_size=4096）—>semantic token（25 tokens/s）
  - speech_tokens = linguistic token + semantic token
- Language Model：基于 Step-1，130B 参数，text tokens + audio tokens—>LLM—>text token—>text
- Speech Decoder：text —>3B language model—>audio token—>cosyvoice —> audio waveforms
  - cosyvoice：audio token —>Flow Matching—> mel-spectrogram —> HifiGAN—> audio waveforms
### 训练方法
1. 预训练：使用包含音频、文本和图像的3.3万亿个多模态数据标记进行训练，采用创新的双码本分词器框架实现跨模态对齐，并通过分散数据处理和模型布局等方法提升训练效率。
2. 后处理：针对TTS和ASR任务进行监督微调，并结合高质量合成数据和人类反馈进行强化学习，以实现对语音生成的多维度精细控制，包括情感、方言、语速等。
### 效果评估
在多个基准测试中表现出色，在 ASR 任务中，双码本方法显著提升了语音识别准确率；在 TTS 任务中，其在语音质量和说话人相似性方面均达到领先水平；在实时对话中，其在事实性、相关性和整体聊天体验方面均优于其他开源模型。
本地部署，各种模型占用磁盘空间 257G，其中主要是LLM就占了246G，整个系统用了 6 张 A6000 才跑起来，不愧是号称业界首个集语音理解与生成控制一体化的产品级开源实时语音对话系统，光看这模型大小就诚意满满。 ​​​测试了下长System Prompt，指令遵循能力效果不错。
## Qwen2.5 Omni

Qwen2.5-Omni是阿里巴巴于2025年3月27日凌晨发布并开源的全模态大模型。 该模型能够同时处理文本、图像、音频和视频输入，并实时生成文本与语音。在多模态融合任务的权威测评OmniBench中，Qwen2.5-Omni刷新了业界纪录，全面超越了Google的Gemini-1.5-Pro等同类模型。本节重点关注语音处理部分。

### 模型架构
<center>
<img src="/image/SpeechLLM/Qwen2.5-Omni 架构.png" alt="Qwen2.5-Omni 架构" width="50%" />
</center>

- Speech Encoder：speech waveforms—>mel-spectrogram—>Qwen2-Audio Encoder—>Latent Representations
  - Qwen2-Audio Encoder：梅尔频谱—>两个 1D 卷积层，降采样—>添加正弦位置编码，以保留时序信息—>通过32层 Transformer 进一步提取高层特征—>平均池化和线性投影—> latent representations
  - 注：speech latent representations 会与 text、image 和 video的 embedding 进行融合，然后输入 Qwen2_5OmniThinkerModel 中进行自回归生成
- Language Model：包含 thinker和 talker。thinker 专司多模态理解与文本生成（支持文本、图像、语音输入，输出文本），talker 仅当需要语音输出时启用，负责将文本（包括其高维表征）转为语音 token。
  - Thinker：speech + text + image + video embedding —> 28 层 Transformer Decoder—> high-level representations + text token
  - Talker：high-level representations + text token—>24 层 Transformer Decoder —> speech/text token
- Speech Decoder：speech/text token + thinker_representations—>Flow-Matching DiT—>mel-spectrogram—>BigVGAN—>speech waveforms
  - Flow-Matching DiT：初始化噪声梅尔频谱—>迭代去噪（以 speech token 对应 embedding 投影后的梅尔频谱为参考条件）—输出梅尔频谱
  - BigVGAN：梅尔频谱—>通过 ConvTranspose1d 或插值层逐步增加时间分辨率—>使用多个Residual Blocks 增强特征提取能力—>最后一层卷积将特征映射到波形维度（如 1 通道的 24kHz 音频）
- 多模态实时处理：Qwen2.5-Omni 通过时间对齐机制 TMRoPE 对不同模态的数据进行时间同步，并采用时间交错方法将音频和视频表示按时间顺序排列。同时，它使用分块流式处理技术，将长序列数据分割成更小的块进行处理，从而实现多模态数据的实时输入和高效处理。
### 训练方法
1. 预训练
  - 单模态编码器训练：
    - 方法：训练视觉编码器和音频编码器，冻结thinker（talker未参与训练）
    - 数据：使用大量音频-文本和图像-文本对来增强 LLM 的语义理解能力
  - 多模态数据联合训练：
    - 方法：解冻所有参数
    - 数据：多任务数据集，使模型能够同时处理多种任务和模态，提升其在复杂现实世界数据集上的处理能力。
  - 长序列数据训练：将文本、音频、图像和视频数据的序列长度扩展到 32,768 个 token。增强模型对复杂长序列数据的理解能力。
2. 后训练
  - 指令微调：使用ChatML格式的 instruction-following 数据训练，包括纯文本对话数据、视觉模态对话数据、音频模态对话数据、混合模态对话数据。提升模型对指令的理解和执行能力，能够根据不同模态的输入生成相应的文本和语音响应。
  - 语音生成训练，主要针对 Talker，分三阶段：
    - 上下文延续学习：使用包含多模态上下文和语音响应的对话数据集训练，Talker 通过预测下一个 token来学习语音延续任务
    - 语音生成稳定性增强：采用DPO（直接偏好优化）方法训练，构建包含输入序列、良好生成语音序列和不良生成语音序列的三元祖数据集，根据词错误率（WER）和标点停顿率相关奖励分数对样本排序，使用强化学习优化模型
    - 多说话人指令微调：提升语音响应的自然性和可控性
### 效果评估
1. Text→Text：评估一般的语言理解与生成、数学和科学问题的解决能力、编程代码的生成与理解能力，优于Qwen2.5-7B、Gemma2-9B 等模型
2. Audio→Text：评估音频理解、音频推理、语音交互能力。表现出色，在多个基准测试和数据集上都达到了最先进的水平
3. Image→Text：评估处理图像输入时的理解、推理和文本生成能力，涵盖了从基础视觉问答到复杂的 OCR 任务，以及数学和大学级别问题的解决能力。表现出色，在多个基准测试上达到了最先进的水平。
4. Video (w/o Audio)→Text：评估对视频内容的理解、视频中视觉和动态信息的处理、将视频内容转化为准确的文本描述等能力。在所有比较的数据集上都超过了其他开源全能模型。
5. Multimodality→Text：评估的是模型处理多模态输入（图像、音频和文本）并生成文本响应的能力，即多模态理解能力。在 OmniBench 上全面超越了其他所有模型。

ps：Launch Local Web UI Demo 的交互体验不如MiniCPM-o，还是去 官方Qwen Chat 体验吧。
## 语音编解码对比
通过对各模型架构的分析，总结出以下语音编解码方面的关键差异：
1. 音频表征方式：Moshi是唯一直接在原始音频波形上建模的模型，其他模型均采用梅尔频谱作为中间表征，这表明梅尔频谱在计算效率与特征表达能力之间提供了更好的平衡。
2. LLM输入形式：Moshi、GLM-4-Voice、Step-Audio采用离散token id输入（通过VQ-VAE/RVQ量化），其他模型（如Freeze-Omni、MiniCPM-o）使用连续speech embedding。离散token更适合与文本token联合建模，而连续embedding可能保留更多声学细节。
3. 语音生成策略：文本引导生成成为主流（Moshi的Inner Monologue、GLM-4-Voice的文本延迟解码等），Mini-Omni等模型通过ASR Adapter实现文本条件控制。显示出现有方案普遍依赖文本模态的强语义引导。
# 结语
本文系统梳理了最近一年具有代表性的7个开源语音大模型，包括 Moshi、Mini-Omni、Freeze-Omni、GLM-4-Voice、MiniCPM-o、Step-Audio、Qwen2.5 Omni，并深入分析了其核心组件，包括语音表征、训练策略、流式交互机制等。在语音表征方面，我们从输入与输出的角度对比了不同模型对语义信息与声学信息的编码方式；在训练方法上，探讨了多阶段预训练、模态对齐等关键技术的演进；同时详细解析了全双工交互、实时流式处理等前沿方向的技术实现。

作为新兴技术领域，语音端到端大模型仍存在诸多开放性问题：在语音表征方面，如何平衡语义保真度与声学质量尚未形成统一范式；在训练方法上，多模态联合优化策略仍需探索；而流式交互中的延迟控制、对话状态管理等工程挑战也亟待解决。相信随着Speech LLM能力的持续进化与计算架构的创新，更自然、更智能的人机语音交互将成为可能。
# 参考文献
1. Recent Advances in Speech Language Models: A Survey
2. WavChat: A Survey of Spoken Dialogue Models

注：这里仅列举了除以上模型技术报告外的参考文献
