# Урок 11: Стратегия VCP — сжатие волатильности

## 1. Краткое резюме

Урок посвящён реализации стратегии VCP (Volatility Contraction Pattern — паттерн сжатия волатильности). Создаётся кастомный индикатор `VCPPattern`, который определяет одновременное сжатие объёма и ценового диапазона, нахождение цены на новом максимуме за 100 периодов и достаточный объём. Вход — только при подтверждении всех условий VCP + нахождении цены выше долгосрочной SMA. Данные — SBER через moexalgo (часовые свечи).

---

## 2. Ключевые концепции

- **VCP (Volatility Contraction Pattern)** — паттерн, при котором волатильность цены и объём постепенно сжимаются, предшествуя пробою
- **Кастомный индикатор** — класс, наследующий `bt.Indicator` с атрибутом `lines`. Метод `next` вычисляет значение на каждом баре
- **Условия VCP**:
  1. Сжатие объёма И цены за последние 5 баров
  2. Текущая цена — максимум за 100 периодов
  3. Текущий объём > 80% от 20-дневного среднего
- **`bt.indicators.Highest / Lowest`** — встроенные индикаторы максимума/минимума за N периодов
- **`bt.indicators.StandardDeviation`** — стандартное отклонение цены (мера волатильности)
- **Узкий канал** — `recent_close_min > recent_close_max * 0.7` — цена консолидируется в узком диапазоне

---

## 3. Код урока

```python
import backtrader as bt
import pandas as pd
from moexalgo import Ticker


def process_candles(data, ticker_name):
    df = pd.DataFrame(data)
    df.drop(columns=['end'], inplace=True)
    df.rename(columns={'begin': 'datetime'}, inplace=True)
    df['datetime'] = pd.to_datetime(df['datetime'])
    df['ticker'] = ticker_name
    df = df[['datetime', 'open', 'high', 'low', 'close', 'volume']]
    return df

ticker = 'SBER'
data = Ticker(ticker).candles(start='2022-02-23', end='2024-10-27', period='1h')
df = process_candles(data, ticker)


class VCPPattern(bt.Indicator):
    """
    Паттерн сжатия волатильности (Volatility Contraction Pattern).
    Возвращает 1 если все условия выполнены, иначе -1.
    """
    lines = ('vcp',)

    def __init__(
        self,
        period_short: int = 10,
        period_long: int = 60,
        period_long_discount: float = 0.7,
        highest_close: int = 100,
        mean_vol: int = 20,
    ):
        # Сжатие объёма: короткий средний < длинный средний * discount
        volume_short_avg = bt.indicators.MovingAverageSimple(self.data.volume, period=period_short)
        volume_long_avg = bt.indicators.MovingAverageSimple(self.data.volume, period=period_long)
        self.volume_reduce = volume_short_avg < (volume_long_avg * period_long_discount)

        # Сжатие цены: короткое std < длинное std * discount
        price_short_std = bt.indicators.StandardDeviation(self.data.close, period=period_short)
        price_long_std = bt.indicators.StandardDeviation(self.data.close, period=period_long)
        self.price_contract = price_short_std < (price_long_std * period_long_discount)

        self.highest_close = bt.indicators.Highest(self.data.close, period=highest_close)
        self.mean_vol = bt.indicators.MovingAverageSimple(self.data.volume, period=mean_vol)

    def next(self):
        # Условие 1: сжатие объёма И цены за последние 5 баров
        volume_reduce_5d = list(self.volume_reduce.get(size=5))
        price_contract_5d = list(self.price_contract.get(size=5))
        cond_1 = any(v > 0.0 and p > 0.0 for v, p in zip(volume_reduce_5d, price_contract_5d))

        # Условие 2: цена на максимуме за 100 периодов
        cond_2 = self.data.close[0] == self.highest_close[0]

        # Условие 3: текущий объём > 80% от 20-дневного среднего
        cond_3 = self.data.volume[0] > self.mean_vol[0] * 0.8

        self.lines.vcp[0] = 1 if cond_1 and cond_2 and cond_3 else -1


class TestStrategy(bt.Strategy):
    params = dict(
        period_short=10,
        period_long=60,
        period_long_discount=0.7,
        highest_close=100,
        mean_vol=20,
        sma_long=250,
        sma_short=60,
        recent_price_period=35,
    )

    def log(self, txt, dt=None):
        dt = dt or self.datas[0].datetime.date(0)
        print('%s, %s' % (dt.isoformat(), txt))

    def __init__(self):
        self.dataclose = self.datas[0].close
        self.order = None
        self.buyprice = None
        self.buycomm = None

        self.vcp = VCPPattern(
            self.data,
            period_short=self.params.period_short,
            period_long=self.params.period_long,
            period_long_discount=self.params.period_long_discount,
            highest_close=self.params.highest_close,
            mean_vol=self.params.mean_vol,
        )
        self.sma_long = bt.indicators.MovingAverageSimple(self.data.close, period=self.params.sma_long)
        self.sma_short = bt.indicators.MovingAverageSimple(self.data.close, period=self.params.sma_short)

        recent_close_min = bt.indicators.Lowest(self.data.close, period=self.params.recent_price_period)
        recent_close_max = bt.indicators.Highest(self.data.close, period=self.params.recent_price_period)
        # Узкий ценовой канал — консолидация перед пробоем
        self.narrow_channel = recent_close_min > recent_close_max * 0.7

    def notify_order(self, order):
        if order.status in [order.Submitted, order.Accepted]:
            return
        if order.status in [order.Completed]:
            if order.isbuy():
                self.log('Покупка: %.2f, Сумма: %.2f, Комиссия: %.2f' %
                         (order.executed.price, order.executed.value, order.executed.comm))
                self.buyprice = order.executed.price
                self.buycomm = order.executed.comm
            else:
                self.log('Продажа: %.2f, Сумма: %.2f, Комиссия: %.2f' %
                         (order.executed.price, order.executed.value, order.executed.comm))
            self.bar_executed = len(self)
        elif order.status in [order.Canceled, order.Margin, order.Rejected]:
            self.log('Отмена ордера')
        self.order = None

    def notify_trade(self, trade):
        if not trade.isclosed:
            return
        self.log('P&L: Общая %.2f, С комиссией %.2f' % (trade.pnl, trade.pnlcomm))

    def next(self):
        if self.order:
            return

        cond_1 = self.vcp[0] > 0                              # VCP паттерн активен
        cond_2 = self.data.volume[0] * self.data.close[0] > 2_000_000  # достаточная ликвидность
        cond_3 = self.data.close[0] > self.sma_long[0]        # выше долгосрочного тренда
        cond_4 = self.narrow_channel[0] > 0                   # узкий канал (консолидация)

        buy_signal = cond_1 and cond_2 and cond_3 and cond_4
        close_signal = self.data.close[0] < self.sma_short[0]  # ниже краткосрочного тренда

        if self.position.size == 0:
            if buy_signal:
                self.buy()
        else:
            if close_signal:
                self.close()


if __name__ == '__main__':
    cerebro = bt.Cerebro()
    cerebro.addstrategy(TestStrategy)
    cerebro.addanalyzer(bt.analyzers.PyFolio, _name='PyFolio')

    data = df.copy()
    data['datetime'] = pd.to_datetime(data['datetime'])
    data.set_index('datetime', inplace=True)
    cerebro.adddata(bt.feeds.PandasData(dataname=data))

    cerebro.broker.setcash(100000.0)
    cerebro.addsizer(bt.sizers.PercentSizer, percents=90)
    cerebro.broker.setcommission(commission=0.001)

    print('Стартовый портфель: %.2f' % cerebro.broker.getvalue())
    results = cerebro.run()
    print('Конечный портфель: %.2f' % cerebro.broker.getvalue())
    # Результат на SBER 2022-2024: ~134 710
```

---

## 4. Важные замечания

- **Кастомный индикатор** — `lines = ('vcp',)` объявляет линию. В `next` заполняем `self.lines.vcp[0]`
- **`.get(size=N)` внутри индикатора** — работает так же, как в стратегии. Возвращает list последних N значений
- **`bt.indicators.StandardDeviation`** — встроенный. Не путать с numpy std
- **Результат на SBER** — стратегия показала +34% за 2022-2024 на часовых свечах Сбера

---

## 5. Шпаргалка

| Действие | Код |
|----------|-----|
| Кастомный индикатор | `class MyInd(bt.Indicator): lines = ('myline',)` |
| Запись значения линии | `self.lines.myline[0] = value` |
| Максимум за N баров | `bt.indicators.Highest(self.data.close, period=N)` |
| Минимум за N баров | `bt.indicators.Lowest(self.data.close, period=N)` |
| Стандартное отклонение | `bt.indicators.StandardDeviation(self.data.close, period=N)` |
| Средний объём | `bt.indicators.MovingAverageSimple(self.data.volume, period=N)` |

---

## 6. Применение в боте (T-Invest API)

- **VCP как сигнал** — при получении новой свечи от T-Invest проверяем `vcp[0] > 0` и остальные условия. При совпадении — отправляем рыночный ордер
- **Ликвидность** — `volume * close > 2_000_000` защищает от входа в неликвидные бумаги. Критично для MOEX
- **Выход по SMA** — простой и надёжный выход. Не требует сложных расчётов в реальном времени
