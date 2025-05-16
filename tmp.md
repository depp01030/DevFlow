好的，我明白了！我們來調整一下風格，讓文章更像一場精彩的攻防演練，程式碼範例也會更精簡，直指核心。

-----


### 2️⃣ 商品後台的保衛戰：攻破你的 CRUD

**本章你將學會：**

  * 為何前端驗證不可靠，以及後端驗證的重要性。
  * 隱藏功能按鈕為何擋不住駭客。
  * 商品說明中的 XSS 攻擊與防禦。
  * 圖片上傳的常見漏洞與安全措施。
  * 為何第三方 API 金鑰絕不能放在前端。

-----

#### 回合一：形同虛設的前端驗證

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
    # current_user 可用於記錄操作者或進一步權限檢查
    print(f"User {current_user['username']} is creating product: {product.productName}")
    # 實際儲存到資料庫...
    # product_db.save(product, user_id=current_user['user_id'])
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

<!-- end list -->

```python
# 後端 main.py (使用 Jinja2 模板輸出，預設防 XSS)
from fastapi.templating import Jinja2Templates
from fastapi.requests import Request

templates = Jinja2Templates(directory="templates") # 需要建立 templates 資料夾

# 假設商品資料庫
mock_products = {
    "1": {"name": "安全商品", "description": "<strong>無害的描述</strong>"},
    "2": {"name": "XSS商品", "description": "危險! <script>alert('XSS!')</script>"}
}

@app.get("/products/{product_id}", response_class=HTMLResponse) # FastAPI HTMLResponse
async def read_product(request: Request, product_id: str):
    product = mock_products.get(product_id)
    if not product:
        raise HTTPException(status_code=404, detail="商品不存在")
    # Jinja2 預設會對 description 進行 HTML 逸脫
    return templates.TemplateResponse("product_detail.html", {"request": request, "product": product})
```

````html
<h1>{{ product.name }}</h1>
<p>商品說明：{{ product.description }}</p> ```
如果需要在後端儲存前就清理，可以使用 `bleach`：
```python
# import bleach
# cleaned_description = bleach.clean(user_input_description, tags=['p', 'strong'], strip=True)
# db.save(cleaned_description)
````

-----

*(後續章節將繼續採用此精簡攻防風格編寫)*