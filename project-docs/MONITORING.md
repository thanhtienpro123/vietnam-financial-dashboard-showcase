# Monitoring Plan

## Mục tiêu

Hệ thống phải biết khi nào dữ liệu sai, chậm, thiếu, model drift, job lỗi, hoặc dashboard đang hiện kết quả stale.

## Loại monitoring

### 1. System health

Metrics:

- API latency p50/p95/p99.
- API error rate 4xx/5xx.
- DB connection pool usage.
- Redis memory/evictions.
- Worker queue length.
- Job runtime và failure rate.

Tools:

- Prometheus metrics endpoint.
- Grafana dashboards.
- Sentry cho exception.
- Structured JSON logs.

### 2. Data freshness

Theo dõi `last_success_at` cho từng source:

- Market OHLCV: sau giờ giao dịch, cần cập nhật trong ngày.
- Intraday/realtime quote: tuỳ gói dữ liệu, cần stale threshold tính bằng phút.
- Macro: tháng/quý, threshold tính bằng ngày.
- Bank rates: ngày/tuần, threshold theo source.
- News: threshold tính bằng phút/giờ.

Alert examples:

- `market_data_stale`: VNINDEX/stock bars chưa cập nhật sau 18:00 Asia/Ho_Chi_Minh.
- `news_feed_stale`: không có tin mới trong 2 giờ với source active.
- `bank_rate_stale`: lãi suất ngân hàng quá 7 ngày chưa refresh.

### 3. Data quality

Checks:

- Duplicate key: symbol + date + interval.
- Missing OHLCV.
- Negative price/volume.
- Return outlier vượt threshold, cần flag chứ không auto delete.
- Corporate action gap: split/dividend làm giá nhảy bất thường.
- Macro revision: cùng period nhưng value thay đổi, cần lưu version.
- Financial statement unit mismatch: VND, thousand VND, million VND, billion VND.

Suggested library:

- Great Expectations hoặc Pandera cho validation.

### 4. Model monitoring

Theo dõi:

- Forecast error theo rolling window: MAE, RMSE, directional accuracy.
- Drift của feature: PSI/KS test cho returns, volume, volatility, macro features.
- Prediction distribution drift.
- Model age: ngày từ lần train cuối.
- Backtest degradation: model mới phải vượt baseline trước khi promote.

Alert:

- Forecast MAE 20 ngày gần nhất cao hơn baseline 25%.
- Directional accuracy thấp hơn 50% trong 30 ngày.
- Feature drift PSI > 0.25.

### 5. Risk monitoring

Theo dõi:

- VaR breach count: số ngày loss vượt VaR.
- Kupiec-style backtest cho VaR nếu có đủ mẫu.
- GARCH convergence failure.
- Copula fit failure hoặc unstable correlation.
- Portfolio concentration: ticker/sector vượt constraint.
- Liquidity risk: position > X% ADV.

Alert:

- 99% daily VaR breach hơn 2 lần trong 60 trading days.
- GARCH job fail 3 lần liên tiếp.
- Portfolio loss vượt threshold người dùng đặt.

## Bảng metadata cần có

### `ingestion_runs`

```text
id
source_name
source_type
status
started_at
finished_at
records_read
records_written
records_rejected
checksum
error_message
```

### `data_quality_checks`

```text
id
run_id
table_name
check_name
status
severity
observed_value
threshold
sample_records
created_at
```

### `model_runs`

```text
id
model_name
model_family
target
symbols
train_start
train_end
validation_start
validation_end
params_json
metrics_json
artifact_uri
status
created_at
```

### `risk_runs`

```text
id
portfolio_id
horizon_days
confidence_level
method
var_value
cvar_value
worst_loss_value
prob_loss_over_budget_pct
params_json
created_at
```

## Logging convention

Mỗi log event nên có:

```text
timestamp
level
service
job_id
source
symbol
event_name
message
duration_ms
records
error_type
```

## Dashboard monitoring view

Nên có một trang admin:

- Source status: green/yellow/red.
- Last successful ingestion.
- Failed jobs.
- Data quality issues.
- Model registry và drift status.
- VaR breach status.
- Manual re-run button cho job.

## Runbook ngắn

### Connector lỗi

1. Kiểm tra status code/error message.
2. Kiểm tra source thay đổi schema/API.
3. Chạy lại với sample symbol.
4. Nếu source lỗi, dùng fallback source nếu có.
5. Ghi incident note nếu data gap ảnh hưởng dashboard.

### Data stale

1. Xác nhận lịch nghỉ lễ/giao dịch.
2. Kiểm tra worker queue.
3. Kiểm tra API key/rate limit.
4. Chạy manual backfill.
5. Dashboard phải hiện badge stale thay vì hiện như data mới.

### Model drift

1. So sánh với baseline.
2. Kiểm tra feature drift và data outliers.
3. Retrain với window mới.
4. Chỉ promote nếu validation/backtest đạt ngưỡng.

### VaR breach

1. Ghi nhận breach theo portfolio và horizon.
2. Kiểm tra data return có chính xác không.
3. Chạy stress test cập nhật.
4. Hiện cảnh báo cho user: loss thực tế vượt expected risk model.
