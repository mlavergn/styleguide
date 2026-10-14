# SQLSTYLE.md

Authoritative database schema and normalization conventions. All schema must conform.

## A. Critical Constraints

### A.1 Forbidden

- A.1.a. Never interpolate data into SQL — use parameterized queries (`?`) for all values.
- A.1.b. Never load bulk rows into application code to filter, search, sort, or aggregate — do it all in SQL.
- A.1.c. Never write migration code — all DDL is idempotent (`IF NOT EXISTS`) and the schema is the source of truth; databases are recreatable from source data.

## B. Architecture & File Organization

### B.1 Project Layout

- B.1.a. Star schema — normalized dimension tables surrounding a central fact table.

### B.3 Access Control & Visibility

- B.3.a. A single pre-joined `CREATE VIEW` is the primary read interface — application code queries the view, not raw tables. JOIN required dimensions, LEFT JOIN optional ones, and `COALESCE(col, '')` optional text so callers never see NULLs.

## C. Code Construction & Logic

### C.1 Logic Layout & Readability

- C.1.a. **Dimension tables (lookups)** — one concept per table, kept small. Autoincrement integer PK plus a single unique lookup column; optional metadata columns on the dimension itself.

```sql
CREATE TABLE IF NOT EXISTS countries (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  code TEXT NOT NULL UNIQUE,
  latitude REAL, longitude REAL
);
```

- C.1.b. **Fact table** — text PK from source data (e.g. stream ID). FK columns reference dimensions (`NOT NULL` for required, nullable for optional). Boolean state as `INTEGER NOT NULL DEFAULT 0` directly on the fact table, never separate join/flag tables. Raw/denormalized fields (URLs, free text) and classification columns stored inline with sensible defaults.

```sql
CREATE TABLE IF NOT EXISTS streams (
  id TEXT PRIMARY KEY,
  country_id INTEGER NOT NULL REFERENCES countries(id),
  category_id INTEGER REFERENCES categories(id),
  is_favorite INTEGER NOT NULL DEFAULT 0,
  media_type TEXT NOT NULL DEFAULT 'live'
);
```

- C.1.c. **Indexes** — index FK columns used in WHERE and columns used for sorting/grouping (e.g. `started_at`). Composite indexes for common query patterns, partial indexes for selective flags. No redundant indexes — if a composite covers a single-column query, don't add a separate single-column index.

```sql
CREATE INDEX IF NOT EXISTS idx_streams_country_category ON streams(country_id, category_id);
CREATE INDEX IF NOT EXISTS idx_streams_favorite ON streams(id) WHERE is_favorite = 1;
```

- C.1.d. **Supporting tables** — configuration as key-value (`key TEXT PRIMARY KEY, value TEXT NOT NULL`); activity/event tables with autoincrement PK, FK to the fact table, ISO 8601 timestamps as TEXT, durations as REAL. Supporting tables may FK the fact table when their lifecycle is tied to it.

## D. Systems & State Management

### D.2 Resource Management

- D.2.a. WAL journal mode; foreign keys enabled (`PRAGMA foreign_keys=ON`).
- D.2.b. `INSERT OR REPLACE` for upserts, `INSERT OR IGNORE` for idempotent inserts. Wrap bulk inserts in transactions.
