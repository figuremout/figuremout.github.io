---
title: "A Survey of KV Cache Optimization"
tags:
- LLM
draft: true
# date: 2024-09-01
# cover:
#     image: /images/test.png
#     hiddenInSingle: false
---
keyword: kv, cache, attention, inference, serving

综述分类见 
- KV Cache Compression, But What Must We Give in Return? A Comprehensive Benchmark of Long Context Capable Approaches
- pagedattention

1. KV Cache quantization
2. Token dropping
3. prompt compression
4. Exploring KV Cache-free architecture

# 注意力头之间共享
- GQA
- MQA

# 层间共享
- CLA

# 提示词前缀共享
- pagedattention: kv cache 分块，请求内 beam search 可以共享上下文，请求间共享可以共享相同的上下文
- sglang: 请求间提示词前缀匹配
- Chunkattention ACL2024：https://aclanthology.org/2024.acl-long.623/： 一个前缀感知的自注意力模块，它可以跨多个请求检测匹配的提示前缀，并在运行时在内存中共享它们的键/值张量，以提高 KV 缓存的内存利用率。

# kv cache quantization
- minicache: 跨层压缩 KV 缓存，KV 缓存状态在 LLM 中深层部分的相邻层之间表现出高度相似性。为了便于合并，我们建议将状态分解为幅度和方向分量，对状态向量的方向进行插值，同时保持其长度不变。

- cachegen: 首先，CacheGen 使用自定义张量编码器，利用 KV 缓存的分布式属性将 KV 缓存编码为更紧凑的比特流表示形式，解码开销可以忽略不计，以节省带宽使用。其次，CacheGen 会调整 KV 缓存不同部分的压缩级别来应对可用带宽的变化，以保持低上下文加载延迟和高生成质量。

- flexgen: FlexGen 在压缩KV缓存方面采取了两种策略：一是细粒度的组量化，二是注意力机制的稀疏化。从计算好的注意力矩阵中选择出前K个最高分数对应的索引，只保留这些token的v cache

- DecoQuant ACL2024 https://aclanthology.org/2024.acl-long.133/: 一种基于张量分解方法的新型无数据低位量化技术，可以有效压缩KV缓存

- llm on device 容忍感知压缩：用注意力分数计算token的信息密度，信息密度越低其kv cache可容忍的压缩率越高

- LCKV ACL2024 https://aclanthology.org/2024.acl-long.602/: 我们提出了一种新颖的方法，仅计算和缓存少量层的 KV，从而显着节省内存消耗并提高推理吞吐量。

- PyramidInfer ACL2024 https://aclanthology.org/2024.findings-acl.195/：我们发现影响后代的关键键和值的数量逐层减少，我们可以通过注意力权重的一致性来提取它们。基于这些发现，我们提出了 PyramidInfer，一种通过分层保留关键上下文来压缩 KV 缓存的方法。

- ThinK: 与基于序列长度优化内存的现有方法不同，我们在 KV 缓存的通道维度中发现了大量冗余，如注意力权重中不均匀的幅度分布和低秩结构所示。为此，我们提出了 ThinK，一种新颖的依赖于查询的 KV 缓存修剪方法，旨在最大限度地减少注意力权重损失，同时选择性地修剪最不重要的通道。

- A Simple and Effective L2 Norm-Based Strategy for KV Cache Compression EMNLP2024 ：我们分析了基于 Transformer 的纯解码器模型中的注意力分布，并观察到注意力分配模式在大多数层中保持一致。令人惊讶的是，我们发现缓存的 KV 对上的 L2 和注意力分数之间存在明显的相关性，其中密钥嵌入的低 L2 通常会导致解码期间的高注意力分数。这一发现表明，KV 对的影响可能是由查询之前嵌入的密钥本身决定的。基于这一观察，我们基于 L2 键嵌入来压缩 KV 缓存。

- VATP EMNLP2024: 然而，注意力中的键和值（KV）缓存的内存成本极大地限制了LLM的实际应用。最近的工作探索了 LLM 中用于减少 KV 缓存的令牌剪枝，仅依靠注意力分数作为令牌重要性指标。然而，我们对价值向量规范的调查揭示了一种明显不均匀的模式，质疑它们仅依赖于注意力分数。受此启发，我们提出了一种新方法：价值感知令牌修剪（VATP），它使用注意力分数和价值向量的 ℓ1 范数来评估令牌重要性。

- FlattenQuant COLING2024：该方法通过展平张量中的大通道来显着降低张量的最大值，以实现每张量的低比特量化，同时精度损失最小。

# Token dropping
只计算关键 KV 还是计算全部后筛选

- NACL ACL2024
- StreamingLLM
- H2O: Heavy-Hitter Oracle for Efficient Generative Inference of Large Language Models NIPS2023: 我们的方法基于一个值得注意的观察，即在计算注意力得分时，一小部分标记贡献了大部分价值。我们称这些标记为重击者（ 赫 2 ）。通过全面调查，我们发现（ 我 ）的出现 赫 2 是自然的，并且与文本中标记的频繁共现密切相关，并且（ 我 我) 删除它们会导致性能显著下降。

# prompt comression
- LLMLingua: Compressing Prompts for Accelerated Inference of Large Language Models: 这是一种粗到最新的及时压缩方法，涉及预算控制器，以在高压缩比下保持语义完整性，这是一个令牌级迭代的迭代压缩算法，以更好地模拟压缩的模型内容，以及基于指令调整的语言模型之间的分布对齐的方法
- Learning to compress prompt in natural language formats: 现有作品依赖于将长提示上下文压缩到软提示中。但是，软提示压缩遇到了不同LLM的可传递性限制，尤其是基于API的LLM。为此，这项工作旨在以LLM可转让性以自然语言的形式压缩冗长的提示。

# 存储感知
- Pacman: PM 感知压缩方法，用于 PM 上的日志结构 KV 存储
- flashattention: 减少注意力机制访问慢速的 GPU HBM

# idea
目标/创新点，用适当 5% 的精度换取：
1. KV Cache size 减小
2. 推理加速

基于观察：
- KV 在层间表现高度相似，可以压缩
- KV 在层内表现稀疏性，有很多决定哪些 token 重要的方法
- KV 可以量化，同时解码耗时要可忽略不计

1. KV 缓存状态在 LLM 中深层部分的相邻层之间表现出高度相似性
2. 注意力机制的稀疏化。从计算好的注意力矩阵中选择出前K个最高分数对应的索引，只保留这些token的v cache



稀疏注意力证明就近的 token 更重要，近的压缩率低，远的压缩率高

VQ-rec 的压缩方法

LSTM 中间状态表示之前上下文的 kv

语义感知压缩：注意力分数不能充分考虑上下文的语义信息以及词汇的实际贡献，可以采取NLP中的实体识别技术，判断每个kv cache的有用实体，按照块中含有实体的内容来决定压缩率

考虑到上下文的压缩率计算方法，优化对 KV Cache 压缩率的选择

融合一下：跨层 + 就近 + 提取关键 KV +

# Memory sharing
## Within Request
请求内不同 decoding strategy: vLLM

## Across Requests
请求间： SGLang

# Compaction
- pagedattention 提到了一嘴

# Change log
- 2024-09-01: Initial release
- 2024-09-02: Add ...

https://huggingface.co/docs/transformers/kv_cache
