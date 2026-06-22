# audit-kit — portable code-review skills

Two self-contained review skills that run against **any** repo, with **no ecosystem dependencies**
(no shared knowledge tree, no training pipeline, no Supabase/treebird assumptions). They detect the
DB/ORM and language from the repo itself.

- **`sql-review`** — SQL migrations & queries: migration safety, RLS correctness, function security,
  concurrency, blanket grants, query review. DB-agnostic (raw SQL, Prisma, Drizzle, Kysely, etc.).
- **`privacy-review`** — privacy/security moral inventory for apps holding sensitive user data:
  stated-vs-actual gap, identity-source triage, encryption posture, identity linkage, reachability.

## Install (drop into any repo)

```bash
cp -r audit-kit/sql-review     <target-repo>/.claude/skills/sql-review
cp -r audit-kit/privacy-review <target-repo>/.claude/skills/privacy-review
# then in that repo:  /sql-review <file>   |   /privacy-review
```

Or keep them user-global: `cp -r audit-kit/* ~/.claude/skills/`.

## Optional: gold pairs (`--gold`)

Both skills accept `--gold`. When set, each **verified** finding (and each verified `clear`) is
appended as one JSON line to `.audit/gold-pairs.jsonl` in the reviewed repo. These are
review-decision training examples — portable, schema-documented, and yours to collect.

> "Send back as a collaborator": after a review, hand the `.audit/gold-pairs.jsonl` file back (PR it,
> attach it, or `gh gist`). It carries no secrets — only the snippet, the verdict, and the fix. The
> emitting skill MUST redact secret values from `snippet` (keys, tokens, connection strings → `‹redacted›`).

### Gold-pair schema (`.audit/gold-pairs.jsonl`, one object per line)

```jsonc
{
  "schema": "audit-kit/gold-pair@1",
  "skill": "sql-review",            // or "privacy-review"
  "check": "rls_correctness",       // the check id that produced it
  "severity": "error",              // error | warning | info | critical | high | medium | clear
  "repo": "owner/name",             // best-effort from `git remote`; "" if none
  "commit": "81b2c7b",              // short HEAD at review time
  "path": "prisma/schema.prisma:42",// file:line (or file: if line unknown)
  "lang": "sql",                    // detected language/ORM tag
  "snippet": "CREATE TABLE ...",    // the reviewed code, secrets redacted
  "finding": "table created without ENABLE ROW LEVEL SECURITY",
  "fix": "ALTER TABLE x ENABLE ROW LEVEL SECURITY; + per-op policies",
  "verified": true,                 // ONLY true findings are emitted (see skill: verify before write)
  "false_alarm_of": null            // if set, names a dismissed claim — a negative example
}
```

**Why `verified` matters:** a gold pair is only worth keeping if the finding was confirmed against
the actual source (not a model guess). Both skills run a verify step and emit a pair only when the
verdict is confirmed — including confirmed *non*-findings (`severity: "clear"`, or `false_alarm_of`
set), which are the most valuable negative examples.
