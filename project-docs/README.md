# Vietnam Financial Market Analytics Dashboard

Dashboard tài chính Việt Nam chạy local, gồm backend FastAPI và frontend React/Vite. Ứng dụng tập trung vào thị trường chứng khoán Việt Nam: vĩ mô, bảng giá, báo cáo tài chính, định giá, CAPM, PTKT, tối ưu danh mục, dự báo, rủi ro và nhật ký giao dịch.

## Tài Liệu Workflow Đầu Tư

Tài liệu hướng dẫn chi tiết nằm ở:

- [docs/dashboard_workflow_vi.md](docs/dashboard_workflow_vi.md): giải thích quy trình dùng dashboard để ra quyết định đầu tư, từ chu kỳ ngành, định giá, CAPM, PTKT, Markowitz, dự báo, rủi ro tới nhật ký giao dịch.
- [docs/software_stack_vi.md](docs/software_stack_vi.md): giải thích kiến trúc phần mềm, frontend, backend, nguồn dữ liệu, luồng API, cache/model local và các file nên mở khi muốn sửa hệ thống.
- [docs/forecast_models_vi.md](docs/forecast_models_vi.md): giải thích thư viện mô hình dự báo, công thức/cách hiểu từng model, dữ liệu train/test, chỉ số đánh giá, epoch DL và nơi chỉnh tham số.

Workflow chính:

```text
Chu kỳ ngành Fourier -> Định giá -> CAPM -> PTKT -> Markowitz -> Dự báo -> Rủi ro -> Ra quyết định -> Nhật ký giao dịch
```

## Cấu Trúc Dự Án

- `backend/app`: FastAPI backend.
- `frontend/src`: React/Vite frontend.
- `frontend/src/main.tsx`: màn hình chính của dashboard.
- `frontend/src/lib/api.ts`: API client.
- `frontend/src/styles.css`: CSS giao diện.
- `frontend/public/vietnam-provinces.topojson`: dữ liệu bản đồ hành chính Việt Nam.
- `scripts/dev.ps1`: chạy dashboard local.
- `scripts/public.ps1`: chạy dashboard public qua tunnel.
- `deploy/`: bộ deploy production lên VPS/domain.

## Cách Chạy Local

Chạy bằng nút có sẵn trên Windows:

```powershell
.\Start Local Dashboard.cmd
```

Hoặc chạy script:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\scripts\dev.ps1
```

URL local:

- Frontend: `http://127.0.0.1:5173/`
- Backend API: `http://127.0.0.1:8001/api/v1`
- Health check: `http://127.0.0.1:8001/api/v1/health`

Nếu UI còn cache cũ, nhấn:

```text
Ctrl + F5
```

## Cách Chạy Public Tạm

```powershell
.\Public Dashboard.cmd
```

Script sẽ build frontend, chạy backend và mở Cloudflare tunnel. Copy link `https://*.trycloudflare.com` trong cửa sổ tunnel để chia sẻ dashboard public.

Lưu ý:

- Link `trycloudflare.com` là link tạm.
- Phải giữ cửa sổ `Public Dashboard.cmd` đang mở.
- Nếu tắt cửa sổ public, link sẽ ngừng hoạt động.
- Ai có link đều có thể mở dashboard; nếu cần kiểm soát người xem, nên deploy lên VPS/domain và thêm đăng nhập.

## Deploy Link Vĩnh Viễn

Nếu muốn có một link cố định để người được phép có thể dùng dashboard, dùng bộ production riêng trong:

- [deploy/README_DEPLOY.md](deploy/README_DEPLOY.md): hướng dẫn đưa dashboard lên VPS/domain.
- [deploy/docker-compose.prod.yml](deploy/docker-compose.prod.yml): chạy backend + web production bằng Docker Compose.
- [deploy/Caddyfile](deploy/Caddyfile): phục vụ frontend và proxy `/api/v1` vào backend nội bộ.

Bộ này không thay đổi cách chạy local hoặc public tạm hiện tại.

## Các Module Chính

### Vĩ Mô Việt Nam

Theo dõi dữ liệu vĩ mô, dòng tiền nước ngoài, lãi suất, tỷ giá, xuất nhập khẩu và histogram giá/dao động cổ phiếu theo ngành.

### Bản Đồ Tin Tức

Hiển thị tin tức kinh tế, chính sách và thị trường theo khu vực hoặc theo mã cổ phiếu. Bản đồ có lớp hành chính Việt Nam và lớp đảo/quần đảo gồm Phú Quốc, Hoàng Sa, Trường Sa. Tên đảo/quần đảo chỉ hiện khi rê chuột để tránh che bản đồ.

### Báo Cáo Tài Chính

Lấy báo cáo tài chính năm từ `vnstock` khi nguồn trả dữ liệu hợp lệ. Các giá trị giữ theo quy ước dấu của nguồn gốc, nên một dòng chi phí có số dương có thể là hoàn nhập/ghi giảm chi phí hoặc quy ước trình bày của nguồn.

### Thư Viện Mô Hình Định Giá

Bao gồm các nhóm định giá như DCF, relative valuation, P/E, P/B, P/S, EV/EBITDA, excess return, EVA, Altman Z-score. Phần `Edit inputs` cho phép sửa input numeric trước khi tính lại.

### CAPM Và Alpha

Tính beta, alpha, expected return, tracking error và các chuỗi scatter để xem cổ phiếu phản ứng với benchmark như thế nào.

### PTKT Tốt Theo Ngành

Scan các mã trong ngành theo MA, MACD, RSI và volume. Backend ưu tiên dữ liệu lịch sử thật từ `vnstock Quote.history`; nếu nguồn chưa sẵn, dashboard có cache/fallback có nhãn nguồn thay vì trả bảng trống.

### Markowitz Và Danh Mục

Tối ưu tỷ trọng danh mục, tính volatility, Sharpe, risk contribution và correlation heatmap.

### Dự Báo

So sánh ARIMA, XGBoost, VN-Edge, Edge-90 và các model deep learning có implementation thật. Khi so sánh model cần dùng cùng train/test/horizon, đọc cả accuracy, coverage, RMSE và MAE.

### Rủi Ro

Tính VaR, CVaR, stress test, Monte Carlo và copula để xem lỗ tiềm năng trong điều kiện xấu.

### Nhật Ký Giao Dịch

Ghi lại mã, giá mua, tỷ trọng, lý do mua, trạng thái bán/chưa bán và kết quả để học lại quyết định.

## Nguồn Dữ Liệu

Các nguồn đang dùng hoặc được thiết kế để dùng:

- `vnstock`: bảng giá, lịch sử giá, hồ sơ công ty, báo cáo tài chính, shares outstanding.
- CafeF: giao dịch nhà đầu tư nước ngoài, dữ liệu thị trường và một số trang tin/dữ liệu đối chiếu.
- Tuổi Trẻ RSS: tin kinh tế/chính sách cho bản đồ tin tức.
- NSO/PxWeb: GDP, CPI, xuất nhập khẩu và nhóm hàng chính.
- Tổng cục Hải quan: báo cáo xuất nhập khẩu theo tháng, đọc từ PDF công bố.
- FRED: T10Y3M; có mirror CSV để tránh kẹt khi FRED timeout.
- Vietstock: link đối chiếu thủ công cho tài chính/định giá.
- `yfinance`: nguồn thị trường phụ khi cần dữ liệu quốc tế/benchmark.

## Data Governance

- Lưu raw response khi cần truy lỗi nguồn dữ liệu.
- Mỗi bảng normalized nên có `source`, `source_url`, `as_of_date`, `ingested_at`, `checksum`.
- Không trộn dữ liệu adjusted/unadjusted nếu chưa có flag rõ ràng.
- Dùng lịch giao dịch Việt Nam khi tính return, rolling windows và backtest.
- Backtest phải tránh look-ahead bias: ngày công bố dữ liệu khác với kỳ dữ liệu.

## Cảnh Báo Pháp Lý Và Chất Lượng

- Dashboard phục vụ nghiên cứu cá nhân, không phải khuyến nghị mua/bán.
- Trước khi crawl/API từ VNDIRECT, TCBS, SSI, FireAnt, CafeF, Vietstock hoặc nguồn khác, cần kiểm tra terms of service và quyền sử dụng dữ liệu.
- Nếu dùng cho production hoặc chia sẻ rộng, nên mua data vendor/API chính thức, đặc biệt cho realtime, báo cáo tài chính và dữ liệu giao dịch.
- Kết quả forecast/risk chỉ là ước lượng từ model, cần hiển thị confidence, assumption, model version và thời điểm dữ liệu.
- Khi dữ liệu thiếu hoặc nguồn bị rate limit, dashboard phải hiện cảnh báo/cache/fallback có nhãn rõ ràng, không được vẽ kết quả đẹp giả.
- Các mô hình định giá phụ thuộc mạnh vào chất lượng BCTC, đơn vị dữ liệu, peer set và cách chuẩn hóa earnings/book value.
- Với dữ liệu bản đồ, lớp Phú Quốc, Hoàng Sa, Trường Sa trong dashboard là lớp trực quan phục vụ định vị tin tức, không phải bản đồ pháp lý hoặc bản đồ ranh giới chủ quyền.

## Kiểm Tra Build

Backend:

```powershell
.\.venv\Scripts\python.exe -m compileall backend\app
```

Frontend:

```powershell
cd frontend
npm.cmd run build
```

Vite có thể cảnh báo chunk lớn. Đây là warning tối ưu bundle, không phải lỗi runtime.

## Ghi Chú Vận Hành

- Deep learning thật chạy chậm, nên test một cổ phiếu trước.
- Danh mục nên chạy ít model forecast để tránh treo.
- NSO/Hải quan hiện được cache theo giờ; muốn ép lấy mới ngay thì cần endpoint refresh riêng.
- Nếu một nguồn dữ liệu trả thiếu, hãy chờ cache/API hồi lại hoặc đối chiếu thủ công trước khi kết luận.
- Nếu app đã mở nhưng UI cũ, nhấn `Ctrl + F5`.

## Tài Liệu Tham Khảo

### Nguồn dữ liệu thị trường và vĩ mô

- [Vnstock docs](https://vnstocks.com/docs/vnstock-data)
- [Vnstock PyPI](https://pypi.org/project/vnstock/)
- [Vnstock GitHub](https://github.com/vnstock-official/vnstock)
- [CafeF dữ liệu giao dịch](https://cafef.vn/du-lieu/Lich-su-giao-dich-can-3.chn)
- [National Statistics Office of Vietnam - Import export](https://www.nso.gov.vn/import-export)
- [NSO/PxWeb API endpoint](https://pxweb.gso.gov.vn/api/v1/en/)
- [FRED T10Y3M](https://fred.stlouisfed.org/series/T10Y3M)
- [FRED CSV mirror cho T10Y3M](https://fred.stlouisfed.org/graph/fredgraph.csv?id=T10Y3M)
- [Vietstock](https://vietstock.vn/)
- [yfinance PyPI](https://pypi.org/project/yfinance/)

### Backend, frontend và biểu đồ

- [FastAPI docs](https://fastapi.tiangolo.com/)
- [Uvicorn docs](https://www.uvicorn.org/)
- [Pydantic docs](https://docs.pydantic.dev/)
- [React docs](https://react.dev/)
- [Vite docs](https://vite.dev/)
- [Recharts docs](https://recharts.org/)
- [Lucide React](https://lucide.dev/guide/packages/lucide-react)

### Python data science

- [pandas docs](https://pandas.pydata.org/docs/)
- [NumPy docs](https://numpy.org/doc/)
- [SciPy docs](https://docs.scipy.org/doc/scipy/)
- [scikit-learn docs](https://scikit-learn.org/stable/)
- [XGBoost Python docs](https://xgboost.readthedocs.io/en/stable/python/index.html)

### Forecast, risk và portfolio

- [NeuralForecast TimesNet](https://nixtlaverse.nixtla.io/neuralforecast/models.timesnet.html)
- [NeuralForecast model overview](https://nixtlaverse.nixtla.io/neuralforecast/docs/capabilities/overview.html)
- [StatsForecast ARIMA](https://nixtlaverse.nixtla.io/statsforecast/docs/models/arima.html)
- [StatsForecast automatic forecasting](https://nixtlaverse.nixtla.io/statsforecast/docs/how-to-guides/automatic_forecasting.html)
- [statsmodels SARIMAX](https://www.statsmodels.org/stable/generated/statsmodels.tsa.statespace.sarimax.SARIMAX.html)
- [arch docs](https://arch.readthedocs.io/)
- [Copulas docs](https://sdv.dev/Copulas/)
- [PyPortfolioOpt docs](https://pyportfolioopt.readthedocs.io/)

### Tin tức, PDF và HTTP client

- [feedparser docs](https://feedparser.readthedocs.io/)
- [pypdf docs](https://pypdf.readthedocs.io/)
- [Requests docs](https://requests.readthedocs.io/)
- [HTTPX docs](https://www.python-httpx.org/)
