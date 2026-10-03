# KV Cache 量化深挖（容量、品質與 Continuous Batching）

## 人話

權重佔顯存是「固定房租」；**KV cache** 是「每個在線請求按長度計費的流動佔用」。併發一高、context 一長，KV 往往比權重還吃卡——這時只做 GPTQ／AWQ 壓權重，像把倉庫貨架壓扁，但訪客行李還是塞滿大廳。**KV 量化**把每層的 K／V 從 FP16／BF16 收到 FP8／INT8／INT4，直接放大「同時能掛多少序列／多長 context」。後端類比：**連線 session 狀態從完整對象改成緊湊編碼**——容量明顯升，但解碼錯一點會讓長尾請求（工具 JSON、長對話）先崩。

> 權重路徑見 [`quantization-serving.md`](./quantization-serving.md)（那邊只提醒「權重≠KV」）；分頁與 prefix 見 [`paged-attention-prefix-kv.md`](./paged-attention-prefix-kv.md)；步級調度見 [`continuous-batching.md`](./continuous-batching.md)。本篇專講 **KV 這一刀**。

## 面試短答

- **為何 KV 在高併發／長 context 變主角**：權重 ≈ 常數；KV ≈ `num_layers × 2 × num_kv_heads × head_dim × seq_len × dtype_bytes × num_seqs`（GQA 已縮小 heads，但乘上 seq×seqs 仍暴漲）。70B 級 FP16 權重可放進多卡，但 32k×高併發時動態 KV 先把剩顯存吃光。
- **省什麼**：主要是**有效 `max_num_seqs`／可服務 context 長度**（容量）。對單請求延遲：decode 常是 memory-bound，位寬降有時略降 TPOT；但若 kernel 要反量化／精度路徑不成熟，延遲幾乎不變甚至變差——**別承諾「量化＝更快」**。
- **精度階梯（實務直覺）**：  
  - **FP8 KV**：硬體友好時吞吐／品質通常最穩，顯存約半（相對 FP16）；  
  - **INT8 KV**：容量再好一點，多數對話尚可，需評測；  
  - **INT4 KV**：容量最猛，長尾／長 context／結構化輸出風險最高，當「最後一刀」而非預設。
- **品質風險熱點**：長 context 遠端依賴、數字／代碼、**工具呼叫 JSON**、多輪累積誤差；prefill 寫入的量化噪聲會一路跟著 decode。評測要含長尾 goldenset，不是只看 MMLU。
- **與 PagedAttention**：量化改的是 **page／block 裡每個元素的位寬**；分頁仍解決碎片。同顯存預算下，更小 dtype → 同 page 可裝更多 token 或更多活序列。Prefix 共享的 block 也跟著省，但錯精度共享仍是串台／品質問題，隔離規則不變。
- **與 Continuous Batching**：CB 的硬上限常是「KV 槽位」。KV 量化抬高槽位 → `max_num_seqs`／batched tokens 有空間加大；若只加大併發不守 P99，prefill 插隊會把 TPOT 打炸——容量與尾延遲仍要分開調。
- **上線閘門**：相對 FP16／BF16 KV baseline：任務分數＋工具 JSON 正確率＋長 context 子集；壓測目標併發下 TTFT／TPOT／OOM；一鍵 rollback（配置關 KV quant，不必重訓權重）；shadow／金絲雀見 online eval。
- 後端類比：權重量化 ≈ 壓縮 jar；KV 量化 ≈ 壓縮每個連線的 session buffer——連線數上去才看得出收益。

## 常見追問（含答案）

**Q：已經做了權重 INT4，為什麼還要動 KV？什麼時候動了幾乎沒感覺？**  
A：權重 INT4 降低的是**固定**顯存；若線上是「短 prompt、低併發」，剩顯存本來夠 KV，再壓 KV **容量幾乎不漲、延遲也不一定好**。相反：長 RAG／多輪、`max_num_seqs` 被 KV OOM 卡住、或想同卡從 8 併發拉到 20——KV 量化才是主杠杆。面試一句：**權重管「模型塞不塞得下」；KV 管「同時服務多少長請求」**。

**Q：FP8／INT8／INT4 KV 怎麼選？會不會和權重 dtype 綁死？**  
A：不綁死——常見「權重 INT4／FP8 + KV FP8／INT8」混搭。選型：品質敏感／工具重 → 先 FP8 KV；容量仍不夠再試 INT8；INT4 僅在有強評測＋可回滾且業務能接受偶發格式錯時。看 serving 框架對該 GPU 的 **KV quant kernel 是否正式支援**，避免 silent fallback。

**Q：KV 量化後 Continuous Batching 的 `max_num_seqs` 可以直接加倍嗎？**  
A：顯存帳上「 theoretically 可以」，但 **prefill 算力、調度公平、prefix 命中** 沒加倍。實務：按 KV 佔用監控逐步上調；同時盯 P99 TTFT／TPOT；必要時開 chunked prefill、拆互動／批隊列。容量公式給上界，SLO 決定你敢用多少上界。

**Q：如何設計 rollback／eval gate，避免「顯存好看、客服 JSON 掛了」？**  
A：1）開關與權重 checkpoint 解耦，配置即可回 FP16／BF16 KV；2）gate：工具／結構化輸出正確率、長 context 子集、安全拒答，相對 baseline 不超約定掉點；3）金絲雀先吃內部流量；4）告警：KV OOM 下降卻伴隨 JSON parse 失敗率上升 → 自動停量或回滾。

## 小練習

**題：** 70B 客服＋工具呼叫，已上 AWQ-4bit 權重；A100 80GB 單卡 continuous batching。監控：權重顯存已降，但平均 context 8k、目標併發 16 時仍頻繁 KV OOM；P50 TPOT 尚可。有人提 INT4 KV，有人提只加卡。你怎麼決策？列出步驟、一個「先試 FP8／INT8」的理由、兩個否決 INT4 的信號。

**參考答案：**  
1. **步驟**：先算／量測當前 KV 佔用佔剩顯存比例；確認框架對 FP8／INT8／INT4 KV 的支援；用含 function-call JSON＋長多輪的 golden 對比 baseline；在目標 16 併發壓測 OOM 率與 P99；設配置 rollback。  
2. **先試 FP8／INT8**：工具 JSON 對量化噪聲敏感，INT4 常先在結構化輸出翻車；FP8／INT8 通常就能把 8k×16 的 KV 牆推開，風險較可控。  
3. **否決 INT4 的信號**：工具呼叫／JSON 正確率掉破門檻；或長 context 子集（遠端指代／跨段數字）明顯崩，即使 OOM 消失也不上。  
4. **加卡**：若品質門檻不允許再壓 KV、或單卡算力已被 prefill 打滿（延遲 SLO 已炸），水平擴容／拆池，而不是死磕 INT4。
