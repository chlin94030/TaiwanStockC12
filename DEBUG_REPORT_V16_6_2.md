# Alpha Radar V16.6.2 — Research Data Fix Debug Report

## 問題重現

洋基工程（6691）在 V16.6.1 展開「顯示營收 / EPS / 法人明細」後可見：

1. 月營收只有 2026-08 一列，Plotly 卻把 `2026-08` 當 datetime，單點圖被自動展開成 Jul 31 / Aug 1 的微秒刻度。
2. EPS 僅顯示 2026Q2 `17.46`，但沒有清楚區分「單季 EPS」與「上半年累積 EPS」。
3. 外資 / 投信 / 自營商 / 合計仍為 `—`。
4. V16.6.1 lazy-load 在卡片底部才執行；即使下方 detail 已抓到新資料，上方 summary 在同一 rerun 仍可能先用舊 snapshot 畫完。

## 根因

- TWSE `t187ap05_L` / TPEx `mopsfin_t187ap05_O` 官方 OpenAPI 是 latest snapshot，不能直接提供近 12 個月歷史。
- `t187ap06` income-statement OpenAPI 同樣是最新期間，Q2/Q3/Q4 為年初至當季累計數。
- 舊 fallback 沒有 TWSE / TPEx official per-stock institutional path，所以 FinMind 失敗時法人區只能顯示空值。
- Plotly 對單一 `YYYY-MM` 字串會推斷為日期軸；未指定 category 時會產生不合理的細時間刻度。
- EPS 不能像營收／毛利那樣直接用累計值差分：EPS 分母是加權平均股數，配股／分割可能使不同季的股數基準不同。

## V16.6.2 修正

### 1. 月營收歷史
- 保留 FinMind 優先。
- FinMind history 不足時，display-only path 依最新官方月份，按需讀取 MOPS `t21sc03_{roc_year}_{month}_{0|1}.html`。
- Big5 / CP950 / UTF-8 防禦式解碼。
- 以欄名而非固定欄位位置解析公司代號、當月營收、MoM、YoY。
- SQLite research cache 避免重複抓取。

### 2. 財報 / EPS 語意
- MOPS `ajax_t163sb04` 依市場、年度、季度按需回補。
- 累計營收與累計毛利可差分還原單季毛利率。
- 官方累計 EPS 保留原值，不做 Q2-Q1 的錯誤單季 EPS 推算。
- 畫面在 official fallback 明確標示「官方累積 EPS」；只有 FinMind 提供單季 EPS 時才顯示「單季 EPS」。

### 3. 三大法人
- 上市：FinMind 無資料時使用 TWSE T86，按欄名解析，單位由股轉張。
- 上櫃：使用 TPEx `tpex_3insti_daily_trading` official OpenAPI 最新日資料。
- scope label 顯示實際取得幾個交易日，不再一律稱「近月」。

### 4. UI / Plotly
- 月份、季度 x-axis 強制 `type="category"`。
- 僅 1 點時不畫趨勢圖，只顯示數值表與「僅最新一期」說明。
- toggle ON 時先 merge on-demand detail 再繪製摘要，修正上方法人仍為 dash 的時序問題。
- 每區顯示資料來源與 coverage。

## 測試

- `python -m compileall -q .` → PASS
- V16.6 architecture regression 23 tests → PASS
- V16.6.1 research-detail regression → PASS
- V16.6.2 MOPS monthly parser → PASS
- V16.6.2 MOPS quarterly parser → PASS
- V16.6.2 official cumulative EPS semantics → PASS
- V16.6.2 TWSE T86 parser with reordered fields → PASS
- V16.6.2 TPEx institutional parser → PASS
- V16.6.2 single-point chart suppression → PASS
- V16.6.2 category-axis regression → PASS
- V16.6.2 detail preload / top-summary refresh → PASS
- App import / decision views / main smoke → PASS

## 重要限制

本容器 DNS 不可用，無法直接對 TWSE / TPEx / MOPS 做 live HTTP end-to-end；實際連線測試以 deterministic mocked payload / HTML 做 parser regression。所有新官方來源均 fail-open：資料源失敗只影響該 research detail，不會讓推薦頁或既有 ranking 失效。
