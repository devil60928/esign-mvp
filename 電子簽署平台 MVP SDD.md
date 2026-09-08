# 電子文件簽署平台 — MVP 系統設計文件（SDD）
> 跨職能團隊分析：PM｜SA｜SD｜PG｜QA｜Security Architect｜UX/UI Designer
> 版本：v0.1（Draft for Review）｜ 基準文件：派遣工作單、安全衛生教育訓練確認單、個資蒐集同意書

---

## 0. 現有文件欄位分析（Field Discovery）

以三份範例文件實際盤點，並依需求分類 A~F。這是「文件欄位模型」的資料來源，**目的不是描述這三份文件，而是驗證欄位模型是否具備擴充性**。

| 分類 | 欄位範例（來源文件） | 備註 |
|---|---|---|
| A. 基本資料 | 姓名、身分證字號/居留證號碼、國籍、出生年月日、聯絡電話、通訊地址（派遣工作單、教育訓練單、個資同意書皆重複出現） | **跨文件共用度最高**，是「一次輸入、多文件共用」的核心驗證點 |
| B. 工作資料 | 工作地點/職稱、派遣期間（起訖日）、客戶(要派單位)名稱、部門、職稱、薪資（時薪196元＋加給）、工作時間/休息時間 | 僅派遣工作單使用，屬「文件專屬欄位」 |
| C. Checkbox/選擇 | 月領／週領／隔日領（單選）；教育訓練 7 項＋其他（複選，且**不勾選不影響效力**——這是特殊 Business Rule）；輪班制/固定班別選擇 | 需支援「非強制勾選但仍需完成頁面」的邏輯 |
| D. 聲明/同意 | 「已詳細閱讀並充分了解」「同意蒐集處理利用個資」「願意遵守相關規定」「已完成教育訓練說明」 | 通常與 Checkbox 或簽名綁定，需可個別稽核 |
| E. 簽名 | 派遣工作單簽名＋身分證字號＋電話＋地址（簽名區塊本身還帶欄位！）、教育訓練確認簽名、個資同意書簽名＋日期 | **同一使用者、同一次流程需要多次簽名**，須設計「一次簽名、多處套用」或「逐一簽署但共用同一手寫筆跡」策略 |
| F. 系統產生 | 開立日期、文件編號、講師/單位/日期（教育訓練單由「甲方填寫」非員工）、簽署時間 | 注意：教育訓練單有「員工不得自行填寫」的欄位（講師簽核），代表系統需支援**雙角色欄位權限**（不只是使用者一路填到底） |

**PM 觀察到的關鍵需求（非顯而易見）：**
1. 同一份文件內可能有「使用者輸入」與「他人（主管/講師）輸入」的欄位混合 → Field 需要 `FillerRole` 屬性，不能假設整份文件都是同一人填寫。
2. 「未逐項勾選不影響效力」→ Validation Rule 需支援 `RequiredButNotBlocking`，不是所有 checkbox 都要擋流程。
3. 身分證字號在三份文件重複出現、但個資同意書要求的是「同意蒐集」而非「填寫」→ 同一欄位在不同文件可能是 Display-only 或 Input，需要 Field Mapping 層去 decouple「資料」與「文件呈現」。

---

## 1. Product Vision

**一句話願景：** 打造一個「Template Driven」的企業級電子簽署平台，讓 HR／管理端只需上傳新合約＋定義欄位，不需要工程師修改程式，即可讓員工／派遣人員／外籍人員在手機上以「一直按下一步」的方式完成填寫與簽署，並產出具備法律與稽核效力的 PDF。

**不做什麼（Anti-Goal）：**
- 不是 PDF 檢視器＋填表工具（那是 Adobe Fill & Sign 等級，不是企業簽署平台）。
- 不針對單一文件寫死流程／if-else。
- 第一版不做多方簽署、不做 FIDO/Passkey（留架構位置，不做功能）。

**成功指標（MVP 驗收）：**
- 新增一份新合約（如「保密協議」）時，工程師改動 = 0（只需 Template 設定＋PDF 座標 Mapping）。
- 手機完成率（Started → Signed）作為核心 UX KPI。
- 每一份完成文件都可追出完整 Audit Trail（時間、IP、Device、Hash）。

---

## 2. User Journey（以派遣人員為例）

```
收到簽署通知（Email/簡訊/連結）
  → 進入文件清單（可能同時有 3 份待簽）
    → 點選「派遣工作單」→ 文件說明頁（3-5分鐘、共5頁）
      → 基本資料頁（姓名/身分證等，僅需填一次，其餘2份文件自動帶入）
        → 逐步填寫工作資料（引導式，非整頁PDF）
          → Checkbox / 聲明確認
            → 簽署前確認清單（✓✓✓）
              → 手寫簽名
                → 完成頁（含文件編號、下載PDF）
                  → 返回清單 → 系統自動帶出下一份待簽文件（安全衛生單）
                    → （因基本資料已存在）直接跳到文件專屬欄位
                      → 簽名 → 完成
                        → 個資同意書（同上，最短路徑）
                          → 全部完成 → 清單顯示「已完成」
```

**關鍵 UX 洞察：** 使用者的心理模型不是「簽一份文件」，而是「今天要辦完一件事（入職/派遣手續）」。因此清單頁應顯示「整體進度」（3份中完成1份），而非讓使用者感覺每份文件是獨立任務。

---

## 3. UX Flow（Guided Signing State Machine）

```
[文件清單] → [文件說明] → [基本資料]* → [文件閱讀/確認] → [待完成清單]
   → (迴圈：點選未完成項目 → 自動定位/高亮 → 填寫 → 驗證 → 回到清單)
   → [簽署前確認] → [手寫簽名] → [完成頁]

* 基本資料頁若偵測到 User Profile 已有值，此頁可能整頁 Skip，
  僅在「文件閱讀」頁面用 Read-only 卡片呈現「以下資料將套用於本文件」+ [這不是我/需修改] 連結
```

**待完成清單 UX（核心元件，需求書第六節）：**
- 顯示「待完成 N 項」，非「已填 X/Y」，降低使用者心理負擔。
- 每項可點擊 → Auto-scroll + Highlight + Tooltip → 完成後自動勾選並跳下一項（類似 TypeForm / DocuSign 混合體驗）。

---

## 4. Wireframe（關鍵畫面文字稿）

沿用需求書已提供的線框（基本資料頁、簽名頁），此處補充「待完成清單」與「簽署前確認」兩個未展開的畫面：

**待完成清單**
```
┌──────────────────────────┐
│ 派遣工作單－待完成 3 項    │
│                          │
│ ✓ 姓名 / 身分證字號       │
│ ✓ 聯絡電話 / 地址         │
│ ○ 薪資發放方式  [前往]   │
│ ○ 工作時間      [前往]   │
│ ○ 教育訓練確認  [前往]   │
│                          │
│        [ 全部完成後繼續 ] │
└──────────────────────────┘
```

**簽署前確認**
```
┌──────────────────────────┐
│ 簽署前確認                │
│ 派遣工作單 v1.2 · 共5頁    │
│                          │
│ ✓ 基本資料完成            │
│ ✓ 必填欄位完成            │
│ ✓ 聲明同意已勾選          │
│                          │
│ [ 返回修改 ] [ 前往簽名 ] │
└──────────────────────────┘
```

---

## 5. Screen List

| # | 畫面 | 說明 |
|---|---|---|
| S1 | 文件清單 Dashboard | 顯示待簽/已簽文件、整體進度 |
| S2 | 文件說明頁 | 名稱、頁數、預估時間、CTA |
| S3 | 基本資料頁 | 跨文件共用資料輸入 |
| S4 | 文件閱讀頁 | 分頁瀏覽 PDF 內容（非填寫） |
| S5 | 待完成清單 | 引導剩餘欄位 |
| S6 | 動態欄位填寫頁 | 依 FieldType 渲染的 Modal/Sheet |
| S7 | 簽署前確認頁 | Checklist Summary |
| S8 | 電子簽名頁 | Signature Canvas |
| S9 | 完成頁 | 結果、下載、返回清單 |
| S10 | Resume/Draft 提示 | 「上次完成80%，是否繼續？」 |
| S11（後台） | 管理後台 - Template 管理 | Phase 2 |
| S12（後台） | 稽核紀錄查詢 | Audit 檢視（角色限定） |

---

## 6. Page Navigation

```
Dashboard(S1)
  └─ 選擇文件 → S2 → [首次] S3 → S4 ↔ S5 ↔ S6（可跳頁但不可跳過 Required）
       └─ 全部完成 → S7 → S8 → S9 → 回 S1（自動高亮下一份待簽文件）
       └─ 中途離開 → 自動存 Draft → 下次進入 S1 顯示 Resume 提示（S10）→ 導回離開時的畫面
```
導覽原則：**只能前進到「下一個未完成項」，但可任意回頭修改已完成項**（不可跳過必填，但允許回溯編輯，編輯後重新驗證）。

---

## 7. Field Model（通用欄位定義，非文件專屬）

```json
{
  "fieldId": "FLD-0007",
  "type": "IDField",              // TextField/NumberField/DateField/PhoneField/AddressField/IDField/Select/Radio/Checkbox/Textarea/Signature/Initial/Readonly/SystemField
  "label": "身分證字號／居留證號碼",
  "required": true,
  "editable": true,
  "visibility": "visible",
  "dataSource": "HRSystemAPI",     // UserProfile/HRSystem/API/PreviousDocument/ManualInput/SystemGenerated
  "fillerRole": "Applicant",       // Applicant / Supervisor / Trainer / System（回應0.節的雙角色需求）
  "validation": {
    "regex": "^[A-Z][12]\\d{8}$",
    "maxLength": 10
  },
  "maskRule": "A12****789",         // 顯示遮罩，儲存政策另定
  "requiredButBlocking": true,      // 對應「勾選不影響效力」的例外情境可設 false
  "sharedAcrossDocuments": true,    // 跨文件共用旗標
  "pdfMapping": { "page": 1, "x": 120, "y": 480, "width": 200, "height": 24 }
}
```

**欄位狀態矩陣：**

| 狀態 | 意涵 |
|---|---|
| ReadOnly + AutoFilled | 例如 HR API 帶入之身分證 |
| Editable + Required + Blocking | 例如姓名，未填無法送出 |
| Editable + Optional | 例如「其他」教育訓練項目 |
| Checkbox + Required + NonBlocking | 對應教育訓練「不勾選不影響效力」 |
| SystemField | 文件編號、簽署時間，前端永不顯示為輸入框 |

---

## 8. Document Template Model

```
DocumentTemplate
 ├── templateId, name, version, status(Draft/Published/Deprecated)
 ├── pdfFileRef, totalPages, estimatedMinutes
 ├── fields: [DocumentField...]
 ├── signatureRequirements: [SignatureRequirement...]
 └── validationRules: [ValidationRule...]

DocumentField
 ├── fieldId, type, page, x, y, width, height
 ├── required, dataSource, validation, maskRule, fillerRole
 └── linkedProfileAttribute (供跨文件共用比對，例如 "user.idNumber")
```

**新增合約 SOP（驗證「Template Driven」是否成立）：**
1. 上傳 PDF → 系統自動產生頁面截圖供標欄位用（Coordinate Picker 工具，Phase 2 後台功能，MVP 可用設定檔/CSV 匯入座標）。
2. 定義 Field 清單＋是否 `sharedAcrossDocuments`。
3. 定義 SignatureRequirement（幾處簽名、哪些角色）。
4. Publish → 立即可派送，**不需部署**。

---

## 9. Workflow Model

狀態機（對齊需求書第十三節，並補上 Resume/Expire 分支）：

```
Created → Assigned → Opened → BasicInfoCompleted → DocumentReviewed
  → FieldsCompleted → SignatureStarted → Signed → PDFGenerated → Completed
                    ↘ (中斷) Draft/Paused → (逾期) Expired
```

每個狀態轉換記錄：`timestamp, actor, ip, userAgent, device, action, result, correlationId`（與 Audit Log 共用結構，見第11節）。

**Business Rule：** 一個 Workflow 可綁定多份 DocumentTemplate 實例（對應「今天要簽3份」情境），Workflow 完成 = 所有子文件皆 Completed。

---

## 10. Signature Model

```
Signature
 ├── signatureId, workflowId, documentId, signerUserId
 ├── signatureImage (base64/blob ref)
 ├── signatureHash (SHA256 of image+timestamp+documentHash)
 ├── signedAt, ip, device, userAgent
 ├── consentChecked: boolean  // 對應「我已閱讀並同意」勾選框，需獨立記錄
 └── signatureLevel: "Basic" | "Enhanced"  // Basic=手寫圖檔；Enhanced=+OTP/簡訊驗證，Phase2/3
```

**同一流程多次簽名策略：** 使用者第一次簽名後，系統快取「本次 Session 簽名圖檔」，後續文件簽名頁提供「使用剛才的簽名」快捷按鈕，但**仍需使用者按一次「確認使用」動作**並各自產生獨立 signatureHash，避免「一次簽名套用全部」被質疑法律效力不足。

---

## 11. Audit Log Model

參考附件 Workflow Evidence Report 的呈現方式（Package/Document/Workflow/Recipients Summary），設計結構化 Log：

```json
{
  "documentId": "DOC-20260907-000001",
  "workflowId": "WF-000123",
  "userId": "U123456",
  "action": "SIGN_DOCUMENT",
  "timestamp": "2026-09-07T15:20:31+08:00",
  "ip": "203.0.113.10",
  "userAgent": "...",
  "device": "iPhone15,Safari",
  "documentVersion": "1.2",
  "documentHash": "SHA256:xxxx",
  "result": "SUCCESS",
  "correlationId": "corr-abc-123"
}
```
每個 Workflow 完成後，可依此產生對外用「Evidence Report」（同附件圖檔格式），作為法律憑證輸出，這屬於 PDF Service 的延伸功能而非另建系統。

---

## 12. Security Architecture

| 層面 | 措施 |
|---|---|
| 傳輸 | HTTPS強制、HSTS |
| 身份 | Token/Session（短期AccessToken + Refresh），簽署動作需 Re-auth 或至少 Session 未過期檢查 |
| 防護 | CSRF Token、輸出編碼防XSS、輸入白名單驗證、API Rate Limiting |
| 防重放 | 簽署請求需帶一次性 nonce + correlationId，Server端比對防止重送 |
| 資料保護 | PII 欄位（身分證、健康資料）Encryption at Rest（AES-256）＋ Encryption in Transit；Masking 於前端與Log層強制執行，**Log絕不寫入明碼身分證** |
| 存取控制 | RBAC：Applicant/HR/Supervisor/Admin/Auditor 角色分權，Auditor 僅能讀 Audit Log 不可讀 PII 明碼 |
| 完整性 | Signature Hash + Document Hash，任何 PDF 被竄改即 Hash 不符，Tamper Detection Job 定期驗證儲存體 |
| 資料生命週期 | Data Retention（依個資同意書載明 3-5年）＋ Data Deletion API＋ Data Export（供當事人行使個資法權利） |

**分級簽署設計（需求書第十六節）：**
- **Basic**：手寫簽名圖 + Hash（MVP）
- **Enhanced**：Basic + OTP/SMS/Email驗證（Phase 2）
- **Advanced**：Enhanced + FIDO/Passkey + 憑證簽章（Phase 3）
架構上以 `signatureLevel` 欄位 + Strategy Pattern 實作，避免日後大改。

---

## 13. System Architecture

```
[Web Frontend (React, Mobile-First)]
        ↓ HTTPS
[BFF / API Gateway]  ── AuthN/AuthZ, Rate Limit
        ↓
┌───────────────┬────────────────┬────────────────┬───────────────┐
│ Document       │ Workflow        │ Signature       │ PDF            │
│ Service        │ Service         │ Service         │ Service        │
└───────────────┴────────────────┴────────────────┴───────────────┘
        ↓                ↓                 ↓                ↓
             [Audit Service]（所有Service事件皆發送至此，非同步/Event-driven）
        ↓
[PostgreSQL (交易資料)] + [Object Storage/S3 (PDF/簽名圖檔)] + [Redis (Session/Draft Cache)]
        +
[Monitoring/Observability: 集中式Log、Metrics、Alerting]
```

**Modular Monolith vs Microservice 建議：**
- **MVP 先做 Modular Monolith**：Document/Workflow/Signature/PDF 以模組（同一部署單元）方式劃分邊界，共用DB但Schema分Namespace。
- **Audit Service 建議一開始就用非同步事件（即使仍在同一應用內以Event Bus/Outbox Pattern實作）**，因為稽核資料量成長快、寫入頻繁，且未來最可能獨立拆分為專責服務（合規需求常要求Audit Log與業務系統隔離）。
- **PDF Service**（生成/浮水印/Hash計算）為CPU密集工作，建議MVP即以獨立Worker（可同Repo、非同進程）處理，避免拖慢主API回應（PDF產出可非同步+輪詢/WebSocket通知完成）。
- 不建議MVP拆Microservice：團隊規模、維運成本、以及目前尚未有跨團隊獨立擴展的壓力。

---

## 14. API Design（節錄核心端點）

```
GET   /api/workflows/me                     # 我的待簽清單
GET   /api/workflows/{id}/documents/{docId} # 文件狀態+欄位定義
PUT   /api/documents/{docId}/fields         # 批次更新欄位值（含Draft自動存檔）
GET   /api/profile/prefill                  # 依User Profile/HR API取得可預填資料
POST  /api/documents/{docId}/signature      # 上傳簽名圖，回傳signatureHash
POST  /api/documents/{docId}/complete       # 觸發驗證+PDF生成
GET   /api/documents/{docId}/pdf            # 下載最終PDF
GET   /api/documents/{docId}/audit          # (Auditor角色) 稽核紀錄查詢
POST  /api/documents/{docId}/draft/resume   # 續簽
```
所有寫入類 API 皆需 Idempotency-Key，防止Network重試造成重複簽署。

---

## 15. Database ER（核心表）

```
User(id, name, idNumberEncrypted, nationality, phone, address, ...)
DocumentTemplate(id, name, version, status, pdfRef)
DocumentField(id, templateId, type, page, x, y, required, dataSource, ...)
Workflow(id, ownerId, type, status, createdAt, completedAt)
WorkflowDocument(id, workflowId, templateId, signerId, status)
FieldValue(id, workflowDocumentId, fieldId, valueEncrypted, filledBy, filledAt)
Signature(id, workflowDocumentId, signerId, imageRef, hash, level, signedAt, ip, device)
AuditLog(id, workflowId, documentId, userId, action, timestamp, ip, device, result, correlationId)
```
`FieldValue.valueEncrypted` 對PII欄位一律加密儲存，非PII欄位可明碼（效能考量），由 DocumentField 上的 `piiFlag` 決定儲存策略。

---

## 16. Frontend Component Architecture

```
<SigningFlowRouter>
 ├── <DocumentIntroScreen>
 ├── <BasicInfoForm>        (共用元件，跨文件reuse)
 ├── <DocumentReader>       (分頁PDF Viewer)
 ├── <PendingFieldsChecklist> → <FieldEditorSheet type={FieldType}>
 │        ├── <TextFieldInput> <DateFieldInput> <PhoneFieldInput>
 │        ├── <CheckboxGroup> <RadioGroup> <SelectInput>
 │        └── <SignatureCanvas>  (Canvas API, touch/pointer events統一處理)
 ├── <PreSignConfirmation>
 └── <CompletionScreen>
```
所有 FieldEditor 皆由 `DocumentField.type` 動態渲染（Factory Pattern），新增欄位類型只需註冊新Component，不動主流程程式碼——這是支撐「Template Driven」在前端的落實。

---

## 17. PDF Rendering Architecture

- **技術選型建議**：後端使用 `pdf-lib`（Node）或 `PyMuPDF/reportlab`（Python）在既有PDF上以座標疊字/疊圖，而非重新排版整份PDF（保留原始文件法律外觀）。
- 中文/外文姓名需內嵌支援中文之字型（如思源黑體）避免產生亂碼方塊。
- 簽名圖以 PNG（透明背景）疊圖方式貼入指定座標，需依 DPI 做等比例縮放。
- Checkbox：以疊圖「✓」或黑色方塊呈現於欄位座標，而非修改原PDF表單欄位（多數合約PDF並非AcroForm）。
- 完成後執行 **Flatten**（防止使用者事後用PDF編輯器竄改欄位），並計算最終 Hash。
- **PDF/A** 判斷：MVP不強制，若合約需長期歸檔（如個資同意書法定保存3-5年），建議Phase2導入PDF/A-2b長期保存格式。

---

## 18. Error Handling（Draft/Auto-Save/Resume）

| 情境 | 處理策略 |
|---|---|
| 網路中斷/API Timeout | 前端Local Buffer暫存輸入，背景重試（Exponential Backoff），成功後才清空Buffer |
| Session過期 | 攔截401 → 保留當前欄位輸入於localStorage(非PII) → 導向重新登入 → 登入後自動Resume |
| 使用者關閉瀏覽器/手機鎖定 | 每個欄位變更皆Debounce後即時PUT至Draft API（非等到整頁提交才存檔） |
| PDF產生失敗 | 記錄失敗Log＋提供「重新產生」，不可讓使用者重簽名（簽名資料已保存，僅重跑PDF Service） |
| 簽名失敗（畫布未偵測到觸控） | 前端偵測空白Canvas，禁用「完成簽署」按鈕並提示 |

---

## 19. QA Test Plan（重點項目）

- **Functional**：欄位驗證規則（Regex/MaxLength）、跨文件資料共用是否正確帶入且可個別修改不互相污染。
- **Browser/Mobile**：iOS Safari／Android Chrome 的 Canvas 觸控事件相容性（含高DPI裝置簽名清晰度）。
- **PDF Verification**：比對產出PDF文字/座標與Template定義是否一致（自動化比對工具，逐欄位截圖比對）。
- **Signature Verification**：簽名Hash重算比對、Flatten後PDF不可再編輯之驗證。
- **Boundary Test**：欄位最大長度、特殊字元（外籍人員姓名含空格/符號）、身分證格式邊界。
- **Resume Test**：模擬中斷於各個Workflow狀態，驗證Resume後導向正確畫面且資料不遺失。

## 20. Security Test Plan

- Penetration Test：CSRF/XSS/SQLi/IDOR（尤其 `/documents/{docId}` 是否可被平行使用者存取他人文件）。
- Replay Attack模擬：重送已完成之簽署請求應被拒。
- PII洩漏檢測：全鏈路Log Grep掃描是否曾寫入明碼身分證字號。
- Rate Limit驗證：簽名/OTP端點暴力測試。
- Tamper Detection：手動修改已存PDF後觸發Hash比對Job，驗證是否即時告警。

---

## 21. MVP Scope（Phase 1）

Template、PDF Viewer、基本資料頁（含跨文件共用）、待完成清單引導、Checkbox/Text/Date/Signature元件、PDF產出（含Hash/Flatten）、Workflow狀態機、Audit Log、基本RBAC、Auto-Save、Resume。**明確排除**：OTP、多人序列簽署（附件截圖顯示的Serial Workflow概念，MVP先只支援單一簽署人+固定他人填寫欄位如講師簽核，不做完整多方會簽UI）、管理後台Template視覺化編輯器（MVP用結構化設定檔/CSV上傳取代）。

## 22. Phase 2 / Phase 3 Roadmap

- **Phase 2**：OTP/簡訊驗證（Enhanced簽署）、多人序列簽署UI（呼應附件之Serial Workflow）、管理後台＋Template視覺化座標編輯器、Email/簡訊通知系統。
- **Phase 3**：FIDO/Passkey、多租戶（供集團旗下多公司共用平台）、第三方電子簽章（如自然人憑證）介接、進階Audit分析儀表板。

---

## 23. SA/PG/QA/SD 討論重點（Workshop Notes）

- **SA↔SD**：Field Mapping的座標系統若PDF版本更新（如條款文字增加一行導致後續欄位位移），需有Template Versioning機制，舊版Workflow仍需綁定舊版Template座標，不可讓新版覆蓋舊版履歷。
- **PG↔QA**：Signature Canvas在低階Android裝置的觸控延遲需納入效能測試基準，避免簽名筆跡失真被質疑法律效力。
- **Security↔SA**：`FillerRole`（如教育訓練單的講師欄位）代表系統需支援「同一Workflow下、不同使用者於不同時間點填寫不同區塊」，這是Workflow Model需要重新檢視的地方（目前假設單一Signer，但講師簽核其實是第二個Actor）。
- **PM↔全體**：「未逐項勾選不影響效力」這條Business Rule若理解錯誤，可能導致QA誤判為Bug，需要在Validation Rule文件明確定義並在Demo時特別展示。

---

## 24. Technical Risk

| 風險 | 影響 | 緩解 |
|---|---|---|
| PDF座標式疊圖對「條款文字量會變動」的合約脆弱 | 版面錯位、法律效力疑慮 | Template Versioning + 上版前QA視覺比對工具 |
| 手寫簽名法律效力於部分情境不足（如高價值合約） | 爭議風險 | Signature Level分級設計，關鍵文件強制Enhanced |
| 多文件共用基本資料，若使用者中途發現資料錯誤要修改，可能影響已簽署文件的一致性 | 資料完整性/稽核疑慮 | 已完成簽署文件之基本資料應為「快照」不可回溯修改，僅未簽署文件套用最新資料 |
| PII加密欄位查詢效能（如管理端搜尋姓名） | 查詢效能下降 | 建立Searchable Encryption或另存Hash索引欄位供精確比對查詢 |

## 25. Open Questions

1. 「講師/單位/日期」欄位由誰在系統中操作？是否需要獨立的內部審核介面（而非單純Signer視角）？
2. 個資同意書之「使用期間」（保留3-5年）是否需要系統自動排程到期提醒/刪除？由誰觸發？
3. 外籍人員的姓名格式（英文/護照拼音）與身分證/居留證號碼欄位驗證規則是否需分開兩套Regex？
4. 附件Evidence Report所使用的第三方服務（SH平台）是否為既有系統，本平台是否需與其並存或取代？
5. MVP是否要支援使用者用「已離職/已完成派遣」身分回頭下載歷史簽署文件？（涉及Data Retention與存取權限設計）

---

*本文件為第一階段產品規劃與架構評審輸出，尚未進入程式開發。建議下一步：SA針對第7-11節產出正式Schema、SD針對第13節產出部署圖、PG針對第16-17節評估技術選型POC、QA同步草擬第19-20節測試案例明細。*
