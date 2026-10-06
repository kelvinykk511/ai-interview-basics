# 2026-10-06 · AI 工程溫習（Speculative × CB + P/D 分離）

水準：工程面試向；問題與參考答案同頁。

1. [Speculative Decoding × Continuous Batching](../05-mlops/speculative-x-continuous-batching.md)
2. [Prefill／Decode 分離（P/D Disaggregation）](../05-mlops/prefill-decode-disaggregation.md)

今日目標：能講清「為何 speculative 低負載有效、高負載縮水，及動態開關／自適應 k／prompt lookup／分池對策」，以及「prefill 與 decode 資源特性不同、干擾、P/D 分離的好處、KV 傳輸代價、幾時值得」。
