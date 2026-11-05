# Long Context vs RAG：成份文件塞入去，定係先檢索？

> 2026-11-05 · AI warm-up · 工程面試向

## 人話

而家好多模型支援 20 萬甚至過百萬 token 嘅 context。於是有人問：「仲使唔使做 RAG？直接將成份文件塞入 prompt 咪得？」

答案係：**睇數據量、成本、延遲同準確度要求**，兩者係互補唔係取代。

- **Long context**：簡單（冇 chunking、冇 index、冇檢索漏召回），模型睇到全局，適合「讀晒呢份 100 頁合約再答」。但每次請求都要為全部 token 付費、prefill 時間長（TTFT 高），而且內容越長，模型對**中間位置**嘅資訊越容易忽略（lost in the middle）。
- **RAG**：只送相關片段，平、快、可以覆蓋幾百萬份文件、可以做權限過濾同引用出處。但檢索漏咗就答唔到，需要維護 index。

## 面試短答（30 秒版）

> 語料細（幾萬到十幾萬 token）、靜態、要全局理解 → 可以直接 long context，配合 prompt cache 攤分重複前綴嘅成本。語料大、經常更新、多租戶要權限隔離、要低延遲同可追溯引用 → RAG。實務最常見係**混合**：RAG 先揀出候選文件，再將完整文件（而唔係細 chunk）放入較長 context，等模型有足夠上下文。決策要用 eval 數據講：同一個 golden set 比較準確率、成本/請求、P95 TTFT。

## 常見追問（連答案）

**Q1：Long context 有咩隱藏成本？**
A：
1. **錢**：按 input token 計費，10 萬 token × 每日 10 萬請求非常貴；prompt cache 只幫到**前綴完全相同**嘅部分。
2. **延遲**：prefill 計算量隨長度上升，TTFT 可以去到幾秒到幾十秒。
3. **Serving 資源**：KV cache 隨長度線性增長，長請求佔顯存，降低同一 GPU 可以並發嘅請求數。
4. **質量**：「needle in a haystack」測試通過唔代表真實任務好——多個事實要跨段落推理、或者資訊放喺中段時，準確度會跌。

**Q2：點樣評估 long context 夠唔夠好？**
A：唔好只信廠商嘅 needle 測試。用自己嘅 golden set，將答案所在位置（開頭/中間/結尾）同 context 長度做分組（slice），睇準確率隨位置同長度嘅變化；再加多跳問題（需要結合兩段資訊）。同時量度成本同 P95 延遲，畫出「準確率 vs 成本」曲線再決定。

**Q3：「混合」具體點做？**
A：常見模式：
- **Small-to-big retrieval**：用細 chunk 做檢索（精準），命中後擴展返整個章節或者原文件送入模型（上下文完整）。
- **檢索後長 context**：RAG 揀 top 5 份完整文件（總共 5 萬 token），而唔係 top 20 個 300 token 碎片。
- **Context caching**：固定嘅大文件（產品手冊、規則）放 prompt 前綴並開 prompt cache，用戶問題放尾。

**Q4：多租戶 / 權限場景點解仍然要 RAG？**
A：Long context 要你**先決定塞乜入去**——呢個決定本身就係一次檢索同權限過濾。如果將所有租戶嘅文件塞入同一個 prompt，就會有越權洩漏風險。權限過濾必須喺送入模型之前做，所以檢索層（帶 tenant/ACL filter）始終需要。

**Q5：文件經常更新點算？**
A：Long context 每次請求讀最新文件，天然冇 index lag，但如果用 prompt cache，文件一改 cache 就失效。RAG 要處理增量 reindex 同新鮮度。如果文件細又常改，long context（唔 cache）可能更簡單；如果大又常改，就要 RAG + 增量索引。

## CEX 例子

- **上幣規則 / 費率表問答**（總共 3 萬 token，每月改一兩次）：直接 long context + prompt cache，簡單又準。
- **客服知識庫 + 用戶工單歷史**（幾百萬份、每日新增、按用戶隔離）：RAG，tenant filter 喺檢索層做。
- **合規審閱一份 200 頁監管文件**：long context 單次讀晒；但要求模型逐條引用頁碼，並抽樣人手核對中段條款。

## 小練習（題目 + 參考答案）

**題目：** 產品經理話：「新模型支援 100 萬 token，我哋將成個客服知識庫（80 萬 token）每次塞入去，唔使再維護 RAG。」每日 5 萬個請求。你點回應？

**參考答案：**
1. **成本先算**：80 萬 × 5 萬 = 每日 400 億 input token。就算 prompt cache 命中打大折扣，都要用實際價錢計清楚，多數情況遠高過 RAG（每請求幾千 token）。
2. **延遲**：80 萬 token prefill（冇 cache 命中時）TTFT 可能幾十秒，客服場景接受唔到；cache 有 TTL，低峰期過咗期又要重新 prefill。
3. **質量**：喺自己嘅 golden set 上，按答案位置同多跳問題做對比，好可能中段內容準確率下降；RAG 可以附出處方便核對。
4. **更新同權限**：知識庫一改，全量前綴 cache 失效；如果有按地區/用戶等級嘅內容，塞晒入去會有越權風險。
5. **建議**：做一次 A/B eval（同一 golden set，比較準確率、成本、P95 TTFT）。可能嘅折中：保留 RAG，但改成 small-to-big，送入完整相關文章；最常用嘅核心規則（幾萬 token）放固定前綴 + prompt cache。
