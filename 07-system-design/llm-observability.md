# LLM 可觀測性：Token、TTFT、成本與 Trace

## 人話

LLM 請求不像普通 HTTP「200 就完」——你要知道：**花了多少 token、錢、等多久出第一個字、卡在 retrieve 還是 decode、哪次幻覺、哪次 schema 失敗**。沒有這套觀測，限流與調優都在盲飛。面試要能講：**請求級 trace 串起 RAG／模型／工具、紅線指標、以及如何把成本歸到租戶／功能**。

## 面試短答

- **核心指標（請求級）**：  
  - **TTFT**（Time To First Token）：首字延遲，體感。  
  - **TPOT／ITL**（Time Per Output Token／Inter-Token Latency）：生成是否卡頓。  
  - **E2E latency**：整段結束。  
  - **Token**：`prompt_tokens`、`completion_tokens`、（若有）cached／reasoning tokens。  
  - **Cost**：按模型價目表折算 USD；歸屬到 `tenant_id`／`feature`／`model`。  
  - **品質代理**：schema_fail、refuse_rate、citation_miss、tool_error、用戶重新生成率。
- **Trace 結構**：一次用戶提問 = 一條 root span；子 span 含 `retrieve`、`rerank`、`llm.prefill`、`llm.decode`、`tool.*`、`validate`。用同一 `trace_id`／`request_id` 貫穿 Gateway 與業務日誌。
- **日誌**：存 prompt／回應要有**脫敏與抽樣**（PII、密鑰、完整對話很貴也有合規風險）。至少留 hash、長度、模型、usage、錯誤碼；出問題再按政策拉全量。
- **儀表盤分層**：金絲雀看 TTFT/P99 與 error；產品看成本／日活功能；品質看評測集回歸，不要只看「平均延遲」。
- 後端類比：TTFT ≈ TTFB；token ≈ 計費單位的「payload 大小」；trace ≈ OpenTelemetry 跨 DAO／RPC；成本歸屬 ≈ 雲賬單按 tag。

## 常見追問（含答案）

**Q：只監控平均 latency 為什麼不夠？**  
A：LLM 延遲長尾極重（隊列、長上下文、冷啟動）。要看 **P95/P99 TTFT 與 E2E**，並按 prompt 長度分桶。平均被短句「你好」拉低，掩蓋長文檔 RAG 的災情。

**Q：如何歸因「慢在哪」？**  
A：拆 span：向量檢索、重排、拼 prompt、供應商 queue、首 token、整段。若 TTFT 高但 retrieve 正常 → 多半隊列／prefill／供應商；若 retrieve 佔 80% → 索引／過濾／top-k。沒有 span 就只能猜。

**Q：Streaming 下 usage 與成本怎麼記？**  
A：多數在最終 `done` 才有準確 usage；取消時可能只有部分。Gateway 在結束（含 abort）時落一筆：已知 tokens + 狀態（completed/cancelled/error）+ 估算費用。對賬要用供應商账单對 `request_id`。

**Q：品質要怎麼「線上」觀測？離線 golden set 夠嗎？**  
A：離線回歸擋大退步；線上還要：用戶顯式反饋、重新生成、人工抽樣、關鍵路徑規則校驗（必引 citation、schema）。LLM-as-judge 可抽樣，但要控偏差與成本，不能替代業務指標。

## 小練習

**題：** 上線一週後成本翻倍，但 QPS 沒變。列你會查的四個觀測切片，以及各自可能根因。  

**參考答案：**  
1. **按模型拆成本**：是否默默切到更貴模型／更大 context 窗口。  
2. **按 feature／租戶**：某個新功能或大客戶把 TPM 打爆（長文貼上、多輪帶全文歷史）。  
3. **prompt vs completion tokens**：重試、修復 JSON、agent 多步 tool 導致 completion 暴漲；或 retrieve 塞了過多 chunk。  
4. **cached／失敗重試率**：cache 命中崩跌、429/5xx 重試、用戶狂點重新生成。對每片看伴隨的 TTFT 與 error，避免只盯總金額。
