# RISTT — KCC Timetable Viewer

A single-file, dependency-free web viewer for **aSc Timetables** XML exports, built for Raha International School's Khalifa City Campus (2026–2027).

Drop the HTML file and an XML export side by side, serve the folder, and staff get a browsable timetable with class / teacher / subject views, Week A–B filtering, a teacher allocation report and CSV export. There is no build step, no framework and no server-side code.

---

## ⚠️ Before you publish this repository

`secondary-timetable.xml` contains a `<students>` block with **94 real student records** — first name, last name, full name and class assignment. The viewer ignores those records, but **GitHub Pages serves the XML file verbatim**, and anyone can fetch it directly. Committing it also puts those names in git history permanently.

Remove the block before the first push (PowerShell):

```powershell
$p='secondary-timetable.xml'; $s=Get-Content $p -Raw; $s=[regex]::Replace($s,'<students\b.*?</students>','<students/>','Singleline'); [IO.File]::WriteAllText((Resolve-Path $p),$s,(New-Object Text.UTF8Encoding $false))
```

This removes all 94 records and leaves every timetable entity untouched — cards, lessons, teachers, classes and groups all still parse identically. `primary-timetable.xml` already has an empty `<students>` block and needs no change, and no `studentids` attributes are populated in either file.

If the file has already been pushed, rewriting history is not sufficient on its own — treat the data as disclosed and follow your school's data-protection process.

---

## Repository layout

```
index.html               # the entire application: markup, CSS and JS in one file
secondary-timetable.xml  # default Secondary source (aSc export)
primary-timetable.xml    # default Primary source (aSc export)
README.md
```

| Source | Periods | Classes | Teachers | Subjects | Lessons | Cards |
|---|---|---|---|---|---|---|
| Secondary | P1–P6 | 36 (G6–G10, DP1, DP2, Careers Programme) | 76 | 52 | 526 | 1275 |
| Primary | P1–P10 | 50 (EY1/EY2, Mini, G1–G5) | 61 | 24 | 582 | 2263 |

---

## Running it

The viewer loads its XML with `fetch()`, which browsers block on `file://`. **Opening `index.html` by double-clicking will not auto-load the timetable** — you will get a "not found" status and have to use the manual file picker.

Serve the folder over HTTP instead:

```bash
python -m http.server 8000
```

Then open <http://localhost:8000>.

### GitHub Pages

Push the three files to a repository and enable Pages (Settings → Pages → *Deploy from a branch*, root of `main`). Keep `index.html` and the XML files in the **same directory** — the fetch paths are relative.

---

## How it works

`parseAscXml()` reads the aSc export with `DOMParser` and flattens it into a single `events` array. One aSc `<card>` becomes one event **per day it runs on**, joined to its `<lesson>` for subject, teacher, class, group and room IDs, which are then resolved to display names.

```
<cards>/<card> ──lessonid──> <lessons>/<lesson> ──ids──> subjects, teachers,
                                                          classes, groups, classrooms
        │
        └─ days="00001" ─> expandDays() ─> one event per weekday
```

Everything downstream — the grids, the dashboard, the allocation report and the evidence blocks — is a filter or aggregation over that one array. Switching sources re-parses and caches per source in `DATA_CACHE`.

### Interpretation rules worth knowing

- **Week A/B weighting.** In the allocation report a card that runs only in Week A or only Week B counts as **0.5** toward the weekly total (`eventWeight`). A card running every week counts as 1.
- **Meetings are excluded** from the dashboard's lesson counts and subject chips (`isMeeting`) — "Dept Meeting" in the Secondary export, "Grade Level Meeting" in the Primary one.
- **Friday times are division-specific.** A Friday card reports its Friday time, not the Monday–Thursday time the XML carries against that period number. This feeds the CSV export and copy-summary as well as the grid.
- **"Weighted periods"** on the dashboard counts every card with Week A/B at 0.5, so it agrees exactly with the Teacher allocations tab. "Number of lessons" is the narrower figure that drops meetings, break and lunch — the two tiles are meant to differ, and their captions say how.
- **DP blocks are relabelled** from `Block 1…6` to `Block A…F` for display (`relabelDpBlock`), to avoid collision with aSc group numbering.
- **Student records are ignored** by the parser; only the count is reported.
- **Card colour** is a hue hashed from the subject ID, so a subject keeps a stable colour across views.
- **Break and lunch are not lessons.** Primary schedules them as subject cards (see below); `isNonTeachingSlot` renders those as muted slots and drops them from lesson counts and subject chips. They are left in the teacher allocation report, where they are currently a no-op — no teacher is assigned to one.

### Friday, and why the Primary period numbers skip

The two divisions shape Friday differently, and the difference is the single most confusing thing in the data.

**Secondary (MYP)** — 4 teaching periods, break is a real aSc `<break>` element so it consumes no period. Row order and period number coincide:

`P1 7:45–8:15` (30 min) · `P2 8:15–9:15` · `P3 9:15–10:15` · *Break 10:15–10:30* · `P4 10:30–11:30`

**Primary (PYP)** — 5 teaching periods of 35 minutes. PYP has **no aSc `<break>` elements at all**: break and lunch are scheduled as ordinary subject cards that *occupy a period*. So the 9:45 break **is aSc period 4**, and the last two lessons are aSc periods **5 and 6**:

`P1 8:00–8:35` · `P2 8:35–9:10` · `P3 9:10–9:45` · *Break = period 4, 9:45–10:00* · `P5 10:00–10:35` · `P6 10:35–11:10` · *Pack up 11:10–11:30*

That is why the Primary Friday grid runs P1, P2, P3, P5, P6 with no P4. The numbers shown are the aSc period numbers, so the grid, the CSV export and aSc itself all agree.

`FRIDAY_SCHEDULE` encodes this: each row carries an `ascPeriod` naming the aSc period it draws cards from, deliberately decoupled from the row's position. A `slot` row may also carry an `ascPeriod`, meaning it *absorbs* that period — which is how the Primary break claims period 4.

Monday–Thursday the same applies, and it varies by year group: Grades 1–5 break at period 4 and take lunch at period 8; Early Years breaks at periods 3 and 7 with no lunch card; Mini classes have no break or lunch card at all. Because it varies, these stay as cards rather than fixed grid rows — a fixed row would be wrong for a third of the school.

**If you add a period to a Friday, add a row to `FRIDAY_SCHEDULE`.** `fridayCoverageWarning()` checks every Friday aSc period that carries cards against the rows that claim one, and prints a warning in the status line and the browser console if any period is unclaimed. An unclaimed period renders nowhere — that is exactly how 52 Primary lessons were invisible before this check existed.

---

## Tabs

**Timetable viewer** — Pick Class, Teacher or Subject, then an entity. Renders two grids side by side: **Monday–Thursday** (with Mentor, Break and Lunch rows on Secondary) and a separate **Friday** grid using Friday's shortened bell times. Filter by Week A / B / All, toggle detailed vs compact cards, export the current view to CSV, copy it as text, or print (print CSS strips the chrome).

**Daily schedule** — Static reference tables of the bell schedule for both divisions, Mon–Thu and Friday.

**Teacher allocations** — Every teacher with a scheduled load: total weighted hours, hours per subject, hours per grade level with section breakdown, and rooms used. Search across teacher/subject/grade/room, filter by load band (≤16 / 17–24 / 25+), sort by name or hours, export to CSV. Multi-class cards count once toward the teacher total and are split proportionally across the grade breakdown.

**Secondary changelog** and **Secondary key points** — Secondary-only. The key-points tab renders "evidence blocks" that compute live from the loaded XML (Maths concurrency, DP2 Maths groups, staff load bands, DP1 Biology/Global Politics demand, language groups, ConnectEd/ToK slots, provisional-item checks). Buttons inside each block jump to the relevant class/teacher/subject in the timetable tab.

### Bell schedule, Monday–Thursday

Friday differs enough that it has [its own section above](#friday-and-why-the-primary-period-numbers-skip).

| | Secondary (MYP) | Primary (PYP) |
|---|---|---|
| P1 | 7:45–8:45 | 8:20–9:00 |
| P2 | 8:45–9:45 | 9:00–9:40 |
| Mentor | 9:45–10:00 | — |
| Break | 10:00–10:20 | timetabled: P4 (G1–5), P3 & P7 (EY) |
| P3 | 10:20–11:20 | 9:40–10:20 |
| P4 | 11:20–12:20 | 10:20–11:00 |
| Lunch | 12:20–13:00 | timetabled: P8 (G1–5) |
| P5 | 13:00–14:00 | 11:00–11:40 |
| P6 | 14:00–15:00 | 11:40–12:20 |
| P7–P10 | — | 12:20–14:50 |

Secondary breaks are aSc `<break>` elements and sit between periods. Primary breaks are cards inside a period, which is why they appear in the grid rather than as a separator row.

---

## Updating a timetable

1. Export from aSc Timetables as **aSc Timetables 2012 XML**.
2. Rename it to `secondary-timetable.xml` or `primary-timetable.xml`.
3. **Strip the student records** (see the warning above).
4. Replace the file in the repository and push.

The **Timetable source** selector also accepts alternative filenames — it tries each candidate in order until one loads:

- Secondary: `secondary-timetable.xml`, `timetable.xml`, `Secondary XML with CP.xml`, `Secondary V3 with CP.xml`
- Primary: `primary-timetable.xml`, `Primary timetable.xml`, `Primary XML.xml`, `Primary.xml`, `primary.xml`

For a one-off look at an export you do not want to commit, use **Manual XML load** — it parses in the browser and nothing is uploaded anywhere.

---

## Known limitations

These are current behaviours of the code, verified against the shipped XML files. Worth knowing before you trust a number.

- **Secondary period times are hardcoded** in `SECONDARY_PERIOD_TIMES` and override whatever the XML says. They currently match the shipped export exactly, so a future corrected export would be silently ignored.
- **Bell times live in three places** — the XML, the JS constants, and the static Daily schedule tables. Changing one does not change the others.
- **Evidence blocks match subjects by exact name** (`MATH`, `CONNECTED`, `Theory of Knowledge`, `Dept Meeting`, `MCore`) and a couple of teacher names. If an export renames a subject, the affected block silently renders empty rather than erroring.
- **Grade labels for Primary EY/Mini classes** fall back to the full class name (e.g. `EY1KR Hoopoe`) in the allocation table, because they do not start with a digit.
- **XML load failures are indistinguishable.** A malformed file and a missing file both report "not found", because the candidate loop swallows the parse error.
- The **Week A/B filter is shown for Primary**, which only defines "All weeks".
- CSV exports are not hardened against spreadsheet formula injection; a name beginning with `=`, `+`, `-` or `@` is written as-is.

---

## Browser support

Modern evergreen browsers. Uses `DOMParser`, `fetch`, optional chaining, `??`, CSS custom properties and CSS grid. The clipboard buttons require a secure context (`https://` or `localhost`).

## Licence

No licence file is present. Add one before making the repository public — without it, default copyright applies and others may not reuse the code.
