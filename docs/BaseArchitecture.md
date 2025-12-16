## 一、整體資料夾結構

```
src/
│
├─ shared/
│  ├─ components/          # 純 UI（Button、Modal）
│  ├─ composables/         # 通用行為（usePagination）
│  ├─ services/            # 高複雜度通用domain邏輯（AuthService）
│  ├─ store/               # 全域狀態（auth, user, app）
│  └─ types/               # 全域 type / enum
│
├─ features/
│  └─ todo/
│     ├─ components/       # UI 元件
│     │  └─ TodoList.vue
│     │
│     ├─ composables/      # 主商業邏輯（預設寫這）
│     │  └─ useTodo.ts
│     │
│     ├─ services/         # ❗當功能商業邏輯複雜度較高時才抽出
│     │  └─ TodoDomain.ts
│     │
│     ├─ store/            # ❗ 原則上集中於shared層，非必要不出現在features中
│     │  └─ todo.store.ts
│     │
│     └─ models/           # type / interface / enum
│        └─ Todo.ts
│
└─ main.ts
```

---

## 二、資料夾層級說明

### 1️⃣ `app/`（應用層）

| 項目 | 說明 |
|------|------|
| **用途** | 全站級設定 |
| **概述** | 全域狀態、路由、初始化邏輯 |

#### ✅ 可以放

- 全域 store
- router
- app 初始化流程

#### ❌ 不可以放

- 業務邏輯
- feature 專用狀態
- UI 元件

---

### 2️⃣ `features/`（功能模組核心）

> 💡 **一個資料夾 = 一個完整功能世界**

#### 設計原則

| 原則 | 說明 |
|------|------|
| 功能內聚 | 所有相關檔案放在同一 feature 內 |
| 功能獨立 | feature 彼此獨立 |
| 可刪除性 | feature 可被整個刪除而不影響其他功能 |

---

## 三、Feature 內部結構與檔案職責

### 📁 `features/todo/components/`

| 項目 | 說明 |
|------|------|
| **放什麼** | Vue `.vue` UI 元件 |

#### 每個檔案的職責

- 畫面呈現
- 使用者互動
- 呼叫 composable

#### 使用邊界

| 規則 | 說明 |
|------|------|
| ❌ | 不寫商業邏輯 |
| ❌ | 不直接操作 store |
| ❌ | 不呼叫 API |
| ✔ | 只呼叫 `useXxx` |

---

### 📁 `features/todo/composables/`

| 項目 | 說明 |
|------|------|
| **放什麼** | `useXxx.ts` - 功能行為與流程管理 |

#### 每個檔案的職責

- 組合行為
- 管理流程
- 呼叫 store / service
- 管理生命週期

#### 使用邊界

| 規則 | 說明 |
|------|------|
| ✔ | 可 import store / service / class |
| ✔ | 功能流程 |
| ✔ | API orchestration |
| ✔ | 驗證邏輯及商業邏輯 |
| ✔ | 狀態管理（ref / reactive） |
| ❌ | 不被反向 import |

---

### 📁 `features/todo/store/`

| 項目 | 說明 |
|------|------|
| **放什麼** | Pinia store |

#### 每個檔案的職責

- 儲存狀態
- 提供最小狀態修改方法

#### 使用邊界

| 規則 | 說明 |
|------|------|
| ✔ | 只能被 composable / component 使用 |
| ❌ | 不寫流程判斷 |
| ❌ | 不呼叫 API |
| ❌ | 不組合多功能 |

---

### 📁 `features/todo/services/`

| 項目 | 說明 |
|------|------|
| **放什麼** | class / service / domain logic |

#### 每個檔案的職責

- 商業規則
- 計算邏輯
- 狀態轉換
- 與 Vue 無關的邏輯

#### 使用邊界

| 規則 | 說明 |
|------|------|
| ✔ | 較複雜的演算法邏輯 |
| ✔ | 可被單元測試 |
| ✔ | 可被後端或其他前端共用 |
| ❌ | 不 import Vue |
| ❌ | 不 import store |
| ❌ | 不 import composable |

---

### 📁 `features/todo/models/`

| 項目 | 說明 |
|------|------|
| **放什麼** | interface / type / enum |

#### 每個檔案的職責

- 定義資料結構
- 當作模組間契約

#### 使用邊界

| 規則 | 說明 |
|------|------|
| ✔ | 可被所有層 import |
| ✔ | 統一資料結構來源 |
| ❌ | 不寫邏輯 |

---

### 📁 `features/todo/index.ts`

| 項目 | 說明 |
|------|------|
| **放什麼** | feature 對外出口 |

#### 每個檔案的職責

- export 公開 composable / component / type
- 隱藏內部實作

#### 使用邊界

| 規則 | 說明 |
|------|------|
| ✔ | 對外只暴露「可用的東西」 |
| ❌ | 不 export store / service（除非必要） |

---

## 四、`shared/` 資料夾規範

| 項目 | 說明 |
|------|------|
| **用途** | 跨 feature 共用模組 |

### 📁 `shared/components/`

- 通用 UI 元件
- 無業務邏輯

### 📁 `shared/composables/`

- 不屬於任何單一 feature 的行為
- 例如：`usePagination`、`usePermission`

### 📁 `shared/services/`

- 共用工具 class
- 與 Vue 無關

### 📁 `shared/types/`

- 共用 type / interface

---

## 五、檔案層級責任總表

> 💡 **AI 判斷用參考表**

| 類型 | 檔案 | 能做 | 不能做 |
|------|------|------|--------|
| UI | `.vue` | 畫面、互動 | 商業邏輯 |
| 行為 | `useXxx.ts` | 流程、調度 | domain 規則 |
| 狀態 | `*.store.ts` | 狀態存取 | 流程判斷 |
| 規則 | `*.ts` class | 計算、規則 | Vue 依賴 |
| 契約 | `*.ts` type | 定義結構 | 行為 |

---

## 六、依賴邊界（硬性規則）

### ✅ 合法依賴方向

```
.vue
 ↓
composable
 ↙        ↘
store    service
 ↓          ↓
type / interface
```

### ❌ 禁止方向

| 禁止來源 | 禁止目標 |
|----------|----------|
| service | → composable |
| store | → composable |
| class | → Vue |

---

## 依賴關係圖

```
┌─────────────────────────────────────────────────────────────┐
│                         .vue (UI)                           │
│                    畫面呈現、使用者互動                        │
└─────────────────────────┬───────────────────────────────────┘
                          │ 呼叫
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                   composable (useXxx.ts)                    │
│                    行為組合、流程管理                          │
└──────────────┬─────────────────────────┬────────────────────┘
               │ 操作                     │ 呼叫
               ▼                         ▼
┌──────────────────────────┐  ┌───────────────────────────────┐
│     store (*.store.ts)   │  │    service (class / service)  │
│        狀態儲存            │  │     商業規則、計算邏輯          │
└──────────────┬───────────┘  └───────────────┬───────────────┘
               │                              │
               └──────────────┬───────────────┘
                              │ 使用
                              ▼
┌─────────────────────────────────────────────────────────────┐
│               models (type / interface / enum)              │
│                      資料結構定義                             │
└─────────────────────────────────────────────────────────────┘
```

