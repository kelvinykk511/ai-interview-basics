# 二元量化／RaBitQ：1-bit 碼要有估距，唔係對每維取 sign 就完

> 2026-11-24 · AI warm-up · 工程面試向  
> 索引結構見 [vector-index-tradeoffs.md](./vector-index-tradeoffs.md)；PQ／int8 見 [product-quantization-ivf-pq.md](./product-quantization-ivf-pq.md)；MRL／截維見 [matryoshka-variable-dimensions.md](./matryoshka-variable-dimensions.md)；換模／版本見 [embedding-refresh-drift.md](../05-mlops/embedding-refresh-drift.md)。LLM 權重同 KV 量化係另一回事，見 [quantization-serving.md](../05-mlops/quantization-serving.md)、[kv-cache-quantization.md](../05-mlops/kv-cache-quantization.md)。**本篇講：embedding 二元量化同 PQ／int8／MRL 有何分別、RaBitQ 點解唔係天真 Hamming、點樣 over-fetch＋精排，以及版本同切片評估。**

## 人話

一條 1024 維 float32 向量佔 4KB，五千萬條約 200GB，單機放唔落。量化係「每維用少啲 bit」，同 Matryoshka 減維度係唔同軸。

最粗係**二元量化**：每維變成 1 bit，1024 維得 128 字節，約壓 32 倍；距離可以用 XOR＋popcount（Hamming）好快計。但天真噉對每維取 `sign(x_i)`，估距誤差可以好大，召回唔穩。

**RaBitQ**（Gao & Long, SIGMOD 2024）係有理論誤差界嘅隨機化二元量化：先把向量正規化到單位球面，再用隨機正交旋轉（Johnson-Lindenstrauss 類；實作常用 Hadamard 快速近似，唔使真係存一張 D×D 矩陣），然後量化成 D-bit 字串。打分用專門設計嘅**無偏**距離估計，而唔係兩側都變 bit 之後只打對稱 Hamming。論文對比 PQ：PQ 常用查表估距，估計可以有偏、亦無單次誤差保證；RaBitQ 有較尖嘅誤差界，並可用 bitwise／SIMD 實現。

後端類比：唔係「把 float 截斷成 flag」咁簡單，而係換一套有保證嘅估距編碼。面試官想聽：同 PQ／int8／MRL／權重量化有何分別、點解仍要 rerank、query 同 index 要點對齊、可唔可以丟原向量。

## 面試短答

- **先分清四條軸，再加第五條「唔關事」。** MRL：減 d。Scalar int8：d 不變、每維 8 bit。PQ：分段 codebook，通常每段 4／8 bit。Binary／RaBitQ：約每維 1 bit。LLM 權重（GPTQ／AWQ／FP8）同 KV cache 量化係 serving 問題，唔好混入 embedding 索引。
- **天真 sign／對稱 Hamming 只係弱基線。** RaBitQ 靠旋轉＋無偏估計。生產上多數係 **asymmetric**：文檔入庫量化，query 保持旋轉後嘅較高精度再用估計量，通常好過兩側都變 bit。Query 同文檔必須用同一個旋轉／方案版本。
- **二元階段幾乎一定要 over-fetch，再用更高精度精排。** 目標 top-10 時，二元層先取更多候選（常見係數百），再讀 float／int8／殘差做 refine。只把 Hamming top-10 直接送生成，長尾同近鄰好易亂。
- **原向量（或至少精排所需精度）通常要留。** 可以放 NVMe 或分層，只對候選隨機讀。若量化後刪晒原向量，就冇 refine，誤差直接進最終排名；要咁做必須用業務 slice 證明 end-to-end 可接受，並留回滾。
- **唔同量化方法、旋轉、維度，唔好混同一個 collection。** `index_version` 要帶 `embed_model + dims + quant_scheme`（含旋轉／估計量版本）。改方案等於切庫，藍綠＋alias，同 embedding 換模紀律一致。
- **同 IVF／HNSW、同 MRL 都可疊加，但要分開評估。** 量化壓向量，圖或倒排桶負責候選；RaBitQ 唔自動取代 nprobe／efSearch 調參。亦可 MRL 截短再二值，當兩軸 ablate，唔好當成同一招。
- **質量損失唔均勻。** 平均 recall 看起來可以，精確標識、近義產品（ETH vs ETHFI）、長尾 query 可能先跌。要分 slice，唔好只報平均，更唔好單憑 ANN recall 停 BM25／cross-encoder。
- **內存帳要連圖邊、倒排桶、精排讀盤一齊算。** 1024-d binary ≈ 128B／條，五千萬條約 6.4GB 碼本身；HNSW 邊同原向量精排層另計。先用內存預算倒推，再用抽樣 flat／全精度真值畫 recall–P99 曲線。

## 常見追問（連答案）

**Q1：已經有 IVF-PQ，仲要唔要換 RaBitQ？**  
A：唔係改名升級就換。PQ 成熟、生態多；RaBitQ 賣點係 1-bit 碼＋有界無偏估計、bitwise 快。用同一組 query，對「PQ＋rerank」同「RaBitQ＋rerank」畫 recall–P99–內存。若現役 PQ 已達標，唔使為新方法切庫。

**Q2：可唔可以只存 binary，唔留 float？**  
A：極省內存時有人咁做，但精排能力大幅下降，近鄰同長尾容易亂。優先把原向量或 int8 放冷存，只對候選讀返嚟 refine。丟之前要用業務 slice 證明端到端可接受，並保留舊索引回滾。

**Q3：Query 要唔要一併量化成 bit？**  
A：對稱 Hamming 最簡單，但通常較差。RaBitQ 類方法按估計量做，query 側多數保持旋轉後嘅較高精度（asymmetric）。兩邊必須同一旋轉／版本；query cache key 要帶 quant 版本，唔好同全精度向量共用快取。

**Q4：同 Matryoshka、int8 點配合？**  
A：正交，唔互相取代。可以 MRL-256＋RaBitQ 做粗召回，再用同條向量嘅更高維或更高 bit 精排。每加一軸都要獨立 ablate；版本釘埋 d 同 quant。有 MRL 唔等於可以跳過量化評估，反之亦然。

**Q5：量化後平均 recall@10 只差一個百分點，可唔可以停 BM25 同 cross-encoder，直接全量切？**  
A：唔可以單憑平均 ANN recall 做呢兩個決定。量化改嘅係估距，對精確代號、拼寫、稀有產品名未必有幫助。先看 slice 同端到端，再用藍綠切流；lexical／精排層要獨立證明先好停。

## 小練習

**題：** 5000 萬 chunk、1024 維 float32（約 205GB 原向量）。同事建議：全庫用天真 `sign(x_i)` 做成 binary HNSW，原地覆寫現役 float 索引，原向量刪除以省盤；query 亦取 sign 之後只打 Hamming，取 top-10 直接送生成；同一 collection 混入尚未量化嘅新文檔；平均 recall@10 同舊 HNSW 只差一個百分點就全量切，並停 BM25。另外有人話「模型已經支持 Matryoshka，截到 256 維就唔使量化」。你會點評，同埋會點落地？

**參考答案：**  
1. **唔好把天真 sign 當生產方案。** 至少用 RaBitQ（或庫內等價嘅非對稱／有界估計）對同一組 query 對比 baseline。  
2. **唔好原地覆寫，亦唔好未證明就刪晒原向量。** 藍綠建新索引；原向量或高 bit 留作 over-fetch 後精排。  
3. **唔好把量化同未量化向量混進同一個 collection。** 新文檔走同一 pipeline，或分開 collection／alias。  
4. **Hamming top-10 直接生成太緊。** 二元層要 over-fetch，再用更高精度 refine。  
5. **平均只差 1% 唔等於可以全量切，更唔等於停 BM25。** 要睇 ETH／ETHFI、錯誤碼、長尾等 slice 同端到端；lexical 獨立評估。  
6. **MRL 同量化唔互斥、亦唔互相取代。** 截維解決嘅係 d，量化解決嘅係每維 bit；可以疊加，但要分開量。  
7. **落地：** 內存帳分開碼／圖／精排層；`index_version` 釘 `model + dims + quant_scheme`；抽樣畫 recall–P99，最後睇答案質量；用 alias 切流同回滾。
