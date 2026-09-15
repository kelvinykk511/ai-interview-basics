# Multi-tenant RAG 隔離

## 人話

多租戶 RAG = 很多客戶／業務線共用同一套檢索＋生成服務，但**絕對不能檢到別人的文件**。後端直覺跟「共用 Elasticsearch／MySQL、用 tenant_id 過濾」一樣：漏過濾一次就是數據越權。向量庫多了 ANN 近似檢索，過濾要做對位置（pre-filter / post-filter），否則又慢又漏。

## 面試短答

- **隔離層級**（由強到弱）：*(1) 物理*：每租戶獨立 index／collection／甚至獨立叢集；(2) *邏輯*：同一 index，每向量帶 `tenant_id`（及 `workspace_id` 等）metadata，查詢強制 filter；(3) *應用*：只靠 prompt「不要提其他客戶」——**面試要說這不夠，幾乎等於沒隔離**。
- **過濾時機**：*Pre-filter*：ANN 只在允許集合內搜（正確性好，實現看向量庫）；*Post-filter*：先取 Top-K 再丟棄不屬於租戶的——K 太小會「檢空」，K 太大浪費。生產常要「帶 filter 的 ANN」或分 shard。
- **身份從哪來**：tenant 來自**已驗證的 session／JWT／內部 service token**，絕不是模型或使用者文字裡自稱的「我是租戶 A」。檢索 API 由後端組 filter，模型不可擅自改 metadata 條件。
- **寫入路徑**：ingest 時強制打上租戶標籤；禁止跨租戶 upsert；刪除／輪換 key 時按租戶範圍。CEX：做市商報告、VIP 工單、內部 SOP 可能同平台不同密級——還要 **RBAC／密級**，不只 tenant。
- **快取與 Prompt cache**：cache key 必須含 tenant（及權限版本）；共用「熱門 chunk 快取」若沒隔離會串租。
- **評估**：越權測試是一等公民——用租戶 B 的問題應檢不到 A 的 doc；加 red-team 集專門打「猜 id／忽略 filter」話術。

## 常見追問（含答案）

**Q：一個大 index + metadata filter，還是每租戶一個 index？**  
A：看規模與合規。租戶少／文件少：共用 index + 強制 filter 成本低。租戶多、數據敏感、或要獨立配額／加密：分 index（或分 namespace）。面試講 tradeoff：運維複雜度 vs 隔離強度與 noisy neighbor。

**Q：HNSW 上做 filter 有什麼坑？**  
A：若實現是「先圖搜再過濾」，高選擇性 filter（很小的租戶）可能 Top-K 全被濾光 → 召回變零。要選支援 filtered search 的引擎，或按租戶分圖／分 shard，並對「小租戶」單獨測召回。

**Q：Agent 自己產生 filter 條件可以嗎？**  
A：高風險。應用層應注入不可覆寫的 tenant predicate；模型頂多在**租戶內**選 folder／標籤。否則 injection 一句「tenant_id=*」就穿庫。

**Q：刪租戶／GDPR 刪除怎麼做？**  
A：按 `tenant_id`（或 doc id 列表）硬刪向量＋物件儲存＋審計；注意 replica／backup／邊緣 cache；刪除後用「已知舊 id 查詢應 miss」做驗收。不能只軟刪應用表卻留著向量。

## 小練習

**題：** 共用向量庫服務兩家機構 A/B。API 是 `POST /retrieve {query, topK}`，gateway 已能解析登入者的 `tenantId`。工程師想把 `tenantId` 當可選參數讓前端傳。你怎麼改設計？並寫一條你會加的自動化測試。

**參考答案：**  
- **設計**：`tenantId` **只**從已驗證身份取，請求體若帶 tenant 一律忽略或 400；檢索層 API 簽名為內部 `retrieve(tenantId, query, topK)`，filter 在服務端強制 `metadata.tenant_id == tenantId`（必要時 AND 密級）。禁止模型／前端覆寫。Cache key = `(tenantId, queryHash, indexVersion)`。  
- **測試**：用 A 的 token 查一句「只出現在 B 語料裡的獨特字串」→ 命中數為 0；再用 B 的 token 應能命中。另測：請求體偽造 `tenantId=B` 但 token 是 A → 仍只見 A（或 400），且 audit 記一筆。
