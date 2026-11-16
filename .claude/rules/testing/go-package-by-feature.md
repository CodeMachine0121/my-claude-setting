# Testing — Go Package-by-Feature

> 先讀 [../testing.md](../testing.md) 的共用原則。本檔只寫這個架構特有的部分。
> 架構本身見 [../architecture/go-package-by-feature.md](../architecture/go-package-by-feature.md)。

## 測試重心

- **重兵部署在 `UseCase` 層**：正常流程（happy path）與邊界錯誤（edge cases）都要測。
- **Domain Model（`domain.go`）直接單元測試**：不需要 mock，給輸入、驗結果。
- 顆粒度是**元件級（component-level）**：
  - 對外依賴（`Repository` / `Proxy`）→ **mock**，模擬各種回應與錯誤。
  - 內部無狀態業務元件 → **真實實例**直接注入。

## 工具

- 斷言：`github.com/stretchr/testify` 的 `assert` / `require`。
- Mock：`testify/mock`。
- 寫法：**Table-Driven Tests** + `t.Run()` 子測試。每個子測試自己 new 一份 mock。

## 檔案擺放

| 檔案 | 位置 | 說明 |
| :--- | :--- | :--- |
| 單元測試 `*_test.go` | 同 feature 資料夾，**同一個 package** | 例：`internal/user/usecase_test.go` |
| Mock `*_mock.go` | 同 feature 資料夾 | 例：`internal/user/repository_mock.go`。獨立成檔，給多個測試共用 |
| 整合 / E2E 測試 | 專案根目錄 `test/` | |

## 與共用規則的差異（本架構覆寫）

| 共用規則 | 本架構改成 |
| :--- | :--- |
| 測試用外部 `<pkg>_test` package（黑箱） | 測試與原始碼**同一個 package**，方便直接取用同目錄的 mock |
| mock 由工具產生 | 可手動建立，但**必須嵌入 `mock.Mock`**、方法內只呼叫 `m.Called(...)`，不准寫任何自訂邏輯（那就是 Fake 了） |

同 package 只是為了方便取用，**仍然只呼叫公開 method、只驗業務結果**，不碰私有欄位。

## 範本

### `internal/user/repository_mock.go`

```go
package user

import (
	"context"

	"github.com/stretchr/testify/mock"
)

type MockUserRepository struct {
	mock.Mock
}

func (m *MockUserRepository) FindByEmail(ctx context.Context, email string) (*User, error) {
	args := m.Called(ctx, email)
	if args.Get(0) == nil {
		return nil, args.Error(1)
	}
	return args.Get(0).(*User), args.Error(1)
}
```

### `internal/user/usecase_test.go`

```go
package user

import (
	"context"
	"testing"

	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/mock"
)

func TestUserUseCase_Register(t *testing.T) {
	tests := []struct {
		name          string
		inputEmail    string
		mockSetup     func(m *MockUserRepository)
		expectedError error
	}{
		{
			name:       "Success: Register new user",
			inputEmail: "new@example.com",
			mockSetup: func(m *MockUserRepository) {
				m.On("FindByEmail", mock.Anything, "new@example.com").Return(nil, ErrNotFound)
			},
			expectedError: nil,
		},
		{
			name:       "Fail: Email already exists",
			inputEmail: "exist@example.com",
			mockSetup: func(m *MockUserRepository) {
				m.On("FindByEmail", mock.Anything, "exist@example.com").Return(&User{Email: "exist@example.com"}, nil)
			},
			expectedError: ErrEmailAlreadyExists,
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			// 1. 初始化 mock 並設定預期行為
			mockRepo := new(MockUserRepository)
			tt.mockSetup(mockRepo)

			// 2. 初始化 UseCase（外部依賴用 mock，內部純邏輯元件傳真實實例）
			uc := NewUserUseCase(mockRepo)

			// 3. 執行
			_, err := uc.Register(context.Background(), tt.inputEmail)

			// 4. 驗證結果與 mock 呼叫
			if tt.expectedError != nil {
				assert.ErrorIs(t, err, tt.expectedError)
			} else {
				assert.NoError(t, err)
			}
			mockRepo.AssertExpectations(t)
		})
	}
}
```
