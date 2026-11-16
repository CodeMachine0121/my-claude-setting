# Architecture — Go Package-by-Feature（業務領域分層）

> 先讀 [../architecture.md](../architecture.md) 的共用原則（依賴方向、禁 static、三步搬家）。本檔只寫這個架構特有的部分。

適用：產品 / 平台等級的 Go API 專案。

**同一個業務模組的資料模型、商業邏輯、介面定義與測試集中在同一個資料夾**。高內聚、低耦合，不用跨資料夾跳轉，也避免循環依賴。

## 目錄結構

```text
my-api-project/
├── api/                    # API 規格（OpenAPI / Swagger、Proto）
├── build/                  # 打包與 CI/CD（Dockerfile、pipeline）
├── cmd/
│   └── server/
│       └── main.go         # 組裝根：讀設定、建連線、組 DI、啟動 Gin
├── configs/                # 設定檔範本（config.yaml.example）
├── internal/               # 私有程式碼（編譯器保護，外部不可 import）
│   ├── user/               # 【業務領域模組】
│   │   ├── entity.go       # 乾淨的資料模型：只有欄位與 ORM tag
│   │   ├── domain.go       # Domain Model（UserDomain）：業務行為住這裡
│   │   ├── dto.go          # HTTP 進出的資料物件（XxxRequest / XxxDto）
│   │   ├── repository.go   # 對外資料存取的介面
│   │   ├── repository_mock.go # 介面的 mock（testify/mock）
│   │   ├── usecase.go      # 商業邏輯（struct + method）
│   │   ├── usecase_test.go # UseCase 測試
│   │   └── delivery.go     # Gin handler 與路由註冊
│   ├── order/
│   │   └── ...
│   └── infrastructure/     # 基礎設施實作
│       └── mysql/
│           └── user_repository.go
├── pkg/                    # 可對外共用的套件（非必要就不開，避免變雜物間）
├── test/                   # 整合測試、E2E 測試
├── go.mod
├── go.sum
└── README.md
```

## 角色

| 檔案 | 角色 | 規則 |
| :--- | :--- | :--- |
| `entity.go` | Entity（資料模型） | **只有欄位與 ORM tag**，不放業務邏輯。可帶 `ToDomain()` / `ToDto()` 這類形狀轉換 |
| `domain.go` | Domain Model | `XxxDomain` 後綴。**所有計算、驗證、狀態轉換都放這裡**。用 `NewXxxDomain(...)` 建立，正規化 / clamp 寫在裡面。不為它定介面 |
| `dto.go` | 進出形狀 | 只有資料。命名照 [../naming.md](../naming.md)（`Request` / `Dto`） |
| `repository.go` | 對外介面 | 由**消費端（UseCase）**定義 |
| `usecase.go` | 商業邏輯 | **struct + method**，用 `NewXxxUseCase(...)` 注入依賴與設定 |
| `delivery.go` | HTTP 入口 | **struct + method**。只做 DTO 綁定 / 驗證、把業務錯誤轉成 HTTP 狀態碼 |
| `cmd/server/main.go` | 組裝根 | 啟動時由下往上組一次：`Repository → UseCase → Handler` |

- **Handler 內嚴禁直接 `new` UseCase 或 DB 連線。** 一律從建構子注入。
- UseCase 的呼叫鏈：

```
UseCase → repository 介面拿到 entity → entity.ToDomain() → 執行業務行為
        → domain.ToEntity() 存回 / domain.ToDto() 回傳
```

- **entity 與 domain model 不直接回給 Handler**，一律先轉成 DTO。
- `internal/infrastructure/` 實作 feature 定義的介面；依賴方向是 infrastructure → feature，不能反過來。

## 介面：什麼要抽，什麼不抽

遵循 Go 的 **"Accept interfaces, return structs"**，不過度設計。

| 東西 | 抽介面？ | 怎麼注入 |
| :--- | :--- | :--- |
| 碰外部的元件（Repository、Proxy） | **要**，定義在消費端 | 實作由 `main.go` 注入；測試注入 mock |
| 內部純業務邏輯（演算法、計算元件） | **不要** | 直接注入**真實實例** |

被注入的內部元件**必須無狀態（stateless）**，避免並發請求時 data race。

## 與共用規則的差異（本架構覆寫）

採用本架構時，以下以本檔為準：

| 共用規則（[../naming.md](../naming.md) 等） | 本架構改成 |
| :--- | :--- |
| 介面集中在 `domain/interface/`、一介面一檔 | 介面放在**消費它的 feature 資料夾**的 `repository.go` |
| `Application` / `Controller` 後綴 | `UseCase` / `Handler`（檔名 `usecase.go` / `delivery.go`） |
| 檔名對齊主型別 | feature 內用角色當檔名（`entity.go`、`domain.go`、`usecase.go`…） |

沒列在表上的共用規則（禁 static、`toXxx()` 轉換、Proxy 命名、Code First、金額用 decimal…）照常適用。

## 測試

見 [../testing/go-package-by-feature.md](../testing/go-package-by-feature.md)。
