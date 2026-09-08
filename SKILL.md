---
name: prepare-flow-skill
description: 在使用者發起 Guru-X BPM 流程前，執行完整的前置準備動作，依序完成認證、流程定位、取得 FormId 與資料表資訊、解析表單欄位、取得使用者組織資料、整合外部資料（如 OCR）、產生 formData 並儲存流程草稿。
---

# Prepare Flow Skill

## Overview

本 skill 用於在使用者發起 Guru-X BPM 流程「之前」，完成從認證、流程定位、
欄位解析、資料整合到「草稿儲存」的完整前置準備工作。

本 skill 依賴：
- `bpm-api-management`：取得 AccessToken
- `flow-guide`：查詢流程庫、定位流程 ID
- `ocr-skill`（選用）：若使用者上傳單據，需先透過 OCR 取出結構化資料

## 動作流程

1. **取得認證 Token**：呼叫 `bpm-api-management` 完成 SSO 登入取得 `AccessToken`。
2. **定位流程 Process ID**：透過 `flow-guide` 查詢流程庫目錄樹與各目錄下的流程列表，依使用者指定的流程名稱找出對應的 `ProcessId`。若在預設目錄找不到，須歷遍所有目錄。
3. **取得流程對應的 Form ID 與 Table 資訊**：透過 `process-form-tableIdentitys` API 取回 `FormId` 及 `GlobalTables` 資訊。
4. **取得表單定義並解析欄位**：呼叫 `form/{FORM_ID}/definition`，從 `Src` 中萃取所有 `$bind` 欄位名稱，整理為需填寫欄位清單（含 `ctype`、`fieldLabel`、`allowBlank`、`lockBinding` 等屬性）。
5. **取得使用者組織資料**：呼叫 `org/bpmou/user/{account}/positions`，只取 `UserDefaultRole: true` 的記錄，取得 `Id` 與 `OUID`。
6. **整合外部資料（如 OCR）**：若有單據，透過 `ocr-skill` 取得 `date`、`total_amount`、`items` 等結構化資料。
7. **組出 formData 字串**：依表單 `$bind` 欄位，將使用者資料、OCR 結果、使用者輸入映射為單一 JSON 字串（`formData` 屬性值為 escape 過的 JSON 字串）。
8. **儲存流程草稿**：呼叫 `bpm/workflow/process/{PROCESS_ID}/draft/save`，將 `header` 與 `formData` 送出，取回 Draft `Id`。
9. **輸出完整結果**：以標準 JSON 結構回傳。

## API Endpoints

BASE_URL=https://gaiaix-poc.metaguruai.com:15101

### 1. 取得流程 Form Id 與 Table 資訊
```
GET {BASE_URL}/api/processadmin/{PROCESS_ID}/process-form-tableIdentitys?prefixTableName=true
```

**必要 Headers：**
```
Authorization: Bearer {ACCESS_TOKEN}
accept: text/plain
```

**回應結構：**
```json
{
  "Success": true,
  "Data": {
    "FormId": "839641089945669",
    "GlobalTables": [
      {
        "DataSourceName": "Default",
        "IsRepeatableTable": false,
        "TableName": "BDS_費用申請單",
        "DBTableName": "BDS_費用申請單"
      }
    ]
  }
}
```

### 2. 取得表單定義
```
GET {BASE_URL}/api/form/{FORM_ID}/definition
```

**解析要點：**
- 從回應 `Data.Src` 字串中搜尋所有 `"$bind": "xxx"` 對應的欄位。
- 需保留：`ctype`（欄位類型）、`fieldLabel`（顯示名稱）、`allowBlank`（是否必填）、`lockBinding`（是否鎖定）等屬性。
- `lockBinding: true` 的欄位通常為系統自動帶入，不需要使用者輸入。

### 3. 取得使用者組織資料
```
GET {BASE_URL}/api/org/bpmou/user/{account}/positions
```

**篩選規則：**
- 只取 `UserDefaultRole: true` 的記錄。
- 取得 `Id`（角色 ID）與 `OUID`（組織 ID），用於後續 `Department`、`Applicant` 欄位。

### 4. 儲存流程草稿
```
POST {BASE_URL}/api/bpm/workflow/process/{PROCESS_ID}/draft/save
```

**必要 Headers：**
```
Authorization: Bearer {ACCESS_TOKEN}
Content-Type: application/json
accept: text/plain
```

**Request Body：**
```json
{
  "header": {
    "comments": "",
    "ouid": "bpmou.{OUID}",
    "ownerPosition": "bpmou.{ID}"
  },
  "formData": "{\"FormNo\":null,\"Department\":\"bpmou.{OUID}\",\"Applicant\":\"{UID}\",\"ApplyDate\":\"\",\"Amount\":0,\"Description\":\"\"}"
}
```

**注意：** `formData` 必須為 escape 過的 JSON 字串，非物件。

**回應結構：**
```json
{
  "Success": true,
  "Code": 200,
  "Data": {
    "Id": 839874835624005,
    "DraftType": "Draft"
  }
}
```

## 資料映射規則

| 目標欄位 (`$bind`) | 來源 | 值格式 |
|---|---|---|
| `FormNo` | 系統產生 | `null` |
| `Department` | 使用者組織 | `bpmou.{OUID}` |
| `Applicant` | 使用者角色 | `bpmou.{Id}` |
| `ApplyDate` | OCR `date` 或系統日期 | `YYYY-MM-DD` |
| `Amount` | OCR `total_amount` | 數字 |
| `Description` | 使用者輸入 | 字串 |

> 實際欄位需依步驟 4 解析出的 `$bind` 清單動態決定，非固定為上述欄位。

## 使用範例（完整流程）

```bash
BASE_URL="https://gaiaix-poc.metaguruai.com:15101"
TOKEN="{ACCESS_TOKEN}"
PROCESS_ID="839641510686789"
FORM_ID="839641089945669"
USER_ACCOUNT="prliu"

# 步驟 3：取得 FormId 與資料表資訊
curl -X GET \
  "{BASE_URL}/api/processadmin/${PROCESS_ID}/process-form-tableIdentitys?prefixTableName=true" \
  -H "Authorization: Bearer ${TOKEN}"

# 步驟 4：取得表單定義並解析 $bind
curl -X GET \
  "{BASE_URL}/api/form/${FORM_ID}/definition" \
  -H "Authorization: Bearer ${TOKEN}"

# 步驟 5：取得使用者組織資料（僅取 UserDefaultRole=true）
curl -X GET \
  "{BASE_URL}/api/org/bpmou/user/${USER_ACCOUNT}/positions" \
  -H "Authorization: Bearer ${TOKEN}"

# 步驟 8：儲存流程草稿
curl -X POST \
  "{BASE_URL}/api/bpm/workflow/process/${PROCESS_ID}/draft/save" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "header": {
      "comments": "",
      "ouid": "bpmou.821457851600965",
      "ownerPosition": "bpmou.810161144467525"
    },
    "formData": "{\"FormNo\":null,\"Department\":\"bpmou.821457851600965\",\"Applicant\":\"bpmou.810161144467525\",\"ApplyDate\":\"2015-12-10\",\"Amount\":210,\"Description\":\"\"}"
  }'
```

## Output Format

完整前置準備完成後，以下列 JSON 結構回傳：

```json
{
  "processName": "費用申請單",
  "processId": "839641510686789",
  "formId": "839641089945669",
  "tables": [
    {
      "dataSourceName": "Default",
      "tableName": "BDS_費用申請單",
      "dbTableName": "BDS_費用申請單",
      "isRepeatableTable": false
    }
  ],
  "fields": [
    { "bind": "FormNo", "label": "表單編號", "ctype": "sn", "lockBinding": true },
    { "bind": "Department", "label": "申請部門", "ctype": "dept", "lockBinding": true },
    { "bind": "Applicant", "label": "申請人", "ctype": "initiator", "lockBinding": true },
    { "bind": "ApplyDate", "label": "申請日期", "ctype": "starttime", "lockBinding": true },
    { "bind": "Amount", "label": "申請金額", "ctype": "number", "allowBlank": false },
    { "bind": "Description", "label": "費用說明", "ctype": "textarea", "allowBlank": true }
  ],
  "user": {
    "account": "prliu",
    "id": 810161144467525,
    "ouid": 821457851600965,
    "ouName": "產品管理部"
  },
  "formData": "{\"FormNo\":null,\"Department\":\"bpmou.821457851600965\",\"Applicant\":\"bpmou.810161144467525\",\"ApplyDate\":\"2015-12-10\",\"Amount\":210,\"Description\":\"\"}",
  "draft": {
    "id": 839874835624005,
    "draftType": "Draft"
  },
  "initiationUrl": "https://gaiaix-poc.metaguruai.com:15102/forms/post?processId=839641510686789"
}
```

## 注意事項

1. AccessToken 具時效性（預設 3600 秒），若過期需重新取得。
2. 若透過流程名稱找不到對應流程，須歷遍所有目錄後再回報找不到。
3. 若流程存在多個資料表（如包含重複性子表），須完整回傳 `GlobalTables` 內所有項目。
4. 使用者位置資料須嚴格篩選 `UserDefaultRole: true`，避免使用非預設角色。
5. `formData` 欄位在 `draft/save` API 中必須為 escape 過的 JSON 字串，非物件；`Amount` 為數字型別，`FormNo` 通常為 `null`。
6. `header.ouid` 與 `header.ownerPosition` 需帶前綴 `bpmou.`。
7. OCR 結果若無法辨識明細或關鍵欄位，須標註低置信度或請求人工確認，不得捏造資料。
8. 所有輸出必須為合法 JSON，不得含猜測性資料。
