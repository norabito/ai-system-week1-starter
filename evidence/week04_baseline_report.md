# Week 4 - Baseline Report v1.0

學號：7114029034  

姓名：范睿凱  


## 1. Task & Dataset

- Task：將爬蟲獲取文章與行銷文章做語意比對，並透過指標與人工採樣標記評估模型。
- Unit：一篇文章。
- Input：`content` 欄位以及行銷文章。
- Target：`score` 欄位。
- Dataset version：`dataset_v1.0`，檔案為 `crawl/data.json`。
- Dataset size：200 rows。
- Labels：此任務非分類任務。

## 2. Baseline Plan

本週建立 baseline ladder，先確認簡單方法與既有預訓練語意模型的表現，再決定是否需要更複雜的方法。

| Role | Method | Why reasonable | Expected weakness |
| --- | --- | --- | --- |
| Baseline 1 | TF-IDF + Cosine Similarity | 最簡單的文字相似度 baseline，可檢查關鍵字重疊是否已能支援基本比對 | 無法理解同義詞、上下文與真正語意 |
| Baseline 2 | Keyword / Rule-based Similarity | 可作為可解釋的簡單 reference | 對同義詞、不同表達方式及隱含語意較弱 |
| Candidate | `paraphrase-multilingual-MiniLM-L12-v2` | 支援多語言，預訓練好的語意比對模型 | 通用語料訓練，不一定完全符合論壇文章與行銷文章的特定領域 |
| Future Candidate | Domain-specific fine-tuning | 若通用預訓練模型無法適應特定領域，可利用人工標記資料進一步調整 | 需要更多人工標記資料、訓練成本與避免 overfitting 的設計 |

本週不直接加入 LLM、RAG、Agent 或複雜 reranking pipeline，因為 Week 4 主要先建立可重現的 baseline evidence。

## 3. Fair Comparison / Experiment Spec

- Dataset：`data.json`
- Information boundary：使用 `content` 與 `marketing_content`；不能使用人工審查結果或其他事後欄位。
- Split：若有需後續訓練，少於 30 字的純灌水文、廣告文、純表情符號文章不進入模型。
- Split ratio：Train 70%、Validation 15%、Test 15%。
- Evaluation sample：隨機抽樣至少 50 篇論壇文章，並由人工判定其與候選行銷文章的相關性。
- Random seed：42。
- Test lock：Yes，同一份人工標記 evaluation set 比較所有方法。
- Primary Metric：Recall@5，方向越高越好。
- Threshold：暫定 `0.35`，`0.25 ~ 0.30` 的模糊區間交由人工確認，主要是行銷文章非短文，分數通常不會到太高。
- Failure criterion：若系統判定超過門檻的資料中，人工檢查發現實際不相關的比例高於 15%，判定系統失敗。
- Success criterion：Recall@5 ≥ 85%，且 Precision ≥ 75%。
- Extreme-score monitoring：若整批資料的 score 大量集中於接近 0 或 1，觸發人工檢查。
- Runtime / Cost：記錄各方法執行時間與模型推論成本。

Split rows(若有需要後續訓練)：

| split      |   rows  |
|:-----------|--------:|
| train      |     140 |
| validation |     30  |
| test       |     30  |

## 4. Results

尚未執行實驗，不預先填入模型分數。

| method | dataset_information | Recall@5 | Precision@5 | Threshold Precision | Runtime | Complexity | Note |
| --- | --- | ---: | ---: | ---: | ---: | --- | --- |
| TF-IDF Cosine | content + marketing_content | 待實測 | 待實測 | 待實測 | 待實測 | Low | Keyword overlap baseline |
| Rule-based Similarity | content + marketing_content | 待實測 | 待實測 | 待實測 | 待實測 | Low | Explainable baseline |
| MiniLM Embedding | content + marketing_content | 待實測 | 待實測 | 待實測 | 待實測 | Low-Mid | Week 2 planned pretrained baseline |


實驗結果需要回答：

- MiniLM 是否達到 Recall@5 ≥ 85%？
- Precision 是否達到 ≥ 75%？
- MiniLM 的提升是否發生在 TF-IDF 無法處理的同義詞或語意表達？
- 是否出現 score 很高但人工判定不相關的案例？
- `0.35` 發送門檻是否能有效控制錯誤發送？
- `0.25 ~ 0.30` 是否適合作為人工審查區間？

如果預訓練模型沒有超過簡單 baseline，就沒有足夠證據支持增加模型複雜度。

## 5. Failure Analysis

實驗後至少抽取代表性的 false positive、false negative 與 threshold failure cases。

| method | case_id | forum_content | marketing_content | score | ground_truth | prediction | failure_type | possible_cause | decision_impact |
| --- | --- | --- | --- | ---: | --- | --- | --- | --- | --- |
| 待填 | 待填 | 待填 | 待填 | 待填 | Relevant / Not Relevant | 待填 | 高分誤判 / 低分漏判 | 待分析 | 高 / 中 / 低 |

預計重點分析

1. 高分但實際不相關

可能原因：兩篇文章使用大量相同關鍵字但實際討論目的不同；行銷文章使用泛用詞；論壇文章提到產品名稱但實際是在抱怨；或模型對特定領域詞彙理解不足。此類錯誤會直接影響行銷發送，屬於高風險 failure case。

2. 低分但實際相關

可能原因：兩篇文章使用不同詞彙描述相同需求、使用同義詞或口語、論壇文章過短缺少上下文，或兩者文字風格差異過大。此類錯誤會降低 Recall@5。

3. Score 接近門檻

若 score 落在 `0.25 ~ 0.30`，不直接決定，而交由人工雙重確認。

## 6. Minimum Sufficient Solution

目前的 minimum sufficient solution 候選為：

```text
預訓練 multilingual sentence embedding
+
固定 similarity scoring
+
0.35 threshold
+
0.25 ~ 0.30 模糊門檻人工判定
```

最終是否採用 `paraphrase-multilingual-MiniLM-L12-v2`，必須以實驗結果決定。

若 MiniLM 達到 Recall@5 ≥ 85%、Precision ≥ 75%，且 threshold false-positive rate ≤ 15%，並沒有明顯極端分數，則可暫時視為 minimum sufficient solution。

若未達門檻，優先分析資料品質、label、模型能力與 threshold，而不是直接增加模型複雜度。

## 7. Solution Choice Note

Current evidence

已決定以 `paraphrase-multilingual-MiniLM-L12-v2` 作為預訓練語意模型 baseline。需透過 TF-IDF、rule-based 與 MiniLM 的公平比較，確認預訓練模型是否提供增量 evidence。

Remaining failure

即使 MiniLM 的平均 Recall / Precision 達標，仍需要檢查高分誤判、低分漏判、模糊門檻案例、極端 score 集中、灌水文、廣告文。

Minimum solution decision

若 MiniLM 達成 Recall@5 ≥ 85%、Precision ≥ 75%，且人工抽樣的不相關比例不超過 15%，則目前不需要加入更複雜模型。

Justification

確認模型判定 score 是否真的與人工判定的相關性一致，若差距過大再考慮進行模型fine-tuning。

## 8. Reproducibility / Run Record

- Notebook：`notebooks/week04_baseline.ipynb`
- Dataset：`crawl/data.json`
- Dataset source：論壇爬蟲資料 + 公司行銷文章
- Evaluation labels：人工相關性標記
- Random seed：42
- Baseline 1：TF-IDF + Cosine Similarity
- Baseline 2：Keyword / Rule-based Similarity
- Candidate：`paraphrase-multilingual-MiniLM-L12-v2`
- Output artifacts：`outputs/week04_results.csv`、`outputs/week04_failure_cases.csv`、`outputs/week04_score_distribution.csv`
- Python packages：`pandas`、`scikit-learn`、`sentence-transformers`
- Commit message suggestion：`Complete Week 4 semantic matching baseline report`

## 9. Mini Defense

### Q1. 為什麼你的 Baseline 是合理的？以及 Baseline 的出處

首先需要確認簡單方法與預訓練語意模型各自能達到什麼程度，也就是說若簡單文字嵌入語意比對就可以達到不錯的效果，或許就沒有必要提高模型複雜度。TF-IDF + Cosine Similarity 是最基本的文字相似度方法；Keyword / Rule-based Similarity 承接前幾週的文字清洗與人工判斷邏輯；`paraphrase-multilingual-MiniLM-L12-v2` 則承接 Week 2 原先規劃的多語言預訓練語意模型 baseline。三者使用相同資料、相同人工標記與相同 evaluation set，因此可以公平比較。

### Q2. 複雜模型增加的效益主要集中在哪些案例？

若 MiniLM 比 TF-IDF / rule-based 表現更好，預期效益應集中在簡單關鍵字方法難以處理的案例，例如使用不同詞彙但表達相同需求、同義詞或不同語法、沒有直接出現產品名稱但表達相似需求，以及需要理解長文本整體語意的案例。因此不能只看整體 Recall@5，而需要檢查模型是否真的改善這些 failure cases。

### Q3. 如果複雜模型只提升 1–2%，你還會選它嗎？為什麼？

不一定。如果只提升 1～2% 的平均 Recall 或 Precision，卻增加執行、維護與監控成本，而且沒有改善重要錯誤案例，就沒有足夠理由選用複雜模型。但如果這 1～2% 集中在高風險案例，例如降低高分誤判、提升 Recall@5 或降低 threshold 下的錯誤發送比例，即使整體提升不大，也可能具有實際價值。
