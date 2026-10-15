# 2026-10-15 · AI 工程溫習（GPU 容量擴縮 + Hallucination grounding 深化）

水準：工程面試向；問題與參考答案同頁。

1. [GPU 容量規劃與自動擴縮](../05-mlops/gpu-capacity-autoscaling.md)
2. [Hallucination Grounding 深化](../04-prompt-rag/hallucination-grounding-deep.md)

今日目標：能講清「單卡 SLO-QPS → replica／HPA／有界隊列／冷啟動；擴縮跟 queue／TTFT 唔盲跟 GPU%」，以及「retrieval vs faithfulness 分鍋、claim decomposition、NLI／槽位校驗、abstention 嘅 risk–coverage、線上幻覺訊號」；並同 multi-LoRA 單卡帳、基本 citation 篇交叉引用。

**Suggested next：** embedding／vector tradeoffs refresh（ANN／hybrid／維度／reindex 成本）；RAG eval online／shadow；GPU vs cost routing（細模型／cache／大卡分流）；prefill–decode 擴縮分池深化。
