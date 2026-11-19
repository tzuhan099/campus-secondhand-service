# 02｜RESTful API 設計（RESTful API Design）

> **專案：** 校園二手物交易與預約平台  
> **狀態：** Design Only  

## 1. 設計原則與共通約定

- **Base URL：** `/api/v1`。下列路徑表格省略前綴，實際呼叫需加上 `/api/v1`。
- **資料格式：** HTTP／JSON；以資源名詞 `users`、`items`、`requests` 為路徑，HTTP Method 表示操作。
- **身分識別：** `POST /users` 不需驗證；其餘寫入及交易請求查詢採 `Authorization: Bearer <opaque-token>`，Token 為高熵隨機字串，伺服器只保存雜湊；`GET /items` 可匿名。
- **日期時間：** 統一臺北時間 `Asia/Taipei`（UTC+08:00），API 欄位為 RFC 3339 字串，例如 `2026-12-05T15:00:00+08:00`；面交時間必填且未來有效。
- **金額：** 正整數新臺幣（integer）；`original_price` 可省略或傳 `null`，不做查證。
- **狀態欄位：** `Item.status` 為 `AVAILABLE`／`RESERVED`／`SOLD`；`TradeRequest.status` 為 `PENDING`／`REJECTED`／`WITHDRAWN`／`INVALIDATED`／`RESERVED`／`CANCELLED`／`COMPLETED`。
- **代碼慣例：** 成功新增 `201`、查詢／更新 `200`、輸入或欄位校驗失敗 `422`、未驗證 `401`、無權限 `403`、資源不存在 `404`、狀態／唯一性／額度衝突 `409`；未預期伺服器錯誤 `500`。

## 2. API 總覽（7 Operations）

| ID | Resource | Method | Path | 用途 | FR |
|---|---|---|---|---|---|
| API-01 | User | `POST` | `/users` | 建立暱稱身分與操作憑證 | FR-01 |
| API-02 | Item | `POST` | `/items` | 刊登商品 | FR-02 |
| API-03 | Item | `GET` | `/items` | 商品查詢、分類／課程篩選 | FR-03 |
| API-04 | Item | `PATCH` | `/items/{itemId}` | 編輯商品或公開交易條件 | FR-02 |
| API-05 | TradeRequest | `POST` | `/items/{itemId}/requests` | 建立直接預約或交易提案 | FR-04 |
| API-06 | TradeRequest | `GET` | `/items/{itemId}/requests` | 查看該商品有權存取的請求 | FR-05 |
| API-07 | TradeRequest | `PATCH` | `/requests/{requestId}` | 接受／拒絕／撤回提案、取消／完成預約 | FR-05、FR-06 |

**只有這 7 條 API。** 商品公開詳情由 `GET /items` 的完整列表資料提供，可用 `item_id` 在結果中找到指定商品；不另提供 `GET /items/{itemId}`。交易請求的單筆詳情由 `GET /items/{itemId}/requests` 所回傳清單提供，不另增加 `GET /requests/{requestId}`。

## 3. API 詳細設計

### API-01｜`POST /users`

- **用途：** 建立全站唯一暱稱，取得後續操作憑證。
- **驗證：** 不需要。
- **Request Body：** `{ "nickname": "campus_user" }`；`nickname` 為非空白字串（建議長度 2–30 字元）。
- **成功：** `201 Created`，回傳 `user_id`、`nickname`、`access_token`。Token 僅此時顯示，後續不再提供明文查詢。
- **失敗：** `409` 暱稱重複；`422` 暱稱欄位無效；`500` 系統錯誤。
- **對應：** FR-01／BR-01。

### API-02｜`POST /items`

- **用途：** 賣家刊登商品，預設狀態 `AVAILABLE`。
- **驗證：** 必須，系統由憑證取得 `seller_id`，不得由請求指定別人的身分。
- **Request Body：**

```json
{
  "name": "二手電風扇",
  "category": "APPLIANCE",
  "description": "功能正常，使用一年",
  "selling_price": 500,
  "original_price": null,
  "meeting_time": "2026-12-05T15:00:00+08:00",
  "meeting_place": "校園圖書館一樓",
  "department": null,
  "course_name": null
}
```

- **必填：** `name`、`category`、`description`、`selling_price`、`meeting_time`、`meeting_place`；`original_price`、`department`、`course_name` 選填。
- **成功：** `201 Created`，回傳建立的 Item（含 `item_id`、`seller_id`、`status: "AVAILABLE"`）。
- **失敗：** `401` 缺失／無效憑證；`422` 欄位錯誤、非正數金額、非未來面交時間；`500` 系統錯誤。
- **對應：** FR-02／BR-02～BR-04。

### API-03｜`GET /items`

- **用途：** 查詢可公開的商品資料與篩選。
- **驗證：** 不需要。
- **可選 Query Parameters：** `keyword`（商品名稱包含關鍵字）、`category`（三分類 enum）、`department`、`course_name`、`status`（`AVAILABLE`／`RESERVED`／`SOLD`；不填預設 `AVAILABLE`）。條件同時指定時採 AND。
- **成功：** `200 OK`，回傳 `items` 陣列，每筆包含 Item 公開欄位：`item_id`、`seller_id`、`name`、`category`、`description`、`department`、`course_name`、`selling_price`、`original_price`、`price_ratio_percent`（有原價時為 `selling_price / original_price × 100`，否則 `null`）、`meeting_time`、`meeting_place`、`status`。不回傳憑證、交易提案或其他私人資訊。
- **失敗：** `422` 篩選條件值不合法；`500` 系統錯誤。
- **對應：** FR-03／BR-02、BR-03。

### API-04｜`PATCH /items/{itemId}`

- **用途：** 賣家修改自己的 `AVAILABLE` 商品；可局部修改欄位，未提供的欄位保留原值。
- **驗證：** 必須，僅商品賣家可操作。
- **Path：** `itemId`（正整數）。
- **Request Body：** 可包含 `name`、`category`、`description`、`selling_price`、`original_price`、`meeting_time`、`meeting_place`、`department`、`course_name` 中一項或多項；不允許變更 `item_id`、`seller_id`、`status`。例如：

```json
{
  "selling_price": 480,
  "meeting_time": "2026-12-06T14:00:00+08:00"
}
```

- **成功：** `200 OK`，回傳更新後 Item。
- **失敗：** `401` 無效憑證；`403` 非賣家；`404` 商品不存在；`409` 商品非 `AVAILABLE`，或**有任何 `PENDING` 提案時試圖變更公開價格／面交時間／地點**；`422` 欄位無效；`500` 系統錯誤。
- **特別規則：** 如果只是修改描述、名稱、分類、系所或課程等非交易條件，存在 `PENDING` 提案仍可編輯。沒有提案次數返還或賣家改條件自動失效的流程。
- **對應：** FR-02／BR-07、BR-08。

### API-05｜`POST /items/{itemId}/requests`

- **用途：** 共用同一個交易請求資源建立 `DIRECT`（直接預約）或 `PROPOSAL`（交易條件提案）。
- **驗證：** 必須；非商品賣家的買家才能建立。
- **Path：** `itemId`（正整數）。

**A. 直接預約 Request**

```json
{ "request_type": "DIRECT" }
```

- 必須使用目前公開售價、面交時間、地點，不能夾帶調整條件。
- 只有商品 `AVAILABLE` 時可成功建立；`TradeRequest.status` **立即設為 `RESERVED`**，`Item.status` 同時轉為 `RESERVED`。

**B. 正式交易提案 Request**

```json
{
  "request_type": "PROPOSAL",
  "proposed_price": 450,
  "proposed_meeting_time": "2026-12-06T14:00:00+08:00",
  "proposed_meeting_place": "學生活動中心門口"
}
```

- `proposed_price`、`proposed_meeting_time`、`proposed_meeting_place` 可各自選填；至少一項應與當時公開條件不同；未指定的採商品公開值。
- 改價僅允許 0 < `proposed_price` < `selling_price`。面交時間必須是未來，地點不得空白。
- 每位買家對同一商品累計最多兩筆 `PROPOSAL`，同時最多一筆自己的 `PENDING`。成功建立後狀態為 `PENDING`，不鎖定商品。

**共同成功回應：** `201 Created`，回傳 `TradeRequestResponse`，包含 `request_id`、`item_id`、`buyer_id`、`request_type`、`status`、`proposed_*`、`agreed_price`、`agreed_meeting_time`、`agreed_meeting_place`、`created_at`。`agreed_*` 為計算後的有效條件快照，在 `PENDING` 階段代表待接受的條件，`RESERVED` 階段代表成立條件。

**失敗：** `401` 無效憑證；`403` 賣家操作自己商品；`404` 商品不存在；`409` 商品非 `AVAILABLE`、同買家已有 `PENDING` 或提案額度用盡、鎖單衝突；`422` 缺失或不合理的提案值；`500` 系統錯誤。

**對應：** FR-04／BR-05、BR-06、BR-08～BR-10。

### API-06｜`GET /items/{itemId}/requests`

- **用途：** 查看某件商品已建立的交易請求；不額外提供通用歷史查詢 API。
- **驗證：** 必須。
- **Path：** `itemId`（正整數）。
- **成功：** `200 OK`，回傳 `requests` 陣列（`TradeRequestResponse`）。商品賣家可見該商品所有請求；其他使用者只看自己提出的請求（可能為空陣列）。
- **失敗：** `401` 無效憑證；`404` 商品不存在；`500` 系統錯誤。
- **權限：** 不得把他人提案返回非當事人的買家；回應不含任何憑證。
- **對應：** FR-05／BR-08。

### API-07｜`PATCH /requests/{requestId}`

- **用途：** 依當前狀態修改交易請求狀態。統一使用 `{ "status": "目標狀態" }`；不是任意可寫欄位。
- **驗證：** 必須。
- **Path：** `requestId`（正整數）。
- **Request Body 範例：** `{ "status": "RESERVED" }`（賣家接受提案）；其他允許的目標為 `REJECTED`、`WITHDRAWN`、`CANCELLED`、`COMPLETED`。

| 目前狀態 | 目標狀態 | 誰可以操作 | 同時產生的效果 |
|---|---|---|---|
| `PENDING`（僅 `PROPOSAL`） | `RESERVED` | 商品賣家 | 原子性鎖單，Item → `RESERVED`，其餘 `PENDING` → `INVALIDATED` |
| `PENDING` | `REJECTED` | 商品賣家 | 提案終止；保留已使用次數 |
| `PENDING` | `WITHDRAWN` | 該提案買家 | 提案終止；保留已使用次數 |
| `RESERVED` | `CANCELLED` | 該請求買家或商品賣家 | Item → `AVAILABLE`，保留歷史 |
| `RESERVED` | `COMPLETED` | 商品賣家 | Item → `SOLD` |

- **成功：** `200 OK`，回傳更新後 `TradeRequestResponse`。
- **失敗：** `401` 無效憑證；`403` 非合法操作者；`404` 請求不存在；`409` 狀態轉換不允許、面交時間已過、鎖單競爭失敗；`422` 目標狀態不合法；`500` 系統錯誤。
- **限制：** 用戶端不能直接指定 `INVALIDATED`；它僅由成功鎖單流程批次產生。`DIRECT` 在建立時已 `RESERVED`，不需要額外接受步驟。終止狀態不可重新開啟。
- **對應：** FR-05、FR-06／BR-09～BR-11。

## 4. HTTP Status Codes 與統一錯誤格式

| HTTP Code | 使用時機 | 例子 |
|---|---|---|
| `200 OK` | 查詢或更新成功 | `GET /items`、`PATCH /requests/{requestId}` |
| `201 Created` | 新資源建立成功 | 建立 User、Item、TradeRequest |
| `401 Unauthorized` | 缺少或無效操作憑證 | 未帶 Bearer Token 卻建立提案 |
| `403 Forbidden` | 操作人身分合法但無權限 | 非賣家修改商品；賣家預約自己商品 |
| `404 Not Found` | 指定資源不存在 | `itemId` 或 `requestId` 不存在 |
| `409 Conflict` | 已存在資源／業務狀態衝突 | 暱稱重複、提案額度用盡、重複鎖單、提案待處理時改公開條件 |
| `422 Unprocessable Content` | Body／參數格式或值不合法 | 低於等於零的售價、不合法分類或時間 |
| `500 Internal Server Error` | 未預期系統錯誤 | 儲存失敗 |

**ErrorResponse 範例：**

```json
{
  "error": {
    "code": "ITEM_ALREADY_RESERVED",
    "message": "商品已被其他交易請求鎖定"
  }
}
```

錯誤 `code` 建議值：`NICKNAME_CONFLICT`、`INVALID_INPUT`、`UNAUTHORIZED`、`FORBIDDEN`、`NOT_FOUND`、`ITEM_ALREADY_RESERVED`、`PENDING_REQUEST_EXISTS`、`PROPOSAL_LIMIT_REACHED`、`ITEM_TERMS_LOCKED`、`INVALID_STATUS_TRANSITION`、`MEETING_TIME_PASSED`、`INTERNAL_ERROR`。同一語意的錯誤代碼應在未來 OpenAPI 契約一致使用。

## 5. 主要流程與一致性

### 5.1 直接預約

```text
POST /items/{itemId}/requests {"request_type":"DIRECT"}
  → 檢查商品 AVAILABLE 與非本人商品
  → 原子性建立 TradeRequest (DIRECT, RESERVED)，鎖定 Item (RESERVED)
  → 同步 INVALIDATED 其他 PENDING 提案
  → 201 Created；競爭失敗 → 409 Conflict
```

### 5.2 提案與賣家接受

```text
POST /items/{itemId}/requests {"request_type":"PROPOSAL", ...}
  → 確認 AVAILABLE、同買家 PENDING 不重複、累計提案 < 2
  → 建立 TradeRequest (PROPOSAL, PENDING)，保存條件快照
  → 201 Created

PATCH /requests/{requestId} {"status":"RESERVED"}（賣家）
  → 檢查 PENDING、商品 AVAILABLE、面交時間尚未過去
  → 原子性將該請求／商品改為 RESERVED
  → 其他 PENDING → INVALIDATED
  → 200 OK；鎖單競爭失敗 → 409 Conflict
```

### 5.3 編輯商品時的交易條件保護

```text
PATCH /items/{itemId}
  → 檢查賣家身分及商品 AVAILABLE
  → 若修改 selling_price／meeting_time／meeting_place 且存在 PENDING 提案
      → 409 Conflict，請先處理待處理提案
  → 否則更新商品；不返還也不重置提案次數
```

### 5.4 取消與完成

```text
PATCH /requests/{requestId} {"status":"CANCELLED"}
  → 預約買家／賣家可取消 RESERVED
  → 同一交易：TradeRequest → CANCELLED，Item → AVAILABLE

PATCH /requests/{requestId} {"status":"COMPLETED"}
  → 僅賣家可確認 RESERVED
  → 同一交易：TradeRequest → COMPLETED，Item → SOLD
```

取消後歷史請求不恢復；買家若要重新預約，建立新 `DIRECT` 或在仍有額度的情況下建立新 `PROPOSAL`。平台不提供站內協調或後續改約。

## 6. FR–API Mapping（需求追溯）

| FR | 主要 Operation | 補充 |
|---|---|---|
| FR-01 建立使用者 | API-01 | 其餘受保護操作以 Bearer Token 識別 |
| FR-02 刊登與編輯 | API-02、API-04 | 資料欄位、狀態與 PENDING 禁改規則 |
| FR-03 商品查詢 | API-03 | Query 篩選與價格資訊 |
| FR-04 建立交易請求 | API-05 | `DIRECT` 或 `PROPOSAL` |
| FR-05 查詢與處理請求 | API-06、API-07 | 當事人存取、接受／拒絕／撤回 |
| FR-06 取消與完成 | API-07 | 請求與商品狀態同步更新 |

## 7. 後續 OpenAPI／Architecture 對應

| REST Resource | 預計 OpenAPI Schemas | Service／Repository | Entity |
|---|---|---|---|
| `users` | `CreateUserRequest`、`CreateUserResponse` | `UserService`／`UserRepository` | `User` |
| `items` | `CreateItemRequest`、`UpdateItemRequest`、`ItemResponse`、`ItemListResponse` | `ItemService`／`ItemRepository` | `Item` |
| `requests` | `TradeRequestCreateRequest`、`TradeRequestUpdateRequest`、`TradeRequestResponse`、`TradeRequestListResponse` | `TradeRequestService`／`TradeRequestRepository`（並協調 Item） | `TradeRequest`、`Item` |
| 共通 | `ErrorResponse`、Bearer Token Security Scheme | Router／Error Handler | — |

**一條追溯示例：**  
`FR-04 → API-05 POST /items/{itemId}/requests → TradeRequestCreateRequest / TradeRequestResponse → TradeRequestService → ItemRepository + TradeRequestRepository → Item + TradeRequest`。

## 8. 一致性檢查與設計取捨

- API 數量固定 **7 條**；不增設獨立問答、留言、預約、登入／登出 API。
- Entity 固定 **3 個**：`User`、`Item`、`TradeRequest`；`Offer` 和 `Reservation` 統一為 `TradeRequest`。
- 正式提案上限 **2 次**，不返還；`PENDING` 期間賣家不得變更公開售價／面交時間／地點。
- 面交**日期時間與地點都必填**；所有外部日期時間統一臺北時間。
- 接受提案與直接預約都會產生有效鎖單，並使其他 `PENDING` 提案失效。
- `PATCH /requests/{requestId}` 只負責合法狀態轉換，不能直接改寫交易金額與面交條件。
- 所有驗收及鎖單描述屬**設計規劃**，尚未進行實作或實測。
