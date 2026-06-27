# Micro-futures Tools — AMarkets MT4

Калькулятор для торгівлі на AMarkets MT4 ECN рахунку.

**Live:** https://1artifintel-art.github.io/micro-futures-tools/calculator.html

---

## Структура сторінки

Сторінка складається з двох секцій:

1. **Position Size Calculator** — розрахунок розміру позиції
2. **Розрахунок витрат** — двоколонковий layout (витрати зліва, свопи справа)

---

## Секція 1 — Position Size Calculator

### Вхідні дані
| ID | Поле | Опис |
|----|------|------|
| `sym` | Символ | Select: EURUSD, GBPUSD, AUDUSD, NZDUSD, USDCAD, USDCHF, USDJPY, XAUUSD, XAGUSD, WTI, NGAS, SP500 |
| `maxloss` | Макс. збиток ($) | Default: 500 |
| `slpips` | Стоп-лос (пунктів) | Default: 50 |
| `entry` | Ціна входу | Автозаповнюється при зміні символу |
| `tp` | Тейк-профіт | Автозаповнюється при зміні символу |
| `dir` | Напрямок | buy / sell |
| `bal` | Баланс рахунку ($) | Default: 5000 |

### Розрахунок (`calc()`)
```
lots = maxloss / (slpips × pipVal)
lotsR = round(lots, 2)
loss = lotsR × slpips × pipVal
profit = lotsR × tpPips × pipVal
rr = profit / loss
slPrice = entry ± slpips × pipSize
pct = loss / bal × 100
```

### Результат
- Об'єм (лоти), Макс. збиток, Потенц. профіт, R:R, Ціна стоп-лосу, Ризик від балансу

---

## Секція 2 — Розрахунок витрат

### Layout
- **Ліва колонка**: Параметри позиції → Дати позиції → Витрати → Спред
- **Права колонка**: Таблиця Свопи · ECN · 1 лот

Дані в ліву колонку підтягуються автоматично з першої секції через `syncToCosts()`.

---

## SPECS — специфікації інструментів

```js
const SPECS = {
  EURUSD:{digits:5, pipSize:0.0001, pipVal:10,  type:"Forex", commRT:5, swapLval:-10.00, swapSval:2.00},
  GBPUSD:{digits:5, pipSize:0.0001, pipVal:10,  type:"Forex", commRT:5, swapLval:-4.00,  swapSval:-5.00},
  AUDUSD:{digits:5, pipSize:0.0001, pipVal:10,  type:"Forex", commRT:5, swapLval:-3.00,  swapSval:-5.00},
  NZDUSD:{digits:5, pipSize:0.0001, pipVal:10,  type:"Forex", commRT:5, swapLval:-5.00,  swapSval:0.00},
  USDCAD:{digits:5, pipSize:0.0001, pipVal:7,   type:"Forex", commRT:5, swapLval:2.00,   swapSval:-7.00},
  USDCHF:{digits:5, pipSize:0.0001, pipVal:12,  type:"Forex", commRT:5, swapLval:7.00,   swapSval:-19.00},
  USDJPY:{digits:3, pipSize:0.01,   pipVal:6,   type:"Forex", commRT:5, swapLval:6.00,   swapSval:-11.00},
  XAUUSD:{digits:2, pipSize:0.01,   pipVal:1,   type:"CFD",   commRT:7, swapLval:-66.00, swapSval:34.00},
  XAGUSD:{digits:3, pipSize:0.001,  pipVal:5,   type:"CFD",   commRT:7, swapLval:-195.00,swapSval:21.00},
  WTI:   {digits:2, pipSize:0.01,   pipVal:10,  type:"CFD",   commRT:5, swapLval:-4.00,  swapSval:-140.00},
  NGAS:  {digits:3, pipSize:0.001,  pipVal:30,  type:"CFD",   commRT:5, swapLval:-183.00,swapSval:-180.00},
  SP500: {digits:1, pipSize:0.1,    pipVal:1,   type:"CFD",   commRT:5, swapLval:-1.00,  swapSval:0.00},
};
```

- `pipVal` — вартість 1 піпа в $ на 1 лот (з калькулятора AMarkets)
- `commRT` — комісія ECN round-turn в $ на 1 лот ($5 Forex/CFD, $7 XAU/XAG)
- `swapLval` / `swapSval` — своп в $/лот/ніч (з калькулятора AMarkets, ECN, 27.06.2026)

---

## SWAP_PTS — пункти свопів з MT4

```js
const SWAP_PTS = {
  EURUSD:{L:-10.3, S:2.3},  GBPUSD:{L:-4.9,  S:-4.1},
  AUDUSD:{L:-3.2,  S:-4.6}, NZDUSD:{L:-5.2,  S:0.3},
  USDCAD:{L:2.2,   S:-10},  USDCHF:{L:5.7,   S:-15.3},
  USDJPY:{L:9.3,   S:-17.9},XAUUSD:{L:-65.5, S:34.2},
  XAGUSD:{L:-39,   S:4.3},  WTI:   {L:-0.4,  S:-14},
  NGAS:  {L:-6.1,  S:-6},   SP500: {L:-12.9, S:2.1},
};
```

Відображаються як редаговані input поля в таблиці. При зміні пунктів — автоматично перераховуються $ і оновлюється `SPECS[sym].swapLval/swapSval`.

**Потрійний своп:** Forex/метали — середа (dow=3), CFD (WTI/NGAS/SP500) — п'ятниця (dow=5).

---

## Блок Дати позиції

| ID | Поле |
|----|------|
| `c-date-open` | Дата відкриття |
| `c-date-close` | Дата закриття (план) |
| `c-nights` (hidden) | Кількість ночей (без вихідних) |
| `c-weekend` (hidden) | Кількість потрійних свопів |

`calcDates()` — рахує кількість торгових ночей між датами і кількість потрійних свопів.

```
totalNights = nights + tripleCount × 2
```

---

## Блок Витрати

### Формула
```
commTotal = commRT × lots
swapNight = (dir==="buy" ? swapLval : swapSval) × lots
swapTotal = swapNight × totalNights
totalCostSigned = swapTotal - commTotal
```

- `totalCostSigned` < 0 → витрата (червоний, "-$X")
- `totalCostSigned` > 0 → прибуток від свопу (зелений, "+$X")

### Відображення
| ID | Що показує |
|----|------------|
| `cc-comm` | Комісія ECN: `-$X.XX` |
| `cc-comm-rt` | `$5/лот` або `$7/лот` |
| `cc-swap` | Своп: `+$X.XX` або `-$X.XX` |
| `cc-swap-label` | `×N ніч` або `×N ніч (з потрійним)` |
| `cc-total` | Загальні витрати зі знаком |

---

## Блок Спред

| ID | Поле |
|----|------|
| `c-ask` | Ask |
| `c-bid` | Bid |
| `c-spread` (hidden) | Спред в піпах |

`calcSpread()`:
```
spreadPips = |ask - bid| / pipSize
spreadUSD = spreadPips × pipVal × lots
```

Після введення Ask/Bid з'являється підсекція:
- **Спред**: X.X піп
- **До беззбитку**: X.X піп · $X.XX (= беззбитковий рух від комісії/свопу + спред)

---

## Таблиця Свопи · ECN · 1 лот

Колонки: **Символ | Ком. | Long | Short | 3×**

- **Ком.** — commRT з SPECS ($5 або $7)
- **Long/Short** — два рядки: input пунктів (редагований) + $ значення
- **3×** — день потрійного свопу (Ср або Пт)

`onSwapPtsChange(sym, dir, pts)` — при зміні пунктів:
1. Перераховує $ через `ptVal = baseUSD / basePts`
2. Оновлює `SPECS[sym].swapLval/swapSval`
3. Оновлює `SWAP_PTS[sym]`
4. Викликає `calcCosts()`

---

## JS функції

| Функція | Коли викликається |
|---------|------------------|
| `calc()` | При зміні будь-якого поля секції 1 |
| `syncToCosts()` | Після `calc()` — синхронізує символ/лоти/напрямок в секцію 2 |
| `calcCosts()` | При зміні дат, свопів, будь-чого в секції 2 |
| `calcSpread()` | При зміні Ask або Bid |
| `calcDates()` | При зміні дат |
| `updateBreakevenWithSpread()` | В кінці `calcCosts()` та `calcSpread()` |
| `renderSwapTable()` | При ініціалізації |
| `onSwapPtsChange(sym,dir,pts)` | При зміні пунктів в таблиці |
| `onSym()` | При зміні символу |

---

## Комісія ECN AMarkets

- **$5 round-turn** на лот — Forex + WTI/NGAS/SP500
- **$7 round-turn** на лот — XAUUSD, XAGUSD
- Списується при закритті угоди одним записом

---

## Дані актуальні на

- Свопи: **27.06.2026** (з калькулятора AMarkets ECN)
- Комісії: постійні (з сайту AMarkets)
- Оновлення свопів: вручну через input поля в таблиці або через `SPECS`/`SWAP_PTS` в коді
