# FOR_CLAUDE — micro-futures-tools

Документація для Claude про калькулятор позицій та витрат на AMarkets MT4.

**URL:** https://1artifintel-art.github.io/micro-futures-tools/calculator.html  
**Репо:** https://github.com/1artifintel-art/micro-futures-tools  
**Детальна документація:** README.md в цьому репо

---

## СТРУКТУРА СТОРІНКИ

Одна HTML сторінка з двома секціями:

1. **Position Size Calculator** — розрахунок лотів, SL, профіт, R:R
2. **Розрахунок витрат** — двоколонковий layout

**Ліва колонка:** Параметри позиції → Дати позиції → Витрати → Спред  
**Права колонка:** Таблиця Свопи · ECN · 1 лот

---

## SPECS — дані інструментів (актуальні на 30.06.2026)

```js
EURUSD:{digits:5, pipSize:0.0001, pipVal:10,  tickVal:1,     commRT:5, swapLval:-8.80,  swapSval:2.10}
GBPUSD:{digits:5, pipSize:0.0001, pipVal:10,  tickVal:1,     commRT:5, swapLval:-4.30,  swapSval:-4.40}
AUDUSD:{digits:5, pipSize:0.0001, pipVal:10,  tickVal:1,     commRT:5, swapLval:-3.10,  swapSval:-4.70}
NZDUSD:{digits:5, pipSize:0.0001, pipVal:10,  tickVal:1,     commRT:5, swapLval:-5.00,  swapSval:0.50}
USDCAD:{digits:5, pipSize:0.0001, pipVal:7,   tickVal:1,     commRT:5, swapLval:2.20,   swapSval:-10.00}
USDCHF:{digits:5, pipSize:0.0001, pipVal:12,  tickVal:1,     commRT:5, swapLval:6.10,   swapSval:-15.40}
USDJPY:{digits:3, pipSize:0.01,   pipVal:6,   tickVal:0.645, commRT:5, swapLval:5.68,   swapSval:-11.22}
XAUUSD:{digits:2, pipSize:0.01,   pipVal:1,   tickVal:1,     commRT:5, swapLval:-63.50, swapSval:33.00}
XAGUSD:{digits:3, pipSize:0.001,  pipVal:5,   tickVal:5,     commRT:5, swapLval:-203.50,swapSval:25.00}
WTI:   {digits:2, pipSize:0.01,   pipVal:10,  tickVal:10,    commRT:5, swapLval:-4.00,  swapSval:-130.00}
NGAS:  {digits:3, pipSize:0.001,  pipVal:30,  tickVal:30,    commRT:5, swapLval:-183.00,swapSval:-180.00}
SP500: {digits:1, pipSize:0.1,    pipVal:1,   tickVal:0.1,   commRT:5, swapLval:-1.26,  swapSval:0.21}
```

- `pipVal` — $ за 1 піп на 1 лот
- `tickVal` — $ за 1 MT4 пункт на 1 лот = contract_size × point_size
- `commRT` — комісія round-turn $/лот ($5 для всіх інструментів, підтверджено на amarkets.com)
- `swapLval/swapSval` — $/лот/ніч = SWAP_PTS × tickVal

---

## SWAP_PTS — пункти свопів з MT4 (редаговані в таблиці, актуальні на 30.06.2026)

```js
EURUSD:{L:-8.8,  S:2.1},  GBPUSD:{L:-4.3,  S:-4.4},
AUDUSD:{L:-3.1,  S:-4.7}, NZDUSD:{L:-5.0,  S:0.5},
USDCAD:{L:2.2,   S:-10},  USDCHF:{L:6.1,   S:-15.4},
USDJPY:{L:8.8,   S:-17.4},XAUUSD:{L:-63.5, S:33},
XAGUSD:{L:-40.7, S:5},    WTI:   {L:-0.4,  S:-13},
NGAS:  {L:-6.1,  S:-6},   SP500: {L:-12.6, S:2.1}
```

Свопи змінюються щотижня — оновлюються вручну через таблицю на сторінці (inputs).  
Після зміни пунктів — автоматично перераховуються $ і зберігаються в localStorage.

---

## ФОРМУЛА СВОПУ

```
swap$ = pts × tickVal
```

де `tickVal = contract_size × point_size` — фіксований для кожного інструменту.

Виняток: **USDJPY** — `tickVal = 100000 × 0.001 / JPY_rate ≈ 0.645` (наближено, залежить від курсу).

---

## ФОРМУЛИ РОЗРАХУНКІВ

### Position Size Calculator
```
lots     = maxloss / (slpips × pipVal)
loss     = round(lots,2) × slpips × pipVal
profit   = round(lots,2) × tpPips × pipVal
rr       = profit / loss
slPrice  = entry ± slpips × pipSize
```

### Витрати
```
commTotal        = commRT × lots
swapNight        = (buy ? swapLval : swapSval) × lots
totalNights      = nights + tripleCount × 2
swapTotal        = swapNight × totalNights
totalCostSigned  = swapTotal - commTotal
```

- `totalCostSigned` < 0 → витрата (червоний)
- `totalCostSigned` > 0 → прибуток від свопу (зелений)

### Дати і потрійний своп
```
tripleDay = CFD(WTI/NGAS/SP500) ? 5(п'ятниця) : 3(середа)
Для кожного дня між відкриттям і закриттям:
  якщо не вихідний → nights++
  якщо dow === tripleDay → tripleCount++
totalNights = nights + tripleCount × 2
```

### Спред
```
spreadPips = |ask - bid| / pipSize
spreadUSD  = spreadPips × pipVal × lots
беззбиток  = (|totalCostSigned| / pipVal / lots) + spreadPips  піп
```

---

## ЗБЕРЕЖЕННЯ СВОПІВ (localStorage)

- При зміні пунктів в таблиці → `localStorage.setItem("swapPts", JSON.stringify(SWAP_PTS))`
- При завантаженні сторінки → читає `swapPts`, викликає `onSwapPtsChange` для кожного інструменту
- Долари перераховуються через `swap$ = pts × tickVal` (не через коефіцієнти від старих значень)

---

## JS ФУНКЦІЇ

| Функція | Тригер |
|---------|--------|
| `calc()` | Зміна будь-якого поля секції 1 |
| `syncToCosts()` | Після `calc()` |
| `calcCosts()` | Зміна дат, свопів, будь-чого в секції 2 |
| `calcSpread()` | Зміна Ask або Bid |
| `calcDates()` | Зміна дат |
| `updateBreakevenWithSpread()` | В кінці `calcSpread()` |
| `renderSwapTable()` | При ініціалізації |
| `onSwapPtsChange(sym,dir,pts)` | Зміна пунктів в таблиці |
| `onSym()` | Зміна символу |

---

## ВІДОМІ ОСОБЛИВОСТІ

- Свопи зберігаються в localStorage як пункти, $ — завжди похідні через `tickVal`
- USDJPY tickVal наближений (~0.645) — залежить від поточного курсу JPY
- Своп може бути позитивним (дохід) → `totalCostSigned` > 0, показується зеленим
- Спред показується окремо у блоці Спред, не входить у Загальні витрати
- Блок Спред активується тільки після введення Ask і Bid

*Оновлено: 30.06.2026*
