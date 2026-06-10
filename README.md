# 🥞 厚鬆餅堡早餐訂購系統

Vue 3 + Vite 製作的早餐團購訂單網站,後端資料庫連結 Microsoft 365 的 Excel,可即時更新訂單、自動加總金額、標記付款狀態。

未完成 M365 設定前,網站會自動以「本機展示模式」運作(資料存在瀏覽器),完成設定後即自動切換為「M365 Excel 即時連線」。

---

## 一、快速開始(本機展示)

需要先安裝 [Node.js](https://nodejs.org/)(18 以上)。

```bash
# 1. 安裝套件
npm install

# 2. 啟動開發伺服器
npm run dev
# 瀏覽器開啟 http://localhost:5173

# 3. 打包成可上線的靜態檔(產生 dist/ 資料夾)
npm run build
```

此時不需要任何設定即可操作:選餐 → 加入購物車 → 輸入姓名 → 送出訂單,
訂購內容會出現在右側,可切換付款狀態、刪除、自動加總,並可「匯出 Excel(CSV)」。

---

## 二、餐點與價格

| 餐點 | 單點 | 薯餅套餐 |
|------|------|----------|
| 香鷄蛋厚鬆餅堡(NEW) | $79 | $124 |
| 豬肉蛋厚鬆餅堡 | $79 | $124 |
| 火腿蛋厚鬆餅堡 | $71 | $116 |
| 豬肉厚鬆餅堡 | $69 | $114 |

要修改價格或餐點,編輯 `src/data/menu.js` 即可。

### 換上商品照片
把四張商品圖放進 `public/images/`,檔名對應:
`chicken-egg.png`、`pork-egg.png`、`ham-egg.png`、`pork.png`。
找不到圖片時會自動顯示預設的鬆餅圖示,不影響功能。

---

## 三、連結 Microsoft 365 Excel(即時資料庫)

純前端網站要「即時讀寫 M365 Excel」,需透過 **Microsoft Graph API**,
並在 **Azure** 註冊一個應用程式取得登入權限。以下為一次性設定,完成後永久有效。

### 步驟 1：在 OneDrive / SharePoint 建立訂單 Excel

1. 在公司的 OneDrive 或 SharePoint 建立一個 Excel 檔,例如 `早餐訂單.xlsx`。
2. 在第一個工作表的 A1 起,輸入以下 9 個欄位標題:

   | A | B | C | D | E | F | G | H | I |
   |---|---|---|---|---|---|---|---|---|
   | 訂單編號 | 訂購人 | 餐點 | 規格 | 單價 | 數量 | 小計 | 付款狀態 | 建立時間 |

3. 選取 A1:I1(含標題的這一列),點 **插入 → 表格**,勾選「我的表格有標題」。
4. 選取表格後,在左上角「表格名稱」欄把名稱改為 **`Orders`**(要與設定一致)。
5. 儲存。

> 程式會把每筆訂單新增為表格的一列,並用「付款狀態」欄寫入「已付款 / 未付款」。

### 步驟 2：在 Azure 註冊應用程式

1. 登入 [Azure 入口網站](https://portal.azure.com) →「Microsoft Entra ID」→「應用程式註冊」→「新增註冊」。
2. 名稱填 `早餐訂購系統`。
3. 支援的帳戶類型:選「僅此組織目錄中的帳戶」(公司內部使用)。
4. 重新導向 URI:平台選 **單頁應用程式 (SPA)**,網址填 `http://localhost:5173`
   (之後正式上線網址也要回來這裡一起加上,例如 `https://你的網域`)。
5. 建立後,在「概觀」頁記下兩個值:
   - **應用程式 (用戶端) 識別碼** → 對應 `VITE_MS_CLIENT_ID`
   - **目錄 (租用戶) 識別碼** → 對應 `VITE_MS_TENANT_ID`

### 步驟 3：設定 API 權限

1. 左側「API 權限」→「新增權限」→「Microsoft Graph」→「委派的權限」。
2. 勾選:`User.Read`、`Files.ReadWrite`、`Sites.ReadWrite.All`。
3. 加入後,點「代表組織授與管理員同意」(若你不是管理員,請 IT 協助按這個)。

### 步驟 4：取得 Excel 檔的 item id

訂單 Excel 在 OneDrive 上的 item id,可用瀏覽器取得 access token 後查詢,
最簡單的方式是用 [Graph Explorer](https://developer.microsoft.com/graph/graph-explorer):

1. 用公司帳號登入 Graph Explorer。
2. 查詢:`GET https://graph.microsoft.com/v1.0/me/drive/root/search(q='早餐訂單')`
3. 回傳結果中的 `id` 欄位就是 **item id** → 對應 `VITE_EXCEL_ITEM_ID`。

> 若檔案放在 SharePoint 團隊網站而非個人 OneDrive,`excelService.js` 中的
> `/me/drive/items/...` 需改為 `/sites/{site-id}/drive/items/...`,做法已於檔案內註解說明。

### 步驟 5：填寫 .env

把 `.env.example` 複製成 `.env`,填入上面取得的值:

```
VITE_MS_CLIENT_ID=你的用戶端識別碼
VITE_MS_TENANT_ID=你的租用戶識別碼
VITE_MS_REDIRECT_URI=http://localhost:5173
VITE_EXCEL_ITEM_ID=你的Excel item id
VITE_EXCEL_TABLE_NAME=Orders
```

重新執行 `npm run dev`。畫面右上角會顯示「M365 Excel 即時連線」,
首次送出訂單會跳出 Microsoft 登入視窗,授權後訂單就會即時寫入 Excel,
多人開啟網頁、按右側「⟳ 更新」即可看到彼此的最新訂單與付款狀態。

---

## 四、即時更新與加總是怎麼運作的

- **新增訂單**:`POST .../workbook/tables/Orders/rows/add` 新增一列。
- **讀取訂單**:`GET .../workbook/tables/Orders/rows`,按「更新」即重新抓取。
- **付款狀態**:`PATCH .../rows/itemAt(index=n)` 更新該列的「付款狀態」欄。
- **加總金額**:由前端即時計算總金額、已收、未收(也可在 Excel 內用 SUM 公式)。

> 想要「全自動即時同步」(不必按更新),可改用 Graph 的 webhook/訂閱,
> 或在 `orderStore.js` 加一個 `setInterval(refresh, 15000)` 每 15 秒自動刷新。

---

## 五、檔案結構

```
breakfast-order/
├─ index.html
├─ package.json
├─ vite.config.js
├─ .env.example          ← 複製成 .env 填入 M365 設定
├─ public/images/        ← 放商品照片
└─ src/
   ├─ main.js
   ├─ App.vue            ← 主畫面(菜單 + 購物車 + 訂購內容)
   ├─ data/menu.js       ← 餐點與價格
   ├─ auth/msalConfig.js ← Microsoft 登入設定
   ├─ services/
   │  ├─ excelService.js ← Graph Excel 讀寫
   │  └─ orderStore.js   ← 資料層(Excel / 本機自動切換)
   └─ components/
      ├─ MenuItem.vue    ← 單一餐點卡片
      └─ OrderSummary.vue← 訂購內容 / 加總 / 付款狀態
```
