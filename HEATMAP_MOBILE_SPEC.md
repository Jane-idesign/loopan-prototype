# 台股熱力圖 — iOS / Android 實作規格

> 對應原始碼：`Jane-idesign/twstock-proxy`  
> 規格版本：2026-05-18

---

## 目錄

1. [畫面結構](#1-畫面結構)
2. [資料模型](#2-資料模型)
3. [API 端點](#3-api-端點)
4. [方塊大小計算（getSz）](#4-方塊大小計算getsz)
5. [Treemap 排版演算法（Squarified）](#5-treemap-排版演算法squarified)
6. [顏色系統（pct2color）](#6-顏色系統pct2color)
7. [文字渲染規則](#7-文字渲染規則)
8. [Tooltip 規格](#8-tooltip-規格)
9. [圖例（Legend）](#9-圖例legend)
10. [互動行為](#10-互動行為)
11. [主題（Dark / Light）](#11-主題dark--light)
12. [元件狀態](#12-元件狀態)
13. [實作建議（iOS / Android）](#13-實作建議ios--android)

---

## 1. 畫面結構

```
Screen (375 pt 基準)
├── NavigationBar — 「熱力圖」標題
├── TabBar (上市 / 上櫃)
├── SegmentedControl (市值 / 成交量 / 成交額)
├── TimeRangeBar (即時 / 週 / 二週 / 月 / 季 / 半年 / 年)
├── HeatmapCanvas          ← 核心元件
│   └── [StockTile × N]
│       ├── tileName (股票名稱)
│       └── tilePct  (漲跌幅)
├── Tooltip (點擊後浮現，點畫布其他處消失)
└── Legend (漲停 → 平盤 → 跌停 色條)
```

### 畫布尺寸

| 參數 | 計算方式 | 結果（375pt 基準） |
|------|---------|-----------------|
| 寬度 W | `screenWidth - 40 (左右各 20pt margin) - 8 (container padding 各 4pt)` | **327 pt** |
| 高度 H | `max(300, containerHeight - 8)` | 動態，依剩餘空間 |

> **注意**：W/H 直接傳入 treemap 演算法，單位為 pt（point），與 px 視裝置 DPI 比例換算。

---

## 2. 資料模型

### StockItem

```swift
// Swift
struct StockItem {
    let code: String        // 股票代號，e.g. "2330"
    let name: String        // 股票名稱，e.g. "台積電"
    let close: Double       // 收盤價（元）
    let open: Double        // 開盤價
    let high: Double        // 最高價
    let low: Double         // 最低價
    let diff: Double        // 漲跌（元）
    let pct: Double         // 漲跌幅（%），e.g. 2.35
    let vol: Double         // 成交量（張）
    let amount: Double      // 成交額（元）
    let mktcap: Double      // 市值（元）
    var pastClose: Double?  // 歷史模式：期初收盤價
}
```

```kotlin
// Kotlin
data class StockItem(
    val code: String,
    val name: String,
    val close: Double,
    val open: Double,
    val high: Double,
    val low: Double,
    val diff: Double,
    val pct: Double,
    val vol: Double,
    val amount: Double,
    val mktcap: Double,
    val pastClose: Double? = null
)
```

### TileRect（排版輸出）

```swift
struct TileRect {
    let stock: StockItem
    var x: Double
    var y: Double
    var w: Double
    var h: Double
}
```

---

## 3. API 端點

Base URL：`https://twstock-proxy-xi.vercel.app`

| 功能 | Method | Path |
|------|--------|------|
| 上市即時行情 | GET | `/api/twse?type=day` |
| 上市公司基本資料（發行股數） | GET | `/api/twse?type=company` |
| 上市歷史漲跌幅（週） | GET | `/api/twse?type=hist_1w` |
| 上市歷史漲跌幅（二週） | GET | `/api/twse?type=hist_2w` |
| 上市歷史漲跌幅（月） | GET | `/api/twse?type=hist_1m` |
| 上市歷史漲跌幅（季） | GET | `/api/twse?type=hist_3m` |
| 上市歷史漲跌幅（半年） | GET | `/api/twse?type=hist_6m` |
| 上市歷史漲跌幅（年） | GET | `/api/twse?type=hist_1y` |
| 上櫃即時行情 | GET | `/api/tpex?type=quotes` |
| 上櫃歷史漲跌幅（週） | GET | `/api/tpex?type=hist_1w` |
| _(上櫃其餘週期同上市格式)_ | | `/api/tpex?type=hist_{period}` |

Timeout：15 秒。

### 上市資料解析（TWSE）

```
漲跌幅 pct = (Change / (ClosingPrice - Change)) × 100
成交量 vol（張）= TradeVolume ÷ 1000
市值（元）= 已發行普通股數（股）× 收盤價
```

ETF 過濾條件（符合任一即排除）：
- 代號開頭為 `0`（00xxxx 系列）
- 名稱含：ETF、基金、債、指數、REITs、期信、正2、反1、多空
- 代號非純 4 碼數字

### 上櫃資料解析（TPEx）

```
市值（元）= (Capitals ÷ 10 ÷ 1000) × 收盤價
           （Capitals 為資本額（元），面值 10 元，÷1000 換算成張）
成交量 vol（張）= TradingShares ÷ 1000
```

額外過濾：代號開頭為 `9` 的排除。

---

## 4. 方塊大小計算（getSz）

用於決定每支股票在 treemap 中佔的**相對面積**。

```swift
func getSz(stock: StockItem, sizeMode: SizeMode, scale: Double = 0.65) -> Double {
    let rawValue: Double
    switch sizeMode {
    case .vol:    rawValue = max(stock.vol, 0.01)
    case .amount: rawValue = max(stock.amount, 1.0)
    case .mktcap: rawValue = max(stock.mktcap, 1.0)
    }
    return pow(rawValue, scale)
}
```

### Scale 參數說明

| scale 值 | 效果 |
|---------|------|
| 0.3 | 差異壓縮（各股方塊大小接近） |
| **0.65**（預設） | 台積電 vs 鴻海約 4 倍；vs 中小股約 96 倍 |
| 1.0 | 使用原始值（台積電極端突出） |

> 原始值若直接使用，台積電市值 ~20 兆 vs 小股 ~10 億，比值達 2 萬倍，treemap 會極端不均。使用 `pow(v, 0.65)` 壓縮到合理視覺比例。

---

## 5. Treemap 排版演算法（Squarified）

### 輸入

```
items: [{ ...StockItem, _sz: Double }]   // 已按 getSz 降序排列
rect:  { x, y, w, h }                    // 畫布矩形（pt）
```

### 輸出

```
tiles: [TileRect]   // 每個 tile 的 x, y, w, h
```

### 演算法虛擬碼

```
squarify(items, rect):
    total = sum(item._sz for item in items)
    normalize: item._area = item._sz × (rect.w × rect.h) / total
    call sqRow(items, rect, result)
    return result

sqRow(items, rect, result):
    if items is empty: return
    if items has 1 item:
        place it filling entire rect; return

    isHorizontal = (rect.w >= rect.h)
    row = []
    bestAspect = ∞

    // 貪心：逐一加入 item，直到長寬比開始變差
    for each item in items:
        row.append(item)
        aspect = worstAspect(row, rect, isHorizontal)
        if aspect <= bestAspect:
            bestAspect = aspect
        else:
            row.removeLast()     // 這個 item 不放進這 row
            break

    placeRow(row, rect, result, isHorizontal)

    // 計算剩餘空間，遞迴處理剩餘 items
    usedArea = sum(item._area for item in row)
    if isHorizontal:
        rowWidth = min(rect.w, usedArea / rect.h)
        newRect = { x: rect.x + rowWidth, y: rect.y,
                    w: rect.w - rowWidth, h: rect.h }
    else:
        rowHeight = min(rect.h, usedArea / rect.w)
        newRect = { x: rect.x, y: rect.y + rowHeight,
                    w: rect.w, h: rect.h - rowHeight }

    if remaining items exist and newRect.w > 1 and newRect.h > 1:
        sqRow(remainingItems, newRect, result)

placeRow(row, rect, result, isHorizontal):
    total = sum(item._area for item in row)
    offset = 0
    if isHorizontal:
        rowWidth = min(rect.w, total / rect.h)
        for each item in row:
            h = rowWidth > 0 ? item._area / rowWidth : 0
            result.append({ ...item, x: rect.x, y: rect.y + offset,
                             w: rowWidth, h: h })
            offset += h
    else:
        rowHeight = min(rect.h, total / rect.w)
        for each item in row:
            w = rowHeight > 0 ? item._area / rowHeight : 0
            result.append({ ...item, x: rect.x + offset, y: rect.y,
                             w: w, h: rowHeight })
            offset += w

worstAspect(row, rect, isHorizontal):
    total = sum(item._area for item in row)
    side = isHorizontal ? rect.h : rect.w
    sliceWidth = total / side   // 這個 row 的厚度
    worstRatio = 1.0
    for each item in row:
        itemSide = item._area / sliceWidth
        ratio = max(sliceWidth, itemSide) / min(sliceWidth, itemSide)
        worstRatio = max(worstRatio, ratio)
    return worstRatio
```

### 注意事項

- 傳入前需先按 `getSz` **降序排列**（最大的先排）
- 只取前 N 大（預設 N=30，可選 50/100/200/300/500）
- 輸出的 x/y/w/h 為相對於畫布左上角的**座標（pt）**

---

## 6. 顏色系統（pct2color）

### 映射規則

| 條件 | 語義 | Dark 主題色 | Light 主題色 |
|------|------|------------|------------|
| `pct >= 9.9` | 漲停 | `rgba(255,51,58,0.85)` | `rgba(255,51,58,0.65)` |
| `pct > 5` | 大漲 | `rgba(255,51,58,0.50)` | `rgba(255,51,58,0.35)` |
| `pct > 0` | 小漲 | `rgba(255,51,58,0.25)` | `rgba(255,51,58,0.10)` |
| `pct == 0` | 平盤 | `#1D2228` | `#F5F8FA` |
| `pct > -5` | 小跌 | `rgba(0,171,94,0.25)` | `rgba(0,171,94,0.10)` |
| `pct > -9.9` | 大跌 | `rgba(0,171,94,0.50)` | `rgba(0,171,94,0.35)` |
| `pct <= -9.9` | 跌停 | `rgba(0,171,94,0.85)` | `rgba(0,171,94,0.65)` |

```swift
// Swift
func pct2color(_ pct: Double, theme: Theme) -> UIColor {
    let colors = theme == .dark ? darkColors : lightColors
    if pct >= 9.9  { return colors[0] }
    if pct > 5     { return colors[1] }
    if pct > 0     { return colors[2] }
    if pct == 0    { return colors[3] }
    if pct > -5    { return colors[4] }
    if pct > -9.9  { return colors[5] }
    return colors[6]
}
```

> **台灣市場慣例**：漲 = 紅色、跌 = 綠色（與美股相反）。

---

## 7. 文字渲染規則

### 常數

| 名稱 | 值 | 說明 |
|------|----|------|
| PAD | 4 pt | 方塊左右上下各 4pt padding |
| charW | `fontSize × 0.95` | 中文全形字寬估算 |
| halfCharW | `12 × 0.62` | 最小字體下的字元寬（fit 判斷用） |
| nameLineH | `fontSize × 1.35` | 股名行高 |
| pctLineH | `pctFontSize × 1.35` | 漲跌幅行高 |
| PCT_MIN | 11 pt | 漲跌幅最小字體 |

### 字體大小（依面積插值）

```
tileArea = tileWidth × tileHeight

nameFontSize = round(12 + 4 × clamp((tileArea - 1200) / (12000 - 1200), 0, 1))
             = 12px（area ≤ 1200）～ 16px（area ≥ 12000）線性插值

pctFontSize  = max(12, nameFontSize - 1)
```

### 漲跌幅顯示判斷

```
pctStr = 漲跌幅字串，e.g. "+2.35%"（負數不加+）
pctStrWidth = pctStr.count × (PCT_MIN × 0.62)

showPct = (tileWidth >= pctStrWidth + PAD×2)   // 寬度夠放
       && (tileHeight >= nameLineH + PCT_MIN×1.35 + 2)  // 高度夠放名稱+漲跌幅
```

### 股票名稱顯示判斷

```
maxCharsPerLine = floor((tileWidth - PAD×2) / charW)

如果 showPct == true:
    → 名稱強制 1 行，超過 maxCharsPerLine 字元則截斷

如果 showPct == false:
    if name.length <= maxCharsPerLine:
        → 顯示 1 行
    else if tileHeight >= nameLineH × 2 + 2px:
        → 折 2 行（第一行 maxCharsPerLine 字，第二行其餘，共最多 maxCharsPerLine×2 字）
    else:
        → 截斷為 1 行
```

### 方塊是否顯示文字

```
showText = (tileWidth > PAD×2 + charW) && (tileHeight > nameLineH × 0.8)
```

tileWidth 或 tileHeight ≤ 6pt 的方塊完全不渲染（跳過）。

### 文字對齊

- 水平：置中（center）
- 垂直：股名 + 漲跌幅整體置中，兩者 column 排列（stock-tile 為 flex column + justify-content: center）
- 股名顏色：Dark `rgba(255,255,255,0.95)`、Light `rgba(0,0,0,0.78)`
- 漲跌幅顏色：Dark `rgba(255,255,255,0.82)`、Light `rgba(0,0,0,0.65)`

---

## 8. Tooltip 規格

### 觸發

- 點擊任一 StockTile → 顯示 Tooltip
- 點擊畫布空白處 → 隱藏 Tooltip

### 內容（即時模式）

```
[公司名稱] [代號]
今日漲跌幅    +2.35%     ← 漲紅跌綠
今日最高      150.50
今日最低      148.00
成交量        12,345 張
市值          X.X 兆
```

### 內容（歷史模式：週/月/季等）

```
[公司名稱] [代號]
近一週漲跌幅  +5.12%
期初收盤      145.00
目前收盤      152.40
今日成交量    12,345 張
市值          X.X 兆
```

### 位置計算

```
tooltipWidth  = 180 pt（min-width）
tooltipHeight ≈ 220 pt（估算）

left = tile.centerX - tooltipWidth/2
left = clamp(left, 4, canvasWidth - tooltipWidth - 4)

top = tile.bottom + 4
if top + tooltipHeight > canvasHeight + 20:
    top = tile.top - tooltipHeight - 4   // 翻到上方
```

### 外觀

| 屬性 | Dark | Light |
|------|------|-------|
| 背景 | `rgba(20,20,22,0.97)` | `#FFFFFF` |
| 邊框 | 無 | `1pt solid #E0E0E0` |
| 圓角 | 14 pt | 14 pt |
| 陰影 | `0 8px 32px rgba(0,0,0,0.85)` | `0 4px 20px rgba(0,0,0,0.12)` |
| 上下 padding | 14 pt | 14 pt |
| 左右 padding | 16 pt | 16 pt |

漲跌幅顏色：
- 漲 → Dark: `#FF6B6B`、Light: `#E53935`
- 跌 → Dark: `#4CD97B`、Light: `#2E7D32`

---

## 9. 圖例（Legend）

固定顯示於畫布下方，7 色色條 + 標籤。

```
[漲停] [>5%] [0-5%] [0%] [0-5%] [<-5%] [跌停]
```

- 色條高度：5 pt，各色塊等寬，圓角 2 pt，間距 3 pt
- 標籤字體：9 pt，顏色 Dark: `#666`、Light: `#8E8E93`
- 標籤 margin-top：4 pt，各標籤置中對齊對應色塊

---

## 10. 互動行為

### 控制項

| 元件 | 類型 | 選項 | 預設 |
|------|------|------|------|
| 市場 | Tab（上下互斥） | 上市 / 上櫃 | 上市 |
| 面積基準 | SegmentedControl | 市值 / 成交量 / 成交額 | 市值 |
| 時間範圍 | ScrollableButtonBar | 即時/週/二週/月/季/半年/年 | 即時 |
| 顯示數量 | ButtonGroup | 30/50/100/200/300/500 | 30 |

### 資料快取策略

- 同一市場的即時資料：**session 內快取**，切換市場才重新呼叫 API
- 歷史資料：**以 `{market}_{period}` 為 key** 快取，同 key 不重複請求
- 切換面積基準（市值/成交量/成交額）：**不重新請求 API**，直接以快取資料重新排版
- 歷史模式下切換面積基準：同上，以 merge 後的 `currentData` 重新排版

### 歷史模式資料 Merge

```
histData = [{ code, pct, pastClose }]   // 從歷史 API 取得
merged = baseData.map(stock => {
    hist = histData.find(code == stock.code)
    return { ...stock, pct: hist?.pct ?? null, pastClose: hist?.pastClose ?? null }
})
```

`pct` 為 null 的股票仍渲染，顏色對應平盤色（因 `isNaN` → `COLORS[3]`）。

### 點擊方塊

- 顯示 Tooltip（內容依當前時間模式切換，見第 8 節）
- 點擊其他方塊→切換 Tooltip 目標
- 點擊畫布空白處→關閉 Tooltip

---

## 11. 主題（Dark / Light）

| 屬性 | Dark | Light |
|------|------|-------|
| 畫面背景 | `#000000` | `#FFFFFF` |
| 畫布背景 | `#000000` | `#FFFFFF` |
| 平盤色 | `#1D2228` | `#F5F8FA` |
| 標題文字 | `#FFFFFF` | `#1C1C1E` |
| Tab 未選中 | `#666` | `#AAAAAA` |
| Tab 選中 | `#FFFFFF` | `#1C1C1E` |
| SegmentedControl 背景 | `#2C2C2E` | `#E5E5EA` |
| SegmentedControl 選中 | `#636366` + 白字 | `#FFFFFF` + 深字 + shadow |
| TimeBar 背景 | `#2C2C2E` | `#E5E5EA` |
| Tab active 底線顏色 | `#7C82FF` | `#7C82FF` |
| Loading spinner 顏色 | `#7C82FF` | `#7C82FF` |

---

## 12. 元件狀態

### HeatmapCanvas 狀態機

```
idle
  ↓ 進入畫面 / 切換市場
loading  → 顯示 spinner + 文字「載入台股資料中...」
  ↓ 成功
rendered → 顯示 treemap tiles
  ↓ 失敗
error    → 顯示錯誤訊息 + 可能原因說明

rendered ─(切換時間範圍)→ histLoading → rendered / error
rendered ─(切換面積基準)→ rendered（純 re-render，無網路請求）
```

### 錯誤訊息內容

```
⚠️ 資料載入失敗
[錯誤訊息]
可能是非交易時段或 API 暫時無法存取
請重新整理頁面
```

---

## 13. 實作建議（iOS / Android）

### 渲染方式

**推薦：Canvas 繪製**（效能優於大量 subview）

- iOS：`UIView` 搭配 `draw(_ rect:)` / `CALayer`，或 SwiftUI `Canvas`
- Android：繼承 `View` 覆寫 `onDraw(canvas: Canvas)`

Treemap tile 數量最多 500 個，用原生 view 堆疊效能差；Canvas 繪製可流暢支援。

### Squarified Treemap 函式庫

| 平台 | 建議 |
|------|------|
| iOS (Swift) | 自行實作（演算法見第 5 節，約 60 行）或使用 [SwiftTreemap](https://github.com/niceyeti/Treemap) |
| Android (Kotlin) | 自行實作或參考 [Weave](https://github.com/nicktindall/treemap) 的 Java 版移植 |

演算法本身不依賴 UI 框架，可直接移植虛擬碼（約 80 行 Kotlin/Swift）。

### 文字繪製注意事項

1. 字體大小以面積動態計算（見第 7 節），建議在排版後、繪製前批次計算
2. 中文字寬估算：`fontSize × 0.95`（全形）；英數：`fontSize × 0.62`
3. 超過 2 行不折行，直接截斷（不加省略號）
4. 文字用 `NSAttributedString` (iOS) / `StaticLayout` (Android) 測量實際寬度可提高精確度

### Tooltip

- iOS：`UIView` 加到 heatmap superview，z-order 置頂
- Android：`PopupWindow` 或在 `onDraw` 上層再疊一個 `View`

### 效能優化

- 只在資料或尺寸變化時重新執行 squarify（O(n log n) 複雜度）
- 切換面積基準：只重新執行 squarify + redraw，不呼叫 API
- 切換主題：只更換顏色並 redraw，不重排版
- 畫布尺寸改變（旋轉/resize）：需重新 squarify

### 數字格式化

```swift
// 大數值格式（成交量/市值顯示）
func fmtBig(_ n: Double) -> String {
    if n >= 1e8 { return String(format: "%.1f億", n / 1e8) }
    if n >= 1e4 { return "\(Int(n / 1e4))萬" }
    return "\(Int(n.rounded()))"
}

// 一般數值（價格）
func fmtPrice(_ n: Double) -> String {
    return String(format: "%.2f", n)
}
```

---

*規格取自 `Jane-idesign/twstock-proxy` 原始碼，由 Claude Code 分析轉換。*
