# Hybrid Search + Rerank（RAG 檢索加深）

## 人話

純向量檢索擅於「意思接近」，但專有名詞、錯誤碼、訂單號、合約條款編號常被埋掉——關鍵字搜（BM25）反而穩。Hybrid = **兩邊都查，再合在一起**；Rerank = 先用便宜粗排撈一大票，再用貴一點的模型精排 Top 幾條再餵 LLM。後端類比：ES 的 `multi_match` + 業務重排，或「候選集很大 → 精排服務」。

## 面試短答

- **為什麼 hybrid**：Dense（embedding）抓語義／同義改寫；Sparse（BM25／SPL ADE 等）抓 exact token。RAG 裡兩者互補，尤其是 CEX 後端文件：產品代號、錯誤碼、API path 要 exact。
- **融合（fusion）**：常見 *RRF（Reciprocal Rank Fusion）*：不依賴兩邊分數可比，用名次 `1/(k+rank)` 加總再排序。也可加權分數，但要校準量綱。面試先講 RRF，再說「可加權／可學習 fusion」。
- **Rerank**：檢索先取較大候選（例如 50–100），用 cross-encoder／專用 rerank API 對 `(query, doc)` 打分，只留 5–10 進 context。延遲與成本換召回與精準。
- **Pipeline**：query 改寫／多 query → hybrid 召回 →（可選）metadata filter → rerank → 截斷進 prompt。Filter 仍要後端強制（見 multi-tenant）。
- **何時別硬上**：語料極短且全是關鍵字、或延遲預算極緊（可只 dense + 小 k）；文件極少時 hybrid 收益有限。

## 常見追問（含答案）

**Q：RRF 的 k 是什麼？分數還要正規化嗎？**  
A：k 是平滑常數（常見 60），避免 rank=1 權重爆炸、也讓中後段仍有一點分。RRF 用名次，**通常不必**把 BM25 與 cosine 拉到同一量綱；若改成加權分數融合才要校正／學習權重。

**Q：Rerank 放在 filter 前還是後？**  
A：先 **強制 tenant／權限 filter**，再對「合法候選」rerank。先 rerank 再濾會浪費算力，也有機率把不該見的文檔送進精排服務（合規風險）。

**Q：候選從 10 提到 100 再 rerank，一定更好嗎？**  
A：多數情況召回上限變高，但延遲／費用線性漲，且髒候選多會干擾。要用 golden set 畫「候選數 vs hit＠k／端到端分數／p95」曲線，找拐點，不要拍腦袋。

**Q：BM25 索引和向量索引如何保持一致？**  
A：同一 ingest pipeline：同一 `doc_id`／chunk_id、同一版本號寫入兩邊；刪改要雙寫或 outbox；切 alias／藍綠時兩邊一起切。評估時對同一 query 集比「只 dense／只 sparse／hybrid／+rerank」。

## 小練習

**題：** 客服 RAG 常搜「錯誤碼 E-1204」與「用戶說錢沒到帳但話很口語」。現狀只做 top-8 向量搜，E-1204 相關 runbook 常進不了 context。你如何改架構？列出你會看的 3 個指標。

**參考答案：**  
- **架構**：chunk 保留 `error_code` 等 metadata；ingest 同步 BM25（或 keyword 欄位）；查詢走 hybrid（vector + BM25）→ RRF；再對 top-50 做 rerank 取 8；對「看起來像錯誤碼」的 query 可加重 sparse 或先 keyword 必召回。  
- **指標**：(1) 檢索 hit＠8／MRR（golden 裡標好正確 runbook）；(2) 錯誤碼類子集的 hit＠8（單獨切片）；(3) 端到端：延迟 p95 + 成本／問，以及忠实度／「是否引用正確 runbook」（規則或 judge）。上線前 A/B 或陰影對比舊 pipeline。
