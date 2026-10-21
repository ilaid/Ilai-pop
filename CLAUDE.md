# Ilai-pop

Two unrelated things live here:

- `index.html` — a single file trade journal (Supabase backed).
- `pine/SavFX_Quarterly_Cycles.pine` — a TradingView Pine v5 indicator drawing
  the Quarterly Theory cycle ladder as rows of boxes in its own pane.

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

A cell is labelled with the chain of quarters leading to it, highest level
first, three links by default. Each row drops off its own end as a finer one
comes in, so no chart stacks more than four rows.

### Decisions already settled — do not revisit without being asked

- The trading day opens at 18:00 on the script's own scale (`time + 1h`,
  exchange time). This matches the script this one replaced; do not "fix" it.
- Quarters are plain calendar quarters. April is not a year boundary.
- The four year cycle is 2023, 2024, 2025, 2026, written `(year + 1) % 4 + 1`.
- A week belongs to the month holding its **Tuesday**; a week whose Tuesday and
  Thursday fall in different months is dead and carries no number.
- The quarter split divides the quarter's three months by **market days**
  (Mon-Fri), not calendar days.
- Friday, a dead week and the weekend carry no number: colourless cell, and a
  mark holds the link's place in the chain so every row reads the same levels
  in the same order whatever the day.
- Cell edges are measured off the open of the bar the cell starts on, in whole
  days, then half a bar back, so an edge falls between two candles rather than
  through the middle of one. Never off a calendar midnight.

### Branch

Work on `claude/savfx-multi-level-indicators-odbgl7` and push there.
