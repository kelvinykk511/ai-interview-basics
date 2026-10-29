# 離線批量推理（Batch Inference / Batch API）

## 人話
唔係每個 LLM 調用都要即時答。打標、摘要歷史工單、離線 eval、embedding 重建呢類「今晚做完就得」嘅工作，丟去 **Batch API**（OpenAI / Anthropic 都有）：一次交一個 JSONL 檔，供應商喺 **最多 24 小時** 內做完，價錢大約 **五折**，而且用獨立配額，唔會食咗線上 TPM。代價係：冇延遲保證、要自己處理部分失敗同對帳。

自建 GPU（vLLM 等）同一道理：離線任務用大 batch、塞滿 GPU、可以用 spot/閒時資源，吞吐優先而唔係 TTFT 優先。

## 面試短答
> 線上同離線分開兩條路：線上走同步 API，保 TTFT 同 SLA；離線（打標、回填、eval）走 Batch API 或自建離線隊列，吞吐優先、成本約五折、配額獨立。我會把大任務切成多個 batch 檔（每行帶 `custom_id` 做冪等鍵），提交後記錄 batch_id 同狀態機，輪詢或 webhook 收結果；結果按 `custom_id` 對回原始記錄，因為輸出順序唔保證。部分失敗、過期的行抽出來重送，唔好成批重跑。整個 job 要可恢復：崩咗重啟能從 DB 狀態續跑，唔會重複提交雙倍收費。

## 關鍵設計點
- **冪等鍵 `custom_id`**：用業務主鍵（如 `ticket:12345:v3`，連 prompt 版本），結果按佢 upsert；同一 custom_id 已成功就唔重送。
- **狀態機**：`PENDING → SUBMITTED(batch_id) → IN_PROGRESS → COMPLETED / EXPIRED / FAILED`，再到行級別 `succeeded / errored / expired`。提交前先落庫「準備提交」，拿到 batch_id 即寫入——防止「提交成功但未記錄 → 重啟又提交一次」。
- **切檔**：供應商有單檔行數/大小上限（如 5 萬行、~200MB 級別，以官方文件為準）；按上限切，並按 prompt 長度控制每檔 token 總量，避免撞 batch 隊列 token 配額。
- **結果不保序**：一定按 custom_id 對，唔可以靠行號。
- **部分失敗**：解析 error 檔，分類：4xx（請求本身壞，如超 context）→ 修正或標記放棄；5xx/過期 → 重放入下一批；設最大重試次數。
- **版本釘死**：batch 可能跑十幾個鐘，模型 snapshot、prompt 版本寫入每行 metadata，結果可追溯（同上次「prompt/模型版本管理」連住）。
- **成本與觀測**：按 job 記錄 input/output tokens、成功率、平均完成時間；設預算上限，超支自動暫停。
- **輸出要校驗**：離線冇人即時睇，用 JSON schema 校驗 + 抽樣人工/LLM-judge 檢查，壞輸出唔好直接寫入正式表（先寫 staging 表再切換）。

## 常見追問（連答案）
**Q1：咩情況唔應該用 Batch API？**
答：用戶喺等嘅（聊天、客服即時回覆）、有 SLA 嘅風控判斷、或者結果要喺幾分鐘內用嘅。Batch 最壞 24 小時，仲可能過期，只適合可延遲任務。折衷：急嘅少量走同步 API，其餘入 batch。

**Q2：Batch 同自己用同步 API + 限流慢慢跑，有咩分別？**
答：同步慢跑：全價、食線上配額（可能擠爆線上用戶）、要自己做重試限流，但即時可控。Batch：約五折、獨立配額、供應商幫你排程，但完成時間不確定、要處理檔案與對帳。量大又唔急，用 batch；量細或要可控完成時間，用同步 + 低優先隊列。

**Q3：Job 跑到一半服務重啟，點樣唔重複收費？**
答：所有狀態落庫。提交前寫「submitting + 檔案 hash」，拿到 batch_id 寫回；重啟時先查 DB 有冇 batch_id，有就去查狀態而唔係重新提交；若「submitting」但冇 batch_id，先用供應商 list batches 按 metadata/檔案名查有冇已建立，冇先重交。同「轉帳先記錄再打外部」同一套思路。

**Q4：自建 GPU 做離線推理，點同線上共存？**
答：線上優先：離線任務用低優先隊列，GPU 有空閒先跑（或用獨立池/spot 節點）；batch size 開大、max_num_seqs 大，追吞吐；線上流量升時可搶佔/暫停離線任務。要監控線上 TTFT，唔好俾離線拖慢。

**Q5：點估一個 batch job 成本？**
答：先抽樣 1% 行計 token（用目標模型 tokenizer），乘總行數，output 用 max_tokens 上界或歷史平均；乘 batch 價（約同步價五折），加重試緩衝 10–20%。提交前同預算比，超就拒絕或分日跑。

## 小練習（附參考答案）
**題目：** 交易所要為過去 2 年 300 萬張客服工單打「問題類別 + 情緒」標籤，餵俾 BI。你點設計？如果跑完發現 4% 行失敗、1% 輸出 JSON 唔合法，點處理？

**參考答案：**
1. 走 Batch API（唔急、量大、慳一半錢、唔搶線上配額）。先用 500 張做小樣：定 prompt + JSON schema、用人工標註對比準確率，同時估 token 成本。
2. 每行 `custom_id = ticket_id:prompt_v2:model_snapshot`；按上限切成約 60 個檔，控制並行提交數量；每個 batch 狀態落 DB。
3. 結果按 custom_id upsert 入 staging 表；schema 校驗唔過嘅 1% 標記 invalid。
4. 4% 失敗：讀 error 檔分類——超 context 嘅先截斷/摘要再送；5xx/過期嘅自動放入重試批，最多 3 次；仍失敗嘅標 `unlabeled` 唔阻塞報表。JSON 唔合法嘅 1%：改用 structured output/嚴格 schema 重送，或者用便宜模型修復再校驗。
5. 抽樣 1% 做品質檢查（人工或 LLM-judge），達標先由 staging 切到正式表；記錄每行 prompt/模型版本，日後換模型可增量重跑。
