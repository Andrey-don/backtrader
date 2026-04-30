# Урок 05: Optuna — оптимизация стратегий Backtrader

## 1. Краткое резюме

Урок посвящён интеграции Optuna для автоматического подбора оптимальных параметров торговой стратегии. Данные — фьючерс GD1! (золото) через `price_loaders.tradingview`, часовые свечи 2400 баров. Стратегия: кроссовер двух SMA с TP/SL. Optuna перебирает параметры (100 триалов) и находит лучшую комбинацию. Финальный прогон с лучшими параметрами: 116 625.62 (+16.6%).

---

## 2. Ключевые концепции

- **Функция `objective(trial)`** — обёртка: внутри создаётся `cerebro`, запускается стратегия, возвращается итоговый баланс
- **Пространство поиска** — `suggest_int` для периодов, `suggest_float` (или устаревший `suggest_uniform`) для float-параметров
- **`study.best_trial.params`** — словарь лучших параметров после оптимизации
- **`trial.params.get('key')`** — получить конкретный параметр из словаря лучшего триала
- **`cerebro.broker.set_cash()`** — альтернативная форма `setcash()`, оба варианта работают

---

## 3. Код урока

### Загрузка данных

```python
import backtrader as bt
import pandas as pd
import optuna
import warnings

warnings.filterwarnings("ignore")

# from price_loaders.tradingview import load_asset_price


def process_tradingview_data(symbol, bars, timeframe, exchange=None):
    df = load_asset_price(symbol, bars, timeframe, exchange)
    df['time'] = pd.to_datetime(df['time']).dt.tz_localize(None) - pd.Timedelta(hours=3)
    df.rename(columns={'time': 'datetime'}, inplace=True)
    df = df[['datetime', 'open', 'high', 'low', 'close', 'volume']]
    return df


symbol = "GD1!"
df = process_tradingview_data(symbol, 2400, "1H")
```

### Стратегия

```python
class MyStrategy(bt.Strategy):
    params = (
        ("short_ma", 10),
        ("long_ma", 20),
        ("take_profit", 0.02),
        ("stop_loss", 0.01),
    )

    def __init__(self):
        self.short_ma = bt.indicators.SimpleMovingAverage(self.data.close, period=self.params.short_ma)
        self.long_ma = bt.indicators.SimpleMovingAverage(self.data.close, period=self.params.long_ma)
        self.dataclose = self.datas[0].close
        self.order = None

    def log(self, txt, dt=None):
        dt = dt or self.datas[0].datetime.date(0)
        print('%s, %s' % (dt.isoformat(), txt))

    def notify_order(self, order):
        if order.status in [order.Submitted, order.Accepted]:
            return
        if order.status in [order.Completed]:
            if order.isbuy():
                self.log('Покупка выполнена, Цена: %.2f, На сумму: %.2f, Комиссия %.2f' %
                         (order.executed.price, order.executed.value, order.executed.comm))
                self.buyprice = order.executed.price
                self.buycomm = order.executed.comm
            else:
                self.log('Продажа выполнена, Цена: %.2f, На сумму: %.2f, Комиссия %.2f' %
                         (order.executed.price, order.executed.value, order.executed.comm))
            self.bar_executed = len(self)
        elif order.status in [order.Canceled, order.Margin, order.Rejected]:
            self.log('Отмена ордера')
        self.order = None

    def notify_trade(self, trade):
        if not trade.isclosed:
            return
        self.log('Операционная прибыль, Общая %.2f, С учетом комиссии %.2f' %
                 (trade.pnl, trade.pnlcomm))

    def next(self):
        if self.position:
            if self.position.size > 0:
                if self.data.close[0] >= self.position.price * (1 + self.params.take_profit):
                    self.close()
                elif self.data.close[0] <= self.position.price * (1 - self.params.stop_loss):
                    self.close()
            elif self.position.size < 0:
                if self.data.close[0] <= self.position.price * (1 - self.params.take_profit):
                    self.close()
                elif self.data.close[0] >= self.position.price * (1 + self.params.stop_loss):
                    self.close()

        elif not self.position and self.short_ma[0] > self.long_ma[0] and self.short_ma[-1] <= self.long_ma[-1]:
            self.buy()
        elif not self.position and self.short_ma[0] < self.long_ma[0] and self.short_ma[-1] >= self.long_ma[-1]:
            self.sell()
```

### Оптимизация через Optuna

```python
def objective(trial):
    short_ma = trial.suggest_int('short_ma', 9, 25)
    long_ma = trial.suggest_int('long_ma', 33, 65)
    # suggest_uniform устарел в Optuna 3.x — используй suggest_float
    take_profit = trial.suggest_float('take_profit', 0.01, 0.05)
    stop_loss = trial.suggest_float('stop_loss', 0.005, 0.01)

    cerebro = bt.Cerebro()

    data = df.copy()
    data['datetime'] = pd.to_datetime(data['datetime'])
    data.set_index("datetime", inplace=True)
    cerebro.adddata(bt.feeds.PandasData(dataname=data))

    cerebro.addstrategy(MyStrategy, short_ma=short_ma, long_ma=long_ma,
                        take_profit=take_profit, stop_loss=stop_loss)

    cerebro.broker.set_cash(100000)
    cerebro.addsizer(bt.sizers.PercentSizer, percents=50)
    cerebro.broker.setcommission(commission=0.001)

    cerebro.run()
    return cerebro.broker.getvalue()


if __name__ == '__main__':
    study = optuna.create_study(direction='maximize')
    study.optimize(objective, n_trials=100)

    print("Number of finished trials:", len(study.trials))
    print("Best trial:")
    trial = study.best_trial
    print("Value:", trial.value)
    for key, value in trial.params.items():
        print(f"    {key}: {value}")
```

Результат оптимизации (пример):

```
short_ma: 9
long_ma: 35
take_profit: 0.03649585851873802
stop_loss: 0.007079163715405193
```

### Финальный прогон с лучшими параметрами (результат: 116 625.62)

```python
if __name__ == '__main__':
    cerebro = bt.Cerebro()

    data = df.copy()
    data['datetime'] = pd.to_datetime(data['datetime'])
    data.set_index("datetime", inplace=True)
    cerebro.adddata(bt.feeds.PandasData(dataname=data))

    # Передаём лучшие параметры через trial.params.get()
    cerebro.addstrategy(MyStrategy,
                        short_ma=trial.params.get('short_ma'),
                        long_ma=trial.params.get('long_ma'),
                        take_profit=trial.params.get('take_profit'),
                        stop_loss=trial.params.get('stop_loss'))

    cerebro.broker.set_cash(100000)
    cerebro.addsizer(bt.sizers.PercentSizer, percents=50)
    cerebro.broker.setcommission(commission=0.001)

    cerebro.run()
    print(f"Итоговый баланс: {cerebro.broker.getvalue()}")
    # Итоговый баланс: 116625.62
```

---

## 4. Важные замечания

- **`suggest_uniform` устарел** — в Optuna 3.x использовать `suggest_float(name, low, high)` вместо `suggest_uniform`
- **`cerebro.broker.set_cash()` vs `setcash()`** — оба варианта работают. `set_cash` — pythonic-стиль
- **Количество триалов** — 100 триалов достаточно для 4 параметров при демонстрации. Для production: 200–500
- **`trial.params.get('key')`** — безопасное извлечение из словаря (не бросает KeyError)
- **Данные внутри `objective`** — `df` доступен как глобальная переменная внутри функции

---

## 5. Шпаргалка

| Действие | Код |
|----------|-----|
| Создать study | `study = optuna.create_study(direction='maximize')` |
| Запустить оптимизацию | `study.optimize(objective, n_trials=100)` |
| Параметр int | `trial.suggest_int('name', min, max)` |
| Параметр float | `trial.suggest_float('name', min, max)` |
| Лучшие параметры | `study.best_trial.params` |
| Получить параметр | `trial.params.get('short_ma')` |
| Лучшее значение | `study.best_trial.value` |

---

## 6. Применение в боте (T-Invest API)

- **Оффлайн-процесс** — Optuna запускается до старта бота, на исторических данных (последние 6 месяцев из T-Invest API)
- **Результат — конфиг бота** — найденный словарь параметров (`short_ma: 9, long_ma: 35, take_profit: 0.036`) сохраняется в `.env` или JSON-конфиг live-бота
- **Переоптимизация** — рынок меняется; запускать Optuna раз в 1–2 месяца на свежих данных и обновлять параметры бота
