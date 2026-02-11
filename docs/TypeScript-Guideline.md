# TypeScript 指引

## 目錄

1. [TypeScript 特性說明](#1-typescript-特性說明)
2. [tsconfig 設定](#2-tsconfig-設定)
3. [結構型別系統](#3-結構型別系統)
4. [型別推論](#4-型別推論)
   - 4.1 一般變數
   - 4.2 函式參數
   - 4.3 公開函式回傳值
   - 4.4 `as` 型別斷言
5. [基本型別使用注意](#5-基本型別使用注意)
   - 5.1 any / unknown / never
   - 5.2 null / undefined
6. [interface & type 選擇](#6-interface--type-選擇)
7. [Enum 使用注意](#7-enum-使用注意)
8. [Type Guard 型別守衛](#8-type-guard-型別守衛)
9. [型別檔案組織](#9-型別檔案組織)
10. [Utility Types 工具型別](#10-utility-types-工具型別)
11. [常見錯誤](#11-常見錯誤)

---

## 1. TypeScript 特性說明

核心特性包含：
- 靜態型別檢查：在編譯階段即檢查變數、函式參數與回傳值的型別一致性，提早發現錯誤。
- 型別推斷：在不明確標註型別的情況下，根據程式碼上下文自動推斷型別，降低標註負擔。
- 結構型別系統：以物件的結構判斷相容性，而非依賴明確的繼承或實作宣告，提高組合與重構彈性。
- 流程導向型別分析：根據程式控制流程（if / switch / return 等）動態收斂型別範圍，提升型別精確度。
- 編譯期重構支援：透過型別關聯性，協助工具在重新命名、調整介面時檢查所有受影響的使用點。

---

## 2. tsconfig 設定

[參考](https://ithelp.ithome.com.tw/articles/10263733)

### 2.1 `strict: true`：TypeScript 的嚴格模式，會同時開啟多個型別檢查選項，建議不要關閉

嚴格檢查設定，包含：
- `strictNullChecks`：啟用嚴格的 null 檢查
- `noImplicitAny`：在表達式和聲明上有隱含的 any類型時報錯 
- `strictFunctionTypes`：啟用檢查function型別 

### 2.2 `allowJs: true`：接受 JS、TS 共存

```json
{
    "compilerOptions": {
        "allowJs": true,       // 允許 import .js 檔
        "checkJs": false       // 不檢查 .js（漸進遷移用）
    }
}
```

JS 專案要慢慢轉 TS 時用這個。

### 2.3 `noEmit: true`：TS 只做檢查，不產生檔案

```json
{
    "compilerOptions": {
        "noEmit": true
    }
}
```

現在主流做法 : 讓 Vite 負責編譯及打包，TS 只負責型別檢查

---

## 3. 結構型別系統

[參考](https://medium.com/redox-techblog/structural-typing-in-typescript-4b89f21d6004)

TS 關注的是結構本身而非名稱，如果兩個結構所有組成特徵都屬於同一類型，則該值被認為是等價的，範例如下 : 

```typescript
interface Point { x: number; y: number; }

const obj = { x: 10, y: 20, z: 30 };
function logPoint(p: Point) { ... }
logPoint(obj);  // Runtime 通過，因為有 x 和 y
```

名稱對編譯器來說不重要，因為它們本質上只是同一類型的別名 (結構即類型)

---

## 4. 型別推論

[參考](https://willh.gitbook.io/typescript-tutorial/basics/type-inference)

如果沒有明確的指定型別，TypeScript 會依照型別推論（Type Inference）的規則推斷出一個型別。

```typescript
let myFavoriteNumber = 'seven';
myFavoriteNumber = 7;

// index.ts(2,1): error TS2322: Type 'number' is not assignable to type 'string'.
```

等價於

```typescript
let myFavoriteNumber: string = 'seven';
myFavoriteNumber = 7;

// index.ts(2,1): error TS2322: Type 'number' is not assignable to type 'string'.
```

TypeScript 會在沒有明確的指定型別的時候推測出一個型別，這就是型別推論。

### 4.1 一般變數：大部分情況不用標註型別

```typescript
// 多此一舉，TS 推斷的結果跟你寫的一樣
const name: string = "Adam";
const count: number = 42;
const items: number[] = [1, 2, 3];

// 不用寫，TS 自己會推斷
const name = "Adam";      // string
const count = 42;         // number
const items = [1, 2, 3];  // number[]
```

**但這幾種情況要標：**

```typescript
// 初始值是 null，不標會推斷成 Ref<null>，之後賦值報錯
const user = ref<User | null>(null);

// 空陣列，不標會推斷成 never[]
const items = ref<Item[]>([]);

// 複雜表達式，看不出是什麼型別
const config: AppConfig = getConfig();
```

判斷標準：看初始值能不能一眼知道型別。能 → 不標型別，不能 → 標註型別。

### 4.2 函式參數：要標型別

```typescript
// 不標的話 TS 會當成 any，等於沒檢查
function greet(name) {        // name 是 any
    return "Hello " + name;
}

// 標了才有意義
function greet(name: string) {
    return "Hello " + name;
}
```

### 4.3 公開函式回傳值：要標型別

```typescript
// 不標的話，改了實作可能不小心改到回傳型別
export function getUser(id: string) {
    // 原本回傳 User
    // 某天有人改成回傳 User | null
    // 呼叫端不會報錯，但行為變了
}

// 標了就是契約，改了會報錯
export function getUser(id: string): User {
    // 如果回傳 User | null，這裡會報錯
}
```

### 4.4 `as` 型別斷言：少用

```typescript
// as 是強制告訴 TS 型別
const data = JSON.parse(str) as User;  // 若型別錯誤只會在執行時發生錯誤

// 建議先做型別檢查再做型別斷言
const data = JSON.parse(str);
if (isUser(data)) {
    // 這裡 data 是 User
}
```

---

## 5. 基本型別使用注意

[TS型別參考](https://ithelp.ithome.com.tw/m/articles/10264244)

用小寫：`string`、`number`、`boolean`

大寫的 `String`、`Number` 是 JS 的 wrapper 物件，不要用。

### 5.1 any / unknown / never

#### any：預設禁止

會傳染，碰到的都變 any
不要使用 any，真的不知道就用 unknown，真的要用要寫註解說明原因

```typescript
function fetchData(): any {
    return JSON.parse('...');
}

const data = fetchData();     // any
const name = data.user.name;  // any，整條的型別都會被判斷成any
name.whatever();              // 不會報錯，但 Runtime 會出錯
```

#### unknown：外部資料用這個

用之前要先檢查型別，比 any 安全。

```typescript
function parseInput(input: unknown) {
    // 不能直接用
    input.name;  // 報錯

    // 要先檢查
    if (typeof input === 'object' && input !== null && 'name' in input) {
        console.log(input.name);  // OK
    }
}

// catch 的 error 也用 unknown
try {
    // ...
} catch (error: unknown) {
    if (error instanceof Error) {
        console.log(error.message);
    }
}
```

#### never：窮盡檢查用

定義 : `never` 表示「不可能有值」。

當你 switch 把所有可能都處理完，剩下的就是「不可能」，型別變成 `never`。

```typescript
type Status = 'pending' | 'active' | 'done';

function handle(status: Status) {
    switch (status) {
        case 'pending': return 1;   // 處理完，排除 'pending'
        case 'active': return 2;    // 處理完，排除 'active'
        case 'done': return 3;      // 處理完，排除 'done'
        default:
            // 走到這裡，status 已經沒有可能的值了
            // TS 推斷 status 是 never
            const _check: never = status;  // OK，因為 status 確實是 never
    }
}
```

**如果你漏了一個 case**

```typescript
function handle(status: Status) {
    switch (status) {
        case 'pending': return 1;
        case 'active': return 2;
        // 忘記 'done' 了！
        default:
            // status 還剩 'done'，不是 never
            const _check: never = status;
            // 報錯：Type '"done"' is not assignable to type 'never'
    }
}
```

漏處理就會報錯，不會讓你忘記。

**實際場景：** 之後有人加了 `type Status = 'pending' | 'active' | 'done' | 'cancelled'`，如果 switch 沒加 `case 'cancelled'`，TS 會報錯提醒你。

### 5.2 null / undefined

專案開了 `strictNullChecks`，意思是 TS 會認真檢查 null 和 undefined，不能隨便用。

```typescript
function getUser(): User | null {
    return null;
}

const user = getUser();
console.log(user.name);  // 報錯：user 可能是 null
```

#### `?.` Optional Chaining：安全取值

如果是 null/undefined 就不往下取，回傳 undefined。

```typescript
const user = getUser();
console.log(user?.name);        // 如果 user 是 null，回傳 undefined
console.log(user?.profile?.avatar);  // 可以一路串下去
```

#### `??` Nullish Coalescing：給預設值

左邊是 null 或 undefined 時，用右邊的值。

```typescript
const name = user?.name ?? 'Anonymous';  // user.name 是 null/undefined 就用 'Anonymous'

// ?? 跟 || 不一樣，|| 會把 0、'' 也當成 falsy
const count = data.count || 10;   // count 是 0 的話會變成 10
const count = data.count ?? 10;   // count 是 0 的話還是 0，只有 null/undefined 才用 10
```

#### `!` Non-null Assertion：告訴 TS「我確定不是 null」

```typescript
const user = getUser();
console.log(user!.name);  // 告訴 TS「user 一定不是 null」

// 什麼時候用：你比 TS 更清楚情況
const element = document.getElementById('app')!;  // 你確定這個 element 存在

// 不要這樣用：為了讓 TS 閉嘴
const user = getUser();
console.log(user!.name);  // 如果你不確定 user 會不會是 null，不要用 !
```

總結：優先用 `?.` 和 `??`，而 `!` 只在你真的確定不是 null 的時候用。

---

## 6. interface & type 選擇

兩個都能定義型別，但有些事只有一個能做。

### 6.1 interface：定義物件結構

```typescript
interface User {
    id: string;
    name: string;
    email: string;
}

interface Admin extends User {  // 可以 extends
    permissions: string[];
}
```

interface 可以被 extends，也可以被 implements（給 class 用）。

### 6.2 type：定義 Union、字面量聯合

```typescript
// 這個 interface 做不到
type Status = 'pending' | 'active' | 'done';  // 字面量聯合
type Result = Success | Error;                 // Union type

type ID = string | number;  // 這樣也通
```

### 6.3 type：定義函式型別

```typescript
type Handler = (event: Event) => void;
type Callback<T> = (data: T) => void;
```

interface 也能定義函式，但語法醜：

```typescript
interface Handler {
    (event: Event): void;  // 比較少人這樣寫
}
```

### 6.4 結論

| 情境 | 用哪個 |
|------|--------|
| 物件結構 | `interface` |
| Union / 字面量聯合 | `type` |
| 函式型別 | `type` |
| 需要 extends / implements | `interface` |

其實很多情況兩個都能用，可依團隊風格自行定義

---

## 7. Enum 使用注意

### 7.1 字串 Enum：對應後端的固定值

```typescript
enum OrderStatus {
    Pending = 'PENDING',
    Shipped = 'SHIPPED',
    Delivered = 'DELIVERED',
}

// 後端回傳 'PENDING'，前端用 OrderStatus.Pending
```

### 7.2 字面量聯合：前端自己用的選項

```typescript
type Tab = 'home' | 'profile' | 'settings';

// 比 enum 簡單，編譯後不會產生額外程式碼
```

### 7.3 為什麼不用數字 Enum

```typescript
enum Status {
    Pending,   // 0
    Active,    // 1
    Done,      // 2
}

// 問題 1：看 log 看不出意思
console.log(status);  // 輸出 1，1 是什麼？

// 問題 2：可以亂塞數字
const s: Status = 999;  // 不會報錯！

// 問題 3：中間插入會改變後面的值
enum Status {
    Pending,   // 0
    Review,    // 1  ← 新增的
    Active,    // 2  ← 從 1 變成 2，舊資料爆了
    Done,      // 3
}
```

---

## 8. Type Guard 型別守衛

[參考](https://vocus.cc/article/6602922dfd89780001f76d51)

在 JS 裡我們常用 `typeof`、`instanceof` 判斷型別，TS 認得這些，會自動收窄型別 ( Narrowing )

### 8.1 typeof

```typescript
function process(value: string | number) {
    if (typeof value === 'string') {
        // 這個 if 區塊裡，TS 知道 value 是 string
        return value.toUpperCase();
    }
    // 這裡 TS 知道 value 是 number
    return value.toFixed(2);
}
```

### 8.2 instanceof

```typescript
function handle(error: Error | string) {
    if (error instanceof Error) {
        // 這裡 error 是 Error
        return error.message;
    }
    // 這裡 error 是 string
    return error;
}
```

### 8.3 in

```typescript
interface Cat { meow(): void; }
interface Dog { bark(): void; }

function speak(animal: Cat | Dog) {
    if ('meow' in animal) {
        // 這裡是 Cat
        animal.meow();
    } else {
        // 這裡是 Dog
        animal.bark();
    }
}
```

### 8.4 自訂 Type Guard

有時候 TS 判斷不出來，可以自己寫：

```typescript
interface User {
    id: string;
    name: string;
}

// 回傳 `value is User` 告訴 TS：如果回傳 true，value 就是 User
function isUser(value: unknown): value is User {
    return (
        typeof value === 'object' &&
        value !== null &&
        'id' in value &&
        'name' in value
    );
}

// 使用
const data: unknown = JSON.parse(str);
if (isUser(data)) {
    // 這裡 data 是 User
    console.log(data.name);
}
```

---

## 9. 型別檔案組織

[參考](https://www.webdong.dev/zh-tw/post/how-to-organize-your-typescript-types/)

### 9.1 原則：型別與功能綁定

```
src/
├── features/
│   └── user/
│       ├── components/
│       ├── services/
│       └── types/
│           ├── user.ts      # User, UserProfile
│           └── api.ts       # UserApiResponse
├── shared/
│   └── types/
│       ├── common.ts        # ID, Timestamp
│       └── api.ts           # ApiResponse<T>, PaginatedResponse<T>
```

### 9.2 純型別引入用 `import type`

```typescript
// 一般 import：會被打包進去
import { User } from './types/user';

// import type：編譯後會消失，不會被打包
import type { User } from './types/user';

// 混合
import { createUser, type User } from './user';
```

用 `import type` 的好處：
- 明確表示「這只是型別」
- 避免循環引用問題
- 打包更乾淨

---

## 10. Utility Types 工具型別

TS 內建的型別工具

### 10.1 Partial\<T\>：所有屬性變成可選

```typescript
interface User {
    id: string;
    name: string;
    email: string;
}

// 更新時不用傳全部
function updateUser(id: string, updates: Partial<User>) {
    // updates 可以是 { name: 'new' } 或 { email: 'new@mail.com' } 或全部
}

updateUser('1', { name: 'Adam' });  // OK，不用傳 id 和 email
```

### 10.2 Pick\<T, K\>：只挑幾個屬性

```typescript
// 只要 id 和 name
type UserPreview = Pick<User, 'id' | 'name'>;
// 等於 { id: string; name: string; }
```

### 10.3 Omit\<T, K\>：排除某些屬性

```typescript
// 不要 password
type SafeUser = Omit<User, 'password'>;
// User 有的都有，除了 password
```

### 10.4 Record\<K, V\>：鍵值對映射

```typescript
// key 是 string，value 是 number
type Scores = Record<string, number>;
const scores: Scores = {
    math: 90,
    english: 85,
};

// key 限定幾個選項
type StatusCount = Record<'pending' | 'active' | 'done', number>;
const counts: StatusCount = {
    pending: 5,
    active: 10,
    done: 20,
};
```

### 10.5 Required\<T\>：所有屬性變成必填

```typescript
interface Config {
    host?: string;
    port?: number;
}

// 驗證後確保都有值
const config: Required<Config> = {
    host: 'localhost',
    port: 3000,
};
```

### 10.6 Readonly\<T\>：所有屬性變成唯讀

```typescript
const user: Readonly<User> = { id: '1', name: 'Adam', email: 'a@b.com' };
user.name = 'Bob';  // 報錯：Cannot assign to 'name' because it is a read-only property
```

---

## 11. 常見錯誤

### 11.1 `as any` 到處用

```typescript
// 不好：any 會傳染
const data = response.data as any;
const name = data.user.name;  // name 也是 any，等於沒檢查

// 好：定義正確型別
interface UserResponse {
    user: { name: string };
}
const data = response.data as UserResponse;
```

### 11.2 `ref(null)` 推斷錯誤（Vue）

```typescript
// 不好：推斷成 Ref<null>，之後塞 User 會報錯
const user = ref(null);
user.value = { name: 'Adam' };  // 報錯

// 好：明確給型別
const user = ref<User | null>(null);
```

### 11.3 可推斷的也標型別

```typescript
// 囉唆
const count: number = 0;
const name: string = 'Adam';

// TS 會自己推斷，不用寫
const count = 0;
const name = 'Adam';
```

### 11.4 `@ts-ignore` 藏問題

```typescript
// 不好：不知道為什麼要 ignore
// @ts-ignore
const result = someWeirdFunction();

// 好：用 @ts-expect-error 說明原因，問題修好後會提醒你移除
// @ts-expect-error: 第三方套件型別定義有問題，等他們修
const result = someWeirdFunction();
```

### 11.5 型別和實作不同步

```typescript
// 型別說回傳 User，實際回傳 User | null
function getUser(id: string): User {
    const user = db.find(id);
    return user;  // 可能是 null！
}

// 型別要誠實
function getUser(id: string): User | null {
    const user = db.find(id);
    return user;
}
```

### 11.6 過度使用 `!`（Non-null Assertion）

```typescript
// 不好：為了讓 TS 閉嘴
const user = getUser();
console.log(user!.name);  // 如果 user 是 null，Runtime 爆

// 好：處理 null 的情況
const user = getUser();
if (user) {
    console.log(user.name);
}
// 或
console.log(user?.name ?? 'Anonymous');
```

### 11.7 沒用 `import type`

```typescript
// 不好：型別也被打包進去
import { User } from './types';

// 好：明確表示只是型別
import type { User } from './types';
```

---
