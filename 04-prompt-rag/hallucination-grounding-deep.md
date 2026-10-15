# Hallucination Grounding 深化：Claim、Faithfulness、Abstention

## 人話

[`grounding-citations.md`](./grounding-citations.md) 講清「要引用、後端核對 citation、低分拒答」。上線後仲會遇到：**有引用但句子過度推論**、**檢索錯卻引用得很認真**、**該說不知道卻硬答**。本篇把 grounding 拆成可測嘅工程環節——像後端唔止要 HTTP 200，仲要**業務不變式＋對帳**。後端類比：citation ≈ `sourceId`；claim decomposition ≈ 把響應拆成可對帳字段；NLI／faithfulness ≈ 用第二條校驗服務對每條主張做「證據是否蘊含」；abstention ≈ 熔斷／降級而唔係返回假成功。

## 面試短答

- **錯誤歸因（先分鍋）**：  
  - **Retrieval error**：根本冇檢到對文／檢到過期／錯租戶 → 引用再靚都係錯。  
  - **Attribution／faithfulness error**：文有到，但答案誇大、合併錯段、編造文中沒有的數字。  
  - **Unsupported speculation**：模型用世界知識「補完」——產品上常當幻覺。  
  面試一定要分開講，否則 eval 同修復會改錯層（狂改 prompt 救唔到 retrieval miss）。
- **Claim decomposition**：長答拆成原子主張（誰、何時、多少、狀態），每條獨立綁 evidence span；關鍵槽位走結構化抽取＋規則／DB 校驗，散文只潤色。同「先填槽再生成」一致。
- **Faithfulness／NLI 式檢查**：對每條 claim 問「evidence 是否支持？」——可用小型 NLI／專用 entailment 模型、或 LLM-as-judge（要注意 [`llm-as-judge-calibration.md`](./llm-as-judge-calibration.md) 偏差）。不支持 → 刪句／改寫／整答拒答。**引用存在 ≠ 句子被蘊含**。
- **Abstention 校準**：目標唔係「永遠答」，而係 **risk-cover tradeoff**——高風險域（資金、合規、醫療）寧可多拒。校準訊號：retrieval score、rerank 分、claim-NLI 分、自我一致性（多樣答案分歧）、工具／DB 校驗失敗。閾值用驗證集調到目標 false-answer rate。
- **Attribution vs retrieval（debug 流程）**：golden 標「應檢中 doc」→ 若 miss 就修 chunk／召回／權限；若 hit 但仍錯答 → 查 prompt、過長 context 淹沒、或缺 claim 級校驗。線上用「有 citation 但人工標幻覺」專項看板。
- **Online 幻覺訊號（唔靠等人投訴）**：引用缺失率、citation id 不在 context 率、NLI 不支持率、高風險槽位校驗失敗率、用户「舉報不實」標籤、同問多答分歧率；同 A/B 或 canary 綁。
- **同基本 citation 嘅關係**：基本篇 = 強制指針＋後端存在性校驗；本篇 = **語義蘊含＋拒答校準＋分層歸因**。兩者要疊加，唔好只做其中一個。
- 後端類比：retrieval miss ≈ 查錯庫／錯分片；faithfulness fail ≈ 讀到 row 但組裝 DTO 時加料；abstention ≈ 校驗失敗返回 422 而唔係 200 + 垃圾 body。

## 常見追問（含答案）

**Q：已經強制 `[doc:id]` 且後端校驗 id 存在，點解仲會幻覺？**  
A：因為校驗只證明「引用指針合法」，唔證明「句子被該段文字支持」。典型：引用一段講「1–3 個區塊」，答案寫成「保證 24 小時」——id 過關，語義唔過關。要加 claim 級 faithfulness／關鍵槽位結構化校驗。

**Q：Faithfulness 用 LLM-as-judge 會唔會自己都幻覺？點減輕？**  
A：會。減輕：① 判據收窄成「support / refute / NEI」，唔好打開放分；② evidence 只塞該 claim 對應 span，減少噪音；③ pairwise／anchors 同人工 κ 校準（見 judge 篇）；④ 高風險欄位改規則／DB，唔經 judge；⑤ judge 同 prod 模型／prompt 隔離，防 self-preference。

**Q：點分辨「檢索錯」同「生成不忠實」？面試點講排查？**  
A：離線：標注應召回 doc／chunk → recall@k；再喺「已命中」子集量 answer faithfulness。線上：若 top 證據同問題無關（人工或分類器）→ retrieval；若證據明顯含答案但輸出偏離 → faithfulness。修復：前者改 index／query rewrite／權限過濾；後者改生成約束、claim 校驗、減 context 噪音。

**Q：Abstention 會唔會令產品「成日唔答」？點校準？**  
A：用驗證集畫 **risk–coverage 曲線**：x = 答出率，y = 答錯率（或嚴重幻覺率）。揀業務可接受嘅錯答上限，倒推閾值（檢索分、NLI、一致性）。分域閾值：資金／法務緊、FAQ 鬆。另設降級路徑（轉人工、只展示來源片段讓用户自己睇），唔係裸拒。

**Q：Claim decomposition 會唔會太重、延遲爆？**  
A：唔使每條閒聊都全做。實務：① 高風險意圖先觸發；② 只對金額、地址、時限、狀態等槽位做；③ NLI 用細模型批量；④ 可異步抽樣審計為主、同步只拦硬校驗失敗。同後端「熱路徑校驗 vs 異步對帳」一樣。

## 小練習

**題：** RAG 客服上線後，citation 覆蓋率 95%，但人工抽樣仍有 8%「嚴重不實」（多為到帳時間、手續費）。檢索日誌顯示多數 case top-3 有正確 SOP。設計一條「同步攔截＋離線歸因」方案，並列三個 dashboard 指標。

**參考答案：**  
1. **同步**：對到帳時間／手續費等槽位做結構化抽取（來自 cited span）＋規則校驗；答案中對應句子必須同槽位一致，否則改寫或拒答。可選：對其餘 claim 跑輕量 support/NEI，NEI 超閾則刪句。  
2. **離線歸因**：嚴重不實 case 標 `retrieval_miss | faithfulness_fail | stale_doc | other`；本場景預期以 faithfulness／stale 為主——若 hit 仍錯就加強槽位校驗；若 doc 過期就修 freshness（見 rag-freshness）。  
3. **Dashboard**：① citation 存在但 NLI/槽位失敗率；② 嚴重不實率（人工／舉報）按歸因分類；③ abstention 率同用户滿意度（防拒太多）。  
4. **一句**：指针校驗之上加語義／槽位對帳，先分鍋再改層，用 risk–coverage 校準拒答。
