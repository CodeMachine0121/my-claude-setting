# Architecture — 入口

本檔只做兩件事：

1. 告訴你**這個專案該讀哪一份架構規則**。
2. 列出**不論哪種架構都成立**的共用原則。

各架構自己的資料夾結構、層次、命名覆寫，寫在 `architecture/` 底下。

## 選架構

| 專案型態 | 讀哪份 |
| :--- | :--- |
| Clean / Onion 分層（`domain/` `application/` `controller/` `infrastructure/`） | [architecture/layered.md](architecture/layered.md) |
| Go API，依業務領域切資料夾（`internal/<feature>/`） | [architecture/go-package-by-feature.md](architecture/go-package-by-feature.md) |

怎麼判斷專案用哪一種：

1. **專案文件有寫**（README、`docs/`）→ 照寫的。
2. **沒寫** → 看既有目錄結構，跟上表比對。
3. **新專案、還沒有結構** → 先問使用者，不要自己挑。

選定架構後，測試也讀對應的那份（見 [testing.md](testing.md)）。

> 要加新架構：在 `architecture/` 開一份新檔，在上表補一行；測試若有差異，`testing/` 也補一份。

## 共用原則（所有架構都成立）

### 依賴方向指向核心

- **依賴方向一律指向業務核心**；核心不認識 HTTP、ORM、任何第三方 SDK。
- 外部資源（DB、外部 API）一律**透過介面**被核心使用，實作放在基礎設施那一側（DIP——細節依賴抽象）。
- **組裝根**（如 `main`、`Program`）是唯一知道所有具體型別的地方，負責 DI 組裝、設定讀取、路由註冊。其他地方不准自己 `new` 依賴。
- 應用程式原始碼放在語言慣用的內部目錄（Go 的 `internal/`、.NET 的專案資料夾），**不對外暴露**。
- **不使用「port」一詞或資料夾**——對外抽象一律稱「介面」。

### 行為住在它操作的資料旁邊

#### model 內一律不得有 `static` method

**entity、domain model、DTO、VO 一律不得有任何 `static` method**，不論公開或私有。model 是資料，不是工具箱。

最常見的違規是**用 static 做 model 之間的轉換**：

```
❌ OrderDomain.fromEntity(order)      ← 轉換掛在目標身上，還是個 static
❌ MoneyVo.fromString("299", "TWD")   ← 名為工廠，實為 static 工具
✅ order.toDomain()                   ← 轉換掛在來源身上，是 instance method
✅ new MoneyVo("299", "TWD")          ← 建構就用建構子，正規化寫在建構子內
```

**要從 A 變成 B，method 就寫在 A 身上，叫 `toB()`。** 轉換讀的是 A 的 property——寫在 B 上就得把 A 整個傳進去，那正是 Feature Envy。

建立時的驗證或正規化（非法 enum → 安全預設值、數值 clamp），**一律寫在建構子裡**，不為此開 static 工廠。
（沒有建構子語法的語言，如 Go，慣用的 `NewXxx` 就是建構子，不算 static。）

#### `private static` 一律不留

看到 `private static`，**不要只是改成 public**。三步搬家：

1. **看它的參數來自哪一個 module** —— 參數是誰，行為就屬於誰（Feature Envy）。
2. **把參數換成該 module 本身**，把整個 method **搬進那個 module**。
3. 搬進去後，method 內部**改用自身的 property 運算**，參數就消失了。

```
❌ service 裡的 private static ToDto(entity)      ← 參數全部來自 entity，行為卻住在 service
✅ entity 自己的 ToDto() → 回傳一個新的 DTO       ← 由 entity 實例呼叫，內部讀自身 property
```

這類「資料形狀轉換」不算業務邏輯，放在 entity 上沒問題。

#### `private` method 也要先過門檻

| 被幾個 public method 使用 | 做法 |
| :--- | :--- |
| **2 個以上** | 有共用價值，可以留成 private helper |
| **只有 1 個** | **直接 inline 回原本的 method** |

抽 method 的理由只有「消除重複」與「行為屬於別人」，不是「這段太長」。太長是拆物件的訊號。

### `utilities/`

唯一允許放 helper 類別的地方，且只收「不得已」的純技術性工具（門檻見 [code-style.md](code-style.md)）。預設不該有東西進去。
