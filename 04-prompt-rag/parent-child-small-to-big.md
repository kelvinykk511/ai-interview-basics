# Parent-child／small-to-big：細塊檢索、大塊生成

> 2026-11-26 · AI warm-up · 工程面試向  
> 切塊基線見 [chunking-strategies.md](./chunking-strategies.md)；Contextual Retrieval 見 [contextual-retrieval.md](./contextual-retrieval.md)；Late Chunking 見 [late-chunking.md](./late-chunking.md)；RAPTOR 見 [raptor-hierarchical-summaries.md](./raptor-hierarchical-summaries.md)；組裝與長窗見 [long-context-vs-rag.md](./long-context-vs-rag.md)。**本篇講：按文件結構做「細單位召回、父段落／視窗送生成」、同 RAPTOR／CR／late chunking 有何分別、擴展策略同去重、ACL 同引用，以及 token 預算。**

## 人話

檢索同生成要嘅「塊大小」唔一樣。細塊（一句規則、一條錯誤碼）向量更尖，容易命中「VIP ETH 提現費幾多」；但把呢一句單獨丟給 LLM，佢睇唔到上一節嘅適用條件、例外同生效日，就會答得似是而非。大塊（整章）生成上下文夠，但向量被稀釋，精確題反而撈不準。

**Parent-child／small-to-big** 就係拆成兩步：**索引用小塊（child）**，命中之後**把對應嘅父段落、章節或前後視窗（parent／window）送進生成**。Sentence-window 係同一家人：句子做檢索單位，送出時擴成 ±N 句。Auto-merge 亦係：同一父節底下命中足夠多個兄弟，就升一級，只送一次父節，而唔係把重疊上下文貼三次。

後端類比：二次索引用細 key 查找，回表取完整行；唔好把整張表塞進索引，亦唔好只把主鍵值當 API 回應。面試官想聽：佢跟嘅係**文件結構**而唔係語義聚類、索引放邊層、擴展會唔會炸 token／洩漏權限、同 CR／late／RAPTOR 點疊加。

## 面試短答

- **解決嘅係「命中之後生成睇幾多」，唔係「點先搵對塊」。** 召回仍靠 child 嘅 dense／BM25／rerank。CR、late chunking 改善嘅係 child 向量或稀疏文字夠唔夠語境去被搵到；parent-child 唔代替呢一步。
- **跟目錄樹，唔係 RAPTOR。** Parent 係章、節、段呢啲作者寫好嘅結構。RAPTOR 按 embedding 相似度跨文檔砌摘要樹，內部節點仲係生成出嚟嘅摘要。兩者可以並存，唔好當成同一個開關。
- **通常只索引 child，parent 放 docstore。** 向量庫／倒排對住 `child_id`，payload 帶 `parent_id`、標題路徑、ACL、span。生成時按 id 取父文。把巨大 parent 同 child 一齊丟進同一個 HNSW，會同細粒度分數混在一起，又浪費內存。
- **擴展要有策略，唔係命中就整篇送。** 常見三種：（1）固定父節（一節 SOP）；（2）sentence-window（±N 句，N 要入版本）；（3）auto-merge（兄弟命中夠密就升一級）。先 rerank child，再擴 top 命中；用 `parent_id` 去重，同一父節只送一次。
- **寫死 token 預算同去重。** top-k 擴完好易把同一章貼幾次，或者把 80 頁手冊送進去，貴、慢、lost in the middle。上限按「system＋對話＋輸出」倒推；超預算就少擴、降層級，或只擴 rerank 後嘅前幾條。
- **ACL 跟擴展走，唔跟 prompt。** child 同佢可擴到嘅 parent／window 必須同一授權邊界。禁止把公開 FAQ 嘅句子視窗擴進相鄰嘅內部 SOP；拼接進同一個 parent 之前就要按密級切開。權限變更要同步到 child 同 parent。
- **Citation 指向命中嘅 span／child，唔只章名。** 父節係上下文，出處仍要落到具體規則句，方便對帳同排錯。父文同 child 不一致（只改咗其中一邊）要當 ingest 錯誤，唔好靠模型猜邊句先啱。
- **版本同評估要分兩截。** `chunk_strategy_version` 要寫明 child 粒度、parent 粒度、window 大小、merge 閾值。評估：child 嘅 Hit@k／MRR 量「搵唔搵到」；擴展後量答案完整度、faithfulness、token 同 P99。只報平均 Hit@k 會睇唔到「找對但答缺條件」或「擴太大引錯章」。

## 常見追問（連答案）

**Q1：已經做 Contextual Retrieval 或 late chunking，仲要唔要 parent-child？**  
A：通常要，因為職責不同。CR／late 幫你把對嘅細塊搜出來；parent-child 幫你喺生成時補條件同前後文。可以疊加：child 用標題 prepend／CR／late 編碼，命中後仍取父節。但唔好無節制把父節、RAPTOR 祖先、語境段落三重塞入同一窗。

**Q2：點解唔直接把 parent 都 embed，一齊檢索？**  
A：可以做「粗召回用節、精排用句」嘅分層，但要當兩個 collection 或兩段分數，唔好混在同一個 HNSW 裡用同一條閾值。默認只索 child 更乾淨。若只索引 parent，精確費率／錯誤碼會被整節向量稀釋，回到切得太粗嘅老問題。

**Q3：三個 child 命中同一章，要送三次嗎？**  
A：唔要。按 `parent_id` 合併成一次父節；若用 sentence-window，重疊視窗要拼成連續一段再送。Auto-merge 喺兄弟命中密度高時直接升到父節，通常比貼三個重疊窗更省、更不易打散模型注意力。

**Q4：擴展會唔會洩漏無權內容？**  
A：會，若 parent／window 跨過授權邊界。例如同一檔誤把公開說明同內部處理手冊拼成一節，句子檢索命中公開句，±N 句就帶出內部流程。要喺建樹時按 ACL 切 parent，檢索層硬過濾，禁止靠「模型唔好讀旁邊」擋洩漏。

**Q5：文件改一行，要重做啲乜？**  
A：受影響嘅 child 要重切／重嵌，對應 parent 文檔要一齊更新，映射唔可以指住舊 span。切塊策略或 parent 粒度一變，視作新 `chunk_strategy_version`，走新 collection／藍綠，唔好同一庫混兩套父子定義。高變更事實（費率）仍應以受控表為準，parent 只補條件敘述。

## 小練習

**題：** 提現 SOP 同幫助中心混在同一批文件。同事方案：向量庫只 embed 整章 parent；每次命中就把 80 頁 SOP 全文送進模型；同一章被三條細規則打中就貼三次；公開 FAQ 同內部處理步驟被拼成同一個 parent，靠 prompt 叫模型「唔好引用內部」；citation 只標「第 4 章」；無 token 上限。你會點評，同埋會點落地？

**參考答案：**  
1. **只索引 parent 不夠準。** 費率、錯誤碼、單條規則用 child 做 dense＋BM25＋rerank。  
2. **禁止命中就送 80 頁。** 擴到所屬節或 sentence-window，先 rerank 再擴，寫死 token 預算，超預算就少擴。  
3. **同一 `parent_id` 只送一次**；夠密就 auto-merge 升一級，唔重複貼。  
4. **公開同內部要喺建樹前切開**，parent／window 不可跨 ACL；prompt 擋唔住洩漏。  
5. **引用落到具體 child／條款句**；ingest 保證父子一致；費率數字仍查受控表。用「搵對細塊」同「擴展後答得完整且唔越權」兩組 slice 驗收。
