# GPU 容量規劃與自動擴縮（LLM Serving）

## 人話

[`multi-lora-capacity-planning.md`](./multi-lora-capacity-planning.md) 講**一張卡裡**底座／adapter／KV 點分帳；本篇講**多卡、多 replica** 層：要幾張卡？何時加？隊列頂唔頂得住？後端類比：**一台 JVM 嘅 heap sizing** vs **K8s HPA + 有界隊列 + 限流**——前者係單機算力帳，後者係集群彈性。LLM 特別之處：冷啟動要載權重（分鐘級）、擴縮指標唔係單純 CPU／RPS，而係**GPU 利用率 + 隊列深度 + TTFT／TPOT SLO**。面試要講得出「先用公式估 floor，再用壓測定 ceiling，HPA 跟隊列唔好盲跟 GPU%」。

## 面試短答

- **單 replica 容量帳（floor）**：  
  `可賣 QPS ≈ f(max_num_seqs, 平均 gen 長度, TPOT, prefill 佔比)`；先按目標 P95 context 估 KV → 定 `max_num_seqs`（見 multi-LoRA 篇），再倒推「一卡能穩接幾多互動請求」。公式只係 floor，長尾 prompt／工具調用會食穿。
- **卡數估算**：  
  `replicas ≈ ceil( peak_QPS / per_replica_QPS_at_SLO ) × (1 + headroom)`；headroom 通常 20–40%（突發、rebalance、滾動升級）。批處理池同互動池**分池**，唔好用同一 HPA 曲線。
- **擴縮信號（優先級）**：  
  1）**入場隊列深度／排隊時延**（最貼用戶體感）；2）**TTFT／TPOT P99** 穿 SLO；3）**KV／顯存水位**（接近 OOM 要先縮併發或拒收，唔係盲目加卡）；4）GPU SM 利用率作輔助——利用率低但隊列長 → 可能係 prefill 重或調度問題，唔係「加卡就好」。
- **HPA 陷阱**：跟 RPS 會miss「長 prompt 少 RPS 卻打爆 GPU」；跟 GPU% 會miss「隊列已爆但 GPU 仲喺 decode 慢火」。實務：**自定義 metrics（queue_wait_ms、inflight、kv_used）+ 穩定窗**，避免震盪。
- **冷啟動**：模型權重載入、CUDA graph／編譯、連線池預熱 → scale-up 延遲遠高過 Java pod。對策：過佈（over-provision）最小副本、預熱池、預測性擴縮（活動／發版窗口）、或 gateway 排隊 + 友好降級。
- **縮容安全**：有 in-flight decode 唔好硬殺；drain 完再下線；縮得太兇 → 下一波尖峰又冷啟動。設 `minReplicas` 按業務谷底，唔好 set 0（除非真 batch／可接受冷啟）。
- **隊列與背壓**：有界隊列 + 超時 + 429／降級（短答、更細模型、拒答）；無限隊列 = 假活。同 [`llm-gateway-resilience.md`](./llm-gateway-resilience.md) 限流／隔離一齊講。
- **成本路由（一句）**：簡單／可 cache 請求走細模型或 semantic cache；重活留大卡——擴縮之前先分流，往往比加卡更便宜。
- 後端類比：`max_num_seqs` ≈ 連線池大小；queue depth ≈ Tomcat accept queue；HPA on queue ≈ 跟線程池排隊擴實例；冷啟動 ≈ 大 fat JAR 首次 JIT＋暖庫。

## 常見追問（含答案）

**Q：點樣由「業務 SLO」推到「要幾張 A100／H100」？講一輪。**  
A：1）定 SLO（P99 TTFT／TPOT、可接受拒收率）；2）量長度分佈 → 估 KV → 單卡 `max_num_seqs`；3）壓測得出**喺 SLO 內**嘅 per-replica 可持續 QPS（唔係峰值瞬間吞吐）；4）`replicas = peak_QPS / that × headroom`；5）分池（互動／批／冷 LoRA）；6）加 gateway 隊列上限同 429；7）用一週真實流量回放校準。面試強調：**用「SLO 內可持續 QPS」唔好用「GPU 打滿時嘅 token/s」**。

**Q：HPA 跟邊個指標？點解唔好淨跟 GPU utilization？**  
A：GPU% 高可能係健康滿載 decode；GPU% 低可能係卡喺 CPU tokenizer、等工具、或請求太短。用戶痛點係**排隊同 TTFT**。建議主指標：`queue_wait_p95`、`inflight / max_num_seqs`、`ttft_p99`；GPU%／SM active 作輔助同容量告警。擴縮要有 cooldown，防震盪。

**Q：尖峰突然 3×，卡加唔切，系統應該點表現？**  
A：優先**有界背壓**：超隊列 → 429／「稍後重試」＋ Retry-After；可選降級路由（細模型、只摘要、關工具）；保住已入場請求嘅 SLO，唔好無限入隊令人人超時。同時觸發 scale-out；若冷啟太慢，平時 `minReplicas` 就要預留尖峰一部分。CEX 客服類：資金相關寧可排隊提示，唔好幻覺亂答。

**Q：同 multi-LoRA 容量篇點分工？面試被問「你點規劃 GPU」點答？**  
A：兩層：① **單卡內**——W + adapter pin + KV → `max_num_seqs`（multi-LoRA 篇）；② **集群**——per-replica SLO QPS → replica 數、HPA、隊列、分池（本篇）。先做 ① 定「一卡賣幾多」，再做 ② 定「要幾卡同點彈」。漏咗任何一層都會 OOM 或 P99 爆。

**Q：縮容時 in-flight 長生成點處理？**  
A：標記 pod Terminating → 停接新請求 → 等 decode 完成或超過 graceful 期限再殺；超長請求可遷到「長任務池」或設 `max_tokens` 上限。同 K8s `preStop` + gateway 摘流一致。盲縮會截斷 SSE，用戶當故障。

## 小練習

**題：** 互動客服 70B，單卡喺 P99 TTFT < 1.2s 下可持續 ~8 QPS；日峰 40 QPS、低谷 5 QPS；偶發行銷活動可到 80 QPS 持續 20 分鐘。冷啟動載模約 4 分鐘。設計 replica／HPA／隊列策略，並寫兩條驗收指標。

**參考答案：**  
1. **基線**：`minReplicas = ceil(5/8) → 至少 2`（低谷都留冗餘同滾升）；常規峰 `ceil(40/8×1.3) ≈ 7` 作 `max` 或目標水位。  
2. **HPA**：跟 `queue_wait_p95` 同 `inflight/capacity`；cooldown ≥ 冷啟時間；活動窗口**預先** scale 到 ~10–12（80/8×1.2），唔好等隊列爆先加。  
3. **隊列**：有界（例如 per-gateway 許可达 1～2× 總 inflight）；超限 429；批處理／離線評分另池，唔食互動卡。  
4. **驗收**：① 日常峰 P99 TTFT < 1.2s 且 429 < 0.5%；② 活動期預擴後 429 < 2%、無 OOM；冷啟期間靠隊列＋提示頂住，唔靠無限排隊。  
5. **一句**：單卡定 SLO-QPS → 乘峰値加 headroom → HPA 跟隊列唔跟虛高 GPU% → 冷啟用過佈／預熱補。
