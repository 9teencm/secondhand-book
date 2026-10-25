# BookSwap｜二手教科書「雙向換書」媒合

> **Design Only — 本專案目前僅有規劃與設計文件，尚未實作程式、資料庫或測試。**

## 一句話價值主張

不用逐則翻找社團貼文，系統直接告訴你「誰手上的書剛好是你要的、而你手上的書也剛好是他要的」，一次換成。

## 使用者與問題

**使用者**：同校的大學生，學期初要找下學期課本、學期末想出清上學期課本。

**問題**：目前的管道是社團或群組貼文。貼文是**單向**的（只寫「出售」或只寫「徵求」），想換書的人必須自己把兩邊的貼文在腦中配對，而且要翻很多則才會偶然撞到一組。多數「其實可以互換」的組合，雙方根本不知道對方存在。

## 系統亮點

**自動找出雙向可成交的配對（Mutual Match）。**

一般的二手書平台只能做到「搜尋有沒有人在賣我要的書」——那是單向查詢。本系統同時比對兩個方向：

> 我有《資料結構》、我要《作業系統》；小明有《作業系統》、小明要《資料結構》
> → 系統主動把我和小明配成一組，雙方直接換書。

這個亮點不是靠介面或 AI，而是靠資料模型支撐：每位成員同時維護 `Listing`（我有）與 `Wish`（我要）兩份清單，配對條件是兩份清單的**交叉比對**（見 [Business Rules BR-04](docs/01-requirements.md#business-rules)）。因為是以物易物，所以**不需要金流**。

## 核心使用流程（5 步）

1. 小美刊登她要讓出的書：《資料結構》(ISBN 9789861579993)
2. 小美登記她想要的書：《作業系統》(ISBN 9780133591620)
3. 小明做同樣的事，方向剛好相反
4. 小美查詢自己的配對清單，系統回傳「小明 — 可互換」，並列出雙方各自要給出與拿到的書
5. 線下換書完成後，小美把自己的刊登下架

## 本次不做什麼（Out of Scope）

| 不做 | 理由 |
|---|---|
| 註冊、登入、密碼、權限 | 題目範圍限制；成員以 `memberId` 直接識別 |
| 金流、定價、議價 | 以物易物模式本來就不需要 |
| 站內聊天、通知推播 | 聯絡方式交給使用者自行處理，不是本題核心 |
| 評價、信用分數、檢舉 | 屬於平台治理，會把題目撐成大型平台 |
| 三方以上的循環換書（A→B→C→A） | 演算法複雜度大幅上升，與「小而美」衝突 |
| 書況照片上傳、全文搜尋 | 以 ISBN 比對即可達成亮點 |
| 程式碼、資料庫、可執行 API、自動化測試 | 本次作業為設計階段 |

## 文件導覽（建議閱讀順序）

| # | 文件 | 內容 |
|---|---|---|
| 1 | [docs/01-requirements.md](docs/01-requirements.md) | 問題、使用者、情境、範圍、FR、Business Rules、Acceptance Criteria |
| 2 | [docs/02-api-design.md](docs/02-api-design.md) | Resources、Paths、HTTP Methods、Status Codes、FR↔API 對應 |
| 3 | [docs/03-openapi.yaml](docs/03-openapi.yaml) | OpenAPI 3.1.0 契約 |
| 4 | [docs/04-architecture-data-model.md](docs/04-architecture-data-model.md) | 分層責任、Request Flow、Entity、Relationship、Traceability |
| 5 | [docs/05-ai-collaboration.md](docs/05-ai-collaboration.md) | AI 協作過程與自己的判斷修正 |

## 系統規模

- **Entities**：3（Member、Listing、Wish）
- **API Operations**：4
- **Match 不是 Entity**：配對結果每次查詢即時計算，不落地儲存（取捨理由見 [BR-05](docs/01-requirements.md#business-rules)）
