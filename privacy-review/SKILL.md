---
name: privacy-review
description: A fearless privacy & security inventory of ANY app that holds sensitive user data — hunts the gap between what privacy/security the code CLAIMS and what it actually ENFORCES. DB/ecosystem-agnostic. Reports 🔴/🟠/🟡/🟢 per check; optional --gold emits verified findings as portable training pairs. Use before a public release, before shipping a sensitive feature (matching/sharing/DMs/AI-over-user-data), or after adding auth/payments/PII/AI.
---

# /privacy-review — fearless code moral inventory

The most dangerous pattern in apps holding sensitive data is **stated intent the code doesn't
enforce**: "stored encrypted" over a plaintext column; "we can't read your data" over a server-held
key; "only X users can do Y" enforced in a spec but not the query. This skill hunts that gap. **No
ecosystem dependencies** — works on any stack, detects DB/auth from the repo.

## Usage

```
/privacy-review                 # whole app, sensitive surfaces first
/privacy-review <path|feature>  # scope to a feature/dir
/privacy-review --live          # also verify against the deployed DB/grants, not just source
/privacy-review --gold          # append verified findings to .audit/gold-pairs.jsonl (see README)
```

Work **sensitive-first**: enumerate the highest-harm content (health/recovery/sexual, private
messages, financial, precise location) and review the surfaces that touch it before anything else.

## Severity

| Icon | Level | Meaning |
|------|-------|---------|
| 🔴 | critical | Live or trivially-reachable exposure of sensitive content / identity / money. Fix before release. |
| 🟠 | high | Real exposure behind a non-trivial condition, or a safety control that's unenforced. |
| 🟡 | medium | Weakness, hygiene gap, or a claim that's technically true but easy to misread. |
| 🟢 | clear | Verified correct — record it so the next reviewer doesn't re-derive it. |

---

## Check 1 — stated-vs-actual gap (the heart of it; run first, run hard)
For every place code/comments/docs/marketing **claim** a privacy property, find the enforcement.
```bash
grep -rinE "encrypt|private|secure|anonym|pseudonym|only you|cannot read|zero.?knowledge|not stored|deleted|hashed|HIPAA|GDPR|consent" . | grep -v node_modules
```
For each claim ask *where is this enforced, and what happens if I don't cooperate?*
- "Stored encrypted" → a real crypto primitive, or a cleartext column with an aspirational comment?
- "Pseudonymous / we don't know who you are" → keyed to a pseudonym, or one join from the email?
- "We can't read it" → does the **server hold the key**? Then the honest claim is "encrypted, we hold the keys," not zero-knowledge.
- "X only for Y users" → enforced in the handler/query, or only described?

🔴/🟠: a claimed protection with **no enforcement**, or one enforced in a single path but bypassable via another (a second endpoint, an admin route, a job, a direct API/RPC call). Severity = sensitivity of what it "protects."

## Check 2 — identity-source triage (privileged / elevated code)
For anything that runs with elevated rights (Postgres `SECURITY DEFINER`, admin handlers, anything
bypassing row-level checks), the question isn't "does it have a grant" — it's **whose identity does
it act on, and can an arbitrary caller reach a privileged action.**

| Bucket | Signature | Severity |
|--------|-----------|----------|
| 1 · Param-identity | takes a target `user_id`/id as a **parameter**, acts on it with no `caller == target` check | 🔴 cross-user read/write |
| 2 · Unguarded privileged write | mutates balances/roles/subscriptions with no caller-auth check, reachable by ordinary users | 🔴 forgery / escalation |
| 3 · Self-scoped | identity derived **internally** from the session; all access scoped to it | 🟢 safe (revoking anon is *hygiene*, not a fix) |
| 4 · Trigger/event fns | fired by events, not callable as an endpoint | ⚪ N/A here (still check search_path/injection) |

Triage rule: a missing access-revoke is *hygiene*; the *critical* finding is Bucket 1 or 2 only — don't drown the report by escalating 3/4. Review the **latest/deployed** definition of a redefined function.

## Check 3 — sensitive-content classification & encryption posture
Classify every store of user content by harm-if-leaked, then state the **honest** posture:

| Posture | Who can read | Honest claim |
|---------|--------------|--------------|
| True E2E (client key the server never holds) | only the user | "Not even we can read this" ✅ |
| Encrypted at rest (operator holds the key) | operator, provider, breach-with-key, subpoena | "Encrypted & access-controlled" — **NOT** "even we can't read it" |
| Plaintext | anyone with DB/operator/breach access | be honest, or fix it |

🔴 highest-tier content stored plaintext (🟠 if tightly access-controlled and not externally reachable).
🟠 encrypted-at-rest **marketed as** zero-knowledge (false claim — only true E2E earns it).
Content the AI/server must read to function **cannot** be zero-knowledge — flag any plan claiming both; resolve per-field (encrypt what the server never reads; access-control the rest).
🟡 partial encryption: identity/contact fields (`name`, `phone`, `email`, `dateOfBirth`) left plaintext while medical/notes are encrypted → a DB breach still exposes *who*. Note it as a conscious tradeoff, not silent.

## Check 4 — identity linkage & pseudonymity
Can sensitive content be **joined back to a real identity** (email/billing/phone)? Trace the FK /
ownership column on each sensitive table.
🟠 "anonymous to the system / the AI" claimed, but content keys directly to the auth/billing identity → not structurally separated. True separation needs a pseudonymous content id + a restricted mapping table the content/AI layer can't read.

## Check 5 — access-control reachability (verify, don't assume)
For each sensitive surface answer **"who can actually reach this?"** not "what does the policy say."
- RLS / row checks enabled on every sensitive table? Per-operation (no blanket `FOR ALL`)? Identity from the session, not the request body?
- Endpoint/function execution: who holds it? Is **unauthenticated** access possible via a default grant or a missing auth check?
- **Second paths**: a sensitive read reachable via an admin route, a report, an export, a cron job, a webhook, or a direct API call that skips the guarded UI path?
- **App-level tenant isolation with no DB backstop** (single DB role, ownership enforced only in code): flag every query on a tenant table that lacks the owner predicate — one miss = cross-tenant leak. (See the kit's RLS-hardening note for the centralized-scope fix.)

## Check 6 — third-party & egress (where sensitive data leaves)
- 🟠 sensitive content sent to an external AI/processor (OpenAI/Anthropic/transcription/OCR) with **no DPA/BAA**, no retention/no-training assurance, no redaction layer → compliance gap (HIPAA/GDPR/local law).
- 🟡 PII in plaintext email/SMS/webhook bodies stored indefinitely by a third party (Resend/Twilio/etc.).
- 🟡 uploaded files stored at a **public** ACL when they should be private + proxied; signed-URL/CDN links that never expire and are persisted.
- 🟠 secrets/keys hardcoded, logged, or committed; tokens passed where they can be captured.
- 🟡 rate limiting that doesn't actually gate (in-memory per-instance on serverless → resets/forks; no shared store) on a public/auth/PII endpoint.

---

## Output

Sensitive surfaces first. Per finding: `🔴/🟠/🟡/🟢 <check> — <finding> (file:line) → <fix>`. Then a
ranked must-fix list. A verified `🟢 clear` is a valid result — don't manufacture findings.

## Verify before you assert (and before you emit a gold pair)

Confirm every 🔴/🟠 against the **actual source**, not a pattern guess — read the handler, follow the
enforcement, check for a guard elsewhere. Models routinely over-call IDOR/auth bugs that the code
actually denies; a wrong "critical" handed to a repo owner burns trust. If a suspected finding is
safe, record `🟢 clear` (and with `--gold`, a `false_alarm_of` negative example). State your
confidence; separate confirmed from suspected.

## `--gold` emission

When `--gold` is set, after verification append one JSON line per **verified** finding (and per
verified clear/dismissed claim) to `.audit/gold-pairs.jsonl`, per the kit README schema. **Redact all
secret values** in `snippet` (keys, tokens, connection strings, PII samples → `‹redacted›`) — these
pairs are meant to be shareable. Stamp `repo`/`commit` from git (best-effort).
