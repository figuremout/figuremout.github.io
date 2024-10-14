---
title: 'A Brief Survey on LLM for Sequential Recommendation'
tags:
- LLM
- RecSys
date: 2024-06-24
draft: true
---

**Contrastive loss (Chopra et al. 2005)** is one of the earliest training objectives used for deep metric learning in a contrastive fashion.

# What is Sequential Recommendation?

# Why LLM for Rec?

# Why Sequential Recommendation?

# Sequential Recommendation Challenges

data sparsity:


# Methods

# Related Works
**SASRec** **BERT4Rec** There are many ([Wang et al., 2024](#Wang_et_al.,_2024)) ways to modify an image [[5]](#Wang_et_al.,_2024) while retaining its semantic meaning. We can use any one of the following augmentation or a $E=mc^2$ composition of multiple operations. **VQ-Rec**[^test]

**Contrastive Learning**, **SimCSE**, **CT4Rec**
# Thoughts

# Datasets
McAuley-Lab/Amazon-Reviews-2023

Most work take the paradigm that 

splitting: leave-one-out strategy

Here is a simple EDA (Exploratory Data Analysis) of Amazon-Reviews-2023 and MovieLens 1M dataset.

<script src="https://gist.github.com/figuremout/598d7cfb7e610e81881cdad9f09f5a4a.js"></script>

# Contrastive Learning Objectives

# Evaluation

# Citation
Cited as:
```
Weng, Lilian. (May 2021). Contrastive representation learning. Lil’Log. https://lilianweng.github.io/posts/2021-05-31-contrastive/.
```

Or via BibTex
```bibtex
@article{weng2021contrastive,
  title   = "Contrastive Representation Learning",
  author  = "Weng, Lilian",
  journal = "lilianweng.github.io",
  year    = "2021",
  month   = "May",
  url     = "https://lilianweng.github.io/posts/2021-05-31-contrastive/"
}
```

# References
- Devlin J, Chang M W, Lee K, et al. Bert: Pre-training of deep bidirectional transformers for language understanding[J]. arXiv preprint arXiv:1810.04805, 2018.
- Radford A, Narasimhan K, Salimans T, et al. Improving language understanding by generative pre-training[J]. 2018.
- Boz A, Zorgdrager W, Kotti Z, et al. Improving Sequential Recommendations with LLMs[J]. arXiv preprint arXiv:2402.01339, 2024.
- Neelakantan A, Xu T, Puri R, et al. Text and code embeddings by contrastive pre-training[J]. arXiv preprint arXiv:2201.10005, 2022.
- <span id="Wang_et_al.,_2024"/>Wang H, Liu X, Fan W, et al. Rethinking Large Language Model Architectures for Sequential Recommendations[J]. arXiv preprint arXiv:2402.09543, 2024.
- BehnamGhader P, Adlakha V, Mosbach M, et al. Llm2vec: Large language models are secretly powerful text encoders[J]. arXiv preprint arXiv:2404.05961, 2024.
- Gao T, Yao X, Chen D. Simcse: Simple contrastive learning of sentence embeddings[J]. arXiv preprint arXiv:2104.08821, 2021.
- Lee C, Roy R, Xu M, et al. NV-Embed: Improved Techniques for Training LLMs as Generalist Embedding Models[J]. arXiv preprint arXiv:2405.17428, 2024.
- Harte J, Zorgdrager W, Louridas P, et al. Leveraging large language models for sequential recommendation[C]//Proceedings of the 17th ACM Conference on Recommender Systems. 2023: 1096-1102.
- Wu L, Zheng Z, Qiu Z, et al. A survey on large language models for recommendation[J]. arXiv preprint arXiv:2305.19860, 2023.
- Lin J, Dai X, Xi Y, et al. How can recommender systems benefit from large language models: A survey[J]. arXiv preprint arXiv:2306.05817, 2023.
- Kang W C, McAuley J. Self-attentive sequential recommendation[C]//2018 IEEE international conference on data mining (ICDM). IEEE, 2018: 197-206.
- Sun F, Liu J, Wu J, et al. BERT4Rec: Sequential recommendation with bidirectional encoder representations from transformer[C]//Proceedings of the 28th ACM international conference on information and knowledge management. 2019: 1441-1450.
- Hou Y, He Z, McAuley J, et al. Learning vector-quantized item representation for transferable sequential recommenders[C]//Proceedings of the ACM Web Conference 2023. 2023: 1162-1171.
- Radford A, Wu J, Child R, et al. Language models are unsupervised multitask learners[J]. OpenAI blog, 2019, 1(8): 9.
- Brown T, Mann B, Ryder N, et al. Language models are few-shot learners[J]. Advances in neural information processing systems, 2020, 33: 1877-1901.
- Zhu F, Wang Y, Chen C, et al. Cross-domain recommendation: challenges, progress, and prospects[J]. arXiv preprint arXiv:2103.01696, 2021.
- Wang S, Hu L, Wang Y, et al. Sequential recommender systems: challenges, progress and prospects[J]. arXiv preprint arXiv:2001.04830, 2019.
- Li B, Zhou H, He J, et al. On the sentence embeddings from pre-trained language models[J]. arXiv preprint arXiv:2011.05864, 2020.

