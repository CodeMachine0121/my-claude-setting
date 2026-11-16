# Architecture — 前端 Feature-based（TypeScript + Vite + Bun）

> 先讀 [../architecture.md](../architecture.md) 的共用原則。本檔只寫這個架構特有的部分。

適用：產品級前端專案。核心與 UI 框架解耦，Vue、React 等都適用。

- 套件管理與腳本：**Bun**
- 建置與開發伺服器：**Vite**
- 測試：**Vitest**（見 [../testing/frontend-feature-based.md](../testing/frontend-feature-based.md)）

**同一個業務模組的局部元件、UseCase、API 服務、型別集中在同一個資料夾**，提高內聚、降低跨資料夾耦合。

## 目錄結構

```text
my-frontend-project/
├── [framework-dirs]/         # 框架特有目錄（路由 pages/app、佈局 layouts…）
├── assets/                   # 未編譯靜態資產（scss、images、global styles）
├── components/               # 全域共用、無業務色彩的 UI 基礎元件（BaseButton、AppModal）
├── constants/                # 全域常數（API endpoints、錯誤代碼、列舉）
├── hooks/ (或 composables/)  # 全域共用的技術型邏輯（useDebounce）
├── modules/                  # 【業務領域模組】
│   ├── user/
│   │   ├── components/       # 只給這個模組用的 UI 元件
│   │   ├── usecases/         # 業務流程與狀態（*.usecase.ts，或 Hook / Composable）
│   │   ├── services/         # 呼叫後端的 API 函式（user.api.ts）
│   │   ├── types/            # 這個領域的型別（DTO、entity）
│   │   ├── utils/            # 這個領域的純計算與轉換函式
│   │   └── __tests__/        # 這個模組的 UseCase Test 與 Component Test
│   └── order/
│       └── ...
├── plugins/                  # 啟動外掛、第三方初始化（i18n、HTTP 攔截器）
├── public/                   # 靜態公開檔案（favicon、robots.txt）
├── stores/                   # 全域跨頁面狀態（Pinia、Zustand、Redux…）
├── utils/                    # 全域共用的純技術工具（formatDate、currencyFormat）
├── main.ts                   # 應用程式進入點（組裝根）
├── vite.config.ts            # Vite 建置 + Vitest 設定
├── tsconfig.json
└── package.json              # 用 Bun 執行
```

## 各層職責

| 層 | 放哪 | 可以做 | 不准做 |
| :--- | :--- | :--- | :--- |
| **UI 元件** | `.vue` / `.tsx` | 畫面結構、樣式、DOM 事件綁定、呼叫 UseCase | 直接 `fetch` / `axios`；複雜計算或業務分支 |
| **UseCase** | `usecases/*.usecase.ts`（或 Hook / Composable） | 業務流程、表單驗證、狀態轉換（loading、error）、呼叫 `services/` | 依賴 UI 渲染細節。盡量保持純 TypeScript |
| **API 服務** | `services/*.api.ts` | 封裝 HTTP 請求與回傳型別。是前端與外部系統的邊界 | 放業務邏輯 |
| **領域工具** | `modules/<feature>/utils/` | 這個領域的純計算與轉換（純函式、無狀態） | 碰 API、碰 UI |

依賴方向：

```
UI 元件 → UseCase → services/*.api.ts → 後端
                  → modules/<feature>/utils/
```

## 元件分級

- **根目錄 `components/`**：只放無業務色彩、高度可複用的設計系統元件（UI Kit）。
- **`modules/<feature>/components/`**：帶特定業務邏輯與資料結構的局部元件。

## 工具函式分級

- **根目錄 `utils/`**：只放純技術工具（日期、貨幣格式）。**不准放業務規則。**
- **`modules/<feature>/utils/`**：這個領域的純計算函式。只給這個模組用，不跨模組 import。

## 與共用規則的差異（本架構覆寫）

採用本架構時，以下以本檔為準：

| 共用規則 | 本架構改成 |
| :--- | :--- |
| 業務行為放 `XxxDomain`（Domain Model） | 業務計算寫成 `modules/<feature>/utils/` 的**純函式**；流程與狀態放 UseCase |
| 禁 helper，不得已才放 `utilities/` | 允許根目錄 `utils/`（純技術）與 `modules/<feature>/utils/`（領域純計算） |
| 資料一律用 `class` | API 的 DTO（wire 形狀）可以用 TS `type` 描述，不用 new class。entity 仍用 `class` |
| 外部 API 一律 `Proxy` + `I` 介面 | 用 `services/*.api.ts` 匯出函式（如 `apiCheckout`），不另定介面 |
| `Application` / `Controller` 後綴 | `UseCase`（檔名 `*.usecase.ts`）；UI 元件照框架慣例命名 |
| 介面集中、一介面一檔 | 不適用（前端不為 API 層定介面） |

沒列在表上的共用規則（依賴方向、model 內禁 static、`toXxx()` 轉換、全名不縮寫、禁 `any`…）照常適用。

## 測試

見 [../testing/frontend-feature-based.md](../testing/frontend-feature-based.md)。
