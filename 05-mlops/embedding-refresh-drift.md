# Embedding 刷新與漂移（增量重嵌、版本釘選、語料／查詢漂移）

> 2026-11-12 · AI warm-up · 工程面試向  
> 換模型／換向量空間見 [`embedding-reindex-ops.md`](./embedding-reindex-ops.md)；源文件改完索引跟唔跟上見 [`../04-prompt-rag/rag-freshness-index-lag.md`](../04-prompt-rag/rag-freshness-index-lag.md)；**本篇講：同一條檢索鏈上，向量幾時會「過期」、點增量重嵌、點釘版本，以及語料／查詢漂移點監察。**

## 人話

Freshness 係「呢篇而家嘅正文有冇入到索引」；reindex-ops 係「換咗座標系要重建」。仲有第三件事：即使模型冇換、正文暫時都冇改，**向量仍然可以變得唔啱用**——例如 vendor 靜靜升級咗 API、你改咗 query／document prefix、索引入面混咗舊 chunk 策略、或者交易所上咗一堆新幣、用戶問法變咗，舊近鄰圖同舊金標都跟唔上。

後端類比：唔單止「從庫延遲」（freshness），亦唔單止「換序列化協議」（換模型）。仲要管 **schema／analyzer 版本有冇釘死**、**增量索引同定期重建**、以及 **流量分佈漂咗沒有**。面試官想聽你分得清呢三個根因，同埋增量 upsert 嘅作業設計。

## 面試短答

- **三條線分清。** Freshness = 源正文 vs 索引快照；換模 reindex = 不相容向量空間，必須新 collection；**Refresh** = 同一（或聲明相容嘅）模型下，重算向量／重整索引，令版本一致、髒資料追上、或對抗漂移。
- **每條向量都要帶版本。** `embed_model_id` + `embed_version`（釘死，唔好只寫 `latest`）+ `chunk_strategy_version` + `content_hash` + `embedded_at`。Query 端要用同一組版本；對唔上就拒絕混查，而唔係 silently 撈亂近鄰。
- **增量重嵌嘅觸發。** 正文 `content_hash` 變 → dirty queue upsert；刪除要 tombstone／確實從索引拎走；chunk 策略或預處理改變 → 受影響範圍重建（通常新 collection，唔好喺同一個 index 混兩種切法）；模型微版本／prefix／正規化改變 → 當新空間，走藍綠（見 reindex-ops）。
- **正文冇改都可能要重嵌。** 換咗 instruction prefix（`query:` / `passage:`）、tokenizer／正規化、或供應商唔保證 bit-compatible 嘅「小升級」。呢類唔係 freshness 問題。
- **作業設計。** CDC／變更佇列 + 冪等 `chunk_id` upsert + checkpoint；失敗重試；優先級（費率、上幣公告先於長尾 FAQ）；batch embed 慳成本。大範圍 refresh 用新 index 建完再切 alias，避免查詢期「半新半舊策略」。
- **同一模型嘅增量係合法嘅；同一 collection 混版本唔合法。** 同一 `embed_version` 下更新單篇 OK。唔可以將 v2 向量 upsert 進 v1 index 就算維度相同。
- **索引結構都會漂。** HNSW 長期大量增刪之後圖品質可能差過一次重建；刪除若只軟刪，召回會摻垃圾。大規模變更後要有「重建 vs 增量」嘅門檻。
- **查詢側快取。** Query embedding cache 必須把 `embed_version` 加進 key；模型一 pin 新版本，舊 query 向量全部失效。
- **漂移監察。** 語料：新上幣／新產品文件覆蓋率、dirty 佇列 lag、各 `embed_version` 佔比（應該係 100% 單一現役版本）。查詢：query 向量分佈、空結果率、新產品／新代號 slice 嘅命中；金標要定期加新題，唔好永遠考舊 FAQ。
- **CEX 直覺。** 上線一個新代幣或改名（舊名仍在公告、新名喺規則），舊向量空間嘅近鄰可能仍把流量帶到舊文。freshness 只保證「舊文係最新正文」，唔保證「新實體已經可被語義召回」。要有上幣／改名 pipeline：文件入庫 → 優先 embed → canary query（新代號、新產品名）必須 Hit。

## 常見追問（連答案）

**Q1：文件冇改，做乜要 refresh embedding？**  
A：三種常見原因：（1）embedding 模型或編碼參數變咗（prefix、正規化、vendor 版本）；（2）chunk／預處理變咗，舊切法同新 query 不匹配；（3）索引結構因長期增刪退化，要重建。純「源正文已變但未入索引」係 freshness，唔好混。

**Q2：點避免 vendor 把 `text-embedding-X` 暗升級搞亂近鄰？**  
A：Pin 具體 version／snapshot，唔用 `latest`；寫入時存 `embed_version`；啟動同定期對帳（用固定 probe 句子嘅向量 hash／checksum）。一旦發現 probe 向量變咗，當新模型：新 collection、對比 eval、切 alias。合約同 changelog 要跟，唔好假設維度不變就相容。

**Q3：百萬 chunk，點決定增量還是全量重建？**  
A：只係部分文件正文變更 → dirty queue 增量 upsert。涉及模型、prefix、chunk 策略、或現役 index 入面版本混雜 → 新 collection 全量（或按 namespace）再建切換。經驗門檻：刪改比例好高、HNSW 召回回退、或版本佔比唔係單一，傾向重建。成本用「要重嵌嘅 chunk 數 × 單價」估，熱點文件優先。

**Q4：Query 用咗新模型、文檔還係舊向量，會點？**  
A：跨空間近鄰分數無意義，表現會「靜靜變差」而唔一定報錯。防護：檢索請求帶版本，同 index metadata 對唔上就 fail close 或只走舊鏈；feature flag／alias 原子切；query cache key 含版本。呢個同「維度都係 1536 所以得」係同一個陷阱。

**Q5：語料漂移同查詢漂移有咩分別？點監控？**  
A：語料漂移＝庫裡嘅實體同用詞變咗（新產品、改名、下架刪除未清）。查詢漂移＝用戶問法／意圖分佈變咗（活動期、市場波動、新功能）。監控：前者睇覆蓋率、刪除 lag、新 doc canary 召回；後者睇 query 分佈、slice 空結果／reformulation、線上檢索指標（見 [online-retrieval-eval](../07-system-design/online-retrieval-eval.md)）。金標要跟隨漂移補題，否則離線分數好睇、線上新查詢掛。

## 小練習

**題：** 交易所知識庫用釘死嘅 `embed-v3`。本週做咗三件事：（a）上線新幣 $KELVIN，CMS 已發布產品頁同費率；（b）為咗「更對題」有人把 query 側 prefix 從無改成 `query:`，文檔側未改、亦未重建；（c）同事建議把所有「超過 180 日未重嵌」嘅 FAQ，即使 `content_hash` 不變，每夜批量重嵌回同一個 collection。Bot 開始對新幣名召回不穩，舊 FAQ 亦有用戶話「答得怪」。請分開診斷三件事，並寫你會落地嘅版本同作業規則。

**參考答案：**  
1. **(a) 係 freshness + 覆蓋問題，唔係換模。** 新頁要高優先入隊 embed，發布流水線加 canary query（幣名、合約簡稱、費率標題）必須 Hit@k；只保證舊文已更新唔夠，新實體要能被語義召回。易變費率仍應 live API 覆寫（見 freshness 篇）。  
2. **(b) 係編碼參數變更，等同空間不對稱。** Query prefix 改咗但 passage 未用對應方式重嵌，近鄰會漂。應視為新 `embed_version`：文檔側一齊按官方對稱 prefix 重建到新 collection，eval／shadow 後切 alias，禁止只改 query 側。  
3. **(c) 錯誤。** 同一 `embed-v3`、正文不變，重嵌結果應幾乎相同，耗成本之餘若 batch 中途混入其他版本就有風險；「過了 180 日」本身唔係重嵌理由。要重嵌嘅係 dirty hash、版本變更、或已證實嘅索引結構退化（另開重建 job，唔好默默覆寫時混版本）。  
4. **規則：** 每條向量寫死 model+version+chunk_strategy+hash；query cache key 含版本；禁止 `latest`；現役 collection 只接受單一 `embed_version` upsert；新幣／改名走優先隊列 + canary；大規模策略變更走藍綠，唔走同 index 混寫。
