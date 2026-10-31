# 2026-10-31 · AI warm-up

水準：工程面試向；問題與參考答案同頁。

1. [Agent 長期記憶（Long-term Memory Store）](../07-system-design/agent-long-term-memory.md)
2. [文檔解析 Ingestion：PDF、表格、OCR](../04-prompt-rag/document-ingestion-parsing.md)

**練習：** 交易所客服 AI「記住用戶」（異步抽取、只從用戶發言抽、upsert 決策、工單只存 id、拒記提現地址/密碼/2FA）；合規 2 萬份 PDF 接入 RAG（類型分流、layout/OCR、表格保留表頭、冪等鍵 doc_hash+parser_version、死信、分層抽樣 + eval 集驗收）。
