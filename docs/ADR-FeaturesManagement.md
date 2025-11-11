# ADR-001: Features/UI 管理架構

## 狀態
記錄於 2025-11-11，狀態：已採用

## 背景
- 地圖頁面存在多個功能模組（停車資訊、活動、天氣等），需要一致的狀態切換邏輯。
- 功能元件的 UI 行為（如切換、暫存、刷新）具有共通模式，但實作細節不同。
- 先前使用事件總線或分散式狀態管理時，缺乏統一入口，測試與擴充困難。

## 決策
- 建立可重用的組合式函式 `useFeatureManager`（`src/composables/features/useFeatureManager.ts`）：
  - 將實際邏輯抽離為 `createFeatureManager`，確保單例行為且利於測試。
  - 透過 `FeatureStateHandler` 計算下一個 UI 狀態，並以 `featureState`、`currentFeatureId` 等響應式狀態向外暴露。
  - 提供 `handleFeatureState`、`closeFeature`、`stashFeature` 作為唯一入口，集中錯誤處理。
- 以 `FeatureList`（`FeatureRegistry.ts`）作為功能註冊表：
  - 將功能 ID 對應到 `UIBehaviorTypes` 與具體 Vue 組件，確保新增/調整功能只需修改單一位置。
  - 由 `FeatureHost.vue` 動態渲染 `currentFeatureComponent`，保持 UI 入口簡潔。
- 避免使用 EventBus 進行主要 UI 狀態管控，改以具類型安全的 Pinia/composables 架構。

## 後果
- **優點**
  - 統一的入口點讓功能狀態切換可預期，並支援 UI 行為的擴充（透過 `UIBehaviorTypes`）。
  - 減少跨元件隱式通訊，提升可測試性與可讀性。
  - Feature 元件只需關注自身視圖邏輯，與管理層鬆耦合。
- **缺點**
  - 新增 UI 行為類型時需同步調整 `FeatureStateHandler`，需要額外維護成本。
  - 單例管理需注意清除時機（例如測試或特殊頁面重整）。

## 使用說明
- 在頁面或組件中呼叫 `const { handleFeatureState, currentFeatureId } = useFeatureManager()` 取得狀態與操作方法。
- 透過 `FeatureHost.vue` 自動渲染 `currentFeatureComponent`，外部只需維護 `FeatureRegistry` 中的對應組件。
- 若需手動關閉或暫存功能，直接呼叫 `closeFeature()`、`stashFeature()`。
- 測試時可透過 `createFeatureManager()` 工廠函式建立獨立實例，避免與單例干擾。

## 擴充說明
- 新增功能模組時在 `FeatureRegistry.ts` 擴充對應項目，並根據需求更新 `UIBehaviorTypes` 或 `FeatureStateHandler`。
- 若需支援新的互動模式，可在 `UIBehaviorTypes` 新增枚舉值，並於 `FeatureStateHandler` 補上轉換邏輯。
- 如日後需要跨頁共享不同組合的功能列表，可拆分 `FeatureRegistry` 為多份設定檔，再由 `useFeatureManager` 接收參數載入。

## 參考與實作細節
- `useFeatureManager` 暴露的狀態與方法由 `FeatureHost.vue` 消費，並由 `Map.vue` 在按鈕操作時呼叫。
- `FeatureStateHandler` 內部實作封裝行為轉換邏輯，保持 `useFeatureManager` 輕量。
- 若未來功能數量增加，可在 `FeatureRegistry.ts` 內持續擴充而不影響既有邏輯。

## 替代方案
- **集中式 Store（單一 Pinia store 管理所有功能）**：雖然可將狀態集中，但會混雜純邏輯與 UI 行為，當功能數增加時會造成 store 過於肥大，且不易拆測。
- **Vue provide/inject 直接穿透狀態**：可以提供共享狀態，但會迫使子層組件直接操作狀態物件，缺乏像 `FeatureStateHandler` 這樣的行為抽象，難以維持一致的 UI 行為。

## 後續工作
- 若需要跨頁面重設狀態，應提供明確的重置 API。
- 對 `FeatureStateHandler` 與 `useFeatureManager` 撰寫更細緻的單元測試，確保行為擴充時安全。

