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

## SPECS — дані інструментів (актуальні на 27.06.2026)

```js
EURUSD:{digits:5, pipSize:0.0001, pipVal:10,  commRT:5, swapLval:-10.00, swapSval:2.00}
GBPUSD:{digits:5, pipSize:0.0001, pipVal:10,  commRT:5, swapLval:-4.00,  swapSval:-5.00}
AUDUSD:{digits:5, pipSize:0.0001, pipVal:10,  commRT:5, swapLval:-3.00,  swapSval:-5.00}
NZDUSD:{digits:5, pipSize:0.0001, pipVal:10,  commRT:5, swapLval:-5.00,  swapSval:0.00}
USDCAD:{digits:5, pipSize:0.0001, pipVal:7,   commRT:5, swapLval:2.00,   swapSval:-7.00}
USDCHF:{digits:5, pipSize:0.0001, pipVal:12,  commRT:5, swapLval:7.00,   swapSval:-19.00}
USDJPY:{digits:3, pipSize:0.01,   pipVal:6,   commRT:5, swapLval:6.00,   swapSval:-11.00}
XAUUSD:{digits:2, pipSize:0.01,   pipVal:1,   commRT:5, swapLval:-66.00, swapSval:34.00}
XAGUSD:{digits:3, pipSize:0.001,  pipVal:5,   commRT:5, swapLval:-195.00,swapSval:21.00}
WTI:   {digits:2, pipSize:0.01,   pipVal:10,  commRT:5, swapLval:-4.00,  swapSval:-140.00}
NGAS:  {digits:3, pipSize:0.001,  pipVal:30,  commRT:5, swapLval:-183.00,swapSval:-180.00}
SP500: {digits:1, pipSize:0.1,    pipVal:1,   commRT:5, swapLval:-1.00,  swapSval:0.00}
```

- `pipVal` — $ за 1 піп на 1 лот (з калькулятора AMarkets ECN)
- `commRT` — комісія round-turn $/лот ($5 для всіх інструментів)
- `swapLval/swapSval` — $/лот/ніч (з калькулятора AMarkets ECN, 27.06.2026)

---

## SWAP_PTS — пункти свопів з MT4 (редаговані в таблиці)

```js
EURUSD:{L:-10.3, S:2.3},  GBPUSD:{L:-4.9,  S:-4.1},
AUDUSD:{L:-3.2,  S:-4.6}, NZDUSD:{L:-5.2,  S:0.3},
USDCAD:{L:2.2,   S:-10},  USDCHF:{L:5.7,   S:-15.3},
USDJPY:{L:9.3,   S:-17.9},XAUUSD:{L:-65.5, S:34.2},
XAGUSD:{L:-39,   S:4.3},  WTI:   {L:-0.4,  S:-14},
NGAS:  {L:-6.1,  S:-6},   SP500: {L:-12.9, S:2.1}
```

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
tripleDay = CFD(WTI/NGAS/SP500) ? 5(пятниця) : 3(середа)
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

### Таблиця свопів — зміна пунктів
```
ptVal   = baseUSD / basePts
newUSD  = newPts × ptVal
→ оновлює SPECS[sym].swapLval/swapSval і SWAP_PTS[sym]
→ викликає calcCosts()
```

---

## JS ФУНКЦІЇ

| Функція | Тригер |
|---------|--------|
| `calc()` | Зміна будь-якого поля секції 1 |
| `syncToCosts()` | Після `calc()` |
| `calcCosts()` | Зміна дат, свопів, будь-чого в секції 2 |
| `calcSpread()` | Зміна Ask або Bid |
| `calcDates()` | Зміна дат |
| `updateBreakevenWithSpread()` | В кінці `calcCosts()` і `calcSpread()` |
| `renderSwapTable()` | При ініціалізації |
| `onSwapPtsChange(sym,dir,pts)` | Зміна пунктів в таблиці |
| `onSym()` | Зміна символу |

---

## ВІДОМІ ОСОБЛИВОСТІ

- Свопи в таблиці редаговані — при оновленні пунктів перераховуються $ і оновлюється SPECS
- Своп може бути позитивним (дохід) — тоді `totalCostSigned` > 0, показується зеленим
- Спред не входить у "Загальні витрати" — показується окремо у блоці Спред як частина беззбитку
- Блок Спред показує результат тільки після введення Ask і Bid

*Оновлено: 27.06.2026*
