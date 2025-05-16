# 筆記地圖
# CH1 專案導覽與部署地圖

## 1.1 專案介紹與部署全圖

* 專案目標與核心功能介紹
* 專案選型與技術棧

  * 前端部署：Vercel
  * 後端部署：Render
  * 資料庫：ClawCloud
  * 圖床服務：Cloudflare R2
* 系統架構圖

  * 各模組如何串聯、溝通
  * 為什麼不選 serverless 或一體化平台？
* 備選方案比較與淘汰原因（Firebase、Supabase、VPS、Kubernetes 等）
* 資料流節點概觀（CH2 詳述）

  * router / service / schema / dao 的責任分工邏輯

---

## 1.2 系統架構與模組分工

* 核心功能與資料流介紹（不含資安考量）

  * 商品新增流程概述（CRUD 初探）
  * 圖片儲存與查詢流程
* 泳道圖

  * 使用者點擊後，資料如何在前後端間流動？
  * 常見的資料流瓶頸點（資料庫連線、圖片延遲等）
* 架構設計的可維護性與可擴展性初探

 ---

## 1.3 內外網與 NAT 網路概念

* 網路通訊基礎知識

  * Router、IP、Port 是什麼？
  * 網卡、網段、子網路簡介
* 127.0.0.1 vs 公網 IP vs ngrok

  * localhost (127.0.0.1) 真實意義與應用
  * 為什麼朋友無法連上你的網頁？
  * 為什麼前端部署到外網後功能會失效（新增不了商品）？
* NAT (Network Address Translation) 運作原理

  * NAT 如何讓內網與外網溝通？
  * ngrok 怎麼打洞與穿透防火牆？
  * ngrok 的限制與替代方案（如 Cloudflare Tunnel）
*之後要分享vibe coding的作品時，不要再分享localhost:8000了*

---

## 1.4 系統資安概觀

* 資料流中的安全風險

  * 哪些地方容易洩漏資訊？
  * .env 檔案外漏的風險
  * 金鑰、token 管理與最佳實踐
* 登入與權限控制概念

  * 驗證哪些東西？
  * JWT vs Session 的差異（初探）
  * HttpOnly、Secure、SameSite 的重要性
* 圖床安全機制

  * 圖片為什麼要簽名，不能直接公開？
  * 圖床安全：R2 vs 公開連結的差異
* CORS（跨域資源共享）

  * CORS 的用途與重要性
  * 忘記設定 CORS 的實務後果（案例簡介）

---

## 1.5 自動部署與測試流程

* 部署基本流程介紹

  * GitHub Actions 的用途與基本觀念
  * 本地測試與線上環境的差異
  * 環境變數如何管理？本地 vs 雲端差異
* 實務上的測試流程

  * 為什麼要測試？
  * 單元測試、整合測試、端到端（E2E）測試簡介
  * 測試案例如何設計（FastAPI + Pytest + Mock 初步介紹）
* 部署與測試時常遇到的問題

  * 網路、效能、第三方 API 等常見錯誤點
  * 部署後測試點有哪些（build → deploy → 測試實務）

--- 

### 📙 CH2. 實作細節（實際怎麼做 + 結構怎麼拆）

> 「從底層逐步長出一個完整服務，邊走邊學觀念」

#### 2.1 開發環境與共通工具

* VSCode、Git、Terminal 基本指令
* `.env` 怎麼讀、怎麼設計開發與部署變體？
* `requirements.txt` vs `poetry` 是什麼？（如果有用）

#### 2.2 後端模組與結構（FastAPI）

* 專案資料夾分層（schema/router/service/dao）
* Pydantic Schema 預設值設計、欄位驗證、轉 camelCase
* FastAPI 路由拆分原則（prefix、tags）
* 👉 **\[補]：JWT 權限驗證怎麼做？Exception handler 怎麼用？**
* Docker 打包與環境參數傳遞（含 `.env`）

#### 2.3 資料庫層與 ORM

* 連線與啟動設定
* Alembic or 手動建表流程
* 👉 **\[補]：如何設計一個 schema（e.g., 商品 + 圖片）才有擴充性？**

#### 2.4 圖片系統

* 資料夾路徑、圖片命名規則
* 本地存 vs R2 上傳 vs 簽名 URL
* 👉 **\[補]：使用者能看到的圖片 URL 是怎麼組出來的？**

#### 2.5 前端模組（React + Zustand）

* 前端資料夾結構與架構設計
* 狀態管理：productStore, imageStore
* 表單資料傳送與圖片上傳流程
* 👉 **\[補]：如何切分元件？如何避免 useEffect 造成錯誤？**
* 👉 **\[補]：環境變數與 baseURL 統一管理的設計**

#### 2.6 前端部署與資安設計

* Vercel 上部署流程與 CI 設定
* `fetch` 資安設計：JWT、Authorization Header
* 👉 **\[補]：如何防止前端硬編密鑰？怎麼避免圖片被隨便讀？**


---

## CH2. 實作細節（備案）
*補充：terminal*
- 後端
    - 補充：怎麼啟動一個python專案、git（不介紹）、web server
    - 本地
    - 後端架構分層
    - 資料夾結構
    - schema 預設值順序
    - 回傳資訊怎麼夾帶
    - 部署 docker
- 資料庫
    - 資料庫服務器
    - 資料庫連線
    
- 圖床
    - 不要暴露圖床網址
    - 簽名？
- 前端
    - refererence
        - https://kuro.tw/posts/2019/07/31/%E8%AB%87%E8%AB%87%E5%89%8D%E7%AB%AF%E6%A1%86%E6%9E%B6/
    - npm/js/瀏覽器v8引擎講解
    - 前端架構分層
    - 前端資料夾結構
    - 前端專案啟動
    - 前端資安
    - 前端部署