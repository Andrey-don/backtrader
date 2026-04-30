# Модуль 04: Торговый робот T-Invest API (Т-банк / Тинькофф)

## 1. Краткое резюме

Архитектура live-бота на T-Invest API — модульная система: хранение учётных данных (`credentials`), утилиты для загрузки и маппинга исторических данных (свечей), модули проверки состояния счёта (баланс, маржа, позиции) и основные исполняющие скрипты (`bot_trading_market.py` и `bot_trading_limit.py`). Бот работает в бесконечном цикле `while True`: загружает новые данные, обрабатывает через торговую логику (скользящие средние), по сигналу выставляет рыночный или лимитный ордер, логирует все действия.

---

## 2. Ключевые концепции

- **Токен** — ключ доступа к API Т-банка. Выпускается в личном кабинете (раздел "Токены TBank Invest API"), прописывается в `credentials.py`.
- **Аккаунт (Account)** — торговый счёт. Может быть несколько (акции, фьючерсы). У каждого свой `ACCOUNT_ID` — нужен при выставлении заявок и проверке баланса.
- **Заявка (Ордер)** — команда на сделку. Два типа: маркет (по рыночной цене) и лимитный (по заданной цене).
- **Свечи (Bars)** — исторические OHLCV данные. Доступны интервалы: 1m, 5m, 15m, 1h, 1d.
- **FIGI** — уникальный идентификатор инструмента в системе T-Invest. Нужен для всех операций с конкретной акцией или фьючерсом.
- **Sandbox** — тестовый контур API с виртуальным балансом без риска реальных денег. Отдельный токен.
- **Стакан (Orderbook)** — таблица текущих лимитных заявок на покупку/продажу от всех участников.

---

## 3. Код

### Подключение к API

```python
# creds/credentials.py
TINKOFF_TOKEN = "ваш_токен"
TINKOFF_ACCOUNT_ID = "ваш_account_id"

# creds/accounts.py — получить список счетов
from tinkoff.invest import Client
TOKEN = "ваш_токен"

with Client(TOKEN) as client:
    accounts = client.users.get_accounts()
    for account in accounts.accounts:
        print(f"Account ID: {account.id}, Type: {account.type}, Name: {account.name}")
```

### Получение данных (свечи, портфель, позиции)

```python
# Получение FIGI по тикеру
def get_figi_by_ticker(client, ticker, class_code):
    all_sources = [
        client.instruments.shares().instruments,
        client.instruments.futures().instruments,
        client.instruments.etfs().instruments,
    ]
    for instruments in all_sources:
        for inst in instruments:
            if inst.ticker == ticker and inst.class_code == class_code:
                return inst.figi
    raise ValueError(f"FIGI не найден для {ticker} в классе {class_code}")

# Проверка открытой позиции по инструменту
def get_open_figi_position(client, figi):
    portfolio = client.operations.get_portfolio(account_id=TINKOFF_ACCOUNT_ID)
    for pos in portfolio.positions:
        if pos.figi == figi:
            return pos.quantity.units + pos.quantity.nano / 1e9
    return 0

# Получение доступных средств
def calculate_max_lots_to_buy(client, figi, class_code, currency="rub"):
    limits = client.operations.get_withdraw_limits(account_id=TINKOFF_ACCOUNT_ID)
    available = 0
    for m in limits.money:
        if m.currency.lower() == currency.lower():
            available = m.units + m.nano / 1e9
    return available
```

### Выставление заявок

```python
# Отмена старых лимитных ордеров перед выставлением нового
def cancel_existing_limit_orders(client, figi, direction):
    try:
        orders = client.orders.get_orders(account_id=TINKOFF_ACCOUNT_ID).orders
        for order in orders:
            if (order.figi == figi
                    and order.direction == direction
                    and order.order_type == OrderType.ORDER_TYPE_LIMIT):
                print(f"⚠️ Отменяем старый лимитный ордер: {order.order_id}")
                client.orders.cancel_order(
                    account_id=TINKOFF_ACCOUNT_ID,
                    order_id=order.order_id
                )
    except Exception as e:
        print(f"❌ Ошибка при отмене ордеров: {e}")

# Получение шага цены для лимитного ордера
def place_limit_order(client, figi, quantity, target_price, direction):
    try:
        futures = client.instruments.futures().instruments
        min_price_increment = 0.01
        for fut in futures:
            if fut.figi == figi:
                min_price_increment = (fut.min_price_increment.units
                                       + fut.min_price_increment.nano / 1e9)
                break
        # далее: client.orders.post_order(...)
    except Exception as e:
        print(f"❌ Ошибка при выставлении лимитного ордера: {e}")
```

### Основной цикл бота

```python
def trade(ticker, class_code):
    print(f"\n[{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}] Старт торговли по {ticker}")
    logging.info(f"Старт торговли по {ticker}")

    df = fetch_and_save_candles_by_ticker(ticker, class_code, interval_str="5min", days=2)
    signal = run_strategy(df)
    # далее: логика открытия/закрытия позиции по сигналу

if __name__ == "__main__":
    import time
    while True:
        trade("CRM5", "SPBFUT")    # фьючерс
        # trade("SBER", "TQBR")   # акция
        time.sleep(40)             # пауза для соблюдения rate limits
```

---

## 4. Важные замечания

- **Rate limits** — использовать `time.sleep(40)` или `schedule`. Без паузы API заблокирует IP.
- **Try/Except везде** — поле `initial_margin_on_buy` у фьючерсов может отсутствовать → падение без обработки ошибки.
- **Лоты vs акции** — для акций 1 лот = 10 акций (умножать). Для фьючерсов множитель = 1. Иначе ошибка "недостаточно средств".
- **Sandbox vs реал** — разные токены. Ордера в sandbox не попадают в реальный стакан.
- **FIGI обязателен** — все операции через FIGI, не через тикер напрямую.

---

## 5. Шпаргалка

| Действие | Метод API / Функция |
|---|---|
| Получить список счетов | `client.users.get_accounts()` |
| Доступные средства | `client.operations.get_withdraw_limits(account_id=...)` |
| Открытые позиции | `client.operations.get_portfolio(account_id=...)` |
| Найти акцию (FIGI) | `client.instruments.shares().instruments` |
| Найти фьючерс (FIGI) | `client.instruments.futures().instruments` |
| Скачать свечи | `fetch_and_save_candles_by_ticker()` (кастомная утилита) |
| Активные ордера | `client.orders.get_orders(account_id=...)` |
| Отменить ордер | `client.orders.cancel_order(account_id=..., order_id=...)` |

---

## 6. Архитектура production-бота

```
Исторические данные (MOEXALGO / T-Invest)
         ↓
  Бэктест (Backtrader) — проверка стратегии
         ↓
  ML-модель (CatBoost / XGBoost) — сигнал BUY / SELL / HOLD
         ↓
  run_strategy(df) — интеграция сигнала в цикл бота
         ↓
  T-Invest API — выставление маркет/лимитного ордера
         ↓
  while True + time.sleep(40) — автономная работа
```

**Папки проекта Tink_Vesper:**
```
creds/          — токен и account_id
bots/           — bot_trading_market.py, bot_trading_limit.py
strats/         — simple_strat.py (логика стратегии)
utils/          — bars.py, data_loader.py, account_money.py,
                  account_positions.py, futures_selector.py
utils/datasets/ — CSV файлы: SBER, GAZP, CRM5, RUM5 (5m, 15m, 1h)
```
