# Урок 01: Вводная информация по Backtrader

## 1. Краткое резюме

Урок охватывает базовую установку и настройку библиотеки backtrader для тестирования торговых стратегий на акциях, фьючерсах или криптовалютах. Рассматривается полный цикл: инициализация движка cerebro, загрузка котировок из pandas (на примере акций «Аэрофлот»), создание собственной стратегии (`bt.Strategy`) с логированием и использованием индикатора SMA. Также разбирается обработка ордеров, учёт брокерской комиссии и формирование детализированного HTML-отчёта с помощью библиотеки quantstats.

---

## 2. Ключевые концепции

- **Cerebro** — основной движок платформы, который тестирует стратегию
- **Индексация цен** — внутри backtrader последняя актуальная цена имеет индекс `[0]`, предыдущая — `[-1]`
- **Логика бара (next)** — при переходе с одного бара на другой вызывается метод `next`, в котором описывается основная логика стратегии
- **notify_order** — метод-обработчик для отслеживания статусов ордеров (выполнение, отклонение, маржин-колл)
- **notify_trade** — метод для подсчёта прибыли за закрытый круг сделки (купили → продали)
- **Комиссии и сайзеры** — для реалистичности тестов устанавливается комиссия брокера и размер сделки
- **bt.indicators** — встроенный модуль: одной строкой инициализируются SMA, EMA, осцилляторы и другие инструменты
- **Анализаторы** — для статистики подключается `bt.analyzers.PyFolio`, данные передаются в quantstats

---

## 3. Код урока

```python
import backtrader as bt
import datetime
import warnings
import quantstats

warnings.filterwarnings('ignore')


class TestStrategy(bt.Strategy):
    params = (
        ('maperiod', 15),  # период скользящей средней
        ('exitbars', 5),   # выход из сделки через N баров
    )

    def log(self, txt, dt=None):
        """Логирование: печатает дату и сообщение"""
        dt = dt or self.datas[0].datetime.date(0)
        print('%s, %s' % (dt.isoformat(), txt))

    def __init__(self):
        self.dataclose = self.datas[0].close
        self.order = None
        self.buyprice = None
        self.buycomm = None
        self.bar_executed = 0

        # Индикатор SMA
        self.sma = bt.indicators.SimpleMovingAverage(
            self.datas[0], period=self.params.maperiod
        )

    def notify_order(self, order):
        """Вызывается при изменении статуса ордера"""
        if order.status in [order.Submitted, order.Accepted]:
            return  # ждём исполнения

        if order.status in [order.Completed]:
            if order.isbuy():
                self.log('Покупка выполнена: Цена: %.2f, Стоимость: %.2f, Комиссия: %.2f' %
                         (order.executed.price, order.executed.value, order.executed.comm))
                self.buyprice = order.executed.price
                self.buycomm = order.executed.comm
            else:
                self.log('Продажа выполнена: Цена: %.2f, Стоимость: %.2f, Комиссия: %.2f' %
                         (order.executed.price, order.executed.value, order.executed.comm))
            self.bar_executed = len(self)

        elif order.status in [order.Canceled, order.Margin, order.Rejected]:
            self.log('Ордер отклонён/отменён (маржа или другая причина)')

        self.order = None  # сбросить текущий ордер

    def notify_trade(self, trade):
        """Вызывается при закрытии сделки"""
        if not trade.isclosed:
            return
        self.log('Операционная прибыль: Общая %.2f, Чистая (с комиссией) %.2f' %
                 (trade.pnl, trade.pnlcomm))

    def next(self):
        """Основная логика стратегии — вызывается на каждом баре"""
        if self.order:
            return  # есть активный ордер — не посылать новый

        if not self.position:
            # --- ВХОД: цена закрытия выше SMA ---
            if self.dataclose[0] > self.sma[0]:
                self.log('Ордер на покупку создан')
                self.order = self.buy()
        else:
            # --- ВЫХОД: прошло N баров ИЛИ цена упала ниже SMA ---
            if len(self) >= (self.bar_executed + self.params.exitbars) or \
               self.dataclose[0] < self.sma[0]:
                self.log('Ордер на продажу создан')
                self.order = self.sell()


if __name__ == '__main__':
    cerebro = bt.Cerebro()

    # Загрузка данных из pandas DataFrame
    # data_copy = df.copy()
    # data_copy['datetime'] = pd.to_datetime(data_copy['datetime'])
    # data_copy.set_index('datetime', inplace=True)
    # data_feed = bt.feeds.PandasData(dataname=data_copy)
    # cerebro.adddata(data_feed)

    cerebro.addstrategy(TestStrategy)

    cerebro.broker.setcash(100_000.0)
    cerebro.addsizer(bt.sizers.FixedSize, stake=10)
    cerebro.broker.setcommission(commission=0.001)

    cerebro.addanalyzer(bt.analyzers.PyFolio, _name='pyfolio')

    print('Стартовый портфель: %.2f' % cerebro.broker.getvalue())
    results = cerebro.run()
    print('Конечный портфель: %.2f' % cerebro.broker.getvalue())

    # Формирование HTML-отчёта
    strat = results[0]
    returns, positions, transactions, gross_lev = strat.analyzers.pyfolio.get_pf_items()
    returns.index = returns.index.tz_convert(None)
    quantstats.reports.html(returns, output='reports/strategy.html', title='Анализ по стратегии')
```

---

## 4. Важные замечания

- **Исполнение ордеров** — сделка открывается не на баре сигнала, а на **открытии следующего бара**
- **self** — привязка атрибутов обязательно через `self` (например `self.order`). Без этого значение не изменится и логика сломается
- **Вложенность if/else** — в методе `next` строго следить за отступами: `else` относится только к своему `if`
- **Monthly Returns** — в HTML-отчёте график месячной доходности должен быть монолитным (нейтральные и зелёные цвета). Нестабильный — признак несбалансированной стратегии

---

## 5. Шпаргалка

| Действие | Код |
|----------|-----|
| Инициализация движка | `cerebro = bt.Cerebro()` |
| Проверка капитала | `cerebro.broker.getvalue()` |
| Установка депозита | `cerebro.broker.setcash(100000)` |
| Установка комиссии | `cerebro.broker.setcommission(commission=0.001)` |
| Размер позиции | `cerebro.addsizer(bt.sizers.FixedSize, stake=10)` |
| Загрузка DataFrame | `bt.feeds.PandasData(dataname=df)` |
| Добавление данных | `cerebro.adddata(data_feed)` |
| Добавление стратегии | `cerebro.addstrategy(TestStrategy)` |
| Запуск тестирования | `results = cerebro.run()` |
| Добавление анализатора | `cerebro.addanalyzer(bt.analyzers.PyFolio, _name='pyfolio')` |
| Генерация HTML-отчёта | `quantstats.reports.html(returns, output='strategy.html')` |

---

## 6. Применение в боте (T-Invest API)

Каркас `TestStrategy` с методами `notify_order`, `notify_trade` и `next` универсален — его можно напрямую адаптировать для live-торговли через T-Invest API.

При переходе к live-торговле особенно важен `notify_order`:
- **Защита от дублирования** — бот не отправит новый ордер, пока текущий в статусе `Submitted` или `Accepted`
- **Обработка отказов** — при `Rejected` или `Margin` бот корректно сбрасывает флаг и возобновляет работу
- **Логирование** — метод `log()` станет основой для мониторинга действий бота в реальном времени
