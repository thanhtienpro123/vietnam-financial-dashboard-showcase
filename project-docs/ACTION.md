# ACTION Plan

## Phase 0 - Foundation

Mục tiêu: tạo khung app, schema dữ liệu và quy ước engineering.

- Tạo repo structure: `backend/`, `frontend/`, `data/`, `notebooks/`, `infra/`, `docs/`.
- Backend FastAPI với health check, config, logging JSON, error handling.
- PostgreSQL/TimescaleDB docker compose.
- Alembic migrations cho bảng raw source, market bars, macro indicators, bank rates, news, model runs.
- CI local: lint, format, type check, unit test.
- File `.env.example` cho API keys và DB URL.

Deliverable:

- App chạy local.
- `/health` trả về OK.
- Migration đầu tiên chạy thành công.

## Phase 1 - Data ingestion MVP

Mục tiêu: có dữ liệu dùng được cho dashboard và notebook.

- Implement connector `vnstock` cho OHLCV, index, company, financial statements.
- Implement connector World Bank/IMF cho macro Việt Nam.
- Implement connector oil/global market qua yfinance/FRED/EIA tuỳ API key.
- Tạo bank-rate ingestion với interface chung:
  - manual CSV upload first;
  - scraper/plugin riêng cho từng ngân hàng sau.
- Implement raw data storage: lưu response gốc và checksum.
- Chuẩn hoá trading calendar, timezone Asia/Ho_Chi_Minh.

Deliverable:

- Lấy được sample: VNINDEX, 5 ticker lớn, GDP/CPI/inflation, WTI/Brent proxy, 3 bank rates từ CSV.
- Data quality report: missing, duplicate, stale, outlier.

## Phase 2 - Dashboard MVP

Mục tiêu: người dùng xem được thị trường và portfolio cơ bản.

- Next.js dashboard shell: sidebar, top market bar, watchlist.
- Market overview: index cards, candlestick, volume, sector heatmap.
- Macro page: chart macro overlay với VNINDEX.
- Portfolio input: ticker, shares/amount, budget, weights.
- Portfolio output: return, volatility, Sharpe, drawdown, correlation matrix.

Deliverable:

- Web dashboard dùng được với data local.
- API endpoints cho market/macro/portfolio.

## Phase 3 - Financial statements và FCFF

Mục tiêu: phân tích intrinsic value có audit trail.

- Ingest financial statements theo ticker/quarter/year.
- Normalize financial statement fields.
- FCFF engine:
  - historical FCFF;
  - forecast assumptions;
  - WACC;
  - terminal value;
  - sensitivity table.
- UI assumption editor:
  - base/bull/bear;
  - sliders/input;
  - save scenario.
- Explanation engine: hiện lý do default assumption dựa trên historical metrics.

Deliverable:

- DCF/FCFF cho 1 ticker mẫu.
- Export scenario JSON.
- Sensitivity WACC x terminal growth.

## Phase 4 - Forecast lab

Mục tiêu: so sánh model dự báo có backtest.

- Feature builder:
  - lag returns;
  - rolling mean/volatility;
  - volume shock;
  - macro variables;
  - news sentiment nếu có.
- Baseline models: naive, moving average.
- Statistical: ARIMA/SARIMAX/AutoARIMA.
- ML: XGBoost.
- DL: LSTM, TimesNet.
- Walk-forward backtest.
- Model registry table.
- API forecast result with confidence interval/quantiles.

Deliverable:

- Leaderboard model theo ticker/index.
- Forecast chart với uncertainty band.
- Metrics và model version được lưu.

## Phase 5 - Advanced risk

Mục tiêu: trả lời "có thể mất bao nhiêu tiền, xác suất bao nhiêu".

- Distribution analysis từng ticker và portfolio.
- VaR/CVaR:
  - historical;
  - parametric normal/t;
  - Monte Carlo.
- ARCH/GARCH volatility forecast.
- Copula simulation cho joint downside.
- Stress scenarios:
  - VNINDEX -5%/-10%;
  - banking sector shock;
  - oil +10%/+20%;
  - VND depreciation;
  - interest rate +100 bps.
- Risk contribution by ticker/sector.

Deliverable:

- Risk dashboard: VaR, CVaR, worst-case, probability of loss > threshold.
- Scenario table theo budget VND.

## Phase 6 - News và map

Mục tiêu: tin tức cập nhật, có liên kết với ticker/ngành/địa điểm.

- RSS/API connectors.
- Dedup by URL/content hash.
- Vietnamese text cleaning, keyword extraction, ticker/entity mapping.
- Sentiment/topic classifier.
- Geocoding locations.
- News map với marker severity.
- Link news to abnormal return/volume.

Deliverable:

- News page.
- Map page.
- Alerts cho tin có severity cao.

## Phase 7 - Production hardening

Mục tiêu: hệ thống chạy ổn định, có quan sát và phục hồi.

- Auth/user workspace.
- Role permissions.
- Rate limiting.
- Retry/backoff/circuit breaker cho connectors.
- Cache policy.
- Backup/restore DB.
- Monitoring và alerting theo `MONITORING.md`.
- Docker compose production-like.
- Deployment guide.

Deliverable:

- Release candidate có monitoring, backup, runbook.

## Thứ tự ưu tiên 2 tuần đầu

1. FastAPI + DB schema + Docker compose.
2. Ingest `vnstock` OHLCV/index/financial statements.
3. Ingest macro World Bank/IMF.
4. Portfolio basic metrics.
5. FCFF engine basic.
6. Next.js market + portfolio pages.

## Definition of Done

- Mỗi connector có unit test với sample fixture.
- Mỗi bảng normalized có source metadata.
- Mỗi model/risk result có version, input window, generated_at.
- Dashboard không hiện kết quả nếu data stale quá ngưỡng.
- Mỗi output đầu tư/rủi ro có disclaimer và assumption visible.
