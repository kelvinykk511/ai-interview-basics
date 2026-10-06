# Speculative Decoding × Continuous Batching：為何高負載時加速會消失

## 人話

[`speculative-decoding.md`](./speculative-decoding.md) 講「小模型先猜 k 個 token，大模型一次驗證」；[`continuous-batching.md`](./continuous-batching.md) 講「每一步誰上桌」。兩個疊在一起，面試最愛問：**為何 benchmark 單路快 2 倍，一上線高併發就冇乜效果，甚至更慢？**

核心直覺：speculative 賺的是「**GPU 閒置的算力**」。低併發時 decode 被顯存頻寬卡住，算力有剩，順手多驗幾個 token 幾乎免費；高併發時 batch 已經把算力塞滿，多驗的 token 要真金白銀搶算力，猜錯的更是白做。後端類比：**預取（prefetch）**——機器閒時預取很划算；機器滿載時預取只會跟正常請求搶 IO，命中率唔高就倒蝕。

## 面試短答

- **為何低負載有效**：decode 每步要把整份權重從顯存讀一遍，batch 小時是 **memory-bandwidth bound**；驗證 k+1 個位置和驗證 1 個位置成本差不多，接受率夠高就等於一次前向出多個 token。
- **為何高負載縮水**：batch 大了以後 decode 趨向 **compute bound**；每條序列每步要貢獻 k+1 個 token 進 batch，**吃掉 `max_num_batched_tokens` 預算**，擠走 prefill chunk → 其他請求 TTFT 上升；被拒絕的 token 是純浪費算力。
- **顯存也要付**：draft 模型權重 + draft 的 KV + 暫存的投機 token KV，都會壓低可容納的 `max_num_seqs`。
- **接受率因請求而異**：JSON／代碼／RAG 抄原文接受率高；創作、高 temperature 低。同一 batch 內每條接受長度不同（ragged），引擎要分別推進。
- **工程對策**：
  1. **按負載動態開關**：batch size／排隊長度超過門檻就關掉或縮小 k（部分引擎有按 batch size 自動停用 speculation 的開關）。
  2. **按請求自適應 k**：滑動窗口接受率低就降 k 或關閉該請求的投機。
  3. **更便宜的草稿**：n-gram／prompt lookup（RAG、代碼補全抄上下文時特別好用，零額外模型）、EAGLE／Medusa 類 head（共用底座特徵，比獨立 draft 模型省顯存）。
  4. **分池**：低延遲互動池開 speculation；高吞吐批處理池關掉。
- **怎樣量**：別只看單路 tokens/s。看**生產併發下**的 goodput（滿足 SLO 的吞吐）、每請求 TPOT P50/P99、接受率／平均接受長度、TTFT 有冇被擠高。
- **正確性**：標準 rejection sampling 下輸出分佈與原模型一致（lossless）；但數值上 batch 組合不同可能令 greedy 結果有細微差異，golden test 要允許。

## 常見追問（含答案）

**Q：壓測顯示 batch=1 時快 2.3 倍，batch=64 時慢 5%，你點解釋？點處理？**  
A：batch=1 時 decode 頻寬受限，驗證多 token 近乎免費；batch=64 時算力已飽和，每條多出 k 個驗證 token 直接加計算量，加上接受率不到 100%，被拒的部分是淨損失；draft 本身也佔算力和顯存。處理：設負載門檻，batch 超過約某值（壓測找交叉點）自動關閉或降 k；或者只在互動池開。結論要講「交叉點靠壓測找，不是固定常數」。

**Q：Speculative 會唔會影響其他請求的 TTFT？**  
A：會。CB 每步有 token 預算，投機序列每步佔 k+1 份額，留給 prefill chunk 的空間變少，新請求要多等幾步才完成 prefill。所以要同時盯 TTFT，不能只報 TPOT 改善。

**Q：接受率點樣監控？低到幾多應該關？**  
A：按請求類型／模型版本／temperature 分桶記錄「平均接受長度」。關閉門檻用成本算：當「接受 token 數 ÷ (k+1)」帶來的加速，抵唔住 draft 成本加驗證浪費，即壓測顯示 TPOT 不再改善時就關。實務上做成自適應，不是寫死一個數。

**Q：同 prefix caching、量化有冇衝突？**  
A：大致正交：prefix cache 省 prefill；量化省權重／KV 顯存；speculative 省 decode 步數。要注意的是三者都在搶同一份顯存和 token 預算，上線時要一齊壓測，量化後接受率也可能變（draft 和 target 的分佈差距改變）。

## 小練習

**題：** 你們的客服 Copilot（RAG，回答大量引用文檔原文）白天高峰併發 200，夜間批量摘要任務併發 800。現在全局開了獨立 draft 模型的 speculative，k=5。監控見：白天 TPOT P50 改善，但 TTFT P99 惡化；夜間吞吐比關閉時低 12%。請給方案。

**參考答案：**  
1. **分池／分時**：夜間批量池直接關 speculation（高併發、吞吐優先，已證實倒蝕 12%）。  
2. **白天換草稿方式**：RAG 大量抄原文，改用 prompt lookup／n-gram 草稿，接受率高而且唔使額外 draft 模型的顯存，可以騰出 KV 容量。  
3. **降 k + 自適應**：k 從 5 降到 2–3，按請求接受率動態調整；負載超過壓測交叉點自動關閉。  
4. **保 TTFT**：給 prefill 保留 token 預算（chunked prefill 配額），避免投機 token 擠走新請求的 prefill。  
5. **驗收**：白天 TTFT P99 回到基線、TPOT P50 仍有改善；夜間吞吐 ≥ 關閉 speculation 的基線；接受率面板按請求類型分桶。
