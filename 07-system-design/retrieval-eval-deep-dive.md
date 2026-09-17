# Retrieval Eval 深挖（Hybrid / Rerank 怎麼證明有用）

## 人話

改了 hybrid、開了 rerank，「感覺答案變好」不夠面試也不夠上線。要把 **檢索層**和 **生成層**拆開量：先證明「該找的段落有沒有進候選」，再證明「模型有沒有據實用上」。後端類比：先測搜尋相關性，再測組裝 API 的業務正確性——不要只看一個「整體滿意度」。

## 面試短答

- **標註單位**：每題至少標「必須命中的 doc_id／chunk_id」（可多個），不要只標最終一句話——否則分不清是檢索還是生成的問題。
- **檢索指標**：Recall＠k、Hit＠k（有沒有任一金標進 Top-k）、MRR、nDCG＠k（有序時）。Hybrid／rerank 實驗要固定同一 k 定義（rerank 前候選 vs 進 prompt 的 k）。
- **切片分析**：專有名詞／錯誤碼、長尾問法、多跳（答案跨兩段）、負例（應拒答／應說不知道）。總平均漲、錯誤碼切片跌 = 回歸。
- **對照實驗**：baseline dense → +BM25 → +RRF → +rerank；每次只動一層。記錄候選寬度、延遲、費用。
- **與生成聯立**：檢索變好但端到端沒變 → 查 prompt 有沒有逼模型「只根據資料、否則拒答」、context 是否被截斷、citation 是否後端校驗。
- **線上**：點踩、重問、「查看來源」點擊、人工抽檢；離線 golden 過門檻才能全量。

## 常見追問（含答案）

**Q：沒有大量標註怎麼辦？**  
A：先做小而難的 golden（50–100）：從客訴／工單抽；用「已知正確 doc」當弱標；LLM 幫擬問題再人工改。關鍵是**穩定可回歸**，不是一開始就要上萬題。

**Q：Hit＠k 提升但用戶仍嫌胡謅？**  
A：金標進了 Top-k 不代表進了 prompt（被截斷），或模型忽略檢索。補：進 prompt 的 hit、citation 後端校验通过率、忠实度 judge；必要時強制「無來源不答」。

**Q：Rerank 模型換了如何回歸？**  
A：同一候選輸入重跑 golden；比 nDCG／Hit＠最終 k；看切片；設門檻＋陰影流量。Rerank 是獨立服務版本，要跟 embedding／chunk 版本一樣進變更紀錄。

**Q：k 選 5 還是 20？**  
A：進 LLM 的 k 受 context 與噪音影響：太大稀釋、太小漏召回。用曲線：檢索 hit 與端到端分数、p95 延遲 vs k；hybrid+rerank 常是「寬召回 + 窄進模」。

## 小練習

**題：** 你們上了 RRF hybrid，整體 Hit＠8 從 0.62 → 0.71，但「訂單／Tx hash 類」問題客訴變多。怎麼查？給一個你會加進 CI 的檢查。

**參考答案：**  
1. **切片**：把 golden 分成語義問 vs exact-id 問；分開報 Hit＠8／MRR。猜因：fusion 權重偏 dense、或 BM25 欄位沒索引 hash／orderId、或 rerank 偏好長段落蓋過短 ID 段。  
2. **對照**：只 sparse / 只 dense / hybrid / +rerank 四條曲線看 exact 子集。  
3. **CI**：每次改檢索配置，跑 golden；**exact-id 子集 Hit＠8 不得低於門檻**（例如 0.9），否則 fail build；並輸出失敗題的 Top-8 id 方便排查。
