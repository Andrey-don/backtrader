# Модуль 05: Примеры стратегий для торговых роботов (ByBit + QUIK)

## 1. Краткое резюме

Данный модуль посвящен созданию торговых роботов для криптовалютной биржи ByBit и российского фондового/срочного рынка через терминал QUIK. В качестве основного стека используются Python, библиотека `pybit` (Unified Trading) и `pandas_ta` для вычисления индикаторов на ByBit, а также фреймворк `backtrader` с коннектором `BackTraderQuik` для алгоритмической торговли в QUIK.

---

## 2. Ключевые концепции

- **Закрытый бар (Closed Bar):** Свеча, которая полностью сформировалась на выбранном таймфрейме; торговля по закрытым барам исключает перерисовку сигналов внутри текущего периода.
- **Кроссовер (Crossover):** Сигнал, возникающий при пересечении быстрой и медленной скользящих средних (или индикатора и триггерного уровня), отслеживаемый через сравнение текущего `[-1]` и предыдущего `[-2]` значений.
- **Датафрейм (DataFrame):** Структура данных pandas, используемая для хранения исторических цен (klines) и вычисления индикаторов через `pandas_ta`.
- **Сайзер (Sizer):** Механизм в Backtrader, определяющий объем позиции. Для акций и фьючерсов в QUIK логика сайзеров отличается, что требует корректировки во избежание ошибок при закрытии позиций.

---

## 3. Стратегии ByBit (pandas_ta)

### upper_strat (Стратегия повышения цены)
**Логика:** Сравнивает текущую цену закрытия с предыдущей. Если прирост больше заданного порога, возвращает сигнал на покупку `up`.
**Параметры:** `threshold` (по умолчанию 0.003 или 0.3%)
**Особенности:** Простая импульсная стратегия. Расчёт: `(текущая - предыдущая) / предыдущая`

### down_strat (Стратегия понижения цены)
**Логика:** Возвращает сигнал на покупку при резком падении цены.
**Параметры:** `threshold` (по умолчанию 0.003)
**Особенности:** Эффективна на падающем рынке для ловли быстрых откатов с автоматическим закрытием по Take Profit.

### ema_strat (Стратегия EMA)
**Логика:** Пересечение двух экспоненциальных скользящих средних. Покупка при пересечении быстрой EMA медленной снизу вверх, продажа — сверху вниз.
**Параметры:** `short_period=10`, `long_period=40`

### range_strat (Стратегия SMA)
**Логика:** Пересечение текущей цены закрытия и простой скользящей средней (SMA).
**Параметры:** `long_period=120`

### rsi_strat (Стратегия RSI)
**Логика:** Сигнал `down` (продажа) при выходе RSI из зоны перекупленности, `up` (покупка) при выходе из перепроданности.
**Параметры:** `period=14`, `overbought=70`, `oversold=30`
**Особенности:** Для жёсткого отсева ложных сигналов рекомендуется тестировать параметры 80/20 или 85/15.

### bollinger_bands_strat (Стратегия Bollinger Bands)
**Логика:** Сигнал `up` — предыдущая цена была выше нижней границы (Lower Band), текущая стала ниже (пробой вниз). Сигнал `down` — обратный пробой.
**Параметры:** `period=20`, `std_dev=2`

### macd_rsi_strat (Стратегия MACD + RSI)
**Логика:** Покупка, если MACD пересекает сигнальную линию снизу вверх и RSI ниже перепроданности. Продажа при обратных условиях.
**Параметры:** MACD (12, 26, 9), RSI (14, 70, 30)

---

## 4. Стратегии QUIK (Backtrader)

### LimitCancel / LimitCancel_Fut
**Тип ордера:** Лимитный
**Логика:** Заявка на покупку на заданный процент (`limit_pct`) ниже цены закрытия. Если за 1 бар не исполняется — отменяется. Скрипты разделены для акций и фьючерсов из-за разницы сайзера.

### Market_Fut
**Тип ордера:** Рыночный
**Логика:** Пробитие линий Bollinger Bands. Открывает длинную позицию (`self.buy()`), если цена ниже средней линии и пробивает нижнюю границу. При наличии позиции — ищет сигнал на продажу (`self.sell()`).

### Market_Fut_mod
**Тип ордера:** Рыночный
**Логика:** Пересечение короткой (10) и длинной (25) скользящих средних. Короткая выше длинной — покупка, ниже — продажа.
**Особенности:** Явная реализация TP/SL через статические параметры `take_profit=2%`, `stop_loss=1%`, отсчитываемые от `position.price`.

---

## 5. Код (ключевые фрагменты)

### Получение свечей (ByBit)
```python
def klines(symbol):
    try:
        resp = session.get_kline(
            category='linear', symbol=symbol,
            interval=timeframe, limit=500
        )['result']['list']
        resp = pd.DataFrame(resp)
        resp.columns = ['Time', 'Open', 'High', 'Low', 'Close', 'Volume', 'Turnover']
        resp = resp.set_index('Time')
        resp = resp.astype(float)
        resp = resp[::-1]  # разворот: последняя свеча имеет индекс [-1]
        return resp
    except Exception as err:
        print(err)
```

### Работа с закрытыми барами (ByBit)
```python
def klines(symbol, timeframe):
    """Получает бары для символа и таймфрейма, возвращая только закрытые бары."""
    # Сравнение времени закрытия свечи с datetime.now()
    # Если бар ещё не закрыт — iloc[-1] пропускается
```

### Размещение рыночного ордера (ByBit)
```python
def place_order_market(symbol, side):
    price_precision, qty_precision = get_precisions(symbol)
    mark_price = float(
        session.get_tickers(category='linear', symbol=symbol)
        ['result']['list'][0]['markPrice']
    )
    order_qty = round(qty / mark_price, qty_precision)

    if side == 'buy':
        tp_price = round(mark_price + mark_price * tp, price_precision)
        sl_price = round(mark_price - mark_price * sl, price_precision)
        session.place_order(
            category='linear', symbol=symbol, side='Buy',
            orderType='Market', qty=order_qty,
            takeProfit=tp_price, stopLoss=sl_price,
            tpTriggerBy='MarkPrice', slTriggerBy='MarkPrice'
        )
```

### Отмена ордеров (QUIK / Backtrader)
```python
def next(self):
    if not self.position and self.order and self.order.status == self.order.Accepted:
        self.cancel(self.order)  # снимает неисполненную заявку
```

### TP/SL на основе position.price (QUIK Market_Fut_mod)
```python
if self.position.size > 0:
    if self.data.close[0] >= self.position.price * (1 + self.p.take_profit):
        self.sell()
    elif self.data.close[0] <= self.position.price * (1 - self.p.stop_loss):
        self.sell()
```

### Главный цикл (ByBit)
```python
while True:
    balance = get_balance()
    if balance is not None:
        pos = get_positions()
        for elem in symbols:
            if elem in pos:
                continue
            signal = bollinger_bands_strat(elem)
            if signal == 'up':
                place_order_market(elem, 'buy')
            elif signal == 'down':
                place_order_market(elem, 'sell')
    sleep(10)
```

---

## 6. Важные замечания

- **Конфликт множественных стратегий (ByBit):** При использовании нескольких стратегий в цикле `while True` — использовать `continue` после исполнения ордера или строгую `if/elif` приоритизацию. Без этого стратегии могут дублировать ордера.
- **Ошибка копипасты в QUIK:** В `Market_Fut.py` и `Market_Fut_mod.py` класс называется `class LimitCancel(bt.Strategy)` — забыли переименовать при копировании. Исправить на актуальное название.
- **Разница сайзеров в QUIK:** Для акций и фьючерсов — разный `sizer`. Обязательно использовать файлы с постфиксом `_Fut` для срочного рынка, иначе ошибка при закрытии позиций.
- **Копирование DataFrame:** Использовать `df.copy()` перед манипуляциями с колонками для индикаторов — избегает `SettingWithCopyWarning` и неожиданного изменения исходных данных.

---

## 7. Шпаргалка

| Стратегия | Платформа | Индикаторы | Тип ордера | Особенность |
|---|---|---|---|---|
| upper_strat / down_strat | ByBit | Price Action (%) | Market | Ловля резких импульсов / откатов |
| ema_strat | ByBit | EMA (10, 40) | Market | Классический трендфолловинг |
| range_strat | ByBit | SMA (120) | Market | Пересечение цены и длинной SMA |
| rsi_strat | ByBit | RSI (14) | Market | Зоны перекупленности / перепроданности |
| bollinger_bands_strat | ByBit | Bollinger Bands | Market | Пробой верхней / нижней границы |
| macd_rsi_strat | ByBit | MACD + RSI | Market | MACD-сигнал фильтруется зонами RSI |
| LimitCancel_Fut | QUIK | Price Action | Limit | Заявка ниже закрытия, отмена через 1 бар |
| Market_Fut | QUIK | Bollinger Bands | Market | Торговля фьючерсами по полосам Боллинджера |
| Market_Fut_mod | QUIK | SMA (10, 25) | Market | TP/SL от position.price |
