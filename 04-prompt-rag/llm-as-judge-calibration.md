# LLM-as-judge 偏差校準

## 人話（為什麼要懂）

Golden set 幾百題可以人工標，但每次改 prompt／模型／檢索，要對幾千條線上抽樣打分——人審唔夠快、成本高。**LLM-as-judge** 就係叫另一個（通常更強或專門）模型，按 rubric 幫你打分或揀邊個更好。後端類比：好似 canary 自動評分服務——快、可擴，但評分器本身都要對齊人工 QA，否則你以為變好，其實係裁判口味變咗。

問題係：裁判 LLM 唔係中立儀器。佢會偏偏好長文、偏好自己家族模型嘅輸出、第一個出現嘅答案、甚至分數一路往上漂。冇校準嘅 judge ≈ 冇校準嘅監控儀表：數字有，決策靠唔住。

工程上你要答得清：**點解用 judge、有邊啲 bias、點校準、線上／離線點用、同 prod prompt 點隔離**。

## 面試短答

- **點解用**：規模化離線／線上 eval；gold label 稀缺時當近似標註；成本遠低於全量人工。唔係取代人，係放大人嘅抽樣。
- **常見偏差**：
  - **Position bias**：pairwise 時偏好先出現嘅答案。
  - **Verbosity bias**：偏好更長、更「完整」嘅輸出，即使事實冇更好。
  - **Self-preference / same-family bias**：同廠／同系列模型互相打高分。
  - **Score inflation**：分數慢慢整體上移，門檻失效。
  - **Rubric drift**：裁判 prompt／範例改咗但冇版本化，歷史分數唔可比。
- **校準手段**：
  - Pointwise（單條打分）vs **pairwise**（二選一）——pairwise 往往更穩，但要 **swap 位置／隨機順序** 消 position bias。
  - Rubric 要有 **anchors**（幾分＝咩樣例）；可附 **reference answer**。
  - **Blind**：judge 模型家族盡量同 generator 分開；固定 temperature／decoding。
  - **Ensemble**：多個 judge 投票／平均。
  - **人工抽樣對齊**：算 Cohen’s κ 或 Spearman／Pearson 相關；κ 太低就唔好當唯一閘門。
- **線上 vs 離線**：離線固定題集做回歸閘；線上抽樣 + canary，唔好讓 judge 分數單獨決定全量放量。
- **隔離**：judge 用嘅 system／rubric prompt **唔要**隨便餵返去 prod 生成 prompt（污染生成行為、亦可能洩漏評分標準俾攻擊者／用戶）。
- **後端類比**：自動化 canary 評分器必須對齊人工 QA；指標要版本化、可回放。

## 常見追問（連答案）

**Q：Pointwise 同 pairwise 點揀？**  
A：要絕對分（過唔過門檻、趨勢）用 pointwise + 清晰 anchors；要比較兩個系統／兩個 prompt 用 pairwise 更可靠。生產常見：離線 A/B 用 pairwise（swap 位置），線上監控用 pointwise 加人工抽檢。兩者都要同人工標對齊。

**Q：點防 verbosity bias？**  
A：Rubric 明確寫「冗長唔加分／答非所問扣分」；要求先列必須覆蓋嘅要點再打分；可加長度正規化或「同樣正確下偏好簡潔」；對照人工標睇 judge 係咪系統性偏長文。必要時用規則先截斷再評（或分開評「正確性」同「可讀性」）。

**Q：Cohen’s κ 幾高先夠？**  
A：冇萬能數字。實務：關鍵安全／合規題以人為準，judge 只做粗篩；一般質量題希望同人有中等以上一致（例如 κ 大致 ≥ 0.4–0.6 再當輔助閘，視標註難度）。重點係**持續抽樣對齊**同睇 disagreement 類型，而唔係一次過報一個 κ 交差。

**Q：Judge 同 generator 同一個模型家族有咩問題？**  
A：Self-preference：家族內輸出風格／口癖互相抬分，A/B 會偏向「換湯唔換藥」嘅自家模型。做法：換另一廠商／系列做 judge、或多 judge ensemble；關鍵決策仍抽人審。

**Q：線上用 LLM-as-judge 有咩風險？**  
A：成本同延遲（每條多一次 LLM）；分數漂移導致誤報／漏報；judge prompt 洩漏或被注入影響評分；把 judge 當唯一放量條件會放大偏差。做法：抽樣而非全量、固定版本、與人工／行為指標（👎、重問率）交叉驗證、canary 多指標。

## 小練習

**題：** 客服 bot 用 GPT 類模型生成，你用同一系列更大模型做 pairwise judge 比較「舊 prompt vs 新 prompt」，新 prompt 勝率 68%。上線前你會做邊三步校準，先至信呢個數？

**參考答案：**  
1. **Swap 位置**：每個 pair 交換 A/B 順序各評一次，只保留兩邊一致嘅勝出（或平均），消除 position bias。  
2. **換家族／ensemble**：至少加一個非同系列 judge，或規則＋judge；睇勝率仲唔仲穩。  
3. **人工抽樣對齊**：抽 50–100 對人標，算一致率／κ；同時檢查 verbosity（新 prompt 係咪只係更長）。通過先至用勝率做放量依據，並保留線上 👎／重問率做 canary，唔單靠 judge。
