# 向量索引取捨：HNSW vs IVF-PQ、量化、過濾檢索

> 補充 `embeddings-vector-db.md`（嗰篇講基礎），呢篇講生產上點揀同點調。

## 人話
向量檢索 = 喺幾千萬個點入面揾最近嘅 k 個。暴力（flat）最準但每次 O(N·d)。ANN 用「少少唔準」換速度同內存。三個旋鈕：**索引結構、壓縮、過濾**。

- **HNSW**：多層圖，好似 skip list + 鄰居網絡；頂層粗跳、落層精搜。參數 `M`（每點鄰居數）、`efConstruction`（建圖質量）、`efSearch`（查詢候選隊列，越大越準越慢）。優點：recall 高、延遲低、可增量插入。缺點：食內存（原向量 + 圖邊，layer0 每點約 2M 個 4-byte id，M=16 ≈ 128B）、刪除靠 tombstone、build 慢。
- **IVF**：k-means 分 `nlist` 個桶（倒排），查詢只掃最近 `nprobe` 個桶。類比 DB 分區裁剪。省內存、build 快；`nprobe` 太細會漏召回；數據漂移後 centroid 過時要 retrain。
- **PQ（product quantization）**：768 維切 m 段，每段用 256 個碼字 → 每段 1 byte。768×4B=3072B → m=96 得 96B，壓 32 倍。距離有誤差 → 先粗排，再用原向量 rerank（refine）top-N。
- **SQ int8**：壓 4 倍，誤差細；**binary**：壓 32 倍，一定要配 rerank。
- **維度**：Matryoshka 類 embedding 可截短（如 3072→1024），用少少質量換一半以上內存。
- **內存估算**：1 億 × 768 × 4B ≈ 307GB float32，單機放唔落 → 要 PQ / 分片 / DiskANN 類磁碟索引。

### 過濾檢索（多租戶、權限）
- **pre-filter**：先按條件篩，再喺子集搜。篩剩好少時直接暴力就最準。
- **post-filter**：先搜 top-k 再篩。篩得狠會得返 0 條 → 要 over-fetch，仍有空結果風險。
- **圖內過濾**：遍歷 HNSW 時跳過唔符合嘅點。選擇率好低時圖「斷路」，recall 跌。
- 實務：按**選擇率**切換策略；大租戶獨立 namespace/partition，小租戶共享 + payload filter。權限過濾必須喺檢索層硬性執行，唔可以靠 LLM。

### 評估
recall@k（對比 flat ground truth）、P99、QPS/成本、內存；最後一定要睇 end-to-end 答案質量——ANN recall 95%→99% 嘅影響往往細過 chunking 或加 reranker。

## 面試短答
「HNSW recall 高、延遲低、支持增量，但食內存、刪除要 compaction；IVF-PQ 壓縮率高、適合億級，但要調 nprobe、定期 retrain，並用原向量 rerank 補精度。我會先用內存預算倒推索引類型，用 flat 抽樣做 ground truth 畫 recall–P99 曲線揀參數；多租戶按選擇率揀 pre/post/圖內過濾，大租戶獨立分區。」

## 常見追問（連答案）
1. **HNSW 點處理刪除/更新？** tombstone 標記、查詢跳過；刪多咗連通性差、recall 跌、浪費內存 → 定期 compaction/rebuild。更新 = 刪 + 插。好似 LSM 嘅 tombstone + compaction。
2. **efSearch 點揀？** 用 flat 結果做真值，量 recall@10 對 P99 曲線，揀達標（如 ≥0.95）嘅最細值；efSearch ≥ k。可以按請求分級（高價值查詢用大 ef）。
3. **recall 升咗答案冇變好？** ANN recall 只係貼近「embedding 空間嘅真最近鄰」；瓶頸可能係 embedding 模型、chunking 或冇 rerank。要量 end-to-end。
4. **IVF 數據漂移點算？** 新數據擠喺少數桶 → 桶大小不均、延遲飆、recall 跌。監控桶大小分佈，定期 retrain，用藍綠索引 + alias 切換（同 `05-mlops/embedding-reindex-ops.md`）。
5. **多租戶點保證唔漏、唔洩漏？** ACL 喺檢索層硬過濾；post-filter 要 over-fetch 兼處理空結果；ACL 變更要即時同步入索引 payload；大租戶獨立 namespace。

## 小練習
**題目**：5000 萬 chunk、1024 維 float32，要 P99 < 50ms、recall@10 ≥ 0.95，每節點內存 64GB。點設計？

**參考答案**：
- 先算：50M × 1024 × 4B ≈ 205GB，單機 float32 放唔落。
- 方案 A（單機）：IVF-PQ，nlist 約 3–6 萬（≈4–8×√N），PQ m=128 → 128B/條 ≈ 6.4GB 喺內存；原向量放 NVMe，只對 top-200 候選讀返嚟 rerank。調 nprobe 達到 recall 目標。
- 方案 B（分片）：2–4 個 shard，每 shard HNSW + int8（50M×1024B ≈ 51GB + 圖約 6.4GB，平均分落 shard），查詢 fan-out，延遲 = 最慢 shard，要控尾延遲。
- 如果模型支持 Matryoshka，截到 512 維再減一半。
- 驗證：抽樣 query 跑 flat 做真值，畫 recall–P99 曲線；最後睇 end-to-end 答案質量。
