# 02 — RESTful API Design

專案：BookSwap｜二手教科書「雙向換書」媒合
狀態：Design Only
Base URL：`/api/v1`

---

## 1. Resources

| Resource | 說明 | Collection Path |
|---|---|---|
| `Member` | 成員（唯讀，由校內既有資料提供，本系統不建立） | `/members` |
| `Listing` | 一筆「我有、可讓出」的書 | `/listings` |
| `Wish` | 一筆「我要」的書 | `/wishes` |
| `Match` | 雙向互換配對結果。**不是獨立資源**，是 Member 底下的衍生子資源，每次查詢即時計算（BR-05） | `/members/{memberId}/matches` |

**命名原則**

- Path 一律以**名詞複數**表示資源集合，動作由 HTTP Method 表達（沒有 `/createListing`、`/doMatch` 這種動詞路徑）
- `matches` 掛在 `/members/{memberId}` 之下，因為「配對」只有在指定某位成員的立場時才有意義（誰給出、誰拿到是相對的）
- 跨文件欄位命名一律 `camelCase`，與 [03-openapi.yaml](03-openapi.yaml) 的 schema 屬性名完全一致

---

## 2. API Operations

共 4 條。

### 2.1 建立刊登（我有這本書）

| 項目 | 內容 |
|---|---|
| **Resource** | Listing |
| **Method / Path** | `POST /api/v1/listings` |
| **用途** | 成員刊登一本可讓出的教科書，建立後狀態為 `AVAILABLE`，即刻納入配對計算 |
| **對應 FR** | FR-01 |
| **Request Body** | `ListingCreateRequest`：`memberId`, `isbn`, `title`, `condition` |
| **Response Body** | `Listing` |

| Status | 時機 | 來源規則 |
|---|---|---|
| `201 Created` | 建立成功，附 `Location: /api/v1/listings/{listingId}` | FR-01 |
| `400 Bad Request` | ISBN 非 13 碼數字、`condition` 不在列舉內、必填欄位缺漏 | BR-01, BR-06 |
| `404 Not Found` | `memberId` 不存在 | BR-09 |
| `409 Conflict` | 該成員對同一 ISBN 已有 `AVAILABLE` 的 Listing | BR-02 |

---

### 2.2 建立願望（我要這本書）

| 項目 | 內容 |
|---|---|
| **Resource** | Wish |
| **Method / Path** | `POST /api/v1/wishes` |
| **用途** | 成員登記一本想取得的教科書 |
| **對應 FR** | FR-02 |
| **Request Body** | `WishCreateRequest`：`memberId`, `isbn`, `title` |
| **Response Body** | `Wish` |

| Status | 時機 | 來源規則 |
|---|---|---|
| `201 Created` | 建立成功，附 `Location: /api/v1/wishes/{wishId}` | FR-02 |
| `400 Bad Request` | ISBN 格式錯誤、必填欄位缺漏 | BR-01 |
| `404 Not Found` | `memberId` 不存在 | BR-09 |
| `409 Conflict` | 該成員對同一 ISBN 已有 Wish | BR-02 |

> Wish 沒有 `condition` 欄位：想要一本書的人不會先指定書況，加了也沒有規則會用到它。

---

### 2.3 查詢雙向互換配對清單 ★ 亮點

| 項目 | 內容 |
|---|---|
| **Resource** | Match（Member 的衍生子資源） |
| **Method / Path** | `GET /api/v1/members/{memberId}/matches` |
| **用途** | 以 `{memberId}` 的立場，回傳所有雙向可成交的配對。每筆明確指出「對方是誰、我給出哪本、我拿到哪本」 |
| **對應 FR** | FR-03 |
| **Path Parameter** | `memberId`（string，required） |
| **Response Body** | `MatchList`：`memberId` + `matches[]` |

| Status | 時機 | 來源規則 |
|---|---|---|
| `200 OK` | 查詢成功。**沒有任何配對時仍回 `200` 與空陣列**，不是 `404` — 「查得到但目前沒結果」與「成員不存在」是兩回事 | FR-03, BR-04 |
| `404 Not Found` | `memberId` 不存在 | BR-09 |

**為什麼是 `GET` 而不是 `POST /matches`**：這是一次查詢，不產生任何伺服器端狀態（BR-05 配對不落地），具冪等性與可快取性，語意上就是 `GET`。

---

### 2.4 下架刊登

| 項目 | 內容 |
|---|---|
| **Resource** | Listing |
| **Method / Path** | `DELETE /api/v1/listings/{listingId}` |
| **用途** | 將刊登狀態改為 `REMOVED`，使其退出所有配對計算（書已換出或不想換了） |
| **對應 FR** | FR-04 |
| **Path Parameter** | `listingId`（string，required） |
| **Response Body** | 無 |

| Status | 時機 | 來源規則 |
|---|---|---|
| `204 No Content` | 下架成功 | FR-04 |
| `404 Not Found` | `listingId` 不存在，**或**該 Listing 已是 `REMOVED` | BR-03 |

**為什麼用 `DELETE` 而不是 `PATCH .../status`**：對使用者而言這就是「把這筆刊登拿掉」。底層採 soft delete（保留紀錄、狀態改為 `REMOVED`）是儲存層的實作選擇，不應該外洩到 API 語意。`DELETE` 具冪等性，重複呼叫的結果一致（資源不存在 → `404`）。

---

## 3. Requirement ↔ API Mapping

| FR | 需求 | API Operation | 主要 Status Codes |
|---|---|---|---|
| FR-01 | 刊登可讓出的教科書 | `POST /api/v1/listings` | 201 / 400 / 404 / 409 |
| FR-02 | 登記想要的教科書 | `POST /api/v1/wishes` | 201 / 400 / 404 / 409 |
| FR-03 | 查詢雙向互換配對清單 ★ | `GET /api/v1/members/{memberId}/matches` | 200 / 404 |
| FR-04 | 下架刊登 | `DELETE /api/v1/listings/{listingId}` | 204 / 404 |

**覆蓋檢查**：4 條 FR 全部有對應 API；4 條 API 全部對應到某條 FR，沒有「設計了但沒人要求」的多餘端點。

---

## 4. 共用錯誤格式

所有 `4xx` 回應使用同一個 `Error` schema（在 OpenAPI 中以 `$ref` 共用）：

```json
{
  "code": "DUPLICATE_LISTING",
  "message": "Member M001 already has an available listing for ISBN 9789861579993."
}
```

| `code` | HTTP Status | 使用時機 |
|---|---|---|
| `VALIDATION_ERROR` | 400 | 欄位格式或列舉值不合法（BR-01, BR-06） |
| `MEMBER_NOT_FOUND` | 404 | `memberId` 不存在（BR-09） |
| `LISTING_NOT_FOUND` | 404 | `listingId` 不存在或已下架（BR-03） |
| `DUPLICATE_LISTING` | 409 | 同成員同 ISBN 已有 AVAILABLE Listing（BR-02） |
| `DUPLICATE_WISH` | 409 | 同成員同 ISBN 已有 Wish（BR-02） |

---

## 5. 一致性自我檢查

- [x] 每條 Path 都是名詞，沒有動詞
- [x] 建立用 `POST` 並回 `201` + `Location`；查詢用 `GET` 回 `200`；移除用 `DELETE` 回 `204`
- [x] 空結果回 `200 + []`，不回 `404`
- [x] `409` 只用於「違反唯一性」，格式錯誤一律 `400`
- [x] 所有錯誤共用同一個 `Error` schema
- [x] 本文件的欄位名稱與 [03-openapi.yaml](03-openapi.yaml)、[04-architecture-data-model.md](04-architecture-data-model.md) 逐字一致
