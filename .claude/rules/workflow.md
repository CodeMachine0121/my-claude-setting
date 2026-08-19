# Workflow

## Commit 前自檢

commit 前一定要跑過該語言對應的三件事並全綠：

| 檢查 | 說明 |
| :--- | :--- |
| **build** | 編譯 / 型別檢查全過 |
| **static analysis** | vet / lint / analyzer 無新增警告 |
| **test** | 全部測試通過 |

若專案沒有 pre-commit hook 或 lint 設定檔，代表這些**靠人工遵守**，不要假設自動化已存在。

## SDD + TDD 節奏

1. 新功能先開 `.sdd/{YYYY-MM-DD}-{feature-slug}/`，寫 `BRIEF.md` / `PRD.md`
2. 更新 `UL-MAP.md`（通用語言地圖，命名以此為準）
3. 依 TC（test case）編號逐步實作，由內而外：

   ```
   domain model 單元測試 → repository → domain service → application
   → controller → 組裝根（DI / 路由） → API 測試集更新
   ```

4. **每個 commit 對應一個小步驟**，訊息標註對應的 TC 編號 + 該步驟做了什麼

## 文件不可漂移

- 修改業務邏輯或新增功能時，UL-MAP 與該功能切片的 PRD 需同步更新。
- **新增功能請開新的 `.sdd/{date}-{feature-slug}/` 資料夾**，不要回頭改舊切片的 PRD。
- 新增 / 修改路由需同步更新 API 測試集（Postman 等）。
