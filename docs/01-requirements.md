# 01｜需求規格（Requirements）

> **專案：** 校園二手物交易與預約平台  
> **狀態：** Design Only

## 1. 問題背景與系統目標（Problem & Goal）

校園二手物常刊登在格式不一致的社群貼文中，買家不容易依類別、系所與課程快速尋找商品；即使找到商品，也可能無法清楚比較賣家原先設定的售價、面交日期時間及地點，以及買家希望調整的交易條件。多人同時表達購買意願時，也容易發生重複保留的情況。

**目標：** 建立一個小型網際服務，讓賣家刊登明確交易條件、買家篩選商品並選擇直接預約或有限次數的交易提案；**成功預約者獲得唯一鎖定**，避免多位買家同時預約同一商品。只支援線下面交，不打造完整電商平台。

## 2. 目標使用者（Users）

| 使用者 | 工作與需求 |
|---|---|
| 賣家 | 刊登／編輯商品、查看並處理提案、確認面交完成或取消預約。 |
| 買家 | 查詢商品、直接預約或提出交易條件、查看自己的請求、撤回提案或取消預約。 |
| 訪客 | 查看公開商品資訊；受保護的寫入與交易請求查詢需操作憑證。 |

使用者以學生為主，但不驗證其真實校園身分。以全站唯一 `nickname` 和隨機操作憑證識別系統內的操作人，不實作完整會員登入或帳號復原。

## 3. 主要情境（Scenario）與亮點

賣家刊登二手電風扇，公開售價新臺幣 500 元、面交時間 `2026-12-05T15:00:00+08:00`、地點「校園圖書館一樓」。買家 A 看到商品後，可以直接預約；也可以提交一筆提案，希望改成 450 元或更晚的面交時間。

- **路徑 A：直接預約。** A 成功建立 `DIRECT` 交易請求，該請求和商品立即轉為 `RESERVED`。
- **路徑 B：提案後預約。** A 建立 `PENDING` 的 `PROPOSAL`，賣家接受時直接將同一交易請求轉為 `RESERVED` 並鎖定商品，不需 A 再次確認。
- **防撞單：** 若 B 已先成功預約，A 的提案接受不能再成功；若 A 先成功鎖定，B 的直接預約也不能成功，衝突回傳 `409 Conflict`。

**可觀察亮點：** 一件商品在任何時刻至多一筆有效 `RESERVED` 交易請求，直接預約與接受提案共用同一個鎖定規則。

## 4. 功能範圍（Scope）

### 4.1 In Scope

1. 建立全站唯一暱稱與隨機操作憑證。
2. 刊登與編輯商品；支援分類、系所及課程篩選。
3. 公開二手售價、面交日期時間及地點；購買原價選填且未查證。
4. 同一買家對同一商品最多兩次正式交易條件提案；賣家接受或拒絕，買家可撤回。
5. 依公開條件直接預約，或接受提案時自動預約；防止重複鎖單。
6. 查看有權存取的交易請求、取消有效預約、完成線下面交紀錄。

### 4.2 Out of Scope

- 公開問答、預約專屬留言、私人聊天、即時通知、圖片／附件上傳或站內改約。
- 金流、物流、評價、檢舉、競標、候補、自動到期與排程取消。
- 完整帳密／JWT／SSO、校園身分認證、跨裝置登入、憑證遺失復原。
- 收據查證、地圖／行事曆／付費 API、真實敏感個資。
- 實作程式碼、資料庫、可執行 API 或測試（本次為設計文件）。

**範圍限制：** 由於本次不提供站內聯絡或預約後變更條件，買賣雙方須依成立時記錄的日期時間、地點自行面交；若無法履約，可取消預約，不能在系統內改約。

## 5. 功能需求與驗收標準（FR & Acceptance Criteria）

以下每項以可檢查的 Given／When／Then 描述**預期行為**，不是已實測通過的結果。

### FR-01｜建立使用者與操作識別

**需求：** 訪客輸入 `nickname` 建立使用者，取得 `user_id` 與僅建立時回傳一次的隨機操作憑證；受保護操作依 `Authorization: Bearer <token>` 識別使用者。

- **AC-01a：** Given 暱稱尚不存在，When 建立使用者，Then 回傳 `201 Created`，包含 `user_id`、暱稱及操作憑證。
- **AC-01b：** Given 暱稱已存在，When 再建立同名使用者，Then 回傳 `409 Conflict`。
- **AC-01c：** Given 未提供或憑證無效，When 呼叫受保護 API，Then 回傳 `401 Unauthorized`；有效憑證但操作他人資源則回傳 `403 Forbidden`。

對應：BR-01、BR-08、BR-11；API-01，及其他受保護 API。

### FR-02｜刊登及編輯商品

**需求：** 賣家可刊登商品並編輯其 `AVAILABLE` 商品；有待處理交易提案時，禁止修改公開交易條件。

- **AC-02a：** Given 賣家已識別，When 提供名稱、分類、物況、售價、未來面交時間及非空地點，Then 建立 `AVAILABLE` 商品並回傳 `201 Created`。
- **AC-02b：** Given 必填欄位缺漏、售價非正值、時間非未來或分類不合法，When 刊登／編輯，Then 回傳 `422 Unprocessable Content`。
- **AC-02c：** Given 商品 `AVAILABLE` 且無 `PENDING` 提案，When 賣家調整公開價格、面交時間或地點，Then 更新成功。
- **AC-02d：** Given 商品仍有 `PENDING` 提案，When 賣家修改公開售價、面交時間或地點，Then 回傳 `409 Conflict`，商品及提案都不變；修改商品描述等非交易條件仍可成功。
- **AC-02e：** Given 商品 `RESERVED` 或 `SOLD`，When 嘗試編輯，Then 回傳 `409 Conflict`。

對應：BR-02、BR-03、BR-04、BR-07、BR-08；API-02、API-04。

### FR-03｜查詢與篩選商品

**需求：** 訪客可依商品名稱、分類、系所、課程及狀態查詢公開商品資料；預設只顯示 `AVAILABLE`。

- **AC-03a：** Given 有已刊登的商品，When `GET /items` 不提供狀態篩選，Then 僅回傳 `AVAILABLE` 商品及其公開售價、面交時間地點。
- **AC-03b：** Given 使用分類、系所、課程或名稱關鍵字條件，When 查詢，Then 結果符合所有已指定條件；教科書系所與課程可空。
- **AC-03c：** Given 商品提供購買原價，When 查詢，Then 回傳可供顯示的售價占原價百分比；未提供時比率為 `null`，不要求憑據。

對應：BR-02、BR-03；API-03。

### FR-04｜建立交易請求（提案或直接預約）

**需求：** 買家透過同一個資源建立 `PROPOSAL` 或 `DIRECT` 請求。

- **AC-04a：** Given 商品 `AVAILABLE` 且買家非賣家，When 送出 `DIRECT` 請求，Then 回傳 `201 Created`；請求與商品同時變成 `RESERVED`，記錄公開交易條件快照。
- **AC-04b：** Given 商品 `AVAILABLE`，When 送出有效 `PROPOSAL` 且尚有額度、無其他自己未處理提案，Then 建立 `PENDING` 請求；未調整的欄位沿用公開條件。
- **AC-04c：** Given 提案至少一項條件不同且有效，When 建立，Then 消耗一次提案額度；如價格調整，必須大於 0 且低於公開售價。
- **AC-04d：** Given 同一買家已有一筆 `PENDING` 或累積兩次已提交提案，When 再提案，Then 回傳 `409 Conflict`，不建立新請求。
- **AC-04e：** Given 商品已被預約／售出，或賣家試圖對自己商品送出請求，When 建立，Then 回傳 `409 Conflict` 或 `403 Forbidden`（賣家本人）；不建立新請求。

對應：BR-05、BR-06、BR-08、BR-09；API-05。

### FR-05｜查看與處理交易請求

**需求：** 買家可查看自己對特定商品的交易請求；賣家可查看自己商品的全部請求。買家可撤回待處理提案，賣家可接受或拒絕待處理提案。

- **AC-05a：** Given 有效操作憑證，When 查詢商品請求清單，Then 賣家可取得該商品全部請求，買家僅能看到自己的請求；不得洩漏其他買家的提案。
- **AC-05b：** Given `PENDING` 提案，When 提案買家將狀態改為 `WITHDRAWN`，Then 更新成功並保留已用次數。
- **AC-05c：** Given `PENDING` 提案且商品 `AVAILABLE`，When 商品賣家將狀態改為 `REJECTED`，Then 更新成功；買家在剩餘額度內可再提案。
- **AC-05d：** Given `PENDING` 提案且商品 `AVAILABLE`，When 商品賣家將狀態改為 `RESERVED`，Then **同一原子性交易**鎖定商品與該請求，記錄接受的價格／時間／地點，並使其他 `PENDING` 請求 `INVALIDATED`。
- **AC-05e：** Given 商品已被其他請求鎖定，When 賣家同時接受另一筆提案，Then 回傳 `409 Conflict`，不形成第二筆有效預約。

對應：BR-05、BR-06、BR-08、BR-09、BR-10；API-06、API-07。

### FR-06｜取消預約與完成交易

**需求：** 有效預約的買家或賣家可以取消；僅賣家可以確認已完成面交。

- **AC-06a：** Given 請求為 `RESERVED` 且商品 `RESERVED`，When 其買家或商品賣家將請求狀態改為 `CANCELLED`，Then 請求保留取消紀錄，商品恢復 `AVAILABLE`。
- **AC-06b：** Given 請求 `RESERVED`，When 賣家將請求狀態改為 `COMPLETED`，Then 請求變成 `COMPLETED`，商品變成 `SOLD`。
- **AC-06c：** Given 請求已為終止狀態（如 `CANCELLED`／`COMPLETED`），When 再提出狀態變更，Then 回傳 `409 Conflict`。
- **AC-06d：** Given 商品因取消而重新開放，When 其他買家預約，Then 建立**新的**請求；歷史提案／預約不恢復，也不重置已使用提案額度。

對應：BR-09、BR-10、BR-11；API-07。

## 6. 業務規則（Business Rules）

| 編號 | 規則 |
|---|---|
| **BR-01** | `nickname` 全站唯一。建立時產生 `user_id` 與高熵隨機憑證；僅儲存憑證雜湊。 |
| **BR-02** | 商品分類限 `TEXTBOOK`、`APPLIANCE`、`DAILY_GOODS`；系所、課程選填。 |
| **BR-03** | `selling_price` 為正整數新臺幣；`original_price` 選填，若填寫也須為正整數；原價不查證。 |
| **BR-04** | 面交日期時間 `meeting_time`、面交地點 `meeting_place` 均必填；時間須晚於提交當下，統一臺北時間。 |
| **BR-05** | 買家對同一商品最多成功提交兩次 `PROPOSAL`，同時只能有一筆自己的 `PENDING`；每次提交計 1 次，拒絕、撤回、失效皆不返還。 |
| **BR-06** | 提案至少更改一項公開交易條件；未填欄位沿用目前公開值；若更改售價，提案價格須大於 0 且低於公開售價；提案無自動到期或反向出價。 |
| **BR-07** | 賣家只可編輯自己 `AVAILABLE` 的商品。若有任何 `PENDING` 提案，禁止變更公開售價、面交時間或地點（`409`），但可改一般商品描述等非交易條件；不設提案次數返還機制。 |
| **BR-08** | 只有憑證識別的本人可修改自己的資源；賣家不可向自己的商品提案或預約；賣家可看商品全部請求，買家只可看自己的請求。 |
| **BR-09** | `DIRECT` 建立即鎖定；賣家接受 `PENDING` 的 `PROPOSAL` 時同樣立即鎖定。商品與交易請求狀態更新必須原子化，同商品最多一筆 `RESERVED` 請求。 |
| **BR-10** | 鎖定後其他 `PENDING` 提案轉為 `INVALIDATED`；先成功鎖定者優先，競爭失敗回傳 `409`；歷史請求及消耗次數保留。 |
| **BR-11** | `RESERVED` 買家或賣家可取消；賣家可完成交易。取消後商品回 `AVAILABLE`，完成後商品為 `SOLD`；終止狀態不可再轉換。 |
| **BR-12** | 不設公開問答、站內留言、金流、物流、排程、帳號復原或驗證校園身分；面交依請求成立時的時間地點自行進行。 |

### 6.1 交易請求類型與狀態

| 請求類型 | 建立時狀態 | 可變更的狀態 |
|---|---|---|
| `DIRECT` | `RESERVED` | `CANCELLED`／`COMPLETED` |
| `PROPOSAL` | `PENDING` | `REJECTED`／`WITHDRAWN`／`RESERVED`／`INVALIDATED`；進入 `RESERVED` 後可變更為 `CANCELLED`／`COMPLETED` |

- `INVALIDATED` 僅用於其他請求已搶先完成鎖單（`ITEM_RESERVED`）；它是終止狀態且不返還提案次數。
- 各請求若已 `REJECTED`／`WITHDRAWN`／`INVALIDATED`／`CANCELLED`／`COMPLETED`，不得再重新開啟；後續有效操作須建立新的請求。
- 提案額度計算：同一買家及商品的 `PROPOSAL` **累計成功建立數量**不得超過 2，與是否 `PENDING` 或已結束無關；`DIRECT` 不計入次數。

### 6.2 商品狀態與同步關係

```text
Item.AVAILABLE ──(DIRECT 成立／PROPOSAL 被接受)──> Item.RESERVED
Item.RESERVED ──(取消該筆預約)───────────────> Item.AVAILABLE
Item.RESERVED ──(賣家確認完成)───────────────> Item.SOLD
```

商品與請求在同一交易中同步轉換，不容許只鎖定商品卻沒有對應 `RESERVED` 請求。同時預約時必須以原子性條件更新或同等交易控制處理競爭，而非只靠先讀取狀態判斷。

### 6.3 交易條件與時間規範

- `Item` 公開條件：`selling_price`、`meeting_time`、`meeting_place`；皆必填。
- `PROPOSAL` 可選填 `proposed_price`、`proposed_meeting_time`、`proposed_meeting_place`，至少一項與公開值不同；未提供即採公開值。
- `TradeRequest` 必須保存建立時的**有效價格、面交時間、面交地點快照**；賣家接受提案時以該快照成立預約；`DIRECT` 建立當下直接記錄公開條件。
- 時間統一 `Asia/Taipei`（UTC+08:00）；API 使用 ISO 8601／RFC 3339，例如 `2026-12-05T15:00:00+08:00`。時間需為未來時間；若提案在面交時間已過後才被接受，回傳 `409 Conflict`，賣家須處理商品公開條件後才可繼續交易。
- 不提供站內後續溝通或預約後改約；不自動到期或取消。

## 7. 非功能需求與設計限制（NFR）

| 編號 | 要求 | 規劃層面的檢查方式 |
|---|---|---|
| NFR-01 | HTTP／JSON、資源導向 RESTful API | 所有操作具有 Method、Path、狀態碼及 Schema 規劃 |
| NFR-02 | 受保護操作使用憑證與資源擁有者權限檢查 | 無效憑證為 `401`，不屬於授權對象為 `403` |
| NFR-03 | 商品鎖定與交易請求狀態具原子性 | 同商品最多一筆 `RESERVED` 請求，競爭時回傳 `409` |
| NFR-04 | 金額以正整數新臺幣處理 | 避免浮點精度誤差 |
| NFR-05 | 交易歷史不覆寫 | 取消後新預約建立新 `TradeRequest`，保留舊請求 |
| NFR-06 | Design Only 與模擬資料 | 本次不需要可執行程式、資料庫與自動化測試 |

## 8. 三個主要 Entity（概要）

| Entity | 識別／核心欄位 | 關聯 |
|---|---|---|
| `User` | `user_id`（PK）、`nickname`（UNIQUE）、`credential_hash` | 一位使用者可刊登多筆商品、提出多筆請求 |
| `Item` | `item_id`（PK）、`seller_id`（FK → User）、`name`、`category`、`description`、`department`（可空）、`course_name`（可空）、`selling_price`、`original_price`（可空）、`meeting_time`、`meeting_place`、`status` | 一筆商品有多筆歷史請求 |
| `TradeRequest` | `request_id`（PK）、`item_id`（FK → Item）、`buyer_id`（FK → User）、`request_type`、`status`、`proposed_*`（可空）、`agreed_price`、`agreed_meeting_time`、`agreed_meeting_place`、`created_at` | 一筆請求屬於一件商品與一位買家 |

`proposed_*` 代表買家原本要求更動的欄位；`agreed_*` 保存建立請求時計算出的**完整有效交易條件快照**，在 `PENDING` 階段先作為待接受條件，進入 `RESERVED` 後即為成立時約定條件。

## 9. FR → API 對應（預定）

| FR | API | 對應功能 |
|---|---|---|
| FR-01 | API-01 `POST /users` | 建立簡化身分 |
| FR-02 | API-02 `POST /items`、API-04 `PATCH /items/{itemId}` | 刊登、編輯 |
| FR-03 | API-03 `GET /items` | 商品查詢及篩選 |
| FR-04 | API-05 `POST /items/{itemId}/requests` | 直接預約或建立提案 |
| FR-05 | API-06 `GET /items/{itemId}/requests`、API-07 `PATCH /requests/{requestId}` | 查詢、接受／拒絕／撤回提案 |
| FR-06 | API-07 `PATCH /requests/{requestId}` | 取消或完成交易 |

**跨文件追溯示例：**  
`FR-04 → POST /items/{itemId}/requests → TradeRequestCreateRequest / TradeRequestResponse → TradeRequestService → Item + TradeRequest`。

## 10. 本次設計假設與一致性核對

1. 限定 **7 API／3 Entity／6 FR／12 BR**，不另增加問答、聊天室或獨立預約資料表。
2. `DIRECT`／`PROPOSAL` 共用 `TradeRequest`，商品只可能被一筆 `RESERVED` 請求鎖定。
3. 商品仍 `AVAILABLE` 且無待處理提案時才可變更公開交易條件；不返還提案額度。
4. 面交時間與地點皆必填，避免依賴系統內聊天確認後續安排。
5. `PATCH /requests/{requestId}` 只修改狀態，由原狀態、目標狀態與呼叫者身分決定是否合法；拒絕不合法轉換。
6. 本文件的 Acceptance Criteria 是**未來實作時可驗收的行為**，不是目前已有實測結果。
