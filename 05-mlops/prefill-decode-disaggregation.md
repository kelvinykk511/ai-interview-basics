# Prefill／Decode 分離（P/D Disaggregation）

## 人話

一次 LLM 請求有兩段：**prefill**（一次過讀完整個 prompt、建好 KV cache，產出第一個 token）和 **decode**（之後逐個 token 生成）。兩段的「體質」完全不同：

- prefill：一次處理幾千個 token，**吃算力（compute bound）**，決定 TTFT。
- decode：每步只出一個 token，但要讀整份權重和 KV，**吃顯存頻寬（memory-bandwidth bound）**，決定 TPOT。

放在同一張卡上，一個長 prompt 的 prefill 會令正在 decode 的所有用戶卡一下（TPOT 毛刺）；反過來 decode 佔滿 slot，新請求的 prefill 又要排隊（TTFT 上升）。Chunked prefill 可以在同機內減輕，但始終係搶同一份資源。

P/D 分離就是：**prefill 機器只做 prefill，做完把 KV cache 傳給 decode 機器繼續生成**。後端類比：**讀寫分離／CQRS**，或者把「批量匯入」和「線上查詢」拆成兩個服務池，各自按自己的 SLO 擴容；KV 傳輸就像 session 遷移。

## 面試短答

- **動機**：兩階段資源特性不同、互相干擾；分開後 TTFT 和 TPOT 可以**各自按 SLO 調**，互不拖累。
- **流程**：路由器 → prefill 節點算 prompt、產生 KV + 首 token → 經高速網絡（NVLink／RDMA／InfiniBand）把 KV 傳到 decode 節點 → decode 節點逐 token 生成並串流回客戶端。
- **好處**：
  1. 消除 prefill／decode 干擾，P99 TPOT 更穩。
  2. **獨立擴縮**：長 prompt 多（RAG、agent）就加 prefill 節點；長輸出多就加 decode 節點。
  3. 兩邊可以用不同並行策略甚至不同硬件（prefill 偏算力、decode 偏顯存容量與頻寬）。
- **代價**：
  1. **KV 傳輸**：每 token KV 大小 ≈ `層數 × KV頭數 × head_dim × 2(K,V) × 每元素位元組`。以 70B、GQA（80 層、8 個 KV 頭、128 維、FP16）為例約 0.31 MB／token，8k prompt ≈ 2.6 GB，要靠高頻寬網絡和逐層流水傳輸才不拖 TTFT。
  2. **調度複雜**：P:D 節點比例要隨流量調；decode 節點要預留 KV 容量才接單。
  3. **故障面變大**：decode 節點掛了 KV 就沒了，要重新 prefill；傳輸失敗要重試或回退到同機處理。
  4. **短 prompt 不划算**：傳輸開銷可能大過省到的干擾。
- **幾時值得**：規模大、prompt 長、TPOT SLO 嚴、流量形狀多變。小規模或短 prompt：**continuous batching + chunked prefill** 通常已經夠。
- **相關實作**：DistServe、Mooncake（以 KV cache 為中心的架構）等研究和系統；主流推理框架（如 vLLM、SGLang）和 NVIDIA Dynamo 都有 P/D 分離方向的支援。面試講清取捨比背名字重要。

## 常見追問（含答案）

**Q：Chunked prefill 已經可以減少干擾，點解仲要分離？**  
A：Chunked prefill 是把大 prefill 切細、和 decode 混在同一步，降低單步卡頓，但兩者仍搶同一張卡的算力和 token 預算，而且只能一齊擴容。分離解決的是「**資源隔離 + 獨立擴縮**」。規模細時 chunked prefill 性價比更高；規模大、SLO 嚴先值得分離。

**Q：KV 傳輸會唔會令 TTFT 變差？**  
A：首 token 由 prefill 節點產生，可以先串流返客戶端，所以 TTFT 主要取決於 prefill 排隊加計算；傳輸影響的是「第二個 token 幾時出」。優化方法：逐層傳輸（算完一層就開始傳）、用 RDMA、KV 量化減少位元組、短 prompt 不走分離路徑。

**Q：Prefix cache 在分離架構下放邊？**  
A：Prefix cache 主要省 prefill，所以要在 prefill 層做 **KV-aware 路由**（同一 system prompt／同一文檔前綴盡量打到有緩存的節點），甚至做跨節點共享的 KV 存儲池（Mooncake 一類思路）。否則路由隨機會令命中率大跌。

**Q：P:D 比例點定？**  
A：由流量形狀決定：平均 prompt 長、輸出短（RAG 問答）→ prefill 重，多配 prefill；prompt 短、輸出長（寫作、代碼生成）→ decode 重。用壓測量出每類節點在 SLO 下的容量，按實際 token 分佈算比例，再按監控（prefill 排隊時間 vs decode KV 佔用率）動態調整。

## 小練習

**題：** 你們的內部知識庫助手：平均 prompt 6k token（RAG 塞文檔），平均輸出 300 token，高峰 QPS 40。現在 8 張卡混部（CB + chunked prefill），監控見 TPOT P99 經常毛刺到 3 倍，TTFT 尚可。老闆問要唔要上 P/D 分離。你點答？

**參考答案：**  
1. **先判斷症狀**：TPOT 毛刺而 TTFT 正常，典型是長 prefill 干擾 decode，P/D 分離正好對症。  
2. **先試便宜方案**：調小 chunked prefill 的 chunk 大小、限制每步 prefill token 預算、加 prefix cache（RAG 常共用 system prompt）；如果 P99 已達標就唔使分離。  
3. **若仍不達標再分離**：流量是 prefill 重（6k 入／300 出），初始比例可偏 prefill 多（例如 5P:3D，再按壓測調）；prefill 層做 KV-aware 路由保 prefix 命中；確認卡間有 RDMA／NVLink 級網絡，否則每請求約 2 GB 級的 KV 傳輸會成為新瓶頸。  
4. **補故障路徑**：decode 節點掛 → 重新 prefill；傳輸失敗 → 回退混部路徑。  
5. **驗收**：TPOT P99 ≤ 基線 1.3 倍、TTFT P99 不倒退、同樣 QPS 下總卡數不增加（或增幅有明確的 SLO 收益支撐）。
