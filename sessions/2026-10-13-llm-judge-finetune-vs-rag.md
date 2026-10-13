# 2026-10-13 · AI 工程溫習（LLM-as-judge 校準 + Fine-tune vs RAG）

水準：工程面試向；問題與參考答案同頁。

1. [LLM-as-judge 偏差校準](../04-prompt-rag/llm-as-judge-calibration.md)
2. [Fine-tune vs RAG 取捨](../05-mlops/finetune-vs-rag.md)

今日目標：能講清「LLM-as-judge 嘅 position／verbosity／self-preference 等偏差、pairwise swap、rubric anchors、人工 κ 對齊、judge 同 prod prompt 隔離」，以及「RAG 管會變／可引用／多租戶知識、FT／LoRA 管風格同格式行為、hybrid 與反模式（唔好 FT 背費率、唔好 FT 入 PII／秘密）」。

**Suggested next：** GPU 容量規劃／自動擴縮；embedding／vector tradeoffs refresh（如需要）；hallucination grounding 深化。
