# RAPTOR／層次摘要：按相似度砌樹，唔係 GraphRAG 換皮

> 2026-11-24 · AI warm-up · 工程面試向  
> GraphRAG 見 [graphrag-entity-linking.md](./graphrag-entity-linking.md)；parent-child 見 [chunking-strategies.md](./chunking-strategies.md)；迭代檢索見 [agentic-iterative-retrieval.md](./agentic-iterative-retrieval.md)；路由見 [query-corpus-routing.md](./query-corpus-routing.md)。**本篇講：RAPTOR 點樣用遞迴聚類＋摘要做層次檢索、collapsed tree 同 traversal 點揀、同 GraphRAG／parent-child 有何分別，以及摘要節點點解唔可以當受控事實。**

## 人話

普通 RAG 係「問句變成向量，撈最似嘅原文塊」。呢招對「ETH 喺某條鏈嘅 VIP 提現費幾多」通常夠——答案就喺某一行。但若問題係「呢份季度事故覆盤主要有邊幾類根因？」，可能冇任何一塊寫齊全貌：要先有中層摘要先搵到主題，再落到細節。

**RAPTOR**（Recursive Abstractive Processing for Tree-Organized Retrieval，Sarthi et al., 2024）大致係：把語料切成 leaf chunk 並 embedding → 按向量相似度聚類（論文用降維後嘅 GMM，可以軟歸屬，一塊入多個 cluster）→ 用 LLM 為每個 cluster 寫 abstractive 摘要 → 再把摘要 embedding、再聚類、再摘要，層層向上直到唔再適合分。查詢有兩條路：由根向下嘅 **tree traversal**，或者把所有層級節點攤平一齊做向量檢索（**collapsed tree**；論文發現通常更好）。

後端類比：唔係按目錄樹（章節）做 rollup，而係按「內容似唔似」砌出一棵摘要樹。GraphRAG 嘅 community report 亦係層次摘要，但分組信號係實體關係圖，唔係 chunk embedding。面試官想聽：同 GraphRAG／parent-child 有何分別、摘要可唔可以當費率真源、增量更新、ACL，以及 collapsed 同 traversal 點揀。

## 面試短答

- **先問清問題類型，唔好默認全庫建樹。** 單跳事實、錯誤碼、費率行，優先受控目錄／SQL／hybrid RAG。RAPTOR 擅長「跨塊主題、長文唔同粒度」。兩者可以並存，由路由決定。
- **RAPTOR ≠ GraphRAG。** RAPTOR：無抽實體，按 embedding 相似度聚類，摘要講「聲近嘅塊」。GraphRAG：先抽取再按圖分社區，摘要講「互相有關係嘅實體」。詞彙唔似但實體相連嘅跨文檔題，GraphRAG 較對口；長文主題、相似文本嘅層次 rollup，RAPTOR 通常更平、建庫亦較平價（唔使先抽取）。
- **亦唔等於 parent-child／small-to-big。** Parent-child 跟文件結構：小塊檢索、命中後把父段落／整節送進 context。RAPTOR 可以跨文檔把語義近嘅塊歸一堆，樹唔等於目錄。
- **內部節點係摘要，唔係 source of truth。** 摘要會寫滑、過時、把多塊口徑捏成一句。生成仍要能回落到 leaf／受控表做 citation；費率、限額、生效日只認受控表，唔認祖先摘要。
- **查詢可先考慮 collapsed tree，但要管預算同去重。** Collapsed：所有層級入同一個向量庫，按相似度取到 token 預算，抽象層級由問句自己揀。Traversal：根層 top-k，再只喺子節點裡揀，第一層行錯會 cascade（同迭代檢索認錯實體一樣）。無論邊條，父摘要同子 chunk 一齊中會重複佔窗，要去重／多樣性，唔好無上限拼接。
- **建庫同增量都貴過一次 embedding。** 每層都要聚類＋LLM 摘要。加一篇或改一行，祖先摘要可能 stale，聚類邊界都可能變。實務：高變更源（費率）唔靠摘要當真值；受影響祖先失效或定期重建。版本釘 `raptor_index_version`（embed 模型、聚類參數、摘要 prompt、層數）。
- **ACL 要喺建樹前硬過濾。** 一個 cluster 若混入無權 SOP，摘要等於把機密寫進共用週報。分租戶／分密級建樹，或只對已授權子集聚類；唔可以先全球摘要再靠 prompt 擋洩漏。
- **唔取代 hybrid／BM25／迭代檢索。** 精確代號、產品別名仍要 lexical；需要關係跳轉時仍可能要圖譜或受控表。評估要分 slice：事實題、主題題、摘要相對 leaf 嘅 faithfulness／stale、ACL。只報平均 Hit@k 會把「主題題全軍覆沒」或「摘要答對但引用錯費率」沖淡。

## 常見追問（連答案）

**Q1：已經有 GraphRAG，仲要唔要再上 RAPTOR？**  
A：通常唔使兩套都默認上。先用主題／跨塊 slice 衡量現役缺口。若失敗其實係「詞彙唔似但實體相連」嘅跨文檔關係，GraphRAG 較對口。若係長文或相似文本嘅層次主題，RAPTOR 更平（唔使抽取）。兩者都係層次摘要，成本疊加要有明確分工，唔好當成互改名。

**Q2：Collapsed tree 同 traversal 點揀？**  
A：論文發現 collapsed 通常更好，因為每條問題所需嘅抽象層級唔固定，攤平後可以一齊比。Traversal 好似層層鑽，但根層揀錯就下唔到正確枝。生產可先 collapsed；要控延遲就限制候選層，或先用較高層摘要粗召回再展開 leaf。兩條路都要寫死 token 預算同父子去重。

**Q3：費率改咗，更新 leaf 再 embedding 夠唔夠？**  
A：唔夠。祖先摘要可能仲寫住舊數字。高變更事實應查受控表；RAPTOR 只保留主題入口。改文後要令受影響祖先失效或納入定期重建，禁止用過期摘要作答。呢點同 GraphRAG 社區報告一樣。

**Q4：可唔可以同 parent-child 一齊用？**  
A：可以，但唔好當成同一機制。SOP 章節用 parent-child 保結構上下文；跨公告／覆盤嘅主題用 RAPTOR。同一請求唔好無節制把父段落、RAPTOR 祖先、leaf 三重塞入 context，否則貴、慢、仲容易 lost in the middle。

**Q5：可唔可以靠摘要 prompt 叫模型「唔好寫入機密 SOP」來當授權？**  
A：唔可以。無權 chunk 根本唔應進入同一棵樹嘅 cluster。ACL 喺建樹／檢索層執行，同多租戶篇、GraphRAG 篇同一原則。

## 小練習

**題：** 知識源包括：產品目錄（幣、鏈、費率，強一致）、季度事故覆盤（長文、跨產品主題）、有 ACL 嘅合規 SOP、幫助中心。同事建議全庫起一棵 RAPTOR 樹，取代 GraphRAG 同 hybrid；「VIP ETH 提現費」同「呢季事故主要根因類型」都先用 collapsed tree 搜摘要；費率變更只更新 leaf embedding；SOP 同公開材料一齊聚類，靠摘要 prompt 叫模型唔好洩漏。你會點評，同埋會點落地？

**參考答案：**  
1. **全庫一棵樹唔得。** SOP 同公開材料要分樹或分密級；cluster 一混，摘要就洩漏，prompt 擋唔住。  
2. **費率題唔應把內部節點當真值。** 樹最多做消歧或主題入口，最終數字查產品目錄／費率表。  
3. **只更新 leaf embedding 唔夠。** 祖先摘要可能保留舊費率；受影響節點要失效或定期重建。  
4. **「呢季根因類型」可以走 RAPTOR**（只限有權數據），設 token 預算同父子去重，citation 回 leaf；唔因此取消 hybrid／目錄路由。GraphRAG 亦唔係必須被取代——若題型係「實體之間點互相影響」而詞彙唔近，圖或受控關係可能仍然更對口。  
5. **落地：** 預設 collapsed；釘死 `raptor_index_version`；slice 分開事實／主題／faithfulness／ACL；引用唔到受控表或原文就 abstain。
