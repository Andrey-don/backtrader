# Урок М2-05: Оптимизация гиперпараметров (Optuna + Backtrader)

## 1. Краткое резюме

Optuna используется для автоматического подбора наилучших параметров ML-моделей и торговых стратегий. В контексте ML — минимизирует ошибку прогноза (MSE). В связке с Backtrader — подбирает периоды индикаторов, стоп-лоссы и тейк-профиты так, чтобы максимизировать конечный баланс депозита после исторического бэктеста.

---

## 2. Ключевые концепции

- **Оптимизация** — процесс перебора параметров алгоритма для достижения лучшего результата (минимизации ошибки или максимизации прибыли).
- **Objective (целевая функция)** — функция, которая выполняет вычисления (обучает ML-модель или прогоняет Cerebro) и возвращает итоговую метрику.
- **Trial (триал)** — одна итерация оптимизации: Optuna проверяет одну конкретную комбинацию параметров.
- **Study** — объект Optuna, управляющий триалами и хранящий историю лучших найденных параметров.
- **Direction** — указание что делать с метрикой: `minimize` (уменьшать ошибку) или `maximize` (увеличивать баланс).

---

## 3. Код урока

### Блок 1: Оптимизация ML-модели (Lasso)

```python
import optuna
from sklearn.linear_model import Lasso
from sklearn.metrics import mean_squared_error

def objective(trial):
    alpha = trial.suggest_float('alpha', 0.01, 1.0)
    model = Lasso(alpha=alpha)
    model.fit(X_train, y_train)
    y_pred = model.predict(X_test)
    mse = mean_squared_error(y_test, y_pred)
    return mse

study = optuna.create_study(direction='minimize')
study.optimize(objective, n_trials=100)

print('Best trial:')
trial = study.best_trial
print('Value (MSE): ', trial.value)
```

### Блок 2: Оптимизация торговой стратегии в Backtrader

```python
import backtrader as bt
import optuna
import pandas as pd

def objective(trial):
    short_ma = trial.suggest_int('short_ma', 5, 25)
    long_ma = trial.suggest_int('long_ma', 25, 42)
    take_profit = trial.suggest_float('take_profit', 0.01, 0.5)
    stop_loss = trial.suggest_float('stop_loss', 0.005, 0.3)

    cerebro = bt.Cerebro()

    data_bt = df.copy()
    data_bt['datetime'] = pd.to_datetime(data_bt['datetime'])
    data_bt.set_index('datetime', inplace=True)
    data_feed = bt.feeds.PandasData(dataname=data_bt)
    cerebro.adddata(data_feed)

    cerebro.addstrategy(MyStrategy, short_ma=short_ma, long_ma=long_ma,
                        take_profit=take_profit, stop_loss=stop_loss)

    cerebro.broker.set_cash(100000)
    cerebro.addsizer(bt.sizers.PercentSizer, percents=10)
    cerebro.broker.setcommission(commission=0.001)

    cerebro.run()
    return cerebro.broker.getvalue()

if __name__ == '__main__':
    study = optuna.create_study(direction='maximize')
    study.optimize(objective, n_trials=100)

    trial = study.best_trial
    print("Value: ", trial.value)
    for key, value in trial.params.items():
        print(f"    {key}: {value}")
```

---

## 4. Важные замечания

- **Время вычислений** — каждый триал запускает полный бэктест Cerebro. 100 триалов = 100 бэктестов. Для сложных стратегий может занять часы.
- **Переобучение (главный риск)** — Optuna найдёт идеальные параметры для прошлого периода, которые могут не работать на новых данных. Обязательна валидация на независимой выборке (out-of-sample).
- **`suggest_uniform` устарел** — в новых версиях Optuna заменён на `suggest_float`. В коде выше уже исправлено.

---

## 5. Шпаргалка

| Действие | Функция / Параметр |
|---|---|
| Создание сессии | `optuna.create_study(direction='...')` |
| Направление для ML | `direction='minimize'` |
| Направление для трейдинга | `direction='maximize'` |
| Запуск оптимизации | `study.optimize(objective, n_trials=100)` |
| Целое число (периоды MA) | `trial.suggest_int('name', min, max)` |
| Дробное число (TP/SL) | `trial.suggest_float('name', min, max)` |
| Лучшие параметры | `study.best_trial.params` |

---

## 6. Применение в торговом боте (T-Invest / MOEX)

**Правильный процесс — с валидацией:**

1. **Разделение данных** — например, 2021–2023 для Optuna (`n_trials=200`), 2024 год отложить.
2. **Оптимизация** — Optuna выдаёт `best_params` (периоды MA, пороги стопов).
3. **Валидация** — прогнать один бэктест на данных 2024 года (которые Optuna не видела).
4. **Решение** — если 2024 год в плюсе → параметры робастны, можно внедрять. Если в минусе → стратегия переобучилась, менять логику входа.
5. **Скользящая оптимизация** — запускать Optuna раз в 1–2 месяца для "освежения" параметров под меняющуюся волатильность рынка.
