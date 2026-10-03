# 2026-10-03 · AI 工程溫習（KV Cache 量化 + Multi-LoRA 容量規劃）

水準：工程面試向；問題與參考答案同頁。

1. [KV Cache 量化深挖](../05-mlops/kv-cache-quantization.md)
2. [Multi-LoRA × Continuous Batching 容量規劃](../05-mlops/multi-lora-capacity-planning.md)

今日目標：能講清「為何高併發／長 context 時 KV 比權重更吃顯存、FP8／INT8／INT4 KV 取捨與回滾閘門」，以及「W_base＋adapters＋KV 容量帳、常駐／LRU／merge、SLO 驅動 sizing」。
