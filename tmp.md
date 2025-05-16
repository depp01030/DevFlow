

#### 回合五：前端洩漏的第三方 API 金鑰

你使用了 Google Maps API 來顯示地圖，並把 API Key 直接寫在前端 JavaScript：

```javascript
// 前端 map.js (極度危險！)
// const Maps_API_KEY = "AIzaSyYOUR_VERY_REAL_API_KEY_HERE"; // ❌❌❌
// const script = document.createElement('script');
// script.src = `https://maps.googleapis.com/maps/api/js?key=${Maps_API_KEY}&callback=initMap`;
// document.head.appendChild(script);
```

**駭客攻擊：F12 Sources 一秒偷走你的金鑰**

駭客打開 F12 -\> Sources (原始碼)，找到你的 JS 檔案，API Key 就暴露了。他們可以用你的 Key 瘋狂呼叫 Google Maps API (或其他任何你洩漏的第三方服務金鑰，如 AWS、OpenAI)，導致你的帳單暴增破產。

**你的防守：金鑰永不露面，後端代理請求**

  * **金鑰存後端**：將 API Key 儲存在後端伺服器的環境變數中，絕不寫入程式碼，更不能傳到前端。
  * **後端代理**：前端請求你自己的後端 API，你的後端再拿著金鑰安全地去請求第三方服務，然後將結果回傳給前端。

<!-- end list -->

```python
# 後端 main.py (代理請求概念)
import httpx # pip install httpx, 現代化的 HTTP 客戶端
import os

# 從環境變數讀取，例如: export THIRD_PARTY_API_KEY="your_actual_key"
THIRD_PARTY_API_KEY = os.environ.get("THIRD_PARTY_API_KEY_EXAMPLE") 

@app.get("/api/proxy/some-service")
async def proxy_some_service(query: str, current_user: dict = Depends(get_current_user_from_token)):
    if not THIRD_PARTY_API_KEY:
        raise HTTPException(status_code=503, detail="服務暫時不可用 (金鑰未配置)")

    # 這裡可以加入你自己的快取、速率限制、日誌等邏輯
    # current_user 可用於判斷此用戶是否有權限使用此代理服務
    
    external_service_url = f"https://api.example.com/data?query={query}&apiKey={THIRD_PARTY_API_KEY}"
    
    async with httpx.AsyncClient() as client:
        try:
            response = await client.get(external_service_url)
            response.raise_for_status() # 如果是 4xx 或 5xx 會拋出異常
            # 你可能只想回傳部分數據，或對數據進行處理
            return response.json() 
        except httpx.HTTPStatusError as e:
            # 不要直接回傳第三方錯誤的詳細內容，避免洩漏過多資訊
            error_detail = f"外部服務錯誤: {e.response.status_code}"
            if e.response.status_code == 401 or e.response.status_code == 403:
                 error_detail = "外部服務認證失敗 (可能是伺服器金鑰問題)"
            raise HTTPException(status_code=status.HTTP_502_BAD_GATEWAY, detail=error_detail)
        except Exception as e:
            raise HTTPException(status_code=status.HTTP_500_INTERNAL_SERVER_ERROR, detail=f"代理請求時發生未知錯誤: {str(e)}")

# 前端 JS 呼叫你的代理 API
# fetch("/api/proxy/some-service?query=something")
#   .then(res => res.json())
#   .then(data => console.log(data));
```

這樣，第三方 API Key 永遠不會離開你的伺服器。

-----

### 3️⃣ 資料查詢的隱私攻防：你的 API 在洩密嗎？
 
-----

#### 小結：哪些資料算機敏？

  * **個人身份識別資訊 (PII)**：完整姓名、身分證號、完整地址、電話、Email (除非刻意公開)、生日。
  * **帳戶/系統資訊**：**密碼/密碼雜湊 (絕對禁止)**、API 金鑰、內部 ID (除非刻意設計為公開 ID)、詳細錯誤訊息。
  * **業務/營運資訊**：未公開的財務數據、客戶名單等。

**原則：預設不給，按需提供。不確定的，就當作機敏處理。**

-----

### 📎 補充篇 (精簡概念)

#### JWT (JSON Web Token)

一種開放標準 (RFC 7519)，用於安全地在雙方之間傳輸資訊的 Token。

  * **過期 (`exp`)**：Token 有效期限，過期後失效。
  * **撤銷 (Revocation)**：讓未過期的 Token 失效 (如登出、改密碼)。通常用黑名單機制 (如 Redis 存 JTI)。
  * **刷新 (Refresh Token)**：用一個長效期的 Refresh Token 去換取短效期的 Access Token，兼顧安全與體驗。

#### F12 / 開發者工具

駭客的第一個情報站。他們能：

  * 看 HTML/CSS/JS 原始碼。
  * 分析網路請求 (API 端點、參數、回應)。
  * 執行任意 JavaScript (偷 localStorage、繞過前端驗證)。
  * 修改 DOM 結構 (顯示隱藏元素)。
  * 查看 Cookie、localStorage 等儲存。

#### 圖床安全進階

  * **URL 簽名 (Signed URLs)**：產生帶有簽名和過期時間的圖片 URL，常用於 CDN (如 AWS S3 Presigned URL)。防止盜連和永久連結。
  * **後端 Relay/Proxy**：圖片不直接暴露 URL，而是透過後端 API 進行權限驗證後再串流給前端。完全掌控存取，但增加伺服器負載。

-----

### 🛡️ 總結：安全無小事，攻防永不息

看完這些攻防演練，你應該體會到，網站安全並非一勞永逸，而是一場持續的攻防戰。駭客總在尋找新的突破口，而開發者則需要不斷學習、保持警惕，從每個功能設計之初就融入安全思維。

記住：

  * **永遠不要相信用戶輸入。**
  * **後端是最後的防線。** 

希望這篇文章能幫助你打造更安全的網路世界！