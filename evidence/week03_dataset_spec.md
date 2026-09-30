# Week 3 - Dataset Spec v0.2

學號：7114029034
姓名：范睿凱
專題名稱：論壇文章與行銷文章語意比對系統

## A. 資料怎麼形成

### 1. Source / Owner

資料主要由兩部分組成：

1. 論壇網站公開文章

   - 由爬蟲從目標論壇網站取得公開文章。
   - 主要取得文章標題、網址、作者、內文與張貼日期等資訊。
   - 資料用途為找出可能對行銷內容有興趣的潛在對象。

2. 公司自有行銷文章 / 行銷郵件

   - 由行銷公司提供作為語意比對的目標文本。
   - 每一篇行銷文章代表一個行銷主題或行銷內容。

### 2. Unit Alignment（承接 W2）

W2 原先定義一筆資料為一篇論壇文章，包含：

- `title`
- `url`
- `author`
- `content`
- `post_date`
- `score`

根據爬取到的content與行銷文章作語意比對，得到`score`欄位，例如：

```text
論壇文章 A × 行銷文章 1 → score = 0.82
論壇文章 A × 行銷文章 2 → score = 0.31
論壇文章 A × 行銷文章 3 → score = 0.67
```

模型會針對同一篇論壇文章與多篇行銷文章進行語意比對，再依照 `score` 排序。

### 3. Time Range

論壇文章資料以爬蟲實際執行期間取得的資料為主，並記錄每篇文章的 `post_date`。

通常會將爬蟲進行排程，在每天固定時間取得最新文章

行銷文章則記錄其建立或使用時間，例如：

```text
marketing_article_id
marketing_article_date
```

### 4. Fields

#### 論壇文章欄位

| Field        | Type     | 說明        | 是否輸入模型 |
| ------------ | -------- | --------- | ------ |
| `article_id` | String   | 論壇文章唯一識別碼 | 否      |
| `title`      | String   | 論壇文章標題    | 否，為了擔心釣魚標題  |
| `url`        | String   | 原始文章網址    | 否      |
| `author`     | String   | 文章作者      | 否      |
| `content`    | String   | 文章內文      | 是      |
| `post_date`  | Datetime | 文章發布日期    | 否      |
| `crawl_date` | Datetime | 爬蟲取得日期    | 否      |


#### 行銷文章欄位

| Field               | Type     | 說明        | 是否輸入模型 |
| ------------------- | -------- | --------- | ------ |
| `marketing_id`      | String   | 行銷文章唯一識別碼 | 否      |
| `marketing_content` | String   | 行銷文章內容    | 是      |
| `marketing_date`    | Datetime | 行銷文章建立日期  | 否      |

#### 模型與人工評估欄位

| Field             | Type    | 說明           |
| ----------------- | ------- | ------------ |
| `score`           | Float   | 模型產生的語意相似度分數 |
| `relevance_label` | Integer | 人工判定是否相關     |
| `reviewer_id`     | String  | 標記者識別碼       |
| `review_reason`   | String  | 人工判定原因，可選填   |

其中：

```text
relevance_label = 1 → 相關
relevance_label = 0 → 不相關
```

`score` 為模型輸出，不能直接當作人工真實答案（ground truth）。

### 5. Label / Codebook

本資料集的主要人工標籤為 `relevance_label`。

| Label | 定義                                     |
| ----- | -------------------------------------- |
| `1`   | 論壇文章內容與行銷文章主題具有實質相關性，該文章有與行銷內容語意高度相似的合理性 |
| `0`   | 論壇文章與行銷文章沒有足夠的主題或相關性 |

人工標記時，不應單純依照兩篇文章是否出現相同關鍵字判斷。

為避免人工判斷過於主觀，應建立簡單的人工標記規則，並由人工檢查部分資料。

預計隨機抽取至少 50 篇論壇文章進行人工相關性標記，作為實際業務場景的評估資料。

### 6. Inclusion / Exclusion

#### Inclusion

納入資料需符合：

1. 論壇文章具有可取得的公開內容。
2. `content` 不為空值。
3. 文章內容具有足夠文字可進行語意分析。
4. 行銷文章具有完整的文字內容。
5. 文章可以確認其來源與取得時間。

#### Exclusion

以下資料不進入模型：

1. 少於 30 字的灌水文章。
2. 廣告文章。
3. 主要由表情符號或無意義符號組成的文章。
4. 內容為空或爬蟲取得失敗的文章。
5. 重複的文章。
6. 無法確認內容來源的異常資料。


## B. 證據怎麼驗證

### 7. Population / Coverage

Population 定義為目標論壇網站可以公開取得、且符合資料納入條件的文章。

目前 Dataset 並不代表整個網路上的所有文章

Coverage 主要需要檢查：

- 是否涵蓋到多使用者的文章。
- 是否某些行銷主題完全沒有對應資料。

### 8. Train / Validation / Test Rule

由於本任務的主要目標是語意比對，人工標記資料應盡可能保留不同主題與相關/不相關案例，讓模型學到更廣泛資料。

### 9. Leakage Risks

本資料集主要存在以下資料洩漏風險：

#### Risk 1：同一篇文章重複出現

如果相同論壇文章被不同人複製、因為各種原因未被去重而被分到 Train 與 Test，模型可能看過非常相似的內容。

#### Risk 2：時間洩漏

如果使用未來建立的行銷文章或未來資料資訊去判斷過去文章，可能造成不符合實際部署情境的結果。

因此應保留 `post_date` 與 `marketing_date`。

#### Risk 3：人工標記資訊進入模型

人工標記資料不可以作為模型輸入。

`score` 是模型輸出，也不能在訓練或測試時被當成真實標籤。

### 10. Quality Issues

本資料集預計檢查以下品質問題：

| Check                | 檢查內容                             |
| -------------------- | -------------------------------- |
| Missing Values       | `content`、`url`、`post_date` 是否缺失 |
| Duplicate            | 是否存在完全相同文章                       |
| Duplicate Content    | 是否存在相同內文                         |
| Short Text           | 是否存在少於 30 字文章                    |
| Spam / Advertisement | 是否存在灌水或廣告內容                      |
| Text Length          | 文章長度是否極端異常                       |
| Label Distribution   | 相關與不相關資料是否嚴重不平衡                  |
| Score Distribution   | 模型分數是否大量集中於 0 或 1                |

### 11. Version / Provenance

Dataset 保留版本與資料來源資訊。

例如：

```text
Dataset Version: v0.2

Source:
- Forum crawler
- Company marketing articles

Collection Date:
- 2026-09 ～ 2026-10

Processing:
1. Web crawling
2. Text cleaning
3. Duplicate removal
4. Human labeling
5. Train / Validation / Test split
6. Model evaluation
```

每次修改資料清洗規則、增加新的論壇來源或重新標記資料時，都應建立新的 Dataset version。

例如：

```text
v0.1 → 原始爬蟲資料
v0.2 → 加入文字清洗與人工標記
v0.3 → 修正人工標記規則
```

如此可以追蹤模型結果是使用哪一版本資料產生。

### 12. Known Limitations

目前資料集有以下限制：

1. 論壇來源可能有限，無法代表所有論壇或社群網站。
2. 人工相關性標記可能具有主觀性。
3. 目前預計的 50 篇人工標記資料規模仍較小。
4. 不同論壇的文字風格可能造成模型表現差異。
5. 行銷文章與論壇文章的文本長度可能存在明顯差異。
6. 使用預訓練語意模型的 Benchmark 成績，不一定能完全代表實際論壇文章與行銷文章的比對效果。
7. `score` 是模型產生的相似度分數，不能直接視為真實的「購買意願」或「一定會接受行銷」。
8. 即使 Precision 與 Recall 達到設定門檻，也不代表模型可以完全取代人工審查。

---

## C. Dataset Audit Summary

### Quality Finding

目前 Dataset Audit 應至少檢查：

- Rows / Columns
- Missing Values
- Exact Duplicates
- Duplicate Content
- Text Length
- Label Distribution
- Score Distribution

### Leakage / Split Risk

本專題最大的洩漏風險是同一篇論壇文章或高度相似文章同時進入 Train 與 Test。

因此需要確認：

```text
article_id
url
content
post_date
```

另外，`relevance_label`、人工審查結果以及模型產生的 `score` 都不能被當成模型輸入。

### Label / Coverage Finding

本專題與一般分類問題不同，真正需要評估的是：

```text
論壇文章
      ↓
與多篇行銷文章進行語意比對
      ↓
產生 score
      ↓
依 score 排序
      ↓
選出可能相關的行銷對象
```

人工標記的 `relevance_label`主要用於模型評估

當模型分數大量集中於接近 0 或 1 時，需要觸發人工檢查，以確認是否存在模型失效、論壇內容異常、干擾詞或行銷策略與資料來源不符合等問題。

### Dataset Audit Conclusion

W3 確認：

```text
這些資料從哪裡來？
→ 資料是否乾淨？
→ 標籤怎麼產生？
→ Train/Test 是否洩漏？
→ 資料是否能代表實際論壇？
→ 最後是否能支持 Recall@5 與 Precision 的業務評估？
```