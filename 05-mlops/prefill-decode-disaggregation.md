# Prefill–Decode 分離 serving

## 人話

LLM 推理分兩段：prefill 一次過讀晒 prompt（算力密集，決定 TTFT）；decode 逐個 token 出（記憶體頻寬密集，決定 TPOT/ITL）。同一張卡混做，長 prompt 嘅 prefill 會卡住其他人 decode → 出字一頓一頓。分離 = prefill 池同 decode 池分開機器，prefill 算完 KV cache 傳去 decode 池。

## 面試短答

- 問題係 prefill/decode 互相干擾。單機緩解：chunked prefill（長 prefill 切塊，每步有 token budget，同 decode 混 batch）。
- 分離好處：兩邊各自守 SLO（TTFT vs ITL）、各自擴縮、各自揀並行策略/硬件。
- 代價：KV cache 傳輸（要 NVLink/RDMA/InfiniBand）、調度複雜、短 prompt 低 QPS 唔值得。
- 業界例子：DistServe、Splitwise、Mooncake、vLLM/SGLang 嘅 disaggregated 模式、NVIDIA Dynamo。
- KV 大小 = 2(K,V) × layers × kv_heads × head_dim × bytes × tokens。Llama-3-8B（32 層、8 KV heads GQA、head_dim 128、FP16）= 128KB/token；8k prompt ≈ 1GiB，100Gbps(12.5GB/s) 約 86ms，400Gbps 約 21ms。

## 常見追問（連答案）

1. **prefill:decode 比例點定？** 按 workload 入/出 token 比同 SLO profiling 每池 goodput。RAG 長入短出 → prefill 重；chat 長輸出 → decode 重。兩池獨立 autoscale：prefill 跟排隊時間/TTFT，decode 跟 KV 使用率/ITL/running seqs。
2. **幾時唔值得？** prompt 短、QPS 低、冇 RDMA、模型細單卡搞掂 → chunked prefill + prefix caching 已夠。
3. **同 prefix caching 點配合？** 共享 system prompt/RAG 前綴 KV 喺 prefill 池命中；可做分層 KV store（GPU/CPU/SSD，Mooncake 思路）；路由要 cache-aware（同前綴送同節點）。
4. **失敗模式？** decode 節點死 → 進行中請求 KV 冇咗要重 prefill（或由 KV store 拎返）；網絡慢令 TTFT 反升；decode 池 KV 爆 → 拒收或搶佔。
5. **監控咩？** TTFT、TPOT/ITL p99、goodput（SLO 內請求/秒）、KV transfer latency、KV 使用率、隊列長度。

## 小練習

**題：** RAG 服務平均輸入 6k tokens、輸出 200 tokens，混合部署。用戶投訴出字一頓一頓：ITL p99 由 40ms 跳到 400ms，但 TTFT 正常。點診斷點解決？

**參考答案：** 一頓一頓 = ITL 抖動，典型係長 prefill 同 decode 擠同一個 iteration。驗證：將 ITL spike 對齊新請求到達同每步 batched prefill token 數。短期：開 chunked prefill、限 max_num_batched_tokens（如 2k）、開 prefix caching 減重複前綴。中期：QPS 高又有 RDMA 就 P/D 分離，6k:200 輸入重 → prefill 池較大；KV 傳輸估算：8B 模型 6k×128KB ≈ 786MB，100Gbps 約 63ms，要計入 TTFT 預算。
