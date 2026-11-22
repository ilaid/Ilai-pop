# Ilai-pop

Two unrelated things live here:

- `index.html` — a single file trade journal (Supabase backed).
- `pine/SavFX_Quarterly_Cycles.pine` — one TradingView Pine v5 indicator in
  three parts: the Quarterly Theory cycle ladder as rows of boxes in its own
  pane, the Quarter Dividers (a vertical line at each turn of the block the
  chart is read against, 90 min on 5m and 22.5 min on 1m, those two timeframes
  only, each carrying the Q1 to Q4 of the block it opens), and the Devil Marks.
  Everything but the ladder reaches the price chart with `force_overlay` while
  the script itself is `overlay = false`, which is the only way to have both at
  once.
  The **True Opens were taken out** at the owner's request, and with them the
  Premium/Discount zone box, which was computed from them. They live on as a
  standalone script; do not put either back without being asked.

## Working on the indicator

**Always print the whole `.pine` file in the reply, in one `pine` code block,
after any change however small.** The file is pasted straight into
TradingView's editor, so a diff or a snippet is not usable on its own. Say what
changed in a line or two first, then the full file.

There is no Pine compiler here, so the logic is checked by mirroring the
arithmetic in a throwaway Python script under the scratchpad directory and
running it against real dates, before the change is committed. Say plainly that
the script was never compiled.

### What the indicator is

Nine levels, each a quarter of the one above, drawn one row per level with the
slowest at the bottom of the pane:

| row | cell | numbers |
|---|---|---|
| Quadrennial | 1 year | which year of the four year cycle, 2023 opens it |
| Yearly | 1 quarter | Q1 Jan-Mar .. Q4 Oct-Dec, April opens Q2 and the year's true open |
| Quarter Split | ~16 market days | which quarter of the quarter, daily chart only |
| Weekly | 1 week | which week of the month, by the week's Tuesday |
| Daily | 1 day | Mon 1 .. Thu 4, Friday carries no number |
| 6 Hour | 6 hours | Asia 1, London 2, AM 3, PM 4 |
| 90 Minute | 90 min | four per session |
| 22.5 Minute | 22.5 min | four per 90 min |
| 5.625 Minute | 5.625 min | four per 22.5 min |

On 5m the chart reads day, session, 90 min, 22.5 min — the 5.625 min row only
comes in below 5m, and the daily row steps aside for it there.

A cell is labelled with the chain of quarters leading to it, highest level
first, three links by default. Each row drops off its own end as a finer one
comes in, so no chart stacks more than four rows.

### Decisions already settled — do not revisit without being asked

- One shared clock for everything: the trading day opens at 18:00 on the **New
  York wall clock**, full stop. The rows and the true opens must never run on
  different clocks, and every container and boundary the true opens use is
  taken from the row grid.
- The time of day is read off the clock face — `hour()`/`minute()`/`second()`
  in New York, plus `time % 1000` for the milliseconds — never as the time
  elapsed since local midnight. Counting from midnight makes the changeover
  Sunday 25 hours long, which slid the whole ladder an hour and left the
  trading day that opened that evening numbered as Sunday, so it lost its day
  number and fell back to week 4 of the month before.
- There is no "Exchange" clock option and no need for one. The only knob is
  "Shift From New York (hours)", which belongs on 0 and is there for moving the
  ladder on purpose, not for putting daylight saving right.
- **The ladder follows the owner's written rule set, and nothing else.** It
  was handed over in full and every date in it is reproduced exactly; it is the
  specification, so check any change against it before committing.
  - **Rule 1, the year and its quarters (the weekly chart).** The year opens on
    the **Monday of the week holding the first Tuesday of January**. Every week
    belongs to the year and quarter of **its own Tuesday**. The year is four
    quarters of **exactly 13 weeks**, Q1 being January; a 53-week year gives
    the extra week to **Q4** (14 weeks). A quarter opens on a Monday and closes
    the Sunday before the next. 2024: 1.1, 1.4, 1.7, 30.9 (Q4 14 weeks).
    2025: 6.1, 7.4, 7.7, 6.10. 2026: 5.1, 6.4, 6.7, 5.10, closing 3.1.2027.
  - **Rule 2, the quarter of the quarter (the daily chart).** The weekly
    quarter's **trading days** divided in four: a cell opens after a quarter, a
    half and three quarters of them, **rounded to nearest, a half rounding up**.
    2026: **Q1** 63 td, 5.1 · 28.1 · 20.2 · 13.3 — **Q2** 62, 6.4 · 28.4 ·
    19.5 · 11.6 — **Q3** 64, 6.7 · 28.7 · 19.8 · 11.9 — **Q4** 62, 5.10 ·
    27.10 · 17.11 · 10.12. 2025 and 2024 are in the Pine header comment.
  - **The nine holidays** that are not trading days: New Year's Day, MLK (3rd
    Mon Jan), Presidents (3rd Mon Feb), Memorial (last Mon May), Juneteenth
    (19.6), Independence (4.7), Labor (1st Mon Sep), Thanksgiving (4th Thu Nov),
    Christmas (25.12). A Saturday holiday moves to the Friday, a Sunday one to
    the Monday — **except New Year's Day, which does not move back to the
    Friday**, so a Saturday one costs no trading day. **Good Friday IS a
    trading day**: the futures trade it. That is the one place this parts
    company with the equity calendar.
  - Because every boundary is a Monday, a bar's date is **snapped forward to
    its Monday** (`wkSnap`) before its Tuesday is taken. The bar that opens a
    week comes out dated to the Sunday on any symbol whose week opens before
    18:00 New York — the FX week opens at 17:00 — and without the snap that
    whole week fell into the quarter before and the next quarter started a
    **week late**. On a daily chart the snap is 0.
  - The inverse (`tdDayOf`) counts the candidate day's **own** holiday in, which
    is what puts the first cell on the right day when a quarter opens on a
    holiday: 1.1.2024 is a Monday and shut, so Q1 2024 opens on the 2nd.
  - Earlier attempts — quarters on the day of the month the year opened on
    (2.1/2.4/2.7/2.10), quarters on the first Monday of Jan/Apr/Jul/Oct,
    counting candles instead of trading days, and a date list typed into the
    settings — are all **wrong and were removed**. Do not bring any of them back.
- The four year cycle is 2023, 2024, 2025, 2026, written `(year + 1) % 4 + 1`,
  off the QT year.
- A week belongs to the month holding its **Tuesday**; a week whose Tuesday and
  Thursday fall in different months is dead and carries no number.
- Friday, a dead week and the weekend carry no number: colourless cell, and a
  mark holds the link's place in the chain so every row reads the same levels
  in the same order whatever the day.
- Cell edges sit on real bars, never on a computed calendar time. A cell opens
  half a bar before its own first candle, and `cell()` closes the cell before it
  at that same coordinate, so two neighbours share one edge. A computed edge can
  land in a hole in the data — 1 January 2027 is a Friday and a holiday, so the
  quarter's first candle is Monday the 4th — and TradingView places a time
  inside a gap by the average bar spacing, which drifts it on to candles that
  belong to the cell before. The cell still running keeps its computed end, so
  the row reaches to the right like the one below it, until its successor opens
  and takes the edge over.

### Branch

Work on `claude/savfx-multi-level-indicators-odbgl7` and push there.
