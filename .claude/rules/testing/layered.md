# Testing — Clean / Onion 分層

> 先讀 [../testing.md](../testing.md) 的共用原則。本檔只寫這個架構特有的部分。

## 各層策略

| 層 | 策略 |
| :--- | :--- |
| **Domain Model** | 直接單元測試（業務行為的核心所在） |
| **Domain Service** | **不為它定介面**，以具體實例注入。跨物件編排較重的 service 另寫專屬測試，直接注入 mock 的 repository / proxy |
| **Application** | 注入**真實的 domain service（連帶真實 domain model）**，**只 mock 最外層的介面** |

Application 測試會**連帶測到 domain service 與 domain model**——不要為了隔離而 mock domain service。

## 黑箱視角

測試檔一律從**外部視角**（黑箱）import 受測目標，只測公開行為。

## 測試放置位置

**依語言慣例放置，但一律與受測 package/專案分離**：

- 語言支援同 repo 分層目錄的（如 Go）：各層底下開 `tests/` 子資料夾，測試檔用外部 `<pkg>_test` package，`testdata/` 一併放進去。
- 語言慣用獨立測試專案的（如 .NET / TS）：獨立測試專案，目錄結構鏡射受測專案。

mock 由 mocking 工具產生，放 `domain/interface/` 的子資料夾，沿用工具預設命名。

檔名 / package 命名 follow 該語言慣例。
