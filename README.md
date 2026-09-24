# week-USDCAD

A small Python app that pulls the major Canadian banks' quarterly FX forecasts every week, lets a person
check and correct them, and writes one clean table for the weekly report.

Status: **design done, feasibility tested (Sep 2026), build not started.**

## The app
1. Open the app. It pulls all banks right away (target: under 30 seconds).
2. One row per bank: forecasts for the 4 quarters, publish date, status, and an **Open source** link to
   the page or PDF the numbers came from (always shown, so anything can be checked).
   - ✓ pulled and passed all checks
   - ⚠ pulled, but looks odd (far from the other banks, a big jump since last week, older than 45 days)
   - ✗ failed: boxes stay empty with the reason and a "where to find it" note, and the person types the numbers in
3. Every value can be edited. The average row updates live.
4. **Generate** writes the output for all banks and all pairs: `output/<date>/` with the table as CSV,
   the downloaded source files, and one more week added to `history.csv`.
   The report itself (PDF/PowerPoint) is built by separate tools that read this table.

## Rules
- Banks: BMO, CIBC, TD, Scotiabank, National Bank. RBC later (internal).
- Quarters: the current quarter + the next 3. The window moves forward on the first day of a new quarter.
- Value: the bank's quarterly forecast (end of period; BMO publishes quarterly averages, used as is).
- CIBC has no current-quarter column: its dated "current" column is used for the current quarter.
- Use the bank's own quote at its published decimals. Flip the quote only if missing, and flag it.
- Old forecast: use the latest, show its publish date. Missing value: blank + flag, never guessed.

## How each bank is pulled
One tailored script per bank. Only TD is read from a web page; the others download a PDF.

| Bank | Start page | Steps | USDCAD row |
|---|---|---|---|
| TD | https://economics.td.com/us-forecast-tables | read the table on the page | "CAD per USD" |
| Scotia | https://www.scotiabank.com/ca/en/about/economics/forecast-snapshot.html | find the `forecastYYYYMMDD.pdf` link → PDF | "Canadian dollar (USDCAD)" |
| National Bank | https://www.nbc.ca/about-us/news-media/financial-news/financial-analysis.html | find the monthly monitor / `forex.pdf` link → PDF | "CAD per USD" |
| CIBC | https://economics.cibccm.com/#/ | the report list the home page loads → "Interest Rate & FX Forecast" PDF | "USD-CAD" |
| BMO | https://economics.bmo.com/?contentType=FEEDS | anonymous session (same as clicking Accept) → report list → latest "Canadian Economic Outlook" PDF | "C$/US$ : qtr. avg." |

BMO needs no browser: plain HTTP does the same as clicking Accept (tested: about 4 seconds).
Headless Chrome is the fallback if BMO starts blocking it.

## Robustness
- **PDFs are read by position**, not by line order: the row is found by its label, and each value is
  matched to the quarter heading above it. (In plain text extraction some tables come out shifted by one
  row, which silently gives wrong numbers.)
- A bank is filled in automatically only if: the label is found exactly once, all 4 quarter headings are
  found, every value is in a plausible range, and a publish date is found. Otherwise it goes to manual.
- Warnings (don't block): far from the other banks' median, big jump since last week, older than 45 days.
- Each bank runs on its own with retries and a time limit. One failure never stops the others.
- Expected breakage (estimate): National Bank, Scotia low; TD low–medium; CIBC medium (internal report
  list); BMO medium–high (research platform, bot protection). A break shows as ✗ for that bank only.
- The source files are saved every week, and offline tests check every reader against saved copies.
  `pull.py --check` runs all readers and reports their status.

## Other pairs: EURUSD, GBPUSD, EURCAD, GBPCAD
Checked in the same documents (Sep 2026):

| Bank | EURUSD | GBPUSD | EURCAD | GBPCAD |
|---|---|---|---|---|
| TD | ✓ page | ✓ page | calculated | calculated |
| CIBC | ✓ FX PDF | ✓ FX PDF | calculated | calculated |
| Scotia | ✓ PDF | ✓ PDF | calculated | calculated |
| National Bank | ✓ forex.pdf | ✓ forex.pdf | ✓ forex.pdf | ✓ forex.pdf |
| BMO | ✓ International Economic Outlook | ✓ same | ✓ Canadian Economic Outlook | calculated |

"Calculated" = EURUSD × USDCAD (or GBPUSD × USDCAD) from the bank's own forecasts, flagged as calculated.
Built from 2-decimal inputs, the result can be off by about ±0.01.

## Planned layout
```
USDCAD Forecasts.bat   double-click to start the app
app.py                 the window (tkinter, comes with Python)
pull.py                pull without the window; --check = health check
usdcad/common.py       shared: download with retries, quarter window, PDF row by position, checks
usdcad/banks/*.py      td.py, scotia.py, nbc.py, cibc.py, bmo.py (30–60 lines each)
tests/                 saved source files + reader tests
requirements.txt       pdfplumber (+ playwright only as the BMO fallback)
```

## Build order
1. Repo, `requirements.txt`, shared helpers, the 5 bank scripts, tests → `pull.py` prints the table.
2. The app window, including the manual path.
3. Other pairs.
4. Copy to the work PC.

## Disclaimer
Not affiliated with any bank. The forecasts and documents belong to the banks that publish them. This tool
only reads their public pages. It does not store or redistribute their documents in this repository.
