### SARATHI: Efficient LLM Inference by Piggybacking Decodes with Chunked Prefills

论文地址：https://arxiv.org/abs/2308.16369



背景：大模型推理 Pipline 并行时，是由不同长度的 prefill 和 decoding 混合组成，这导致不同的micro-batches 有不同的执行时间，产生GPU bubble

decoding_bubble

两种技术： chunked-prefills 、decode-maximal batching  解决 inefficient decodes 和 pipeline bubbles

将prefill阶段拆分为固定长度片段，依次执行各片段，执行过程中，捎带decoding片段，在Pipline 并行时，可以或得更高的吞吐