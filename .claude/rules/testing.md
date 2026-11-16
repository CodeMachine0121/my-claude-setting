# Testing — 入口

本檔只做兩件事：

1. 告訴你**這個專案的測試該讀哪一份**。
2. 列出**不論哪種架構都成立**的測試原則。

## 選測試做法

**跟著專案的架構走**（架構怎麼選見 [architecture.md](architecture.md)）：

| 架構 | 讀哪份 |
| :--- | :--- |
| Clean / Onion 分層 | [testing/layered.md](testing/layered.md) |
| Go Package-by-Feature | [testing/go-package-by-feature.md](testing/go-package-by-feature.md) |
| 前端 Feature-based | [testing/frontend-feature-based.md](testing/frontend-feature-based.md) |

## 共用原則（所有架構都成立）

### 只測業務行為

- 驗證「給定輸入 → 得到什麼業務結果」，不驗證內部呼叫了哪些私有方法、欄位怎麼存。
- 只透過**公開 method** 驅動受測目標。重構內部實作時，測試不該跟著壞。
- 不為框架、ORM、第三方 SDK 本身寫測試。
- 一律 **table-driven / 參數化測試**，一個案例一組輸入輸出。正常流程與邊界錯誤都要有。

### Mocking：只 mock 介面，且只用 mocking 套件

- **一律用 mocking 套件對「介面」產生替身**（testify/mock、mockery、NSubstitute、Moq、Jest mock…）。
- **禁止手寫假物件**——不准出現 `FakeDbContext`、`InMemoryXxxRepository`、`StubPaymentProxy` 這類自己刻邏輯的假實作。
- 需要被 mock 的東西一定要先有介面。沒有介面就先補介面，不要為了測試改用具體型別。
- mock 只設在**最外層的邊界**：repository / proxy（DB、外部 API、時間、亂數）。

### 內部邏輯用真的

- 內部的業務元件（domain service、domain model、無狀態計算元件）**注入真實實例**，不 mock。
- 這會連帶測到下層邏輯——是刻意的「**測試力度放大**」，也省下過度 mock 的維護成本。
