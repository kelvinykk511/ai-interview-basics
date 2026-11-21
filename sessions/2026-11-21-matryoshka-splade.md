# 2026-11-21 AI warm-up

- [Matryoshka／可變維度（截前綴、重正規化、唔同 d 唔混庫）](../04-prompt-rag/matryoshka-variable-dimensions.md)
- [SPLADE／Learned Sparse（詞表稀疏＋擴展，對照 BM25／ColBERT）](../04-prompt-rag/splade-learned-sparse.md)

練習：256 維原地覆寫全維 HNSW 且唔 normalize、非 MRL 一併 `:256`、停 int8／BM25、只睇平均 nDCG；用 SPLADE 一次過取代 BM25 同 ColBERT、寫入現役 HNSW、query 唔 cache、平均分升就全量切。
