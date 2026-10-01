# RAG 新鮮度與 Index Lag

## 人話

RAG 答的是**索引裡的快照**，不是源系統當下真相。文件在 CMS／DB 改了，到「檢索能搜到新內容」中間有一段 **index lag**。後端直覺：像讀從庫／搜尋引擎的 eventual consistency——產品若假裝「永遠最新」，高風險場景（費率、提現狀態、公告）會直接答錯。換 embedding 模型要重建空間，見 [`../05-mlops/embedding-reindex-ops.md`](../05-mlops/embedding-reindex-ops.md)；**本篇講資料時效／與 source of truth 的落差**。

## 面試短答

- **定義**：`lag = update_in_source → visible_in_retrieval`（含可被正確 embed／commit／可被 query 命中）。答案來自 index snapshot ≠ live SoT。
- **Lag 來源**：爬蟲／CDC 延遲、embed 佇列 backlog、向量庫 commit／refresh、多層 cache（query／result／embedding）、副本 lag、人工審核隊列。
- **產品真話**：暴露「知識庫最後同步時間／該語料新鮮度 SLA」，不要裝永遠即時。費率表、活動規則要分鐘～小時級；穩定 FAQ／SOP 可放寬到日級。
- **常見模式**：寫路徑 **sync upsert**（改完立刻 embed＋upsert）；**near-real-time CDC**；**dual-read**（穩定段落走 index，易變欄位走 live API）；**retrieve then verify**（檢索後用源 API 核對關鍵字段）。
- **CEX 風險**：錯費率／錯提現狀態／過期公告 = 高嚴重度。原則：**易變事實走 live API／DB**；RAG 扛穩定 SOP、操作說明、歷史公告歸檔。
- **怎麼量**：watermark（`max(indexed.updated_at)` vs source）、freshness SLO（例如 p95 lag < 5min）、發布後 **canary query**（剛上線的標題／關鍵句是否可檢回）。
- **和 reindex-ops 的邊界**：reindex = 模型／空間／chunk 策略變更；freshness = **同一模型下資料是否跟上源**。兩者都會讓「檢到的不像真的」，根因不同。

## 常見追問（含答案）

**Q：為什麼「向量庫裡有這篇」仍可能答過期內容？**  
A：可能命中舊 chunk（沒刪舊版本）、cache 回了舊結果、或 dual index 切換不完整。新鮮度要管 **刪除／版本／cache 失效**，不是只管 insert。

**Q：所有知識都 sync upsert 不行嗎？**  
A：寫放大、embed 成本、尖峰會打爆佇列；大 PDF／多語言語料不適合同步阻塞寫路徑。實務：熱語料（費率、限額說明）同步或 CDC；長尾文檔異步＋接受較長 SLA。

**Q：Dual-read 怎麼設計才不像兩套真相？**  
A：檢索負責「找哪份 SOP／哪條公告」；金額、狀態、開關等 **volatile fields** 用源 API 覆蓋進 prompt（或後端填槽）。生成層禁止用過期 chunk 裡的數字覆蓋 live 字段。產品文案區分「說明來自知識庫」vs「狀態來自系統」。

**Q：怎麼向面試官證明 freshness 有 SLO？**  
A：畫流水線階段耗時（CDC → embed → commit）；報 watermark 與 p95／p99 lag；發布流水線加 canary query；告警：lag 破線、佇列深度、replication lag。順便講失敗模式：embed worker 掛了但源系統仍在更新。

**Q：和 grounding／citation 怎麼一起講？**  
A：Citation 證明「依據哪段」；freshness 證明「那段夠不夠新」。過期但引用正確仍是產品事故。高風險槽位：有 citation 也要 live verify。可交叉看 [grounding-citations](./grounding-citations.md)。

## 小練習

**題：** 交易所客服 RAG 昨晚改了「ETH 提現手續費」，CMS 已發布，但今早 bot 仍報舊費率。流水線是：CMS → 每 30 分鐘 crawl → embed 佇列 → 向量庫（refresh 可見約 1 分鐘）→ 閘道有 10 分鐘 result cache。請：(1) 估算最壞可見 lag；(2) 指出至少三個改進（含產品／架構）；(3) 說明哪些問題改走 live API 而不是催索引。

**參考答案：**  
1. **最壞 lag**：crawl 週期 30m + embed 排隊（假設繁忙再數十分鐘，面試先按「佇列未知」分開講）+ refresh ~1m + result cache 10m → **僅排程與 cache 已可 >40 分鐘**；若 embed backlog 更大更糟。  
2. **改進**：費率／限額改 **寫路徑 upsert 或 CDC**（發布即入隊高優先）；對費率 query **bypass 或縮短 result cache**（或 cache key 帶 `content_version`）；產品顯示「知識庫同步時間」；發布後 canary query；watermark 告警。  
3. **該走 live API 的**：當前費率、提現是否暫停、KYC 等級限額、工單／充值狀態——用配置／計費服務查，RAG 只解釋「費率如何計、從哪看」。穩定 SOP／操作步驟可繼續 RAG。與換模重建分開：本題是 **資料時效**，不是 [embedding-reindex-ops](../05-mlops/embedding-reindex-ops.md)。
