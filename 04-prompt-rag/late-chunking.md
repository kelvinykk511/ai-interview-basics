# Late Chunking（先整篇編碼再切向量）

> 2026-11-14 · AI warm-up · 工程面試向  
> 切塊同 parent-child 見 [`chunking-strategies.md`](./chunking-strategies.md)；Contextual Retrieval 見 [`contextual-retrieval.md`](./contextual-retrieval.md)；embedding 模型選型見 [`embedding-model-selection.md`](./embedding-model-selection.md)；**本篇講：另一種補「孤立 chunk 丟失語境」嘅方法——用長上下文 embedding 模型先對全文做 token 編碼，再按 chunk 邊界做 pooling。**

## 人話

普通流程係 **early chunking**：先切塊，再讓 embedding 模型分別睇每一塊。塊同塊之間冇 attention，所以「手續費為 0.1%」呢句嘅向量睇唔到前文嘅「BTC 鏈上提現」。

**Late chunking**（Jina, 2024）調轉順序：用一個**長上下文** embedding 模型，先把**整篇（或模型能裝下嘅最長一段）**過 transformer，得到每個 token 已經「睇過全文」嘅向量；然後先按你原本嘅切塊邊界，對屬於呢一塊嘅 token 做 mean pooling，得到呢一塊嘅向量。Chunk 邊界仍然要，只係用喺 pooling，唔係用喺「分開編碼之前」——所以叫 late。

後端類比：early 像每段獨立建文件摘要；late 像先對整章做語境化編碼，再按段落匯總。面試官想聽：佢只幫 dense、對 BM25 冇自動好處、文件超模型窗口點辦、同 Contextual Retrieval／parent-child 點揀。

## 面試短答

- **解決嘅係「向量孤立」，唔係「生成時上下文不夠」。** 命中之後要俾 LLM 睇完整一節，仍然用 parent-child／small-to-big。Late 改善召回；parent 改善生成；可以疊加。
- **前提係長上下文 embedding 模型。** Token 要能在同一段 forward 裡互相注意。短窗模型（512／8192 裝唔下你嘅文件）做唔到「全文語境」，只能對窗口內有效。
- **文件長過模型窗口，仍要先切大段。** 論文嘅講法係 encode 整篇或「能裝下嘅最大一段」。40k token 嘅 SOP、模型只有 8k，就要按章切成 ≤8k 嘅窗口再 late chunk——語境範圍係呢一章，唔係整本手冊。窗口策略要入版本。
- **只改 dense，唔改稀疏文字。** BM25 見到嘅仍係原文。短句缺產品名時，稀疏側依然弱。實務常係：dense 走 late chunking，BM25 至少 prepend 標題路徑，或對稀疏側做 Contextual BM25。
- **Query 獨立編碼，唔做 late chunking。** 用戶 query 好短，沒有「所屬文件」。非對稱係正常嘅：document 向量帶文件語境，query 向量帶用戶意圖。唔好把 query 同某篇 SOP 一齊 late encode。
- **切塊邊界改咗，通常要重新 forward。** 理論上若你存低全部 token 向量，可以只重新 pool；實務上 token 級向量太佔空間，很少持久化，所以改 chunk 大小／重疊等於重嵌呢篇文。成本按「文件數 × 長窗 encode」估，唔係「chunk 數 × 短 encode」——長窗一次可能貴過 N 次短窗，要量。
- **唔可以同 early-chunk 向量混庫。** 兩種幾何唔同，即使模型 id 一樣。要獨立 collection／alias，`embed_version` 寫明 `late_chunking=true` + 窗口策略；query 側用同一模型嘅普通 encode（唔 pooling 全文）。
- **Mean pooling 仍會稀釋。** 塊太長，重要 token 被平均掉；所以切塊策略依然重要，late 唔係「切越大越好」。亦依賴模型本身嘅長文檔品質，唔好假設任何 long-context embedder 都自動贏。
- **同 Contextual Retrieval 點揀。** CR 改**文字**，dense 同 BM25 都受惠，但要多一次 LLM 呼叫，仲有語境寫錯嘅風險。Late 改**編碼過程**，無需生成語境、少一個寫錯實體名嘅來源，但要長窗 embedder、幫唔到 BM25、超窗要再切。可以用 eval 對打：標題 prepend（基線）→ late 或 CR → 兩者疊加（例如 late dense + contextual BM25）。用自己嘅 slice（費率短句、條款號、跨段指代）決定，唔好按部落格數字拍板。
- **CEX 直覺。** 短規則句、跨段「上述三項」、幣種只在章節標題出現——正正係孤立編碼嘅重災區。選哪條路都要：版本釘死、增量按文件 hash、上線走 shadow，同 embedding-refresh 同一套作業。

## 常見追問（連答案）

**Q1：Late chunking 係咪唔使再切塊？**  
A：要切。模型產出嘅係 token 序列，檢索單位仍係 chunk。只係切塊發生在 pooling 時，令每塊向量帶有窗口內嘅跨句語境。

**Q2：同一個模型，early 同 late 嘅向量可以放同一個 HNSW 嗎？**  
A：唔可以。同一 query 向量對兩種 document 向量嘅近鄰不可比，會「靜靜變差」。獨立 collection，alias 切換；query cache key 含 `late_chunking` 同模型版本。

**Q3：文件 40k、模型 8k，late chunking 仲有冇用？**  
A：有，但語境半徑變成「8k 窗口／章」而唔係全書。按標題切章，章內再 late。跨章指代（「見第一章）仍可能斷，呢類要用標題路徑、交叉引用展開，或 CR 在語境裡寫明指向。

**Q4：同 prompt 裡塞長文、同 CR，延遲成本點比？**  
A：Late 嘅額外成本在 **ingest 嘅長窗 embed**，query 路徑仍係一次短 query encode + ANN，唔增加 TTFT。CR 嘅額外成本在 ingest 嘅 LLM 語境生成（可用文件前綴 cache）。兩者都唔應該喺用戶請求裡現做。生成階段塞整本係另一條題（見 long-context vs RAG），KV／TTFT／成本同檢索無關，唔好混。

**Q5：改 chunk 重疊或大小，可唔可以只重算 pooling？**  
A：只有你持久化咗該文件嘅 token 向量先得；大多數系統冇存，就要再跑 transformer。設計上把 `chunk_strategy_version` 同 `late_chunking` 綁在一起，策略變更走重建 job，唔好在現役 index 混兩種邊界。

## 小練習

**題：** 客服 SOP 平均 40k token，現役 embedder 窗口 8k。主管聽完 late chunking，提出四句判斷，請逐條對或錯，並寫你會落地嘅方案：（1）「上咗 late chunking 就可以停 BM25」；（2）「query 都要同命中文件一齊做 late encode 先對稱」；（3）「模型窗口 8k，所以 late 對呢批 SOP 完全無用」；（4）「我哋把舊 early 向量同新 late 向量一齊 upsert 進同一個 collection，先灰百分之十流量」。

**參考答案：**  
1. **錯。** Late 只豐富 dense。費率、錯誤碼、條款號仍然需要 BM25／hybrid。最少畀稀疏側標題路徑或 contextual BM25。  
2. **錯。** Query 應獨立短 encode。把 query 同某篇 SOP 綁住再 encode 既做唔到（檢索前唔知邊篇），亦會破壞近鄰幾何。  
3. **錯。** 可按章切成 ≤8k 窗口，章內 late，仍然能讓「手續費為 0.1%」吃到本章標題同幣種。只係唔能吃到跨章語境。  
4. **錯。** 唔可混庫。應建新 collection，shadow／slice eval（短規則句、跨段指代、exact-id）過閘後切 alias；灰發佈嘅係 alias／流量切到**純 late 庫**，唔係同一個 index 混幾何。  
5. **落地：** `embed_version` 標明 late_chunking + 窗口策略；按章切窗；dense late + BM25 帶路徑；改切塊就重建該批文件；引用同生成仍可用 parent 擴段。
