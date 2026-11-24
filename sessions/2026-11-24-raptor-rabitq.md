# 2026-11-24 AI warm-up

- [RAPTOR／層次摘要（按相似度砌樹，對照 GraphRAG community／parent-child）](../04-prompt-rag/raptor-hierarchical-summaries.md)
- [二元量化／RaBitQ（1-bit 碼＋無偏估距，要 over-fetch 精排）](../04-prompt-rag/binary-quantization-rabitq.md)

練習：全庫一棵 RAPTOR 樹取代 GraphRAG／hybrid、費率同季度主題都搜摘要、只更新 leaf、SOP 同公開材料一齊聚類；天真 sign 原地覆寫 HNSW 並刪原向量、Hamming top-10 直接生成、混未量化新文檔、平均差 1% 就停 BM25，並以為有 MRL 就唔使量化。
