# LLM Streaming（SSE）：首字體感、取消與背壓

## 人話

用戶不想等整段答完才一次跳出——所以 API 一邊生成一邊把 token **流式推送**。常見協定是 **SSE（Server-Sent Events）**：一條 HTTP 長連線，伺服器持續 `data:` 推事件。對你這種後端來說，它很像「chunked response + 心跳」，但要認真處理：**首包延遲、中途取消、斷線重連、代理超時、以及下游 GPU 還在算卻沒人聽**。

## 面試短答

- **為何 stream**：降體感等待（TTFT 有字就出）、可提早停（用戶按 Stop）、UI 可邊顯示邊解析。
- **SSE vs WebSocket**：單向伺服器→客戶端用 SSE 就夠（多數 LLM chat）；要雙向控流／多路複用才上 WS。SSE 走普通 HTTP，易過 CDN／負載均衡；注意中間代理 idle timeout。
- **事件形狀**：常見是 `delta`（增量 token）+ 最終 `done`／usage；有的還推 tool_call 片段。客戶端要做**緩衝拼接**，不能假設每個 event 是完整詞。
- **取消（cancel）**：客戶端關連線不夠——閘道要**把取消傳到推理 worker**，停 decode、釋放 KV，否則 GPU 白算。像取消長查詢要真正 kill 查詢。
- **超時分層**：連線建立 / TTFT / 整段 wall-clock / idle-between-tokens 分開設。只設一個 60s 總超時，長答會誤殺；只設 idle，卡死 prefill 又抓不到。
- **背壓**：客戶端讀得慢 ≠ 模型該無限緩衝。閘道要有有界 buffer；滿了就降速／斷開／丟棄並記 metrics，避免 OOM。
- 後端類比：SSE ≈ chunked HTTP；cancel ≈ 取消 Future／查詢；TTFT ≈ TTFB；idle timeout ≈ 反向代理的 proxy_read_timeout。

## 常見追問（含答案）

**Q：SSE 斷線後怎麼「續傳」？LLM 能從第 N 個 token 接上嗎？**  
A：多數供應商**不支援**真正的 token 級 resume。實務：客戶端保留已顯示文字；若要重試，帶同一 conversation／把已生成當 assistant 前綴再請求（貴、且可能略偏），或乾脆重生成並標「已中斷」。產品上要區分「網路抖一下」vs「用戶取消」。

**Q：Nginx／ALB 把 stream 卡住或一次吐完，常見原因？**  
A：緩衝（`proxy_buffering on`）、gzip 整段壓縮、idle timeout、HTTP/1.0 下游。要關代理緩衝、拉長 read timeout、確保 chunked／HTTP/1.1+，並用心跳（comment 行或 ping event）撐過空檔。

**Q：Stream 時 usage／計費 token 何時才準？**  
A：很多 API 在**最後一個 event**才給完整 usage。中途取消可能只有部分；閘道要在 `done` 或連線結束時落帳，並處理「取消後仍收到尾包」的競態。

**Q：同一用戶連打 Stop 再開新請求，GPU 側要注意什麼？**  
A：舊請求必須可取消且 KV 可回收；否則「幽靈請求」佔顯存。請求要有 `request_id`，取消與新請求用同一 conversation 鎖或版本號，避免亂序寫入對話歷史。

## 小練習

**題：** 你們用 SSE 做客服 bot。監控顯示 P99 TTFT 正常，但用戶投訴「字跳一下就停半晌」或「按停止後帳單還在跑」。列三個最可能根因與你會加的觀測點。  

**參考答案：**  
1. **Token 間隔（TPOT）抖動／隊列**：首包快但 decode 卡住——看 `time_between_tokens` P99、GPU 利用率、continuous batch 是否被長 prompt 擠爆。  
2. **代理／客戶端緩衝**：事件已出閘道但瀏覽器或 CDN 緩衝——對比「閘道發出第一個 delta 的時間」vs「瀏覽器 onmessage」；關 proxy_buffering、加 flush。  
3. **取消未傳到 worker**：Stop 只關了前端 EventSource，推理仍跑完——看 cancel 成功率、取消後 worker 是否還產生 token、KV 釋放延遲；確保閘道在 disconnect 時 abort 上游並冪等落帳。
