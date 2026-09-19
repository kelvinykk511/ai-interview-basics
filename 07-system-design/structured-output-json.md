# Structured Output／JSON Mode：讓模型產出可解析契約

## 人話

後端最怕 LLM「幾乎是 JSON」——少括號、多解釋、字段名飄。**Structured output**（JSON mode／JSON Schema／tool 參數約束）就是把「自由作文」收成「符合 schema 的物件」，好讓你的 Java 服務 `ObjectMapper` 一次讀進 DTO。面試要講清：**約束在哪一層做、失敗怎麼重試、跟 function calling 差在哪、以及為何仍要校驗**。

## 面試短答

- **目標**：穩定字段、類型、枚舉；方便下游規則引擎、DB、工作流，而不是給人讀散文。
- **常見做法**：  
  1. **Prompt 約定**（弱）：「只輸出 JSON」——便宜但不穩。  
  2. **JSON mode**（中）：模型被限在合法 JSON 語法，但不保證字段齊。  
  3. **JSON Schema / guided decoding**（強）：解碼時屏蔽不合規 token，貼合 schema。  
  4. **Tool／function calling**：把「要填的參數」當 tool schema——本質也是結構化輸出。
- **為何還要校驗**：供應商實現強度不一；枚舉可能出現近義詞；數字可能字串化；嵌套可選字段會缺。閘道應：**schema validate → 有限次修復重試 → 降級**（規則／人工／安全默認）。
- **與 RAG／agent 搭配**：先 retrieve，再「填表」；工具調用參數用 schema，避免模型自由發明參數名。
- 後端類比：像要求客戶端送 OpenAPI 合規 body——gateway 校驗，不靠「請對方禮貌一點」。

## 常見追問（含答案）

**Q：JSON mode 和「帶 JSON Schema 的 structured output」差在哪？**  
A：JSON mode 多半保證**語法**是 JSON；structured／schema 還約束**鍵、類型、required、enum**。只開 JSON mode，仍可能 `{"answer": "..."}` 缺你要的 `risk_level`。要契約穩定就上 schema 或 tool parameters。

**Q：模型死活不按 schema，你會怎麼設計重試？**  
A：1）把校驗錯誤（缺字段／類型錯）當**可重試信號**，同一 turn 帶錯誤信息再生成（限 1–2 次）。2）換更強約束（schema／tool）或更強模型。3）關鍵路徑不要無限重試——超時後走規則默認或人工隊列，並打 metric（schema_fail_rate）。

**Q：Structured output 會不會傷「推理品質」？**  
A：有時會：模型邊想邊被格式綁死，複雜推理題可能變差。實務：先自由鏈／草稿（可選），再**第二步只做抽取成 JSON**；或允許 `reasoning` 字段 + 最終 `result` 對象。延遲與成本會升，要產品取捨。

**Q：和 function calling 何時選哪個？**  
A：要**副作用／查系統**（下單、查庫、調 API）→ tool calling + 後端執行。只要**結構化數據回給你的服務**（分類、抽取、評分）→ structured output／JSON schema 即可，少一層「假 tool」。兩者都要 idempotency 與校驗。

## 小練習

**題：** 風控服務要 LLM 輸出 `{ "decision": "pass"|"review"|"block", "reasons": string[], "score": number }`。有人提議「prompt 寫清楚就好，省錢不開 schema」。你怎麼反駁，並給最小可靠設計？  

**參考答案：**  
- 反駁：省的是 token／功能費，賠的是解析失敗、錯誤枚舉、下游 NPE；風控路徑失敗成本遠高於 schema。  
- 最小設計：用 **JSON Schema 或 tool parameters** 鎖 enum／required；閘道 **校驗**；失敗則 **一次帶 error 的修復重試**；仍失敗 → 安全默認 `review` + 進人工隊列；監控 `parse_fail`、`schema_fail`、重試次數；日誌保留 raw 文本便於回放。不要把未校驗字串直接反序列進生產決策。
