# Agent 軌跡評估

## 人話

Agent 唔係答一句，而係一連串「諗→叫工具→睇結果→再叫」。淨睇最後答案唔夠：可能撞啱、可能行 20 步燒晒錢、可能中途做咗危險操作（例如退兩次款）。好似 code review 唔止睇 test 過唔過，仲要睇過程有冇做壞嘢。

**評三層：**

1. **Outcome**：最終狀態啱唔啱。最好用 state-based check —— sandbox 跑完查 DB／環境狀態（訂單取消咗？只退一次款？），唔好比對文字。
2. **Process（軌跡）**：工具揀啱未、參數啱未、有冇多餘／重複調用、有冇違規步驟（未確認就做寫操作）、順序約束（先查後改）。可以同 reference trajectory 計 tool call precision／recall，但唔好要求一模一樣。
3. **Cost／Safety**：步數、token、延遲、成本、policy 違規率。

**可靠性**：agent 有隨機性，每題要跑 k 次。`pass@k` = k 次至少一次成功（適合可重試場景）；`pass^k` = k 次全部成功（衡量穩定性，生產更重要）。例：單次成功率 0.8，`pass^3 = 0.8³ = 0.512`。

**基建**：工具 mock／record-replay 令 eval 可重現；sandbox 每題 reset；LLM judge 睇軌跡要用 rubric（逐條 yes／no），唔好打 1–10 分；線上 trace 失敗案例回流做 regression set。

## 面試短答

Agent eval 分 outcome、process、cost 三層：outcome 喺 sandbox 用狀態斷言判；process 檢查工具選擇、參數、順序約束同禁止動作；cost 睇步數、token、延遲。每題跑多次報 pass^k；工具用 record-replay 保證可重現；線上失敗 trace 回流 regression set，每次改 prompt、模型或工具 schema 都要跑。

## 常見追問（含答案）

**Q1：點解唔直接比對 reference trajectory？**  
A：合法路徑唔止一條（先查訂單定先查用戶都得），exact match 會誤殺；用「必須包含」「禁止出現」「順序約束」assertion 更穩，或 LLM judge + rubric。

**Q2：Mock 工具定真環境？**  
A：replay 平、快、可重現，但 agent 走咗錄音冇覆蓋嘅路徑就冇回應 —— 要標 off-script 或者用有狀態嘅假服務（in-memory DB）。staging 真環境捉到整合問題但慢同 flaky。分層：PR 跑 replay，nightly 跑 staging。

**Q3：LLM judge 評軌跡信得過？**  
A：要校準：抽約 100 條人工標，計同人嘅一致度（如 Cohen's kappa）；rubric 拆成二元問題；長軌跡要截或摘要；judge 用唔同模型避免自我偏好；judge 版本要鎖定。

**Q4：多輪對話點測？**  
A：用 LLM 扮用戶（user simulator，有 persona 同目標），但會漂移或太合作；要固定 seed、限制輪數、人工抽查 simulator 行為。

**Q5：線上點監控？**  
A：每步一個 span（工具名、參數、延遲、錯誤）；指標有任務完成率、平均步數、工具錯誤率、loop 偵測（同工具同參數連續重複）、成本 p95；異常 trace 抽樣入 regression set。

## 小練習

**題：** 退款 agent 有 `get_order`、`check_policy`、`issue_refund` 三個工具。新 prompt 版本 outcome 準確率由 85% 升到 90%，但用戶投訴「退咗兩次款」。你嘅 eval 漏咗咩？點補？

**參考答案：**

漏咗 (1) **狀態斷言** —— 只判最終回覆講「已退款」，冇查 `refunds.count == 1`；(2) **process 約束** —— `issue_refund` 最多一次、必須喺 `check_policy` 之後；(3) **故障場景** —— 每題只跑一次、冇注入 timeout，agent 遇到工具超時就重發。

補法：sandbox 斷言退款紀錄數；加「禁止重複寫操作」assertion；故障注入（tool timeout／5xx）測重試；工具層加 idempotency key（同交易所出金冪等同一道理）；每題跑 5 次報 pass^5；上線加寫操作前 policy guardrail。
