---
title: "理解大模型推理中的 GPU 通信：从集合通信到并行策略"
date: 2026-10-14 00:00:00 +0800
categories: [LLM, Inference]
tags: [LLM Serving, GPU Communication, Collective Communication, Tensor Parallelism, Pipeline Parallelism, Expert Parallelism]
math: true
toc: true

---

在多 GPU 推理服务中，张量并行、流水线并行和专家并行描述的是模型或计算如何被切分；Send/Recv、AllReduce、AllGather 和 AllToAll 等概念，描述的则是切分之后 GPU 之间如何交换数据。

这篇主要用来理解常见通信操作的数据流向，以及它们在不同并行策略中的作用。前两部分先建立通信原语本身的直觉，后面再结合 Megatron-LM、GPipe 和 GShard 中的结构图理解实际使用场景。

| 通信方式 | 直观理解 | 推理服务中的典型位置 |
|---|---|---|
| Send / Recv | 指定两张 GPU 点对点传递数据 | Pipeline Parallel 的相邻 stage |
| Broadcast | 一张 GPU 将同一份数据发给所有 GPU | 参数、配置或控制信息分发 |
| Reduce | 多张 GPU 的数据归约到一张 GPU | 局部结果汇总 |
| AllReduce | 归约后让所有 GPU 都得到结果 | Tensor Parallel 的局部结果合并 |
| AllGather | 收集所有分片，并让所有 GPU 都得到完整结果 | 分片激活或参数重组 |
| ReduceScatter | 归约数据，同时把结果切分给各 GPU | 分片计算与通信优化 |
| AllToAll | 每张 GPU 分别向其他 GPU 发送不同数据 | Expert Parallel 的 token 路由 |

## 一、先看通信完成后数据变成了什么

![TACCL Figure 2](/assets/img/posts/2026-10-14-gpu-communication/01-taccl-collective-primitives.png)

_图 1：AllGather、AllToAll 与 AllReduce 的初始数据和通信结果。图源：TACCL Figure 2。_

[TACCL](https://www.usenix.org/conference/nsdi23/presentation/shah) 的 Figure 2 给出了三种集合通信最直接的对比。每一组上方是 4 张 GPU 的初始缓冲区，下方是通信结束后的缓冲区。

- AllGather：每张 GPU 最终都拿到所有 GPU 的数据分片。
- AllToAll：每张 GPU 把不同数据块发给不同 GPU，最终收到来自所有 GPU、但目的地属于自己的数据。
- AllReduce：相同位置的数据先做归约，图中采用求和，然后让所有 GPU 都得到相同的归约结果。

## 二、这些集合通信之间有什么关系

![ForestColl Figure 4](/assets/img/posts/2026-10-14-gpu-communication/02-forestcoll-collective-relations.png)

_图 2：Broadcast、Reduce、ReduceScatter、AllGather 与 AllReduce 的关系。图源：ForestColl Figure 4。_

[ForestColl](https://arxiv.org/abs/2402.06787) 的 Figure 4 把几种容易混淆的集合通信放在了一张图中。最值得记住的是：

```text
AllReduce = ReduceScatter + AllGather
AllReduce = Reduce + Broadcast
```

Broadcast 可以理解为一个根节点把完整数据发给所有节点；Reduce 则是所有节点把数据归约到一个根节点。图中的 `reversed` 表示可以通过反转通信方向和调度来构造另一种操作，并不是数学意义上的逆运算。

### 1. Scatter 与 Gather 放在哪里理解

Gather 是让多个 GPU 把各自的数据分片集中到一个根 GPU；Scatter 则是相反方向，由根 GPU 把不同分片发送给不同 GPU。

它们是基础的单根通信操作。现代 LLM 并行论文更常直接讨论 AllGather、ReduceScatter、AllReduce 或 AllToAll，因为这些操作才是张量并行、分片参数和 MoE 主计算路径中的常见通信方式。

## 三、Tensor Parallel 中的 AllReduce

![Megatron-LM Figure 3b](/assets/img/posts/2026-10-14-gpu-communication/03-megatron-tp-allreduce.png)

_图 3：Megatron-LM 中自注意力层的张量并行。图源：Megatron-LM Figure 3(b)。_

[Megatron-LM](https://arxiv.org/abs/1909.08053) 将同一层中的注意力头和权重矩阵切到多个 tensor-parallel rank。图中可以先把上下两个分支理解为两张 GPU：它们分别计算局部的注意力结果和输出投影。

推理前向时，每张 GPU 先得到自己的局部结果 `Z1`、`Z2`；绿色的 `g` 执行 AllReduce，把这些局部结果求和，使两张 GPU 都得到相同的完整输出 `Z`。

这就是 TP 通信频繁的原因：它切分的是 Transformer 层内部的矩阵，因此几乎每一层都需要合并局部结果。

## 四、Pipeline Parallel 中的 Send/Recv

![GPipe pipeline schedule](/assets/img/posts/2026-10-14-gpu-communication/04-gpipe-p2p-pipeline.png)

_图 4：GPipe 的流水线调度。原图包含训练的前向、反向和参数更新阶段。_

[GPipe](https://arxiv.org/abs/1811.06965) 将模型按层切成多个 pipeline stage。图中每一行代表一个 stage，横向代表时间；`F(i,j)` 表示 stage `i` 对 micro-batch `j` 做前向计算。

相邻 stage 之间不需要执行 AllReduce，只需把上一段网络产生的激活通过点对点 Send/Recv 发送给下一段。

原图描述的是训练，所以还包含 `B`（反向传播）和 `Update`。理解推理服务时可以忽略这两部分，只看 `F`：一个 micro-batch 的激活沿 stage 0 → 1 → 2 → 3 单向传递。

## 五、Expert Parallel 中的 AllToAll

![GShard Figure 3](/assets/img/posts/2026-10-14-gpu-communication/05-gshard-alltoall-expert.png)

_图 5：GShard 中使用 AllToAll 完成 expert dispatch 与 combine。图源：GShard Figure 3。_

[GShard](https://arxiv.org/abs/2006.16668) 中存在两次方向相反的 AllToAll：

1. AllToAll Dispatch：路由器根据每个 token 选中的 expert，把 token 从当前 GPU 发送到 expert 所在 GPU。
2. AllToAll Combine：expert 计算完成后，把 token 的结果送回原来的位置，并重新拼回序列。

与 AllGather 不同，AllToAll 不会让每张 GPU 获得全部 token。每个 token 只会被发送到需要处理它的目的地，因此不同 GPU 接收到的数据通常不同。

## 六、把通信方式映射回并行策略

| 并行方式 | 主要通信 | 推理时传递的内容 |
| --- | --- | --- |
| Data Parallel | 请求分发；副本间通常不通信 | 不同副本处理不同请求 |
| Tensor Parallel | AllReduce；有些实现使用 AllGather / ReduceScatter | 同一层的局部激活或局部计算结果 |
| Pipeline Parallel | Send / Recv | stage 边界处的激活 |
| Expert Parallel | AllToAll | 路由到不同 expert 的 token 表示 |

Data Parallel 在纯推理服务中的情况比较特殊：多个模型副本通常独立处理不同请求，副本之间不需要像训练阶段那样同步梯度。因此，推理 DP 的主要数据流往往发生在请求路由器与模型副本之间，而不是 GPU 副本之间的 AllReduce。

