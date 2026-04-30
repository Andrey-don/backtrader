# Урок 01: Backtrader — вводное

**Курс:** VesperfinCode. Модуль 2 «Алготрейдинг Про» (Арина Веспер)
**Файл:** `E:\[Арина Веспер]...\01_Бэктестинг стратегий\01. Вводная информация по Backtrader\1. Backtrader - вводное\`

---

## 1. Что такое Backtrader

- Python-библиотека для **бэктестинга** торговых стратегий
- Поддерживает: акции, фьючерсы, криптовалюту
- Установка: `pip install backtrader`
- Дополнительно для отчётов: `pip install quantstats`
- Огромная библиотека встроенных индикаторов (`bt.indicators.*`)
- Активно поддерживается сообществом

---

## 2. Архитектура: движок Cerebro

**Cerebro** — центральный движок Backtrader. Всё проходит через него.

```python
import backtrader as bt

if __name__ == '__main__':
    cerebro = bt.Cerebro()                        # создать движок
    cerebro.broker.setcash(100_000.0)             # установить депозит
    cerebro.broker.setcommission(commission=0.001) # комиссия 0.1%
    cerebro.addsizer(bt.sizers.FixedSize, stake=10) # размер позиции (кол-во акций)

    print('Стартовый портфель: %.2f' % cerebro.broker.getvalue())
    results = cerebro.run()
    print('Конечный портфель: %.2f' % cerebro.broker.getvalue())
```

> По умолчанию депозит = 10 000 у.е.

---

## 3. Загрузка данных (Finam-парсер → Cerebro)

В ноутбуке урока данные загружаются напрямую с Finam через HTTP-запрос.

```python
from urllib.parse import urlencode
from urllib.request import urlopen
from datetime import datetime
import pandas as pd
from io import StringIO

def download_finam_data(ticker, period, start_date, end_date):
    FINAM_URL = "http://export.finam.ru/"
    market = 0

    # Словарь тикер → ID инструмента на Finam
    tickers = {
        'SBER': 3, 'GAZP': 16842, 'LKOH': 8, 'AFLT': 29,
        'GMKN': 795, 'NVTK': 17370, 'ROSN': 17273, 'VTBR': 19043,
        # ... полный список в исходном ноутбуке
    }
    periods = {
        'tick': 1, 'min': 2, '5min': 3, '10min': 4,
        '15min': 5, '30min': 6, 'hour': 7, 'daily': 8, 'week': 9, 'month': 10
    }

    start_date_obj = datetime.strptime(start_date, '%d.%m.%Y')
    end_date_obj = datetime.strptime(end_date, '%d.%m.%Y')
    start_date_rev = start_date_obj.strftime('%Y%m%d')
    end_date_rev = end_date_obj.strftime('%Y%m%d')

    params = urlencode([
        ('market', market), ('em', tickers[ticker]), ('code', ticker),
        ('df', start_date_obj.day), ('mf', start_date_obj.month - 1), ('yf', start_date_obj.year),
        ('from', start_date),
        ('dt', end_date_obj.day), ('mt', end_date_obj.month - 1), ('yt', end_date_obj.year),
        ('to', end_date),
        ('p', period),
        ('f', ticker + "_" + start_date_rev + "_" + end_date_rev),
        ('e', ".csv"), ('cn', ticker), ('dtf', 1), ('tmf', 1),
        ('MSOR', 0), ('mstime', "on"), ('mstimever', 1),
        ('sep', 1), ('sep2', 1), ('datf', 1), ('at', 1),
    ])

    url = FINAM_URL + ticker + "_" + start_date_rev + "_" + end_date_rev + ".csv?" + params
    txt = urlopen(url).readlines()
    txt_str = b''.join(txt).decode()

    data = pd.read_csv(
        StringIO(txt_str),
        parse_dates={'datetime': ['<DATE>', '<TIME>']},
        dayfirst=True,
    )
    data.rename(columns={
        '<TICKER>': 'ticker', '<PER>': 'per',
        '<OPEN>': 'open', '<HIGH>': 'high',
        '<LOW>': 'low', '<CLOSE>': 'close', '<VOL>': 'volume',
    }, inplace=True)
    return data


# Загрузка AFLT дневные свечи 2022-2023
df = download_finam_data("AFLT", 8, "20.01.2022", "21.11.2023")
```

### Вставка данных в Cerebro

```python
data = df.copy()
data["datetime"] = pd.to_datetime(data["datetime"])
data.set_index("datetime", inplace=True)
data_feed = bt.feeds.PandasData(dataname=data)
cerebro.adddata(data_feed)
```

**Важно:**
- Индекс DataFrame должен быть типа `datetime`
- `bt.feeds.PandasData` принимает стандартные OHLCV-колонки
- Источник данных не важен — подходит Finam, moexalgo, T-Invest API и др.

---

## 4. Структура класса Strategy

Каждая стратегия — это класс, наследующийся от `bt.Strategy`.

```python
class TestStrategy(bt.Strategy):

    params = (
        ('exitbars', 5),   # параметры стратегии — здесь выход через N баров
        ('ma_period', 15), # период скользящей средней
    )

    def log(self, txt, dt=None):
        """Логирование — печатает дату и сообщение."""
        dt = dt or self.datas[0].datetime.date(0)
        print('%s, %s' % (dt.isoformat(), txt))

    def __init__(self):
        # Взятие цены закрытия текущего бара
        self.dataclose = self.datas[0].close

        # Хранение текущего ордера (чтобы не дублировать)
        self.order = None
        self.bar_executed = 0

        # Добавление индикатора SMA
        self.sma = bt.indicators.SimpleMovingAverage(
            self.datas[0], period=self.params.ma_period
        )

    def notify_order(self, order):
        """Вызывается при изменении статуса ордера."""
        if order.status in [order.Submitted, order.Accepted]:
            return  # ждём исполнения

        if order.status in [order.Completed]:
            if order.isbuy():
                self.log('Покупка выполнена, цена: %.2f, комиссия: %.2f' %
                         (order.executed.price, order.executed.comm))
                self.bar_executed = len(self)
            elif order.issell():
                self.log('Продажа выполнена, цена: %.2f, комиссия: %.2f' %
                         (order.executed.price, order.executed.comm))

        elif order.status in [order.Canceled, order.Margin, order.Rejected]:
            self.log('Ордер отклонён/отменён/маржа')

        self.order = None  # сбросить текущий ордер

    def notify_trade(self, trade):
        """Вызывается при закрытии сделки (круга)."""
        if not trade.isclosed:
            return
        self.log('Операционная прибыль: %.2f, с учётом комиссии: %.2f' %
                 (trade.pnl, trade.pnlcomm))

    def next(self):
        """Основная логика стратегии — вызывается на каждом баре."""
        self.log('Close, %.2f' % self.dataclose[0])

        # Если есть активный ордер — не посылать новый
        if self.order:
            return

        if not self.position:
            # --- ВХОД В РЫНОК ---
            if self.dataclose[0] > self.sma[0]:
                self.log('Ордер на покупку создан, %.2f' % self.dataclose[0])
                self.order = self.buy()
        else:
            # --- ВЫХОД ИЗ РЫНКА ---
            if self.dataclose[0] < self.sma[0]:
                self.log('Ордер на продажу создан, %.2f' % self.dataclose[0])
                self.order = self.sell()
            # Альтернативный выход — по числу баров:
            # if len(self) >= (self.bar_executed + self.params.exitbars):
            #     self.order = self.sell()
```

---

## 5. Индексация баров — ВАЖНО

| Запись | Значение |
|--------|----------|
| `self.dataclose[0]` | Текущая цена закрытия |
| `self.dataclose[-1]` | Предыдущая цена закрытия |
| `self.dataclose[-2]` | Позапрошлая цена закрытия |
| `self.sma[0]` | Текущее значение SMA |

> Это **обратная** логика от pandas, где `-1` — последний элемент.
> В Backtrader `0` — **последний (текущий)**, `-1` — **предпоследний**.

---

## 6. Встроенные индикаторы `bt.indicators`

```python
# Простая скользящая средняя
self.sma = bt.indicators.SimpleMovingAverage(self.datas[0], period=15)

# Экспоненциальная скользящая средняя
self.ema = bt.indicators.ExponentialMovingAverage(self.datas[0], period=15)

# MACD
self.macd = bt.indicators.MACD(self.datas[0])

# RSI
self.rsi = bt.indicators.RSI(self.datas[0])

# Адаптивная скользящая средняя
self.ama = bt.indicators.AdaptiveMovingAverage(self.datas[0])

# ADX
self.adx = bt.indicators.ADX(self.datas[0])
```

> Полный список индикаторов — в автодополнении: `bt.indicators.<Tab>`

---

## 7. Типы ордеров

```python
# Рыночный ордер на покупку (исполняется на открытии следующего бара)
self.order = self.buy()

# Рыночный ордер на продажу
self.order = self.sell()

# Закрыть текущую позицию
self.close()
```

> Ордер всегда исполняется на **открытии следующего бара** после сигнала.

---

## 8. Добавление стратегии и запуск

```python
if __name__ == '__main__':
    cerebro = bt.Cerebro()

    # Загрузка данных
    data_feed = bt.feeds.PandasData(dataname=data)
    cerebro.adddata(data_feed)

    # Добавление стратегии
    cerebro.addstrategy(TestStrategy)

    # Настройки
    cerebro.broker.setcash(100_000.0)
    cerebro.broker.setcommission(commission=0.001)
    cerebro.addsizer(bt.sizers.FixedSize, stake=10)

    # Добавление анализатора для отчёта
    cerebro.addanalyzer(bt.analyzers.PyFolio, _name='PyFolio')

    print('Стартовый портфель: %.2f' % cerebro.broker.getvalue())
    results = cerebro.run()
    print('Конечный портфель: %.2f' % cerebro.broker.getvalue())
```

---

## 9. Отчёт через QuantStats

```python
import warnings
import quantstats

warnings.filterwarnings('ignore')

# Получить данные из результатов
strat = results[0]
portfolio_stats = strat.analyzers.getbyname('PyFolio')
returns, positions, transactions, gross_lev = portfolio_stats.get_pf_items()
returns.index = returns.index.tz_convert(None)

# Сгенерировать HTML-отчёт
quantstats.reports.html(returns, output='PerformanceStrat.html', title='Анализ по стратегии')
```

**Метрики в отчёте:**

| Метрика | Описание |
|---------|----------|
| Time in Market | % времени в активной позиции |
| Cumulative Return | Совокупная доходность |
| CAGR | Среднегодовая доходность |
| Sharpe Ratio | Коэффициент Шарпа (доходность/риск) |
| Max Drawdown | Максимальная просадка |
| Volatility | Волатильность |
| Calmar Ratio | Доходность / макс. просадка |
| Strategy Monthly Returns | График помесячной доходности |

> **Критерий хорошей стратегии по графику Monthly Returns:** цвета должны быть нейтральными и зелёными, без резких красных пятен.

---

## 10. Параметры стратегии (для оптимизации Optuna)

```python
class MyStrategy(bt.Strategy):
    params = (
        ('exitbars', 5),    # выход через N баров
        ('ma_period', 15),  # период SMA/EMA
        # ... другие параметры
    )
```

> Все параметры, которые нужно оптимизировать, **обязательно** выносить в `params`. Optuna будет перебирать их значения автоматически.

---

## 11. Шаблон минимального рабочего бота

```python
import backtrader as bt
import pandas as pd
from datetime import datetime


class MyStrategy(bt.Strategy):
    params = (('ma_period', 20),)

    def __init__(self):
        self.dataclose = self.datas[0].close
        self.order = None
        self.sma = bt.indicators.SimpleMovingAverage(
            self.datas[0], period=self.params.ma_period
        )

    def notify_order(self, order):
        if order.status in [order.Submitted, order.Accepted]:
            return
        if order.status == order.Completed:
            action = 'Покупка' if order.isbuy() else 'Продажа'
            print(f'{action}: {order.executed.price:.2f}')
        self.order = None

    def next(self):
        if self.order:
            return
        if not self.position:
            if self.dataclose[0] > self.sma[0]:
                self.order = self.buy()
        else:
            if self.dataclose[0] < self.sma[0]:
                self.order = self.sell()


if __name__ == '__main__':
    cerebro = bt.Cerebro()

    # --- Загрузка данных (подставьте свой DataFrame) ---
    # data['datetime'] = pd.to_datetime(data['datetime'])
    # data.set_index('datetime', inplace=True)
    # cerebro.adddata(bt.feeds.PandasData(dataname=data))

    cerebro.addstrategy(MyStrategy)
    cerebro.broker.setcash(100_000.0)
    cerebro.broker.setcommission(commission=0.001)
    cerebro.addsizer(bt.sizers.FixedSize, stake=10)

    print('Старт: %.2f' % cerebro.broker.getvalue())
    cerebro.run()
    print('Финиш: %.2f' % cerebro.broker.getvalue())
```

---

## 12. Ключевые методы стратегии — шпаргалка

| Метод | Когда вызывается |
|-------|-----------------|
| `__init__` | Один раз при старте — инициализация индикаторов |
| `next` | На каждом баре — **основная логика** |
| `notify_order` | При изменении статуса ордера |
| `notify_trade` | При закрытии сделки (круга buy→sell) |
| `start` | Перед первым баром |
| `stop` | После последнего бара |

---

## 13. Что менять при создании новой стратегии

1. **`__init__`** — добавить нужные индикаторы
2. **`next`** — написать логику входа (buy) и выхода (sell)
3. **`params`** — вынести все настраиваемые параметры
4. **`notify_order` и `notify_trade`** — оставить как есть (шаблон универсальный)
