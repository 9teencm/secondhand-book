# 04 — Architecture & Data Model

專案：BookSwap｜二手教科書「雙向換書」媒合
狀態：Design Only（不建立實際資料庫）

---

## 1. 分層架構

```
            HTTP Request
                 │
                 ▼
┌────────────────────────────────┐
│  Router  (接口層)               │  ← 只認識 HTTP
│  - 路由對應 operationId         │
│  - 解析 path param / body       │
│  - 依 OpenAPI schema 做格式驗證  │
│  - 把 Service 例外翻成 status    │
└───────────────┬────────────────┘
                │ 純資料物件（不含 Request/Response）
                ▼
┌────────────────────────────────┐
│  Service (業務層)               │  ← 只認識 Business Rules
│  - 執行 BR-01 ~ BR-09           │
│  - 雙向配對演算法（亮點）         │
│  - 不知道 HTTP，也不知道 SQL     │
└───────────────┬────────────────┘
                │ Entity
                ▼
┌────────────────────────────────┐
│  Repository (資料層)            │  ← 只認識儲存
│  - Entity 的查詢與寫入           │
│  - 本階段以模擬資料（in-memory）  │
│  - 不含任何業務判斷              │
└────────────────────────────────┘
```

### 各層責任

| 層 | 負責 | **不**負責 |
|---|---|---|
| **Router** | HTTP method / path 對應、反序列化、依 OpenAPI 做語法層驗證（型別、`pattern`、`enum`、`required`）、設定 status code 與 `Location` header、把 Service 丟出的錯誤對應成 `Error` schema | 任何業務判斷（例如「是否重複刊登」不在這裡判斷） |
| **Service** | 語意層驗證（成員是否存在、是否重複、狀態是否允許）、BR-04 雙向配對計算、組裝回傳用的資料結構 | 解析 HTTP、決定 status code、產生 SQL |
| **Repository** | 依條件取出 / 寫入 Entity，例如 `findAvailableListingsByIsbnIn(...)`、`existsAvailableListing(memberId, isbn)` | 判斷重複算不算錯誤（那是 Service 的事） |

**分層的實際好處（對應本題）**：BR-05 決定配對不落地。因為配對邏輯完全在 Service，將來若改成持久化 Match，只需新增一個 Repository 與改 Service，Router 與 OpenAPI 契約不必動。

### 元件清單

| Component | 層 | 職責 |
|---|---|---|
| `ListingRouter` | Router | `POST /listings`、`DELETE /listings/{listingId}` |
| `WishRouter` | Router | `POST /wishes` |
| `MatchRouter` | Router | `GET /members/{memberId}/matches` |
| `ListingService` | Service | FR-01、FR-04 與 BR-01/02/03/06/09 |
| `WishService` | Service | FR-02 與 BR-01/02/09 |
| `MatchService` | Service | FR-03 與 BR-04/05/07/08 ★ 亮點 |
| `MemberRepository` | Repository | 成員存在性查詢 |
| `ListingRepository` | Repository | Listing 的新增、查詢、狀態更新 |
| `WishRepository` | Repository | Wish 的新增與查詢 |

---

## 2. Request Flow — `GET /members/{memberId}/matches`（亮點路徑）

```
Client
  │  GET /api/v1/members/M001/matches
  ▼
MatchRouter
  │  1. 取出 path param memberId = "M001"
  │  2. 依 MemberId schema 驗證格式 ^M[0-9]{3,}$
  │     └─ 不符 → 400 VALIDATION_ERROR  (結束)
  ▼
MatchService.findMutualMatches("M001")
  │  3. MemberRepository.exists("M001")
  │     └─ false → 丟 MemberNotFound → Router 轉 404  (結束, BR-09)
  │
  │  4. myListings  = ListingRepository.findAvailableByMember("M001")
  │  5. myWishes    = WishRepository.findByMember("M001")
  │     ├─ myIsbnsIHave = { L.isbn }
  │     └─ myIsbnsIWant = { W.isbn }
  │
  │  6. 找出「想要我手上任一本書」的人：
  │     wantsMyBooks = WishRepository.findByIsbnIn(myIsbnsIHave)
  │     候選對象 candidates = { w.memberId } − {"M001"}        (BR-07)
  │
  │  7. 找出「這些人手上、而且我想要的書」：
  │     theirListings = ListingRepository
  │                       .findAvailableByMemberInAndIsbnIn(candidates, myIsbnsIWant)
  │
  │  8. 交叉組合 (BR-04)：
  │     for each (partner) in candidates:
  │       for each myL  in myListings  where partner wants myL.isbn:
  │         for each theirL in theirListings where theirL.memberId == partner:
  │           → Match{ partnerId: partner,
  │                    give:    MatchedBook(myL),
  │                    receive: MatchedBook(theirL) }        (BR-08 多組全回)
  │
  │  9. 結果不寫回資料庫                                        (BR-05)
  ▼
MatchRouter
  │ 10. 包成 MatchList{ memberId, matches[] }
  │     空陣列也照常回傳
  ▼
Client  ← 200 OK + MatchList
```

**查詢次數**：固定 5 次 Repository 呼叫，與配對組合數無關（不在迴圈內查詢）。

> `ponytail:` 第 8 步為巢狀迴圈，最壞情況 O(候選人數 × 我的書 × 對方的書)。在單一校園、每人數本書的規模下完全足夠。若清單成長到需要優化，改成以 ISBN 為鍵的 hash 分組即可，不必改 API 契約。

---

## 3. Data Model

### 3.1 ER 圖

```
┌──────────────────┐
│ Member           │
├──────────────────┤
│ PK memberId      │
│    name          │
└────┬────────┬────┘
     │ 1      │ 1
     │        │
     │ N      │ N
┌────▼─────────────┐      ┌──▼───────────────┐
│ Listing          │      │ Wish             │
├──────────────────┤      ├──────────────────┤
│ PK listingId     │      │ PK wishId        │
│ FK memberId      │      │ FK memberId      │
│    isbn          │◄─ ─ ►│    isbn          │
│    title         │ 以 isbn │    title       │
│    condition     │ 值比對  │    createdAt   │
│    status        │ (非 FK) └──────────────────┘
│    createdAt     │
└──────────────────┘
```

**Match 不在圖上**：它是 `Listing × Wish` 交叉比對的計算結果，不落地（BR-05）。

### 3.2 Entities

#### Member

| Attribute | Type | Key | 說明 |
|---|---|---|---|
| `memberId` | string | **PK** | 校內既有識別碼，格式 `^M[0-9]{3,}$`。本系統唯讀，不提供建立 API |
| `name` | string | | 顯示用 |

#### Listing（我有、可讓出的書）

| Attribute | Type | Key | 說明 |
|---|---|---|---|
| `listingId` | string | **PK** | 格式 `^L[0-9]{3,}$` |
| `memberId` | string | **FK → Member.memberId** | 刊登者 |
| `isbn` | string | 比對鍵 | ISBN-13，13 碼數字（BR-01） |
| `title` | string | | 僅供顯示，**不**參與配對比對 |
| `condition` | enum | | `NEW` / `GOOD` / `FAIR` / `POOR`（BR-06） |
| `status` | enum | | `AVAILABLE` / `REMOVED`，單向變更（BR-03） |
| `createdAt` | date-time | | |

**唯一性約束**：`(memberId, isbn)` 在 `status = AVAILABLE` 的範圍內唯一（BR-02 → 違反時 `409`）。這是部分唯一索引，`REMOVED` 的紀錄不佔名額（AC-01-4）。

#### Wish（我要的書）

| Attribute | Type | Key | 說明 |
|---|---|---|---|
| `wishId` | string | **PK** | 格式 `^W[0-9]{3,}$` |
| `memberId` | string | **FK → Member.memberId** | 登記者 |
| `isbn` | string | 比對鍵 | ISBN-13（BR-01） |
| `title` | string | | 僅供顯示 |
| `createdAt` | date-time | | |

**唯一性約束**：`(memberId, isbn)` 唯一（BR-02 → `409`）。

Wish 沒有 `status`：要取消願望就是刪掉它，不需要狀態；而本次 Out of Scope 不含刪除 Wish，所以連狀態欄位都不設。

### 3.3 Relationships

| 關係 | 基數 | 說明 |
|---|---|---|
| Member — Listing | 1 : N | 一位成員可刊登多本書 |
| Member — Wish | 1 : N | 一位成員可登記多筆願望 |
| Listing — Wish | 透過 `isbn` **值**關聯，非外鍵 | 書本沒有獨立的 Book 實體；ISBN 本身就是全球唯一書籍識別，再拉一張 Book 表只會多一次 join，不會多出任何資訊（BR-01） |

### 3.4 資料模型如何支撐已定義的 API

| API | 需要的資料能力 | 模型是否支援 |
|---|---|---|
| `POST /listings` | 檢查 Member 存在、檢查 `(memberId, isbn, AVAILABLE)` 唯一、寫入 | ✅ FK + 部分唯一索引 |
| `POST /wishes` | 檢查 Member 存在、檢查 `(memberId, isbn)` 唯一、寫入 | ✅ |
| `GET /members/{id}/matches` | 以 isbn 集合反查 Wish、以 (member 集合, isbn 集合) 反查 Listing | ✅ `isbn`、`memberId`、`status` 皆為可索引欄位 |
| `DELETE /listings/{id}` | 依 PK 取出、狀態改為 REMOVED | ✅ `status` 欄位 |

---

## 4. Traceability

### 4.1 亮點的完整追蹤鏈

| 層級 | 內容 |
|---|---|
| **FR** | [FR-03](01-requirements.md#5-functional-requirements) — 查詢雙向互換配對清單 |
| **Business Rule** | [BR-04](01-requirements.md#6-business-rules)（四條件同時成立）、BR-05（不持久化）、BR-07（排除自己）、BR-08（多組全回） |
| **Acceptance Criteria** | AC-03-1 ~ AC-03-6 |
| **API** | [`GET /api/v1/members/{memberId}/matches`](02-api-design.md#23-查詢雙向互換配對清單--亮點) → `200` / `404` |
| **OpenAPI** | `operationId: listMutualMatches`；response `$ref: MatchList` → `Match` → `MatchedBook` → `Isbn` / `BookCondition`；錯誤 `$ref: MemberNotFound` → `Error` |
| **Component** | `MatchRouter` → `MatchService.findMutualMatches()` → `ListingRepository` + `WishRepository` + `MemberRepository` |
| **Entity** | `Listing`（`memberId`, `isbn`, `status=AVAILABLE`）× `Wish`（`memberId`, `isbn`），以 `isbn` 值交叉比對 |

### 4.2 其餘需求對照表

| FR | API | OpenAPI operationId | 主要 Schema | Component | Entity |
|---|---|---|---|---|---|
| FR-01 | `POST /listings` | `createListing` | `ListingCreateRequest` → `Listing` | `ListingRouter` → `ListingService` → `ListingRepository` | Listing |
| FR-02 | `POST /wishes` | `createWish` | `WishCreateRequest` → `Wish` | `WishRouter` → `WishService` → `WishRepository` | Wish |
| FR-03 ★ | `GET /members/{memberId}/matches` | `listMutualMatches` | `MatchList` / `Match` / `MatchedBook` | `MatchRouter` → `MatchService` → 三個 Repository | Listing × Wish |
| FR-04 | `DELETE /listings/{listingId}` | `removeListing` | 無 body；錯誤用 `Error` | `ListingRouter` → `ListingService` → `ListingRepository` | Listing |

**反向檢查**：每個 Entity 都被至少一條 FR 使用；每個 Component 都在某條追蹤鏈上；OpenAPI 的 14 個 schema 全部被某個 path 直接或間接 `$ref` 到，沒有孤兒定義。

---

## 5. 實作前待確認事項（交給未來的 AI / 實作者）

1. `Member` 資料從何處匯入？本次假設為既有來源，未定義同步機制。
2. ISBN-13 是否需要檢查碼（checksum）驗證？目前 BR-01 只要求 13 碼數字。
3. BR-08 回傳多組配對時是否需要排序（例如依 `createdAt`）？目前未定義順序。
4. 配對清單是否需要分頁？目前假設單一成員的配對數量不足以需要分頁。
