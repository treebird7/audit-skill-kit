# Contributing

`main` is protected — all changes land through a pull request. External contributors don't have
write access, so the path in is **fork → branch → PR**.

## Fork-and-PR flow

```bash
# 1. Fork on GitHub (button, top-right), then clone YOUR fork
git clone https://github.com/<your-username>/audit-skill-kit
cd audit-skill-kit

# 2. Branch for your change
git checkout -b add-foo-review

# 3. Commit + push to your fork
git commit -am "add foo-review skill"
git push -u origin add-foo-review

# 4. Open the PR against treebird7/audit-skill-kit:main
gh pr create --repo treebird7/audit-skill-kit --base main
```

A maintainer reviews and merges. You can't merge it yourself — that's the protection working as
intended.

## Keeping a fork current

```bash
git remote add upstream https://github.com/treebird7/audit-skill-kit
git fetch upstream && git rebase upstream/main
```

## What makes a good skill PR

These skills are deliberately **ecosystem-agnostic** — no shared knowledge tree, no training
pipeline, no provider lock-in. A new or changed skill should keep that:

- **Self-contained** — detect the stack from the repo; don't assume a specific DB, host, or org.
- **Verify-before-assert** — every check tells the reviewer to confirm a finding against the actual
  source before reporting it (the rule that catches false-positive "criticals").
- **`--gold` support** — emit verified findings to `.audit/gold-pairs.jsonl` per the schema in the
  [README](./README.md), with secrets redacted.
- **No new runtime deps** — these are Markdown skill definitions, not code.

Match the structure of an existing skill (`sql-review`, `privacy-review`, `ts-review`,
`node-review`): frontmatter `name` + `description`, a usage block, severity table, numbered checks,
output format, the verify step, and the `--gold` emission note.
