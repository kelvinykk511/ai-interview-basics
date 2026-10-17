# RAG 改動上線：replay → shadow → canary

> 補充 `online-llm-eval-canary.md`，專講 retriever / index / reranker 改動點樣安全驗證。

## 人話
你換 embedding 模型、改 chunk size、加 reranker，點知線上變好定變差？唔可以直接全量。三步：
1. **離線**：golden set（recall@k、nDCG、答案 judge）+ **replay**（錄低歷史 query 重放，做 regression gate，可重複）。
2. **shadow**：鏡像實時流量，新 pipeline 喺背後跑，結果唔畀用戶睇。類比 DB 遷移雙寫比對、流量回放。
3. **canary / A-B**：小流量真用戶，睇用戶指標，guardrail 觸發自動回滾。

### shadow 比較乜
- 檢索層：top-k overlap（Jaccard / RBO）、空結果率、新 pipeline 延遲同成本。差異大嘅 query 抽出嚟審。
- 生成層（只對差異集）：生成答案做 pairwise judge（記得 swap 位置、抽人工校準），唔使全量生成，慳錢。
- 冇 ground truth：用差異集 + judge、citation 覆蓋/faithfulness、abstention 率做代理；canary 再睇 thumbs、重問率、點擊引用、轉人工率。

### 陷阱
- **無副作用**：shadow 唔寫 cache、唔觸發 tool/寫操作、唔計 billing、log 分開標記。
- **權限**：shadow 都要做 ACL；新索引權限同步唔齊，shadow 好睇但上線就洩漏。
- **新鮮度對齊**：新索引 backfill 未追上，比較唔公平 → 對齊 snapshot。
- **容量**：shadow 雙倍檢索負載 → 採樣 5–10%、async fire-and-forget、低優先級、超時即丟，唔可以拖慢主路徑。
- **統計**：預先定 guardrail 指標同最少樣本量；按 query 類型分層（精確查詢 vs 語義、熱門 vs 長尾），避免平均值掩蓋退化。

## 面試短答
「RAG 改動我分三步：離線 golden set + 歷史 query replay 做 regression gate；再 shadow 一部分實時流量，新舊 pipeline 並行，比 top-k overlap、空結果率、延遲，對差異 query 做 pairwise judge 同人審，shadow 全程無副作用兼守 ACL；最後 canary 1%→5%→25%，用 thumbs、重問率、轉人工率、P99 做 guardrail，出事用 feature flag 或 index alias 秒切返。」

## 常見追問（連答案）
1. **點解唔直接 A/B？** RAG 退化常係少數 query 嚴重錯（權限、錯數字），A/B 用戶指標噪聲大、要時間，而且用戶已經見到壞答案。shadow 零用戶影響，先篩走明顯問題。
2. **overlap 低 = 變差？** 唔係，只代表「唔同」。overlap 用嚟揀邊啲 query 要審，好壞靠 judge / 人工 / 下游指標。
3. **shadow 結論點變上線 gate？** 預先定門檻：空結果率、P99、差異集 judge 勝率 ≥ X、嚴重錯誤（權限/數字）= 0；過咗先 canary 逐步放量，guardrail 觸發自動回滾。
4. **replay 同 shadow 分別？** replay 用歷史 query，可重複、適合 CI regression，但睇唔到新分佈同新鮮度；shadow 係實時真分佈。兩樣都要。

## 小練習
**題目**：客服 RAG 由「向量檢索」改成「BM25 + 向量 hybrid + cross-encoder rerank」。寫上線 plan。

**參考答案**：
1. 離線：歷史工單做 golden set，量 recall@k / nDCG + 答案 judge；replay 過去 2 週 query 做 regression。
2. shadow 10% 實時流量：async、無副作用、同一套 ACL；計 overlap/RBO、空結果率；差異 query 做 pairwise judge + 人審；量 rerank 帶嚟嘅 P99，設超時降級（超時就用無 rerank 結果）。
3. canary 1%→5%→25%：睇 thumbs-down、重問率、轉人工率、P99、每查詢成本；guardrail 觸發用 flag 回滾。
4. 分層檢查：訂單號、產品型號呢類精確查詢（BM25 強項）唔可以退化。
