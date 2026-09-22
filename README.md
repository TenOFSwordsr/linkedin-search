# LinkedIn Dataset Search

A simple web application for searching and filtering a dataset of ~300 LinkedIn profiles.

- **Backend:** Go + Gin + SQLite (FTS5 full-text search, pure-Go driver)
- **Frontend:** Vanilla HTML/CSS/JS single page served by the same server
- **Focus:** search logic, backend structure, frontend-backend communication

## How to Run

### Option A - standalone executable (no dependencies)

`linkedin-search.exe` is a fully self-contained build: the dataset and the
frontend are embedded in the binary via `go:embed`. Just run it:

```bash
linkedin-search.exe
# open http://localhost:8080
```

- It creates its SQLite database (`linkedin.db`) in the working directory on first start.
- Environment variables: `ADDR` (listen address, default `:8080`), `DB_PATH` (database file, default `linkedin.db`).
- Rebuild from source: `CGO_ENABLED=0 go build -trimpath -ldflags "-s -w" -o linkedin-search.exe ./cmd/server`

### Option B - from source

Requirements: Go 1.22+ (Python only needed to regenerate the dataset file).

```bash
go run ./cmd/server
# or
go build -o server.exe ./cmd/server && ./server.exe
```

## Architecture

```
browser (web/static)          Go backend
+-------------------+        +--------------------------------------+
| index.html        |  HTTP  | cmd/server/main.go                   |
| app.js  ----------+------->|   internal/api/router.go   (Gin)     |
| style.css         |  JSON  |   internal/store/store.go  (SQLite)  |
+-------------------+        |   internal/store/search.go (queries) |
                             |   internal/model/profile.go (types)  |
                             +--------------------------------------+
```

- **Storage design (SQL):** three tables -
  - `profiles` - one row per profile; list fields (skills, emails...) stored as JSON text
  - `education`, `experience` - flattened child rows for school/company search
  - `profiles_fts` - SQLite FTS5 virtual table indexing name, title, industry, company, location, summary, and skills for the keyword search
- **Indexing:** the FTS table plus indexes on `job_title`, `industry`, `country`, `company` are created at startup; data is loaded from `data/profiles.json`.
- **Data pipeline:** the raw CSV export has broken rows (unquoted commas / dropped empty fields). `scripts/normalize_data.py` parses the Python-literal list columns and flattens nested structures; a repair pass realigns ragged rows via anchor matching and sanity scoring. The final cleaned dataset ships as `data/profiles.json`.

## API

### `GET /api/profiles` - search + filter + paginate

| Parameter    | Type   | Meaning                                              |
|--------------|--------|------------------------------------------------------|
| `q`          | string | keyword search (OR of prefix terms) over all indexed fields |
| `skill`      | string | exact skill match (e.g. `leadership`)                 |
| `job_title`  | string | substring match on current job title                  |
| `industry`   | string | substring match on industry                           |
| `country`    | string | exact match on country                                |
| `company`    | string | substring match on current company or any experience entry |
| `school`     | string | substring match on any education entry (school/degree/major) |
| `min_years` / `max_years` | number | filter on years of experience |
| `page`, `page_size` | int | pagination (default 20, max 100)              |
| `sort_by`    | string | `relevance` (bm25 when `q` set), `name`, `connections`, `years` |
| `sort_order` | string | `asc` / `desc`                                        |
| `facets`     | bool   | `true` includes facet counts in the response          |

Response: `{ total, page, page_size, profiles: [...], facets? }`

### `GET /api/profiles/:id` - one profile (with education/experience)

### `GET /api/facets` - facet counts for the current filter set (skills, industry, country, company)

## Search & Filter Logic

- **Keyword search (`q`)** uses SQLite FTS5. The input is split into terms and rewritten to `"term1"* OR "term2"*` so any term can match and prefix matching works ("develop" matches "developer"). Ranking uses `bm25` when sorting by relevance.
- **Filters** are AND-combined SQL conditions:
  - `skill` matches a whole array element via `json_each` (so `java` never matches `javascript`)
  - `job_title` / `industry` / `company` / `school` use substring matching
  - `country` is exact
  - year bounds are numeric range checks
- **Facets** count values of skills/industry/country/company over the currently filtered result set, so the UI can show a live breakdown and one-click drill-down.
- The frontend keeps filters in sync, paginates server-side, and shows profile details in a dialog fetched from the detail endpoint.

## Project Layout

```
cmd/server/          entry point
internal/api/        Gin routes + param parsing
internal/store/      SQLite schema, seeding, search queries
internal/model/      shared types
internal/assets/     embedded dataset + frontend (go:embed)
web/static/          frontend source (index.html, app.js, style.css)
data/                profiles.json source (seed data)
scripts/             dataset normalization script
```

The dataset and frontend are duplicated under `internal/assets/` so they
embed into the executable; after editing `data/` or `web/static/`, re-copy
them into `internal/assets/` (or rebuild with a sync step) and rebuild the exe.
