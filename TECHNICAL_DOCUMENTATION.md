# 金寶貝智能照護 — 技術文件

> 專為家長設計的智能健康監測平台：連線 IoT 裝置即時追蹤孩童健康狀況，整合邏輯判斷式動態問卷引擎、照護任務派發、多機構權限管理與邀請碼註冊自動化。

- **發布網址**：https://pragmatic-gold-baby-care.base44.app
- **技術棧**：React 18 + Vite（ESM）+ Tailwind CSS + shadcn/ui + lucide-react；後端採 Base44 平台服務（Auth / Entities / Functions / Email）
- **介面語言**：繁體中文
- **文件版本**：v1.1（2026-10-18 更新；新增路由總表與頁面對照）

---

## 1. 系統架構總覽

系統分為三大模組：

| 模組 | 對象 | 內容 |
|---|---|---|
| 家長 APP | 家長（parent） | 健康儀表板、時間軸、照護任務、問卷填答、IoT 量測、個人檔案 |
| 管理後台 | 超級管理員 / 機構管理員 / 醫師 / 個管師 | 個案管理、問卷建立與派發、任務指派、帳號管理、機構與專科管理、邀請碼管理、同意書管理 |
| 動態問卷引擎 | 全體 | 題組 + 條件分支（branch_rules）、Tier 0–5 分層、版本控制、答後衛教內容 |

```
前端 (React SPA)
 ├── /login /register /forgot-password /reset-password   ← 平台內建 Auth（Email+OTP，無 Google 註冊）
 ├── RoleRedirect (/)                                      ← 依 role 分流：parent → /dashboard，其餘 → /admin
 ├── ParentLayout（底部導覽）→ 家長各頁
 └── AdminLayout（側邊欄，分組可摺疊）→ /admin 各頁

後端 (Base44 Functions, base44/functions/*/entry.ts)
 └── 以 createClientFromRequest(req) 建立客戶端；需越權資料存取時走 asServiceRole（跳過 RLS）
```

### 資料流原則
- **前端一律經 `base44.functions.invoke('<functionName>', payload)` 呼叫後端函式**，不使用絕對 URL。
- 需要跨機構 / 跨使用者存取的邏輯（收案自動化、問卷自動派發、帳號建立）集中在後端函式，以 service role 執行並自帶身分檢查。
- 健康數據區分 `source: 'iot' | 'manual'`；家長端隱藏病歷號等敏感欄位。

---

## 2. 角色與權限矩陣

角色定義於 `base44/entities/User.jsonc`（`role` 欄位，預設 `parent`）。

| 角色 | 主要職責 |
|---|---|
| `super_admin` 超級管理員 | 機構／專科／同意書／問卷／裝置的建立編輯審核；邀請碼申請審核與核發；所有臨床資料實體的寫入；機構管理員帳號建立 |
| `org_admin` 機構管理員 | **僅**管理機構內醫護人員帳號＋提出邀請碼申請；已退出所有臨床資料實體的存取 |
| `doctor` 醫師 | 查看與派發（同機構）個案／任務／問卷／回應；建立藥物；領取與發送邀請碼 |
| `case_manager` 個管師 | 同醫師的查看與派發權；領取（含綁定專科）與發送邀請碼；建立量測紀錄 |
| `parent` 家長 | 量測／手動紀錄、填問卷、簽同意書；僅能存取自己被授權的照護對象資料 |

### RLS（Row-Level Security）設計模式
各實體於 `base44/entities/<Name>.jsonc` 的 `rls` 區塊套用以下模式組合：

1. **家長所有權**：`data.authorized_parents: "{{user.id}}"`（CareRecipient、IoTDevice、Medication、HealthRecord 讀取）。
2. **機構 + 角色限定**：`$and[ {data.organization_id: "{{user.data.organization_id}}"}, {user_condition: {role: "doctor"|"case_manager"}} ]`。
3. **超級管理員全開**：`user_condition: {role: "super_admin"}`。
4. **公開唯讀**：如 Consent `read: {}`（已發布同意書任何人可讀）。
5. **寫入集中**：臨床／管理實體的 `create/update/delete` 原則上限 `super_admin`（CareRecipient、Questionnaire、Specialty、Organization、CaseOrganizationLink 等）；家長可寫自己的量測（`created_by_id`）、任務與問卷回應（`parent_user_id`）。

> 機構管理員刻意不在臨床資料實體的 RLS 內——其職責僅限帳號與邀請碼申請。

---

## 3. 資料模型（Entities）

所有實體具內建欄位：`id`、`created_date`、`updated_date`、`created_by_id`。

| 實體 | 用途 | 關鍵欄位 |
|---|---|---|
| **Organization** | 醫療機構 | name、code（病歷號前綴）、status(active) |
| **Specialty** | 專科 | organization_id、name、code、`is_default`（GP 預設專科） |
| **User**（內建） | 帳號 | role、display_name、organization_id/name、specialty(_id)、phone、license_number |
| **CareRecipient** | 照護對象（孩童個案） | name、gender、birth_date、medical_record_number、blood_type、allergies、`authorized_parents[]`、organization_id(s)、`is_temporary_mrn`、mrn_updated_at/by |
| **CaseOrganizationLink** | 個案 × 機構（支援多機構） | care_recipient_id、organization_id、medical_record_number、mrn 更新追蹤 |
| **CaseAssignment** | 收案指派 | care_recipient_id、case_manager_id、`specialty_id`、join_method(assigned/self_join)、status |
| **InviteRequest** | 邀請碼申請（org_admin 提出） | organization_id、requested_by、quantity、reason、status(pending/approved/rejected) |
| **InviteCode** | 邀請碼 | code（`GB-` + 10 碼）、valid_days/expires_at、status(available→claimed→sent→used/expired/revoked)、`specialty_id`（領碼綁定）、claimed_by（領碼個管師）、used_by、care_recipient_id |
| **ParentAccessRequest** | 舊制家長照護對象申請（已停用新增，舊件仍可審核） | parent_user_id、care_recipient 資訊、status(pending/approved/rejected)、created_mr_number |
| **Questionnaire** | 問卷範本 | title、usage(gp/specialty/special_investigation)、version、status(draft/published/archived)、tier_level(tier0–5)、applicable_age_min/max_months、fill_frequency、requires_dispatch、target_care_recipient_ids、`questions[]` |
| **QuestionnaireResponse** | 有效派發單 / 填答 | questionnaire_id(+version)、care_recipient_id、parent_user_id、status(in_progress/submitted)、answers{}、assigned_by_id/name |
| **CareTask** | 照護任務 | care_recipient_id、parent_user_id、type(medication/appointment/checkup/measurement/other)、status(pending/completed/missed/skipped)、scheduled_at、recurrence、assigned_by_id/name |
| **HealthRecord** | 健康量測紀錄 | care_recipient_id、metric_type(temperature/heart_rate/spo2/weight/height/血壓/呼吸/血糖/步數)、value、unit、`source(iot/manual)`、device_id、recorded_at、authorized_parents[] |
| **IoTDevice** | IoT 裝置 | name、device_type、serial_number、care_recipient_id、status、last_sync、battery_level、authorized_parents[] |
| **Medication** | 藥物 | care_recipient_id、name、dosage、frequency、start/end_date、prescribed_by |
| **Consent** | 同意書範本 | title、type（17 種，如 service_terms、privacy_policy、health_data_collection…）、version、content、is_required、trigger_stage、status |
| **ConsentRecord** | 電子簽章紀錄 | consent_id(+version)、user_id、agreed_at、`ip_address`、`user_agent`、care_recipient_id、status(agreed/withdrawn)、withdrawn_at |
| **AuditLog** | 操作稽核 | operation(create/update/delete/dispatch)、entity_type/id/name、operator_id/role、organization_id、details |

### 問卷題目結構（`Questionnaire.questions[]`）
每題：`id`、`text`、`type`（text/number/single_choice/multiple_choice/scale/boolean/date）、`options[]`、`required`、`order`、`group`（題組）、`next_question_id`（預設下一題）、`branch_rules[]`（依答案 `answer → next_question_id` 跳題／追問）、`scale_min/max`、`tier`（題目分層）、`education_content`（答後條件衛教）。

---

## 4. 後端函式（API）

後端函式位於 `base44/functions/<name>/entry.ts`，HTTP 端點：`https://pragmatic-gold-baby-care.base44.app/functions/<name>`；App 內以 `base44.functions.invoke()` 呼叫。

### 4.1 邀請碼生命週期

| 函式 | 權限 | 輸入 | 行為 / 回傳 |
|---|---|---|---|
| `validateInviteCode` | 公開（註冊頁預檢） | `{code}` | 檢查存在／已用／撤銷／過期 → `{valid, message?, organization_name?}` |
| `consumeInviteCode` | 已登入（註冊驗證後） | `{code}` | 原子兌換：status→`used`、記錄 used_by/email/at；二次讀取驗證避免競態 → `{success}` / 404/409 |
| `issueInviteCodes` | super_admin | `{request_id, valid_days?}` | 核發 1–200 組 `GB-XXXXXXXXXX`，效期 1–365 天；InviteRequest → approved |

**狀態機**：`available` →（個管師/醫師領碼，綁定 specialty）`claimed` →（發送）`sent` →（家長兌換）`used`；任一時點可 `revoked`／到期 `expired`。

### 4.2 家長註冊 → 收案自動化

| 函式 | 權限 | 行為 |
|---|---|---|
| `parentOnboarding` | 已登入家長 | `action:'info'`：回傳已兌換邀請碼的 `{organization_name, specialty_name, case_manager_name, care_recipient_id}`；`action:'complete'`：填入孩童資料後自動執行收案自動化（見 5.2），並將 `care_recipient_id` 回寫邀請碼 |
| `createParentAccount` | org_admin / super_admin | 後台建立帳號：register 後以 service role 回填 role、機構、專科；super_admin 可指定任意機構與角色，org_admin 僅限 doctor/case_manager/parent 且限本機構 |
| `sendVerificationOtp` | super_admin | 對指定 Email 重發驗證 OTP（10 分鐘效期） |

### 4.3 機構管理

| 函式 | 權限 | 行為 |
|---|---|---|
| `setupOrganization` | super_admin | 建立機構（active）＋預設「一般科 GP」專科＋自訂專科＋可選 org_admin 帳號 |
| `fixOrgAdminOrganization` | super_admin | 批次修正多個 Email 的 org_admin 所屬機構 → `{results: [{email, status: updated/already_set/not_found}]}` |

### 4.4 其他

| 函式 | 權限 | 行為 |
|---|---|---|
| `getParentCaseManagers` | 已登入家長 | 查自己授權個案的 active 收案指派（個管師／機構／專科） |
| `resolveUsersByIds` | 已登入 | 以 service role 將 user_ids 解析為 `{id, full_name, display_name, email, role}`（繞過內建 User 可見性限制） |
| `sendTaskNotification` | admin | 以 `TaskAssigned` Email 範本通知家長任務指派 |
| `approveParentAccessRequest` | super_admin | 舊制申請審核；核准後與新流程共用同一套收案自動化共用邏輯 |

### 4.5 共用模組 `base44/shared/parent-onboarding.ts`

`approveParentAccessRequest` 與 `parentOnboarding` 共用：

- `createCareRecipientWithLink`：建立 CareRecipient（`authorized_parents=[parent]`）＋ CaseOrganizationLink；病歷號 = `{機構code}-{流水號6碼}`，標記 `is_temporary_mrn: true`。
- `createCaseAssignment`：建立收案指派（個管師 × 專科）。
- `getDefaultSpecialty`：取得機構 GP 預設專科。
- `calcAgeMonths`：由出生日算月齡（30.4375 天/月）。
- `dispatchGpQuestionnaires`：篩選已發布、usage=gp、月齡符合 min/max 的問卷，排除已派發者後 bulkCreate QuestionnaireResponse（`assigned_by_name: '系統自動派發（GP）'`）。

---

## 5. 核心業務流程

### 5.1 邀請碼註冊流程（唯一註冊管道；無 Google 註冊）
1. **org_admin** 於後台提出 InviteRequest（數量＋用途）。
2. **super_admin** 審核 → `issueInviteCodes` 批次核發。
3. **doctor / case_manager** 於「邀請碼管理」領碼，**必選專科**（預設 GP）→ status `claimed`；發送給家長 → `sent`。
4. 家長於註冊頁輸入邀請碼（即時 `validateInviteCode` 預檢）→ Email＋密碼註冊 → OTP 驗證 → `consumeInviteCode` 兌換。
5. 兌換成功 → 導向 `/care-recipient-application` 設定照護對象（見 5.2）。

### 5.2 收案自動化（無需審核）
家長填寫孩童基本資料＋完成必要同意書簽署後送出 `parentOnboarding(action:'complete')`：
1. 建立照護對象＋機構連結＋虛擬病歷號（is_temporary_mrn）。
2. 以**領碼個管師 × 邀請碼綁定專科**自動收案指派（無綁定則 fallback GP）。
3. 自動派發符合月齡的 GP 問卷（去重後建立 in_progress 派發單）。
4. 邀請碼回寫 `care_recipient_id`（防重複建立）。

**個管師主動收案（個案已存在的補收）**：家長早已註冊兌換、或領碼未綁專科 fallback GP 的個案，個管師可於個案詳情頁點「收案」→ 選擇專科（預設本人登記專科）→ 建立 `join_method: 'self_join'` 的 CaseAssignment（顯示為「自行加入」）；本人已有 active 指派時按鈕顯示「已收案」。

### 5.3 問卷引擎
- 範本（Questionnaire）分 usage：`gp`（授權後自動派發）、`specialty`（收案後派發）、`special_investigation`（申請制）。
- 超管於 Builder 建立：題組、排序、條件分支、分層、衛教內容；支援發布前預覽與版本遞增；歷史版本保護。
- 派發以 **QuestionnaireResponse（有效派發單）** 為來源，非範本本身；`requires_dispatch=true` 的問卷家長不可自行看見，須由個管師派發。
- 填答：家長沿 branch_rules 逐題作答，答後依答案顯示條件型衛教內容；提交時記錄 questionnaire_version。
- 所有問卷相關操作記錄 `assigned_by_id/name`（發送的個管師）。

### 5.4 同意書與電子簽章
- Consent 範本含版本控制與 `trigger_stage`（registration / join_organization / health_feature / iot_binding / questionnaire / care_recipient_application…）。
- 簽署時建立 ConsentRecord：記錄同意書版本、IP、User-Agent、時間戳；撤回記錄 withdrawn_at。
- 個人檔案頁顯示該帳號已簽署同意書列表。

### 5.5 健康數據與 IoT
- 家長可手動輸入（QuickAdd）或經 IoT 量測流程記錄；`source` 區分來源。
- 手動紀錄以**時間軸**呈現（依 recorded_at 排序，同類型不合併）。
- 儀表板以卡片＋摘要數值＋圖表（recharts）呈現趨勢。
- 超管管理 IoTDevice（綁定照護對象、電量、最後同步）。

### 5.6 任務指派
- doctor / case_manager 建立 CareTask 指派家長，可設定排程與重複頻率。
- 指派時記錄 `assigned_by_id/name`，並經 `sendTaskNotification` 寄送 `TaskAssigned` Email 範本。
- 家長於任務頁標記完成／錯過。

---

## 6. 前端結構

```
src/
├── App.jsx                        # 路由 + AuthProvider + QueryClientProvider
├── pages/
│   ├── Login / Register / ForgotPassword / ResetPassword   # Auth（Email+OTP）
│   ├── admin/                    # AdminOverview, CaseManagement, CaseDetail,
│   │                              # TaskDispatch, QuestionnaireBuilder, QuestionnaireResponses,
│   │                              # QuestionnaireDispatch, DeviceManagement, AccountManagement,
│   │                              # OrganizationManagement, SpecialtyManagement,
│   │                              # ParentAccessReview, ConsentManagement, InviteManagement
│   └── parent/                    # Dashboard, HealthTimeline, Tasks, Questionnaires,
│                                  # QuestionnaireTake, Profile, IoTMeasurement,
│                                  # CareRecipientApplication（無底部導覽的 Onboarding 頁）
├── components/
│   ├── layout/                    # ParentLayout（底部導覽）, AdminLayout（可摺疊側邊欄）
│   ├── questionnaire/             # QuestionnaireRenderer, CaseSelector, GroupManager
│   ├── consent/                   # ConsentSigner, ConsentRecords
│   ├── invite/                    # InviteCodePanel, InviteReviewPanel, InviteRequestForm
│   ├── health/                    # QuickAddHealthRecord, HealthTrendChart, HealthMetricCard
│   ├── case/                      # CaseManagerAssignment, MrnEditor
│   └── ui/                        # shadcn/ui 元件
└── lib/                           # AuthContext, audit-log, query-client, app-params…
```

### 路由與守衛
- 認證頁：`/login`、`/register`、`/forgot-password`、`/reset-password`（平台內建流程）。
- `/` → `RoleRedirect`：parent → `/dashboard`；後台角色 → `/admin`。
- 受保護路由以 `ProtectedRoute` 包裹；家長頁在 `ParentLayout` 下、後台頁在 `AdminLayout` 下。
- 後台側邊欄依角色過濾選單、按功能分組、可摺疊，並顯示待審數量徽章。

### 路由總表

| 路由 | 頁面 | 主要使用者 | 說明 |
|---|---|---|---|
| `/login`、`/register` | Login / Register | 公開 | Email＋密碼＋OTP；註冊需邀請碼（validateInviteCode 預檢 → consumeInviteCode 兌換） |
| `/forgot-password`、`/reset-password` | Forgot / ResetPassword | 公開 | 平台內建密碼重設 |
| `/admin/login` | AdminLogin | 公開 | 後台登入入口 |
| `/` | RoleRedirect | 已登入 | 依 role 分流至家長端或後台 |
| `/dashboard` | Dashboard | parent | 儀表板（任務／問卷摘要、照護對象、最新量測） |
| `/timeline` | HealthTimeline | parent | 手動健康紀錄時間軸（同類型不合併，依時間排序） |
| `/tasks` | Tasks | parent | 照護任務清單（完成／錯過） |
| `/questionnaires`、`/questionnaires/:id` | Questionnaires / QuestionnaireTake | parent | 有效派發單列表與逐題填答（branch_rules＋條件衛教） |
| `/profile` | Profile | parent | 個人檔案（含已簽署同意書列表） |
| `/iot-measure` | IoTMeasurement | parent | IoT 量測流程（source=iot） |
| `/care-recipient-application` | CareRecipientApplication | parent（剛兌換邀請碼） | Onboarding：填孩童資料＋簽同意書 → parentOnboarding 收案自動化（無底部導覽） |
| `/admin` | AdminOverview | 後台角色 | 總覽（統計、待辦） |
| `/admin/cases`、`/admin/cases/:id` | CaseManagement / CaseDetail | doctor、case_manager、super_admin | 個案清單與詳情（病歷號補正、個管師指派） |
| `/admin/tasks` | TaskDispatch | doctor、case_manager | 任務建立與指派（Email 通知） |
| `/admin/questionnaires` | QuestionnaireBuilder | super_admin | 問卷範本建立／版本／發布（含預覽） |
| `/admin/responses` | QuestionnaireResponses | doctor、case_manager | 回應檢視 |
| `/admin/questionnaire-dispatch` | QuestionnaireDispatch | doctor、case_manager | 問卷派發（有效派發單） |
| `/admin/devices` | DeviceManagement | super_admin | IoT 裝置管理 |
| `/admin/accounts` | AccountManagement | org_admin、super_admin | 帳號管理（含邀請碼使用歷程：申請→核發→發送→兌換） |
| `/admin/organizations` | OrganizationManagement | super_admin | 機構管理（setupOrganization 建置） |
| `/admin/specialties` | SpecialtyManagement | super_admin | 專科管理（GP 預設專科） |
| `/admin/parent-requests` | ParentAccessReview | super_admin | 舊制申請審核（核准走同一收案自動化） |
| `/admin/consents` | ConsentManagement | super_admin | 同意書範本管理（版本、trigger_stage） |
| `/admin/invites` | InviteManagement | org_admin、super_admin、doctor、case_manager | 依角色分區：申請（InviteRequestForm）／審核核發（InviteReviewPanel）／領碼綁專科與發送（InviteCodePanel） |

### 設計系統
- Token 定義於 `src/index.css`（`:root` / `.dark`）：primary 青綠（`173 68% 42%`）、accent 暖橘、radius 0.75rem、字體 Noto Sans TC。
- 家長端：明亮親和、卡片式、底部導覽；後台：側邊欄＋表格／表單；全面響應式（mobile-first）。

---

## 7. Email

| 範本 | 觸發 |
|---|---|
| `base44/emails/TaskAssigned.html` | 任務指派（`sendTaskNotification`） |
| 平台內建（密碼重設、驗證碼、邀請、開通信件） | Auth 流程 |

Email 品牌色：header `#22b4a3`、按鈕 `#22b4a3`、字體 Noto Sans TC。寄送對象限 App 註冊使用者。

---

## 8. 安全與稽核

- **前端不可信任**：所有權限判斷以 RLS＋後端函式內 role 檢查為準。
- 後端函式需越權存取時使用 `asServiceRole`，且每支函式自行驗證 `auth.me()` 與 role。
- 敏感操作寫入 **AuditLog**（操作者、角色、機構、變更摘要）。
- 同意書簽署留存完整電子簽章證據（IP／裝置／時間戳／版本）。
- 家長端不顯示病歷號；病歷號由機構 code＋流水號自動產生，虛擬號標記可由超管補正。

---

## 9. 已知限制與待辦

**待開發**
- 健康異常警示系統（數值超標通知）
- 推播通知（App 內／行動裝置）
- 報表匯出功能
- IoT 數據模擬 demo
- 次專科收案問卷自動派發機制
- 特殊調查申請流程與衛教條件強化

**已知議題**
- 帳號管理介面驗證狀態顯示不一致
- CaseOrganizationLink 建立後立即 filter 查詢可能因索引傳播延遲而查無資料

**未來規劃**
- 微軟登入 / 雙因子認證整合

---

## 10. 環境與建置

- 建置：`npm run build`（Vite）；開發：`npm run dev`
- 授權與 Schema：`base44/config.jsonc`、`base44/entities/*.jsonc`
- 套件管理：僅使用既有安裝套件（React、Tailwind、shadcn/ui、recharts、framer-motion、react-quill-new…），新增套件需明確指定