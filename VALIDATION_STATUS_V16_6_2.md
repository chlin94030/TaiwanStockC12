# V16.6.2 Validation Status

V16.6.2 是 research display/data-source hotfix，策略核心仍為 `v16.6.0-dynamic-intraday`。

已驗證：編譯、核心架構 regression、動態盤中候選、Continuation/Pullback、on-demand research、MOPS 歷史月營收 parser、MOPS 季財報 parser、官方累積 EPS 語意、TWSE T86、TPEx 法人、單點圖抑制、category 軸、同一 rerun 摘要更新及 Streamlit stub smoke。

未在本容器實測：TWSE / TPEx / MOPS live network，因環境 DNS 不可用。程式保留 timeout、SQLite cache 與 fail-open fallback。V16.6.2 不改 selection weights，因此不需要重跑 2026 walk-forward 才能部署此 hotfix；後續績效驗效仍沿用 V16.5/V16.6 protocol。
