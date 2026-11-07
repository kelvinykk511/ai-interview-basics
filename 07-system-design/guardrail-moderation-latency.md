# Guardrail / Moderation 模型嘅延遲預算

> 2026-11-07 · AI warm-up · 工程面試向  
> 防禦分層見 [`prompt-injection-guardrails.md`](./prompt-injection-guardrails.md)；**本篇只講：審核模型放喺關鍵路徑上點食延遲、點排流水線。**

## 人話

Guardrail / moderation 即係喺主 LLM 前後再加一層「審核」——擋毒性、PII、越權意圖、違規輸出。問題係：審核本身往往又係一個模型（細分類器或者另一個 LLM judge），**每一毫秒都會疊喺用戶感受到嘅 P95 上面**。

後端直覺好似 CEX 風控引擎：入金／出金前一定要過規則，但你唔會每筆都跑一次最貴嘅 ML 分數——快規則先過，貴模型只打灰區。面試官想聽嘅唔係「我哋有 moderation」，而係：**input vs output 邊條路徑同步、邊條可以 speculative／異步；延遲預算點切；失敗時 fail-open 定 fail-closed。**

## 面試短答

- **Input vs output moderation**：*(1) Input*——用戶訊息／檢索 chunk 入閘前擋惡意意圖、PII、越權；（2）*Output*——模型回覆出閘前擋違規內容、洩密、越權建議。Input 擋得住可以慳一次主 LLM；Output 擋嘅係「主模型已經講出嚟」嘅風險。
- **Sync / async / speculative**：
  - **Sync**：審核完先畀主模型（或先畀用戶）——最安全，延遲直接相加。
  - **Async**：主路徑先回，審核事後記 log／事後封禁——只適合低風險、可事後補救。
  - **Speculative**：審核同主 LLM **並行開火**；兩邊都 OK 先 commit 給用戶。用多餘算力換牆鐘，前提係「審核拒絕時可以丟棄主模型結果」。
- **延遲預算要倒推**：產品 P95 例如 2s → 扣走網絡／gateway／工具，剩俾 `moderation + TTFT + 生成`。若小分類器 P95 80ms、LLM judge P95 600ms，後者唔可以無腦打喺每條熱路徑。
- **Cascade（瀑布）**：regex／deny-list → 細 toxicity／PII 模型 → 只對 gray zone 打 LLM judge。好似風控：規則 → 輕量分 → 重模型。
- **Streaming**：用戶期望首 token 快，但 output mod 可能未完。常見做法：*(a)* buffer 首 N token／首句，mod 過先 flush；*(b)* speculative stream（邊出邊審），命中違規即 kill switch 截斷＋撤回已推內容（產品要能接受「講到一半砍」）。
- **Fail-open vs fail-closed**：按風險分級——FAQ／行情解說可 fail-open（審核超時放行 + 異步補審）；提現建議／KYC 指引／資金操作必須 fail-closed（超時當拒絕或轉人工）。
- **指標**：`moderation_p95`、`added_latency`、`block_rate`、`false_positive_rate`（golden 正常樣本）、`moderation_timeout_rate`、每千次請求審核成本。唔好淨睇擋咗幾多，要睇誤殺同埋加咗幾多延遲。
- **CEX 類比**：出金前風控——黑名單／限額規則毫秒級；可疑先打貴模型或人工。審核超時唔等於「當通過」——高風險單會 hold。

## 常見追問（連答案）

**Q1：點解唔可以「全部用同一個大 LLM 連生成帶審核一次搞掂」？**  
A：可以做 self-critique，但**(1)** 主模型自己審自己容易被同一輪 injection 帶偏；**(2)** 審核策略（拒答邊界）同生成策略耦合，版本難獨立演進；**(3)** 延遲上你仍然要等完整生成先「審完」。工程上更常見係**獨立審核路徑**（甚至唔同供應商），同主模型解耦，方便單獨調閾值同做 cascade。

**Q2：Input moderation 同主 LLM 並行（speculative）有冇坑？**  
A：有。主模型可能已經開始生成甚至打咗 tool，審核先至話 reject → 你要能 **cancel 上游＋回滾／忽略副作用**。只讀工具猶可；寫入工具（建單、轉帳建議落地）唔好 speculative 同 input mod 並行到「已經副作用」。實務：input mod 未過之前，只允許啟動**無副作用**嘅主 LLM 推理；tool 要等 mod pass。

**Q3：Streaming 時 output moderation 點做先唔害死 TTFT？**  
A：折衷三檔：*(1)* 低風險場景允許先 stream，輸出同時跑滑動窗口分類器，命中即截斷並標 `moderated=true`；*(2)* 中風險 buffer 首段（例如前 50–100 token 或第一個句號）再 flush；*(3)* 高風險（資金／合規）唔 stream，等完整 output mod。面試金句：**TTFT 同安全係產品取捨，唔係技術必然**——用風險分級決定檔位。

**Q4：審核模型誤殺（false positive）升高，你點查？**  
A：睇 `block_rate` 同 golden 回歸嘅 FP；對比近期 prompt／模型／閾值變更；抽樣被擋請求做人工標註；檢查 cascade 係咪「細模型過緊、LLM judge 冇機會平反」。高風險路徑寧可稍嚴，但 FAQ 路徑 FP 高會直接傷體驗——要分 tier 調參，而唔係全球一個閾值。

## 小練習

**題：** 客服 bot 產品 P95 ≤ 2.0s。現況：主模型 TTFT P95 700ms、生成 600ms；你新加 output LLM-judge moderation P95 500ms（同步、生成後先審再回），上線後 P95 變成 ~1.8–2.3s，偶發超標；另外提現相關問題需要更嚴。請設計一版 **延遲預算 + 流水線**（含 cascade／speculative／fail 策略），並列要盯嘅指標。

**參考答案：**  
1. **倒推預算**：2.0s 牆鐘 − gateway/網絡 100ms ≈ 1.9s 畀業務。目標：moderation 淨增延遲 ≤ 150–200ms（P95），而唔係 +500ms。  
2. **Cascade**：先用 regex + 細分類器（P95 ≤ 50–80ms）打毒性／PII／「提現／API key」意圖；**只有 gray zone**（約 5–15% 流量）先打 LLM judge；明顯 clean 直接放行，明顯違規直接擋。  
3. **Speculative**：主模型生成時並行跑細分類器；若走 judge，對低風險 FAQ 可同生成重疊（生成完若 judge 未完，最多再等一小個 deadline，例如 150ms），超時按 tier 處理。  
4. **分 tier fail**：FAQ／行情 → 審核超時 **fail-open** + 異步補審 + 事後告警；提現／KYC／資金操作 → **fail-closed**（回「需要人工／官方渠道」），唔 stream，必要時跳過主模型直接規則拒答。  
5. **Streaming**：低風險可用 buffer 首句；提現路徑關閉 stream 或只回固定拒答模板。  
6. **指標**：`moderation_p95`、`added_latency`（有／無 mod 對照）、`block_rate` by tier、`fp_rate` on golden、`judge_invoke_rate`（cascade 有冇收窄）、`timeout_fail_open/closed_count`、審核成本 per 1k req。用呢啲確認「P95 回落到 2s 內」同「提現路徑零 fail-open」。
