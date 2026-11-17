# תוספת ל-QT CC — ייצוא השבירות לבודק

5 תוספות קטנות. לא משנות שום חוק או תצוגה, רק מוציאות 3 plot-ים מוסתרים שהבודק קורא.

## 1. בתוך `type Eng` — בסוף, אחרי `array<string> alT`

```pine
    int           evBe       = 0      // ייצוא: שבירות לירידות שנרשמו בצעד הזה (סכום ביטים)
    int           evBu       = 0      // ייצוא: שבירות לעליות
```

## 2. פונקציה חדשה — מיד אחרי `f_addAl` (לפני `f_addPsp`)

```pine
// ייצוא לבודק: 1 = Fill · 2 = Fill זוג · 4 = SSMT · 8 = SSMT זוג · 16 = PSP · 32 = PSP זוג
f_flag(Eng e, int dir, int bit) =>
    int cur = dir == 1 ? e.evBu : e.evBe
    if math.floor(cur / bit) % 2 == 0
        if dir == 1
            e.evBu := cur + bit
        else
            e.evBe := cur + bit
    bit
```

## 3. בתוך `f_addPsp` — מיד אחרי השורה `f_addAl(e, (bear ? "▼" : "▲") + "PSP" ...)`

```pine
    f_flag(e, bear ? -1 : 1, pair ? 32 : 16)
```

## 4. בתוך `f_step`

**א.** מיד אחרי `did := true`:

```pine
        e.evBe := 0
        e.evBu := 0
```

**ב.** ב-Fill — בתוך `if gr.register(not blk, t)`, אחרי שורת ה-`f_addAl`:

```pine
                    if gr.fFirst == t
                        f_flag(e, gr.dir, gr.kind == 0 ? 1 : 2)
```

**ג.** ב-SSMT — מיד אחרי `array.set(e.sLb, k, nS)`:

```pine
                if isNew and not f_closed(nS)
                    f_flag(e, isBull ? 1 : -1, k >= 2 ? 8 : 4)
```

## 5. בסוף הקובץ

```pine
// ═══════════════════════════════════════════════════════════════════
// ייצוא ל-QT CC Test — שבירות מסגרת הגרף שנרשמו בנר הזה, וה-True Open
// ═══════════════════════════════════════════════════════════════════
exStep = tfClass != 0 and barstate.isconfirmed and eCh.lastT == time
plot(exStep ? eCh.evBe : 0, "ייצוא — שבירות לירידות", display = display.none)
plot(exStep ? eCh.evBu : 0, "ייצוא — שבירות לעליות", display = display.none)
plot(tfClass != 0 ? eCh.toP : na, "ייצוא — True Open", display = display.none)
```
