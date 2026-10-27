# Token 限流與多租戶配額（TPM / RPM / 預扣再結算）

## 人話
LLM 唔係「一個請求一份成本」：同樣一次 call，100 token 同 30k token 嘅 GPU 時間同錢可以差幾百倍。所以只限 QPS（RPM）唔夠，要按 **token** 限（TPM），而且 output 長度事前唔知，要 **先預扣、完咗再按實際用量結算**——同交易所下單先凍結保證金、成交後多退少補一樣。

## 面試短答
- 兩個維度：**RPM**（防請求風暴）+ **TPM**（防大 prompt / 長輸出吃光容量），按 tenant / API key / 模型分桶。
- 請求入場：`預估 = input_tokens(用目標模型 tokenizer 算) + max_tokens`，喺 token bucket 預扣；唔夠 → 429 + `Retry-After`。
- 完成後：用 provider 返回嘅 `usage`（prompt/completion/cached tokens）**結算**，多扣嘅退返桶。
- 實作：Redis + Lua 原子扣減（sliding window 或 token bucket），key = `tenant:model:minute`；多 region 用本地配額份額 + 定期對帳，唔好每個請求跨 region 同步。
- 上游 provider 本身都有 TPM 上限 → 喺 gateway 做 **兩層**：租戶配額（公平）+ 全局上游配額（唔好打爆供應商）。

## 常見追問（連答案）
**Q1：點解唔可以只用 max_tokens 做扣減、唔結算？**
A：大部分請求實際輸出遠少於 max_tokens（例如 max 4096、平均 300），只預扣唔退會令實際利用率好低，租戶不停撞 429。反過來只按實際扣（事後扣）就擋唔住突發：1000 個請求同時入嚟，每個都「睇落仲有額」。所以係預扣 + 結算。

**Q2：Streaming 中途 client 斷線點計？**
A：上游可能仲喺度生成，成本照計。gateway 要偵測斷線後主動 cancel 上游（abort request），用已收到嘅 chunk 數 / 最後 usage 結算；有啲 provider 要 `stream_options.include_usage` 先喺最後一個 chunk 回 usage，斷咗攞唔到就用本地 tokenizer 數已輸出內容估算，並標記 `estimated=true` 等日終對帳。

**Q3：中文 token 成本有咩坑？**
A：唔同 tokenizer 對 CJK 效率差好遠，同一段中文喺 A 模型可能係 B 模型 1.5–2 倍 token。所以配額同計費要 **按模型 tokenizer 算**，唔可以用「字數÷4」呢類英文經驗公式；換模型時 TPM 配額同成本預算要重新估。

**Q4：Prompt caching 命中嘅 token 點計配額？**
A：分開記：`cached_input_tokens` 通常較平、甚至部分 provider 唔計入 TPM。內部配額可以按「加權成本單位」而唔係裸 token：`cost_units = input×w_in + cached×w_cache + output×w_out`，咁先反映真實成本（output token 一般貴幾倍）。

**Q5：點保證公平，唔畀一個大客戶餓死其他人？**
A：每租戶保底配額（guaranteed）+ 可借用共享池（burst），共享池用 weighted fair queuing；大客戶超出保底只可以用空閒容量，滿負荷時先被降級（排隊 / 轉平模型 / 429）。同時按優先級分隊列：互動式 > 批處理，batch job 走離峰或 Batch API。

## 小練習（題目 + 參考答案）
**題：** 你哋 gateway 對接一個上游 provider，上游限 2M TPM。A 租戶（VIP）保底 1M TPM，B、C 各保底 300k，其餘 400k 做共享池。某分鐘 B 突然跑批處理，每個請求 input 8k、max_tokens 4k，打出 429 雪崩；同時監控見 A 嘅實際 TPM 只用咗 200k，但 dashboard 顯示「上游已用 1.9M」。點解？點修？

**參考答案：**
1. 「上游已用 1.9M」好可能係 **預扣未結算** 嘅數：B 每請求預扣 12k（8k+4k），但實際輸出可能得幾百 token，大量預扣卡住容量；加上 streaming 斷線冇結算 / 結算漏咗（例如錯誤路徑冇退額）會令桶「漏水」。先對比 `預扣總額 vs usage 結算總額`，搵冇 release 嘅請求（timeout、5xx、client cancel 路徑）。
2. 修：所有終態（成功 / 失敗 / 取消 / 超時）都要 finally 結算，預扣記錄帶 TTL 兜底自動釋放；B 嘅 batch 請求調低 `max_tokens` 或按歷史 p95 輸出長度預扣（並喺超出時補扣）。
3. 隔離：batch 流量入獨立低優先隊列，只用 B 保底 + 共享池閒置部分，唔准借 A 嘅保底；A 未用嘅保底可以短時借出，但 A 一有需求要可搶回（preempt 排隊中嘅 batch，而唔係殺進行中嘅 stream）。
4. 429 雪崩：回 `Retry-After` + 客戶端 SDK 指數退避加 jitter，避免齊齊重試再撞牆。
