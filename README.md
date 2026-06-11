# Sunmi 列印測試專案

**目標**：驗證 Sunmi 機器能否從網頁觸發內建熱感印表機列印收據。

---

## 三種方式比較 + 建議測試順序

### 方式 1：列印外掛 + `window.print()` ← 先測這個

| 項目 | 說明 |
|------|------|
| 難度 | 最簡單 |
| 需要安裝 | Sunmi App 市場的「列印服務（PrintService）」外掛 |
| 原理 | 安裝後，Android 列印框架認得內建印表機，`window.print()` 直接呼叫它 |
| 限制 | 排版受瀏覽器 Print Preview 影響，需要 `@media print` CSS 微調 |

### 方式 2：Sunmi 瀏覽器內建 JS API ← 方式 1 不行時試

| 項目 | 說明 |
|------|------|
| 難度 | 中等 |
| 需要安裝 | Sunmi 官方瀏覽器（SunmiBrowser）|
| 原理 | Sunmi Browser 把印表機 SDK 注入成 `window.SunmiPrinter` 物件，可用 JS 直接呼叫 ESC/POS 指令 |
| 限制 | 只在 Sunmi Browser 裡有效，Chrome/Firefox 無效 |

### 方式 3：Android WebView 殼 + AIDL ← 最後手段

| 項目 | 說明 |
|------|------|
| 難度 | 最難（需要 Android Studio）|
| 需要安裝 | 打包一個 Android APK，把網頁包進去 |
| 原理 | APK 透過 AIDL 直接連接 Sunmi PrinterService，網頁用 `window.AndroidBridge.print(text)` 呼叫 |
| 限制 | 需要編譯 Android 專案，但列印效果最穩定 |

---

## 方式 1 操作步驟（最詳細）

### 步驟 0：把這個測試頁放到網路上

最簡單的作法是用 Cloudflare Pages（你已有 Cloudflare 帳號）：

1. 把 `sunmi-print-test/` 資料夾上傳到 GitHub
2. Cloudflare Dashboard → Pages → 建立專案 → 連接 GitHub Repo
3. Build 設定全部留空（純 HTML，不用 build），直接 Deploy
4. 取得像 `https://sunmi-test.pages.dev` 這樣的 URL

> 或者臨時測試：在你的電腦終端機執行 `npx serve sunmi-print-test`，  
> Sunmi 機器和電腦連同一個 WiFi，然後在 Sunmi 瀏覽器輸入 `http://你電腦的IP:3000`。

---

### 步驟 1：在 Sunmi 機器上安裝列印外掛

1. 在 Sunmi 機器找到 **App Market（應用市場）** 圖示，點開
2. 在搜尋欄輸入：`列印服務` 或 `Print Service`
3. 找到官方的 **Sunmi PrintService**（開發者是 SUNMI），安裝它
4. 安裝完成後**重新啟動瀏覽器**

> 如果 App Market 找不到，也可以試著搜尋 `PrinterPlugin`。

---

### 步驟 2：在 Sunmi 瀏覽器開啟測試頁

1. 打開 Sunmi 機器上的**瀏覽器**（Chrome 或 Sunmi Browser 都可以）
2. 輸入你的測試頁網址（Cloudflare Pages 或本機 IP）
3. 畫面上會看到一張模擬收據的預覽
4. 頂部有「選擇紙張寬度」下拉選單：
   - T2 mini → 選 **58 mm**
   - T2 / T2s → 選 **80 mm**
   - D3 Pro → 選 **80 mm**

---

### 步驟 3：嘗試列印

1. 點擊藍色「**列印收據**」按鈕
2. 如果成功：會彈出「選擇印表機」或直接跳 Print Preview
   - 在印表機清單裡找「**內建印表機**」或「**Sunmi Printer**」，點選
   - 點確認，機器會吐出一張收據
3. 如果失敗：看下一節的排錯指引

---

### 排錯

| 症狀 | 可能原因 | 解法 |
|------|----------|------|
| 按鈕沒反應 | JS 錯誤 | 長按網頁 → 「檢查元素」或 Chrome DevTools 看 console |
| 彈出列印但只有 PDF/系統印表機 | PrintService 沒裝或沒重啟瀏覽器 | 重裝外掛、重啟瀏覽器 |
| 印出來但版面很亂 | `@media print` CSS 沒生效 | 試試不同瀏覽器（Sunmi Browser vs Chrome）|
| 印出空白 | 印表機服務沒跑起來 | 重啟 Sunmi 機器後再試 |

---

## 方式 2：Sunmi Browser JS API（方式 1 不行時）

1. 確認機器上有 **Sunmi Browser**，沒有的話從 App Market 安裝
2. 打開 Sunmi Browser，開啟測試頁
3. 在開發者 console 輸入以下測試：

```javascript
// 確認 API 是否存在
console.log(typeof window.SunmiPrinter)
// 如果輸出 "object"，代表 API 注入成功

// 嘗試印出文字
window.SunmiPrinter.printerText("測試文字\n")
window.SunmiPrinter.lineWrap(3)
```

4. 如果 `typeof window.SunmiPrinter` 是 `"object"`，代表可以用，我再幫你把測試頁升級成使用這個 API。

> 注意：不同 Sunmi 韌體版本的 JS API 名稱可能略有差異，以官方文件 docs.sunmi.com 為準。

---

## 方式 3：Android WebView 殼（方式 1、2 都不行時）

這個需要另外開發，步驟概覽：

1. 安裝 **Android Studio**（免費，官方 IDE）
2. 建立新的 Android 專案，加入 Sunmi Printer SDK 依賴
3. 用 `WebView` 元件載入你的網頁
4. 建立 `JavascriptInterface`，讓網頁能透過 `window.AndroidBridge.print(text)` 呼叫原生印表機
5. 打包成 APK，安裝到 Sunmi 機器

這個階段需要用 Claude Code 另外規劃，先確認方式 1 或 2 不可行再說。

---

## 測試記錄（請填寫）

```
測試日期：
Sunmi 機型：
Android 版本：
測試瀏覽器：

方式 1 結果：
  - PrintService 安裝成功？ Y/N
  - 按列印有彈出印表機選單？ Y/N
  - 有出現內建印表機選項？ Y/N
  - 成功印出？ Y/N
  - 印出的版面正常？ Y/N
  - 備註：

方式 2 結果（如有測試）：
  - Sunmi Browser 有安裝？ Y/N
  - window.SunmiPrinter 存在？ Y/N
  - 備註：
```

---

## 檔案說明

```
sunmi-print-test/
└── index.html    # 測試收據頁面，包含 screen + print 兩套 CSS
```
