# Agent Tool-Calling 可靠性

## 人話

Agent = LLM 決定「下一步呼叫哪個工具」（查單、轉帳預檢、搜知識庫），再根據結果繼續。對後端來說：LLM 像一個**不可信的編排器**——它會選錯 API、漏參數、同一副作用呼叫兩次。可靠性不靠「模型更聰明」，而靠**契約、冪等、超時、權限、狀態機**，跟你做微服務編排同一套肌肉。

## 面試短答

- **工具契約**：JSON Schema／嚴格參數；枚舉狀態；金額用字串或最小單位整數。模型輸出先 **validate**，不合格就重試或澄清，不要直接打下游。
- **副作用分級**：只讀（查餘額、查單）可多試；寫入（下單、鏈上 broadcast）必須 **idempotency key**、確認步、人工／風控閘。面試金句：never let the model be the source of truth for money movement。
- **迴圈控制**：max steps、總 token／latency budget、重複工具呼叫偵測（同一 args 連打 N 次就停）。避免「tool 地獄」燒錢又拖垮 P99。
- **錯誤處理**：工具超時／4xx／5xx 要結構化回餵模型（可重試？用戶可見訊息？）；寫入失敗要明確「未知 vs 確定失敗」（跟支付回調一樣）。
- **權限與範圍**：工具帶 user／tenant scope；LLM 不能「升權」。CEX：客服 agent 不該拿到熱錢包簽名工具。
- **可觀測**：trace 每個 thought→tool→result；記錄選錯工具、schema 失敗率，當 SLO，而不只看最終答對率。

## 常見追問（含答案）

**Q：Function calling 和自己 parse 自然語言比呢？**  
A：優先官方／結構化 function calling（或 JSON mode＋schema）。自解析脆弱、注入面大。面試強調「把工具邊界當 API gateway」。

**Q：模型說呼叫了 transfer，你怎麼保證沒雙花？**  
A：業務層冪等（clientToken／proposalId）；狀態機（created→submitted→confirmed）；工具層對同一 key 重入回同一結果。模型重試≠再執行一筆。

**Q：多工具並行可以嗎？**  
A：無依賴的只讀可以並行降延遲；有寫入或因果依賴必須串行。並行還要合併部分失敗的語義（跟 saga／補償同一思維）。

**Q：怎麼測 agent？**  
A：單工具契約單測；多步 trajectory 用固定 stub 工具；再加線上影子流量。指標：任務成功率、多餘工具呼叫、schema 違規、副作用違規、平均步數／成本。

## 小練習

**題：** 你做一個「幫用戶查充值為何未到帳」的 agent，工具有：`getDepositByTx`、`getNetworkConfirmations`、`searchSOP`、`createSupportTicket`。用戶連問兩次，第二次模型又想 `createSupportTicket`。你怎麼設計才不會開一堆重複工單？  

**參考答案：**  
1. **`createSupportTicket` 帶冪等鍵**（例如 `userId + txHash` 或首輪 conversationId）；重複呼叫回傳既有 ticketId。  
2. **狀態／記憶**：session 已有 open ticket 就禁用或降級該工具；system 提示「已有工單則改為查詢狀態」。  
3. **閘門**：寫入工具需顯式用戶確認或規則（確認數不足才允許建單）；觀測「重複建單率」當告警。讀工具可鬆、寫工具要緊。
