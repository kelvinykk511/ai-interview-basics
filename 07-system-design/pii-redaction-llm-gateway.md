# LLM 應用 PII 脫敏與數據外洩防護

## 人話

用戶資料（姓名、電話、身份證、錢包地址、餘額）一送去第三方 LLM，就出咗你控制範圍；log、trace、cache、eval 數據集都係漏點。防線要喺 gateway 做，唔靠 prompt 叫模型「唔好講」。

## 面試短答

分層：(1) 數據分級：邊級可以出外部模型、邊級只准自托管。(2) gateway 入口偵測+脫敏：規則（regex+校驗：卡號 Luhn、身份證校驗位、電話）+ NER（姓名、地址）；用可逆 token 化（`<PHONE_1>` ↔ vault 映射），回應返嚟再還原，映射 request-scoped 短 TTL。(3) provider 合約：zero data retention、唔用作訓練、數據區域。(4) log/trace/semantic cache/eval set 唔落明文，存脫敏版，或加密+存取審計。(5) 輸出側 DLP：檢查有冇洩露其他用戶資料、system prompt、密鑰。(6) 刪除權：向量庫、cache、fine-tune 數據都要可刪。

## 常見追問

- 脫敏會唔會整差答案？→ 會。用帶類型 placeholder（`<PHONE_HK_1>`）保留語義；同一實體同一 token 保持指代一致；用 eval 比較脫敏前後質量。
- RAG 檢索返嚟文檔有 PII？→ 索引時已脫敏或按 ACL 過濾（唔可以檢到其他人資料）；送 LLM 前再過 gateway；cache key 含 tenant/user。
- 錢包地址算唔算 PII？→ 交易所語境同 KYC 身份綁定後就係個人資料（可鏈上追蹤資產），按敏感數據處理，最少唔落 log 明文。
- regex 夠唔夠？→ 唔夠。結構化數據用規則（高精度）；姓名地址要 NER，會漏報 → 接受殘餘風險，所以 provider 合約+日誌策略係第二道防線；偵測要量 recall。
- debug 要睇原文？→ break-glass：加密存原文、短保留、要審批+審計、按需解密；平時只睇脫敏 trace。

## 小練習

**題：** 客服 bot 用外部 LLM，用戶話「我電話 9123 4567，提款去 0xAbC…123 未到」。設計流程，令 LLM 幫到手但見唔到真資料，同時工具查得到單。

**參考答案：** gateway 偵測 → 換成 `<PHONE_1>`、`<ADDR_1>`，映射存 request-scoped vault（TTL 例如 15 分鐘）。LLM 輸出 tool call `lookup_withdrawal(address="<ADDR_1>")` → 內網 tool 執行層先還原真地址查 DB，結果再脫敏（金額可留、txhash 視政策）先俾 LLM。LLM 回覆含 placeholder → 出口還原先俾用戶。log 只存脫敏版。工具層用登入 auth 嘅 user_id 限制只查自己單，唔信 LLM 傳嚟嘅 user id。

**相關：** [prompt injection/guardrails](./prompt-injection-guardrails.md)、[multi-tenant RAG](../04-prompt-rag/multi-tenant-rag.md)、[semantic-cache](../05-mlops/semantic-cache.md) 筆記。
