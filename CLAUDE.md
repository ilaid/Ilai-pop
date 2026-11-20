# Ilai-pop

Two unrelated things live here:

- `index.html` — a single file trade journal (Supabase backed).
- `pine/SavFX_Quarterly_Cycles.pine` — one TradingView Pine v5 indicator in
  four parts: the Quarterly Theory cycle ladder as rows of boxes in its own
  pane, the True Opens as lines on the candles, the Quarter Dividers (a
  vertical line at each turn of the block the chart is read against, 90 min on
  5m and 22.5 min on 1m, those two timeframes only, each carrying the Q1 to Q4
  of the block it opens), and the Devil Marks.
  Everything but the ladder reaches the price chart with `force_overlay` while
  the script itself is `overlay = false`, which is the only way to have both at
  once.

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
- The ladder's **year is the year as the chart has it**: 1 January is never a
  trading day, so it opens on the first weekday after it and runs until the
  next year opens — 2.1.2026 to 4.1.2027. Its weekdays (261) divided by four
  are the quarters (65 each), so 2026 turns on 2.1, 3.4, 3.7 and 2.10. The
  calendar months play no part, and a turn day belongs to the quarter it opens.
  The TYO stays on 1 April all the same: it is the QT year, not this ladder.
- The four year cycle is 2023, 2024, 2025, 2026, written `(year + 1) % 4 + 1`.
- A week belongs to the month holding its **Tuesday**; a week whose Tuesday and
  Thursday fall in different months is dead and carries no number.
- The quarter split counts the quarter's **weekdays**, the handover day counted
  in, and divides by four, the last cell taking the rest: 66 in Q1 2026, so 17
  a cell and it turns on 2.1, 27.1, 19.2, 16.3. Q4 2026 turns on 2.10, 27.10,
  19.11, 14.12. A turn that lands on a shut day (3.4 is Good Friday, 3.7 is the
  Fourth) is not moved — the cell simply opens on its first real candle, which
  is why Q2 2026 looks like it runs 6.4 to 6.7.
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
