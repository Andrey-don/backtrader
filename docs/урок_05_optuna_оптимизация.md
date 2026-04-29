# Урок 05: Optuna — оптимизация стратегий Backtrader

## 1. Краткое резюме

Урок посвящён интеграции библиотеки Optuna для автоматического подбора оптимальных параметров торговой стратегии в Backtrader. Рассматривается создание целевой функции (`objective`), внутри которой инициализируется и прогоняется движок `cerebro` с параметрами, генерируемыми «на лету». Показано, как задавать диапазоны поиска для целых и дробных чисел, запускать процесс перебора (trials) с целью максимизации прибыли и извлекать лучшие настройки для итогового прогона стратегии.

---

## 2. Ключевые концепции

- **Optuna** — библиотека для оптимизации гиперпараметров, поддерживает алгоритмы от брутфорса до древовидных (TPE)
- **Функция Objective** — обёртка, принимающая объект `trial`, инициализирующая `bt.Cerebro()` и возвращающая итоговый баланс портфеля (`cerebro.broker.getvalue()`) для оценки алгоритмом
- **Пространство поиска (Search Space)** — диапазоны значений: `suggest_int` для периодов, `suggest_float` для процентов (тейк-профит, стоп-лосс)
- **Триалы (Trials)** — количество испытаний (прогонов стратегии). Алгоритм анализирует результаты прошлых триалов и улучшает параметры в следующих. Для 4–5 параметров оптимально ~200 триалов
- **Study** — основной объект Optuna, хранящий историю испытаний. Направление поиска задаётся через `direction='maximize'`

---

## 3. Код урока

```python
import backtrader as bt
import optuna
import warnings

warnings.filterwarnings('ignore')


class MyStrategy(bt.Strategy):
    params = (
        ('short_ma', 20),
        ('long_ma', 50),
        ('take_profit', 0.02),
        ('stop_loss', 0.01),
    )

    def __init__(self):
        self.sma_short = bt.indicators.SMA(self.datas[0], period=self.params.short_ma)
        self.sma_long = bt.indicators.SMA(self.datas[0], period=self.params.long_ma)
        self.order = None

    def next(self):
        # Логика стратегии с использованием self.params.take_profit и self.params.stop_loss
        pass


def objective(trial):
    """Целевая функция для Optuna"""
    # 1. Пространство поиска параметров
    short_ma = trial.suggest_int('short_ma', 9, 25)
    long_ma = trial.suggest_int('long_ma', 33, 65)
    take_profit = trial.suggest_float('take_profit', 0.01, 0.05)
    stop_loss = trial.suggest_float('stop_loss', 0.005, 0.01)

    # 2. Инициализация cerebro внутри функции
    cerebro = bt.Cerebro()

    # data_feed = bt.feeds.PandasData(dataname=df.copy())
    # cerebro.adddata(data_feed)

    # 3. Стратегия с динамическими параметрами от Optuna
    cerebro.addstrategy(
        MyStrategy,
        short_ma=short_ma,
        long_ma=long_ma,
        take_profit=take_profit,
        stop_loss=stop_loss
    )

    # 4. Настройки брокера
    cerebro.broker.setcash(100_000.0)
    cerebro.addsizer(bt.sizers.PercentSizer, percents=50)
    cerebro.broker.setcommission(commission=0.001)

    # 5. Запуск и возврат результата
    cerebro.run()
    return cerebro.broker.getvalue()


if __name__ == '__main__':
    # 1. Создаём study с целью максимизации
    study = optuna.create_study(direction='maximize')

    # 2. Запускаем перебор (рекомендуется 200–500 триалов)
    study.optimize(objective, n_trials=100)

    print('Лучшие параметры:', study.best_trial.params)

    # 3. Финальный запуск с лучшими параметрами
    best = study.best_trial.params

    cerebro_final = bt.Cerebro()
    # cerebro_final.adddata(data_feed)

    cerebro_final.addstrategy(
        MyStrategy,
        short_ma=best['short_ma'],
        long_ma=best['long_ma'],
        take_profit=best['take_profit'],
        stop_loss=best['stop_loss']
    )

    cerebro_final.broker.setcash(100_000.0)
    cerebro_final.addsizer(bt.sizers.PercentSizer, percents=50)
    cerebro_final.broker.setcommission(commission=0.001)

    cerebro_final.run()
    print('Финальный портфель после оптимизации:', cerebro_final.broker.getvalue())
```

---

## 4. Важные замечания

- **Эффект оптимизации** — стратегия на случайных параметрах показывала убыток (99 000), после 100 триалов Optuna подняла результат до 111 625
- **Количество триалов** — для 4–5 параметров достаточно 200 триалов; для уверенности можно 500–700
- **`suggest_uniform` устарел** — в новых версиях Optuna используй `suggest_float` вместо `suggest_uniform`
- **Словарь результатов** — `study.best_trial.params` возвращает словарь, ключи которого совпадают с именами в `suggest_*`

---

## 5. Шпаргалка

| Действие | Код |
|----------|-----|
| Создать study с максимизацией | `study = optuna.create_study(direction='maximize')` |
| Запустить перебор | `study.optimize(objective, n_trials=100)` |
| Параметр int | `trial.suggest_int('name', min_val, max_val)` |
| Параметр float | `trial.suggest_float('name', min_val, max_val)` |
| Лучшие параметры (словарь) | `study.best_trial.params` |
| Извлечь конкретный параметр | `study.best_trial.params['take_profit']` |

---

## 6. Применение в боте (T-Invest API)

- **Оффлайн-процесс** — Optuna запускается до старта бота, на исторических данных (например, последние 6 месяцев из T-Invest)
- **Результат — конфиг бота** — найденный словарь параметров (`short_ma: 9, long_ma: 35, take_profit: 0.036`) закладывается в `.env` или конфигурационный файл live-бота
- **Переоптимизация** — рынок меняется; рекомендуется перезапускать Optuna раз в 1–2 месяца на свежих данных и обновлять параметры бота
