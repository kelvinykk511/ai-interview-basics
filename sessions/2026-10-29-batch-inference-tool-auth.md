# 2026-10-29 · AI warm-up

水準：工程面試向；問題與參考答案同頁。

1. [離線批量推理（Batch API）](../05-mlops/batch-inference-offline-jobs.md)
2. [Agent 工具權限（Least Privilege / Confused Deputy）](../07-system-design/agent-tool-authorization.md)

**練習：** 300 萬張客服工單離線打標（custom_id 冪等、狀態落庫防重複提交、按 custom_id 對結果、部分失敗分類重送、staging 校驗後切換）；`call_internal_api` + admin 帳號嘅 AI 助手重新設計（細粒度工具、session 注入 user_id、delegated token、L0/L1/L2 分級、資金/安全操作交返原流程）。
