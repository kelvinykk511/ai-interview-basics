# Tool-use 延遲與並行工具

## 人話

Agent 慢，常常不是「模型不夠聰明」，而是**多輪：想一下 → 打工具 → 再想 → 再打工具**，每一段都在累加。後端直覺：像一次請求 fan-out 到餘額／風控／KYC 微服務——只讀且互不依賴可以並行；轉帳必須等檢查完再串行。可靠性（契約、冪等、權限）見 [`agent-tool-calling.md`](./agent-tool-calling.md)；**本篇只講延遲、並行與 E2E 預算**。

## 面試短答

- **E2E 延遲拆帳**：`總牆鐘 ≈ Σ(LLM think／選工具) + Σ(tool RTT) + Σ(LLM 消化工具結果) + 網絡開銷`。多輪 agent 會把「LLM 段」放大 N 次；P99 常死在某一輪慢工具或某次長 think，不是平均 token 速度。
- **串行 vs 並行**：只有**獨立、只讀、無因果依賴**的工具才該並行（例：同時 `getDeposit` + `getConfirmations`）。寫入、或「B 的參數來自 A 結果」必須串行。並行還要定義**部分失敗**語義（誰失敗整輪降級？）。
- **Speculative parallel**：在模型還沒說完「要呼叫誰」之前，先開火高機率工具（或常見 SOP 組合）。換延遲、付代價：多餘呼叫、成本、可能打到限流；適合熱路徑且工具便宜／可快取。
- **超時與取消**：每工具有 timeout；整輪有 deadline；用戶中止 stream 要 **cancel in-flight tools**（別讓孤兒轉帳／建單繼續跑）。超時可回**部分結果**讓模型降級回答，不要整輪 5xx 死掉。
- **預算旋鈕**：`max wall-clock`、`max tool rounds`、`max parallel fan-out`。負載高時 degrade：少工具、走快取 SOP、強制「只查不寫」、或跳過 speculative。
- **CEX 類比**：查充值狀態 → 並行拉鏈上確認＋內部充值單 OK；`transfer`／`broadcast` 必須在風控／餘額檢查之後串行。別把 agent 編成「無序 saga」。
- **可觀測**：per-tool latency histogram、parallel span waterfall（哪條腿拖 P99）、slow-tool SLO、取消率／超時率／多餘 speculative 呼叫率。

## 常見追問（含答案）

**Q：E2E 延遲公式裡，哪一段最常被面試官追？**  
A：多輪下的 **Σ LLM after-tool** 與 **串行 tool RTT**。很多人只優化單次 TTFT，忽略「工具回來後模型又要想一轮」；或把本可並行的只讀查詢串成瀑布。答題要會畫 waterfall：每一輪 LLM／每一條 tool span。

**Q：什麼時候絕對不要並行？**  
A：有寫入副作用、或後一步依賴前一步輸出（查完 ticketId 才能 update）、或需要同一把分布式鎖／同一 nonce。只讀但會互相「讀己之寫」的一致性場景也要小心（先寫後讀若並行會讀到舊值）。

**Q：Speculative tool 和「模型先選再打」怎麼取捨？**  
A：熱路徑、工具便宜／可快取、候選集合小 → speculative 省一輪 LLM wait；工具貴、有配額、或寫入 → 等模型。面試金句：speculative 是用**成本與誤打**換 **TTFT／牆鐘**，要有浪費率儀表與熔斷。

**Q：用戶把 SSE 關掉了，後端還在跑三個工具，怎麼辦？**  
A：傳播 cancellation（context cancel／abort signal）到 tool client；寫入工具若已發出，走冪等查詢確認終態（見 [agent-tool-calling](./agent-tool-calling.md)），不要 silently 再試一筆。觀測「orphan tool completion」告警。

**Q：線上負載飆了，怎麼 degrade tool-use？**  
A：降 `max parallel fan-out`、禁 speculative、縮 `max tool rounds`、熱點問題走快取 FAQ／SOP 而不打實時 API、寫入工具改「排隊人工」。目標是保牆鐘 SLO，而不是保「最完整工具軌跡」。

## 小練習

**題：** 客服 agent 平均 3 輪工具。工具：`getDepositByTx`（p50 80ms / p99 1.2s）、`getNetworkConfirmations`（p50 200ms / p99 2s）、`searchSOP`（p50 50ms）、`createSupportTicket`（寫入）。現況全串行，P99 牆鐘 ~8s。產品要求互動 P99 ≤ 4s，且不能雙開工單。請給一個並行＋預算方案（含 speculative 可選），並說明觀測指標。

**參考答案：**  
1. **並行只讀**：同一輪內 `getDepositByTx` ∥ `getNetworkConfirmations` ∥ `searchSOP`（三者獨立）→ 該輪 tool 牆鐘≈ max(p99) 而非 sum。  
2. **寫入串行＋冪等**：僅在「確認數不足／規則命中」且用戶確認後才 `createSupportTicket`；鍵用 `userId+txHash`，防雙開（可靠性細節鏈到 [agent-tool-calling](./agent-tool-calling.md)）。  
3. **預算**：`max tool rounds=3`、整輪 deadline 3.5s、單工具 timeout（deposit 1.5s / confirmations 2s）；超時帶部分結果讓模型答「鏈上確認查詢逾時，已見內部單狀態…」。  
4. **可選 speculative**：用戶訊息含 tx hash 時，在首輪 LLM 未結束前預熱三個只讀工具；浪費率 > 閾值或下游限流則關。  
5. **觀測**：parallel waterfall、每工具 histogram、牆鐘 vs Σ串行基線、取消率、speculative 浪費率、重複建單率。
