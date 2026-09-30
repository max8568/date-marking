# 日期點選標記系統

日期標記工具，可選月份為當月的前 3 個月到後 8 個月，共 12 個月。勾選要看的月份，選一個顏色，然後在月曆上點選或拖曳標記日期。

單一頁面、無建置步驟、無相依安裝——直接用瀏覽器打開 `index.html` 就能用。

線上版本：**https://max8568.github.io/date-marking/**

> 需要連外網。頁面在執行期從 unpkg 載入 React / ReactDOM / Babel，離線無法運作。

---

## 操作方式

| 動作 | 結果 |
|---|---|
| 勾選月份 | 該月份的月曆展開在下方 |
| 點一下沒標記的日期 | 標成目前選色 |
| 點一下**別的顏色**的日期 | 直接改成目前選色（不會取消） |
| 點一下**同顏色**的日期 | 取消標記 |
| 按住往外拖曳 | 指標經過的格子連續標記，可跨月曆卡片 |
| 從**同顏色**的日期起拖 | 整段擦除 |
| Tab + Enter／Space | 與單擊相同 |

拖曳採**筆刷**語意——只作用在指標實際經過的格子，不會自動填滿起點到終點之間的日期。起點決定整段是「上色」還是「擦除」，中途不會改變。

三個控制鍵：`全選`（展開 12 個月）、`清除月份`（收合全部，標記保留）、`清除標記`（清空所有標記，月份保留）。

**觸控裝置**只支援單點標記，沒有拖曳。這是刻意的：拖曳要靠 `touch-action: none`，會讓手指從日期格開始滑動時無法捲動頁面，而月曆通常佔滿整個畫面。

標記**不會被保存**。重新整理就清空。

---

## 檔案

```
index.html    整個應用程式：版面樣板 + 元件邏輯
support.js    DC runtime（產生檔，勿手改）
_ds/          Industry 設計系統：styles.css 提供字體與色彩 token
uploads/      設計參考圖
.thumbnail    工具產生的預覽縮圖
```

`index.html` 是一個 **DC（Design Component）**頁面，由 `support.js` 這個 runtime 驅動。結構分兩塊：

- `<x-dc>` — 宣告式樣板。用 `{{ }}` 綁值、`<sc-for>` 迭代、`<sc-if>` 條件顯示。**樣板裡沒有任何邏輯。**
- `<script type="text/x-dc">` — 一個 `class Component extends DCLogic`。Babel 在瀏覽器裡即時編譯它。

runtime 從 `<helmet>` 取出 `<link>` / `<script>` / `<style>` 搬進 `<head>`，把 `<x-dc>` 換成 React 掛載點，然後以 `renderVals()` 回傳的物件為資料來源渲染樣板。

---

## 程式結構

`renderVals()` 回傳的每個 key 就是樣板能綁的名字；所有畫面資料都在這裡組好，樣板只負責擺放。

### State

```js
state = { months: [], days: {}, color: 'blue', drag: null }
```

| 欄位 | 內容 |
|---|---|
| `months` | 展開中的月份，存的是 `MONTHS` 的索引 `0`–`11`（`0` = 當月往前第 3 個月），遞增排序 |
| `days` | 標記表，`{ '2026-09-13': 'blue', ... }`，鍵是可排序的 `YYYY-MM-DD` |
| `color` | 目前選用的顏色 id |
| `drag` | 拖曳中為 `'mark'` / `'clear'`，否則 `null` |

### 標記規則只有一份

點擊與拖曳共用同兩個純函式，避免兩條路徑各自演化：

```js
const modeFor = (days, key, color) => (days[key] === color ? 'clear' : 'mark');
const applied  = (days, key, mode, color) => { /* 回傳新的 days */ };
```

`modeFor` 就是「同色才清除，其餘一律上色」這條規則的唯一出處。`applied` 是冪等的，所以拖曳時重複進入同一格不會有副作用。

### 顏色

五個標記色定義在 `PALETTE`，各自帶 `bg`（填色）／`fg`（文字）／`ring`（1px 內描邊）。其餘灰階與淺藍色調集中在 `C` 物件——**新增顏色請加進這兩處，不要散在樣式字串裡。**

樣式是拼接字串而非 CSS class，因為 DC 樣板綁的是 `style=""`。每段樣式都提到模組層具名（`dayStyle`、`chipStyle`、`swatchStyle`…），讓 `renderVals()` 專心組資料。

---

## 拖曳實作上的三個坑

這三段程式碼看起來可以刪，但刪了就會壞：

**1. `mouseup` 掛在 `window` 上，不是格子上。** 使用者很常在格子之間的縫隙、其他區塊、甚至頁面外放開滑鼠。掛在格子上會漏掉這些情況，讓筆刷卡住。

**2. `enterDay()` 檢查 `e.buttons === 0`。** 如果在**瀏覽器視窗外**放開，`window` 的 `mouseup` 根本不會送達。這個檢查在指標回到頁面時補上收尾。

**3. `onClick` 只在 `e.detail === 0` 時作用。** `mousedown` 已經標記了該格，如果 `onClick` 也切換一次就會互相抵銷，等於單擊沒反應。鍵盤觸發的 click 其 `detail` 為 `0`，滑鼠則是點擊次數——所以這個判斷把 click 收斂成純鍵盤路徑，`<button>` 的無障礙行為得以保留。

---

## 驗證

沒有測試套件。變更後用 headless Chrome 檢查頁面仍正常掛載：

```bash
# 從 repo 根目錄執行
chrome --headless=new --disable-gpu --virtual-time-budget=12000 --dump-dom \
  "file://$PWD/index.html"
```

輸出應含 `id="dc-root"`、不應殘留 `<x-dc>`。

要自動驗證互動，可在頁面尾端注入探針腳本，程式化派送滑鼠事件。兩個實測得到的注意事項：

- **要派送 `mouseover`／`mouseout`，不是 `mouseenter`。** React 18 的 `onMouseEnter` 是從前兩者合成的，直接派送 `mouseenter` 不會觸發任何 handler。
- **不要用 `getComputedStyle()` 判斷有沒有標記。** 日期格有 `transition: background .12s`，而 `--virtual-time-budget` 下 transition 不會推進，computed 值會一直停在起始的透明色——即使 inline style 已經是正確顏色。請改讀元件 state 或 `getAttribute('style')`。

樣式類的改動可以用截圖雜湊比對來證明沒有動到外觀：固定一組 state，改動前後各截一張圖，比 SHA256。

---

## 已知限制

- 可選月份在頁面載入時依當天日期算出（`MONTHS`）。頁面開著跨過月底不會自動更新，要重新整理。
- 標記只存在記憶體，沒有匯出、匯入或保存。
- 顏色沒有語意，只有「藍／紫／綠／黃／紅」，無法命名各代表什麼。
- 頁面雖然連結了 `_ds/` 的 Industry 設計系統，但只取用其字體 token（`--font-heading` / `--font-body`）。版面採白底圓角柔陰影的淺色調，而非該系統規範的直角髮絲線藍圖風格——這是刻意的取捨。
