# Phase 1 in Power Query – click-by-click (using the tools from your course slides)

Every step below uses a tool from the **Introduction to Power Query** slides:
Trim/Clean (slide 17), Replace Values, Change Type **Using Locale** (slide 23),
Replace Errors / Keep Errors (slide 14), Conditional Column (slide 26), Custom Column,
Index Column (slide 25), Column Quality/Profile (slides 14-16), Unpivot (slide 28) and Group By (slide 27).

Each click becomes a line in **Applied Steps**. For Q1.1, open **Home > Advanced Editor**
and copy the M code Power BI wrote for you (examples are at the end of this guide).

> Column names below must be typed **exactly** as shown, because the DAX measures use them.

---

## STEP 0 – Turn on profiling on the whole dataset (important for Q1.2)
1. **Home > Get Data > Text/CSV** > choose `train_data.csv` > **Transform Data**.
2. Rename the query **Train** (Query Settings > Name).
3. **View** tab: tick **Column quality**, **Column distribution** and **Column profile**.
4. In the status bar at the bottom left, click **"Column profiling based on top 1000 rows"** and change it to **"based on entire data set"**. Without this, the percentages only describe the first 1,000 of the 4,107 rows.
5. In **Applied Steps**, delete the automatic **Changed Type** step (click the X). That keeps every column as text until we've cleaned it. Otherwise Power BI guesses the types and turns messy values into errors too early.
6. Delete the last, empty column (`Column61`): right-click > **Remove**.
7. **Add Column > Index Column > From 0**, then rename it `row_id`. This gives every record an ID for the audit log.

---

## STEP 1 – Clean the 8 date columns

Hold **Ctrl** and select all 8 date columns:
`Reporting Date`, `Date of Birth`, `Date of Onset/Emergence`, `Date of Detection/Seen`,
`Date Specimen Collected`, `Date Specimen Sent`, `Date Specimen Received at Lab`, `Date Lab Tested/ Reviewed`.

**1a. Remove spaces and hidden characters:** **Transform > Format > Trim**, then **Format > Clean**.

**1b. Remove the time part.** With the 8 columns still selected, go to **Transform > Replace Values**:
- Value to find: ` 00:00:00` (with the leading space)
- Replace with: *(leave empty)*
- Open **Advanced options** and make sure **Match entire cell contents** is **unticked**.

**1c. Standardise the separators** (the guide's `30_dec-25` example). Use **Replace Values** again, with *Match entire cell contents* unticked, once for each row:

| Value to find | Replace with |
|---|---|
| `_` | `-` |
| `.` | `-` |
| `;` | `-` |
| `/` | `-` |
| `=` | *(empty)* |
| `--` | `-` |

**1d. Fix misspelled month names.** These are what make some dates unreadable. Replace Values is **case-sensitive**, and *Match entire cell contents* stays unticked:

| Value to find | Replace with | Fixes e.g. |
|---|---|---|
| `Ffeb` | `Feb` | `8-Ffeb-26` |
| `Fab` | `Feb` | `9-Fab-26` |
| `Mare` | `Mar` | `10-Mare-26` |
| `-Mat-` | `-Mar-` | `15-Mat-26` (assumed March) |
| `-Lan-` | `-Jan-` | `17-Lan-26` |
| `-an-` | `-Jan-` | `30-an-26` |
| `-Jn-` | `-Jan-` | `29-Jn-26` |
| `Aprial` | `Apr` | `4-Aprial-26` |
| `Appr` | `Apr` | `3-Appr-21` |
| `Sept` | `Sep` | `31-sept-22` |
| `3o-` | `30-` | `3o-Jan-26` |

**1e. Convert to dates.** With the 8 columns still selected, click the **ABC** icon on one header and choose **Using Locale…**. Set **Data Type: Date** and **Locale: English (United States)**, as on slide 23.
- Values that are real dates become dates.
- Placeholders such as `Pending`, `Not done`, `No specimen collected` or `Yet to be received` become **Error**. So do impossible dates like `31-Apr-26` and `29-Feb-26`.

**1f. Take the data-quality numbers for Q1.2 now, before the next step.**
- Hover over each date column's **Column quality** bar. Write down or screenshot the **Valid / Error / Empty** percentages.
- Error + Empty together = "missing or placeholder" dates.
- Expected training figures (from Python, as a check): specimen-collected about 72% missing, lab-tested about 96% missing, onset under 1%.

**1g. Turn placeholders into blank dates:** with the 8 columns selected, **Transform > Replace Values ▾ > Replace Errors**, type `null`, then OK.

---

## STEP 2 – Audit log of missing dates (a separate query)
1. In the Queries pane, right-click **Train > Duplicate**, and rename the copy **Date Audit**.
2. In Date Audit's Applied Steps, right-click the **Changed Type with Locale** step (1e) > **Delete Until End**. This query now holds the cleaned *text* values, before conversion.
3. Select `row_id` plus the 8 date columns, then right-click > **Remove Other Columns**.
4. Select `row_id`, then **Transform > Unpivot Columns ▾ > Unpivot Other Columns**. Rename `Attribute` to `Date Column` and `Value` to `Raw Value`.
5. Select `Raw Value`, then **Add Column > Duplicate Column**. Change the copy to **Date using Locale (English US)**.
6. On the copy, open the column's options menu (…), then **Keep Errors**. You now have every record and column whose value was a placeholder or invalid, with the original text kept.
7. **Transform > Group By**: group by `Date Column`, operation **Count Rows**. This gives the count per column for your Q1.2 table.
8. Right-click **Date Audit > Enable load** to untick it if you don't want it in the report. You can also keep it as an audit table.

---

## STEP 3 – Flag records that break the date sequence
**Add Column > Custom Column**. Name it `Date Sequence Violation` and paste:

```
let d = List.RemoveNulls({[#"Date of Onset/Emergence"], [#"Date of Detection/Seen"],
                          [Date Specimen Collected], [Date Specimen Sent],
                          [Date Specimen Received at Lab], [#"Date Lab Tested/ Reviewed"]})
in List.Count(d) > 1 and List.AnyTrue(List.Transform({1..List.Count(d)-1}, each d{_} < d{_-1}))
```

Set the column type to **True/False**.

What it does: it puts the six anchor dates in order, skips the blank ones, and returns TRUE if any later step has an earlier date than the step before it. **Handling:** we keep these records and flag them. The DAX timeliness measures leave out any interval that comes out negative.

*(Python found 117 violating records in train; your count should be very close.)*

---

## STEP 4 – Age in Years
1. Select `Age Type`, then **Transform > Format > Trim**, then **Format > Capitalize Each Word**. This fixes `years`, `Months `, `weeks` and `days`.
2. Select `Patient Age`, then **Data Type: Decimal Number**. `NP` becomes an Error, so use **Replace Errors** with `null`.
3. **Add Column > Custom Column**, name `Age in Years`:
   ```
   if [Age Type] = "Years" then [Patient Age]
   else if [Age Type] = "Months" then [Patient Age] / 12
   else if [Age Type] = "Weeks" then [Patient Age] / 52.1775
   else if [Age Type] = "Days" then [Patient Age] / 365.25
   else if [Age Type] = "Minutes" then [Patient Age] / 525960
   else null
   ```
   Set the type to **Decimal Number**, then **Transform > Rounding > Round…** to 3 decimals (Numerical tools, slide 19).
4. **Add Column > Conditional Column**, name `Age Group` (slide 26):

   | Column | Operator | Value | Output |
   |---|---|---|---|
   | Age in Years | is less than | 1 | `<1 year` |
   | Age in Years | is less than | 5 | `1-4 years` |
   | Age in Years | is less than | 10 | `5-9 years` |
   | Age in Years | is less than | 15 | `10-14 years` |
   | Else | | | `15+ years` |

---

## STEP 5 – Vaccination Status (exactly 4 categories)
1. Select `Vaccination Status`, then **Format > Trim**, then **Format > Capitalize Each Word**. After this, `Not vaccinated` and `Not Vaccinated` are identical.
2. **Add Column > Conditional Column**, name `Vacc Status Clean`:

   | Column | Operator | Value | Output |
   |---|---|---|---|
   | Vaccination Status | begins with | `Not Vac` | `Not vaccinated` |
   | Vaccination Status | begins with | `Not Ap` | `Not applicable` |
   | Vaccination Status | begins with | `Vac` | `Vaccinated` |
   | Vaccination Status | equals | `Up To Date` | `Vaccinated` |
   | Else | | | `Unknown` |

   The `begins with "Vac"` rule catches *by history, by card, Vaccinted, histiry, (PEP)*. The Else rule turns *Unknown* and *Not Provided* into `Unknown`.
3. Remove the old `Vaccination Status` column, then rename `Vacc Status Clean` to **`Vaccination Status`**.

---

## STEP 6 – Vaccine Doses (0, 1, 2, Unknown)
1. Select `Vaccine Doses`, then **Format > Trim**, then **Format > lowercase**.
2. **Add Column > Conditional Column**, name `Doses Clean`. The order matters, because the first rule that matches wins:

   | Column | Operator | Value | Output |
   |---|---|---|---|
   | Vaccination Status | equals | `Not vaccinated` | `0` |
   | Vaccine Doses | equals | `0` | `0` |
   | Vaccine Doses | equals | `o` | `0` |
   | Vaccine Doses | equals | `not vaccinated` | `0` |
   | Vaccine Doses | contains | `2` | `2` |
   | Vaccine Doses | contains | `3` | `2` |
   | Vaccine Doses | ends with | `/1` | `1` |
   | Vaccine Doses | contains | `0.5` | `Unknown` |
   | Vaccine Doses | contains | `o.5` | `Unknown` |
   | Vaccine Doses | contains | `1` | `1` |
   | Vaccine Doses | contains | `one` | `1` |
   | Else | | | `Unknown` |

   `0.5ml` is the injection **volume**, not a number of doses, so it becomes `Unknown`. 3 doses are grouped into "2+".
3. Remove the old column and rename `Doses Clean` to **`Vaccine Doses`**.
4. **Conditional Column** `Dose Group`: `0` → `Zero-Dose`, `1` → `1 Dose`, `2` → `2+ Doses`, Else → `Unknown`.

---

## STEP 7 – Other columns the dashboard and DAX need
1. **Disease/Condition**: Trim, then Replace Values with *Match entire cell contents* **ticked**: `MPox` → `Mpox`, `Yellow Fever` → `Yellow fever`.
2. **Outcome**: Trim, then Capitalize Each Word.
3. **Final Epi-classification**: Trim, then lowercase. Then add a **Conditional Column** named `Final Epi-classification (std)`:

   | Operator on Final Epi-classification | Value | Output |
   |---|---|---|
   | contains | `lab` | `Laboratory-Confirmed` |
   | equals | `confirmed` | `Laboratory-Confirmed` |
   | contains | `epi` | `Epidemiologically Linked` |
   | contains | `clinic` | `Clinically Compatible` |
   | contains | `discard` | `Discarded` |
   | contains | `not a case` | `Discarded` |
   | contains | `negative` | `Discarded` |
   | equals | `not provided` | `Missing` |
   | equals | `not applicable` | `Missing` |
   | Else | | `Suspected/Pending/Other` |

4. **Final Laboratory Result**: Trim, then lowercase. Then add a **Conditional Column** named `Final Laboratory Result (std)`: begins with `negative` → `Negative`; begins with `positive` → `Positive`; equals `confirmed` → `Positive`; begins with `pending` → `Pending`; Else → `Not tested / Other`.
5. **Epi week**: Data Type **Whole Number**.
6. **Map location for the map visual:** **Add Column > Custom Column**, name `Map Location`:
   ```
   [Reporting District] & ", " & [Reporting County] & ", Democratic Republic of the Congo"
   ```
7. **IsMeasles** (also used in Phase 4): **Conditional Column**. If `Disease/Condition` equals `Measles`, output `1`. Else `0`. Set the type to Whole Number.

---

## STEP 8 – Apply the same cleaning to test_data.csv
1. Right-click **Train > Duplicate**, and rename the copy **Test**.
2. In Test's Applied Steps, click the gear icon ⚙ next to **Source**, browse to `test_data.csv`, then OK.

   All the cleaning steps now run on the test file. This is the "validate your cleaning pipeline on the test set" part of the brief.
3. **Home > Close & Apply.**

---

## For Q1.1 – M code snippets to quote (copy from Advanced Editor)
Your own Advanced Editor will contain lines like these. Copy your real ones.

```
// Change Type with Locale  -> mixed-format date parsing
#"Changed Type with Locale" = Table.TransformColumnTypes(#"Replaced Value10",
    {{"Date of Onset/Emergence", type date}, {"Date Specimen Collected", type date}}, "en-US"),

// Placeholders -> null
#"Replaced Errors" = Table.ReplaceErrorValues(#"Changed Type with Locale",
    {{"Date of Onset/Emergence", null}, {"Date Specimen Collected", null}}),

// Month-name typo
#"Replaced Value5" = Table.ReplaceValue(#"Replaced Value4","Ffeb","Feb",
    Replacer.ReplaceText,{"Date of Onset/Emergence", "Date of Detection/Seen"}),

// Age conversion (Custom Column)
#"Added Custom" = Table.AddColumn(#"Replaced Errors1", "Age in Years", each
    if [Age Type] = "Years" then [Patient Age]
    else if [Age Type] = "Months" then [Patient Age] / 12 ... )
```

The matching Pandas code is in `01_clean_data.ipynb`, in the `clean_age`, `parse_date` and `MONTH_TYPOS` cells.

**Points worth making in your Q1.1 comparison:**
- **Age:** about equally easy in both. One Custom Column in Power Query, and a dictionary plus a multiplication in Pandas (`df["Patient Age"] * df["Age Type"].map(AGE_FACTOR)`).
- **Dates and placeholders:** Power Query's *Change Type with Locale* + *Replace Errors* is quick, visual, and turns every placeholder into an error you can **count** with Column Quality.
  - But it only understands formats the locale recognises. Every typo needs its own Replace Values step.
  - Python's function handled more odd formats (`Feb-15-26`, `6-Feb-022`, `117-mar-26`), checked dates were in a plausible range, and wrote a full audit log automatically.
  - So Python was more thorough. Power Query was faster to build and easier for a non-programmer to follow and refresh.
- **Expect small differences** between the two tools' counts, a few dozen dates at most. Explaining why (different parsing rules) makes a strong answer.
