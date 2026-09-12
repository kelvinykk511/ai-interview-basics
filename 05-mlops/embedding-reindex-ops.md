# Embedding 換模與 Reindex 運維

## 人話

向量庫裡存的不是「文字」，是「某個 embedding 模型眼中的座標」。換模型（或改維度／正規化方式）＝座標系換了，舊向量和新 query 向量不能混用。後端類比：像換了序列化協議或換了 hash 算法——舊索引要重建，或至少隔離在新 collection，不能 silently 混查。

## 面試短答

- **何時要 reindex**：換 embedding 模型／版本、改 chunk 策略、改预处理（大小寫、HTML strip）、維度或 similarity metric 變更（cosine vs L2）。
- **為什麼不能混用**：不同模型的向量空間不可比；同一句在 A、B 模型的近鄰集合不同。混用 → 檢索亂序或分數無意義。
- **上線策略**：新建 collection／index（或別名雙寫）→ 全量／增量回填 → 用 eval／影子流量對比 → 原子切別名／路由 → 保留舊 index 一段時間可回滾。
- **增量 vs 全量**：文件變更用 upsert（同一 `chunk_id` 覆蓋）；換模型必須**全量**（或按模型版本分 namespace 全量建新）。
- **成本與時間**：文件數 × chunk 數 × embedding API／GPU 成本；要估 QPS、rate limit、失敗重試、checkpoint（從哪裡續跑）。
- **一致性**：reindex 期間讀舊寫？雙寫？還是維護窗口？面試講清「查詢路徑何時切」，避免半新半舊。
- **CEX 直覺**：像換 DB 編碼或換搜尋引擎 analyzer——灰度、對賬、可回滾；資金／合規知識庫切換要有抽樣人工驗收。

## 常見追問（含答案）

**Q：只換 reranker、不換 embedding，要不要全量 reindex？**  
A：通常不用。Rerank 吃的是「候選文件原文／稀疏特徵」，不依賴舊向量空間。但若候選召回變差，問題在第一階段檢索，不是 rerank 能獨自修好。

**Q：同一個模型「小版本升級」一定要 rebuild 嗎？**  
A：看供應商是否保證向量相容。多數 embedding 模型**不保證**跨版本空間不變；安全做法是當新模型：新 index＋對比 eval。若官方明確「bit-compatible」才可原地。

**Q：百萬 chunk 怎麼少停機切換？**  
A：藍綠／別名：`kb_current` → 指向 `kb_v2`；回填期間查詢仍打 `kb_v1`；切換是改路由／alias 一秒級。大庫可分 shard 平行 embed，用 job id＋進度表做斷點續跑。

**Q：Reindex 時文件又更新了怎麼辦？**  
A：記錄 `content_hash`／`updated_at`；回填快照之後的變更進變更佇列，切換前或切換後追趕增量 upsert。切換瞬間要定義「以哪一版為準」避免丟更新。

**Q：怎麼證明新模型比較好？**  
A：離線：同一 golden query 集比 recall@k、MRR、nDCG；線上：影子流量比點擊／解決率／人工差評；還要看延遲與成本。單看「平均 cosine 變高」不夠。

**Q：降維或量化（PQ／OPQ）算換空間嗎？**  
A：算索引參數變更；通常要 rebuild 該 index 結構。若量化與原向量共存（先精排再重排），要講清召回粗排用量化、精排用原向量或 rerank。

## 小練習

**題：** 線上 RAG 用 `text-embedding-A`（1536 維）。產品要換 `text-embedding-B`（也是 1536 維）。同事說「維度一樣，直接把新向量 upsert 進同一個 collection 就行」。錯在哪？你會怎麼做？  

**參考答案：**  
錯在**維度相同 ≠ 同一向量空間**；A／B 的座標不可比，混在同一 index 會讓近鄰搜尋失效。正確做法：建 `collection_B`（或新 namespace）→ 用 B 全量重嵌所有 chunk → 用 golden／影子流量對比 → 原子切路由／alias 到 B → 舊 collection 保留作回滾；文件更新佇列在切換前後追趕，避免丢變更。
