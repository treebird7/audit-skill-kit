---
name: sql-review
description: Review SQL migrations and queries in ANY repo for safety, RLS correctness, function security, concurrency, blanket grants, and query quality. DB/ORM-agnostic. Reports ✅/⚠️/❌ per check; optional --gold emits verified findings as portable training pairs. Use when reviewing a .sql migration, an ORM schema (Prisma/Drizzle/Kysely), or a raw query — especially before deploying schema changes to a multi-tenant or user-data app.
---

# /sql-review — portable SQL & schema review

Runs static checks against SQL (or ORM schema/query code) and reports `✅ / ⚠️ / ❌` per check.
**No ecosystem dependencies** — detects DB and ORM from the repo, never assumes a knowledge tree,
training pipeline, or a specific provider.

## Usage

```
/sql-review <path>            # review a file (.sql migration, schema.prisma, *.ts query module)
/sql-review                   # review SQL/schema changed in the current session or `git diff`
/sql-review <path> --live     # also diff against a reachable DB (psql/$DATABASE_URL) if available
/sql-review <path> --gold     # append verified findings to .audit/gold-pairs.jsonl (see README)
```

## Step 0 — detect the stack (do this first, cheaply)

```bash
ls prisma/schema.prisma drizzle.config.* 2>/dev/null      # ORM?
grep -rilE "createPolicy|enable row level|USING \(|\$queryRaw|\$executeRaw" . | grep -v node_modules | head
```
- Pure `.sql` with DDL (`CREATE/ALTER/DROP/CREATE POLICY/CREATE FUNCTION`) → **migration mode**.
- Prisma/Drizzle schema → translate models to the equivalent table checks; RLS lives in raw SQL
  migrations, so flag tenant tables that have **no** corresponding `ENABLE ROW LEVEL SECURITY`.
- Inline query / snippet → **query mode** (Checks 6 + applicable RLS/function checks).
- Note the dialect (Postgres/MySQL/SQLite). Some checks are Postgres-specific — mark `⚪ skipped` elsewhere.

## Severity

| Icon | Level | Meaning |
|------|-------|---------|
| ❌ | error | Correctness, security, or data-loss risk. Fix before deploy. |
| ⚠️ | warning | Likely issue or heuristic concern. Review. |
| ℹ️ | info | Hygiene / convention. |
| ⚪ | skipped | Not applicable (wrong dialect, no live conn, etc.). |

---

## Check 1 — migration_safety
- ❌ `ALTER TABLE ... DROP COLUMN` / `ALTER COLUMN TYPE` with no `-- ROLLBACK:` note (irreversible + table rewrite).
- ❌ Non-IMMUTABLE function (`now()`, `CURRENT_DATE`, `current_setting()`) in an index expression (apply fails) or `CHECK` constraint (write-time-only trap → move to a `BEFORE` trigger).
- ❌ Adding a `UNIQUE` constraint when existing `INSERT` paths elsewhere lack `ON CONFLICT` / unique-violation handling → those paths start throwing raw constraint errors. Grep other `INSERT INTO <table>` sites.
- ⚠️ Missing idempotency guards: `CREATE TABLE`/`INDEX` without `IF NOT EXISTS`; `DROP ...` without `IF EXISTS`; `CREATE FUNCTION` that could be `CREATE OR REPLACE`.
- ⚠️ `NOT NULL` added to an existing column with no `DEFAULT` (lock + fails on existing rows); new constraint without `NOT VALID` then `VALIDATE` (validation scan locks).
- ⚠️ `CREATE INDEX` without `CONCURRENTLY` on a large/hot table (write lock).
- ℹ️ File not named `<timestamp>_<description>`; missing `-- ROLLBACK:` / `-- Description:` header.

## Check 2 — rls_correctness (Postgres; the highest-value check for multi-tenant apps)
- ❌ `CREATE TABLE` on a client-reachable/tenant table with no `ENABLE ROW LEVEL SECURITY` in the same migration.
- ❌ Single `FOR ALL` policy — split into per-operation (SELECT/INSERT/UPDATE/DELETE); they have different predicates.
- ❌ Policy derives identity from a **request body / function parameter / variable** instead of the session identity (`auth.uid()`, `current_setting('app.user_id')`, JWT claim).
- ⚠️ `INSERT`/`UPDATE` policy missing `WITH CHECK` (the row can be written into a state the policy would forbid reading).
- ⚠️ `USING (true)` / `USING (1=1)` on a tenant/sensitive table → any caller reads every row.
- ⚠️ Authenticated-but-not-owner-scoped: `USING (auth.role() = 'authenticated')` / `USING (auth.uid() IS NOT NULL)` on a per-user table → any logged-in user reads all rows. Needs an owner predicate.
- ⚠️ Permissive UPDATE policy on a **shared/multi-party row** coexisting with a table-wide `UPDATE` grant: RLS scopes rows, not columns → a non-owner who satisfies the row predicate can rewrite *any* column. Fix with column-level `GRANT UPDATE(col,…)`.
- ⚠️ Narrowing a content table while leaving its **association/identity** tables (likes, follows, handles) at `USING (true)` → the relationship is reconstructable by JOIN. Treat them as one anonymity surface.
- ⚠️ `auth.uid()` called inline instead of cached `(SELECT auth.uid())` (perf at scale); missing index on RLS-predicate columns.
- ℹ️ Policy name off-convention (`<table>_<op>_<scope>`).

## Check 3 — schema_drift (`--live` only)
If a DB is reachable (`$DATABASE_URL` + `psql`, or the ORM's introspect): compare declared columns/
types/constraints/**policies + grants** against deployed. Flag:
- ❌ A tightening migration whose `DROP POLICY IF EXISTS "<name>"` doesn't match the **live** policy name → silent no-op, the broad policy survives (permissive-OR keeps the table open). Verify live names before apply.
- ⚠️ Declared-vs-deployed column/type/nullable drift; grants broader live than the migration implies.

## Check 4 — function_security (any `CREATE [OR REPLACE] FUNCTION`)
- ❌ `SECURITY DEFINER` function with no `SET search_path = ...` (search-path hijack → privilege escalation).
- ❌ `SECURITY DEFINER` that takes a target id as a **parameter** and acts on it with no `caller == target` (or service-role) check → cross-user read/write (RLS is bypassed by definer rights).
- ⚠️ Dynamic SQL (`EXECUTE format(...)`) interpolating inputs without `%I`/`%L` quoting → injection.
- ℹ️ Definer function with no comment stating *why* it needs elevation.

## Check 5 — grant_hygiene
- ⚠️ `GRANT ... ON ALL TABLES IN SCHEMA public TO anon, authenticated` (future backend-only tables auto-exposed).
- ⚠️ `ALTER DEFAULT PRIVILEGES ... GRANT ... TO anon/public` (every future object becomes reachable).
- ⚠️ `GRANT ... TO PUBLIC` (every role inherits, incl. service/admin).
- ❌ `ALTER TABLE ... DISABLE ROW LEVEL SECURITY` / `NO FORCE` on a client-exposed table.

## Check 6 — query_review (raw queries / ORM `$queryRaw` etc.)
- ❌ String-concatenated/interpolated user input into SQL → injection. Require parameterized (`$1`, prepared, tagged `sql\`\``).
- ⚠️ Tenant table queried with no tenant predicate in app code (the "forgot the `where therapistId`" leak). For ORMs, flag `findUnique/findMany` on a tenant model with a `where` that lacks the owner key.
- ⚠️ `SELECT *` returning encrypted/sensitive columns into a response unfiltered; N+1 in a loop; missing `LIMIT` on an unbounded user-facing list.

---

## Output

For each applicable check, one line: `❌/⚠️/ℹ️/⚪ <check> — <finding> (file:line)`. Then a short
summary: counts by severity + the top 3 must-fix. **Do not** invent findings to fill the report;
`✅ <check> — clear` is a valid and useful result.

## Verify before you assert (and before you emit a gold pair)

Every ❌/⚠️ must be confirmed against the **actual source line**, not pattern-matched in the abstract.
Read the surrounding code; check whether a guard exists elsewhere (a later `OR REPLACE`, a wrapper, an
app-level scope). If a suspected finding turns out safe, record it as `clear` (and, with `--gold`, as
a `false_alarm_of` negative example). Always review the **latest/deployed** definition of a redefined
object, not the first occurrence.

## `--gold` emission

When `--gold` is set, after verification append one JSON line per **verified** finding (and per
verified `clear`/dismissed claim) to `.audit/gold-pairs.jsonl`, matching the schema in the kit README.
Redact any secret values in `snippet` (keys, tokens, connection strings → `‹redacted›`). Stamp `repo`
from `git remote get-url origin` (best-effort) and `commit` from `git rev-parse --short HEAD`.
