# PQ／IVF-PQ 與 int8：分段碼本壓縮，對照 RaBitQ／MRL

> 2026-11-26 · AI warm-up · 工程面試向  
> 索引總覽見 [vector-index-tradeoffs.md](./vector-index-tradeoffs.md)；二元量化／RaBitQ 見 [binary-quantization-rabitq.md](./binary-quantization-rabitq.md)；MRL／截維見 [matryoshka-variable-dimensions.md](./matryoshka-variable-dimensions.md)；換模／版本見 [embedding-refresh-drift.md](../05-mlops/embedding-refresh-drift.md)。LLM 權重同 KV 量化係另一回事。**本篇講：Product Quantization 同 IVF、int8 標量量化點揀、非對稱距同 over-fetch 精排、碼本漂移與版本，以及同 RaBitQ／MRL 點樣唔好混軸。**

## 人話

一條 1024 維 float32 約 4KB，五千萬條約 200GB，圖索引再加一截，單機内存放唔落。上一篇講過 1-bit 嘅 RaBitQ；呢篇講生產上更常見嘅兩檔：**int8 標量量化**（每維 8 bit，大約壓 4 倍，誤差細），同 **PQ（product quantization）**（把向量切成 m 段，每段用一本小碼書表示，常見每段 1 byte，768 維 m=96 就由 3KB 壓到 96B）。

PQ 嘅直覺唔係「整條向量對一個大碼書」，而係「分段各買一個粗代表」：每段喺自己嘅子空間做 k-means（好常見 256 個中心，所以 8 bit），一條向量變成 m 個代號。查詢時多用 **ADC（asymmetric distance computation）**：query 保持浮點，預先算佢每一段到各中心嘅距離，再對庫內每條碼做 m 次查表加總——快，但係近似距。

**IVF-PQ** 再加一層倒排：先用粗聚類（`nlist` 個桶）決定掃邊幾桶（`nprobe`），桶內先對 **殘差**（向量減桶中心）做 PQ，候選再拉回較高精度精排。後端類比：分區裁剪＋列存壓縮；近似掃描之後一定要回表用原值重排，唔好把壓縮距當成最終分數。面試官想聽：同 RaBitQ／MRL 有何分別、點解唔刪原向量、碼本同 IVF 中心何時要重建、以及精確題仲要唔要 BM25。

## 面試短答

- **量化同截維係唔同軸。** MRL 係對訓練好嘅前綴維度截短再 L2 normalize；PQ／int8 係保留維度、降低精度。可以組合（先截到 512，再 int8 或 PQ），但要當新編碼方案，唔好 silently 疊上去。
- **int8 同 PQ 係不同壓縮檔。** int8（scalar quantizer）每維獨立量化，通常 4×，召回跌得少，中等規模加分片就好常用。PQ 靠子空間碼本，壓縮率高得多（十幾到幾十倍），誤差亦大，必須 **over-fetch＋原向量（或更高精度）精排**。唔好以為「都係量化」就可以用同一套閾值。
- **亦唔等於 RaBitQ。** RaBitQ 係 1-bit 碼＋旋轉＋無偏估距，常見再加非對稱 query；PQ 係分段碼本＋查表加總。壓縮率可以同一個數量級，誤差形態同調參唔一樣，**唔可混同一庫、唔可共用同一條距離閾值**。
- **ADC 係近似距，唔係 cosine 原值。** 唔好用 PQ／IVF 粗分直接當「相似度 > 0.8 就採用」。流程一般係：IVF 縮範圍 → PQ／ADC 拉一擴大候選（例如要 top-10 就先取幾十到幾百）→ 用 float／int8 原向量重算距離再截 k。精排向量可以放 NVMe，但唔好刪到無法 refine。
- **IVF 參數同漂移要當運維項。** `nlist` 常按數據量級取（經驗上大約數倍 √N 嘅量級，要靠曲線調而唔係死背公式），`nprobe` 越大越準越慢。新數據擠喺少數桶、桶大小極不均，就會又慢又漏。監控桶分佈，定期 retrain 中心同按需重建 PQ 碼本，藍綠切 alias。
- **碼本要喺代表集上訓，而且要釘版本。** 編碼參數包括：embed 模型、`pq_m`／每段 bit、是否 residual、OPQ／旋轉、`nlist`、訓練樣本版本。換模、大遷移語料、或碼本改動＝新 index。禁止用幾千條無關語料訓完就套到費率庫；亦唔好把唔同 `m`／碼本寫進同一個 collection。
- **新增可以編碼，碼本唔會自己變好。** 用舊碼本對新向量 encode 係做得到嘅，但分佈變咗，量化誤差同 IVF 桶會劣化。高變更語料要有重建節奏，同 embedding refresh 同一套作業。
- **平均 recall 升／跌一點，唔等於可以停 BM25。** 精確代號、產品別名、條款號仍要 lexical。評估用 flat／高精度抽樣做 ANN 真值，畫 recall–P99／內存曲線，並且按 slice（exact-id、語義、費率）看，唔好只報一個庫平均。

## 常見追問（連答案）

**Q1：點樣喺 int8、PQ、RaBitQ 之間揀？**  
A：先用內存同 P99 倒推。分片之後 int8／HNSW 放得落而且 slice recall 達標，通常先 int8，運維更簡單。再大一個數量級先上 IVF-PQ，並保留精排用原向量。内存極緊、可接受較大 over-fetch 先考慮 1-bit／RaBitQ。用自己嘅曲線選，唔好按部落格壓縮倍數拍板。

**Q2：點解 query 唔一齊量化成 PQ 碼再比 Hamming？**  
A：兩邊都量化（SDC）會疊兩次誤差。ADC 讓 query 保持高精、只量化庫向量，係 PQ 嘅預設打法。同 RaBitQ 強調非對稱 query 係同一類思想：粗掃描用短碼，最終名次用高精距。

**Q3：訓練完碼本就可以刪 float 原向量嗎？**  
A：唔建議作為預設。精排、糾紛復盤、量化回滾都需要高精向量。可以放更平嘅存儲，只對 over-fetch 嘅候選回表。若刪掉，PQ top-k 嘅誤差會直接進生成，費率／名單類題最易「很近但答錯」。

**Q4：OPQ／先旋轉有咩用？同 MRL、RaBitQ 混唔混？**  
A：PQ 假設子空間好切；維度相關就唔好切，**OPQ** 學一個旋轉再分段，通常降誤差。RaBitQ 亦用旋轉，但後面係 1-bit 估距，唔係 m 段碼本。MRL 截前綴係第三條軸。可以組合，但每加一層都要獨立評估同獨立版本號，唔好假設「有 MRL 就唔使量化」或「有旋轉就當已經上咗 RaBitQ」。

**Q5：費率語料大更新，只需把新向量 encode 進舊 IVF-PQ 嗎？**  
A：作為增量追補可以，但不能當長期方案。桶中心同碼本仍反映舊分佈，會出現熱桶、召回下跌。應監控 list 大小同 slice recall，達閾值就抽樣重訓、藍綠切庫。同時 exact-id 題唔可只靠量化 ANN。

## 小練習

**題：** 五千萬條 1024 維向量。同事建議：用 1 千條英文部落格訓一個 PQ 碼本，原地覆寫現役 HNSW 並刪除 float 原向量；用 ADC 分數 top-10 直接生成；把新上線嘅 int8 文檔同舊 PQ 碼寫進同一 collection；語料轉成中英費率公告後永不重訓；又話已經有 MRL 截到 512 維，量化同 BM25 都可以關，因為平均 recall 只跌 1%。你會點評，同埋會點落地？

**參考答案：**  
1. **碼本訓練集要代表目標語料**（中英費率／SOP），唔可以用無關英文 1k 條交差。  
2. **禁止原地覆寫同刪原向量。** 新 index 藍綠；原向量（或至少精排所需高精度）保留，ADC 先 over-fetch 再 refine。  
3. **int8 同 PQ 唔可混庫**，分數不可比。  
4. **分佈變要監控並重建** IVF 中心／碼本，唔可以「encode 完就永遠唔訓」。  
5. **MRL 截維 ≠ 量化**，平均 recall 跌 1% 唔等於 exact-id／費率 slice 可接受，更唔能因此關 BM25。落地：内存預算倒推 int8 分片定 IVF-PQ；釘 `pq_m`／碼本／nlist／是否 residual；用 flat 抽樣畫 recall–P99，按 slice 過閘先切 alias。
