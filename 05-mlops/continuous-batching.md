# Continuous Batching 深挖（調度、迭代與尾延遲）

## 人話

靜態 batch 像「等人湊齊一桌才開灶」：短請求被長請求拖死，GPU 常空轉等最慢那位。**Continuous batching**（vLLM / TGI 系）改成「每一個 decode step 重新組桌」——做完的請求立刻離席、新請求插進來，GPU 盡量每步都滿載。後端類比：**NIO 事件循環 + 動態 worker 池**，不是一次把整批 HTTP 請求堵到全部結束。

> 和 [`paged-attention-prefix-kv.md`](./paged-attention-prefix-kv.md) 的關係：分頁 KV 讓「進進出出」不用搬整段連續顯存；本篇講**調度策略本身**——何時接單、何時優先 decode、為何 P99 仍會炸。

## 面試短答

- **靜態 vs 連續**：靜態 = 一批一起 prefill／一起 decode 到全員結束；連續 = **iteration-level scheduling**，每步 batch 成員可變。
- **兩階段成本**：新請求要先 **prefill**（吃整段 prompt、填 KV，算力尖峰）；已在跑的做 **decode**（每步一個 token，偏記憶體帶寬）。調度器要在「插 prefill」與「繼續 decode」之間取捨。
- **為何吞吐升**：減少 padding 與空等；短請求不再等長請求整段結束；顯存以「活著的序列」為單位佔用。
- **尾延遲來源**：排隊等進 batch；大 prefill 搶算力拖慢同行 decode；超長生成佔用 KV 槽位 → 有效併發下降；調度偏向吞吐時 P99 TTFT／TPOT 變差。
- **常見旋鈕**：`max_num_seqs`（併發上限）、`max_num_batched_tokens`（每步 token 預算）、prefill chunking／分塊 prefill、優先級（互動 vs 批處理隊列）。
- 後端類比：prefill ≈ 冷啟動載入大 payload；decode ≈ 長連線心跳；`max_num_batched_tokens` ≈ 單次 event-loop 預算，避免一次任務餓死別人。

## 常見追問（含答案）

**Q：Continuous batching 和「加大 batch size」差在哪？**  
A：加大靜態 batch 仍是「湊齊再跑、跑完再換」；連續批處理關心的是**步級成員變化**與 **prefill/decode 混跑**。只調大 batch 而不做 iteration scheduling，短請求尾延遲通常仍差。

**Q：為什麼 TTFT 好了、TPOT 卻抖？**  
A：常是調度把算力讓給新來的 prefill（或大塊 prefill），正在流式輸出的請求某幾步被「插隊」變慢。解法：限制每步 prefill token 預算、chunked prefill、互動隊列優先於離線批、分開池（不同 GPU／不同 `max_num_seqs`）。

**Q：和 PagedAttention、speculative decoding 怎麼一起講？**  
A：PagedAttention 解決「進進出出時顯存碎片」；continuous batching 解決「步級誰上誰下」；speculative 用草稿模型加速單序列 decode。三者正交：沒有分頁，連續批容易被碎片卡死；沒有連續批，GPU 利用率上不去；投機解碼是另一條加速軸。可交叉看 [PagedAttention](./paged-attention-prefix-kv.md)、[Speculative Decoding](./speculative-decoding.md)。

**Q：線上聊天與離線摘要要不要同一調度器？**  
A：理想分開：聊天要穩 TTFT／TPOT；離線要吞吐。同卡混部時至少分優先級與配額（類似線程池隔離），避免批任務大 prefill 把互動 P99 打爆。

## 小練習

**題：** 客服對話 GPU 池開了 continuous batching。監控見：平均吞吐 token/s 上升 40%，但 P99 TTFT 從 500ms → 2.5s，P99 TPOT 也抖。RAG 平均 prompt 從 2k 漲到 6k。你會改哪三個旋鈕／架構，並各用一句說明預期效果？

**參考答案：**  
1. **調低每步 prefill 預算／開 chunked prefill**：大 6k prefill 拆步，避免單步搶光算力 → TPOT／同行 TTFT 更穩。  
2. **降 `max_num_seqs` 或拆「互動／批處理」隊列**：限制同時搶 KV／算力的請求數，或批任務降優先 → 排隊可控、P99 TTFT 回落。  
3. **壓 prompt（RAG top-k／重排後截斷）+ 盯 prefix cache 命中**：從根減少 prefill 量；命中系統前綴時 TTFT 再降。指標：queue wait、prefill/decode 時間佔比、KV 佔用、P99 TTFT／TPOT。
