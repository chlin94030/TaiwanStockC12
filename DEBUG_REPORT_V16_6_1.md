# Alpha Radar V16.6.1 — Research Detail Hotfix Debug Report

## 使用者回報

「顯示營收 / EPS / 法人明細」展開後發生錯誤或沒有內容。

## 根因

V16.6 的完整掃描只對受控 research pool 預先載入公司 / 法人資料；盤中動態候選池可擴大到其他已完成日線分析的股票。舊 UI 展開明細時僅讀 snapshot 既有 `research`，沒有 on-demand fetch，因此動態候選股可能沒有明細資料。舊 toggle key 也只有 ticker，缺乏 render scope，存在 duplicate widget key 風險。

## 修正內容

1. `radar_service.py`
   - 新增 `research_on_demand(ticker, data_dir, include_branch=False)`。
   - 只取單股 research data，不改寫 selection / ranking / entry scores。
   - 任何 provider exception 轉為 fail-open payload，不讓 UI crash。

2. `app.py`
   - 展開明細時按需補抓 research data。
   - 與 snapshot 既有資料 merge，已有好資料不會被失敗回應覆蓋。
   - 成功 detail session cache 15 分鐘；失敗只 cache 60 秒。
   - 月營收、EPS / 毛利率、三大法人改成獨立 defensive renderer。
   - 完整歷史不足時仍顯示最新官方備援值。
   - 圖表失敗只降級為表格 / 訊息，不中斷整個股票卡片。
   - toggle / chart key 加入 render scope，避免 duplicate key。

## 回歸測試

- `python -m compileall -q .` — PASS
- `python test_architecture.py` — PASS（23 項 + benchmark）
- `python test_research_details.py` — PASS
  - snapshot research 為空時，toggle 可觸發 on-demand fetch
  - 營收表格 / 圖、EPS 表格 / 圖、法人日別表可正常渲染
  - 同一 ticker 不同 render scope 的 widget key 保持唯一
  - provider failure 不會使 app crash
- `python smoke_app_import.py` — PASS
- `python smoke_decision_views.py` — PASS
- `python smoke_app_main.py` — PASS

## 模型影響

無。V16.6.1 不更動：
- selection weights
- historical analog / empirical confidence
- fundamental / flow scoring
- 1-minute dynamic intraday discovery
- Pullback / Continuation Entry
- entry score / Reward-Risk gates
- Bull / Neutral / Bear position policy

因此部署本 hotfix 不需要重跑完整模型；舊 V16.6 snapshot 可以直接沿用。
