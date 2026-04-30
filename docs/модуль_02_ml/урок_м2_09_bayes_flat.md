# Урок М2-09: Bayes (байесовский классификатор) и Flat (флэт)

## 1. Краткое резюме

GaussianNB — вероятностный алгоритм классификации, предсказывающий рост (1) или падение (0) цены на основе технических индикаторов. В связке с определением флэта через фрактальную размерность и энтропию он позволяет боту понимать текущую фазу рынка: тренд или боковик. Это критично для выбора правильной стратегии — трендовые алгоритмы во флэте сливают, контртрендовые в тренде — тоже.

---

## 2. Ключевые концепции

- **Бинарная классификация** — целевая переменная: 1 = цена выросла, 0 = упала.
- **GaussianNB (наивный Байес)** — вероятностная модель, основанная на теореме Байеса. Лучше работает с распределением данных, чем линейные модели. Точность ~52.7% на сырых индикаторах.
- **Калибровка вероятностей** — корректировка выходных вероятностей модели, чтобы "уверен на 80%" реально означало 8 из 10 верных прогнозов.
- **Флэт (боковик)** — состояние рынка без явного тренда, цена колеблется в горизонтальном диапазоне.
- **Фрактальная размерность** — значение близко к 1.5 = флэт, сильное отклонение = начало тренда.
- **Энтропия** — мера хаоса ряда. Низкая = предсказуемый боковик, высокая = неопределённость и тренд.

---

## 3. Код урока

### Блок 1: Наивный Байес и калибровка вероятностей

```python
import yfinance as yf
import pandas as pd
import numpy as np
from sklearn.model_selection import TimeSeriesSplit, GridSearchCV
from sklearn.naive_bayes import GaussianNB
from sklearn.metrics import accuracy_score
from sklearn.calibration import CalibratedClassifierCV
import talib as ta

# Загрузка данных и целевая переменная (1=рост, 0=падение)
data = yf.download('AAPL', start='2020-01-01', end='2023-01-01')
data['Return'] = data['Adj Close'].pct_change()
data['Direction'] = (data['Return'].shift(-1) > 0).astype(int)

# Технические индикаторы как признаки
data['MA5']  = data['Adj Close'].rolling(window=5).mean()
data['MA10'] = data['Adj Close'].rolling(window=10).mean()
data['EMA12'] = data['Adj Close'].ewm(span=12, adjust=False).mean()
data['EMA26'] = data['Adj Close'].ewm(span=26, adjust=False).mean()
data['MACD'] = data['EMA12'] - data['EMA26']
data['RSI']  = ta.RSI(data['Adj Close'], timeperiod=14)
data['ATR']  = ta.ATR(data['High'], data['Low'], data['Close'], timeperiod=14)
data['ADX']  = ta.ADX(data['High'], data['Low'], data['Close'], timeperiod=14)
data['MOM']  = ta.MOM(data['Adj Close'], timeperiod=10)

for lag in range(1, 6):
    data[f'Return_lag_{lag}'] = data['Return'].shift(lag)

data = data.dropna()

features = data[['MA5', 'MA10', 'MACD', 'RSI', 'ATR', 'ADX', 'MOM',
                  'Return', 'Return_lag_1', 'Return_lag_2',
                  'Return_lag_3', 'Return_lag_4', 'Return_lag_5']]
target = data['Direction']
X, y = features.values, target.values

# Последовательное разделение train/test
split_index = int(len(X) * 0.8)
X_train, X_test = X[:split_index], X[split_index:]
y_train, y_test = y[:split_index], y[split_index:]

# Обучение с подбором гиперпараметра var_smoothing
tscv = TimeSeriesSplit(n_splits=5)
model = GaussianNB()
param_grid = {'var_smoothing': np.logspace(0, -9, num=100)}

grid_search = GridSearchCV(model, param_grid, cv=tscv, scoring='accuracy', n_jobs=-1)
grid_search.fit(X_train, y_train)
best_model = grid_search.best_estimator_

# Калибровка вероятностей
calibrated_model = CalibratedClassifierCV(best_model, cv=tscv, method='sigmoid')
calibrated_model.fit(X_train, y_train)

y_pred = calibrated_model.predict(X_test)
y_prob = calibrated_model.predict_proba(X_test)

accuracy = accuracy_score(y_test, y_pred)
print(f"Точность модели: {accuracy * 100:.2f}%")
```

### Блок 2: Определение флэта (фрактальная размерность + энтропия)

```python
from scipy.stats import entropy

def fractal_dimension(series, window=20):
    fd = []
    for i in range(len(series)):
        if i < window:
            fd.append(np.nan)
        else:
            subset = series[i - window:i]
            diffs = np.diff(subset)
            N = len(diffs)
            R = max(subset) - min(subset)
            S = np.std(diffs)
            if S == 0 or R == 0:
                fd.append(np.nan)
            else:
                fd_value = np.log(N) / (np.log(N) + np.log(R / S))
                fd.append(fd_value)
    return fd

def rolling_entropy(series, window=20, bins=10):
    ent = []
    for i in range(len(series)):
        if i < window:
            ent.append(np.nan)
        else:
            subset = series[i - window:i]
            hist, _ = np.histogram(subset, bins=bins, density=True)
            hist = hist + 1e-8  # избегаем log(0)
            ent.append(entropy(hist))
    return ent

data['Fractal_Dim'] = fractal_dimension(data['Close'].values)
data['Entropy'] = rolling_entropy(data['Close'].values)
data = data.dropna()

def market_state(row):
    if abs(row['Fractal_Dim'] - 1.5) < 0.05 and row['Entropy'] < data['Entropy'].mean():
        return 'Флэт'
    else:
        return 'Тренд/Неопределённость'

data['Market_State'] = data.apply(market_state, axis=1)
print(data[['Close', 'Fractal_Dim', 'Entropy', 'Market_State']].tail())
```

---

## 4. Важные замечания

- **Точность 52.7%** — недостаточно для самостоятельного бота без дополнительных фильтров. Лучше чем монетка, но не намного.
- **Байес vs бустинги** — CatBoost/XGBoost лучше на шумных нелинейных данных. Байес — лучше когда важны откалиброванные вероятности.
- **Порог флэта** — теоретически 1.5, но на практике по таймфрейму может сдвигаться (например до 0.8). Требует настройки.
- **Классификация vs регрессия** — классификатор отвечает "вверх или вниз", но не оценивает размах движения. Для TP/SL всё равно нужна регрессия.

---

## 5. Шпаргалка

| Действие | Функция / Параметр |
|---|---|
| Байесовский классификатор | `GaussianNB()` |
| Калибровка вероятностей | `CalibratedClassifierCV(model, cv=tscv, method='sigmoid')` |
| Вероятности классов | `model.predict_proba(X_test)` |
| Точность | `accuracy_score(y_test, y_pred)` |
| Энтропия распределения | `scipy.stats.entropy(hist)` |
| Гистограмма для энтропии | `np.histogram(subset, bins, density=True)` |

---

## 6. Применение в торговом боте (T-Invest / MOEX)

**Двухуровневая архитектура:**

1. **Фаза рынка** — в реальном времени считать `Fractal_Dim` и `Entropy` по свечам Сбербанка
2. **Флэт определён** (FD ≈ 1.5, Entropy ниже среднего) → переключиться на торговлю от границ канала
3. **Тренд определён** → переключиться на трендовую стратегию (VCP, KAMA, EMA-кроссовер)
4. **Байес внутри флэта** — если откалиброванная вероятность роста > 60–70% → открыть лонг
5. **Всплеск энтропии** → немедленно закрыть позицию (начинается непредсказуемый импульс)
6. **Признак флэта в XGBoost** — добавить `Market_State` (0/1) как признак в датасет, чтобы модель знала о фазе рынка заранее
