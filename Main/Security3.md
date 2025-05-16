## 3️⃣ 查詢圖片／查資料（View Resource）

你設計了一個功能，讓使用者登入後可以查詢自己的圖片列表或商品資料，這些圖片來自後端儲存（如本機目錄、Cloudflare R2、S3 等），使用者點進頁面時會向後端發出一個請求，然後顯示回傳的圖片或商品清單。

但你沒有想到的是：這些 API 和圖片網址，**其實任何人都能打。**

---

## 🔻 3.1 你會怎麼寫圖片或資料查詢？

前端：

```js
fetch("https://example.com/api/products", {
  method: "GET",
  headers: {
    Authorization: `Bearer ${localStorage.getItem("token")}`
  }
});
```

後端（FastAPI）：

```python
@app.get("/api/products")
def get_products():
    return db.query("SELECT * FROM products")
```

圖片網址直接給：

```html
<img src="https://mybucket.r2.cloudflarestorage.com/uploads/userA/image1.jpg" />
```

✅ 一切功能正常。但你沒做的，是身份驗證與授權判斷。

---

## 🧨 3.2 駭客怎麼打你？（不登入也能看圖、偷資料）

### 攻擊 1：複製圖片連結直接存取

* 使用者登入後能看圖片沒錯
* 但圖片網址是公開的，且沒有任何防護（無簽名、無權限 token）
* 攻擊者可以：

  * 在 F12 Network 中複製圖片 URL
  * 登出帳號，或從無痕視窗直接貼上
  * ✅ 照樣看圖

### 攻擊 2：API 沒做身份驗證

* 使用者只要知道 `/api/products` 存在，就能直接打 API
* 如果後端沒驗證 token，或沒從 token 驗出 user\_id，會回傳「全部使用者的資料」

### 攻擊 3：API 回傳過多資訊（過度揭露）

```json
{
  "id": 1,
  "owner_id": 2,
  "name": "商品 A",
  "price": 999,
  "internal_path": "/mnt/data/uploads/userB/secret.png",
  "s3_key": "uploads/userB/secret.png",
  "api_token": "sk_live_..."
}
```

* 攻擊者只要打 API，就能看到這些「不該讓前端知道的」東西

---

## ✅ 3.3 防守：圖片與 API 都要加上授權邏輯

### ✅ 圖片連結的防守方式：

* **絕對不要直接給出儲存服務的真實網址**（如 R2/S3）
* 應該改由後端生成「一次性簽名網址」，或改由後端中繼（proxy）取得圖片再回傳給前端
* 範例：

  ```python
  def get_signed_url(filename):
      return r2_client.generate_presigned_url(...)
  ```

### ✅ API 的防守方式：

* 所有資料查詢 API 必須驗證使用者身份：

  * 檢查 token 是否有效
  * 從 token 解出 user\_id
  * 查詢時加上條件限制：`WHERE owner_id = current_user.id`

### ✅ 清理 API 回傳內容：

* 使用 DTO / Serializer 控制只回傳需要的欄位
* 移除 `s3_key`、`internal_path`、`api_token`、`owner_id` 等敏感資訊

---

## 🧠 額外補充：什麼算是機敏資訊？

| 類型    | 範例                                                 |
| ----- | -------------------------------------------------- |
| 使用者識別 | user\_id、email、電話、身份證字號                            |
| 存取資訊  | access\_token、refresh\_token、session\_id、cookie id |
| 後台密鑰  | API Key、Cloud Storage 實體路徑、S3 key、資料夾名稱            |
| 角色與權限 | isAdmin、role、permission\_list                      |

✅ 如果你有任何一欄**前端不需要用到，卻還回傳出來**，那就是一個風險點。

---

## ✅ 小結：查詢操作常見漏洞與防守建議

| 漏洞類型      | 錯誤做法                 | 防守方式                 |
| --------- | -------------------- | -------------------- |
| 圖片可直接連結   | R2/S3 實體網址寫死在 HTML   | 產生簽名 URL、使用 proxy 中繼 |
| API 無授權邏輯 | 沒有驗證 token、沒有篩選 user | 加入身份驗證 + 權限條件查詢      |
| 回傳過多欄位    | 資料表欄位全送              | 使用 DTO/Schema 過濾必要欄位 |

---
 