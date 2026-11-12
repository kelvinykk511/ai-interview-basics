# 線上檢索評估（無金標時點監察檢索品質）

> 2026-11-12 · AI warm-up · 工程面試向  
> 離線指標見 [`retrieval-eval-deep-dive.md`](./retrieval-eval-deep-dive.md)；RAG 改動上線節奏見 [`rag-shadow-eval.md`](./rag-shadow-eval.md)；生成層線上評估見 [`online-llm-eval-canary.md`](./online-llm-eval-canary.md)；實驗統計見 [`llm-ab-experiment-stats.md`](./llm-ab-experiment-stats.md)；**本篇講：線上點單獨監察「檢索有冇找對」，以及隱式反饋點解會呃你。**

## 人話

離線 golden 同 shadow 解決「改動可唔可以上」；上線之後 query 分佈會漂、金標會過期，你仲要一組 **常駐嘅檢索健康指標**。唔可以只睇 CSAT 或 LLM-as-judge：用戶唔滿意可能係生成胡謅、遲到、或者檢索根本冇把該段搵入 prompt。後端類比：搜尋／推薦要有獨立嘅相關性監控，而唔係只睇最終成交——否則鍋永遠分唔清。

面試官想聽：**log 咗啲咩、用邊啲代理指標、隱式反饋有咩偏差、點同生成層分鍋、以及點用線上流量反哺金標。**

## 面試短答

- **分層指標，檢索同生成分開報。** 系統層：空結果率、返回 k、p95 檢索延遲、錯誤率、版本對齊失敗。相關性代理：citation／來源展開點擊、query reformulation（連問、改寫）、「唔係呢個」類次輪、轉人工、已知 doc canary 嘅 Hit。生成層：faithfulness、拒答率、thumbs。唔好用一個「滿意度」代表檢索。
- **每條請求記檢索痕跡。** 原始 query、改寫後 query、filter／tenant、`embed_version`／index alias、每條候選嘅 `doc_id`／`chunk_id`／score／rank、有冇入到 prompt（被截斷要標）。冇呢啲 log，出咗事無法 replay，亦無法事後標註。
- **隱式反饋有偏差，唔係標籤。** 位置偏差（排第一更容易被點）、UI 偏差（有 citation 先能點）、呈現偏差（生成寫得好先有人願意展開來源）。點擊低可能係生成差而唔係檢索差。只能當 noisy proxy，要校正或只作告警，唔好直接當梯度。
- **Canary／探針 query。** 一組已知答案嘅「哨兵題」（費率標題、新上幣名、訂單號樣例、應拒答題）定期打線上檢索（或讀影子索引），看 Hit@k／空結果。成本低，對新鮮度同覆蓋最敏感。
- **Slice，唔只平均值。** exact-id vs 語義、新產品／新幣、語言、高峰活動。平均 Hit 穩定但新幣 slice 崩，就係漂移。
- **代理分數唔等於相關性。** 平均 cosine／rerank score 上升可能係分數校準漂咗，唔代表更準。睇排序同下游，唔好設「平均分要更高」當 KPI。
- **線上對比檢索器。** 改 retriever 唔好一上就全量。Shadow 先（見 rag-shadow-eval）；要比用戶體驗可用 **interleaving**（同一個結果表交錯 A/B 候選，減少位置偏差）或用戶級 A/B（sticky，預先功效，護欄，見 A/B 篇）。檢索 flag 同 prompt flag 分開。
- **歸因。** 金標或事後標註顯示「正確 chunk 已入 Top-k／已入 prompt」但答案錯 → 唔關 retriever，查 prompt／截斷／模型。正確 chunk 未召回 → 先修檢索。兩邊都要報，避免亂加 rerank 或亂換模。
- **用線上反哺金標。** 抽 reformulation、轉人工、低重疊、新產品、探針失敗嘅 query 做人工標註，定期滾動更新 held-out。注意授權同 PII。金標唔更新，離線分數會同線上脫節。
- **成本。** 唔好對全流量跑 LLM-judge。系統指標全量；judge／人工只對抽樣同差異集。Shadow 檢索要限流，唔拖主路徑。
- **CEX：** 上幣、改費率、公告撤銷呢類 slice 要有獨立告警；資金相關 exact-id（txid、訂單號）Hit 下降係事故級，唔可以等週週 CSAT。

## 常見追問（連答案）

**Q1：冇人工標註，可唔可以只用線上點擊優化檢索？**  
A：唔建議直接閉環。點擊受位置同生成品質污染，優化佢會強化「已經排前／寫得順」嘅文件，長尾同應拒答會更差。正確係：點擊同 reformulation 當告警同抽樣來源；真正改 retriever 仍要離線 golden（含 exact-id 切片）+ shadow，再小流量 canary。若做學習排序，要有偏差校正同凍結嘅評測集。

**Q2：CSAT 跌咗，點知係檢索還是生成？**  
A：對差評樣本回放 log：正確 chunk 有冇入候選、有冇入 prompt、citation 後端校驗過唔過、freshness lag。再睇同期檢索指標（空結果、探針 Hit、exact-id slice）有冇一起跌。若檢索指標穩、faithfulness／胡謅升，鍋在生成或 prompt。兩邊都跌先查版本切換、權限過濾同索引損壞。

**Q3：平均 cosine 升高但客訴增多，點解釋？**  
A：分數可因模型微版本、正規化、query 分佈、或 prefix 改變而整體平移，同相關性無單調關係。應看：探針 Hit、slice 空結果、reformulation 率、差異集人工／judge，而唔係平均分。呢個亦係 embedding refresh 要 pin 版本嘅理由。

**Q4：Interleaving 同 A/B 點揀？**  
A：Interleaving 同一用戶一次結果入面交錯兩套檢索，樣本效率高、位置偏差較易控，適合比「邊套候選更常被點／被引用」；但 UI 要能混排，同 citation 呈現要小心。用戶級 A/B 更乾淨地量 CSAT、轉人工等端到端，但要更大樣本同更長時間（見 A/B 統計篇）。實務：相關性對比用 interleave 或 shadow+judge；產品決策用用戶級 A/B + 護欄。

**Q5：線上評估會唔會洩漏或污染 golden？**  
A：探針同抽樣 log 含用戶 query，要按 PII 規則脫敏、權限同租戶唔能寫進共享評測集。用線上題補金標時，同訓練／合成 prompt 隔離（見 fine-tune 數據管線）。探針流量要標記，唔計入業務 KPI，亦唔好被 semantic cache 當成真實用戶答案去污染。

## 小練習

**題：** 客服 RAG 上線兩個月，離線 golden Hit@8 一直係 0.78，但最近一週「新上幣／活動規則」客訴升、轉人工升，整體 CSAT 只微跌所以沒人告警。同時有人發現平均檢索 cosine 仲略升。你會加邊啲線上監察？點歸因？會唔會因為 golden 冇跌而宣佈檢索沒問題？

**參考答案：**  
1. **唔可以因為整體 golden／CSAT 穩定就宣佈沒事。** 金標可能仍係舊 FAQ，平均會掩蓋新產品 slice。  
2. **即加嘅監察：** 按 slice（新幣、活動、exact-id、舊 FAQ）報空結果率、探針 Hit@k、reformulation、轉人工、citation 展開；新上線文檔嘅 canary query；檢索版本／alias 對齊錯誤；dirty／freshness lag。平均 cosine **唔做**成功 KPI。  
3. **歸因：** 抽該週客訴同轉人工單，回放是否召回咗正確公告／產品頁、有冇入 prompt。若未召回 → 覆蓋／embed 優先級／名稱別名／漂移（見 [embedding-refresh-drift](../05-mlops/embedding-refresh-drift.md)）；若已入 prompt 仍答錯 → 生成或未做 live verify。  
4. **金標滾動：** 把失敗嘅新幣／活動 query 標註後加入 held-out 同 CI 門檻（新產品 slice 不得低於 X）。之後先修覆蓋同 refresh，再用 shadow 驗證，唔好未分鍋就換模型或加 rerank。
