# SPLADE／Learned Sparse：學出嚟嘅稀疏詞權重，同 ColBERT、BM25 都唔同

> 2026-11-21 · AI warm-up · 工程面試向  
> ColBERT 見 [colbert-late-interaction.md](./colbert-late-interaction.md)；hybrid 見 [hybrid-search-rerank.md](./hybrid-search-rerank.md)；生產 rerank 見 [reranker-production.md](./reranker-production.md)。**本篇講：SPLADE 點樣把文檔／查詢編成詞表上嘅稀疏向量、點解能擴展出沒寫明嘅詞、同 BM25／dense／ColBERT 點分工，以及倒排索引同版本要點釘。**

## 人話

BM25 只認文檔裡實際出現嘅詞，權重由詞頻／IDF 公式決定——「提現手續費」未必對到「出金 fee」。Dense embedding 懂語義，但變成一條密向量之後，你唔容易解釋點解命中，精確代號亦可能被平均淡化。

**SPLADE**（SParse Lexical AnD Expansion）用 transformer 預測詞表（例如 BERT WordPiece，約三萬維）上每個詞嘅權重，輸出**一條好稀疏嘅詞項向量**。文檔可以對「沒寫出但相關」嘅詞有權重，呢個叫 **expansion**；查詢亦然。打分通常係稀疏向量點積，可以用**倒排索引**存（詞 → 文檔同權重），同 BM25 運維模型接近，但權重同擴展詞係模型學出嚟。

後端類比：BM25 像按原文建倒排；SPLADE 像先讓模型幫每篇文加「應該被搜到嘅標籤同權重」，再入倒排。ColBERT 則係保留每個 token 一條短密向量，打 MaxSim，唔係詞表稀疏向量。面試官想聽：同 BM25／ColBERT 唔好混、expansion 嘅好處同誤擴展、可唔可以塞進 HNSW、query 側延遲。

## 面試短答

- **SPLADE 係 learned sparse，唔係 BM25 換皮，亦唔係 ColBERT。** BM25：精確詞、公式權重、幾乎無模型推理。SPLADE：模型輸出詞權重＋擴展，倒排打點積。ColBERT：token 級密向量 + MaxSim，要專用索引或只做 rerank。三個可互補，唔互相改名取代。
- **價值在 expansion 同可解釋。** 「出金手續費」可以對到「withdrawal / 提現 / fee」呢類詞；同時你可以 log 頂權重詞做 debug。呢點 dense 做唔到咁直接。
- **誤擴展係真實風險。** 模型可能把「ETH」擴到鄰近產品或活動詞，令費率／風控 SOP 召回變髒。CEX 要對 ticker、錯誤碼、產品別名做 slice；必要時對必召回欄（`error_code`）保留純關鍵字／BM25 字段，唔把 SPLADE 當成唯一 lexical。
- **用倒排，唔好當密向量塞 HNSW。** 維度係詞表大小、絕大多數係 0，幾何係稀疏點積。HNSW＋餘弦既食內存又唔對題。Elasticsearch／OpenSearch／Lucene，或支援 sparse vector 嘅引擎，先係對口。
- **文檔側離線編碼，查詢側要跑模型，延遲同 BM25 唔同一個數量級。** BM25 查詢幾乎係分詞＋讀 posting；SPLADE query encoder 係一次 transformer，要計入 p95，可 cache 熱 query。文檔更新要重新過模型，唔係只 re-tokenize。
- **稀疏度係一等公民。** 訓練有 FLOPS 類正則，推理常再 top-k 截詞。太密：倒排膨脹、尾延遲升；太疏：召回跌。`max_query_terms`／`max_doc_terms` 要釘進版本，改咗等於換索引行為。
- **同 dense 仍然做 hybrid，融合用 RRF。** SPLADE 分數同 cosine 唔同量綱，唔好直接加。係咪「停 BM25」要靠 slice：有時 SPLADE 已覆蓋語義化 lexical，有時精確 ID 仍係 BM25 穩。ColBERT／CE 仍可做後面幾階，唔好一句「有 SPLADE 就拆晒」。
- **版本要同模型、詞表、截斷參數一齊釘。** 換 SPLADE 模型或 tokenizer 就係新 lexical 空間，要獨立 index、shadow、一齊切 alias。cache key 帶 `retriever=splade` + model + term caps。

## 常見追問（連答案）

**Q1：有 SPLADE 係咪可以停 BM25？**  
A：唔好默認停。先對精確代號／錯誤碼／口語改寫分開測。若 SPLADE 在標識 slice 不弱於 BM25，再考慮只留 SPLADE 做 sparse，以減少雙倒排運維；否則保留 BM25 做必召回字段，SPLADE 當第二路 lexical。

**Q2：SPLADE 同 ColBERT 邊個代替邊個？**  
A：唔代替。SPLADE 輸出可入倒排嘅稀疏詞向量，一階成本接近經典搜尋；ColBERT 保留 token 密向量，一階貴、常做二階 MaxSim。可以「SPLADE + dense 召回 → ColBERT 或 CE 精排」。上次講嘅 late interaction 同呢篇 learned sparse 係兩條線。

**Q3：可唔可以把 SPLADE 嘅詞表維向量當普通 embedding 插入現役 HNSW？**  
A：唔應該。稀疏、高維、大量零，同現役 dense 空間完全唔相容，亦浪費圖結構。用倒排或引擎嘅 sparse 類型；同 dense 用 RRF 融合。

**Q4：Query 編碼會拖垮 p95，點辦？**  
A：預算寫死；熱門 query 做編碼 cache（key 含模型同 term cap）；batch 編碼；必要時用 distilled／efficient SPLADE 類細模型。若延遲預算極緊而標識題為主，ID-like 流量可路由回 BM25，唔強迫每條請求都過 SPLADE。

**Q5：加一篇 SOP 或改費率一行，要做乜？**  
A：受影響文檔要重新 SPLADE 編碼再寫倒排，唔係只改 BM25 詞頻。模型或 max terms 變更則要重建該稀疏庫。同 dense 一樣用同一 `doc_id` 雙寫，alias 一齊切。

## 小練習

**題：** 現役係 hybrid（dense + BM25）+ CE rerank。剛上完 ColBERT 做二階評估。同事建議用 SPLADE「一次過取代 BM25 同 ColBERT」，並把 SPLADE 向量寫入現役 HNSW 同 dense 共用；query 側唔 cache；ticker／錯誤碼 slice 只要平均分數上升就可以全量切。你會點評，同埋會點落地？

**參考答案：**  
1. **唔取代 ColBERT，亦唔默認立刻停 BM25。** SPLADE 係 lexical／expansion；ColBERT 係 token late interaction。先把 SPLADE 當額外或替代稀疏路，同 dense 做 RRF，ColBERT／CE 仍按曲線留喺後面。  
2. **唔入現役 HNSW。** 獨立倒排／sparse 索引。  
3. **Query 要計延遲：cache、預算、必要時分流 BM25。**  
4. **平均分唔能做閘。** ticker 誤擴展、錯誤碼必召回要單獨過閘；必要時保留 keyword 字段。  
5. **落地：** 版本釘模型＋term caps；ingest 雙寫；shadow／canary；log 擴展詞方便查誤擴展。通過後先替換或並存 sparse 路，再決定要唔要減 BM25 運維。
