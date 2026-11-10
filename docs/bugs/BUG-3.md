# BUG-3: абоненту на електровелосипеді нараховано 45 безкоштовних хвилин, хоча специфікація T4 їх забороняє (T4)

| Поле | Значення |
|---|---|
| **Правило специфікації** | T4: «Електровелосипед: знижка 20 % на погодинне нарахування за T3. Безкоштовних хвилин на електровелосипеді немає.» (`docs/spec/tariffs.md`, рядки 76–77) |
| **Середовище** | коміт `4606c72` (до виправлення), Node `v26.3.0`, Windows (PowerShell) |
| **Серйозність** | висока — кожна поїздка абонента на електровелосипеді недораховується (знімається 45 хв з рахунку); недоплата до 64 грн за поїздку: різниця в блоках не перевищує 2, тобто 2 × 4000 kop × 0,8 = 6 400 kop (для 75 хв: 96 − 32 = 64 грн) |
| **Частота** | завжди — для будь-якої поїздки електровелосипедом абонента |
| **Знайдено** | `app/test/spec/tariff.test.js` › `T4: абонент, електровелосипед 75 хв → 3 блоки 120 грн, −20% → 96 грн (TC-13)`; прогін `ai-runs/b2-run.txt` |
| **Виправлення** | `3f92d26` — `fix(T4): no free minutes for e-bike subscribers (20% off full T3 charge)` |

## Кроки відтворення

1. Запустити команду з кореня репо (виправлений код):
   ```bash
   node -e "import('./app/src/tariff.js').then(({quote}) => console.log(JSON.stringify(quote({pickupAt:'2026-09-16T10:00:00Z',returnAt:'2026-09-16T11:15:00Z',bike:'ebike',returnStation:'S01'},{type:'subscriber'}),null,2)))"
   ```
2. Запустити ту ж саму команду на початковому коді з `main`:
   ```bash
   node -e "import('../ws12-main/app/src/tariff.js').then(({quote}) => console.log(JSON.stringify(quote({pickupAt:'2026-09-16T10:00:00Z',returnAt:'2026-09-16T11:15:00Z',bike:'ebike',returnStation:'S01'},{type:'subscriber'}),null,2)))"
   ```

## Очікуваний результат (за специфікацією)

Абонент, електровелосипед, 75 хв: безкоштовних хвилин немає → `ceil(75/30) = 3` блоки → 3 × 40 = 120 грн → знижка 20 % → 120 × 0.8 = 96 грн = 9600 kop.

## Фактичний результат (до виправлення, код з main)

```json
{
  "minutes": 75,
  "timeCharge": 12000,
  "discounts": [
    {
      "code": "subscription",
      "amount": 8800
    }
  ],
  "fees": [],
  "total": 3200
}
```

`total = 3200` kop (32 грн) замість 9600 kop (96 грн) — код віднімає 45 безкоштовних хвилин для електровелосипеда, що суперечить T4.

## Доказ

```bash
# Виправлений код (поточний, коміт 3f92d26):
node -e "import('./app/src/tariff.js').then(({quote}) => console.log(JSON.stringify(quote({pickupAt:'2026-09-16T10:00:00Z',returnAt:'2026-09-16T11:15:00Z',bike:'ebike',returnStation:'S01'},{type:'subscriber'}),null,2)))"
```
```json
{
  "minutes": 75,
  "timeCharge": 12000,
  "discounts": [
    {
      "code": "subscription",
      "amount": 2400
    }
  ],
  "fees": [],
  "total": 9600
}
```

```bash
# Початковий код з main (до виправлення):
node -e "import('../ws12-main/app/src/tariff.js').then(({quote}) => console.log(JSON.stringify(quote({pickupAt:'2026-09-16T10:00:00Z',returnAt:'2026-09-16T11:15:00Z',bike:'ebike',returnStation:'S01'},{type:'subscriber'}),null,2)))"
```
```json
{
  "minutes": 75,
  "timeCharge": 12000,
  "discounts": [
    {
      "code": "subscription",
      "amount": 8800
    }
  ],
  "fees": [],
  "total": 3200
}
```

## Мінімальний вхід

Абонент, електровелосипед, будь-яка тривалість (наприклад 75 хв). Якщо прибрати обчислення `Math.max(0, minutes - SUBSCRIPTION_FREE_MINUTES)` для електровелосипеда у `subscriptionCharge` (`app/src/tariff.js:84`), дефект зникає.

## Додатково

Хибне місце (гіпотеза): `app/src/tariff.js:84` — `subscriptionCharge` віднімає `SUBSCRIPTION_FREE_MINUTES` для всіх видів велосипедів, але за T4 для електровелосипеда безкоштовних хвилин немає. Пов'язані тести: TC-13; наявний тест `app/test/unit/tariff.test.js` («subscription on a long e-bike ride») стверджує лише `total < timeCharge`, не перевіряючи точну суму.
