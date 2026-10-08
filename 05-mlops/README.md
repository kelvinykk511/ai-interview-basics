# mlops / 推理與運維

工程向筆記（部署、監控、serving、成本）。

- [LLM Serving：延遲、吞吐與 KV Cache](./llm-serving-basics.md)
- [LLM Streaming（SSE）：首字體感、取消與背壓](./streaming-sse.md)
- [Prompt Caching 與推理成本](./prompt-caching-cost.md)
- [Embedding 換模與 Reindex 運維](./embedding-reindex-ops.md)
- [LLM Gateway：限流、熔斷與冪等重試](./llm-gateway-resilience.md)
- [Speculative Decoding：草稿模型加速推理](./speculative-decoding.md)
- [LoRA／多 Adapter Serving：一底多租戶](./lora-multi-adapter-serving.md)
- [PagedAttention 與 Prefix KV Caching（推理引擎）](./paged-attention-prefix-kv.md)
- [Continuous Batching 深挖（調度、迭代與尾延遲）](./continuous-batching.md)
- [量化 Serving：GPTQ／AWQ／FP8（上線視角）](./quantization-serving.md)
- [KV Cache 量化深挖（容量、品質與 Continuous Batching）](./kv-cache-quantization.md)
- [Multi-LoRA × Continuous Batching：容量規劃](./multi-lora-capacity-planning.md)
- [Speculative Decoding × Continuous Batching：高負載時加速為何消失](./speculative-x-continuous-batching.md)
- [Prefill／Decode 分離（P/D Disaggregation）](./prefill-decode-disaggregation.md)
- [Semantic Cache：LLM 回應快取（命中、誤命中與失效）](./semantic-cache.md)
