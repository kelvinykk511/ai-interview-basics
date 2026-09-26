# PagedAttention 與 Prefix KV Caching（推理引擎）

## 人話

自回歸生成要把每層的 **KV cache** 留在 GPU：序列越長、併發越高，顯存越像「每個連線一塊連續 buffer」。傳統連續分配容易**碎片化**——總量夠卻挖不出整段，被迫降 batch。**PagedAttention**（vLLM 系思路）把 KV 切成固定大小的 **page／block**，用頁表映射邏輯序列→實體 block，像 OS 虛擬記憶體。再配合 **prefix caching**：相同前綴（常見系統提示）的 KV block **跨請求共享**，少做重複 prefill。後端類比：**連線池 + 共享 buffer 分頁**——不要每人 malloc 一大塊連續堆。

> 和 [`prompt-caching-cost.md`](./prompt-caching-cost.md) 的差別：那篇偏 **API／計費層**「穩定前綴少付錢」；本篇是 **serving 引擎裡 KV 分頁與跨請求 block 復用**。目標都像「少做重複 prefill」，層次不同。

## 面試短答

- **痛點**：KV 隨 `batch × layers × seq_len` 漲；預留「最大長度」的連續槽位 → 內部碎片＋外部碎片，實際利用率差，限制 continuous batching。
- **PagedAttention**：KV 存成固定大小 block；block table 記錄每個請求的邏輯 token 落在哪些實體 page。分配／釋放以 page 為單位，活像 malloc→分頁。
- **和 continuous batching**：迭代間可進進出出請求；page 級分配讓「新請求插入、短請求結束還頁」不必搬整段巨大連續 KV。吞吐上去的關鍵常是「顯存碎片不再卡死 batch」。
- **Prefix KV caching**：多請求共享**完全相同**的 prompt 前綴時，共用同一組 KV block（copy-on-write／引用計數）。系統提示 8k、每秒上千次相同前綴 → TTFT／prefill 算力明顯降。
- **命中與淘汰**：按前綴內容（通常 hash／radix）索引；顯存緊時 LRU／引用計數為 0 的 block 可回收。命中率取決於前綴穩定度與熱度。
- **風險**：共享鍵必須是**精確前綴位元級一致**；錯共享 = 上下文串台（租戶／會話錯亂）。與產品「prompt cache」計費命中不是同一個開關，但都要求「固定前、變動後」。
- 後端類比：page table ≈ 緩衝區槽位表；prefix share ≈ 多連線共用只讀 protocol header buffer；錯 key ≈ 連線復用到別人 session。

## 常見追問（含答案）

**Q：為什麼「總顯存還有空」卻還是 OOM 或被迫縮小 batch？**  
A：連續 KV 槽位造成**碎片**：空閒是碎洞，塞不下下一個長請求的整段預留。PagedAttention 用小 page 填洞，有效可用率上升。面試不要只答「買更大 GPU」——先講分配粒度。

**Q：Prefix caching 和 continuous batching 一起開，TTFT 怎麼變？**  
A：命中前綴時，prefill 可跳過已快取段（或大量減少），**TTFT** 常明顯下降；decode 仍按新 token 走。未命中時仍要完整 prefill。高命中＋連續批處理：GPU 更常忙在 decode／新後綴，而不是重複啃同一段系統提示。

**Q：和雲廠商 Prompt Caching（帳單上那行）怎麼對齊回答？**  
A：都是「穩定前綴少做重複 prefill」。雲 API 的 cache 是**產品／計費抽象**；自建引擎的 prefix KV 是**顯存裡的 block 復用**。自建要自己管命中率、淘汰、多租戶隔離；調 API 時你看不到 page table，只能調前綴佈局與看 cache hit 指標。可交叉看 [Prompt Caching 與推理成本](./prompt-caching-cost.md)。

**Q：多租戶下 prefix 共享最怕什麼？如何防？**  
A：最怕 **key 碰撞或錯誤復用**（前綴沒含租戶隔離資訊卻共享、或 hash 實作 bug）→ 答出別人系統提示／工具定義。作法：共享索引鍵 = 精確 token 序列（含租戶專屬系統段）；默認「可共享段」與「租戶私有段」切開；私有段永不進跨租戶池；監控異常命中／抽樣稽核。

## 小練習

**題：** 內部支援 bot：系統提示＋工具 schema 約 6k tokens，所有請求相同；後面接租戶知識 RAG（每次不同）與用戶問題。自建 vLLM 類引擎。如何用 PagedAttention／prefix KV 最大化收益？列兩個會讓命中率崩掉的改動，以及你會看的三個指標。

**參考答案：**  
1. **佈局**：6k 固定系統＋工具放最前，確保 token 級完全一致；RAG／用戶問題嚴格後置。開啟 prefix block 緩存，讓這 6k 的 KV page 跨請求共享。  
2. **會崩命中的改動**：每次在前綴插入時間戳／request_id；或按租戶把不同話術塞進「前綴最前面」又希望跨租戶共享（應改為：共用段 + 租戶段分開，租戶段不共享或短）。  
3. **指標**：prefix cache **hit rate**、TTFT P95/P99、prefill tokens／秒（或 prefill 佔比）、GPU KV 顯存利用率／eviction 次數；另加錯上下文稽核（租戶串台為 0 事故）。
