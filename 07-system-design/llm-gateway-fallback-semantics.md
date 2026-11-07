# AI 網關多模型 Fallback 嘅語義一致性

> 2026-11-07 · AI warm-up · 工程面試向  
> 限流／熔斷／重試見 [`../05-mlops/llm-gateway-resilience.md`](../05-mlops/llm-gateway-resilience.md)；**本篇只講：主模型掛咗切到備援時，行為／格式／工具／安全邊界點樣唔飄。**

## 人話

Gateway 寫咗 fallback（模型 A 502 → 切模型 B）好容易令人以為「服務仲喺度就等於體驗一樣」。現實係：**唔同模型對同一條 system prompt 嘅遵從度、拒答邊界、tool JSON、溫度手感都可以差好遠**。用戶可能突然由「嚴謹引用來源」變成「自信講大話」，或者 tool 參數由合法 schema 變成亂填。

後端類比：切支付通道——API 契約可以統一，但成功率、錯誤碼語義、對賬欄位唔會自動相同。**千萬唔好對前端假裝同一條 Happy Path**；要標 `degraded`／`model_id`，同埋用 capability matrix 決定「呢個請求可唔可以 silently 切」。

## 面試短答

- **Fallback ≠ 同一產品行為**：響應帶 `model_id`、`degraded=true`（或 `fallback_reason`），讓客戶端／客服／審計知道品質可能變。監控按 model 切開，唔好匯總成一條成功率就完。
- **語義漂移風險**（面試要主動數）：
  - Tool schema 遵從度（少欄、多欄、類型錯）
  - 拒答／安全邊界（A 拒講出金教學，B 可能鬆）
  - JSON／structured output 形狀
  - Citation／語氣／長度習慣
  - 默認 temperature／top_p 手感
  - Context window 唔同 → 截斷策略唔同 → 同一 prompt 實際入模內容唔同
- **緩解**：
  - **同一 prompt 模板版本 pin** 兩邊（registry 同一個 `prompt_version`）
  - **Constrained decoding／JSON schema** 強制輸出形狀（見 structured output 筆記）
  - **Capability matrix**：A 支援 tools `{X,Y,Z}`，B 只得 `{X}` → 需要 Y/Z 嘅請求**唔好静默切 B**，應 fail、降級到無工具路徑、或排隊等 A
  - **Golden regression 雙跑**：同一套評測喺 A 同 B 都過門檻先標「可 fallback」
  - 統一 normalize：`stop` sequences、`finish_reason` 映射、max_tokens 按窗口換算
- **路由策略**：優先「功能子集兼容」而唔係「只要有條路返 200」。Shadow 一段 fallback 流量做 pairwise／rubric，先逐步提高自動切換比例。
- **告警**：`fallback_ratio` 飆升時盯 `tool_error_rate`、`schema_fail_rate`、`refusal_delta`（相對主模型拒答率差）、用戶重試率。
- **幾時唔好 fallback**：
  - 請求中途已經產生**不可逆副作用**（tool 已寫入）
  - Streaming 已向用戶 commit 咗半段（唔好無聲換模型續寫）
  - 高風險內容（出金建議、合規結論）未驗證備援模型邊界
- **CEX 類比**：主清算通道掛 → 備援通道；對客戶仍係「提現 API」，但要暴露狀態、對賬欄位對齊、高風險額度可能降級人工——**唔好當 bit-identical**。

## 常見追問（連答案）

**Q1：備援模型 context window 細好多，點處理同一條長 prompt？**  
A：唔可以黙黙截尾當同一請求。要有**顯式 truncation policy**（保留 system + 最近 K 輪 + 強制摘要工具結果）、記錄 `context_truncated=true`，並喺 capability matrix 標「長對話不可 fallback」或走「先摘要再切」。否則 B 睇唔到關鍵工具結果，行為會飄到「幻覺補完」。

**Q2：Structured output 喺 fallback 下點保？**  
A：兩邊都走同一 JSON schema／constrained decoding；Gateway 做 **schema validate**，失敗則重試同模型一次或返回可重試錯誤，**唔好**把畸形 JSON 傳俾業務當成功。若 B 唔支援 JSON mode，呢類請求唔進入自動 fallback 池。評估用 `schema_fail_rate` by model。

**Q3：點解「主模型拒答、備援卻願答」好危險？**  
A：呢個係 **refusal boundary drift**。攻擊者或普通用戶一遇到 A 拒絕，Gateway 自動切 B 就可能變成**繞過安全策略嘅 oracle**。緩解：安全拒答（政策拒答）預設 **唔因 refusal 而 fallback**；只對基礎設施錯誤（5xx、超時、配額）fallback；備援模型要用同一套安全評測集，`refusal_delta` 超閾告警。

**Q4：Streaming 已經推咗 30 個 token，主模型斷線，可唔可以接備援續寫？**  
A：極危險——語氣、事實、已承諾內容可能對唔上，用戶會見到「人格分裂」。實務：已 commit 嘅 stream **唔静默換模續寫**；回可恢復錯誤／請用戶「重新生成」（新 `request_id`，可走備援）。首包前（未 flush 任何 delta）先允許換模。

## 小練習

**題：** 網關主模型 `gpt-strong` 支援 tools：`getBalance`、`getWithdrawStatus`、`quoteFee`；備援 `llama-fast` 只穩妥支援 `getBalance`（另外兩個 schema 遵從率 <70%）。產品話「要 99.9% 可用性，掛咗就切」。請設計 fallback 策略，避免語義漂移同埋唔安全嘅静默降級。

**參考答案：**  
1. **Capability matrix**：請求若 tool allowlist ⊆ `{getBalance}` → 可自動 fallback；若需要 `getWithdrawStatus`／`quoteFee` → **唔切 llama-fast**，改：(a) 快速失敗 + 重試 strong；(b) 降級路徑「只讀餘額 + 引導去官方狀態頁／人工」並標 `degraded`；或 (c) 隊列等待 strong 恢復（短 SLA）。  
2. **只對基礎設施錯誤 fallback**：502/503/429（配額）先切；**policy refusal 唔切**，避免拒答繞過。  
3. **契約**：統一 prompt_version、JSON schema validate、`finish_reason` 映射；響應帶 `model_id` + `degraded`。  
4. **雙跑回歸**：golden（含 tool 軌跡、拒答題、長 context）喺兩個模型分開出分；未達門檻嘅能力唔納入自動池。  
5. **觀測**：`fallback_ratio`、`tool_error_rate`/`schema_fail_rate` by model、`refusal_delta`、高風險 intent 嘅 fallback 次數（應≈0 或走人工）。  
6. **Streaming／副作用**：已 commit stream 或已執行寫入 tool → 禁止换模續跑；新請求新 idempotency key。
