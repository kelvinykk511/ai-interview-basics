# Agent 長任務持久化執行 + 工具副作用冪等

## 人話
Agent 一個任務可能要 20 步、跑幾分鐘：LLM 諗 → call 工具 → 再諗。中途 pod 重啟、LLM 超時、用戶斷線都好常見。如果只放喺內存，就要從頭再嚟；更危險係「重跑」會令「轉帳 / 發郵件 / 落單」呢類有副作用嘅工具執行兩次。
做法同後端 Saga / workflow 引擎一樣：每一步落庫（checkpoint），有副作用嘅工具用冪等鍵，恢復時從最後完成嘅一步繼續。

## 面試短答
1. 狀態外置：每步把 messages、tool_call、tool_result、step 序號寫入 DB（或用 Temporal/LangGraph checkpointer），worker 無狀態。
2. LLM 輸出唔確定：恢復時**唔好重新問 LLM 已經答過嘅一步**，直接重放已記錄嘅 tool_call 決策，否則第二次可能揀另一個工具。
3. 工具分兩類：只讀（可隨便重試）vs 有副作用（要冪等鍵 = task_id + step_id，下游去重）。
4. 先記 intent（PENDING）→ 執行 → 記結果（DONE）；崩喺中間就用冪等鍵查下游狀態，而唔係盲目重做。
5. 高風險動作加 human-in-the-loop：狀態停喺 WAITING_APPROVAL，可以等幾日，批咗再繼續。
6. 防失控：最大步數、token/金額預算、整體 deadline、相同 tool+參數重複 N 次就熔斷。

## 常見追問（連答案）
- **Q：點解唔直接用 Kafka 重試整條任務？**  
  A：整條重跑 = 重新叫 LLM，結果唔同，而且之前步驟嘅副作用會再發生。要 step 級 checkpoint。
- **Q：工具係第三方 API，冇冪等鍵點算？**  
  A：自己包一層 outbox：先寫「準備呼叫 X，key=K」，呼叫後寫結果；恢復時見到 PENDING 就先用業務 ID 查對方有冇做過（例如按 client_order_id 查單），查唔到先決定重試定轉人工。查唔到又唔可以重做就 fail-closed 人工處理。
- **Q：LLM 中途 stream 斷咗，算唔算一步？**  
  A：未完整收到 tool_call / 回答就唔算完成，唔 checkpoint；恢復時重新請求呢一步（呢步冇副作用，安全）。
- **Q：用戶喺 agent 跑緊時改需求？**  
  A：當成新 user message 插入狀態，喺下一步邊界讀取；唔好中斷進行中嘅副作用工具。必要時提供 cancel → 補償（Saga compensation）。
- **Q：checkpoint 越嚟越大？**  
  A：大 tool 結果存 object storage 只留引用；配合上下文管理做摘要，但原始記錄保留用作審計。

## 小練習
**題：** 客服 agent 有「查訂單」「退款」兩個工具。退款 API 呼叫後 worker 立即 OOM 重啟，DB 只記到 PENDING。你點設計恢復？
**參考答案：** 退款呼叫帶冪等鍵 `task123-step4`（或 refund_request_id）。恢復時見 step4=PENDING，唔問 LLM，先用同一個 key 查退款服務：已成功 → 補寫 DONE + 結果，繼續 step5；明確未做 → 用同一 key 重試（下游去重保證最多一次）；狀態未知 / 對方唔支援查詢 → 停喺 NEEDS_REVIEW 轉人工，絕不盲目再退。全程記審計日誌。
