# Home Fitness

Home Fitness 是一個由職訓課程專案持續重構與完善而成的原生 PHP 電商作品，涵蓋商品瀏覽、會員、匿名／會員購物車、結帳與訂單查詢流程。除了購物介面，專案也逐步改善購物車歸屬、交易式結帳、訂單快照、舊密碼遷移與伺服器端驗證。本專案用於展示前端、PHP 後端與全端開發能力，並非實際營運中的商業平台。

## 線上展示

- Demo：https://homefitness-demo.infinityfreeapp.com/
- 此站為作品集展示環境，請使用虛構測試資料，勿輸入真實個人資訊。
- 可自行註冊測試會員，體驗匿名購物車、會員註冊、購物車合併、結帳與訂單查詢流程。

## 專案畫面

### 首頁

![Home Fitness 首頁](docs/screenshots/01-home.png)

### 購物流程

| 商品列表 | 商品詳細 |
| --- | --- |
| ![商品列表](docs/screenshots/02-products.png) | ![商品詳細](docs/screenshots/03-product-detail.png) |

| 購物車 | 訂單完成 |
| --- | --- |
| ![購物車](docs/screenshots/04-cart.png) | ![訂單完成](docs/screenshots/05-order-complete.png) |

響應式頁面亦針對手機版進行調整：

<img src="docs/screenshots/06-mobile-product-detail.png" width="320" alt="商品詳細手機版">

## 專案簡介

以虛構的居家健身品牌為情境，串起從選購到查詢歷史訂單的使用者流程。前端以共用 PHP 元件、分檔 CSS 與 Bootstrap 製作響應式頁面；後端採原生 PHP 與 PDO，將購物車歸屬、資料驗證和訂單一致性放在伺服器端處理。

## 主要功能

| 範圍 | 已完成的使用者功能 |
| --- | --- |
| 商品 | 首頁輪播與推薦商品、二層分類、名稱搜尋、商品列表與分頁、詳細資訊與多圖燈箱；單項數量為 1～49。 |
| 會員 | Email 註冊、可選 Avatar 上傳、登入與登出；註冊成功後自動登入。 |
| 購物車 | 匿名與會員皆可加入商品、修改數量、刪除單項或清空；登入或註冊後自動合併匿名購物車。 |
| 結帳 | 必須登入；預填會員收件資料，可修改本次訂單資料而不回寫會員地址。僅支援貨到付款，固定運費 NT$100；建單時重新驗證商品並由伺服器重新計價。 |
| 訂單 | Order Complete、我的訂單與訂單明細；保存商品及收件資訊快照，訂單編號格式為 `HFYYYYMMDD####`。 |
| 內容 | Brand 品牌介紹與 FAQ 常見問題。 |

## 核心購物流程

匿名使用者點選結帳時會先導向登入；登入成功可返回結帳頁。若直接從註冊頁建立帳號，註冊成功目前會返回首頁，購物車合併後可再前往結帳。

```mermaid
flowchart TD
    A[匿名瀏覽商品] --> B[加入匿名購物車]
    B --> C{已登入？}
    C -- 否 --> D[登入或註冊]
    D --> E[交易式合併購物車]
    E --> F[前往 Checkout]
    C -- 是 --> F
    F --> G[確認本次收件資料]
    G --> H[鎖定購物車與商品]
    H --> I[伺服器重新驗證與計價]
    I --> J[交易式建立訂單快照並清車]
    J --> K[Order Complete]
    K --> L[我的訂單]
    L --> M[訂單明細]
```

## 技術棧

| 層面 | 使用技術 |
| --- | --- |
| 後端 | PHP 8.2+、PDO MySQL；原生 PHP，未使用 PHP Framework。 |
| 資料庫 | MariaDB／MySQL。公開範例資料庫由 MariaDB 匯出。 |
| 前端 | HTML5、CSS、JavaScript、Bootstrap 5.3.8、jQuery 3.7.1、jQuery Validation Plugin 1.13.0、PhotoSwipe 5.4.4、Font Awesome 7.3.0。 |

## 核心工程亮點

1. **交易式結帳**：建立訂單不只是寫入訂單表。同一資料庫 transaction 中鎖定會員購物車與商品、檢查上架狀態和數量、從資料庫價格重新計算金額、建立訂單與明細快照，最後刪除已結帳購物車；任一步失敗即 rollback。
2. **匿名／會員購物車歸屬與合併**：匿名購物車以隨機 Session token 識別，原值留在 Session，資料庫只保存 SHA-256 hash；會員購物車以登入會員 ID 限定範圍。更新及刪除都同時檢查 owner，不只依賴 cart ID。登入或註冊後，以 transaction 合併相同商品並限制數量上限。
3. **訂單快照**：訂單明細保存成交時的商品名稱、單價、數量及小計，訂單主檔保存當次收件資料。歷史訂單因此不依賴之後可能變動的商品名稱、價格或會員地址。
4. **舊會員密碼漸進式升級**：新會員密碼以 `password_hash(PASSWORD_DEFAULT)` 保存；登入使用 `password_verify()`，並在需要時重新雜湊。舊會員的 legacy MD5 密碼在伺服器端成功驗證後立即升級為現代 password hash；前端不計算 MD5。
5. **避免重複建單**：Checkout submission token 的 hash 同時由 Session 驗證並在訂單資料庫欄位加上唯一約束。重送已成功的請求會導回既有訂單，避免快速雙擊或網路重送產生重複訂單。

## 安全與資料一致性設計

| 設計 | 做法 |
| --- | --- |
| Session | Strict mode、HttpOnly、SameSite=Lax；HTTPS 請求使用 Secure cookie，登入與註冊自動登入前更新 Session ID。 |
| 請求與資料 | 修改狀態的請求驗證 CSRF token；註冊、購物車與結帳均有伺服器端驗證；PDO prepared statements 處理資料庫輸入。 |
| 存取範圍 | 購物車查詢／修改帶入匿名 token hash 或會員 ID；訂單查詢同時限定訂單編號與登入會員 ID。 |
| 輸出與上傳 | 共用 `e()`、安全圖片檔名與 JavaScript 值編碼 helper；Avatar 驗證實際檔案類型、尺寸與 Session 中的上傳紀錄。 |

商品 `p_content` 是受控商品資料的 HTML 輸出例外；若日後開放一般使用者或後台編輯，需另加 HTML allowlist 清理。詳見[輸入驗證與輸出編碼](docs/input-output-security.md)。

## 本機安裝

1. Clone repository，進入專案根目錄。準備 PHP 8.2+、MariaDB／MySQL，並啟用 PHP 的 `PDO`、`pdo_mysql`、`fileinfo` extension。
2. 使用資料庫管理工具匯入 `database/expstore.sample.sql`。範例會建立 `expstore`，包含目前所需 schema 與商品／地區資料，不包含會員或訂單資料。**Fresh install 不需再執行** `database/migrations/20260913_order_core.sql`；該 migration 供既有舊資料庫升級參考。
3. 複製 `.env.example` 為 `.env`（Windows PowerShell：`Copy-Item .env.example .env`），設定 `DB_HOST`、`DB_PORT`、`DB_DATABASE`、`DB_CHARSET`、`DB_USERNAME`、`DB_PASSWORD`。建議使用本專案專用的資料庫帳號。
4. 確認 `uploads/` 可由 PHP 寫入。在專案根目錄執行 `php -S 127.0.0.1:8000`，開啟 `http://127.0.0.1:8000/`。此網址僅供本機開發；前端 CDN 資源需可連線。

正式部署需使用 HTTPS、設定 PHP `display_errors=Off`、`log_errors=On`，且禁止 `uploads/` 執行 PHP／CGI。本機 PHP 開發伺服器不套用 `.htaccess`，不能用來驗證上傳目錄的部署保護。完整設定見[本機啟動方式](docs/setup.md)及[上傳目錄部署設定](docs/uploads-security.md)。

## 作品範圍與改善歷程

目前付款方式只有貨到付款，訂單建立時狀態為「待處理」。專案尚未實作 Admin 後台、線上信用卡／ATM／第三方金流、庫存管理、物流追蹤、Email 通知、密碼重設、會員資料或地址簿管理，以及取消訂單／退款流程。

在原始課程功能基礎上，逐步將資料庫憑證移至本機設定、把舊 MD5 會員遷移到 `password_hash()`，整理 Session 與 CSRF，將購物車歸屬由舊 IP 欄位改為匿名 token／會員 ID，再補上伺服器端驗證、輸出編碼、交易式結帳、訂單快照與會員限定的訂單查詢，最後調整表單回饋與購物流程 UX。

更多維護資訊：[專案目錄結構](docs/structure.md) · [前端依賴](docs/dependencies.md)。
