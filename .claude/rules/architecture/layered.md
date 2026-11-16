# Architecture — Clean / Onion 分層

> 先讀 [../architecture.md](../architecture.md) 的共用原則（依賴方向、禁 static、三步搬家）。本檔只寫這個架構特有的部分。

**依賴方向一律指向 Domain（核心）；Domain 不依賴任何人。** 與語言、框架無關。

```
   Controller ───▶ Application ───▶ Domain ◀─── Infrastructure
   (HTTP/API)       (use cases)     (核心)        (Repository/Proxy 實作)
                                      ▲
                    domain/interface/ 放所有對外介面（一介面一檔）
```

## 各層職責

- **Domain（核心）**：models（entity / domain model / dto / vo）、領域計算邏輯、Domain Service，以及**所有對外介面**。**Domain 不 import 任何其他層**（不認識 HTTP、LLM SDK、ORM…）。
- **Application**：依賴 Domain，呼叫 Domain Service 編排用例，拿回 **DTO**（全程不碰 entity / domain model）。
- **Controller**：依賴 Application，只做請求 / 回應轉換與參數 binding。
  - 允許的例外：controller 可 import domain service 的**哨兵錯誤**做狀態碼對映（400 / 502 分流）。
- **Infrastructure**：Repository（持久化）與 Proxy（外部 API 呼叫）的**實作**，實作 domain 介面。
- **組裝根**：唯一知道所有具體型別的地方，負責 DI 組裝、設定讀取、路由註冊。

## Domain 內部結構

domain 底下**統一用一個 `models/` 資料夾**收納所有資料/領域物件，再依種類細分：

```
domain/
├── models/
│   ├── entities/     乾淨的資料模型（Data Model），只有欄位、沒有業務邏輯
│   ├── domains/      Domain Model：業務行為的所在地（充血模型）
│   ├── dto/          domain 對 application 的唯一回傳形狀
│   └── vo/           value object：不可變純資料、無行為
├── service/          Domain Service：application 的唯一呼叫入口，一檔一 service
└── interface/        repository / proxy 介面，一介面一檔（mocks 放子資料夾）
```

## 充血模型：Entity 乾淨，行為放 Domain Model

**我們走充血模型，但行為不寫在 Entity 上。**

| 角色 | 定位 | 內容 |
| :--- | :--- | :--- |
| **Entity**（`models/entities/`） | 乾淨的 **Data Model** | 只有欄位、持久化對應（ORM 標註）。**不放任何業務邏輯、不放計算 method** |
| **Domain Model**（`models/domains/`） | 業務行為的所在地 | 針對某個 entity（或一組 entity）拉出的領域物件，**所有計算、驗證、狀態轉換的 function 都放這裡** |

**Domain Model 是一個由 entity 轉換而來的普通 class，不是介面抽象。**

- **不為 Domain Model 定義介面，也不用介面做繼承。** 它沒有多個實作要替換——它就是那個領域概念本身。需要不同行為時，那是不同的 Domain Model。
- **Domain Model 檔內不得宣告任何 `interface`**（見 [../naming.md](../naming.md)）。參數過多時，把參數群組抽成 **VO**（domain 內部）或 **DTO**（跨 service 邊界）。
- 由 entity 進來的路徑一律是 `entity.toDomain()`；要產生 entity 時一律是 `domainModel.toEntity()`。
- Domain Model 以建構子建立，**建構子內做正規化 / clamp**。
- **禁止散落的 package-level / static 計算函式當「工具類」**——行為要掛在 Domain Model 上。
- **entity 與 domain model 絕不直接回傳給 application**，一律先轉成 DTO。

## Domain Service 規則

- `Service` 後綴**僅**用於跨 model 的編排（取資料、轉 DTO、多物件 / 併發運算）；單一物件的計算放它自己的 Domain Model。
- **同一個 Domain Service 內的公開 use-case method 互不呼叫。** 需要跨方法編排時一律由 Application 層負責。
- 併發編排（同時查多個來源）放 Domain Service，用該語言的標準併發原語。

## 呼叫鏈

```
Application → DomainService → repository/proxy 介面（impl 在 infra）
            → entity → 包成 Domain Model 執行行為 → 轉 DTO → 回傳 DTO
```

## 測試

見 [../testing/layered.md](../testing/layered.md)。
