## 2️⃣ 新建商品（Create Product）

你設計了一個後台表單，讓使用者登入後可以新增商品。表單有商品名稱、描述、價格、圖片等欄位。當使用者填完表單按下送出，前端會發出一個 `POST` 請求到後端 API，存進資料庫。

功能沒問題，操作正常。但駭客早就打開 F12，開始下手了。

---

## 🔻 2.1 你會怎麼寫新增商品的功能？

前端（React/Vue/Javascript 任意框架）：

```js
fetch("https://example.com/api/products", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    Authorization: `Bearer ${localStorage.getItem("token")}` // ❌
  },
  body: JSON.stringify({
    name: "高腰牛仔褲",
    description: "超百搭，強力推薦！",
    price: 999,
    isAdmin: false // ❌ 沒有從 UI 填寫，但偷偷加進去
  })
});
```

後端（FastAPI）：

```python
@app.post("/api/products")
def create_product(data: ProductCreate):
    db.insert(data)
    return {"msg": "ok"}
```

✅ 功能執行成功，資料也存進去了。只不過你根本不知道是誰存的，也沒擋住惡意欄位。

---

## 🧨 2.2 駭客怎麼打你？（篡改參數與角色偽造）

攻擊者打開 DevTools，在 JavaScript console 或攔截 fetch request，**加上後台 UI 不存在的欄位**：

```json
{
  "name": "高腰牛仔褲",
  "description": "<script>alert(1)</script>",
  "price": 1,
  "isAdmin": true
}
```

### 攻擊方式：

* **篡改欄位值**：例如把價格改成 1 元、加上 `isAdmin: true`
* **注入腳本**：在描述欄輸入 `<script>`，等待別人打開商品頁時觸發 XSS
* **使用 token 重播攻擊**：複製 request，在別的時間、帳號再次執行

### 為什麼成功？

* 後端沒驗證請求欄位與來源身份
* 沒有清理輸入內容
* 沒有 escape HTML 輸出

---

## ✅ 2.3 防守：從三層驗證資料來源與內容

### ✅ 表單層（前端）

* 不顯示不該輸入的欄位（如 `isAdmin`）
* 使用型別檢查（price 為數字）
* 對輸入內容做長度與格式限制（例如名稱不得超過 100 字）

### ✅ API 層（後端 router）

* 驗證使用者身份（例如從 token 解出 user\_id）
* 僅允許授權帳號建立資料
* 強制覆蓋敏感欄位（例如 `isAdmin = False` regardless of input）

### ✅ 資料處理層（DB or Response）

* 儲存前：對 description 等欄位進行清洗（移除 script）
* 顯示時：用 `textContent` 或 escape library 避免 HTML 執行

---

## 🧠 額外補充：圖片與 API token 的防護

### 🔐 圖片存放

* 不應讓使用者直接看到圖床實體網址（例如 `r2.cloudflarestorage.com/...`）
* 建議由後端生成一次性簽名 URL 或透過中介 proxy 提供圖片路徑

### 🔐 API token 不可寫死在前端

* 常見錯誤：`const API_KEY = "sk_live_..."` 被打包到前端
* 正確做法：讓前端請求後端，由後端代送 API，並控制速率與授權

---

## ✅ 小結：新建商品的常見資安破口與防守點

| 漏洞類型       | 錯誤做法          | 防守位置        | 建議做法                     |
| ---------- | ------------- | ----------- | ------------------------ |
| 欄位偽造       | 任意欄位可送出       | 後端 API 層    | Schema 檢查 + 權限覆蓋         |
| XSS 注入     | 商品描述未過濾       | 顯示端 + DB 儲存 | 儲存前清洗 + 顯示時 escape       |
| 使用者偽裝      | token 有但沒驗證身份 | API 驗證層     | 從 token 取出 user\_id 對應處理 |
| API key 泄漏 | key 寫死在前端     | 編譯階段        | 後端中繼、環境變數注入              |

---

✅ 下一章，我們將進入第三個操作：「查詢圖片／查資料」，你會看到如果沒有做好授權與資料保護，使用者可以看見別人的圖片、email，甚至直接暴露內部儲存路徑與 token。
