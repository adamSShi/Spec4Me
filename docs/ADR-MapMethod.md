# ADR-002: 地圖共用方法與初始化架構

# ADR-002: 地圖共用方法與初始化架構

## 狀態
記錄於 2025-11-11，狀態：已採用

## 背景
- ArcGIS SceneView/Map 在專案中多處需要執行常見操作（移動視角、管理圖層、查詢等）。
- 早期做法為於各功能模組各自呼叫 ArcGIS API，導致重複程式碼、難以維護，且測試覆蓋率低。
- 需提供統一的初始化流程，以便 Vue 組件取得已準備好的地圖實例與封裝後的共用方法。

## 決策
- 在 `MapMethods.ts` 建立 `MapMethods` 類別，集中封裝 ArcGIS 常用操作：
  - 提供 `addLayer`、`removeLayer`、`clearAll` 等圖層管理 API，並維護 `managedLayerIds` 以避免重複或遺漏移除。
  - `createPointLayer` 使用固定圖層 ID 管理單點圖層，確保新點會替換舊點。
  - `queryPointInPolygon`、`queryBufferArea` 先驗證圖層是否支援查詢，並配置標準的 `createQuery` 參數。
  - `goTo`、`closePopup` 等方法對 `SceneView` 進行安全操作。
- 於 `useArcGISMap.ts` 中建立單例式的組合式函式：
  - 初始化 ArcGIS `Map` 與 `SceneView`，將實例以 `readonly` 暴露供其他組件監聽。
  - 在初始化完成後建立 `MapMethods` 實例並透過 `getMapMethods()` 提供外部存取。
  - 於非生產環境時將實例掛到 `window` 方便除錯。
- 透過 `MapMethods.test.ts` 使用 Vitest 建立完整單元測試：
  - 模擬 ArcGIS Map/SceneView/GraphicsLayer 及 `geodesicBufferOperator`，驗證同步與非同步方法。
  - 覆蓋圖層管理、點圖層建立、緩衝查詢、錯誤處理與整合流程。

## 後果
- **優點**
  - 共享的地圖操作集中管理，減少重複程式與潛在錯誤。
  - `useArcGISMap` 的單例設計避免多重初始化 ArcGIS View，提供一致的生命週期控制。
  - 單元測試可針對封裝類別進行，提升回歸信心。
  - 透過依賴注入（建構子接收 `Map`、`SceneView`），未來可引入替代實作或在測試中注入 mock。
- **缺點**
  - `MapMethods` 直接依賴 ArcGIS API，仍需要在測試中維護對應 mock。
  - 單例 `useArcGISMap` 需要注意在特殊頁面或測試情境下的清理，避免殘留狀態。

## 使用說明
- 在 Vue 組件中透過 `const { initializeMap, getMapMethods } = useArcGISMap(config)` 取得初始化流程與共用方法。
- 先於 `onMounted` 期間呼叫 `initializeMap(container)`；完成後再使用 `getMapMethods()` 取得 `MapMethods` 實例。
- 需要操作地圖時優先呼叫 `MapMethods` 提供的方法，如 `mapMethods.addLayer(layer)`、`mapMethods.queryBufferArea(point, layerId)`。
- 測試時可直接建立 `new MapMethods(mockMap, mockView)` 並注入模擬物件，搭配 `MapMethods.test.ts` 的 mock 參考。

## 擴充說明
- 如需新增地圖操作，可在 `MapMethods` 補充方法，並同步撰寫單元測試以涵蓋新的 ArcGIS 行為。
- 若未來需要多個地圖實例，可調整 `useArcGISMap` 以傳入識別鍵或改為工廠模式，而非單例。
- `initializeMap` 中的 UI 控制（如底圖切換按鈕）可抽出成插件式註冊，減少與核心方法的耦合。
- 若希望支援事件通知，可在 `MapMethods` 中引入型別安全的事件發射介面，但需維持測試可控。

## 參考與實作細節
- `MapMethods` 測試檔（`MapMethods.test.ts`）覆蓋了同步與異步方法，確保 `managedLayerIds`、緩衝查詢等行為正確。
- `useArcGISMap` 提供 `initializeMap`、`switchBasemap`、`cleanup` 等方法，與 `Map.vue` 配合完成地圖初始化與自訂 UI 控制項。
- 若外部需要使用地圖能力，優先透過 `getMapMethods()` 取得封裝 API，避免直接操作 ArcGIS 實例。

## 後續工作
- 評估是否需要對 `MapMethods` 再細分模組（例如查詢相關、圖層管理）以降低單一類別責任。
- 為 `useArcGISMap` 撰寫端到端測試，覆蓋實際初始化流程與 UI 控制項整合。

