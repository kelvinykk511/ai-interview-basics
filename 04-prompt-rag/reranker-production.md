# Reranker 上線實戰

## 人話

向量召回（bi-encoder）係 query 同文檔各自變向量再比距離：快但粗。Reranker（cross-encoder）將 query + 文檔一齊餵入模型逐對打分：準好多但貴。所以兩段式：hybrid 召回 top-K（例如 50）→ rerank → 只揀 top 5–8 入 prompt。

## 面試短答

- Two-stage retrieval：bi-encoder 負責 recall，cross-encoder 負責 precision。
- 成本 = 每個 query K 次 (query+doc) forward，可以 batch；延遲預算要預留。
- K 用自己標註集嘅 recall@K 曲線揀；離線用 nDCG@10 / MRR，線上睇答案正確率 + citation precision。
- 分數跨 query 唔可比，唔好直接用固定閾值，要校準。
- Reranker 掛咗要降級返 RRF 原排序，唔可以拖死成個 RAG。

## 常見追問（連答案）

1. **K 點揀？** 畫 recall@K（gold 喺 top-K 比例）同 rerank 延遲。例：K=50 recall 92%、K=100 94% 但延遲翻倍 → 揀 50，喺平台期起點。
2. **延遲點壓？** batch、FP16/int8、doc 截 256–512 tokens、GPU 跑、蒸餾細模型、cache key=(query hash, doc id, doc version)。
3. **可唔可以用分數做「無相關就拒答」？** raw logit/sigmoid 未校準。喺自己標註集做 Platt / isotonic 校準，再按 risk–coverage 揀閾值。
4. **Reranker timeout？** 設硬 timeout（如 150ms）→ 用 RRF 原排序 + 打 metric + circuit breaker。
5. **長 chunk 被截斷？** cross-encoder 多數 max 512 tokens，答案喺尾會睇唔到 → rerank 用 chunk 唔用全文，或切段取最高分（max-passage）。
6. **LLM listwise rerank vs cross-encoder？** LLM 準但貴、慢、有位置偏差、輸出要 parse；常見做法係 cross-encoder 做主，LLM 只用喺高價值查詢。
7. **多租戶/ACL？** rerank 前先過權限，唔好 rerank 完先 filter（浪費名額 + log 泄漏風險）。

## 小練習

**題：** 交易所客服 RAG，hybrid top-20 直接入 prompt，成日引用錯文檔。加 reranker 後離線 nDCG 升，但 p95 由 1.2s 變 2.0s，SLO 係 1.5s。點做？

**參考答案：** (1) 先確認 gold 喺唔喺 top-20 — 唔喺嘅話 rerank 冇用，要先修召回；(2) 壓延遲：只 rerank top-30、doc 截 256 tokens、GPU batch + FP16、蒸餾細模型、同其他步驟並行、加 cache；(3) rerank 後只入 top-5，prompt 短咗 LLM prefill 快咗，抵銷部分延遲；(4) 150ms timeout 降級 RRF；(5) shadow/canary 比答案正確率同 citation precision，唔淨睇 nDCG。
