# Job Discovery – daily job search dashboard & Excel reports

A local web app for a MERN Stack Developer (Gurugram / Delhi NCR / remote-India) that, when you open the
dashboard or click **Find New Jobs**:

1. searches every configured, permitted job source,
2. scores each listing against your profile (transparent 0–100 score),
3. filters, ranks and de-duplicates across platforms,
4. shows results in a sortable, filterable dashboard with a direct **Open Job** link per job,
5. generates a real `.xlsx` report (`job_report_YYYY-MM-DD_HH-mm.xlsx`),
6. remembers every job it has seen (SQLite), so old listings are not reported as new.

> **Discovery only.** The app never applies to jobs, submits resumes or forms, messages recruiters, creates
> accounts, logs in to job portals or changes anything on external sites. It only shows links – you apply manually.

---

## Integration status (honest)

| Source | Status | What you need |
|---|---|---|
| **Remotive** (public API) | ✅ Works out of the box – verified live | Nothing. One request per search, max ~4/day (cooldown enforced). Currently returns ~10 recent software jobs per call. |
| **Remote OK** (public API) | ✅ Works out of the box – verified live | Nothing. Attribution + link back is shown, as its terms require. |
| **Greenhouse / Lever / Ashby** employer boards (public job-board APIs) | ✅ Implemented, verified against live Greenhouse & Ashby endpoints | Company board slugs in `.env` |
| **Adzuna** (official API, India) | ⚙️ Implemented, needs credentials (not tested live – no key) | Free `ADZUNA_APP_ID` / `ADZUNA_APP_KEY` |
| **JSearch** via RapidAPI (authorized Google-for-Jobs aggregator: LinkedIn, Naukri, Indeed, Glassdoor, company sites) | ⚙️ Implemented, needs credentials (not tested live – no key) | `JSEARCH_RAPIDAPI_KEY` |
| **Jooble** (partner API) | ⚙️ Implemented, needs credentials (not tested live – no key) | `JOOBLE_API_KEY` |
| **LinkedIn, Naukri, Indeed, Wellfound, Foundit, Cutshort, Instahyre, Hirist** | 📝 **Manual import required** | No permitted public search API. Paste job links / CSV in the **Import** tab, or let JSearch bring in listings published on them. |

The app never marks a source as searched unless the request actually succeeded. Failures are shown per source,
the other sources keep working, and a completely failed search is reported as **FAILED** – never as "0 jobs".

The Adzuna, JSearch and Jooble adapters follow those providers' documented response formats and are covered by
fixture-based tests, but they could not be exercised against the real services without keys. If a provider has
changed its format, the source card and the search summary will show the error.

---

## Project structure

```
nainshi/
├── .env.example              # copy to .env – all secrets stay on the backend
├── package.json              # root scripts (setup, dev, test, build, start)
├── server/                   # Node.js + Express REST API
│   ├── src/
│   │   ├── index.js          # starts the API (127.0.0.1:4000)
│   │   ├── app.js            # Express app (helmet, rate limits, timeouts, static client)
│   │   ├── config/           # env loading, default candidate profile
│   │   ├── db/               # SQLite connection, migrations/*.sql, migrate.js
│   │   ├── providers/        # one adapter per job source (BaseProvider interface)
│   │   ├── services/         # normalize, salary, experience, location, skills,
│   │   │                     # matcher, dedup, jobRepository, search, import, export
│   │   ├── routes/api.js     # REST endpoints + validation (zod)
│   │   └── middleware/       # errors, request timeout
│   └── test/                 # Vitest + Supertest (mock data only)
├── client/                   # React + Vite + Bootstrap dashboard
│   └── src/
│       ├── App.jsx           # tabs: Dashboard, Sources, Import, History, Profile
│       ├── components/       # Dashboard, JobTable, Filters, StatCards, ...
│       ├── hooks/useSearch.js# search lifecycle (auto-search on open, polling)
│       └── api.js            # REST client (no secrets)
├── data/                     # SQLite databases (created automatically, git-ignored)
└── reports/                  # copy of every generated Excel report (git-ignored)
```

---

## Install & run

Requirements: **Node.js 20+** (tested with Node 22 on Windows). No global tools needed.

```powershell
cd C:\Users\nainc\OneDrive\Desktop\nainshi
npm run setup            # installs root, server and client deps + creates the database
copy .env.example .env   # optional – the app runs without it (Remotive + Remote OK only)
```

### Daily use (development mode – two processes, hot reload)

```powershell
npm run dev
```

- Dashboard: **http://127.0.0.1:5173**
- API: http://127.0.0.1:4000 (the dashboard proxies `/api` to it)

Or start them separately in two terminals:

```powershell
npm run dev:server   # API on :4000
npm run dev:client   # dashboard on :5173
```

### Single process (production build)

```powershell
npm start            # builds the client, then serves dashboard + API on http://127.0.0.1:4000
```

### Database

Tables are created automatically on start. To run migrations explicitly:

```powershell
npm run db:migrate
```

Migrations live in `server/src/db/migrations/NNN_name.sql` and are applied once each (tracked in `schema_migrations`).

---

## Daily workflow

1. Open the dashboard. If your last search is older than the auto-search interval (default 60 min), a search
   starts automatically; otherwise you see the latest results and the reason no new search ran.
2. Click **Find New Jobs** to search right now. The button is disabled while a search runs; the server allows only
   one search at a time, so refreshing the page or double-clicking simply re-attaches to the running search.
3. Filter by role, location, work mode, salary range, disclosed/undisclosed salary, score, source, category,
   status and new vs previously discovered. Click column headers to sort.
4. Click a title for the full score breakdown, every source link and the description. **Open Job** opens the real
   posting in a new tab (`rel="noopener noreferrer"`).
5. **Download Excel** for the report. A copy is also saved in `reports/`.

---

## Matching rules

Score = sum of five transparent components (shown for every job):

| Component | Max | Rule |
|---|---|---|
| Skills | 40 | 7 points per primary skill found, 3 per additional skill. "MERN" implies MongoDB/Express/React/Node. |
| Title | 25 | Target role or core stack in title = 25; frontend/backend/web dev = 16; generic software engineer = 14. Titles led by a different stack (Java, .NET, PHP, Python, QA, DevOps …) = 0. |
| Experience | 15 | Range includes your 2.75 yrs = 15; slightly above = 9; aimed at juniors = 4–8; well above = 0; not stated = 7 (lower if the title says Senior/Lead). |
| Location | 10 | Gurugram/Gurgaon/Delhi/Noida/Faridabad/Ghaziabad or remote open to India = 10; remote with unspecified region = 7; other Indian city = 2; abroad or region-restricted remote = 0. |
| Salary | 10 | Overlaps ₹8–12 LPA = 10; above = 9; slightly below = 4; below = 0; **not disclosed = 5 (neutral)**. |

Categories:

- **Strong match** – score ≥ 70, experience fits, location compatible, no missing key information.
- **Possible match** – relevant (score ≥ your minimum, default 40) but not strong.
- **Missing info / review required** – relevant, but experience, location eligibility or a usable description is missing.

Salary handling understands `₹8-12 LPA`, `8 - 12 Lacs P.A.`, `8L - 12L`, `₹8,00,000 - 12,00,000 a year`,
`₹50,000 - 80,000 per month` (×12), `50k-80k`, `1.2 Cr`, `Up to 12 LPA`. It never guesses: foreign currencies are
shown as listed but not converted; INR amounts without a pay period below ₹1 lakh are not converted; provider
*estimated* salaries (e.g. Adzuna's `salary_is_predicted`) are treated as **Not disclosed**.

Notice period is your preference, not an employer guarantee: if a listing says "Immediate joiners" or "up to 7 days",
the job is kept and a note is added.

Change any preference in the **Profile** tab – all stored jobs are re-scored immediately.

---

## Duplicate detection & job status

A listing is matched to an existing job, in this order, by:

1. the same source + source job ID,
2. the same canonical URL (tracking parameters removed; LinkedIn `/jobs/view/<id>`, Indeed `viewjob?jk=` normalized),
3. an alternate apply link already known,
4. **cross-platform**: same normalized company (Pvt/Ltd/Inc stripped) + same normalized title + same location
   (Gurgaon = Gurugram; remote = remote) **and** no conflicting salary, experience or posting dates (> 45 days apart).

Two listings from the same provider with different IDs are never merged (they are separate openings). Merged jobs keep
every platform link ("Alternate source URLs").

Discovery status per search: **New**, **Previously discovered**, **Previously discovered – changed** (content hash of
that source's listing changed).

Job status:

- **Listed** – returned by a source API at the shown time. *This does not guarantee the job is still open.*
- **Closed (verified)** – only for employer boards (Greenhouse/Lever/Ashby): the board was fetched successfully and
  the job is no longer on it.
- **Unverified** – manually imported; the app never opens or checks the link.

---

## Excel report

`GET /api/jobs/export` (or the **Download Excel** button) creates `job_report_YYYY-MM-DD_HH-mm.xlsx`:

- **New Jobs** – jobs first discovered in the latest search.
- **All Relevant Jobs** – every stored relevant job (one row per de-duplicated vacancy).
- **Search Summary** – time, sources searched/failed/not searched, counts (fetched, relevant, new, changed,
  duplicates, skipped, closed), errors and source attributions.

Columns include title, company, location, work mode, salary (labelled ₹ LPA or as listed), experience (years),
matched skills, source, posting date, discovery date, score, category, explanation, notice period, **clickable direct
job URL**, alternate URLs, discovery and verification status. Headers are frozen, styled and filterable. Missing values
read "Not disclosed" / "Unknown". If nothing is new, or the search failed, the sheet says so explicitly. History rows
can be exported per search from the **History** tab (`?runId=`).

---

## Configuring legitimate data sources

Edit `.env` (project root) and restart the API. Check the **Sources** tab afterwards.

**Adzuna** – register at <https://developer.adzuna.com/>, create an app, set `ADZUNA_APP_ID`, `ADZUNA_APP_KEY`.
`ADZUNA_COUNTRY=in`. Each search uses up to `ADZUNA_MAX_REQUESTS` calls (one per preferred city + one remote).

**JSearch (RapidAPI)** – subscribe at <https://rapidapi.com/letscrape-6bRBa3QguO5/api/jsearch> (free tier available),
set `JSEARCH_RAPIDAPI_KEY`. This is the legitimate way to receive listings that are published on LinkedIn, Naukri,
Indeed and Glassdoor: the job's publisher becomes the "Source" and every apply option is kept as an alternate link.

**Jooble** – request a key at <https://jooble.org/api/about>, set `JOOBLE_API_KEY` (and `JOOBLE_API_HOST` if your key
was issued for a country site).

**Employer career boards** – many companies host jobs on an ATS with an official public job-board API:

- Greenhouse: careers URL `https://boards.greenhouse.io/<slug>` or `job-boards.greenhouse.io/<slug>` → `GREENHOUSE_BOARDS=<slug>`
- Lever: `https://jobs.lever.co/<slug>` → `LEVER_COMPANIES=<slug>`
- Ashby: `https://jobs.ashbyhq.com/<slug>` → `ASHBY_BOARDS=<slug>`

Use comma-separated lists; add a display name with a colon: `GREENHOUSE_BOARDS=acme,globex:Globex Corp`.
These are complete listings, so jobs removed by the employer are marked **Closed (verified)**.

**Remotive / Remote OK** – enabled by default; disable with `REMOTIVE_ENABLED=false` / `REMOTEOK_ENABLED=false`.
Cooldowns (`*_MIN_INTERVAL_MINUTES`) respect their usage terms; Remotive cannot go below 180 minutes.

**LinkedIn, Naukri, Indeed, Wellfound, Foundit, Cutshort …** – no public, permitted job-search API for candidates.
Their terms prohibit scraping, so the app does not contact them. Use the **Import** tab:

- *Add one job link*: title, company and the URL of a **specific** posting (e.g. `linkedin.com/jobs/view/<id>`,
  `naukri.com/job-listings-…`, `indeed.com/viewjob?jk=…`). Search pages and homepages are rejected.
- *Import CSV*: header row with `title, company, url` (required) and optionally `location, salary, experience,
  skills, description, posted date, work mode, platform`. Up to 500 rows per import.

### Adding another provider

1. Create `server/src/providers/myProvider.js` extending `BaseProvider` and implement
   `searchJobs(criteria, { fetchJson, signal, timeoutMs })` → `{ rawJobs, warnings?, completeScopes? }`,
   `normalizeJob(raw)` → fields for `finalizeJob()` (return `null` to skip), and optionally override
   `isConfigured()` / `configHint()` / `getSourceStatus()`.
2. Register it in `server/src/providers/index.js` and add any keys to `.env.example`.
3. Add a fixture test in `server/test/providers.test.js`.

---

## REST API

| Method & path | Purpose |
|---|---|
| `GET /api/health` | Server, DB and demo-mode status |
| `GET /api/profile` / `PUT /api/profile` | Read / update preferences (validated; partial updates merged; jobs re-scored) |
| `GET /api/sources` | Every source with state: ready, not_configured, disabled, cooldown, manual_import_required |
| `POST /api/jobs/search` `{ "trigger": "manual" \| "auto" }` | Start a search → `202`; `409` if one is running; `auto` may be skipped (`200`, `started:false`) |
| `GET /api/jobs/search/status` | Running search progress or the latest search |
| `GET /api/jobs` | List with `q, role, location, workMode, salaryMin, salaryMax, salaryDisclosed, minScore, source, discovery, category, status, sort, order, page, pageSize` |
| `GET /api/jobs/stats` | Dashboard counters |
| `GET /api/jobs/:id` | Job details incl. description and all source links |
| `GET /api/jobs/export[?runId=]` | Excel report download |
| `POST /api/jobs/import` `{ "csv": "..." }` or `{ "jobs": [...] }` | Manual import |
| `GET /api/search-runs`, `GET /api/search-runs/:id` | Search history |

All SQL is parameterized; inputs are validated with zod. Rate limits: 1500 API requests / 5 min, 10 searches / 5 min,
30 imports / 5 min. Requests time out after 30 s (export 60 s); each provider call after `PROVIDER_TIMEOUT_MS`;
a whole search after `SEARCH_TIMEOUT_MS`.

---

## Tests

```powershell
npm test               # server (Vitest + Supertest) and client (Vitest + Testing Library)
npm run test:server
npm run test:client
```

Covered: profile load/edit/validation, normalization for every provider, salary/experience/location parsing,
match scoring and categories, salary/location filters, duplicate detection (and *not* merging different vacancies),
change detection, verified closures, failed/partial/timeout provider handling, no-source and empty searches,
concurrent-search prevention, rate limiting, SQLite persistence across restarts, Excel sheets/hyperlinks/frozen
headers/filters/empty and failed reports, CSV/JSON import validation, dashboard loading, filtering, sorting, safe links
and auto-search-once behaviour. Mock jobs exist only inside the test files.

### Demo mode

`DEMO_MODE=true` uses a **separate database** (`data/demo.sqlite`) and a demo provider with fake jobs
(companies prefixed `[DEMO]`, links to `example.com`). The dashboard shows a red DEMO banner and reports are named
`DEMO_job_report_…`. Demo jobs can never appear in real reports.

---

## Privacy

- Everything is stored locally in SQLite (`data/jobs.sqlite`). Reports are saved in `reports/`.
- API keys live only in `.env` on the backend; the React app never receives them.
- Your profile is used only for local matching. No resume is uploaded anywhere. No job-portal passwords are stored.
- The servers bind to `127.0.0.1` (not reachable from other machines).
- No resume PDF was found in the project, so the profile was seeded from the details you provided
  (`server/src/config/defaultProfile.js`). Edit it in the **Profile** tab.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `Cannot find native binding` / `@rolldown/binding-…` when running tests or Vite | npm optional-dependency bug. Delete `node_modules` and `package-lock.json` in that folder (`server` or `client`) and run `npm install` again. |
| `better-sqlite3` fails to install | Use Node 20/22 LTS (prebuilt binaries). Otherwise install "Desktop development with C++" (Visual Studio Build Tools) and reinstall. |
| Dashboard shows "Cannot reach the backend" / Vite logs `ECONNREFUSED 127.0.0.1:4000` | The API isn't running or uses another port. Start `npm run dev:server`; if you changed `PORT`, run the client with `VITE_API_TARGET=http://127.0.0.1:<port>`. |
| `EADDRINUSE` on 4000 or 5173 | Another instance is running. Close it, or `netstat -ano | findstr :4000` then `taskkill /PID <pid> /F`. |
| A source shows **Needs API credentials** | Add its keys to `.env` (project root, not `server/`) and restart the API. |
| Source fails with `HTTP 401/403` | Wrong or inactive key / plan. `HTTP 429`: provider rate limit – try later or lower `*_MAX_REQUESTS`. `HTTP 404` on an employer board: wrong slug. |
| Remotive / Remote OK show **Cooldown** | Intentional – their terms ask for infrequent polling. Earlier results stay in your list. |
| "Automatic search waits 60 min" | Opening the dashboard doesn't re-search if you searched recently. Click **Find New Jobs**, or change the interval in Profile. |
| "A job search is already running" | Only one search at a time. The dashboard re-attaches to it automatically. |
| Search status **Failed** / **No source searched** | See the per-source errors on the dashboard or in the Search Summary sheet. It is *not* a zero-results search. |
| Few results | Public remote APIs are small (Remotive ≈ 10 jobs per call). Configure Adzuna/JSearch and employer boards for India-specific coverage, or lower *Min match score* in Profile. |
| Database locked / sync conflicts | The project is inside OneDrive. If OneDrive sync interferes, pause sync or set `DATABASE_PATH` to a folder outside OneDrive (e.g. `C:/jobdata/jobs.sqlite`). |
| Start fresh | Stop the server and delete `data/jobs.sqlite*`. |
#   j o b - a u t o m a t i o n  
 