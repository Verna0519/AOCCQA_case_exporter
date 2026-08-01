---
name: aoccqa-case-exporter
metadata:
  version: 1.0.0
description: >
  共用工具（人工指令觸發、Agent 執行）。把前一個 agent（測試案例產生器）產出的
  測試案例，連同一張 Jira 單，套進 AOCC QA 官方 xlsx 模板並匯出。只整理輸出、
  不做內容判斷或篩選。凡是使用者要「匯出測試案例成 xlsx」「套進 AOCC 模板」
  「接上一個 agent 的案例輸出做成 Excel」「產出 Test Case 檔」「把案例清單存成
  Test_Case 檔」，或提供 Jira 單＋測試案例要打包成交付檔時，都應觸發此 skill，
  即使沒說出 "case-exporter" 這個字。不屬於任何角色，Phase B/C/D 任何階段都可
  直接呼叫，與 AOCCQA-decision-archiver 無關聯。
---

# AOCCQA-case-exporter

把「前一個 agent 的測試案例輸出」+「一張 Jira 單」→ 套進官方模板 →
匯出成 xlsx。**這是確定性的格式化工具，只搬運與套版，不判斷、不篩選、不改寫案例內容。**

## 契約（不可違反）

1. **只整理輸出**：前一個 agent 產出什麼案例，就原樣填進 xlsx，不增刪、不改寫、不過濾、不重排序。
2. **欄位固定**：Test case 分頁沿用模板既有 12 欄，順序與命名不變。
3. **公式不動**：Report 的 Pass/Fail/N-A 統計與比率、Bug list 的嚴重度統計，全部保留原公式。
4. **Bug list / Screenshot 分頁**：原樣保留，不更動任何內容或格式。
5. 呼叫時機不限；與 decision-archiver 無關聯。

## 輸入

需要兩份資料：

**① Jira 單**（用於 Report 分頁與檔名）。需能取得下列欄位：
- Project name
- Assignee
- Summary（標題）
- link（feature / release note 連結）
- MCC#
- Description 內文（用於抓「測試時程」與「測試環境」段落）

**② 前一個 agent 的測試案例輸出**（7 欄標準格式）：
`Test Case ID, Category, Feature, Pre-condition, Test Case, Steps, Expected Result`

## 對應規則（已鎖定）

### Report 分頁 ← Jira

| Report 欄位（標籤格 → 寫入格） | 來源 |
|---|---|
| Project（A2 → **C2**） | **Jira Summary 完整標題（含 `[UAT-QA][XX]` 標籤，原樣填入）** |
| Test date（A3 → **C3**） | Jira Description 內時程，正規化為乾淨區間 **`YYYY/MM/DD-YYYY/MM/DD`**（取整體最早起日～最晚迄日，不帶 Internal Testing/UAT 字樣） |
| Test Version（A4 → **C4**） | 暫留白（未指定；日後要帶再補） |
| Tester（A5 → **C5**） | `AOCCQA_{Assignee}`（取 Jira Assignee；覆蓋模板預填值）— **註：待與 AOCC_ 前綴確認，見下方 Pending** |
| New feature & Release Note（A6 → **C6**） | Jira link（如 `.../jira/browse/DV2IN1-44637`） |
| Test Country（A13 → **C13**） | Jira MCC#（如 `MCC1 (BE-NL/IT/PL/CZ)`） |
| Test Environment（A14 → **C14**） | Jira Description 內文的「測試環境」段落（如 `MCC1 Stage`） |

> **Pending 待確認**：Tester 前綴。使用者口頭指定 `AOCCQA_{Assignee}`，但參考檔顯示 `AOCC_{Assignee}`（模板預填值）。目前腳本採 `AOCCQA_`，確認後再定。

- Pass / Fail / N-A 統計（L2:L4）與比率（P2:P4）：**保留公式，不填值**。
- `Device`（C11 預填 PC / Mobile）、`Browser`（C12 預填 Chrome / Edge）：**保留預填，不動**。
- Description 為半結構化內文，時程/環境段落抓取採「關鍵字定位」；抓不到時該格留白，並在回覆中告知使用者哪一格沒抓到，**不要臆測填值**。

### Test case 分頁 ← 前一個 agent 輸出

資料列從第 2 列起（模板已備到第 201 列，最多 200 筆）。逐筆對應：

| 前一個 agent 欄位 | 寫入模板欄 |
|---|---|
| Test Case ID | A（ID） |
| Category | E（Category） |
| Pre-condition | F（Pre-condition） |
| Test Case | G（Test case） |
| Steps | H（Steps，逐步保留換行） |
| Expected Result | I（Expected result） |
| **Feature** | **捨棄，不寫入任何欄** |

- 執行時欄位 **不填**：B（PC）、C（Mobile）、D（Tablet）、J（Test result）、K（Note）、L（Test Data）。
- 案例數 > 200 時停止並回報使用者，不覆蓋公式範圍。

## 檔名規則（已鎖定）

```
{Summary處理後}_Test Case_{YYYYMMDD}.xlsx
```

Summary → 檔名處理步驟:
1. 移除 `[UAT-QA]` 標籤(不分大小寫)。
2. 開頭剩餘的市場標籤 `[XX] ` 轉為 `XX_`(去中括號、其後空白換底線)。
3. 接上 `_Test Case_{YYYYMMDD}`,`YYYYMMDD` = 匯出當天。

範例:
`[UAT-QA][EU] Customized Bundle maintenance mechanism enhancement`
→ `EU_Customized Bundle maintenance mechanism enhancement_Test Case_20260722.xlsx`
- 檔名不含狀態字樣（依使用者提供之範例為準）。若日後要加「定稿版／草稿版／進度快照」再調整。
- 檔名含非法字元（`/ \\ : * ? " < > |`）時以底線取代，避免存檔失敗。

## 執行步驟

1. 讀 Jira 單，抽出上表 Report 所需欄位與 Summary。
2. 讀前一個 agent 的測試案例輸出，解析成逐筆的 7 欄結構。
3. 把資料整理成 `input.json`（見下方結構），交給腳本。
4. 執行：
   ```bash
   python scripts/export_test_cases.py \
     --template assets/Test_Case_Template_Claude.xlsx \
     --input input.json \
     --outdir /mnt/user-data/outputs
   ```
5. 腳本會套模板、填 Report、填 Test case、依規則產生檔名、存到 outputs，並印出最終路徑。
6. 用 `present_files` 把產出檔交給使用者，並回報：檔名、案例筆數、Report 哪些格有填/留白。

### input.json 結構

```json
{
  "jira": {
    "project": "EU",
    "assignee": "VernaChen",
    "summary": "[UAT-QA] EU_Customized Bundle maintenance mechanism enhancement",
    "link": "https://jira.example.com/browse/XXXX-1234",
    "mcc": "123",
    "test_date": "Internal Testing: 2026/07/20 ~ 2026/07/24",
    "test_environment": "Staging / EU PROD-like",
    "test_version": ""
  },
  "test_cases": [
    {
      "id": "1",
      "category": "Cart",
      "pre_condition": "(1) No special pre-condition",
      "test_case": "Verify bundle discount applies correctly",
      "steps": "1. Add bundle to cart\n2. Open cart",
      "expected_result": "1. Bundle added\n2. Discount shown correctly"
    }
  ]
}
```

- `id` 沿用來源流水號；未提供時腳本自動由 1 順編。
- `test_date` / `test_environment` 抓不到時給空字串，腳本留白並回報。
- `feature` 欄即使前一個 agent 有給，也一律不放進 input（或放了也會被忽略）。

## 邊界與回報

- Description 抓不到時程或環境 → 該格留白 + 明確告知。
- 案例 0 筆 → 停止並回報，不產空檔。
- 案例 > 200 筆 → 停止並回報（模板公式範圍上限）。
- 模板檔缺失 → 停止並回報，不自建替代模板。
