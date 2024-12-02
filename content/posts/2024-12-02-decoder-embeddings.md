+++
title = 'Recent Trends with Text Embeddings: Decoder-Only LLMs"'
date = 2024-12-02T06:34:30-05:00
draft = false
+++

## Introduction

Text embedding models have been dominated by fine-tuned BERT-style bidirectional encoders, like [E5](https://yuvalmerhav.com/posts/e5/), which offered state-of-the-art (SOTA) performance for many sequence-level tasks. However, currently, many of the top performing models on the [MTEB leaderboard](https://huggingface.co/spaces/mteb/leaderboard), a text embedding benchmark for various tasks (retrieval, clustering, classification, etc.) are decoder-only language models at the ~7B scale. These models leverage larger context windows and pretraining on web-scale data.

This post reviews the papers of the following models:

- [E5-mistral-7b-instruct](https://huggingface.co/intfloat/e5-mistral-7b-instruct) 
- [NV-Embed-7b](https://huggingface.co/nvidia/NV-Embed-v2) (currently ranked #1 on MTEB, from Nvidia)
- [GritLM-7b](https://github.com/ContextualAI/gritlm)
- [LLM2Vec](https://github.com/McGill-NLP/llm2vec)

## Spotlight on E5-Mistral

The core ideas across these top models are similar. Let’s start with E5-mistral, introduced in [this paper](https://arxiv.org/pdf/2401.00368). They start with the pretrained Mistral-7b LLM and fine-tune it with LoRA. Models such as Mistral went through extensive auto-regressive pre-training at web scale, which enable them to acquire good text representations, and only minimal fine-tuning is required to transform them into effective embedding models. This contrasts with smaller text encoders, which rely heavily on weakly-supervised contrastive pre-training. This often involves gathering extensive datasets of positive and negative text pairs, typically generated through self-supervised methods, before fine-tuning for specific downstream tasks.

### Fine-Tuning Details

The fine-tuning step is the standard contrastive learning using cosine similarity, where positive pairs are paired with random in-batch negatives. Since it's a decoder-only LLM, they append an [EOS] token to the end of the query and document. These sequences are fed into the LLM, and the final-layer [EOS] vector is extracted to represent the query and document embeddings. With a typical causal attention that next token prediction models utilize (each token can only attend to previous tokens in the sequence), the [EOS] embedding is used since it can attend to all prior tokens in the context (similar to how the [CLS] token is used in encoders). An alternative approach would involve altering the attention mechanism (as we will see later).

The paper also goes in detail on how they generated synthetic data with GPT for training per task with instructions which I’m skipping here. (it’s pretty interesting and worth a read)

## Bi-Directional Attention

One key difference of the other works that followed-up is that all modify Mistral's causal attention to bidirectional during contrastive training. This is more intuitive for sequence-level tasks like embeddings where each token can attend to any other token in the context. All studies report improved embedding representations as a result. Another key difference is how embeddings are extracted. Instead of relying on the final token representation, these models apply various pooling techniques, such as mean pooling or weighted pooling based on attention scores, to generate more robust embeddings. NV-Embed, developed by Nvidia, stands out by being exclusively fine-tuned on public datasets without the use of synthetic data. 

## GritLM: A Hybrid Approach to Generation and Embedding

[GritLM](https://arxiv.org/pdf/2402.09906) differs from the other works by combining generation and embedding tasks in a single model. While I’m not a fan of this idea (will explain why in a bit), it’s worth examining. Typically, fine-tuning a decoder-only LLM for embeddings significantly harms the generation performance. GritLM addresses this by adopting a multi-task learning strategy, fine-tuning the model for both tasks with a hybrid objective that blends bidirectional representation learning and causal generative training.

{{<figure src="/gritlm/grit_architecture.png" alt="gritlm-architecture">}}

The impressive outcome is that GritLM achieves comparable performance on both tasks to models fine-tuned for each task individually. There is an advantage of having a single model for both tasks. The primary motivation behind this unified approach is to enable more efficient inference, particularly in scenarios like retrieval-augmented generation (RAG). However, this comes at the cost of more computationally expensive training, increased model complexity and storage cost. 

### Faster RAG with GritLM

In standard RAG workflows, the process involves: 

1. Embedding documents and indexing them for retrieval
2. Retrieving relevant documents for a query and feeding both the query and retrieved documents to the LLM for answer generation

GritLM introduces a unique optimization: during the indexing phase, when embedding each document, instead of storing document text, the key-value states of the model are cached (a form of self-attention caching similar to prompt caching). This enables the system to skip forward passes for indexed documents during the generation inference. However, this approach lacks the flexibility of traditional RAG, where you can freely structure a prompt with instructions, followed by a list of documents and a query, or vice versa. In GritLM’s setup, the query must always come after the documents, as the cached states are tied to the documents and cannot dynamically attend to new instructions or restructured input sequences. Also, for large LLMs, storing key-value states is significantly more expensive than storing typical document texts. (They also tested other forms of caching but found that document caching works the best). 

### Limitations

While these works are interesting and require much less data to perform very well on a variety of sequence level tasks (retrieval, clustering, classification, etc), there are a couple of limitations related to their adoption:

1. Inference and storage costs: The reliance on bigger models increases inference costs. Embeddings with 4096 dimensions further compound storage challenges. (Results with smaller embedding dimensions would be valuable for comparison)
2. Specifically for GritLM: Even less practical due to its higher storage costs and limited flexibility

As far as evaluation goes, the MTEB benchmark focuses predominantly on shorter sequences, making it challenging to evaluate on long documents that these models support. There's also a potential for test set contamination since Mistral was trained on extensive, unspecified datasets derived from the web, which may lead to inflated performance metrics.


### Final Thoughts

For single-task use cases, smaller encoders remain highly competitive and practical. While they require more data and careful fine-tuning, synthetic data generation with strong LLMs has made this process both easier and more cost-effective. In my view, the best trade-off is still a smaller, efficient model for embeddings paired with a larger, more powerful model for generation. That said, for use cases requiring strong performance across a variety of tasks, larger-scale LLMs show great promise as versatile embedding models. 


