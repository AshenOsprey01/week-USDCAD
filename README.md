# week-USDCAD

A small Python app that pulls the major Canadian banks' **USDCAD quarterly forecasts** every week, shows
them with links to the source documents, lets a person fix anything, and writes one clean table for the
weekly report.

**Status:** specification only. Every source and every step below was tested by hand in September 2026
(Windows 11, Python 3.14). Nothing is built yet. This README is written so an AI coding agent can build the
app step by step from it (see [Build steps](#build-steps-with-prompts)).

---

## 1. Scope
- **Pair:** USDCAD only (CAD per 1 USD, about 1.3–1.4). Other pairs are a later upgrade (section 12).
- **Banks:** BMO, CIBC, TD, Scotiabank, National Bank. RBC later (internal source).
- **Part 1, pulling:** one tailored reader per bank downloads the page/PDF and reads the USDCAD row.
- **Part 2, the app:** a window showing all banks, with source links, editable values and a manual fallback.
- **Out of scope:** the PDF/PowerPoint report. Existing tools read the table this app writes.

## 2. The app (what the user sees)
1. Double-click `USDCAD Forecasts.bat`. The window opens and **pulls all banks right away** (target: under 30 s).
2. One row per bank:
   `Bank | Q3 2026 | Q4 2026 | Q1 2027 | Q2 2027 | Published | Status | Open source | Retry`
   - **Open source** is shown for **every** bank: it opens the exact PDF (or page) the numbers came from,
     or the bank's start page if the pull failed.
   - Status: ✓ passed all checks · ⚠ pulled but looks odd (reason shown) · ✗ failed (reason shown).
   - Failed rows: value boxes stay empty and a **where to find it** note is shown. The user opens the
     source and types the 4 numbers and the publish date.
3. Every value box is editable. An edited value is marked `edited`. An **Average** row updates live.
4. A bank can be marked **Skip this week**.
5. **Generate** (enabled only when every bank has 4 values or is skipped) writes the output (section 4)
   and opens the output folder.

## 3. Data rules
- **Quarters:** the current quarter + the next 3, based on today's date (on Sep 23, 2026: Q3 2026, Q4 2026,
  Q1 2027, Q2 2027). The window moves forward on the first day of a new quarter.
- **Value:** the bank's forecast for that quarter, end of period. BMO only publishes quarterly averages;
  use them as is (agreed), and write `quarterly average` in its notes.
- **Current quarter without a quarter-end column** (CIBC): use the bank's dated "current" column
  (for example `11-Sep`). Note it: `Q3 = rate as of 11-Sep`.
- **Quote direction:** use the bank's own USDCAD row. Only if it is missing, use 1 / CADUSD and flag it.
- **Decimals:** keep what the bank published (most use 2, BMO uses 3). Never round a bank's value.
- **Old forecast:** use the bank's latest document and show its publish date. Banks update at different
  speeds (BMO weekly; Scotia, CIBC, National Bank, TD about monthly).
- **Missing value:** leave it blank with a flag. **Never guess or fill in.**
- **Average:** plain mean of the banks that have a value for that quarter, 4 decimals, with the count
  (`n=5`) in its notes.

## 4. Output (written by Generate)
```
output/<YYYY-MM-DD>/usdcad.csv      the table
output/<YYYY-MM-DD>/sources/        every downloaded page/PDF, as evidence
history.csv                         one line per bank x quarter per week (appended)
```
`usdcad.csv` columns (quarter columns are named after the actual quarters):
```
bank, Q3 2026, Q4 2026, Q1 2027, Q2 2027, published, method, status, notes, document_url, start_page
```
- `method`: `auto`, `manual` or `edited`. `status`: `ok`, `warning` or `failed`.
- The last row is `Average`.

`history.csv` columns: `week, bank, quarter, value, published, method`.

## 5. Project layout
```
USDCAD Forecasts.bat        starts app.py with the project .venv
app.py                      the window (tkinter, ships with Python)
pull.py                     command line: pull all banks and print the table
                            --check   run all readers, print status per bank, exit code 1 if any failed
usdcad/
  common.py                 shared helpers (section 6)
  banks/td.py               one reader per bank (section 7)
  banks/scotia.py
  banks/nbc.py
  banks/cibc.py
  banks/bmo.py
tests/
  fixtures/<date>/          saved source files from one real week
  test_readers.py           parse the fixtures offline, compare with the checked values
requirements.txt            pdfplumber, pytest (+ playwright only for the optional BMO fallback)
```
**Key design rule:** every reader has two functions:
- `fetch() -> Sources`: network only. It downloads and returns the files and URLs.
- `parse(sources, quarters) -> Result`: no network. It reads the numbers.

Tests call `parse` on saved files, so they run offline and prove a reader still works after any change.

`Result` fields: `bank, values {quarter: float|None}, published (date), document_url, start_page,
status, messages [str], notes [str], where_to_find (str)`.

## 6. Shared helpers (`usdcad/common.py`)
**HTTP**
- Always send a normal browser User-Agent, e.g.
  `Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/140.0.0.0 Safari/537.36`.
- Timeout 30 s per request. 3 tries with waits of 2 s and 5 s. Time limit about 90 s per bank.
- BMO needs cookies across requests (a cookie jar / `requests.Session`). Everything was tested with
  Python's built-in `urllib` + `http.cookiejar`; `requests` works the same.
- Each bank runs on its own (thread per bank). One failure never stops the others.

**Quarters**
- `quarter_window(today) -> ["Q3 2026", "Q4 2026", "Q1 2027", "Q2 2027"]`.
- Heading conversions used by the readers: `Q3F`/`Q3f` → Q3 (F = forecast). Months: Mar → Q1, Jun → Q2,
  Sep → Q3, Dec → Q4.

**Reading a PDF table row (pdfplumber, by position)**

Never use plain-text line order. In CIBC's PDF the labels come out shifted by one row, so line order
reads EUR-USD as USD-CAD. In Scotia's PDF, rows get mixed together.
1. `page.extract_words(x_tolerance=...)`: 3 (the default) for most PDFs; **8 for National Bank**
   (its letters are stored one by one).
2. **Find the row:** the label word(s) on one line (same `top` within 3 pt). The label must be found
   **exactly once**, or the reader fails.
3. **Values:** words on that line to the right of the label matching `^\d\.\d{2,4}$`.
4. **Headings:** the quarter heading words above the row (bank-specific, section 7). Build each heading's
   quarter name (e.g. `Q3 2026`) and its horizontal **centre** `(x0 + x1) / 2`.
5. **Match each value to the heading with the nearest centre.** Numbers are right-aligned and headings
   left-aligned, so left edges differ by 5–10 pt; centres line up. Accept a match only within about
   15 pt, and use each heading once.
6. **Years for quarter-only heading rows** (TD, Scotia, BMO): the leftmost year label above the quarter
   row is the year of the first quarter column. Walk left to right and add 1 to the year at every `Q1`
   after the first column. Stop at the first token that is not a quarter (annual columns follow).

**Checks** (a reader returns `failed` if any of these fails)
- label found exactly once
- all 4 target quarters found
- every value between 1.20 and 1.60 (one constant in `common.py`; change it if USDCAD ever leaves that range)
- publish date found

**Warnings** (status `warning`, never blocking)
- a value more than 5% from the median of the other banks for that quarter
- a value that moved more than 3% since last week (from `history.csv`)
- publish date older than 45 days

## 7. Bank recipes
All five were tested with plain HTTP (no browser). Page numbers are from September 2026 documents and
are **not** fixed: search the pages for the table, never hard-code a page number.

### 7.1 TD (web page, no PDF)
- **Start page:** https://economics.td.com/us-forecast-tables
- **Fetch:** GET the page (about 1.2 MB of HTML).
- **Table:** the `<table>` right after the heading text **"Foreign Exchange Outlook"**.
  - Row 1: `Currency | Exchange Rate | 2026 | 2027 | 2028`. Each year cell has `colspan="4"`.
  - Row 2: `Q1 Q2 Q3F Q4F Q1F Q2F Q3F Q4F Q1F Q2F Q3F Q4F`.
  - Target row: first cell `Canadian Dollar`, second cell `CAD per USD`, then 12 values.
  - Cells contain `&nbsp;`, so strip them.
- **Quarters:** expand the year cells by their colspan to give each quarter column its year.
- **Published:** the note above the table, `F: Forecast by TD Economics, September 2026. All forecasts are
  end-of-period.` Parse the month and year.
- **Where to find it (manual):** "TD forecast tables page → Foreign Exchange Outlook table → row
  Canadian Dollar / CAD per USD."
- **Parser:** stdlib `html.parser` or BeautifulSoup. Read the table cells, not the page text.

### 7.2 Scotiabank (page → dated PDF)
- **Start page:** https://www.scotiabank.com/ca/en/about/economics/forecast-snapshot.html
  (the snapshot page itself only shows yearly values, so the quarters come from the PDF it links).
- **Fetch:**
  1. GET the start page. Find links matching
     `/content/dam/scotiabank/sub-brands/scotiabank-economics/english/documents/forecast-tables/forecast(\d{8})\.pdf`
     (e.g. `forecast20260909.pdf`). Take the link with the latest date and prefix `https://www.scotiabank.com`.
  2. GET the PDF.
- **Table:** the page with **"Currencies and Interest Rates"** (page 7 in Sep 2026), block
  `Americas (end of period)`.
  - Target row label: `Canadian dollar (USDCAD)` (match the word `(USDCAD)`). The row right below is
    `Canadian dollar (CADUSD)`; don't mix them up.
  - Year row: `2024 2025 2026 2027`, centred over the groups.
  - Quarter row: `Q4 Q1 Q2 Q3 Q4 Q1 Q2 Q3f Q4f Q1f Q2f Q3f Q4f` (lowercase `f` = forecast). The first
    column is Q4 of the first year. Use the year-walking rule (section 6).
  - 13 values, 2 decimals.
- **Published:** the date in the file name (`YYYYMMDD`), also printed on the page (`September 9, 2026`).
- **Where to find it:** "Scotiabank forecast PDF → page 'Currencies and Interest Rates' → row
  Canadian dollar (USDCAD), columns Q3f to Q2f."

### 7.3 National Bank (page → monthly PDF)
- **Start page:** https://www.nbc.ca/about-us/news-media/financial-news/financial-analysis.html
- **Fetch:**
  1. GET the start page and find the link to
     `/content/dam/bnc/taux-analyses/analyse-eco/mensuel/monthly-economic-monitor-canada.pdf`
     (prefix `https://www.nbc.ca`). The URL stays the same every month; the content changes.
  2. GET the PDF (7 pages).
- **Table:** the page containing **"Economic Forecast as of"** (page 5 in Sep 2026), section
  **"Financial Forecast\*\*"** (`** end of period`).
  - **Use `extract_words(x_tolerance=8)`.** With the default, the text comes out as single letters
    (`C A D p e r U S D`).
  - Target row label: the words `CAD per USD`. The first value is the current rate, then 4 quarters, then 3 yearly values.
  - Heading row: `Current` (with the as-of date below it, `9/03/26` = M/DD/YY), then `Q3 2026`, `Q4 2026`,
    `Q1 2027`, `Q2 2027`. **Each heading is two words** (`Q3`, `2026`): join them and use the centre of the pair.
  - **Trap:** to the right there is a yearly block whose headings, on the row above, are also
    `Q4 2025  Q4 2026  Q4 2027`. Use only the headings on the **same row as the as-of date** (the
    quarterly block). Never the first `Q4 2026` found anywhere.
- **Published:** `Canada – Economic Forecast as of September 4, 2026` on that page (page 1 says `September 2026`).
- **Where to find it:** "National Bank Monthly Economic Monitor (Canada) PDF → page 'Economic Forecast'
  → table Financial Forecast → row CAD per USD."

### 7.4 CIBC (report list → PDF)
- **Start page:** https://economics.cibccm.com/#/ (a JavaScript app; the reader uses the same data the
  home page loads).
- **Fetch:**
  1. GET `https://economics.cibccm.com/api/home/forecast`. It returns a JSON list of forecast reports with
     `PublicationId, PublishedDate, Title, PdfPath`.
  2. Pick `Title == "Interest Rate & FX Forecast"` with the latest `PublishedDate`. If there is no exact
     match, look for titles containing `FX Forecast` and flag it.
  3. GET `https://economics.cibccm.com/api/download?relativePath=<PdfPath>` (returns the PDF).
- **Table:** **"Table: Foreign exchange rates"** (page 2 in Sep 2026).
  - Target row label: `USD-CAD`. The row right above is `CAD-USD`. **Trap:** plain-text extraction
    puts the `CAD-USD` label on the USD-CAD numbers. Position reading gets it right.
  - Two heading rows, one entry per column:
    - years `2026 2026 2027 2027 2027 2027 2028 2028`
    - `11-Sep Dec Mar Jun Sep Dec Jun Dec`
  - Months → quarters (Mar Q1, Jun Q2, Sep Q3, Dec Q4) with the year above.
  - The `DD-Mon` column (`11-Sep`) is the current rate as of that date.
  - If the current quarter has no quarter-end column, use the `DD-Mon` column for it (rule in section 3).
  - Later columns skip quarters (Jun 2028, Dec 2028); that's fine.
- **Published:** `PublishedDate` from the JSON (the PDF page 1 also says `September 11, 2026`).
- **Where to find it:** "CIBC Economics home → Forecast Reports → Interest Rate & FX Forecast (PDF) →
  table Foreign exchange rates → row USD-CAD."

### 7.5 BMO (anonymous session → report list → PDF)
- **Start page:** https://economics.bmo.com/?contentType=FEEDS
- **Why this works:** the site shows a login screen until a visitor clicks **Accept** on the privacy
  banner. Accept only sends `POST /api/authorize-anonymous-user`, which creates an anonymous session. It
  has no CAPTCHA token, so plain HTTP can do the same.
- **Fetch** (keep one cookie jar for all calls; base `https://economics.bmo.com`):
  1. `GET /` (sets session cookies).
  2. `GET /api/session/csrf`. Read the **response header `x-xsrf-token`**. From now on, send it as the request
     header `X-XSRF-TOKEN` on every call, and update it whenever a response carries a new one.
  3. `GET /api/public/sitedata/navbar` → `navbarEntries[0].policyUuids[0]` (the privacy policy id,
     `0b0511b6-7e24-468c-bbf9-2cc30fe51c0f` in Sep 2026).
  4. `POST /api/authorize-anonymous-user` with JSON `{"policyUuid": "<id>"}`.
  5. `GET /api/session/csrf` again (new token), then `GET /api/validateSession` and `GET /api/identify`.
     **Without step 5, the next call returns 401.**
  6. `POST /api/document/v1/query` with the JSON body below → response `documents` (list). Pick the item
     whose `title` starts with `Canadian Economic Outlook for`, with the latest `published`. Take its `encrypt` id.
  7. `GET /api/document/v1/document/<encrypt>` → JSON with `pdfLink` (relative), `title`, `published`.
  8. `GET https://economics.bmo.com<pdfLink>` → the PDF (2 pages). About 4 seconds for all steps.

  Query body for step 6 (`searchSession` = any new UUID):
  ```json
  {"searchTerms": {}, "firmIds": [90210], "useRecommendationEngine": false, "includeUserFollows": false,
   "videoOnly": false, "securityFilterType": "PRIMARY", "securityFilterSymbols": [], "changeActions": [],
   "divisions": [{"creatorId": 31, "firmId": 90210}, {"creatorId": 5, "firmId": 90210}],
   "analysts": [], "analystRank": null, "industryTopicRelevancy": null, "sectors": [], "securities": [],
   "companies": [], "analystTags": [], "sectorTags": [{"creatorId": 10073, "firmId": 90210, "attributeId": 1}],
   "securityTags": [], "documentTypes": [{"creatorId": 1, "firmId": 90210}, {"creatorId": 13, "firmId": 90210},
   {"creatorId": 2, "firmId": 90210}], "regions": [], "countries": [], "brokerSpecificTags": [],
   "brokerCustomFields": [], "mainSubjects": [], "assetClassIds": [], "globalSecurityIds": [], "smartTags": [],
   "useStaticDocumentList": false, "filterIgnoreEntitlements": false, "filterIgnoreRestrictionList": false,
   "searchSession": "<new uuid>", "sort": {"sortField": "PUBLISH_DATE_FACET", "sortOrder": "DESC"},
   "paging": {"page": 1, "pageSize": 50}, "weaviateOffset": 0, "weaviateLimit": 50}
  ```
- **Table:** page 1, the row labelled `C$/US$` followed by `: qtr. avg.`.
  - **Trap:** the row above is `US¢/C$` (CAD in US cents, about 72). Don't use it.
  - Year row: `2025 2026 2027`, centred over 4-quarter groups.
  - Quarter row: `Q1 Q2 Q3 Q4` × 3, then annual columns `2024 2025 2026 2027` on the **same row**.
    Use only the `Q` headings (the year-walking rule stops at the first year token).
  - 16 values (12 quarterly + 4 annual), 3 decimals.
- **Published:** `published` from the document JSON (the title also carries the date).
- **Where to find it:** "BMO Economics home → Accept → latest 'Canadian Economic Outlook' → page 1 →
  row C$/US$ : qtr. avg."
- **Optional fallback, if plain HTTP gets blocked:** Playwright with the installed Chrome
  (`chromium.launch(channel="chrome", headless=True)`).
  - Set a normal User-Agent. The default headless one contains `HeadlessChrome` and gets **403**.
  - Open the start page and click the button named `Accept` (case-insensitive). Then run steps 6–8 with
    `page.request` (it shares the browser's cookies).

## 8. Expected breakage
Estimates. A break always shows as ✗ for that one bank; the others still work.

| Bank | Depends on | Risk |
|---|---|---|
| National Bank | fixed PDF name, table layout | low |
| Scotia | `forecastYYYYMMDD.pdf` link on the snapshot page | low |
| TD | table on the page | low–medium |
| CIBC | internal report list behind the home page, report title | medium |
| BMO | research platform's internal API, anonymous access | medium–high |

The remaining risk no check can catch: a bank changing what a number means without changing its label.
That's why a person looks at the table before Generate.

## 9. When a bank breaks
1. Run `python pull.py --check`. It prints which bank failed and why.
2. That week: enter the bank by hand in the app (Open source → type the 4 numbers).
3. To fix it: save the new page/PDF into `tests/fixtures/<date>/`, then give the AI this README section
   for that bank, the error message and the new file. Ask it to update **only that bank's reader**, and to
   make both the old and the new fixture tests pass.

## 10. Tests
- `tests/fixtures/<date>/` holds one real week of source files (each reader's `fetch()` output, saved by
  `pull.py --save-fixtures`).
- A person checks the numbers against the documents **once** and writes them into `test_readers.py` as the
  expected values.
- Also test the traps:
  - CIBC: the value must not come from the CAD-USD row
  - National Bank: the yearly `Q4 2026` must not be used
  - BMO: the `US¢/C$` row must not be used
  - Scotia: the value must not come from the CADUSD row
- Unit tests for `quarter_window` (for example on Sep 30 vs Oct 1) and for the year-walking rule.

## 11. Build steps with prompts
Give the AI this README first. Build **one step per prompt** and test before moving on. Suggested
opening for every prompt:

> Read README.md. Build only step N below. Follow the bank recipes in section 7 exactly: read PDFs by
> position (section 6), never by text line order, and never guess or fill in numbers. When done, show me
> the command you ran and its output.

| Step | Prompt (after the opening above) | Done when |
|---|---|---|
| 0 | "Set up the project: folder layout from section 5, `.venv`, `requirements.txt` with pinned versions, `.gitignore` (.venv, output, history.csv), git repo." | `python -c "import pdfplumber, tkinter"` works in the venv |
| 1 | "Write `usdcad/common.py` (section 6): HTTP with retries, `quarter_window`, heading conversion, PDF row reading by position, year-walking rule, checks, the `Result` type. Add unit tests for `quarter_window` and year-walking." | unit tests pass |
| 2 | "Write the TD reader (7.1) with separate `fetch` and `parse`, and `pull.py` that prints a table for the banks built so far." | `pull.py` prints TD's 4 values; they match the TD page |
| 3 | "Write the Scotia (7.2) and National Bank (7.3) readers." | values match the PDFs by eye |
| 4 | "Write the CIBC (7.4) and BMO (7.5) readers. Plain HTTP only; no Playwright yet." | all 5 banks print in under 30 s |
| 5 | "Add `pull.py --save-fixtures` and `--check`, and `tests/test_readers.py` using saved fixtures and the values I give you, including the trap tests from section 10." | tests pass offline (Wi-Fi off) |
| 6 | "Build `app.py` (section 2) with tkinter: pull in background threads, a row per bank updating as each finishes, Open source, Retry, editable values, Skip, live Average, and `USDCAD Forecasts.bat`." | the window works; a bank forced to fail shows ✗ with the note and accepts typed values |
| 7 | "Add Generate (section 4): `usdcad.csv`, `sources/`, `history.csv`, and the warnings from section 6." | output files look right; a changed value triggers the jump warning |
| 8 | (optional) "Add the Playwright fallback for BMO (7.5)." | BMO still pulls with plain HTTP disabled |

Then connect your report tools to `output/<date>/usdcad.csv`.

## 12. Later upgrades
- **Other pairs.** Checked in the same documents in September 2026:
  - EURUSD and GBPUSD are published by all five banks: TD page, CIBC FX PDF, Scotia PDF, National Bank
    `forex.pdf`, and BMO's "International Economic Outlook" (same site, same steps).
  - EURCAD is published directly only by BMO (`C$/€` row) and National Bank (`forex.pdf`). GBPCAD only by
    National Bank. The others would be calculated (EURUSD × USDCAD) and flagged, accurate to about ±0.01.
- **RBC** from its internal source.
- **Scheduled pull** (Windows Task Scheduler), so a broken reader is known before release day.

## Disclaimer
Not affiliated with any bank. The forecasts and documents belong to the banks that publish them. This tool
reads their public pages. It does not store or redistribute their documents in this repository.
