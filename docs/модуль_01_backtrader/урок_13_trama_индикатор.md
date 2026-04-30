# Урок 13: TRAMA — адаптивная скользящая средняя для Backtrader

## 1. Краткое резюме

Урок посвящён созданию кастомного индикатора TRAMA (Trend-Regularity Adaptive Moving Average) — адаптивной скользящей средней, которая подстраивает скорость под рыночные условия. Индикатор использует EMA от булевых флагов новых максимумов/минимумов для вычисления коэффициента сглаживания. Разбираются особенности реализации кастомных индикаторов с внутренним состоянием (формулы с рекурсией) в Backtrader. Данные — BTCUSD daily через `price_loaders.tradingview`.

---

## 2. Ключевые концепции

- **TRAMA** — адаптивная MA: `trama[i] = trama[i-1] + tc[i] * (close[i] - trama[i-1])`, где `tc = ema(hl)²`
- **`bt.If(condition, true_val, false_val)`** — условное выражение внутри индикатора (аналог `np.where`)
- **`bt.Or(a, b)`** — логическое ИЛИ для линий индикаторов
- **`self.high(-1)`** — значение линии `self.high` на предыдущем баре (сдвиг внутри индикатора)
- **Рекурсивный расчёт** — в `next` используем `self.lines.trama[0]` который зависит от предыдущего значения той же линии

---

## 3. Код урока

```python
import backtrader as bt
import pandas as pd


class TRAMA(bt.Indicator):
    lines = ('trama',)
    params = (
        ('prd', 3),  # период
    )

    def __init__(self):
        self.high = bt.indicators.Highest(self.data, period=self.p.prd)
        self.low = bt.indicators.Lowest(self.data, period=self.p.prd)

        # Флаг: новый максимум за период
        self.hh = bt.If(self.high - self.high(-1) > 0, 1.0, 0.0)
        # Флаг: новый минимум за период
        self.ll = bt.If(self.low - self.low(-1) < 0, 1.0, 0.0)

        # Объединяем флаги: был ли новый экстремум
        hl = bt.Or(self.hh, self.ll)
        # EMA от флагов — коэффициент адаптации
        ama = bt.indicators.EMA(hl, period=self.p.prd)
        # Коэффициент сглаживания (чем выше активность — тем быстрее MA)
        tc = ama * ama

        # Начальное значение
        self.lines.trama = self.data.close(0)

        # Рекурсивный расчёт TRAMA
        for i in range(self.p.prd, len(self.data)):
            if i == self.p.prd:
                self.lines.trama[i] = self.data.close[i]
            else:
                self.lines.trama[i] = (self.lines.trama[i-1] +
                                       tc[i] * (self.data.close[i] - self.lines.trama[i-1]))


class NoStrategy(bt.Strategy):
    """Стратегия только для отображения значений TRAMA"""

    def log(self, txt, dt=None):
        dt = dt or self.datas[0].datetime.date(0)
        print('%s, %s' % (dt.isoformat(), txt))

    def __init__(self):
        self.trama = TRAMA(self.datas[0])

    def next(self):
        self.log(f"TRAMA {self.trama[0]:.4f}")


if __name__ == '__main__':
    # Загрузка BTCUSD daily через price_loaders.tradingview
    # df_btc = load_asset_price("BTCUSD", 1500, "1D", None)
    # df_btc['time'] = pd.to_datetime(df_btc['time'])
    # df_btc.rename(columns={'time': 'datetime'}, inplace=True)
    # df_btc.set_index('datetime', inplace=True)

    cerebro = bt.Cerebro()
    cerebro.addstrategy(NoStrategy)

    # data_feed = bt.feeds.PandasData(dataname=df_btc)
    # cerebro.adddata(data_feed)

    cerebro.run()
```

---

## 4. Важные замечания

- **`bt.If` вместо `if`** — внутри `__init__` индикатора нельзя использовать обычные `if` для линий. Используем `bt.If(cond, val_true, val_false)`
- **`self.high(-1)`** — сдвиг на 1 бар назад для линии индикатора (аналог `[-1]` в стратегии)
- **Рекурсия в `next`** — TRAMA зависит от своего предыдущего значения. Это нормально для адаптивных MA, но требует осторожности при первоначальной инициализации
- **`prd=3`** — короткий период, быстрая адаптация. Для дневных данных обычно используют `prd=40-60`

---

## 5. Шпаргалка

| Действие | Код |
|----------|-----|
| Условное выражение | `bt.If(condition, true_val, false_val)` |
| Логическое ИЛИ | `bt.Or(line_a, line_b)` |
| Сдвиг линии на 1 бар | `self.high(-1)` |
| Максимум за N баров | `bt.indicators.Highest(self.data, period=N)` |
| EMA от кастомной линии | `bt.indicators.EMA(hl, period=N)` |

---

## 6. Применение в боте (T-Invest API)

- **TRAMA как фильтр тренда** — заменяет обычную SMA. При `close > trama` — восходящий тренд → покупки, при `close < trama` — нисходящий → продажи или пауза
- **Адаптивность** — TRAMA медленнее реагирует в боковике (меньше ложных сигналов) и быстрее в тренде. Это снижает количество ненужных ордеров в боте
- **Расчёт при новой свече** — при получении свечи от T-Invest обновляем буфер данных и пересчитываем TRAMA по той же формуле
