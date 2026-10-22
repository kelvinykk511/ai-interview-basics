# Embedding 模型選型

## 人話

embedding 模型決定「意思相近」點量度。揀錯咗，後面 HNSW、rerank 點調都救唔返召回。唔好睇 MTEB 排行榜就揀，要用自己數據評。

## 面試短答

用自建 eval set（真 query + 標註相關文檔，按 slice 分：精確 ID/幣種代號、語義問題、繁中/粵語/中英夾雜）量 Recall@k、nDCG@10；再比：維度×向量數嘅內存成本、embed 延遲/吞吐、max input tokens、多語能力、API vs 自托管（數據出境、限流、模型被下架）、license。揀「夠好又最平」嗰個，用 rerank 補精度。換模型 = 全量重建索引（向量空間唔兼容），要藍綠切換。

## 常見追問

- MTEB 高分點解唔可靠？→ 以通用/英文為主、可能有資料洩漏；你嘅域（交易所術語、USDT-TRC20、訂單號）同語言分佈唔同；平均分掩蓋 slice 差異。
- asymmetric 檢索要注意乜？→ 好多模型要 query/document 用唔同 prefix 或 instruction（如 "query: " / "passage: "，instruct 模型要 task 描述）；漏咗會靜靜跌幾個點召回。索引同查詢要一致，prefix 寫入版本化 config。
- 維度越大越好？→ 唔一定。1024→3072 內存 ×3，召回可能升唔到 1 點。Matryoshka 模型可截短（3072→768）粗召回，再用全維 rerank；int8 量化再慳約 4 倍。
- API embedding 運維風險？→ 供應商升級/下架要重建；限流決定重建要幾耐（chunk 數 ÷ 吞吐要預算）；數據出境/PII；每條向量記 model+version+prefix 做 namespace。
- 中英夾雜/代號點處理？→ dense 對精確 token（合約地址、ticker）弱，要 hybrid BM25 + RRF；eval 按 slice 設 gate。

## 小練習

**題：** 知識庫 2M chunks，現用模型 A（1536 維、API），候選 B（1024 維、自托管、多語強）。B 喺 eval set 整體 nDCG@10 +3，但「精確代號」slice −5。點決策？

**參考答案：** 唔好淨睇平均。(1) 成本：2M×1536×4B ≈ 12.3GB vs 2M×1024×4B ≈ 8.2GB（未計 HNSW 圖開銷），再比 GPU 自托管成本 vs API 費。(2) 代號 slice 跌 → 試 hybrid（BM25+RRF），若 hybrid 後 B 每個 slice 都 ≥ A 就換。(3) 上線：新 index 藍綠、shadow 比 overlap、canary、alias 回滾，記錄 model version。(4) 若代號 slice 仍跌，保留 A，或 router 對 ID-like query 走 lexical。

**相關：** [vector-index-tradeoffs.md](./vector-index-tradeoffs.md)、[rag-shadow-eval.md](../07-system-design/rag-shadow-eval.md)、[embedding reindex ops](../05-mlops/embedding-reindex-ops.md) 筆記。
