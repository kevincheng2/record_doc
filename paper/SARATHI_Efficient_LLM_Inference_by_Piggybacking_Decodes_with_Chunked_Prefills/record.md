### SARATHI: Efficient LLM Inference by Piggybacking Decodes with Chunked Prefills

论文地址：https://arxiv.org/abs/2308.16369



总结：通过将prefill 分块执行，捎带decoding阶段的方式，减小GPU的使用气泡。牺牲了prefill的性能（KV Cache），提高了系统整体吞吐，但prefill的块大小的选择对整体性能影响较大。

有兴趣可以看一下vllm的代码实现：https://github.com/vllm-project/vllm/pull/9871



背景：大模型推理 Pipline 并行时，是由不同长度的 prefill 和 decoding 混合组成，这导致不同的 micro-batches 有不同的执行时间，产生GPU bubble，这是当前架构下普遍存在的问题



decoding_bubble

两种技术： 使用 chunked-prefills 、decode-maximal batching  两种技术解决 inefficient decodes 和 pipeline bubbles 两个问题



将prefill阶段拆分为固定长度片段，依次执行各片段，执行过程中，捎带decoding片段，在Pipline 并行时，可以或得更高的吞吐。通过将prefill阶段拆开，并在执行过程中，”捎带“ decoding片段，可以更充分利用硬件设备（主要指GPU的算力和带宽）

prefill阶段拆分的长度由平均prefill长度和decoding阶段数量的比例得到（可以理解为平均输入长度和输出tokens长度的比例，一般在推理部署系统启动时根据模型和需求进行配置，现实应用时是可以获取到的）



### 主要贡献

- chunked-prefills，将prefill阶段拆分成固定长度片段
- decode-maximal batching，允许在prefill片段执行过程中，捎带decoding片段
- 将 chunked-prefills 和 decode-maximal batching 应用于Pipline 并行，以减少流水线执行过程中的气泡
- 在多个模型、多个硬件上进行了性能评估，吞吐量提高了1.91倍



### 具体分析

prefill decoding cost

如图所示，展示了transformer每个部分的耗时分析。可以看到，不同的batch下，prefill 阶段时间成本基本相同，这表明一个batch就可以让GPU利用率饱和；而decoding阶段会随着batch增大，平均时间成本会显著降低

prefill decoding feature

上图a展示了prefill和decoding两个阶段随输入长度和batch变化产生的吞吐变化，可以帮助我们更了解两个阶段的特点。b展示了两个阶段算力需求随batch的变化，可以清楚的看到，prefill是compute-bound 操作，decoding是memory-bound操作



Pipline bubble

三种气泡

1. PB1，连续两个prefill过程中token数量不同导致的
2. PB2，prefill和decoding计算时间不同，两个阶段相邻时出现
3. PB3，两个decoding阶段KV Cache的长度不同，导致两个decoding阶段计算时间差异出现



### 实现

#### chunked-prefills 

两个问题

- 分块太小会导致算力使用不充分，可以通过平均输入长度和输出tokens长度的比例计算得到最佳块大小，解决该问题

- 分块引入了额外的KV Cache读开销，后边块计算需要读取前边所有块的KV Cache



两个tip：

块小可以捎带更多的decdoing操作，但是会引入更大的prefill开销，降低prefill的效率；

保证块大小和decoding token数量之和 是 tile size的倍数，可以保证GPU 在做matmuls时不浪费计算。



#### decode-maximal batching

两个问题

- 确定可以携带的最大decoding批次大小，并确定prefill token的数量
- 将prefill和 batch decoding的线性计算融合到一个操作中（不懂。。。）



Decoding_batch 计算方法



decoding阶段的成本主要来自于访存操作（模型权重、KV Cache）。通过prefill捎带decoding阶段之后，prefill加载的模型权重，decoding阶段可以复用，节约了一部分decoding的访存成本。

Maxi-cost 图片



### 评价

对于A6000 GPU上的LLaMA-13B模型，将解码吞吐量提高了10倍，将端到端吞吐量加速了1.33倍。

对于A100 GPU上的LLaMa-33B，实现了1.25倍的端到端吞吐量和高达4.25倍的解码吞吐量。

Pipline 并行一起使用时，将气泡减少了6.29倍，从而使端到端吞吐量提高了1.91倍。



