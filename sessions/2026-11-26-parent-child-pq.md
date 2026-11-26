# 2026-11-26 AI warm-up

- [Parent-child／small-to-big（細塊檢索、大塊生成，對照 RAPTOR／CR／late chunking）](../04-prompt-rag/parent-child-small-to-big.md)
- [PQ／IVF-PQ 與 int8（分段碼本壓縮，對照 RaBitQ／MRL）](../04-prompt-rag/product-quantization-ivf-pq.md)

練習：只索引整章、命中送 80 頁、同一章貼三次、公開同內部拼成同一 parent、citation 只標章名；用無關 1k 英文訓 PQ 並刪原向量、ADC top-10 直接生成、int8 同 PQ 混庫、語料轉變永不重訓、有 MRL 就關量化同 BM25。
