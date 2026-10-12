---
title: "理解大模型推理服务中的并行：DP、TP、PP 与 EP"
date: 2026-10-12 00:00:00 +0800
categories: [LLM, Inference]
tags: [LLM Serving, Data Parallelism, Tensor Parallelism, Pipeline Parallelism, Expert Parallelism]
math: true
toc: true
image:
  path: /assets/img/posts/2026-10-12-llm-parallelism/01-data-parallel-serving.svg
  alt: 大模型推理服务中的数据并行请求路由
---

当一个大语言模型无法只靠一张 GPU 高效服务时，我们会遇到几种名字相似、切分对象却完全不同的并行策略。理解它们最重要的不是先记通信算子，而是先回答一个问题：

> 这一次究竟把什么切开了？

本文只讨论推理服务中最常见的四种策略：数据并行（Data Parallelism，DP）、张量并行（Tensor Parallelism，TP）、流水线并行（Pipeline Parallelism，PP）和专家并行（Expert Parallelism，EP）。

需要提前说明的是，这些概念不是由四篇论文在同一时期分别“发明”的。数据并行和模型并行的思想出现得很早；本文选择的是将相应方法系统化、并对今天的大模型系统影响较大的代表性论文，然后把论文中的训练设计转换到推理服务场景中理解。

| 策略 | 切分对象 | 一个请求如何使用多卡 | 推理服务中的主要目标 |
|---|---|---|---|
| DP | 请求和模型副本 | 请求只进入一个副本 | 提高并发量和总吞吐 |
| TP | 一层中的权重张量 | 每层都由 TP 组共同计算 | 分摊模型权重，降低部分计算延迟 |
| PP | Transformer 层 | 请求依次经过不同 stage | 分摊模型深度和权重 |
| EP | MoE 专家 | token 被发送到选中的专家设备 | 分摊巨量专家参数 |

## 一、数据并行：切分请求

数据并行的思想早于现代大语言模型。以深度学习系统为例，2012 年的 [DistBelief](https://proceedings.neurips.cc/paper_files/paper/2012/hash/6aca97005c68f1206823815f66102863-Abstract.html) 已经使用多个模型副本并行处理数据；后来的分布式训练系统又进一步发展了同步或异步梯度更新方式。

但到了推理服务阶段，我们不再计算和同步梯度。这里的数据并行可以先简单理解为：

> 部署多份模型副本，然后把不同请求分配给不同副本。

![推理服务中的数据并行](/assets/img/posts/2026-10-12-llm-parallelism/01-data-parallel-serving.svg)

_图 1：推理服务中的数据并行。不同请求被路由到不同模型副本；每个副本独立维护本地的 KV Cache。本文绘制。_

假设一份模型可以放进一张 GPU，三张 GPU 可以分别保存三份完整权重：

```text
请求 A ──→ GPU 0：完整模型
请求 B ──→ GPU 1：完整模型
请求 C ──→ GPU 2：完整模型
```

三个请求互不依赖，可以同时完成 prefill 和 decode。副本之间通常不交换 token 的隐藏状态，也不执行训练中的梯度 AllReduce。

### 1. 一个副本不一定只是一张 GPU

如果模型需要两张 GPU 才能放下，可以先用 TP 或 PP 组成一个模型副本，再复制这个多卡副本：

```text
副本 0 = GPU 0、1，TP=2
副本 1 = GPU 2、3，TP=2
副本 2 = GPU 4、5，TP=2
副本 3 = GPU 6、7，TP=2
```

此时一共有四个数据并行副本，所以 $DP=4$；每个副本内部由两张卡协作，所以 $TP=2$。

### 2. 为什么同一个请求通常不能随意换副本

Prefill 产生的 KV Cache 保存在执行该请求的副本中。后续每一次 decode 都需要读取这份缓存，因此调度器通常要把该请求继续发送到原来的副本。

多轮对话也是类似的：如果系统希望复用上一轮的 KV Cache，就需要保持请求亲和性，或者显式迁移、共享缓存。否则换到另一个副本后，新副本只能重新计算上下文。

### 3. DP 解决什么问题

DP 主要增加：

- 能同时服务的请求数；
- continuous batching 可以组织的活跃序列数；
- 整个服务集群的 token 吞吐量。

它通常不能降低单个请求的计算量，也不能帮助一份过大的模型放进单卡。模型放不下时，要使用下面的 TP 或 PP。

## 二、张量并行：切分一层中的矩阵

[Megatron-LM](https://arxiv.org/abs/1909.08053) 系统化了 Transformer 的层内模型并行。它不把整个 Transformer 层分给某张卡，而是把同一层里的权重矩阵切开，让多张 GPU 共同完成这一层。

![Megatron-LM 中的自注意力张量并行](/assets/img/posts/2026-10-12-llm-parallelism/02-tensor-parallel-self-attention.png)

_图 2：Megatron-LM 对自注意力进行张量并行。不同设备计算不同的 Q、K、V 分片和注意力头，输出投影后再合并。图源：Megatron-LM Figure 3(b)。_

### 1. 从矩阵乘法理解 TP

以 Query 投影为例：

$$
Q=XW_Q
$$

如果把 $W_Q$ 沿列方向切成两部分：

$$
W_Q=[W_Q^{(0)},W_Q^{(1)}]
$$

那么两张 GPU 可以分别计算：

$$
Q^{(0)}=XW_Q^{(0)},\qquad Q^{(1)}=XW_Q^{(1)}
$$

最后拼接得到：

$$
Q=[Q^{(0)},Q^{(1)}]
$$

$W_K$ 和 $W_V$ 可以采用相同的切分。因此，不同 GPU 可以负责不同的注意力头，在本地完成各自的 $QK^T$、softmax 和 Value 加权。输出投影再按另一个方向切分，并通过 AllReduce 等通信操作合并部分结果。

MLP 也遵循类似原则：第一层通常采用列并行，第二层采用行并行，使中间激活可以直接留在各自的 GPU 上，减少一次不必要的同步。

### 2. 一个 token 怎样经过 TP 组

假设 $TP=4$，一个 decode token 进入某个 Transformer 层时：

```text
                 ┌─ GPU 0：部分 Attention heads / MLP channels
token hidden ────┼─ GPU 1：部分 Attention heads / MLP channels
                 ├─ GPU 2：部分 Attention heads / MLP channels
                 └─ GPU 3：部分 Attention heads / MLP channels
                                      │
                                      ▼
                              集合通信并合并结果
```

所以 TP 和 DP 的区别非常明显：

- DP 中，一个请求只选择一个副本；
- TP 中，一个请求在每一层都需要 TP 组内的所有 GPU 共同参与。

### 3. TP 如何影响 KV Cache

注意力头被切分后，各 GPU 通常只保存自己负责的 K/V 头。例如共有 32 个 KV heads、$TP=4$，每张 GPU 可以保存其中 8 个 head 的 KV Cache。

这说明 TP 不仅切分模型权重，也可能沿 head 维度分摊 KV Cache。但每张卡保存的是所有 token 的部分 heads，而不是一段 token 对应的完整 K/V。

### 4. Prefill 和 Decode 中的差异

Prefill 一次处理很多 token，矩阵乘法规模较大，多张 GPU 更容易被充分利用。Decode 中每条活跃序列每轮通常只有一个新 token，矩阵计算变小，但每层需要的通信没有消失。

因此 TP 并不是越大越好：

- TP 太小，模型可能放不下，计算也可能不足；
- TP 太大，每张 GPU 的计算变少，AllReduce 延迟反而可能占主导；
- TP 通常更适合放在具有 NVLink 或 NVSwitch 的单个节点内部。

TP 的核心可以概括为：

> 权重少存一点，计算少做一点，但一个 token 的每一层都需要多卡通信。

## 三、流水线并行：切分 Transformer 层

[GPipe](https://arxiv.org/abs/1811.06965) 将顺序网络划分为多个连续 stage，并用 micro-batch 填充流水线。论文主要讨论训练，但“按层切分模型”的基本结构同样适用于推理。

![GPipe 的流水线并行时序](/assets/img/posts/2026-10-12-llm-parallelism/03-pipeline-parallelism.png)

_图 3：GPipe 的 micro-batch 流水线。$F$ 表示前向计算，$B$ 表示反向计算，中间空白区域是 pipeline bubble。推理服务只保留 $F$ 所代表的前向路径。图源：GPipe Figure 2(c)。_

### 1. 模型怎样按层切开

假设一个模型有 40 层、$PP=4$：

```text
Stage 0 / GPU 0：第  1～10 层
Stage 1 / GPU 1：第 11～20 层
Stage 2 / GPU 2：第 21～30 层
Stage 3 / GPU 3：第 31～40 层
```

一个请求必须依次经过四个 stage：

```text
输入
  ↓
Stage 0 ──隐藏状态──→ Stage 1 ──→ Stage 2 ──→ Stage 3
                                                       ↓
                                                     logits
```

相邻 stage 传递的是分区边界上的隐藏状态，而不是每层都进行 TP 式的 AllReduce。这使 PP 在跨节点带宽相对较低时有一定吸引力。

### 2. KV Cache 放在哪里

每个 stage 只保存自己负责的模型层，因此也只保存这些层对应的 KV Cache：

```text
GPU 0：第  1～10 层的 KV Cache
GPU 1：第 11～20 层的 KV Cache
GPU 2：第 21～30 层的 KV Cache
GPU 3：第 31～40 层的 KV Cache
```

对一条序列来说，完整 KV Cache 被按层分散在流水线的各个 stage 中。

### 3. 推理为什么也会出现 bubble

如果只有一个请求，Stage 0 完成后才能把隐藏状态交给 Stage 1，后面的设备在开始时都处于等待状态。一个 token 到达最后一个 stage 后，下一个自回归 token 才能真正开始完整计算。

为了提高利用率，服务系统需要让多个请求或 micro-batch 同时处于不同 stage：

```text
时间 ──────────────────────────────→

Stage 0：A ─ B ─ C ─ D
Stage 1：    A ─ B ─ C ─ D
Stage 2：        A ─ B ─ C ─ D
Stage 3：            A ─ B ─ C ─ D
```

并发请求足够多时，四个 stage 可以同时工作；请求较少或各 stage 计算量不均衡时，就会产生 bubble。

因此 PP 的主要作用通常是让超大模型能够跨 GPU、跨节点放置，并在高并发下提高总吞吐。它一般不会缩短单个请求必须经过的网络深度，单请求延迟还可能因为 stage 间通信而增加。

PP 的核心可以概括为：

> 每张卡只负责一段层，但一个 token 必须顺序走完整条流水线。

## 四、专家并行：切分 MoE 专家

专家并行只适用于 Mixture-of-Experts（MoE）模型。早期稀疏 MoE 工作已经使用门控网络选择专家；[GShard](https://arxiv.org/abs/2006.16668) 则展示了如何把 Transformer 中的 MoE 层切分到大量设备上。

![GShard 中的 MoE 专家切分](/assets/img/posts/2026-10-12-llm-parallelism/04-expert-parallelism.png)

_图 4：GShard 用 MoE 层替换部分 FFN，并把不同专家分布到不同设备。token 通过 All-to-All Dispatch 发送到专家，计算后再通过 All-to-All Combine 返回。图源：GShard Figure 3。_

### 1. 专家本质上是什么

一个 MoE 层包含多个独立 FFN：

$$
\operatorname{FFN}_e(x)
=W_{2,e}\,\sigma(W_{1,e}x)
$$

Router 根据 token 的隐藏状态 $x$ 计算门控分数，只选择 Top-$k$ 个专家：

$$
y=\sum_{e\in\operatorname{TopK}(x)}g_e(x)\operatorname{FFN}_e(x)
$$

这意味着模型虽然可能拥有大量专家参数，但一次前向计算只激活其中很少一部分。

### 2. EP 怎样分布专家

假设有 8 个专家、4 张 GPU：

```text
GPU 0：Expert 0、1
GPU 1：Expert 2、3
GPU 2：Expert 4、5
GPU 3：Expert 6、7
```

某一层的 token 路由结果可能是：

```text
token A → Expert 1、5
token B → Expert 2、7
token C → Expert 1、3
```

于是一次 MoE 层的推理过程为：

```text
token 隐藏状态
      ↓
Router 选择 Top-k 专家
      ↓
All-to-All Dispatch：把 token 发到专家所在 GPU
      ↓
各专家执行 FFN
      ↓
All-to-All Combine：把专家输出送回原来的 GPU
      ↓
按照门控权重合并
```

这里移动的不是完整请求，而是当前 MoE 层中的 token 表示。下一个 MoE 层会重新路由，同一个 token 不一定再次访问相同专家。

### 3. 为什么 Decode 中的 EP 更难利用

Prefill 中 token 较多，路由器通常能给不同专家分配较多工作。Decode 每条序列每轮只有一个新 token，如果 batch 较小，就可能出现：

- 某些专家收到很多 token；
- 某些专家没有收到 token；
- 每张卡实际计算量很小；
- All-to-All 通信时间超过专家计算时间。

因此 MoE 推理通常非常依赖动态 batching、负载均衡和高效的 token dispatch 实现。

### 4. EP 与 KV Cache 的关系

MoE 一般替换 Transformer 中的 FFN，而 KV Cache 来自 Attention 层。因此 EP 直接切分的是专家权重，不会因为专家切分就自然地切分 KV Cache。

实际 MoE 服务仍可能同时使用 TP：TP 负责 Attention 和共享稠密层，EP 负责把不同专家分散到不同 GPU。

EP 的核心可以概括为：

> 专家权重固定在不同设备上，token 根据每一层的路由结果在设备之间移动。

## 五、把四种并行方式放在一起

四种并行发生在不同层次：

```text
服务集群
  └─ DP：复制多个模型副本，按请求分流
       └─ PP：一个副本按 Transformer 层切成多个 stage
            └─ TP：一个 stage 内，把每层矩阵切给多张 GPU
                 └─ EP：遇到 MoE 层时，再按专家路由 token
```

如果这些维度彼此正交，总 GPU 数可以近似写成：

$$
N_{GPU}=DP\times TP\times PP\times EP
$$

这里的 $EP$ 表示一个专家并行组的规模，不是模型中的专家总数。

### 示例：16 张 GPU 部署一个稠密模型

如果模型需要 $TP=2$、$PP=2$ 才能放下，那么一个模型副本需要：

$$
TP\times PP=2\times2=4\text{ GPUs}
$$

16 张 GPU 可以部署四个副本：

$$
DP=\frac{16}{2\times2}=4
$$

每个请求先由 DP 路由到一个副本；进入副本后，它依次经过两个 PP stage；每个 stage 内再由两张 TP GPU 共同计算。

### 最后用四句话记忆

> DP：不同请求去不同模型副本。  
> TP：同一层的矩阵由多张卡共同计算。  
> PP：不同 Transformer 层放在不同 stage。  
> EP：不同 MoE 专家放在不同设备，token 按路由移动。

选择策略时可以先按下面的顺序判断：

1. 一份模型是否能放进一张 GPU？如果不能，考虑 TP 或 PP；
2. 单个副本已经能运行后，是否需要更多并发？如果需要，增加 DP；
3. 模型是否为 MoE？只有 MoE 模型才需要考虑 EP；
4. 通信是在节点内还是跨节点？TP 更依赖高速互联，PP 只在 stage 边界传递隐藏状态，EP 则依赖高效 All-to-All；
5. 服务更看重单请求延迟还是总吞吐？增大并行规模并不总能同时改善二者。

---

参考资料：

1. Dean et al. [Large Scale Distributed Deep Networks](https://proceedings.neurips.cc/paper_files/paper/2012/hash/6aca97005c68f1206823815f66102863-Abstract.html), NeurIPS 2012.
2. Shoeybi et al. [Megatron-LM: Training Multi-Billion Parameter Language Models Using Model Parallelism](https://arxiv.org/abs/1909.08053), 2019.
3. Huang et al. [GPipe: Efficient Training of Giant Neural Networks using Pipeline Parallelism](https://arxiv.org/abs/1811.06965), NeurIPS 2019.
4. Lepikhin et al. [GShard: Scaling Giant Models with Conditional Computation and Automatic Sharding](https://arxiv.org/abs/2006.16668), 2020.
5. Narayanan et al. [Efficient Large-Scale Language Model Training on GPU Clusters Using Megatron-LM](https://arxiv.org/abs/2104.04473), SC 2021.
