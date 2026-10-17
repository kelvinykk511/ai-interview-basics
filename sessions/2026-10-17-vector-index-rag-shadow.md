# 2026-10-17 AI warm-up

- 向量索引取捨：HNSW（M/efSearch、tombstone+compaction、食內存）vs IVF-PQ（nlist/nprobe、PQ 壓 32 倍 + 原向量 rerank、漂移要 retrain）、int8/binary 量化、Matryoshka 截維、內存估算、pre/post/圖內過濾按選擇率切換 → `04-prompt-rag/vector-index-tradeoffs.md`
- RAG 改動上線：replay → shadow（overlap/RBO、差異集 pairwise judge、無副作用、守 ACL、新鮮度對齊、採樣限流）→ canary guardrail + flag/alias 回滾 → `07-system-design/rag-shadow-eval.md`
- 小練習：5000 萬×1024 維 64GB 節點索引設計；hybrid + rerank 上線 plan
- Next：GPU vs cost routing；embedding 模型選型（多語言/領域）；agent eval（軌跡評估）
