---
title: "基座模型到 Agent 的最后一公里：关于提示词的真相"
tags:
- LLM
draft: true
# weight: 1
# date: 2024-09-01
# cover:
#     image: /images/test.png
#     caption: "Title<br>(Image source: [](link))"
#     hiddenInSingle: false
# summary: "Summary on home page"
---
模型是怎么记住对话上下文的？

输入模型的提示词到底是什么样的，真的是你在聊天输入框键入的文字吗？

提示词为什么还分 System Prompt 和 User Prompt，System Prompt 又为什么优先级更高呢？

在我初学 LLM 时，这些问题时常困扰着我，直到我动手写了自己的第一个 Agent [^1]，才发现从基座模型到 Chat 应用（ChatGPT、Claude、通义千问等）还有一段距离，包括微调（RLHF、Instruct Following）、Agent 开发等。首先需要明确的是，很多我们现在看起来理所当然的功能其实并不是模型本身所具有的，LLM 基座模型无非是一个接受一段输入文字、输出一段文字的黑盒函数，其他的功能都是在此基础上进行开发的，或改变输入——构造提示词，或利用输出——调用工具，我们把这样的应用叫做 Agent。


special token

可以参考 https://huggingface.co/blog/llama2#how-to-prompt-llama-2

# Chat

# Function Calling

[^1]: https://github.com/figuremout/autocmd
