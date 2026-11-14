# Contextual Retrieval（chunk 補語境再索引）

> 2026-11-14 · AI warm-up · 工程面試向  
> 切塊策略同標題路徑見 [`chunking-strategies.md`](./chunking-strategies.md)；hybrid + rerank 見 [`hybrid-search-rerank.md`](./hybrid-search-rerank.md)、[`reranker-production.md`](./reranker-production.md)；向量版本／增量重嵌見 [`../05-mlops/embedding-refresh-drift.md`](../05-mlops/embedding-refresh-drift.md)；**本篇講：chunk 單獨編碼會丟失「屬於邊篇文、邊個產品」嘅語境，點用 chunk 專屬短語境同時改善 dense 同 BM25。**

## 人話

傳統 RAG 先把文件切成小塊，再逐塊 embed、逐塊建 BM25。一塊如果只寫「手續費為 0.1%，VIP 另計」，向量同關鍵詞都唔知呢句係 BTC 仲係 ETH、係鏈上提現仲係內部劃轉。用戶問「BTC 鏈上提現費幾多」，呢塊好可能排唔上。

後端類比：你把一條費率規則從配置檔剪出嚟單獨建索引，卻冇帶 `product_id`／路徑——倒排同語義都對唔到。**Contextual Retrieval**（Anthropic, 2024）嘅做法係：對每一塊，用較平嘅 LLM 看着整篇（或所在章節）生成一段 50–100 token 嘅短語境，**prepend 到 chunk 正文**，然後先去做 embedding，亦用同一段文字建 BM25。語境寫進被索引嘅文字裡，唔係只放 metadata。

面試官想聽：同「只 prepend 標題」有咩分別、點控制成本同版本、語境寫錯會點、同 late chunking／parent-child 點分工。

## 面試短答

- **根因係孤立編碼，唔係切得太碎咁簡單。** 細 chunk 對精確檢索有利，但塊內指代（「上述」「該幣」「手續費為」）要靠文件級語境先解釋得清。只加大 chunk 會稀釋向量、撐爆 context，治標唔治本。
- **兩條腿一齊改文字。** Contextual Embeddings = 語境+正文先 embed；Contextual BM25 = 同一串文字入稀疏索引，令「BTC」「Q2」「提現」呢啲只出現喺標題／前文嘅詞，稀疏檢索都搵到。只改向量、BM25 仍用原文，關鍵詞增益會少一截。
- **語境要 chunk-specific，唔好只貼一篇摘要。** Anthropic 自己試過「每塊前加同一段文件摘要」，增益好有限；有效嘅係「呢一塊喺全文入面係講緊咩」。Prompt 大意：給全文 + 呢一塊，只要一段短語境、不要其他輸出。
- **便宜基線先做。** 標題路徑、產品名、條款編號 prepend 進正文（唔只寫 metadata）成本幾乎零，好多「指代不清」已經改善。LLM 語境係下一步，用 slice eval 證明有增益先全面鋪。
- **成本靠文件前綴快取，而唔係每塊重傳全文。** 同一份文件嘅 N 個 chunk，document 前綴重複。用 prompt cache（或等效前綴快取）只付一次文件 I/O。Anthropic 用 Haiku + cache 嘅估算（800-token chunk、8k 文件、約 100-token 語境）係一次性約 **$1.02 / 百萬文件 token**——呢個係 2024 年佢哋標價同假設下嘅數字，落地要按你嘅模型同價格重算，當數量級唔好當常數。
- **版本要釘死，當編碼參數。** 寫入 `contextualizer_model_id` + `context_prompt_version` + 源 `content_hash` + `embed_version`。正文變 → 只重做受影響文件；prompt／contextualizer 模型變 → 視作新編碼參數，走新 collection／藍綠（見 embedding-refresh），禁止同一 index 混「有語境／無語境」或兩個 prompt 版本。
- **語境可以寫錯，要抽檢。** 模型把 ETH 費率塊標成 BTC、把已廢規則寫成仍生效，等於向索引注入錯誤關鍵詞，召回會「更自信地錯」。抽樣人工或規則校驗（產品名、日期、條款號必須能喺源文對上）；唔好把跨租戶、PII、其他文件嘅術語表偷偷塞進語境。
- **長文唔能盲傳「整本」。** Contextualizer 本身有 context 上限。80 頁 SOP 應以章／節做窗口（parent section + 標題路徑），而唔係每次塞全文。呢個窗口版本都要入 `chunk_strategy_version`。
- **同 rerank、parent-child 唔互斥。** CR 係 ingest 時改索引文字；rerank 係 query 時精排（Anthropic 喺自己嘅評測上：contextual embedding+BM25 把 top-20 檢索失敗率由 5.7% 降到 2.9%，約 49%；再加 rerank 到 1.9%，約 67%——**係佢哋語料同 1−Recall@20，唔係你嘅保證**，要用自己嘅 golden／shadow）。Parent-child／small-to-big 改善嘅係「命中之後生成睇幾多」，唔代替「先搵對塊」。
- **CEX 直覺。** 費率、限制、狀態機呢類塊經常係短句＋指代。上幣／改名之後，語境裡嘅產品別名要跟文件走；語境寫錯幣種係資金答錯級事故，要比「少召回一點」更嚴。

## 常見追問（連答案）

**Q1：已經 prepend 章節標題，仲要 LLM 語境做乜？**  
A：標題路徑係便宜且應該先做嘅基線，能補「呢塊屬於邊一節」。LLM 語境補嘅係塊內唔會出現、但檢索 query 會用嘅具體實體同限定（邊隻幣、邊條鏈、對上一個季度嘅基數、呢條規則只適用於 VIP）。兩者可以疊加。冇 eval 就全面上 LLM，係浪費。

**Q2：只做 Contextual Embeddings、BM25 用原文，得唔得？**  
A：得，但少咗稀疏側嘅增益：用戶用精確代號、條款號、函數名時，呢啲詞可能只喺語境裡出現。工程上應用**同一串** `context + chunk` 寫入 dense 同 BM25；融合同 rerank 仍然按 hybrid 篇嚟。

**Q3：點控制百萬 chunk 嘅 contextualizer 成本同延遲？**  
A：呢步喺 ingest，唔喺用戶請求路徑。文件前綴 cache、細／平價模型、按 `content_hash` 增量（正文不變唔重做）、熱點文件（費率、上幣公告）優先。Prompt 或模型變更先抽樣 shadow／slice eval，確認增益再全量重建，唔好 silently 覆寫現役 index。

**Q4：生成時應該送「語境+chunk」還是原文？**  
A：語境通常有用（減少模型再推論「呢句講緊邊個」），但必須當輔助、以源 chunk／引用校驗為準。若語境同原文衝突，信原文並告警 contextualizer。Citation 指向源段落，唔好引用模型寫嘅語境句。

**Q5：query 側要唔要加同樣語境？**  
A：唔好亂加文件語境——用戶 query 本身已帶意圖。要保持同 embedding 模型聲明嘅 query／passage prefix 對稱（見 embedding 選型、refresh 篇）。Query 改寫（指代消解、補產品名）係另一條鏈，版本要分開釘。

## 小練習

**題：** 提現費率知識庫。典型 chunk 原文只有「手續費為 0.1%，VIP 另計」。而家有三個提案：（a）只把 metadata 寫上 `product=BTC_ONCHAIN_WITHDRAW`，正文同 BM25 唔改；（b）用 Haiku 對每塊生成語境再 embed + BM25，但 80 頁 SOP 每次無 cache 整本送入，而且抽樣發現有塊 ETH 費率被寫成「關於 BTC 提現」；（c）同事話 Anthropic 報咗失敗率降 49%，可以直接全量上、唔使 shadow。你會點做？

**參考答案：**  
1. **(a) 不夠。** Metadata 可用於過濾同權限，但唔進入 embedding／BM25 文字就解決唔到「塊內無幣名、無業務線」嘅召回。最少把標題路徑／產品名 prepend 進**正文**。  
2. **(b) 方向對，作業錯。** 要上 contextual embeddings + contextual BM25；長文按章節窗口而唔係盲送 80 頁；啟用文件前綴 cache；ETH 被標成 BTC 必須擋——語境內嘅產品名／鏈名要能喺源文對上，對唔上就拒寫或回退到只 prepend 標題，並告警。呢類錯誤比漏召回更嚴重。  
3. **(c) 唔可以照抄數字。** 49%／67% 係 Anthropic 自己語料、top-20 失敗率。應用費率／幣種／exact-id slice 做離線 golden + shadow（見 retrieval-eval、rag-shadow、online-retrieval-eval），過閘先切 alias。  
4. **版本：** `context_prompt_version` + contextualizer 模型釘進索引 metadata；同現役混寫禁止；生成引用以源 chunk 為準。
