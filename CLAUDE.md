# Backtrader Trading Bot

Проект по разработке самообучающегося торгового бота на Python.
Бэктестинг через Backtrader, live-торговля через T-Invest API (Тинькофф).

## Цель проекта

Создать алготрейдинг-систему для MOEX (Московская биржа) с:
- бэктестингом стратегий на исторических данных
- оптимизацией параметров через Optuna
- ML-сигналами (scikit-learn / нейросети)
- live-торговлей через T-Invest API

## Технический стек

- Python 3.11+, venv в `.venv/`
- **Backtrader** — бэктестинг стратегий
- **QuantStats** — HTML-отчёты по стратегиям
- **Optuna** — оптимизация гиперпараметров
- **scikit-learn, torch** — машинное обучение
- **pandas, pandas-ta** — данные и технический анализ
- **tinkoff-investments** — T-Invest API (live-торговля)
- **python-dotenv** — управление секретами
- Секреты в `.env` (не коммитить): `TINKOFF_TOKEN`, `TINKOFF_SANDBOX`

## Структура проекта

```
backtrader/
├── src/
│   ├── data/           # загрузка и подготовка данных
│   │   ├── finam.py    # парсер Finam (без API)
│   │   └── tinvest.py  # T-Invest API — исторические данные
│   ├── strategies/     # стратегии Backtrader
│   │   ├── base.py     # базовый шаблон стратегии
│   │   ├── sma.py      # SMA-кроссовер
│   │   └── ml.py       # ML-стратегия
│   ├── optimization/   # оптимизация через Optuna
│   │   └── optimizer.py
│   ├── live/           # live-торговля
│   │   └── bot.py      # боевой цикл T-Invest
│   └── utils/          # утилиты
├── notebooks/          # Jupyter-ноутбуки для исследований
├── data/               # CSV/parquet с историческими данными (не в git)
├── reports/            # HTML-отчёты QuantStats (не в git)
├── docs/               # документация уроков курса
│   ├── урок_01_backtrader_вводное.md
│   ├── карта_курса.md
│   └── LLM_обработка_уроков.md
├── .env                # токены и настройки (не в git)
├── .env.example        # шаблон .env
├── requirements.txt    # зависимости
└── CLAUDE.md           # этот файл
```

## Быстрый старт

```bash
# Установка зависимостей
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt

# Настройка секретов
cp .env.example .env
# Заполнить .env своим токеном T-Invest

# Запуск бэктеста (пример)
python src/strategies/sma.py
```

## Шаблон стратегии Backtrader

```python
import backtrader as bt

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
            self.log(f'{action}: {order.executed.price:.2f}')
        self.order = None

    def notify_trade(self, trade):
        if not trade.isclosed:
            return
        self.log(f'P&L: {trade.pnlcomm:.2f}')

    def next(self):
        if self.order:
            return
        if not self.position:
            if self.dataclose[0] > self.sma[0]:
                self.order = self.buy()
        else:
            if self.dataclose[0] < self.sma[0]:
                self.order = self.sell()

    def log(self, txt):
        dt = self.datas[0].datetime.date(0)
        print(f'{dt.isoformat()}, {txt}')

if __name__ == '__main__':
    cerebro = bt.Cerebro()
    cerebro.addstrategy(MyStrategy)
    cerebro.broker.setcash(100_000.0)
    cerebro.broker.setcommission(commission=0.001)
    cerebro.addsizer(bt.sizers.FixedSize, stake=10)
    print(f'Старт: {cerebro.broker.getvalue():.2f}')
    cerebro.run()
    print(f'Финиш: {cerebro.broker.getvalue():.2f}')
```

## Ключевые правила Backtrader

- `self.dataclose[0]` — **текущий** бар (не последний как в pandas!)
- `self.dataclose[-1]` — **предыдущий** бар
- Ордер исполняется на **открытии следующего** бара после сигнала
- `self.order` — хранить текущий ордер, чтобы не дублировать
- Всегда выносить параметры в `params = ((...),)` — для Optuna
- `__init__` — только индикаторы; `next` — только логика входа/выхода

## T-Invest API — sandbox

```python
import os
from dotenv import load_dotenv
from tinkoff.invest import Client, sandbox

load_dotenv()
TOKEN = os.getenv("TINKOFF_TOKEN")
SANDBOX = os.getenv("TINKOFF_SANDBOX", "true").lower() == "true"

with Client(TOKEN) as client:
    # Получить счета в sandbox
    accounts = client.sandbox.get_sandbox_accounts()
```

## Отчёт QuantStats

```python
import warnings, quantstats
warnings.filterwarnings('ignore')

cerebro.addanalyzer(bt.analyzers.PyFolio, _name='PyFolio')
results = cerebro.run()

strat = results[0]
stats = strat.analyzers.getbyname('PyFolio')
returns, positions, transactions, gross_lev = stats.get_pf_items()
returns.index = returns.index.tz_convert(None)

quantstats.reports.html(returns, output='reports/strategy.html', title='Анализ стратегии')
```

## Рабочий процесс

1. Написать/изменить стратегию в `src/strategies/`
2. Запустить бэктест, получить отчёт `reports/*.html`
3. Оптимизировать параметры через Optuna
4. Подключить ML-сигналы
5. Протестировать в T-Invest sandbox
6. Коммит: `git add -A && git commit && git push origin main`

## Источники данных

| Источник | Когда использовать |
|----------|--------------------|
| Finam (парсер) | Бесплатно, MOEX, без API |
| T-Invest API | Реальные свечи MOEX, нужен токен |
| pandas-ta | Расчёт индикаторов на любых данных |

## Курс (материалы)

- **Диск E:** `[Арина Веспер] [Vesperfin] VesperfinCode. Модуль 2 Алготрейдинг Про\`
- **Обработанные уроки:** `docs/урок_*.md`
- **Карта курса:** `docs/карта_курса.md`
- **Код бота Т-Банк (оригинал):** `E:\...\04_Торговый робот\Т-банк (Тинькофф)\Tink_Vesper.rar`

## Важно

- Токен T-Invest **никогда** не коммитить — только в `.env`
- Sandbox-режим: `TINKOFF_SANDBOX=true`
- После каждого рабочего блока: коммит + push
- `.env` внесён в `.gitignore` и защищён в `settings.json`
