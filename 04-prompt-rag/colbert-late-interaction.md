# ColBERT／Late Interaction：token 級 MaxSim，同 late chunking 唔係同一樣嘢

> 2026-11-19 · AI warm-up · 工程面試向  
> Late chunking 見 [late-chunking.md](./late-chunking.md)；單向量 ANN 見 [vector-index-tradeoffs.md](./vector-index-tradeoffs.md)；hybrid + rerank 見 [hybrid-search-rerank.md](./hybrid-search-rerank.md)；生產 reranker 見 [reranker-production.md](./reranker-production.md)；embedding 選型見 [embedding-model-selection.md](./embedding-model-selection.md)。**本篇講：點解要保留 query／document 嘅 token 向量、MaxSim 點計、同「一個 chunk 一條向量」同 cross-encoder 點取捨，以及同 late chunking 呢個同樣有 late 字嘅方法有何分別。**

## 人話

常見 dense 檢索係 **bi-encoder**：整條 query 壓成一條向量、整個 chunk 壓成一條向量，然後打一個餘弦。快、易用 HNSW，但 pooling 之後「E-1204」同周圍敘述被平均埋，精準字面可能變弱。另一端係 **cross-encoder**：query 同 document 一齊入 transformer，token 之間充分交互，準但貴，通常只能精排 top-50／100。

**ColBERT**（Contextualized Late Interaction over BERT）行中間路：query 同 document **分開編碼**（document 仍然可以預先算），但**唔做一次 pooling**，而是保留每個 token 一條較短嘅向量。打分用 **MaxSim**：每個 query token 去 document 裡揾最似嘅嗰個 token，再把呢啲最大值加起來。交互發生喺編碼之後、打分之時，所以叫 **late interaction**。

後端類比：單向量像只存一份文件摘要再比相似；ColBERT 像保留倒排過嘅關鍵短語向量，查詢時逐個詞對最優對齊；cross-encoder 則像每次把查詢同全文一齊精讀。面試官想聽：同 late chunking 唔好混、內存同延遲代價、係做一階檢索定只做 rerank、可唔可以取代 BM25 同 cross-encoder。

## 面試短答

- **先分清三個「late／interaction」。** Late chunking：長窗先全文編碼，再按塊 **pooling 成一條向量**，之後仍係單向量 ANN；解決「塊孤立」。ColBERT late interaction：**唔 pooling 成一條**，用 token 矩陣打 MaxSim。Cross-encoder：編碼階段就交互，不能獨立預計算 document。三個問題唔同，唔互相取代。
- **MaxSim 係對齊，唔係兩條中心向量嘅餘弦。** 公式直覺：對每個 query token 取同 document token 嘅最大相似，再求和。所以 query 裡嘅錯誤碼、幣名、條款號，只要 document 有對應 token，就唔容易被長段平均沖淡。呢點解釋到點解它常在「精確標識 + 語義」混合集上，補到純單向量嘅短板。
- **代價主要喺空間同打分，唔只係模型名。** 單向量 768 維 float32 約 3KB／chunk；ColBERT 若 200 token × 128 維 float16，光向量就約 50KB，再加倒排／質心索引，膨脹可以係十倍以上。一階全庫 ColBERT 要專用索引（ColBERTv2／PLAID 類：質心候選＋壓縮殘差），**唔係**把 token 向量當普通點塞進同一個 HNSW 就當完成。
- **生產上常見係「二階 rerank」，而唔係一開始取代 hybrid。** 先 BM25 + 單向量（可輔 late chunking／CR）撈 100～200，再用 ColBERT MaxSim 精排，最後必要時先上更貴嘅 cross-encoder。語料大、QPS 高時，全庫一階 ColBERT 嘅內存同尾延遲要先算清楚，用自己嘅 slice 證明值得先上。
- **唔可以同單向量庫混用，亦唔可以當同一套 embed_version。** 幾何完全唔同；query 要用對應嘅 ColBERT query encoder，產出 token 矩陣而唔係一條 pooled 向量。Alias、cache key 要帶 `retriever=colbert` 同模型版本。
- **唔好宣告「有 ColBERT 就停 BM25／停 cross-encoder」。** BM25 對必召回嘅錯誤碼仍然穩、成本低；MaxSim 補 pooling，但唔等於倒排。Cross-encoder 仍然係更高上限嘅精排，只係更貴——要靠「候選數 vs 質量 vs p95」曲線決定層級，唔好按論文均值裁棧。
- **Query 仍然獨立短編碼。** 同 late chunking 一樣，唔好把 query 同某篇 SOP 綁住再 encode。Document 側先離線編碼；改模型或 token 長度通常要重建該庫。
- **評估要切開。** 錯誤碼／條款號、口語轉述、長短句、多語別名分開報。只報平均 nDCG 可能顯得「差唔多」，但標識類子集先見效。線上要盯 p95、每請求打分次數同內存。版本同 hybrid 一齊 shadow。

## 常見追問（連答案）

**Q1：Late chunking 同 ColBERT 都有 late，係咪同一類升級？**  
A：唔係。Late chunking 改「幾時 pooling」，結果仍係每塊一條向量，走普通 ANN。ColBERT 改「打分時有冇 token 交互」。上咗 late chunking 唔等於已經做 late interaction；反之，ColBERT 亦唔自動給你長文檔跨句 pooling 嘅全部好處（要睇你點切、點編碼 document）。

**Q2：用 ColBERT 係咪就可以唔使 cross-encoder？**  
A：唔一定。ColBERT 係獨編碼＋輕量交互，通常快過 CE、細過 CE。若切片上 CE 仲能穩定拉開質量，而 p95 仍在預算內，可以「hybrid 召回 → ColBERT 再縮候選 → CE 頂層」。若 CE 帶來嘅增益好少或打爆延遲，先停在 ColBERT。用曲線決定，唔用口號。

**Q3：可唔可以把每個 token 向量插入現役 HNSW，query 再做多次 ANN？**  
A：呢個唔係生產 ColBERT 檢索。token 數量係 chunk 數嘅幾十到上百倍，內存同召回控制都唔劃算，亦失去 MaxSim 嘅候選生成結構。要嘛用 PLAID／官方類索引做一階，要嘛只對 hybrid 候選做 MaxSim rerank（候選少時甚至可暴力對矩陣）。

**Q4：200 萬篇 SOP，直接上一階 ColBERT 可唔可以？**  
A：先算空間同 QPS。200 萬 chunk、每塊約 50KB 向量，光表示已約 100GB 量級，尚未計索引開銷同 replica。若內存／尾延遲唔達標，先做二階 rerank；只有 slice 證明一階明顯贏、並且容量規劃過關，先考慮全庫一階。

**Q5：換 ColBERT 模型或截斷長度，可唔可以同舊 token 向量共存一個 alias？**  
A：唔可以混。query／document 要同一套編碼器同維度。新版本獨立 collection，shadow 過閘再切 alias；cache key 帶模型同 `max_doc_tokens`。

## 小練習

**題：** 現役係 hybrid（單向量 + BM25）+ cross-encoder rerank top-50；單向量側已在評估 late chunking。主管聽完 ColBERT，提出四句判斷：（1）「ColBERT 就係 late chunking 嘅加強版，上咗就可以停 late chunking」；（2）「把 token 向量 upsert 進而家個 HNSW，同舊 chunk 向量共存，先灰 10% 流量」；（3）「有 MaxSim 就可以停 BM25，亦唔使再上 cross-encoder」；（4）「全庫一階換成 ColBERT，ingest 同查詢都只此一家」。請逐條對或錯，並寫你會落地嘅方案。

**參考答案：**  
1. **錯。** 兩者解決唔同問題。Late chunking 豐富單向量語境；ColBERT 係 token 級打分。可以疊加喺唔同層（例如 late 單向量做召回，ColBERT 做精排），唔係替換關係。  
2. **錯。** 唔可混幾何，亦唔應用「token 當普通點塞進舊 HNSW」冒充 ColBERT。獨立索引或只對候選做 MaxSim。  
3. **錯。** BM25 仍應用於標識必召回；CE 是否保留要睇切片增益同 p95，唔能預先裁掉。  
4. **未必，而且風險大。** 先用 ColBERT 做二階 rerank 對打現役 CE；一階全換要先過內存、延遲、slice（錯誤碼 vs 口語）。  
5. **落地：** 保留 hybrid 召回；ColBERT 先作 rerank 層；`retriever=colbert` 獨立版本同 shadow；用標識類同語義類 slice 加 p95／成本決定要唔要升一階、要唔要保留 CE。
