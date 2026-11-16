# Testing — 前端 Feature-based（Vitest）

> 先讀 [../testing.md](../testing.md) 的共用原則。本檔只寫這個架構特有的部分。
> 架構本身見 [../architecture/frontend-feature-based.md](../architecture/frontend-feature-based.md)。

## 工具

- 一律用 **Vitest**：執行、斷言、mock 都用它。直接共用 `vite.config.ts` 的別名與編譯設定。
- Table-driven 用 `it.each`。
- 元件渲染用框架對應的工具（`@vue/test-utils`、`@testing-library/react`），包成專案的 `@/test-utils`。

## 兩層測試，職責不重疊

| 測試類型 | 測什麼 | 用真的（不 mock） | 要 mock | 驗證重點 |
| :--- | :--- | :--- | :--- | :--- |
| **UseCase Test** | `usecases/`、`utils/`、`stores/` | 業務邏輯、純函式、全域狀態 | `services/*.api.ts`、瀏覽器 Storage、路由跳轉 | 業務規則、計算、錯誤處理、邊界條件 |
| **Component Test** | `.vue` / `.tsx` 元件 | 元件本身的渲染與 DOM 互動 | **UseCase 層** | 使用者視角的互動（點擊）、UI 狀態渲染（loading 時按鈕禁用、顯示錯誤） |

## 檔案擺放

- 測試放 `modules/<feature>/__tests__/`，檔名 `*.spec.ts`（也可以直接放在受測檔旁邊）。
- 仍然只 import 公開的東西，只驗業務結果。

## 與共用規則的差異（本架構覆寫）

| 共用規則 | 本架構改成 |
| :--- | :--- |
| 只 mock 介面 | 用 `vi.mock()` mock **整個模組**（`services/*.api.ts`、UseCase） |
| 內部邏輯一律用真實實例 | **Component Test 要 mock UseCase**，讓元件測試只管 UI；業務邏輯交給 UseCase Test |

## 範本

### UseCase：`modules/order/usecases/checkout.usecase.ts`

```typescript
import { apiCheckout } from '../services/order.api'
import type { CartItem } from '../types/order.type'

export class CheckoutUseCase {
  isLoading = false
  errorMessage: string | null = null

  async handleCheckout(cartItems: CartItem[]): Promise<boolean> {
    if (cartItems.length === 0) {
      this.errorMessage = '購物車不能為空'
      return false
    }

    this.isLoading = true
    this.errorMessage = null
    try {
      await apiCheckout(cartItems)
      return true
    } catch (error) {
      this.errorMessage = '結帳失敗，請稍後再試'
      return false
    } finally {
      this.isLoading = false
    }
  }
}
```

### UseCase Test：`modules/order/__tests__/checkout.usecase.spec.ts`

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { CheckoutUseCase } from '../usecases/checkout.usecase'
import * as orderApi from '../services/order.api'

// 1. mock 外部邊界（API 服務）
vi.mock('../services/order.api')

describe('CheckoutUseCase', () => {
  beforeEach(() => {
    vi.clearAllMocks()
  })

  // 2. it.each 做 table-driven
  it.each([
    {
      scenario: '購物車為空時，不應發送 API 並設定錯誤訊息',
      items: [],
      mockApiSuccess: true,
      expectedResult: false,
      expectedError: '購物車不能為空',
      expectApiCalled: false,
    },
    {
      scenario: '正常結帳成功時，應回傳 true 且無錯誤訊息',
      items: [{ id: 'p1', qty: 2 }],
      mockApiSuccess: true,
      expectedResult: true,
      expectedError: null,
      expectApiCalled: true,
    },
    {
      scenario: 'API 發生異常時，應捕獲錯誤並設定提示訊息',
      items: [{ id: 'p1', qty: 2 }],
      mockApiSuccess: false,
      expectedResult: false,
      expectedError: '結帳失敗，請稍後再試',
      expectApiCalled: true,
    },
  ])('$scenario', async ({ items, mockApiSuccess, expectedResult, expectedError, expectApiCalled }) => {
    if (mockApiSuccess) {
      vi.mocked(orderApi.apiCheckout).mockResolvedValueOnce({ orderId: '123' })
    } else {
      vi.mocked(orderApi.apiCheckout).mockRejectedValueOnce(new Error('Network Error'))
    }

    // 3. 建立真實的 UseCase 並執行
    const useCase = new CheckoutUseCase()
    const result = await useCase.handleCheckout(items)

    // 4. 驗證業務結果與狀態
    expect(result).toBe(expectedResult)
    expect(useCase.errorMessage).toBe(expectedError)
    expect(useCase.isLoading).toBe(false)

    if (expectApiCalled) {
      expect(orderApi.apiCheckout).toHaveBeenCalledWith(items)
    } else {
      expect(orderApi.apiCheckout).not.toHaveBeenCalled()
    }
  })
})
```

### Component Test：`modules/order/__tests__/CheckoutButton.spec.ts`

```typescript
import { describe, it, expect, vi } from 'vitest'
// 依 UI 框架引入對應的渲染工具（@vue/test-utils 或 @testing-library/react）
import { renderComponent } from '@/test-utils'
import CheckoutButton from '../components/CheckoutButton'

// 1. mock UseCase，隔離業務邏輯與 API
vi.mock('../usecases/checkout.usecase', () => {
  return {
    CheckoutUseCase: vi.fn().mockImplementation(() => ({
      isLoading: true,
      errorMessage: '庫存不足',
      handleCheckout: vi.fn(),
    })),
  }
})

describe('CheckoutButton', () => {
  it('當處於 loading 狀態且有錯誤訊息時，應正確渲染 UI 並禁用按鈕', () => {
    const { getByRole, getByText } = renderComponent(CheckoutButton, {
      props: { items: [{ id: 'p1', qty: 1 }] },
    })

    // 驗證按鈕狀態與文字
    const button = getByRole('button') as HTMLButtonElement
    expect(button.disabled).toBe(true)
    expect(button.textContent).toContain('處理中...')

    // 驗證錯誤提示有顯示
    expect(getByText('庫存不足')).toBeTruthy()
  })
})
```
