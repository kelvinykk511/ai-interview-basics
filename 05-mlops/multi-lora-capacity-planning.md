# Multi-LoRA × Continuous Batching：容量規劃

## 人話

[`lora-multi-adapter-serving.md`](./lora-multi-adapter-serving.md) 講清「一底多 adapter、路由與版本釘選」；[`continuous-batching.md`](./continuous-batching.md) 講清「每步誰上桌」。合併上線後，面試官要的是**容量帳**：一張卡到底能同時撐多少序列？瓶頸是底座權重、adapter 常駐，還是各請求的 KV？後端類比：**一台共用 JVM（底座）+ 一堆 plugin（LoRA）+ 每個連線一份 session（KV）**——plugin 再小，連線一多還是 session 先爆；plugin 若狂換，又會像 classloader thrash。

## 面試短答

- **顯存三塊（直覺公式）**：  
  `GPU_mem ≈ W_base + Σ(常駐 adapter) + KV(num_seqs, seq_len, dtype) + 啟動／臨時 buffer`。  
  多數多租戶場景：**KV ≫ 單個 adapter**；adapter 總量只有在「同時常駐極多、或 rank／目標模組很大」時才搶眼。
- **併發上界**：CB 的 `max_num_seqs` 實質受 **KV 預算** 卡住；LoRA 不神奇地消滅 KV。先按目標 P95 context 估 KV，再倒推可賣併發；adapter 策略在剩餘顯存裡做。
- **Adapter 常駐策略**：熱門 **pin**（常駐 GPU）；冷門 **LRU／按需載入**；中間層可「批次親和」——調度盡量讓同 adapter 的請求同窗，減少 swap。
- **Thrashing**：租戶長尾多、每個偶發請求都觸發 adapter load／unload → 延遲尖刺、GPU 忙於拷貝而非算 token。表象像「LoRA 很慢」，根因是**快取命中率低**，不是 LoRA 算子本身。
- **Merge vs 共享動態掛載**：單租戶超高 QPS、adapter 極穩 → **merge 進獨立 replica**（延遲簡、無切換，代價是失去共享底座密度）。長尾多、每條 QPS 低 → **共享底座 + 動態 LoRA** 更省卡。
- **隊列隔離**：互動 vs 批處理、或「熱 adapter 池 vs 冷 adapter 池」分開，避免冷載入／大 prefill 打爆熱路徑 P99（同 CB 的優先級／分池思路）。
- **SLO 驅動 sizing**：輸入 = 目標 QPS、P95 prompt／gen 長度、TTFT／TPOT、可接受 OOM／排隊；輸出 = 卡數、`max_num_seqs`、常駐 adapter 數、是否 merge 頭部租戶。用壓測校準，不要只拿公式交差。
- 後端類比：常駐 adapter ≈ 連線池裡 keep-alive 的熱 plugin；LRU ≈ 冷 plugin 換入；merge ≈ 流量最大的租戶獨立部署微服務。

## 常見追問（含答案）

**Q：五十個 LoRA 會不會把顯存燒光？怎麼估「能常駐幾個」？**  
A：先量單 adapter 位元組（rank、作用層數決定，通常遠小於底座）。`可常駐數 ≈ (總顯存 - W_base - KV_預算 - 餘量) / adapter_size`。若算下來只能常駐 8 個，其餘走 LRU——並用「每 adapter QPS × 載入成本」決定誰 pin。面試強調：**先扣 KV 預算再分 adapter 名額**，順序反了會低估 OOM。

**Q：Continuous batching 下，同一步 batch 混多個 adapter 行不行？代價？**  
A：許多引擎支援 **batch 內多 adapter**（分組算 LoRA 增量），功能上可行；代價是實作／kernel 複雜度、可能損失部分吞吐，且若伴隨頻繁換權重頁更糟。實務折衷：**允許混部但偏好同 adapter 聚類**；極端熱租戶獨立 replica。別假設「一開 multi-LoRA 就零開銷」。

**Q：什麼信號表示該把熱 adapter merge 成獨立模型副本？**  
A：單一 `adapter_id` 佔總 QPS 很大且穩定；動態掛載路徑上切換／分組開銷可測；該租戶 SLO 比密度更重要；或合規要求與他人隔離。Merge 後該副本不再服務其他 adapter，密度下降換來延遲與運維簡單——和「熱 key 獨立分片」同一直覺。

**Q：容量規劃面試怎麼講一輪「從需求到卡數」？**  
A：1）業務 SLO（QPS、長度分佈、P99）；2）估 KV → `max_num_seqs`；3）底座＋量化後的 W；4）熱 adapter pin 列表與 LRU 上限；5）是否分池／merge 頭部；6）壓測校準排隊與 OOM；7）留 headroom（突發、長尾 context）。能畫出這條鏈比背框架參數名加分。

## 小練習

**題：** 同一 70B 底座，12 個產品 LoRA。其中 2 個合計 70% QPS，其餘 10 個長尾。單卡 CB，目標互動 P99 TTFT < 1.5s。目前 12 個全動態 LRU，監控見 adapter load 頻繁、P99 毛刺、偶發 KV OOM。給一套「拓撲＋旋鈕」方案，並寫一句容量驗收標準。

**參考答案：**  
1. **拓撲**：2 個熱 adapter **pin**（或直接 merge 成 1～2 個獨立 replica 若單卡算力／SLO 仍緊）；10 個長尾共享另一池或同池 LRU，設 `max_resident_adapters` 上限防 thrash。  
2. **旋鈕**：按 P95 context 重算 KV → 下調或分池 `max_num_seqs`；冷池降低優先級／限 QPS；CB 開 chunked prefill；閘道按 `adapter_id` 路由熱／冷。  
3. **驗收**：熱路徑 P99 TTFT < 1.5s 且 adapter load≈0；整卡 OOM 率≈0；長尾 P99 可放寬但有隊列上限；顯存帳 = W + pin adapters + KV(headroom) 在監控面板對得上。  
4. **一句**：先用 KV 定併發天花板，再用 pin／merge 消 thrash，最後才加卡。
