## LayerKV: Optimizing Large Language Model Serving with Layer-wise KV Cache Management

论文地址：https://arxiv.org/abs/2410.00428

结果：牺牲少量吞吐（3%）换取长文首token时延的大幅降低



长文场景下，首token的时延主要来自于prefill的排队时间，作者认为其根本原因是由于有限的GPU KV Cache 块无法满足长文的分配需求


三个贡献：

1. 发现了有限的GPU KV Cache 块无法满足长文的分配需求，会导致prefill阶段请求排队时延增加，最终导致首token时延增长
2. 设计了LayerKV，引入逐层KV Cache分配、管理和卸载，对系统内存进行了细粒度控制，并结合调度器实现整体优化
3. 对优化进行了评估。发现首token效率提升69倍，SLO减少28.7%



LayerKV 设计的两个关键难点

- 如何在不影响TPOT的同时优化TTFT
- 引入的PCle通信可能会导致计算停滞，降低QPS，影响系统整体吞吐。如何有效管理PCle通信不影响计算效率



解决方法

- SLO-aware Scheduler 解决第一个设计难点，用来确定是否可以提前调度请求的prefill阶段和提前调度多少个请求，确保不会影响当前请求的TPOT
- LayerKV Execution Engine 和 layer-wise KV cache offloading 用来处理计算和通信过程，解决第二个设计难点



SLO-aware Scheduler 请求决策过程

公式1，2 通过记录当前生成token数量和预测未来token生成数量来计算当前请求允许的prefill时延，有一定道理，但关键的预测方法没有详细说明。（也许可以参考：Haoran Qiu, Weichao Mao, Archit Patke, Shengkun Cui, Saurabh Jha, Chen Wang, Hubertus Franke, Zbigniew T. Kalbarczyk, Tamer Basar,and Ravishankar K. Iyer. 2024. Efficient Interactive LLM Serving withProxy Model-based Sequence Length Prediction.）



公式3 预测请求prefill的时间，引入了经验校正因子a，凭经验定义

公式4 计算KV blocks 从GPU移动到CPU上的时间（卸载blocks），强制其必须小于公式3（请求prefill的时间），从而反推移动的 blocks 数量

卸载的关键是保证计算时间与卸载通信时间完全重叠，因为二者异步执行

### 评价

- 首token时延随着上下文长度变化的性能测试
- 通用数据集下，增加请求数量，首token时延的变化