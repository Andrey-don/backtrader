# Урок 15: Стратегия SmartMoney для Backtrader

## 1. Краткое резюме

Урок посвящён реализации стратегии на основе кастомного индикатора MFI (Money Flow Index) в связке с EMA. MFI измеряет давление покупателей/продавцов через объём и типичную цену. Стратегия покупает при перепроданности (MFI < 20) при условии нахождения цены выше EMA, продаёт — при перекупленности (MFI > 80) ниже EMA. Параметры оптимизируются через Optuna. Данные — AAPL через yfinance.

---

## 2. Ключевые концепции

- **MFI (Money Flow Index)** — осциллятор объёма и цены. Формула: `типичная цена × объём`, разделённый на положительный и отрицательный денежный поток
- **`bt.ind.SumN`** — сумма за N периодов (аналог `.rolling(N).sum()` в pandas)
- **`bt.ind.DivByZero`** — деление с защитой от нуля: `DivByZero(a, b, zero=100.0)`
- **`tprice(-1)`** — значение типичной цены на предыдущем баре (сдвиг внутри индикатора)
- **EMA как фильтр тренда** — вход только когда цена по одну сторону от EMA

---

## 3. Код урока

```python
import backtrader as bt
import pandas as pd
import yfinance as yf
import optuna


class MFI(bt.Indicator):
    """Money Flow Index — осциллятор объёма"""
    lines = ('mfi',)
    params = dict(period=14)

    def __init__(self):
        tprice = (self.data.close + self.data.low + self.data.high) / 3.0
        mfraw = tprice * self.data.volume

        # Положительный денежный поток (когда типичная цена растёт)
        flowpos = bt.ind.SumN(mfraw * (tprice > tprice(-1)), period=self.p.period)
        # Отрицательный денежный поток
        flowneg = bt.ind.SumN(mfraw * (tprice < tprice(-1)), period=self.p.period)

        mfiratio = bt.ind.DivByZero(flowpos, flowneg, zero=100.0)
        self.l.mfi = 100.0 - 100.0 / (1.0 + mfiratio)


class MFIEMAStrategy(bt.Strategy):
    params = (
        ('mfi_period', 14),
        ('ema_period', 50),
        ('stop_loss', 0.02),    # 2% стоп-лосс
        ('take_profit', 0.05),  # 5% тейк-профит
    )

    def __init__(self):
        self.ema = bt.indicators.EMA(self.data.close, period=self.params.ema_period)
        self.mfi = MFI(self.data, period=self.params.mfi_period)
        self.stop_price = None
        self.tp_price = None

    def next(self):
        if self.position:
            if self.position.size > 0:
                if (self.data.close[0] <= self.stop_price or
                        self.data.close[0] >= self.tp_price):
                    self.close()
            elif self.position.size < 0:
                if (self.data.close[0] >= self.stop_price or
                        self.data.close[0] <= self.tp_price):
                    self.close()
        else:
            # Long: MFI перепродан + цена выше EMA
            if self.mfi[0] < 20 and self.data.close[0] > self.ema[0]:
                self.buy()
                self.stop_price = self.data.close[0] * (1 - self.params.stop_loss)
                self.tp_price = self.data.close[0] * (1 + self.params.take_profit)

            # Short: MFI перекуплен + цена ниже EMA
            elif self.mfi[0] > 80 and self.data.close[0] < self.ema[0]:
                self.sell()
                self.stop_price = self.data.close[0] * (1 + self.params.stop_loss)
                self.tp_price = self.data.close[0] * (1 - self.params.take_profit)

    def log(self, txt):
        pass

    def notify_trade(self, trade):
        if trade.isclosed:
            self.log(f"Profit: {trade.pnl:.2f}")


def get_yfinance_data(symbol, start='2020-01-01', end='2023-01-01'):
    df = yf.download(symbol, start=start, end=end, progress=False, multi_level_index=False)
    df.reset_index(inplace=True)
    df.rename(columns={'Date': 'datetime', 'Open': 'open', 'High': 'high',
                       'Low': 'low', 'Close': 'close', 'Volume': 'volume'}, inplace=True)
    df['datetime'] = pd.to_datetime(df['datetime'])
    if 'Adj Close' in df.columns:
        df.drop('Adj Close', axis=1, inplace=True)
    return df


def objective(trial):
    mfi_period = trial.suggest_int('mfi_period', 10, 20)
    ema_period = trial.suggest_int('ema_period', 20, 100)
    stop_loss = trial.suggest_float('stop_loss', 0.005, 0.03)
    take_profit = trial.suggest_float('take_profit', 0.02, 0.1)

    cerebro = bt.Cerebro()

    df = get_yfinance_data('AAPL')
    df.set_index('datetime', inplace=True)
    cerebro.adddata(bt.feeds.PandasData(dataname=df))

    cerebro.addstrategy(MFIEMAStrategy,
                        mfi_period=mfi_period,
                        ema_period=ema_period,
                        stop_loss=stop_loss,
                        take_profit=take_profit)

    cerebro.broker.setcash(10000)
    cerebro.addsizer(bt.sizers.PercentSizer, percents=90)
    cerebro.run()
    return cerebro.broker.getvalue()


if __name__ == '__main__':
    study = optuna.create_study(direction='maximize')
    study.optimize(objective, n_trials=50)

    print('Лучшие параметры:')
    for key, value in study.best_trial.params.items():
        print(f"  {key}: {value}")
```

---

## 4. Важные замечания

- **`bt.ind.SumN`** — аналог `rolling(N).sum()`. Использовать для суммирования за окно внутри индикатора
- **`bt.ind.DivByZero`** — защита от деления на ноль. Третий аргумент `zero=100.0` — значение при нулевом знаменателе
- **`tprice(-1)`** — сдвиг в `__init__` индикатора. Аналог `[-1]` в `next` стратегии
- **Стоп/тейк в `next`** — вычисляются и сохраняются в момент открытия позиции. Корректны, так как `self.buy()` в `next` означает что ордер ещё не исполнен; реальнее считать в `notify_order`

---

## 5. Шпаргалка

| Действие | Код |
|----------|-----|
| MFI индикатор | `self.mfi = MFI(self.data, period=14)` |
| Сумма за N баров | `bt.ind.SumN(line, period=N)` |
| Деление с защитой от 0 | `bt.ind.DivByZero(a, b, zero=100.0)` |
| EMA | `bt.indicators.EMA(self.data.close, period=N)` |
| Данные через yfinance | `yf.download(symbol, start=start, end=end, multi_level_index=False)` |

---

## 6. Применение в боте (T-Invest API)

- **MFI + EMA** — сильная связка для фильтрации входов: MFI отсеивает входы против объёмного давления, EMA — против тренда
- **Для MOEX** — заменить `yfinance` на `moexalgo` или T-Invest API для загрузки исторических данных российских акций
- **Оптимизация** — запускать Optuna на исторических данных целевого инструмента (не AAPL) перед запуском бота
