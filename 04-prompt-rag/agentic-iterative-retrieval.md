# Agentic／迭代檢索：單跳唔夠時點再搜、點停

> 2026-11-17 · AI warm-up · 工程面試向  
> 一次改寫／multi-query 見 [`query-rewriting.md`](./query-rewriting.md)；hybrid + rerank 見 [`hybrid-search-rerank.md`](./hybrid-search-rerank.md)；生成時 grounding 見 [`grounding-citations.md`](./grounding-citations.md)；複雜度同「去邊個庫」嘅路由見 [`query-corpus-routing.md`](./query-corpus-routing.md)；線上切片見 [`../07-system-design/online-retrieval-eval.md`](../07-system-design/online-retrieval-eval.md)。**本篇講：檢索後點判斷夠唔夠、幾時再跳、點停、點避免越搜越偏。**

## 人話

用戶問「上星期上架嗰隻新幣，提到佢官網寫嘅主網，VIP 提現費幾多？」單跳搜「新幣 VIP 提現費」，庫裡可能有上架公告、亦可能有費率表，但兩樣都未齊：你要先知道「嗰隻新幣」同「主網」係邊條鏈，先查到正確費率行。

後端類比：唔係 API 失敗就無限重試，而係先分類（參數缺咗 vs 上游空結果 vs 逾時），再決定補參數、換源定放棄，而且有 max attempts 同 trace。**迭代檢索**係同一種控制流：檢索 → 打分／對槽位 → 再搜或生成或拒答。論文裡嘅 Adaptive-RAG、CRAG、Self-RAG 都係呢個圈嘅唔同實現，面試重點係工程約束，而唔係背模型名。

面試官想聽：同「一次改寫」或「top-50 再 rerank」有咩分別、點控制跳數同 TTFT、點阻止第一跳認錯幣之後一路錯落去、內部費率可唔可以隨便用公網 fallback。

## 面試短答

- **先釐清失敗模式。** 寬召回 + rerank 解決「文件在候選裡但排得後」；CR／late chunking 解決「塊嘅向量太孤立」。迭代解決嘅係「今次 query 資訊唔夠，要靠中間結果先寫得出下一條 query」，或者「命中唔覆蓋問題槽位」。三種唔互斥，亦唔互相取代。
- **預設唔開迭代。** 條款號、錯誤碼、單一產品費率，一次 hybrid + rerank 通常夠。無條件每請求 3 跳會打爆 TTFT 同成本，平均指標仲可能看起來「差唔多」——因為大多數簡單題根本唔需要。
- **生產控制流不必訓練 Self-RAG。** 常見三步：（1）複雜度／意圖路由，決定 0 跳、1 跳定允許多跳；（2）檢索後打分——規則槽位（幣、鏈、VIP、生效日）+ 必要時 cross-encoder 或細 LLM grader；（3）行動：夠就生成；部分夠就針對缺失槽位改寫再搜，並合併原結果；不夠就換受控源、澄清或拒答。
- **可以點名論文，但要講清楚落地差異。** Adaptive-RAG：先判斷 query 複雜度，簡單題唔開多跳。CRAG：把檢索結果分成 Correct／Ambiguous／Incorrect，錯就改源或做 refinement。Self-RAG：特製模型用 reflection tokens 決定要唔要檢索、證據支唔支持——base 模型唔會「自然具備」。CEX 內部費率／資產能否找回，**唔好把公網搜尋當成 Incorrect 嘅默認 fallback**（過時、釣魚、口徑唔一致）。
- **預算寫死。** 例如 `max_hops=2`、額外檢索牆鐘上限、grader 次數上限。能並行嘅子 query 就並行，唔好串行盲搜。預算用盡時：有部分受控證據就「按引用答 + 標未覆蓋嘅槽位」；全無證據就 abstain，禁止用模型參數記憶填費率。
- **合併要保留第一跳已過閘嘅文件。** 第二跳係補洞，唔係整庫重嚟。去重後 RRF 或 rerank。若 grader 判定第一跳完全跑偏（錯產品），先作廢再重搜，唔好把錯塊混入生成。
- **防 error cascade。** 第一跳把「該幣」錯誤坐實成某隻 ticker，之後每跳都會圍繞錯實體搜。產品名／鏈名／條款號要對受控詞表；grader 查槽位覆蓋，而唔只睇語義相似。相同或高度相似嘅 query 唔重複搜。
- **可觀測同評估。** 每 hop 記 query、collection、doc_ids、grader 分、行動。離線／線上 slice 要有 multi-hop、新幣別名、槽位缺失集；只報平均 Hit@k 會掩蓋呢類題全軍覆沒。生成階段仍走 citation 校驗（見 grounding 篇）。
- **同 query 改寫嘅分工。** 改寫係檢索前一次（condense、multi-query、保留原句）。Agentic 係檢索後睇結果再決定下一步。可以疊加：先 condense，單跳搜，不夠再定向補搜。版本釘 `retriever_policy_version`（含 max_hops、grader、停止規則），同 prompt／embed 版本分開發布。

## 常見追問（連答案）

**Q1：把 top-50 一次撈完再 rerank，係咪就唔使多跳？**  
A：唔係。寬召回補嘅係排名，補唔到「中間實體仲未出現」。冇主網鏈名，費率表裡正確嗰行可能根本唔會被呢條 query 叫到候選。先做好 hybrid + rerank，再用 multi-hop slice 衡量仲有冇缺口，有先加迭代。

**Q2：每條請求都用大模型做 grader，延遲點辦？**  
A：唔好。第一層用規則／槽位是否出現（query 或標題路徑有冇幣、鏈、VIP）同可選 cross-encoder 閾值。只有灰區先上細 LLM。Grader 結果可按 `query + doc_ids` cache。簡單路由類直接跳過 grader，護住 TTFT。

**Q3：第二跳應該整庫重搜，定只補缺失槽位？**  
A：針對缺失槽位寫更具體嘅 query，保留已過閘文件再合併。整庫重搜更貴，亦可能沖掉已有證據。完全跑偏先作廢重來，呢個分支要喺 policy 裡寫明。

**Q4：點避免無限 loop 或愈搜愈幻覺？**  
A：硬上限、重複 query 短路、詞表校驗、生成唔准把「搜唔到」當成可以靠記憶補數。監控 `hop=max 且仍未過閘` 嘅比率；呢個升咗通常係路由、詞表或新產品冷啟動問題，而唔係再加大 `max_hops` 就解決。

**Q5：係咪一定要訓練 Self-RAG 先叫 agentic RAG？**  
A：唔需要。Self-RAG 係一種訓練方法。線上用分類器 + 檢索 + 打分 + 有限重試已經係同一類控制流。有足量軌跡、想減少一次外部 grader 呼叫時，先考慮特製模型。

## 小練習

**題：** 客服問「上星期上架嗰隻新幣，提到佢官網寫嘅主網，VIP 提現費幾多？」現役係單跳 hybrid + rerank。同事三個方案：（a）每條 query 固定 3 跳，每跳叫大模型改寫再搜，全部文件一齊生成；（b）先判斷是否多約束，單跳先搜；缺「鏈」或「VIP 費率」槽位就先向上架公告解析主網，再針對該鏈 + VIP 費率搜第二次，最多 2 跳，公網搜尋作 fallback；（c）認為已上 Contextual Retrieval 同 late chunking，唔再需要迭代。你會點評，同埋實際上線會點做？

**參考答案：**  
1. **(a) 唔得。** 無條件 3 跳打爆 TTFT／成本；無 grader 會把跑偏文件越堆越多；新幣別名可能第一跳就錯，之後每跳強化錯誤實體。  
2. **(b) 控制流對，fallback 要改。** 複雜度路由、槽位檢查、定向第二跳、`max_hops=2` 係對嘅。內部費率唔能用公網兜底。主網應從受控上架公告或產品目錄解析，並對詞表；對唔上就澄清或拒答，唔好猜。  
3. **(c) 只對咗一半。** CR／late 改善「費率短句缺幣名」呢類單跳召回，但補唔到「先解析中間實體再查表」。應疊加，唔取代。  
4. **落地：** 默認單跳；槽位缺失先定向 hop；每 hop 寫入 trace；slice eval 要有多跳／新幣上架集；生成只引用內部費率表，引用唔到就 abstain。
