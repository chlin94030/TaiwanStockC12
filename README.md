# Alpha Radar V16.6 — 1 分鐘動態盤中版

V16.6 的目標不是再改短／中／長線選股核心，而是把盤中執行層改成真正的「全市場發現 → 動態精算 → 可執行買點」。完整日線模型仍使用正式收盤資料，盤中每 60 秒只刷新即時價量，不把未完成日 K 寫回多年模型。

## 核心流程

### 盤後／盤前
1. 下載／增量更新完整日線快取。
2. 依流動性、歷史特徵、相似案例、基本面與法人資料計算最多 1000 檔完整模型。
3. 產生短線／中線／長線與下一交易日關注清單。
4. 同時保存 `intraday_universe`：所有通過最低價格與成交額條件的流動性股票，只含盤中掃描所需的輕量 metadata，不保存重型 DataFrame。

### 盤中每 60 秒
1. 對 `intraday_universe` 抓一次即時報價（預設最多 1800 檔）。
2. 用相對大盤、量能速度、日內位置、開盤後強弱、買賣壓力、成交金額等做輕量 discovery ranking。
3. 與昨晚短／中／長各 Top 80 的 daily core 合併。
4. 去重後最多 300 檔進入精算層。
5. 精算 Entry Score、ATR、剩餘預期報酬、Reward/Risk、歷史案例 confidence、買入區與部位。
6. 最後只把真正通過門檻者列為可買；若 0 檔，不硬湊推薦。

## 兩條合法進場路徑

### PULLBACK — 拉回承接
價格位於結構買入區，Selection / Entry / Expected Return / Positive Rate / Reward-Risk / Evidence 全部通過時，可顯示「拉回買點佳，可分批進場」或「拉回承接，可小量試單」。

### CONTINUATION — 強勢續攻
若價格已高於昨晚買入區，不再一律判成「等拉回」。必須同時符合更嚴格條件，例如：
- 標的分數與盤中強度足夠；
- 相對大盤明顯強；
- 量比在健康區間；
- 日內位置偏強；
- 距突破區不超過約 1.2 ATR；
- 單日漲幅未進入追高區；
- 保守剩餘報酬、Reward/Risk、歷史案例信心仍足夠。

通過時顯示「續攻確認，可小量試單」，且部位上限比一般拉回買點更低。近漲停、追價風險高、過度延伸的標的仍會被拒絕。

## 盤中診斷

每一檔不符合買進條件時，現在會保留 `failed_gates`，例如：
- 進場分數未達門檻
- 保守剩餘報酬不足
- 報酬／風險不足
- 歷史案例信心或樣本不足
- 續攻量能不在健康區間
- 相對大盤強度不足
- 距突破區過遠
- 追價風險高

頁面上方會統計精算池的主要淘汰原因，不再讓所有 WAIT 狀態都顯示同一句「等拉回」。

## 更新頻率

- Streamlit fragment：`60` 秒。
- 即時 quote cache TTL：`52` 秒。
- 每分鐘刷新即時層；多年日線／基本面／歷史案例不重算。
- 外部行情 provider 若無法一次完成全市場批次，會自動退回 daily core，不讓盤中頁整體失效。

## 主要檔案

- `app.py`：UI、60 秒 fragment、全市場／精算池統計。
- `radar_service.py`：完整日線模型與 `intraday_universe`。
- `intraday_engine.py`：全市場 quote、輕量 discovery、動態候選池、盤中 rerank。
- `recommendation_engine.py`：PULLBACK / CONTINUATION、Reward/Risk、failed gates。
- `strategy_config.py`：所有盤中與進場門檻集中管理。
- `return_first_model.py`：歷史報酬／相似案例核心，本版不改底層特徵架構。
- `test_architecture.py`：離線 regression tests。
- `benchmark_v16_6.py`：1500 檔盤中 discovery CPU benchmark。

## 部署

建議完整覆蓋目前 repository 中同名檔案後重新部署，不要只換 `app.py`。V16.6 改動了 `app.py`、`radar_service.py`、`intraday_engine.py`、`recommendation_engine.py`、`strategy_config.py` 等協作介面。

第一次部署後請手動按一次完整更新，讓新的 snapshot 建立 `intraday_universe`。舊 snapshot 即使沒有此欄位也有 fallback，但完整功能以重建 V16.6 snapshot 為準。

## 測試限制

本容器沒有安裝 Streamlit，也沒有用實際台股即時網路行情做 end-to-end provider 測試；因此 App 渲染採 Streamlit stub smoke test，資料來源以 synthetic/FakeStore integration 測試。真正 MIS／FinMind 每分鐘批次速度仍會受 Streamlit Cloud 與外部行情服務回應時間影響。

## V16.6.1 Research Detail Hotfix

本版只修正「顯示營收 / EPS / 法人明細」的資料載入與 UI 容錯，不更動短／中／長線選股權重、盤中 1 分鐘動態候選池、Continuation Entry、Pullback Entry 或任何排名邏輯。

### 根因

V16.6 為了控制完整更新時間，只會預先補齊有限數量的 research pool；盤中動態候選池則可能包含未在該 research pool 內的股票。舊 UI 的「顯示營收 / EPS / 法人明細」只讀 snapshot 內既有的 `stock["research"]`，按下開關後不會再為該股票即時補抓資料，因此動態候選股可能展開後完全沒有內容。此外，舊 toggle key 只使用 ticker，未來同一檔股票若在同一頁多處出現，有 Streamlit widget key 衝突風險。

### 修正

- 新增 `radar_service.research_on_demand()`：單股明細按需載入，不重跑市場模型、不改排名。
- 展開明細後會重新取得該股的月營收、財報 / EPS、估值與法人資料，再與 snapshot 中已有資料安全合併。
- 成功結果於同一 Streamlit session 快取 15 分鐘；資料源失敗只快取 60 秒，避免暫時性 API 錯誤被鎖太久。
- 月營收、EPS、法人三區各自容錯：某一區資料源失敗，不會讓整個 expander 報錯或空白。
- 只有交易所最新備援值時，仍會以表格顯示最新月營收 / 最新累積 EPS，不再因沒有完整歷史序列而什麼都不顯示。
- 法人若只有彙總值、沒有日別 rows，仍顯示外資 / 投信 / 自營商 / 合計；若完全無資料則顯示明確訊息。
- Widget key 改成「頁面範圍 + rank + snapshot + ticker」，避免同一股票多處出現時衝突。
- V16.6.1 是 display/data-loading hotfix；底層 `ARCHITECTURE_VERSION` 保持 `v16.6.0-dynamic-intraday`，因此不需要刪除既有 dashboard snapshot 或重新建立策略模型。

## V16.6.2 Research Data Fix

本版針對「顯示營收 / EPS / 法人明細」再做資料語意與官方備援修正，**不更動選股、短中長線權重、盤中 dynamic discovery、Pullback / Continuation Entry 或任何排名公式**。

### 修正重點

- 月營收官方 OpenAPI 只提供最新一期時，明細頁會按需回補 MOPS 歷史月營收，最多顯示近 12 個月；回補失敗仍保留最新一期，不讓卡片報錯。
- 月份與季度在 Plotly 一律使用 `category` 軸；只有 1 個資料點時不畫趨勢圖，避免 `2026-08` 被 Plotly 當成日期後出現微秒刻度。
- 官方損益表 Q2/Q3/Q4 是年初至當季累計資料。營收與毛利可差分還原單季毛利率；**EPS 不以相減方式偽造單季 EPS**，因加權平均股數可能因配股／分割改變。官方備援時明確顯示「官方累積 EPS」。FinMind 有單季 EPS 時才顯示單季 EPS。
- FinMind 法人資料不可用時，上市股票明細改以 TWSE T86 取得最近數個交易日；上櫃股票以 TPEx `tpex_3insti_daily_trading` 取得官方最新交易日。資料期間會直接標在畫面，不再硬寫成「近月」。
- 詳細資料 toggle 已開啟時，先載入 research detail 再繪製上方摘要，因此外資／投信／自營商不會在下方已有明細時，上方仍停留在 `—`。
- 營收、財報、法人各區分別標示 source 與 coverage；某一來源失敗時，其餘區塊仍可顯示。
- 新增 `lxml` 依賴供 MOPS HTML 表格解析。

### 資料來源原則

- FinMind：有完整歷史資料時優先使用。
- TWSE / TPEx OpenAPI：最新一期官方備援。
- MOPS 歷史月營收與季度綜合損益表：僅於使用者展開明細時按需回補並寫入 SQLite research cache。
- TWSE T86 / TPEx 三大法人 OpenAPI：FinMind 法人資料失敗時的 display-only fallback。

上述 on-demand 回補只更新畫面與 research cache，不回寫既有 snapshot 排名，因此不會因使用者展開資料而改變短／中／長線或盤中推薦。
