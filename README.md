# AOCCQA-case-exporter

把「前一個 agent（測試案例產生器）產出的測試案例」＋「一張 Jira 單」，套進 **AOCC QA 官方 xlsx 模板**並匯出成 Excel 交付檔。

> **定位：確定性的格式化 / 套版工具。** 只搬運與套版，**不判斷、不篩選、不改寫、不重排序**案例內容。前一個 agent 產出什麼，就原樣填進 xlsx。

---

## 用途

當使用者要做以下任何一件事時，就用這個 skill（即使沒講出 `case-exporter` 這個字）：

- 「把測試案例匯出成 xlsx」
- 「套進 AOCC 模板」
- 「接上一個 agent 的案例輸出做成 Excel」
- 「產出 Test Case 檔 / 把案例清單存成 Test_Case 檔」
- 提供「Jira 單＋測試案例」要打包成交付檔

不屬於任何角色，Phase B/C/D 任何階段都可直接呼叫，與 `AOCCQA-decision-archiver` 無關聯。

---

## 使用條件（前置需求）

### 1. 兩份輸入資料

| 輸入 | 用途 | 需要的欄位 |
|---|---|---|
| **① Jira 單** | 填 Report 分頁 + 產生檔名 | Project name、Assignee、Summary、link（feature/release note）、MCC#、Description（抓「測試時程」與「測試環境」段落） |
| **② 前一個 agent 的測試案例輸出** | 填 Test case 分頁 | 7 欄標準格式：`Test Case ID, Category, Feature, Pre-condition, Test Case, Steps, Expected Result` |

### 2. 必須先有的 MCP

- **ASUS 內部 Jira MCP（`AOCCPM_jira-mcp`，ec-service.asus.com）** — 用來讀取 Jira 單的欄位。
  這是取得「輸入 ①」的來源，必須先在 Claude 完成該 connector 授權才能自動抓單。
  > ⚠️ ASUS 內部 Jira 一律走此 connector，**不要**改用 Atlassian 官方 connector、Atlassian Rovo 或其他第三方工具。
  > 若不透過 MCP，也可由使用者手動把上述 Jira 欄位貼進來，改以人工填 `input.json`。

> 註：`export_test_cases.py` 本身**不呼叫任何 MCP**，只吃 `input.json`。MCP 只出現在「抓 Jira 單」這一步的資料取得階段。

### 3. 執行環境

- Python 3
- 套件：`openpyxl`
  ```bash
  pip install openpyxl
  ```
- 官方模板檔：`assets/Test_Case_Template_Claude.xlsx`（已含在 `.skill` 包內；模板缺失時腳本會停止並回報，不自建替代模板）。

---

## 檔案結構

```
aoccqa-case-exporter.skill        # 打包檔，內含下列三者
  ├─ SKILL.md                     # skill 規格與契約
  ├─ assets/Test_Case_Template_Claude.xlsx   # AOCC QA 官方模板
  └─ scripts/export_test_cases.py # 匯出腳本
aoccqa-case-exporter_SKILL.md     # SKILL.md 的單獨副本（方便閱讀）
export_test_cases.py              # 腳本的單獨副本
```

---

## 運作方式

1. 讀 Jira 單，抽出 Report 所需欄位與 Summary（透過 Jira MCP 或人工提供）。
2. 讀前一個 agent 的測試案例輸出，解析成逐筆的 7 欄結構。
3. 整理成 `input.json`。
4. 執行腳本：
   ```bash
   python scripts/export_test_cases.py \
     --template assets/Test_Case_Template_Claude.xlsx \
     --input input.json \
     --outdir <輸出資料夾>
   ```
5. 腳本套模板 → 填 Report → 填 Test case → 依規則產生檔名 → 存檔，並印出結果（檔名、案例筆數、Report 哪些格留白）。

### 對應規則（已鎖定）

- **Report 分頁 ← Jira**：Summary→C2、測試時程→C3（正規化為 `YYYY/MM/DD-YYYY/MM/DD`）、Tester→C5（`AOCCQA_{Assignee}`）、link→C6、MCC#→C13、測試環境→C14。
- **Test case 分頁 ← agent 輸出**：ID→A、Category→E、Pre-condition→F、Test Case→G、Steps→H、Expected Result→I。**`Feature` 欄一律捨棄**。
- 執行欄位（平台 B/C/D、Test result J、Note K、Test Data L）**不填**。
- Pass/Fail/N-A 統計與比率、Bug list、Screenshot 分頁與所有公式**原樣保留，不動**。

### 檔名規則

```
{Summary處理後}_Test Case_{YYYYMMDD}.xlsx
```
- 移除 `[UAT-QA]` 標籤 → 開頭市場標籤 `[XX] ` 轉為 `XX_` → 接上 `_Test Case_{匯出當天}`。
- 非法字元（`/ \ : * ? " < > |`）以底線取代。

---

## 邊界與回報

| 情況 | 行為 |
|---|---|
| Description 抓不到測試時程或環境 | 該格留白，並明確告知使用者哪一格沒抓到，**不臆測填值** |
| 案例 0 筆 | 停止並回報，不產空檔 |
| 案例 > 200 筆 | 停止並回報（模板公式範圍上限） |
| 模板檔缺失 | 停止並回報，不自建替代模板 |

### 待確認（Pending）

- **Tester 前綴**：使用者口頭指定 `AOCCQA_{Assignee}`，但參考檔顯示 `AOCC_{Assignee}`。目前腳本採 `AOCCQA_`，待確認後再定。
