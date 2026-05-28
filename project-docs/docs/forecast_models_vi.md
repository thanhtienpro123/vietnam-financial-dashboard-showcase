# Giải thích các mô hình dự báo

File này giải thích đúng theo code hiện tại trong `backend/app/services/forecast.py`.

## Cách app chia dữ liệu

Khi chạy dự báo, app chia chuỗi giá thành 3 phần:

```text
Train: học từ quá khứ
Test: giá thật trong quá khứ, dùng để chấm điểm model
Forecast: dự báo tương lai
```

Ví dụ:

```text
Cửa sổ huấn luyện = 2 năm -> khoảng 504 phiên
Cửa sổ kiểm định = 3 tháng -> khoảng 63 phiên
Kỳ dự báo = 1 tháng -> 22 phiên tương lai
```

Metric `Đúng toàn test`, `RMSE`, `MAE` được tính trên phần test, không tính trên phần forecast tương lai.

Giá được đưa về thang giá gốc khi chấm điểm. Vì vậy `RMSE` và `MAE` là đơn vị giá thật, không phải z-score hay dữ liệu chuẩn hoá.

## Công thức chung

Nhiều model trong app dự báo return trước, rồi đổi thành giá:

```text
return_dự_báo = log(giá_dự_báo / giá_hôm_qua)
giá_dự_báo = giá_hôm_qua * exp(return_dự_báo)
```

Với forecast nhiều ngày, các model như XGBoost, VN-Broker, VN-Alpha, VN-Edge, Edge-90 dùng cách recursive:

```text
Dự báo T+1 từ dữ liệu thật
Dự báo T+2 từ dữ liệu thật + giá dự báo T+1
Dự báo T+3 từ dữ liệu thật + giá dự báo T+1 + T+2
...
```

Cách này hợp lý cho forecast nhiều bước, nhưng sai số có thể tích luỹ theo thời gian.

## ARIMA

Code chính:

```text
_arima_path
_best_statsmodels_arima_order
_statsforecast_autoarima_path
```

Thư viện dùng:

```python
from statsmodels.tsa.arima.model import ARIMA
```

Nếu cài được `statsforecast`, app có thử dùng:

```python
from statsforecast.models import AutoARIMA
```

Ý tưởng:

ARIMA dự báo giá dựa trên cấu trúc tự tương quan trong chuỗi thời gian. App lấy log giá:

```text
y_t = log(price_t)
```

Sau đó thử nhiều bộ tham số `(p, d, q)` và chọn bộ có AIC thấp nhất:

```text
p: số lag tự hồi quy
d: số lần sai phân để làm chuỗi ổn định
q: số lag của sai số
```

Công thức trực giác:

```text
giá hôm nay được giải thích bằng giá các ngày trước + sai số các ngày trước
```

Trong test, app walk-forward từng phiên:

```text
Tại mỗi ngày test:
  chỉ dùng dữ liệu trước ngày đó
  fit ARIMA
  dự báo 1 ngày tiếp theo
```

Nhận xét:

- Dùng model ARIMA thật.
- Tốt cho chuỗi có tính tự tương quan ổn định.
- Thường kém với thị trường có nhiều tin tức/đột biến.

## XGBoost

Code chính:

```text
_xgboost_path
_feature_from_history
```

Thư viện dùng:

```python
from xgboost import XGBRegressor
```

Target model học:

```text
target = log(price_t / price_{t-1})
```

Tức model không học trực tiếp giá, mà học return ngày tiếp theo.

Feature gồm:

```text
return lag 1, 2, 3, 5, 10 phiên
rolling mean return 5, 10, 21, 63 phiên
rolling std 10, 21, 63 phiên
momentum 5, 21, 63 phiên
price z-score 63 phiên
```

Công thức giá:

```text
return_dự_báo = XGBRegressor(features)
giá_dự_báo = giá_gần_nhất * exp(return_dự_báo)
```

Thông số hiện tại:

```text
n_estimators = 180
max_depth = 2
learning_rate = 0.045
subsample = 0.92
colsample_bytree = 0.92
```

Nhận xét:

- Dùng XGBoost thật.
- Thường cho RMSE/MAE tốt vì học biên độ return.
- Có thể đúng hướng kém hơn model directional nếu thị trường nhiễu.

## VN-Broker

Code chính:

```text
_vn_broker_path
_broker_technical_return
```

Đây là model custom, tái lập phong cách technical rating công khai của các công ty chứng khoán. Nó không phải model nội bộ/proprietary của broker.

Thành phần tín hiệu:

```text
MA trend: MA5, MA10, MA20, MA50, MA100
Momentum: return trung bình 5, 21, 63 phiên
MACD histogram
RSI
Bollinger/z-score
Breakout 20 phiên
Volatility 63 phiên
```

Trend score:

```text
Mỗi điều kiện đúng xu hướng +1, sai -1:
  price > MA5
  MA5 > MA10
  MA10 > MA20
  MA20 > MA50
  MA50 > MA100
  price > MA20
  price > MA50

trend_score = trung bình các vote
```

Momentum score:

```text
momentum_score = tanh(
  0.55 * mean_return_5 / volatility
  + 0.35 * mean_return_21 / volatility
  + 0.10 * mean_return_63 / volatility
)
```

MACD:

```text
MACD = EMA12(log_price) - EMA26(log_price)
MACD_signal = EMA9(MACD)
MACD_histogram = MACD - MACD_signal
macd_score = tanh(MACD_histogram / volatility)
```

RSI:

```text
rsi_score = tanh((RSI - 0.5) * 3)
```

Bollinger/z-score:

```text
z = (log_price_hiện_tại - mean_20) / std_20
mean_reversion_score = tanh(-z / 2.5)
```

Breakout:

```text
price >= max_20 -> +1
price <= min_20 -> -1
ngược lại -> 0
```

Tổng score:

```text
score =
  0.34 * trend_score
  + 0.24 * momentum_score
  + 0.16 * macd_score
  + 0.10 * rsi_score
  + 0.10 * breakout_score
  + 0.06 * mean_reversion_score
```

Đổi score thành return:

```text
magnitude = min(volatility * 1.15, abs(score) * volatility * 0.92)
broker_return = sign(score) * magnitude
```

Sau đó blend với XGBoost:

```text
return_cuối = 0.68 * broker_return + 0.32 * xgboost_return
giá_dự_báo = giá_gần_nhất * exp(return_cuối)
```

Nhận xét:

- Không phải model ML thư viện.
- Tốt để mô phỏng cách technical rating.
- Trong test gần nhất, hay cho RMSE/MAE tốt, nhưng đúng hướng không phải lúc nào cao nhất.

## VN-Edge

Code chính:

```text
_edge_full_coverage_path(strict=False)
_edge_training_rows
_edge_fit_models
_edge_predict_proba
_edge_price_from_probability
```

Đây là custom directional ensemble. Mục tiêu chính là dự đoán hướng tăng/giảm.

Target:

```text
future_return = log(price_t / price_{t-1})
target = 1 nếu future_return > 0
target = 0 nếu future_return < 0
```

Feature gồm nhiều technical features:

```text
lag return 1, 2, 3, 5, 10, 21, 42
rolling mean 3, 5, 10, 21, 42, 63
rolling std 5, 10, 21, 42, 63
momentum 3, 5, 10, 21, 42, 63
price z-score 20, 60, 126
volatility ratio
RSI
MACD
max drawdown
autocorrelation
skew
kurtosis
```

Model con trong ensemble:

```text
XGBClassifier nếu có xgboost
ExtraTreesClassifier
HistGradientBoostingClassifier
LogisticRegression + StandardScaler
```

Xác suất:

```text
probability = trung bình xác suất tăng của các classifier
```

Đổi probability thành giá:

```text
edge = abs(probability - 0.5) * 2
direction = +1 nếu probability >= 0.5, ngược lại -1
magnitude = hàm của volatility và median abs return
giá_dự_báo = giá_gần_nhất * exp(direction * magnitude)
```

Nhận xét:

- Là model custom thật, có train classifier.
- Không phải model có sẵn trong thư viện.
- Hiện đã forecast full coverage để so công bằng.

## Edge-90

Code chính:

```text
_edge_full_coverage_path(strict=True)
_edge90_path
```

Edge-90 là bản strict của VN-Edge.

Khác biệt chính:

```text
probability_strict = 0.5 + (probability - 0.5) * 0.78
```

Nghĩa là nếu model quá tự tin, app kéo xác suất về gần 0.5 hơn. Điều này làm dự báo thận trọng hơn.

Tên `Edge-90` có lịch sử từ bản cũ: trước đây nó chỉ phát tín hiệu nếu validation đạt ngưỡng 90%. Hiện tại đã sửa để forecast đủ mỗi phiên test, nên nó không còn bỏ trống dự báo nữa.

Nhận xét:

- Không phải model thư viện.
- Là custom strict version của VN-Edge.
- Nên đọc như "Edge thận trọng", không nên hiểu là đảm bảo đúng 90%.

## VN-Alpha

Code chính:

```text
_vn_alpha_path
_blend_forecast_paths
```

Đây là custom ensemble.

Hiện tại công thức đang chạy:

```text
VN-Alpha = 75% VN-Edge + 25% XGBoost
```

Cụ thể:

```text
giá_vn_alpha = 0.75 * giá_vn_edge + 0.25 * giá_xgboost
```

Lý do:

```text
VN-Edge thường giữ hướng tốt hơn.
XGBoost thường giữ sai số giá thấp hơn.
Blend 75/25 giúp giữ đúng hướng của Edge và kéo MAE/RMSE xuống.
```

Nhận xét:

- Không phải model thư viện.
- Là model champion blend hiện tại.
- Nếu muốn tối ưu nghiêm túc hơn, nên quét nhiều mã VN30/VN100 và tối ưu weight theo validation rolling.

## Deep learning NeuralForecast

Code chính:

```text
_build_neural_model
_neural_kwargs
_neuralforecast_path
```

Thư viện:

```python
from neuralforecast import NeuralForecast
from neuralforecast.models import ...
```

Mapping model:

```text
LSTM -> neuralforecast.models.LSTM
GRU -> neuralforecast.models.GRU
Transformer -> VanillaTransformer
TimesNet -> TimesNet
DLinear -> DLinear
NLinear -> NLinear
PatchTST -> PatchTST
iTransformer -> iTransformer
Autoformer -> Autoformer
FEDformer -> FEDformer
Informer -> Informer
N-BEATS -> NBEATS
N-HiTS -> NHITS
```

Input:

```text
y = log(price)
```

App train NeuralForecast trên chuỗi log giá, sau đó đổi ngược:

```text
giá_dự_báo = exp(y_dự_báo)
```

`Epoch DL` trong UI được map vào:

```text
max_steps
```

Nếu chọn epoch cao hơn:

```text
train lâu hơn
có thể học kỹ hơn
nhưng có thể overfit
không chắc tốt hơn
```

Nhận xét:

- Đây là model deep learning thật nếu package `neuralforecast` cài đủ.
- Chạy chậm hơn các model fast.
- Nên test một cổ phiếu, không nên chạy nhiều model deep learning trên cả danh mục.

## Cách chấm điểm

Với mỗi ngày test:

```text
nếu giá_dự_báo > giá_hôm_qua và giá_thật > giá_hôm_qua -> đúng hướng
nếu giá_dự_báo < giá_hôm_qua và giá_thật < giá_hôm_qua -> đúng hướng
ngược lại -> sai hướng
```

`Đúng toàn test`:

```text
số_ngày_đúng_hướng / tổng_số_ngày_test
```

`RMSE`:

```text
sqrt(mean((giá_dự_báo - giá_thật)^2))
```

`MAE`:

```text
mean(abs(giá_dự_báo - giá_thật))
```

`Độ phủ`:

```text
số_ngày_có_dự_báo / tổng_số_ngày_test
```

Hiện tại các model chính được sửa để full coverage, nên độ phủ nên là:

```text
63/63 nếu test 3 tháng
42/42 nếu test 2 tháng
```

## Model nào là "dùng model thật"?

```text
ARIMA: dùng statsmodels ARIMA thật
XGBoost: dùng XGBRegressor thật
Deep learning: dùng NeuralForecast thật nếu cài đủ package
VN-Broker: custom rule/technical model
VN-Alpha: custom ensemble 75% VN-Edge + 25% XGBoost
VN-Edge: custom classifier ensemble
Edge-90: custom strict classifier ensemble
```

## Lưu ý quan trọng

Không model nào được chứng minh đạt 90% ổn định trên toàn thị trường Việt Nam. Kết quả phải đọc theo backtest nhiều mã, nhiều giai đoạn, và cùng một train/test split.

Nếu chỉ nhìn một mã như FPT trong một cửa sổ test, kết quả có thể đẹp nhưng chưa đủ để kết luận model tốt cho toàn thị trường.

## Research/tuning tham số

Trong UI, checkbox `Research tune` có nghĩa là:

```text
1. Cắt một phần cuối tập train làm validation.
2. Lấy tham số đã lưu riêng cho mã cổ phiếu nếu có.
3. Tạo không gian tham số theo range cho từng model.
4. Lấy mẫu có giới hạn trong range để tránh treo máy, nhất là deep learning.
5. Thử lại tham số đã lưu và các biến thể xung quanh nó.
6. Chọn cấu hình có score validation tốt nhất.
7. Chạy test/forecast bằng cấu hình đó.
8. Tự động lưu cấu hình tốt nhất vào backend/app/data/forecast_model_params.json.
```

Nếu không bật `Research tune`, model chạy cấu hình mặc định và không dùng file tham số đã lưu. Như vậy kết quả bình thường không bị lệch bởi các lần research trước.

Lần sau khi bật `Research tune` cho cùng một mã, app sẽ dùng cấu hình đã lưu làm điểm xuất phát để tìm tiếp, thay vì bắt đầu lại từ đầu.

Script research tổng quát:

```powershell
.\.venv\Scripts\python.exe tools\research_model_tuning.py --symbols FPT SSI VNM HPG MWG --models XGBoost VN-Broker VN-Alpha VN-Edge Edge-90 --mode fast
```

Chạy cả một nhóm deep learning nhỏ:

```powershell
.\.venv\Scripts\python.exe tools\research_model_tuning.py --symbols FPT --models DLinear NLinear TimesNet --mode balanced --test-months 2
```

Chạy tất cả model sẽ rất chậm:

```powershell
.\.venv\Scripts\python.exe tools\research_model_tuning.py --symbols FPT --models ALL --mode balanced
```

Output nằm trong `outputs/`:

```text
model_tuning_YYYYMMDD_HHMMSS.csv
model_tuning_YYYYMMDD_HHMMSS.json
```

Cột quan trọng:

```text
score: điểm tổng hợp để xếp hạng
full_directional_accuracy: tỷ lệ đúng hướng trên toàn test
rmse: sai số giá, phạt nặng lỗi lớn
mae: sai số giá trung bình
coverage: độ phủ dự báo
params: cấu hình/epoch của variant đó
```

Công thức score mặc định:

```text
score = 0.60 * đúng_hướng - 0.25 * MAE/mean_price - 0.15 * RMSE/mean_price - penalty_nếu_coverage_thấp
```

Nếu ưu tiên trading/đúng hướng hơn, tăng:

```powershell
--score-direction-weight 0.75 --score-mae-weight 0.15 --score-rmse-weight 0.10
```

Nếu ưu tiên giá sát hơn, tăng:

```powershell
--score-direction-weight 0.40 --score-mae-weight 0.40 --score-rmse-weight 0.20
```
