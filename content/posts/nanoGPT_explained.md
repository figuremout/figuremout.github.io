---
title: "nanoGPT 源码解析：GPT-2 训练、微调及推理"
tags:
- LLM
date: 2024-10-01
cover:
    image: /images/gpt2_arch.png
---
首先回顾一下 OpenAI GPT 系列经典模型：
1. GPT-1 ([Radford et al., 2018](https://openai.com/index/language-unsupervised/)) 参数规模为 117 M，首次将 Transformer 应用于语言模型，并开创了 NLP 领域无监督 pretrain + 有监督 finetune 的训练范式
2. GPT-2 ([Radford et al., 2019](https://openai.com/index/better-language-models/)) 最大参数规模为 1.5 B，发现更大规模的模型可以实现 zero-shot，只需 pretrain，不需要 finetune 就能解决下游任务
3. GPT-3 ([Brown et al., 2020](https://arxiv.org/abs/2005.14165)) 最大参数规模为 175 B，发现模型具有了 ICL 能力（也就是涌现 emergent），不需要传统的 finetune 步骤，在提示词中提供 few-shot 就能让模型更好地学习下游任务

想要研究 GPT-2 源码，可以参考的实现有 [nanoGPT](https://github.com/karpathy/nanoGPT)、[llm.c](https://github.com/karpathy/llm.c)、HF Transformers [GPT2Model](https://huggingface.co/docs/transformers/model_doc/gpt2#transformers.GPT2Model) 等。简单起见，我选择 nanoGPT 进行研究，它复现了[最小版本的 GPT-2](https://huggingface.co/openai-community/gpt2)。

# Config
Config 中存储了 LLM 的一些通用属性，反映了模型的规模和架构特性。

HF Transformers 通过 [Configuration](https://huggingface.co/docs/transformers/main/en/create_a_model#configuration) 定义模型架构并创建相应的 model。HF Transformers 中不同模型有自己的 Config 类，如 [BertConfig](https://huggingface.co/docs/transformers/main/en/model_doc/bert#transformers.BertConfig), [GPT2Config](https://huggingface.co/docs/transformers/model_doc/gpt2#transformers.GPT2Config) 等，它们具有不同的属性。但它们又都是 [PretrainedConfig](https://huggingface.co/docs/transformers/main/en/main_classes/configuration) 的子类，因此也具有一些通用的属性名，如 `hidden_size, num_attention_heads, and num_hidden_layers` 等。
```python
from transformers import BertConfig, BertModel

# Building the config
config = BertConfig()

# Building the model from the config
model = BertModel(config)
```

类似的，nanoGPT 通过 [GPTConfig](https://github.com/karpathy/nanoGPT/blob/9755682b981a45507f6eb9b11eadef8cb83cebd5/model.py#L108-L116) 类定义模型的属性及其默认值。大模型相关 paper 又有自己惯用的一套表示符号，为方便联系起来理解，对比如下：

| | BERT<br>([Devlin et al., 2018](https://arxiv.org/abs/1810.04805)) | GPT-2<br>([Radford et al., 2019](https://openai.com/index/better-language-models/))| GPT-3<br>([Brown et al., 2020](https://arxiv.org/abs/2005.14165)) | nanoGPT | GPT-2 117M |
| - | - | - | - | - | - | - |
| 上下文长度 | - | - | $n_{ctx}$ | `GPTConfig.block_size` | 1024 |
| 隐藏层数量 | L | - | $n_{layers}$| `GPTConfig.n_layer` | 12 |
| 注意力头数量 | A | - | $n_{heads}$ | `GPTConfig.n_head` | 12 |
| 词嵌入向量维度 | H | $d_{model}$ | $d_{model}$ | `GPTConfig.n_embd` | 768 |
| 词表大小 | - | - | - | `GPTConfig.vocab_size` | 50257 |

- 上下文长度：指的是模型可接受输入的 token seq 长度，这是固定的，若更短则应填充，若更长则应截断
- 隐藏层数量：指的是 Transformer 块堆叠的数量
- 注意力头数量：指的是每个 token 对应的注意力头的数量
- 词嵌入向量维度：指的是 token id 对应的嵌入向量的维度
- 词表大小：指的是 Tokenizer 中所有 token 的集合大小

nanoGPT 代码中，还有一些变量用于表示张量的维度，习惯后对于阅读代码有帮助：
- `B`: batch size
- `T`: sequence length，等同于 `block_size`
- `C`: embedding size，等同于 `n_embd`
- `nh`: number of heads，等同于 `n_head`
- `hs`: head size，等同于 `n_embd/n_head`

# Tokenizer
nanoGPT 使用 [tiktoken](https://github.com/openai/tiktoken) 加载 GPT-2 的 BPE 分词器，将连续文本分割成 token，再输出这些 token 在 vocab 中的索引，即 token ids 序列。可以在 [OpenAI Tokenizer](https://platform.openai.com/tokenizer) 体验 BPE 分词器的效果。

nanoGPT 中还实现了一个简易的字符级分词器，对一段较短的 shakespeare 文本进行分词，我们可以通过它一窥 tokenizer 的原理。
1. 读取一段连续的 shakespeare 剧本，如下：
```
First Citizen:
Before we proceed any further, hear me speak.

All:
Speak, speak.

First Citizen:
You are all resolved rather to die than to famish?

All:
Resolved. resolved.

...
```

2. 按字符进行分割，得到无重复的字符集合作为 vocab：
```
 !$&',-.3:;?ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz
```

3. 实现 `encode/decode` 函数。`encode` 将字符串映射为 token id 序列，`decode` 反之。

Tokenize 过程的代码如下：
```python
# permalink: https://github.com/karpathy/nanoGPT/blob/9755682b981a45507f6eb9b11eadef8cb83cebd5/data/shakespeare_char/prepare.py#L19-L40
# 读取文本文件
with open(input_file_path, 'r') as f:
    data = f.read()
print(f"length of dataset in characters: {len(data):,}")

# get all the unique characters that occur in this text
# 字符集合即为 vocab
chars = sorted(list(set(data)))
vocab_size = len(chars)
print("all the unique characters:", ''.join(chars))
print(f"vocab size: {vocab_size:,}")

# create a mapping from characters to integers
# 字符在 vocab 中的索引即为 token id
stoi = { ch:i for i,ch in enumerate(chars) }
itos = { i:ch for i,ch in enumerate(chars) }
def encode(s):
    return [stoi[c] for c in s] # encoder: take a string, output a list of integers
def decode(l):
    return ''.join([itos[i] for i in l]) # decoder: take a list of integers, output a string

# create the train and test splits
# 将文本划分为训练集和测试集，训练集占 90%，测试集占 10%
n = len(data)
train_data = data[:int(n*0.9)]
val_data = data[int(n*0.9):]
```
上面代码中，值得注意的还有划分训练集和测试集的方式，就是简单地把连续文本划分为两部分，训练集 90%，测试集 10%。

# DataLoader
HF Transformers 中通过 [DataLoader](https://huggingface.co/docs/transformers/main/en/training#dataloader) 从数据集中采样出一个 batch 的数据，nanoGPT 通过 `get_batch` 函数实现了类似的功能。
1. 在数据集中随机选出 `batch_size` 个起点，每个起点后连续采样 `block_size` 个 token id 得到 `(batch_size, block_size)` 的 token id seq，作为一个 batch 的输入数据 x
2. 输入数据 x 每行向后偏移一个 token，仍为 `(batch_size, block_size)`，作为一个 batch 的标签数据 y

输入数据 x 中每个 token id 经过模型 forward 后会输出一串长度为 `vocab_size` 的预测分数 logits（见 [Model 章节](#model)），而标签正是输入数据 x 向后偏移一个位置的 token id。

```python
# permalink: https://github.com/karpathy/nanoGPT/blob/9755682b981a45507f6eb9b11eadef8cb83cebd5/train.py#L116-L131
def get_batch(split):
    # We recreate np.memmap every batch to avoid a memory leak, as per
    # https://stackoverflow.com/questions/45132940/numpy-memmap-memory-usage-want-to-iterate-once/61472122#61472122
    if split == 'train':
        data = np.memmap(os.path.join(data_dir, 'train.bin'), dtype=np.uint16, mode='r')
    else:
        data = np.memmap(os.path.join(data_dir, 'val.bin'), dtype=np.uint16, mode='r')
    # 从数据集 (train或val) 中随机采样出维度为 (batch_size, block_size) 的 token id seq，作为输入数据 x
    ix = torch.randint(len(data) - block_size, (batch_size,))
    x = torch.stack([torch.from_numpy((data[i:i+block_size]).astype(np.int64)) for i in ix])
    # 标签 y 为 x 向后偏移一个 token，维度同样为 (batch_size, block_size)
    y = torch.stack([torch.from_numpy((data[i+1:i+1+block_size]).astype(np.int64)) for i in ix])
    if device_type == 'cuda':
        # pin arrays x,y, which allows us to move them to GPU asynchronously (non_blocking=True)
        x, y = x.pin_memory().to(device, non_blocking=True), y.pin_memory().to(device, non_blocking=True)
    else:
        x, y = x.to(device), y.to(device)
    return x, y
```

# Model
GPT-2 的模型架构还是比较简单清晰的。[Figure 1](#fig:GPT_2_arch) 截图自 [LLM Visualization](https://bbycroft.net/llm)，这是一个非常好的 GPT 模型架构交互式可视化网站。该图所示和代码实现完全一致，可以比对着看。

<div id="fig:GPT_2_arch" style="text-align: center;">
    <img src="/images/gpt2_arch.png" style="display: block; margin: 0 auto;"/>
    <p style="color: #999; font-size: 0.9rem;">
        Figure 1. GPT-2 architecture.<br>
        (Image source: <a style="color: inherit; font-size: inherit;" href="https://bbycroft.net/llm">LLM Visualization</a>)
    </p>
</div>

模型各组件的定义（`__init__`）以及组装起来的前向传播过程（`forward`）代码如下：

```python
# permalink: https://github.com/karpathy/nanoGPT/blob/9755682b981a45507f6eb9b11eadef8cb83cebd5/model.py#L118-L330
class GPT(nn.Module):

    def __init__(self, config):
        super().__init__()
        assert config.vocab_size is not None
        assert config.block_size is not None
        self.config = config

        # 定义模型中的层
        self.transformer = nn.ModuleDict(dict(
            # Word Token Embed。index-embed 的索引表，共包含 vocab_size 个维度为 n_embd 的 token embed
            # 根据正态分布随机初始化
            wte = nn.Embedding(config.vocab_size, config.n_embd),
            # Word Position Embed，共包含 block_size 个维度为 n_embd 的 position embed
            wpe = nn.Embedding(config.block_size, config.n_embd),
            drop = nn.Dropout(config.dropout),
            # n_layer 层 transformer 块
            h = nn.ModuleList([Block(config) for _ in range(config.n_layer)]),
            ln_f = LayerNorm(config.n_embd, bias=config.bias),
        ))
        # 模型头
        self.lm_head = nn.Linear(config.n_embd, config.vocab_size, bias=False)
        # with weight tying when using torch.compile() some warnings get generated:
        # "UserWarning: functional_call was passed multiple values for tied weights.
        # This behavior is deprecated and will be an error in future versions"
        # not 100% sure what this is, so far seems to be harmless. TODO investigate
        # 共享权重
        self.transformer.wte.weight = self.lm_head.weight # https://paperswithcode.com/method/weight-tying

        # init all weights
        # 根据正态分布初始化所有权重，bias 初始化为 0
        self.apply(self._init_weights)
        # apply special scaled init to the residual projections, per GPT-2 paper
        for pn, p in self.named_parameters():
            if pn.endswith('c_proj.weight'):
                torch.nn.init.normal_(p, mean=0.0, std=0.02/math.sqrt(2 * config.n_layer))

        # report number of parameters
        print("number of parameters: %.2fM" % (self.get_num_params()/1e6,))

...

    # idx target 其实就是 x y，维度为 (batch size, seq len) 的 token id
    def forward(self, idx, targets=None):
        device = idx.device
        # b 是 batch size，t 是 seq len
        b, t = idx.size()
        # 确保输入的序列长度不超过模型的最大处理长度 block_size
        assert t <= self.config.block_size, f"Cannot forward sequence of length {t}, block size is only {self.config.block_size}"
        # 创建一个 [0, t-1] 的列表，用于表示序列中每个位置的索引
        pos = torch.arange(0, t, dtype=torch.long, device=device) # shape (t)

        # forward the GPT model itself
        tok_emb = self.transformer.wte(idx) # token embeddings of shape (b, t, n_embd)
        pos_emb = self.transformer.wpe(pos) # position embeddings of shape (t, n_embd)
        # token embed + position embed 得到 input embed，维度为 (b, t, n_embd)
        x = self.transformer.drop(tok_emb + pos_emb)
        # 堆叠 n_layer 层的 transformer 块，输出维度为 (b, t, n_embd)
        for block in self.transformer.h:
            x = block(x)
        x = self.transformer.ln_f(x)

        if targets is not None:
            # if we are given some desired targets also calculate the loss
            # 若有目的标签，将 hidden states 传入模型头，输出维度为 (b, t, vocab_size) 的预测分数
            logits = self.lm_head(x)
            # 计算交叉熵损失
            # logits: (b, t, vocab_size) -> (b*t, vocab_size)
            # targets: (b, t) -> (b*t)
            # targets 是一组 token id，忽略其中值为 -1 的 token（如 <pad>）的损失
            loss = F.cross_entropy(logits.view(-1, logits.size(-1)), targets.view(-1), ignore_index=-1)
        else:
            # inference-time mini-optimization: only forward the lm_head on the very last position
            # 推理时只计算最后一个 token 的预测分数，输出维度为 (b, 1, vocab_size)
            logits = self.lm_head(x[:, [-1], :]) # note: using list [-1] to preserve the time dim
            loss = None

        return logits, loss
```

模型结构中值得注意的有以下几处：
- Token embedding 和 position embedding 都是根据正态分布随机初始化的，负责将 token id 映射为 `n_embd` 维度的嵌入向量。
- `lm_head` 是一个模型头，对应于 HF Transformers 的 [Model heads](https://huggingface.co/docs/transformers/main/en/create_a_model#model-heads)。对于语言模型，生成文本任务需要的是分类头，因此 `lm_head` 是一个线性层，负责将 transformer 块输出的 `n_embd` 维度向量映射到 `vocab_size` 维度的 logits。训练时这个 logits 可以直接用于计算误差，但推理时还需要经过 Softmax 转换为采样概率。
- Token embedding 和 `lm_head` 共享权重（Weight Tying）

模型的训练和推理都是通过 `forward` 函数在模型中进行前向传播。可以印证的是，Transformer 是一种 Seq2Seq 模型，每次 forward 同时处理序列中所有 token，同时预测每个 token 的 logits 分数，这个 logits 对应的是下一个 token 的概率。因此输入一段序列，可以同时预测序列中每个 token 的下一个 token，计算所有交叉熵误差的平均，这就解释了为什么训练时标签数据 y 要传入一串连续的 token id 而不是一个。

## Transformer
对模型整体有了概念，接下来研究核心模块——Transformer 块。GPT-2 的 transformer 块和 GPT-1 有所不同。对比 [Figure 1](#fig:GPT_2_arch) 和 [Figure 2](#fig:GPT_1_transformer) 发现，GPT-1 的 transformer 块遵循 Self-attention 原论文 ([Vaswani et al., 2017](https://arxiv.org/abs/1706.03762)) 中的 decoder-only transformer 架构，采用后置层正则化（Post-LN），而 GPT-2 采用的是**前置层正则化**（Pre-LN）。大多数模型都采用前置来增强训练稳定性，尽管这会影响模型性能 ([Zhao et al., 2023](https://arxiv.org/abs/2303.18223)) 。 [^6]

<div id="fig:GPT_1_transformer" style="text-align: center;">
    <img src="/images/GPT_1_transformer.png" style="display: block; margin: 0 auto;"/>
    <p style="color: #999; font-size: 0.9rem;">
        Figure 2. GPT-1 transformer architecture.<br>
        (Image source: <a style="color: inherit; font-size: inherit;" href="https://openai.com/index/language-unsupervised/">Radford et al., 2018</a>)
    </p>
</div>

nanoGPT 的 transformer 块由 `Block` 类定义：
```python
# permalink: https://github.com/karpathy/nanoGPT/blob/9755682b981a45507f6eb9b11eadef8cb83cebd5/model.py#L94-L106
class Block(nn.Module):

    def __init__(self, config):
        super().__init__()
        self.ln_1 = LayerNorm(config.n_embd, bias=config.bias)
        self.attn = CausalSelfAttention(config)
        self.ln_2 = LayerNorm(config.n_embd, bias=config.bias)
        self.mlp = MLP(config)

    def forward(self, x):
        # 前置 LayerNorm
        x = x + self.attn(self.ln_1(x))
        x = x + self.mlp(self.ln_2(x))
        return x
```

## MHA
Transformer 块中最重要的是 `CausalSelfAttention` 层，实现了 **MHA**。李宏毅的 ML 课程是一个比较好的入门课程，尤其是对自注意力机制的教学非常通俗易懂，令我印象深刻。

<div id="fig:MHA-hylee" style="text-align: center;">
    <img src="/images/MHA-hylee.png" style="display: block; margin: 0 auto; width: 80%;"/>
    <p style="color: #999; font-size: 0.9rem;">
        Figure 3. Multi-head Self-attention.<br>
        (Image source: <a style="color: inherit; font-size: inherit;" href="https://speech.ee.ntu.edu.tw/~hylee/ml/ml2021-course-data/self_v7.pdf">李宏毅 ML 课程 PPT</a>)
    </p>
</div>

但是上图并没有详细描述计算过程，因此我根据 nanoGPT 源码绘制了 [Figure 4](#fig:MHA)。

<div id="fig:MHA" style="text-align: center;">
    <img src="/images/MHA.drawio.svg" style="display: block; margin: 0 auto; width: 60%;"/>
    <p style="color: #999; font-size: 0.9rem;">
        Figure 4. Multi-head Self-attention. (2 heads)<br>
    </p>
</div>

1. 每个 token 的嵌入线性映射到 `n_embd` 大小的 Q, K, V 向量。
2. 每个 token 的 Q, K, V 向量分别等分为 `n_head` 份，按 `1 ~ n_head` 编号，每份 `n_embd/n_head` 大小。
3. 对于 1 号注意力头，用每个 token 的 1 号 Q 和所有 token 的 1 号 K 相乘，得到 1 号注意力分数矩阵 `(T, T)`。其他注意力头计算同理，也就是得到 `n_head` 个注意力分数矩阵 `(T, T)`。
4. 注意力矩阵进行单向掩码和 Softmax，实现单向注意力，变为下三角矩阵。
<div id="fig:MHA-hylee" style="text-align: center;">
    <img src="/images/causal_mask.drawio.svg" style="display: block; margin: 0 auto; width: 80%;"/>
    <p style="color: #999; font-size: 0.9rem;">
        Figure 5. Causal mask.<br>
    </p>
</div>

5. 对于 1 号注意力头，用 1 号注意力分数矩阵和所有 token 的 1 号 V 相乘，得到每个 token 的 1 号输出向量，形状为 `(T, n_embd/n_head)`。其他注意力头计算同理，也就是得到 `n_head` 个输出向量 `(T, n_embd/n_head)`。
6. 每个 token 不同注意力头的输出向量合并，得到最终的输出向量 `(T, n_embd)`。

MHA 的实现代码如下：
```python
# permalink: https://github.com/karpathy/nanoGPT/blob/9755682b981a45507f6eb9b11eadef8cb83cebd5/model.py#L29-L76
class CausalSelfAttention(nn.Module):

    def __init__(self, config):
        super().__init__()
        assert config.n_embd % config.n_head == 0
        # key, query, value projections for all heads, but in a batch
        self.c_attn = nn.Linear(config.n_embd, 3 * config.n_embd, bias=config.bias)
        # output projection
        self.c_proj = nn.Linear(config.n_embd, config.n_embd, bias=config.bias)
        # regularization
        self.attn_dropout = nn.Dropout(config.dropout)
        self.resid_dropout = nn.Dropout(config.dropout)
        self.n_head = config.n_head
        self.n_embd = config.n_embd
        self.dropout = config.dropout
        # flash attention make GPU go brrrrr but support is only in PyTorch >= 2.0
        self.flash = hasattr(torch.nn.functional, 'scaled_dot_product_attention')
        if not self.flash:
            print("WARNING: using slow attention. Flash Attention requires PyTorch >= 2.0")
            # causal mask to ensure that attention is only applied to the left in the input sequence
            # 因果掩码为一个 (1, 1, T, T) 的下三角矩阵
            self.register_buffer("bias", torch.tril(torch.ones(config.block_size, config.block_size))
                                        .view(1, 1, config.block_size, config.block_size))

    def forward(self, x):
        B, T, C = x.size() # batch size, sequence length, embedding dimensionality (n_embd)

        # calculate query, key, values for all heads in batch and move head forward to be the batch dim
        # 先经过线性层将 n_embd 维度的嵌入映射为 3*n_embd 维度的张量，再按嵌入维度三等分得到 Q, K, V
        # Q, K, V 维度都是 (B, T, C)
        q, k, v  = self.c_attn(x).split(self.n_embd, dim=2)
        # 按注意力头数分割 Q, K, V 的嵌入维度，转换为 (B, nh, T, hs)，其中 nh = n_head, hs = n_embd/n_head
        k = k.view(B, T, self.n_head, C // self.n_head).transpose(1, 2) # (B, nh, T, hs)
        q = q.view(B, T, self.n_head, C // self.n_head).transpose(1, 2) # (B, nh, T, hs)
        v = v.view(B, T, self.n_head, C // self.n_head).transpose(1, 2) # (B, nh, T, hs)

        # causal self-attention; Self-attend: (B, nh, T, hs) x (B, nh, hs, T) -> (B, nh, T, T)
        if self.flash:
            # efficient attention using Flash Attention CUDA kernels
            y = torch.nn.functional.scaled_dot_product_attention(q, k, v, attn_mask=None, dropout_p=self.dropout if self.training else 0, is_causal=True)
        else:
            # manual implementation of attention
            # 计算注意力矩阵 (B, nh, T, hs) x (B, nh, hs, T) / sqrt(hs) -> (B, nh, T, T)
            att = (q @ k.transpose(-2, -1)) * (1.0 / math.sqrt(k.size(-1)))
            # 将 att 中上三角部分的值替换为 -inf
            att = att.masked_fill(self.bias[:,:,:T,:T] == 0, float('-inf'))
            # 在 softmax 中 exp(-inf) = 0，因此实现因果注意力
            att = F.softmax(att, dim=-1)
            att = self.attn_dropout(att)
            y = att @ v # (B, nh, T, T) x (B, nh, T, hs) -> (B, nh, T, hs)
        # 将同一个 token 的不同注意力头的 y 按嵌入维度重新拼接回 n_embd
        y = y.transpose(1, 2).contiguous().view(B, T, C) # re-assemble all head outputs side by side

        # output projection
        y = self.resid_dropout(self.c_proj(y))
        return y
```

想要明白代码中 Q, K, V 之间的矩阵乘法，先要搞清楚**高维张量乘法**的计算。`@` 操作符其实就是 `torch.matmul`[^1]，它会将最后两维视为矩阵，多余的维度视为批处理维度，若两个高维张量的批处理维度不一致，可以进行广播 [^2]。批处理维度按照逐元素乘积，矩阵进行矩阵乘法，见下面这个 gist 示例。

<script src="https://gist.github.com/figuremout/38b93348d307cf8895088cdc2c71aaf3.js"></script>

因此计算 Q @ K `(B, nh, T, hs) x (B, nh, hs, T)` 时，注意力头之间是相互独立的，即序列中不同位置 token 的 1 号 Q 只会和 1 号 K 相乘。

# Training & Inference & Finetuning
**训练和推理区别不大**。从 [Model 章节](#model)中的 `GPT.forward` 函数可以看出，训练和推理在模型中前向传播的逻辑一致，区别就在于：
1. 输入一整个 token seq 还是只需输入最后一个 token
2. 是否需要标签 y 计算 loss

但是模型前向传播输出的只是 logits，要得到推理的 token，还要让 logits 通过 Softmax (with temperature) 和 Decoding Strategy 得到预测的 token id，最后将 token id decode 为 token（见 [Tokenizer 章节](#tokenizer)）。这个过程大有文章，不仅直接决定了模型的输出，还能通过调整 logits 的分布给生成文本打水印 [^4]。

在推理环节中，温度和 Decoding Strategy 是非常影响模型输出质量的两个因素。

Decoding Strategy 有很多 [^3]，常见的几种如：
- Greedy Search：这是最简单的策略，直接取概率最大的 token。
    - 缺点：生成较长的输出时，可能会产生高度重复的结果。
- Beam Search：每一次迭代保留 num_beams 个概率最大的 token，最终选择联合概率最大的一个序列。
    - 优点：可以保留初始 token 概率较低的高概率序列。
    - 缺点：Beam Search 是最拖慢推理速度的组件 ([Wang et al. 2024](https://arxiv.org/abs/2402.09543))。
- Top-k Sampling：按概率排序 vocab，保留概率最大的 k 个 token，归一化后就在这 k 个 token 中采样。
- Top-p Sampling：按概率排序 vocab，保留从前往后累积概率超过阈值 p 的 token，归一化后进行采样，这样可以根据分布动态调整候选 token 的数量。

温度是用来缩放 logits 的一个超参数，当使用基于概率采样的 Decoding Strategy 时，温度越小，输出多样性越低，反之温度越大，输出多样性越高。因为小于 1 的温度会拉大候选 token 之间的概率差距，使模型更倾向于采样概率较大的 token。极端地说，当最大的采样概率达到 0.999，模型几乎必定采样这个 token，因此对于同样的输入，不管尝试多少次都只会产生一样的回答。若温度较大，各个候选 token 的采样概率分布会变得相对平均，“王侯将相宁有种乎！”，概率相对较小的 token 也不是没有逆袭的机会。

nanoGPT 的推理由 `generate` 函数实现，Decoding Strategy 选择的是 Top-k Sampling：
```python
# permalink: https://github.com/karpathy/nanoGPT/blob/9755682b981a45507f6eb9b11eadef8cb83cebd5/model.py#L118-L330
class GPT(nn.Module):

    ...

    # 根据配置参数创建一个新的 GPT 实例，然后从 Hugging Face 加载对应模型的 state_dict，替换 GPT 的权重
    @classmethod
    def from_pretrained(cls, model_type, override_args=None):
        ...

    @torch.no_grad()
    def generate(self, idx, max_new_tokens, temperature=1.0, top_k=None):
        """
        Take a conditioning sequence of indices idx (LongTensor of shape (b,t)) and complete
        the sequence max_new_tokens times, feeding the predictions back into the model each time.
        Most likely you'll want to make sure to be in model.eval() mode of operation for this.
        """
        # 每轮生成一个 token，共 max_new_tokens 轮
        for _ in range(max_new_tokens):
            # if the sequence context is growing too long we must crop it at block_size
            # idx 为维度 (b, t) 的 token id
            # 若输入序列长度超过 block_size，则截取最后 block_size 个 token
            idx_cond = idx if idx.size(1) <= self.config.block_size else idx[:, -self.config.block_size:]
            # forward the model to get the logits for the index in the sequence
            # 输入模型，获取输出的 logits，维度为 (b, 1, vocab_size)
            logits, _ = self(idx_cond)
            # pluck the logits at the final step and scale by desired temperature
            # 从 logits 中取出最后一个 token 的预测分数，维度为 (b, vocab_size)
            # 根据温度参数 temperature 缩放 logits
            logits = logits[:, -1, :] / temperature
            # optionally crop the logits to only the top k options
            # 只保留 logits 中前 top_k 个最高概率的 token
            # 其他的替换为 -inf，从而使这些在 softmax 时被忽略
            if top_k is not None:
                v, _ = torch.topk(logits, min(top_k, logits.size(-1)))
                logits[logits < v[:, [-1]]] = -float('Inf')
            # apply softmax to convert logits to (normalized) probabilities
            # probs 维度为 (b, vocab_size)
            probs = F.softmax(logits, dim=-1)
            # sample from the distribution
            # 抽取一个样本，返回维度为 (b, 1)
            idx_next = torch.multinomial(probs, num_samples=1)
            # append sampled index to the running sequence and continue
            # 新生成的 token 添加到序列末尾，维度变为 (b, t+1)
            idx = torch.cat((idx, idx_next), dim=1)

        return idx
```

你可能注意到上面的 `GPT` 类还有个 `from_pretrained` 方法，这是我们想要使用 HF 上预训练模型常用的一个方法。它的原理就是从 HF 下载预训练权重，然后根据 Config 实例化一个模型类，将模型的 state_dict 替换为预训练权重。

**微调和预训练没啥区别**，只不过微调是加载了预训练模型的参数，并采用更小的学习率开始训练。查看 [nanoGPT README#finetuning](https://github.com/karpathy/nanoGPT/blob/master/README.md#finetuning) 会发现，执行训练是通过 `python train.py`，而执行微调是通过 `python train.py config/finetune_shakespeare.py`，主要区别就在于微调需要将 `init_from` 变量从 "scratch" 改为 "gpt2-xl"，从而加载 HF 上预训练模型的参数。
```python
# permalink: https://github.com/karpathy/nanoGPT/blob/9755682b981a45507f6eb9b11eadef8cb83cebd5/train.py#L149-L188
if init_from == 'scratch':
    # init a new model from scratch
    print("Initializing a new model from scratch")
    # determine the vocab size we'll use for from-scratch training
    if meta_vocab_size is None:
        print("defaulting to vocab_size of GPT-2 to 50304 (50257 rounded up for efficiency)")
    model_args['vocab_size'] = meta_vocab_size if meta_vocab_size is not None else 50304
    gptconf = GPTConfig(**model_args)
    model = GPT(gptconf)
elif init_from == 'resume':
    ...
elif init_from.startswith('gpt2'):
    print(f"Initializing from OpenAI GPT-2 weights: {init_from}")
    # initialize from OpenAI GPT-2 weights
    override_args = dict(dropout=dropout)
    model = GPT.from_pretrained(init_from, override_args)
    # read off the created config params, so we can store them into checkpoint correctly
    for k in ['n_layer', 'n_head', 'n_embd', 'block_size', 'bias', 'vocab_size']:
        model_args[k] = getattr(model.config, k)

# 接下来就是一样的 training 代码
```

# KV Cache
可惜的是 nanoGPT 没有实现 KV Cache，因此本章参考 [nanoGPTplus](https://github.com/Andrei-Aksionov/nanoGPTplus) 的实现进行理解。从代码上看，nanoGPTplus 并不是 nanoGPT 的 fork 而是完全重构了，功能更全，但相应的也更加复杂。

**推理时，有没有启用 KV Cache 的区别就在于传入模型进行前向传播（隐式执行的 `forward` 函数）的上下文范围**。由于 nanoGPT 没有实现 KV Cache，每次迭代 `for _ in range(max_new_tokens)` 都需要输入整个序列的 idx（不超过上下文长度限制的情况下），因此每次迭代都需要重新计算完整输入序列的 Q, K, V。而实现了 KV Cache 的 nanoGPTplus，传入模型的上下文范围和所处阶段有关：
- **Prefill 阶段**：若没有启用 KV Cache，或启用了 KV Cache 但处于第一次迭代且输入序列长度大于 1，应该截取最后 context_size 个 token 作为上下文，生成第一个 token
- **Decoding 阶段**：若启用了 KV Cache 且不处于第一次迭代，则只取最后一个 token 作为上下文，不断生成下一个 token

```python
# permalink: https://github.com/Andrei-Aksionov/nanoGPTplus/blob/c2eedfe406bf7c8851c393825c09280a7e329242/src/model/gpt_language_model/gpt.py#L16C1-L495
class GPTLanguageModel(nn.Module):

    ...

    @torch.no_grad()
    def generate(
        self,
        idx: Tensor,
        max_new_tokens: int,
        use_kv_cache: bool,
        temperature: float = 1.0,
        top_k_logits: Optional[int] = None,
    ) -> Tensor:
        """Generate new character after the current one.

        Parameters
        ----------
        idx : Tensor
            index of the current character
        max_new_tokens : int
            number of characters to be generated
        use_kv_cache: bool
            use key-value cache for speed up token generation; if true the number of generated tokens
            should not be larger than context size of the model
        temperature : float, optional
            The temperature determines how greedy the generative model is:
            If the temperature is low, the probabilities to sample other but the class with the highest log probability
            will be small, and the model will probably output the most correct text, but rather boring, with small
            variation.
            If the temperature is high, the model can output, with rather high probability, other words than those with
            the highest probability. The generated text will be more diverse, but there is a higher possibility of
            grammar mistakes and generation of nonsense.
            https://ai.stackexchange.com/questions/32477/what-is-the-temperature-in-the-gpt-models, by default 1.0
        top_k_logits : Optional[int], optional
            only top K logits (with the highest value) will be kept, by default None

        Returns
        -------
        Tensor
            tensor containing indices of the provided characters and newly generated

        Raises
        ------
        ValueError
            if using key-value cache and the number of tokens to generate is larger that context size of the model
        """
        if use_kv_cache and (max_new_tokens + idx.shape[-1] - 1) > self.context_size:
            msg = (
                "With kv-cache the number of new tokens should not be greater than context size of the model "
                f"plus size of initial context, but was requested '{max_new_tokens}' new tokens "
                f"with initial context of size '{idx.shape[-1]}' and '{self.context_size}' context size of the model"
            )
            logger.error(msg)
            raise ValueError(msg)
        # in the beginning initialize kv-cache either as None values if kv-cache is disabled,
        # or as empty tensors if enabled
        # 初始化空的 kv_cache 为 num_layers * (2, 0)
        # 其中每一层 Transformer 块的 KV Cache 形状为 (2, 0)
        # kv_cache = [
        #   tensor([], size=(2, 0)),
        #   ...
        #   tensor([], size=(2, 0))
        # ]
        kv_cache = (
            [torch.empty(2, 0, device=idx.device, dtype=idx.dtype) for _ in range(self.num_layers)]
            if use_kv_cache
            else None
        )
        for iteration in trange(max_new_tokens, ascii=True):
            # with kv-cache - use only last token, without - crop to the last block_size
            # also crop to the last block if idx provided with more than 1 token in the
            # beginning of token generation (start words)
            # 注意，启不启用 KV Cache 的区别在于传入模型的 context
            if not use_kv_cache or (iteration == 0 and idx.shape[-1] > 1):
                # 最后 context_size 个 token 作为上下文
                context = idx[:, -self.context_size :]
            else:
                # 最后一个 token 作为上下文
                context = idx[:, -1:]
            # get the predictions
            # 前向传播中可以用到 KV Cache
            # 不为空的 kv_cache 形状为 num_layers * (2, B, nh, T, hs)，包含了每一层的 KV Cache
            logits, kv_cache = self(
                context,
                inference=True,
                kv_cache=kv_cache if use_kv_cache else None,
            )  # (B, T, C), with inference=True -> (1, 1, C)
            # focus only on the last time step and scale by desired temperature
            logits = logits[:, -1, :] / temperature  # becomes (B, C)
            if top_k_logits:
                # topk returns rearranged tensor where the first column contains the highest values,
                # the last column - the smallest values from top K logits ...
                values, _ = torch.topk(logits, min(top_k_logits, logits.shape[-1]))
                # ... that's why we need to compare with the last column
                logits[logits < values[:, -1]] = float("-inf")  # `-1:` is to preserve dimensionality
            # apply softmax on the predictions to get probabilities
            probs = F.softmax(logits, dim=-1)  # (B, C)
            # sample from the distribution
            idx_next = torch.multinomial(probs, num_samples=1)  # (B, 1)
            # append sampled index to the running sequence
            idx = torch.cat((idx, idx_next), dim=1)  # (B, T + 1)

        return idx
```

在启用 KV Cache 且不处于第一次迭代的情况下，会同时将 KV Cache 和上一次迭代生成的一个 token 输入模型进行前向传播。传播到 `CausalSelfAttention` 模块时，由于传入的上下文长度为 1，因此实际只计算了这一个 token 的 K, V，和对应 Transformer 层的 KV Cache 合并后就可以得到完整的上下文 K, V，再进行后续的自注意力计算。

```python
# permalink: https://github.com/Andrei-Aksionov/nanoGPTplus/blob/c2eedfe406bf7c8851c393825c09280a7e329242/src/model/gpt_language_model/attention.py#L274-L436
class CausalSelfAttention(nn.Module):
    def __init__(
        self,
        embeddings_size: int,
        context_size: int,
        head_size: Optional[int],
        num_heads: int,
        bias: bool,
        dropout: float,
        *,
        is_decoder: bool,
    ) -> None:
        """Do the same as multi-head attention but with a single matrix multiplication.

        Instead of creating multiple heads and concatenating the result (in addition to creating separate matrices for
        query, key and value for each head) we can do this in a single pass with a single weight matrix.

        Parameters
        ----------
        embeddings_size : int
            size of the embeddings - the size of input of self-attention
        context_size : int
            the number of tokens that will be used during calculation attention map and
            weighted averaging of value of each token
        head_size : Optional[int]
            the size of output of self-attention;
            if not provided `head_size` will be equal to `embeddings_size` // `num_heads`, so it should be divisible
            without remainder
            注意嗷，和 nanoGPT 一样，注意力头的维度也是要除以注意力头数的
        num_heads : int
            how many self-attention heads to use
        bias : bool
            whether to use bias or not: without bias might be a bit better and faster (but it's not for sure)
        dropout : float
            how many connection between tokens are dropped during each forward pass
        is_decoder : bool
            if it's a decoder masking of 'future' tokens will be applied

        Raises
        ------
        ValueError
            if `embeddings_size` cannot be divided by `num_heads` without remainder
        """
        super().__init__()

        if not head_size:
            if embeddings_size % num_heads != 0:
                log_error(
                    "Embeddings size should be divisible by the number of heads without a residual, "
                    f"but was provided: embeddings_size={embeddings_size}; num_heads={num_heads}",
                )
            head_size = embeddings_size // num_heads

        self.embeddings_size = embeddings_size
        self.context_size = context_size
        self.head_size = head_size
        self.num_heads = num_heads
        self.bias = bias
        self.dropout = dropout
        self.is_decoder = is_decoder

        # key, query and value projections (hence `3 * ...`) for all heads in a single batch
        # 说白了就是 n_embd -> 3*n_embd 的映射，得到每个 input embed 对应的 Q K V
        self.causal_self_attention = nn.Linear(embeddings_size, 3 * self.head_size * self.num_heads, bias=self.bias)
        # output projection
        self.projection = nn.Linear(self.head_size * self.num_heads, embeddings_size, bias=self.bias)
        # regularization
        self.attention_dropout = nn.Dropout(self.dropout)
        self.projection_dropout = nn.Dropout(self.dropout)
        # triangular matrix for masking 'future' tokens
        if self.is_decoder:
            self.register_buffer("tril", torch.tril(torch.ones(self.context_size, self.context_size)))

    def forward(self, x: Tensor, kv_cache: Optional[Tensor]) -> Tensor:
        """Do multi-head attention in a single pass.

        Multiply by weight matrix -> split the result into query, key and value -> reshape each one of them
        into shape (batch, num_heads, time-steps, head_size). The rest is similar to single self-attention head
        forward pass.

        Parameters
        ----------
        x : Tensor
            input tensor of shape (batch, time-step, embedding size)
        kv_cache: Optional[Tensor]
            key-value cache, but only if not None; if None - it means that it's disabled;
            contains cache for keys and value from all previous steps

        Returns
        -------
        Tensor
            output tensor of the same shape as input: (batch, time-step, embedding size)
        """
        # notation:
        # - B  | batch
        # - T  | time-step (sequence length)
        # - C  | embeddings size
        # - hs | head size
        # - nh | number of heads

        B, T, C = x.shape  # noqa: N806

        # single pass for query, key and value; that's why we need to split into 3 parts
        # (B, T, C) -> (B, T, 3 * C) -> Q K V 分别都是 (B, T, C)
        query, key, value = self.causal_self_attention(x).split(
            self.head_size * self.num_heads,
            dim=-1,
        )  # (B, T, C) -> (B, T, 3 * hs * nh) -> (B, T, hs * nh)

        # transform (B, T, nh * hs) -> (B, nh, T, hs) so it's similar to multi-head attention
        # 按注意力头数分割 Q, K, V 的嵌入维度
        key = key.view(B, T, self.num_heads, self.head_size).transpose(1, 2)  # (B, nh, T, hs)
        query = query.view(B, T, self.num_heads, self.head_size).transpose(1, 2)  # (B, nh, T, hs)
        value = value.view(B, T, self.num_heads, self.head_size).transpose(1, 2)  # (B, nh, T, hs)

        # 复用 KV Cache：如果提供了 KV Cache，就将当前的 K 和 V 合并进去
        # kv_cache 变量是一个 (2, B, T, head_size) 的 Tensor
        if kv_cache is not None:
            # 分离出 K Cache 和 V Cache
            key_cached, value_cached = kv_cache.unbind(dim=0)  # (2, B, T, head_size) -> 2 * (B, T, head_size)
            # 将当前的 K 和 V 合并进去
            key = torch.cat((key_cached, key), dim=-2)  # (B, cache + T, head_size)
            value = torch.cat((value_cached, value), dim=-2)  # (B, cache + T, head_size)

        # to obtain attention scores first do dot product of query and key
        attention_scores = query @ key.mT  # (B, nh, T, hs) x (B, nh, hs, T) -> (B, nh, T, T)

        # In order to preserve 1 unit variance of the dot product of two vectors
        # we need to divide by square root of the features size (in our case - attention head size)
        # We need it to make sure that the values after softmax are well spread out, otherwise in worst
        # case scenario the values after the softmax will converge to one-hot encoding (like [0, 0, 1]) and
        # that will mean that the attention will be on a single (or couple of) tokens, and we want it to be
        # spread out (like [0.2, 0.1, 0.7])
        # we want to aggregate information not from a single node
        attention_scores /= math.sqrt(key.shape[-1])  # (B, nh, T, T)

        # if it's a decoder we need to mask 'future' tokens with '-inf' value
        if self.is_decoder:
            # [0.9, -0.6, 0.3] -> [0.9, -inf, -inf]
            # [0.1, 0.5, -0.1] -> [0.1, 0.5, -inf]
            # [0.1, 0.2, 0.3]  -> [0.1, 0.2, 0.3]
            # and after softmax `-inf` becomes 0
            # this doesn't allow current token communicate with future ones

            # TODO: do I correctly apply masking
            # w : (batch, head, q_seq_length, kv_seq_length)   # noqa: ERA001
            # w = torch.matmul(q, k) # noqa: ERA001
            # if self.scale:
            #     w = w / math.sqrt(v.size(-1))  # noqa: ERA001
            # nd, ns = w.size(-2), w.size(-1): ERA001
            # b = self.bias[:, :, ns-nd:ns, :ns]: ERA001
            # TODO: 注意力掩码的形状应该是 (B, nh, cache + T, cache + T)
            attention_scores = attention_scores.masked_fill(self.tril[:T, :T] == 0, float("-inf"))  # (B, nh, T, T)

        # since we want to do weighted averaging we need to transform attention scores into range [0, 1]
        # and sum of all scores should be equal to 1; softmax is a good tool for it
        attention_scores = F.softmax(attention_scores, dim=-1)  # (B, nh, T, T)

        # randomly prevent some nodes from communicating, some of theme randomly are set to zero
        # helps prevent overfitting
        attention_scores = self.attention_dropout(attention_scores)  # (B, nh, T, T)

        # perform the weighted aggregation of the values
        output = attention_scores @ value  # (B, nh, T, T) x (B, nh, T, hs) -> (B, nh, T, hs)
        # re-assemble all head outputs side by side
        output = output.transpose(1, 2).reshape(B, T, self.head_size * self.num_heads)  # (B, T, hs * nh)
        # output projection
        output = self.projection(output)  # (B, T, C)
        return (
            self.projection_dropout(output),  # (B, T, C)
            None if kv_cache is None else torch.stack((key, value)),  # None | # (2, B, nh, T, hs)
        )
```

但是这个实现还是有点问题，当前这个 token 和之前上下文的 KV Cache 合并后上下文长度变长了，注意力掩码的形状应该与该长度匹配 [^5]。

[^1]: https://stackoverflow.com/q/44524901
[^2]: https://pytorch.org/docs/stable/generated/torch.matmul.html
[^3]: https://huggingface.co/docs/transformers/generation_strategies
[^4]: https://huggingface.co/docs/transformers/generation_strategies#watermarking
[^5]: https://huggingface.co/docs/transformers/kv_cache#under-the-hood-how-cache-object-works-in-attention-mechanism
[^6]: https://spaces.ac.cn/archives/9009
