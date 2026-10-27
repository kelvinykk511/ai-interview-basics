# Prompt 與模型版本管理（Prompt Registry、Pin 版本、模型下架遷移）

## 人話
Prompt 其實係「代碼」，模型係「依賴庫」。你唔會畀生產環境用 `latest` 版本嘅 jar，但好多團隊 prompt 寫死喺代碼字串、模型用 `gpt-x-latest` 別名——某日 provider 靜靜升級，輸出格式變咗，下游 JSON parse 爆晒。版本管理就係：**prompt 同模型都要有明確版本、可回滾、可追溯每個回答用咗邊個版本。**

## 面試短答
- **Pin 模型 snapshot**（帶日期嘅版本號），唔用浮動別名；升級當成一次依賴升級走 eval → shadow → canary。
- **Prompt registry**：prompt 模板 + 變量 schema + 模型參數（model、temperature、max_tokens、tools schema、response_format）打包成一個不可變版本 `prompt_id@v12`；改任何一項都係新版本。
- **發佈同部署解耦**：代碼引用 `prompt_id` + label（`prod`/`canary`），label 指向邊個版本喺 registry 改，可以唔重新部署就回滾。
- **每個請求 log**：`prompt_version`、`model_snapshot`、參數、檢索 index 版本 → 出事可以重放（replay）同歸因。
- 用 offline eval 集做 **回歸測試門檻**（CI gate），唔過唔准推 label。

## 常見追問（連答案）
**Q1：Provider 宣佈舊模型 3 個月後下架，你點遷移？**
A：① 盤點：由請求 log 查邊啲 prompt / 服務用緊該 snapshot、流量多少。② 對每個 prompt 喺新模型跑 eval 集，比較質量、格式合規率、延遲、成本（新模型 tokenizer 唔同，token 數同成本都會變）。③ 唔合格嘅 prompt 針對新模型調整 → 出新版本。④ shadow → canary 按流量比例切，監控護欄指標。⑤ 下架日前留 buffer，並設告警：仍有流量打舊 snapshot 就 alert。

**Q2：temperature=0 + 同一 snapshot，點解輸出仍然可能唔一樣？**
A：GPU 浮點運算喺唔同 batch 組合 / kernel 下非完全確定（batch 大小影響運算順序），MoE 路由同 provider 端基礎設施變更都會令結果有細微差異。所以回歸測試唔可以做「逐字比對」，要用結構校驗 + 語義/LLM judge 評分 + 閾值，而唔係 exact match。

**Q3：Prompt 存喺 DB 定 Git？**
A：常見做法係 Git 做真相來源（code review、diff、CI eval gate），發佈時同步到 registry/配置中心畀運行時讀，運行時帶本地 cache + 版本號；咁有審計又可以熱切換。純 DB 改冇 review 好易出事，純 Git 又要重新部署先回滾得。

**Q4：A/B 測兩個 prompt 版本要注意咩？**
A：按用戶/會話做 sticky 分流（同一對話中途換 prompt 會污染結果）；兩邊用同一模型 snapshot 同檢索 index 版本，否則變量混雜；指標要預先定好（任務成功率、拒答率、成本、延遲），樣本量夠先下結論。

**Q5：Prompt 改動點做 code review？**
A：除咗睇文字 diff，PR 要附 eval 報告：通過率變化、退化嘅 case 清單、token / 成本變化。Few-shot 例子、tools schema、system prompt 嘅細小改動都可能大幅影響行為，所以「改一個字」都要過 gate。

## 小練習（題目 + 參考答案）
**題：** 交易所客服 bot 週一早上突然大量回答格式錯（應該回 JSON 卻夾雜 markdown），下游工單系統解析失敗。上週五冇任何部署。你點排查同防止再發生？

**參考答案：**
1. 先查請求 log 嘅 `model_snapshot`：如果用緊浮動別名（例如 `*-latest`），好大機會 provider 週末更新咗底層模型——呢個係最常見原因。同時檢查 prompt registry 有冇人改 `prod` label（「冇部署」唔代表冇改配置）。
2. 止血：label 回滾到上一個已知良好嘅組合（pin 回舊 snapshot 如仍可用）；下游加 **防禦式解析**：用 structured output / JSON schema 模式 + 校驗失敗自動重試一次 + 再失敗轉人工隊列，唔好直接爆。
3. 根治：所有服務改用 pinned snapshot；prompt + 模型參數入 registry 版本化；CI 加格式合規率 eval gate（例如 ≥99.5%）；線上監控「schema 校驗失敗率」做告警，唔好等用戶投訴先知。
4. 加 provider changelog / deprecation 公告嘅訂閱同定期 eval，主動測新 snapshot，而唔係被動被升級。
