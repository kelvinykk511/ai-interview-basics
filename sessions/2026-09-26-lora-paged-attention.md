# 2026-09-26 · AI 工程溫習（LoRA 多 Adapter + PagedAttention／Prefix KV）

水準：工程面試向；問題與參考答案同頁。

1. [LoRA／多 Adapter Serving：一底多租戶](../05-mlops/lora-multi-adapter-serving.md)
2. [PagedAttention 與 Prefix KV Caching（推理引擎）](../05-mlops/paged-attention-prefix-kv.md)

今日目標：能講清「凍結底座 + 低秩 adapter、請求路由／熱插拔與版本釘選」，以及「KV 分頁抗碎片、prefix block 跨請求共享 vs 計費層 prompt cache、錯共享風險」。
