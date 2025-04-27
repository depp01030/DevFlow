# 第 1 章：資料庫設計 — 從需求到資料表

>「 一開始先定義好需求與schema，後續開發會更順利 」-- ChatGPT

## 前言
一開始他這樣跟我說的時候，我秉著，反正是自己開發的專案，後面要加就加也沒差，殊不知這是爆炸的開始。
要記錄什麼樣的資訊，要用什麼格式儲存資訊...Update的api要能改哪些資訊等等
這些都要在一開始就設計好。**（否則後面會很麻煩**
對，後面的確是能改，我也同意有些資訊一開始沒辦法定義清楚。
但至少你要努力定義清楚，並且已經定義好的部分就要盡可能的不要改。**（否則後面會很麻煩**

**真的很麻煩＝＝**

## 目標
- 設計能支援商品資料管理的完整資料結構
- 考慮商品細節（價格、庫存、圖片、規格、配送）並合理劃分
- 預留擴充性（未來可支援多平台上架、分類細分等）
> 因為是針對蝦皮上架的商品，所以這邊的資料結構會針對蝦皮的需求來設計

## 預防針

雖然前言的部分講成這樣。但我現在還是只有一張表
我想未來應該會需要拆出兩三張表，來互相關聯
但現在想先做一個ＭＶＰ(最小可行產品)，剩下的等之後再煩惱。
後面一起見證修改的痛苦。（希望不會

---

## 1. 為什麼要先設計資料結構？
- 沒有規劃的資料結構會讓後端難以維護、前端資料難以使用、功能無法擴充。
- 商品資訊常常具有多樣且不穩定特性（例：有些有尺寸，有些沒有），所以必須靈活設計。

---

## 2. Schema 與 Table欄位的差別
- 資料庫 Schema：設計整個資料系統的藍圖（有哪些表、怎麼關聯）
- 資料表欄位（Table Columns）：設計單張表內部的具體資料格式
- 🔵 小提醒：後續在 FastAPI 內的「schema」是 DTO 概念，負責資料傳輸與驗證，不是資料庫設計。

---

## 3. 本系統的資料需求分析

### 核心需求
- 商品基本資訊（名稱、描述、來源）
- 價格與成本資訊（售價、訂購價、運費、關稅、總成本）
- 圖片管理（主圖、附圖、資料夾路徑）
- 顏色、尺寸規格
- 配送選項（支援不同物流）
- 上架狀態、最低購買數量、備貨天數
- 創建與更新時間戳

---

## 4. 思路與取捨

| 功能區塊          | 決策說明 |
|:----------------|:---------|
| 商品基本資料     | `name`, `description`, `stall_name`, `source`, `source_url` |
| 成本與金額管理   | 由訂購價、國際運費、匯率、上架成本推算出總成本與售價，單獨記錄各成本項目，方便後續調整或分析 |
| 商品分類         | 初期以 `custom_type` 欄位取代複雜分類結構，簡化資料庫設計 |
| 圖片管理         | 使用 `item_folder` 來定義商品資料夾，再以 `main_image` 和 `selected_images` 紀錄主圖與附圖 |
| 規格資訊         | `colors` 和 `sizes` 使用 JSON 格式，靈活支援不同商品的變化 |
| 配送方式         | `logistics_options` 存成 JSON 列表，可彈性支援多種物流方案 |
| 上架狀態與控制   | `item_status`、`min_purchase_qty`、`preparation_days` 便於後台管理商品狀態 |
| 時間管理         | 使用 `created_at`, `updated_at` 自動記錄商品建立與更新時間 |

---

## 5. 初版資料表設計（Products）

| 欄位名稱         | 資料型態       | 說明 |
|:----------------|:-------------|:----|
| id              | INT (PK)     | 商品ID，自動遞增 |
| name            | VARCHAR(255) | 商品名稱 |
| description     | TEXT         | 商品描述 |
| purchase_price  | DECIMAL(10,2) | 訂購價（供應商進貨價） |
| total_cost      | DECIMAL(10,2) | 商品總成本 |
| price           | DECIMAL(10,2) | 商品售價 |
| stall_name      | VARCHAR(100)  | 檔口名稱 |
| source          | VARCHAR(255) | 來源 |
| source_url      | VARCHAR(255) | 商品連結 |
| item_status     | ENUM         | 商品狀態（上架/下架） |
| custom_type     | ENUM         | 自訂分類（如洋裝、褲子） |
| material        | VARCHAR(255) | 材質 |
| size_metrics    | JSON         | 尺寸資訊（字典格式） |
| size_note       | TEXT         | 尺寸補充說明 |
| real_stock      | INT          | 實際庫存 |
| item_folder     | VARCHAR(255) | 商品圖片資料夾相對路徑 |
| main_image      | VARCHAR(255) | 主圖片檔名 |
| selected_images | JSON         | 附圖列表（["1.jpg", "2.jpg"]） |
| shopee_category_id | INT        | Shopee 類別ID |
| min_purchase_qty | INT         | 最低購買數量 *預設1*|
| preparation_days | INT         | 備貨天數 *預設30* |
| colors          | JSON         | 顏色選項 |
| sizes           | JSON         | 尺寸選項 |
| logistics_options | JSON       | 配送選項 *預設某四種*|
| created_at      | DATETIME     | 建立時間 |
| updated_at      | DATETIME     | 更新時間 |

---

## 6. 小結
- 本資料表設計考慮了實際電商運營中常見的需求
- 初期設計以簡潔為主，但預留擴充彈性（如分類表、圖片表）
- 未來若需求變更，可以依此為基礎擴展出更多表或關聯設計

---

7. 未來可能的結構調整
目前為了快速完成 MVP，我們把商品圖片（主圖、附圖）直接存在 products 表內，使用 JSON 欄位紀錄。

但隨著系統規模成長，未來可能會將圖片資料獨立成 images 表，設計成一對多關聯（每個商品可以有多張圖片），
這樣可以更靈活地管理圖片，支援圖片排序、主圖標記、不同平台的圖像需求等情境。


## 未來 Images 表設計 (預計)

### 資料表欄位

| 欄位名稱         | 資料型態       | 說明 |
|:----------------|:-------------|:----|
| id              | INT (PK)     | 圖片 ID |
| product_id      | INT (FK)     | 對應的商品 ID |
| image_name      | VARCHAR(255) | 圖片檔名 |
| is_main         | BOOLEAN      | 是否為主圖片 |
| display_order   | INT          | 顯示順序 |
| created_at      | DATETIME     | 上傳時間 |

---

### 關聯設計

- `product_id` 為外鍵（ForeignKey），對應到 `products.id`
- 一個 `product` 可以有多張圖片（One to Many）
- 主圖條件：`is_main = true`
- 其他圖片則為附圖（`is_main = false`）

---
