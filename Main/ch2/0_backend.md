
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
