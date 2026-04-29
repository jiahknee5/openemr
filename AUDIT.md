# OpenEMR Fork Audit — AgentForge Week 1

**Subject:** OpenEMR fork at `Week 1/openemr/` (release tag `v8.1.1-dev`, see `version.php:18-21`)
**Auditor:** Johnny Chung
**Date:** 2026-04-28
**Purpose:** Defensible engineering audit to scope the AgentForge clinical co-pilot integration. Findings drive `ARCHITECTURE.md` decisions and the agent verification contract. Target use cases: hospitalist overnight delta, hospitalist discharge readiness, resident pre-presentation audit (see `kb/research/healthcare-personas-and-usecases.md`).

**Severity legend:**

- **P0** — Blocks any deploy with real PHI (Protected Health Information — any medical record, identifier, or clinical data tied to an individual patient; under HIPAA, the patient's name, MRN, diagnosis, lab values, notes, etc. all count).
- **P1** — Must be fixed before clinical use.
- **P2** — Technical debt; document and revisit.

**How to read this audit.** Each finding opens with a non-technical explanation aimed at a clinical or business reader, followed by file-anchored technical detail for engineers. Severity is called out in the heading of each finding. The summary table near the end indexes findings by the three target use cases so a reader can jump from a use case to the findings that gate it.

---

## Executive Summary

**OpenEMR 8.1.1-dev is deployable as a FHIR data plane for an external agent — but three structural risks (audit, BAA chain, hot-path indexing) shape every architectural decision downstream.**

### What we audited

| | |
|---|---|
| **Subject** | OpenEMR fork at `Week 1/openemr/` — release tag `v8.1.1-dev` (`version.php:18-21`) |
| **Scope** | Five dimensions: Security, Performance, Architecture, Data Quality, HIPAA/Compliance |
| **Method** | File-anchored static review of `/src`, `/library`, `/interface`, `/sql`, `/docker`; cross-referenced against research in `kb/research/` |
| **Findings** | **28 total** — every claim cites a file and line range |
| **Severity** | **5 P0** · **13 P1** · **5 P2** · 5 architectural observations (§3) |
| **Verdict** | **Deployable with mitigations.** Integration shape is constrained by 3 P0 risk classes — agent must be external, own its own audit, and enforce cite-or-refuse in code. |

### Top 3 P0 findings

| # | Finding | Why it matters | Mitigation |
|---|---|---|---|
| 1 | **Audit logging is conditional and writes PHI in plaintext** (§5.1) | OpenEMR's built-in audit log is not legally sufficient for tracking who accessed what patient when. It's optional (an admin can switch it off — `ApiResponseLoggerListener.php:56`), it dumps full patient records into a single text column when on (PHI in plaintext where any DBA can read it — `sql/database.sql:7758-7776`), and the table can be edited or deleted (no tamper-evidence). HIPAA §164.312(b) requires an authoritative, append-only audit trail; OpenEMR's stock log isn't one. Without a fix, a regulator asking "show me everything Maya saw last week" gets either an empty result or a grep through PHI blobs — and a tampered log would be undetectable. (Full rationale in §A.) | Agent maintains its own append-only audit store, written before every tool call and engineered to the regulation (full mitigation in §A) |
| 2 | **No BAA chain to LLM, host, or observability vendor** (§5.3) | Three vendors touch PHI on the agent's data path: Anthropic (the LLM provider), Railway (the host), and the observability vendor. None of them has a BAA (Business Associate Agreement — a contract under HIPAA §164.504(e) where any vendor that touches PHI on a covered entity's behalf signs a legal commitment to handle it according to the same rules; without a BAA, the vendor can't legally see PHI; with one, they're legally on the hook for breaches) with OpenEMR's operator by default. Without that contract, every patient byte that crosses into the agent is a HIPAA violation under §164.504(e) — even if the agent itself works perfectly. The agent service is the only point in the architecture where a BAA can be enforced for the whole vendor chain. For the week-1 synthetic-data demo this finding is dormant (no real PHI flows); for any deploy with actual patients, it's a hard pre-deploy gate. | Pre-deploy gate: BAA with Anthropic (zero-retention), Railway (HIPAA tier), and the observability vendor. LangSmith for week-1 demo on synthetic data only; self-hosted Langfuse as the production target inside the Railway BAA boundary |
| 3 | **`pnotes` indexed only on `pid`, not `date` — overnight-delta hot path** (§2.2) | The `pnotes` table holds nursing notes — including overnight events like falls, rapid responses, or codes. The hospitalist's most important morning question — "what happened to my patients between 6 PM and 6 AM?" — is a query against this table filtered by patient and time. The patient column is indexed; the time column is not (`sql/database.sql:8670-8689`). Under load, the database optimizer can return a *partial* result set without raising an error — meaning the agent could miss a 03:14 RN-documented fall and report "no overnight events." The hospitalist skips the neuro exam, the subdural is missed, the patient deteriorates. **This is why omission is a P0 even though the data is technically present**: the failure mode is silent, the consequence is clinical, and the agent has no way to know a row was dropped unless we engineer the count parity check ourselves. Per *Nature* 2025, omission rate (3.45%) exceeds hallucination rate (1.47%) in clinical LLM summarization — the dangerous failure isn't the LLM making things up; it's the LLM missing things that exist. | Tool enumerates by primary key with explicit window filter; verifier runs independent `COUNT(*)` parity check and refuses on mismatch |

> **Note on default credentials (§1.2 / §5.7).** This audit also identifies the published default `admin/pass` as a P0 in absolute terms. **It is deprioritized for the week-1 demo** because the deployment runs synthetic data only and the URL is treated as a secret. The finding remains in the audit body and must be re-elevated before any deploy that touches real PHI.

### Three architectural constraints this audit dictates

These aren't best practices — they're consequences of what we found in the code. Each is the answer to a specific question a reviewer might ask. Full rationale in §B below.

- **Agent must run as a separate service, not as code inside OpenEMR.** Building inside OpenEMR's runtime means inheriting 25 years of legacy code paths we can't fully audit. Building outside gives us a clean, observable boundary — every byte of PHI that crosses into the agent crosses a wire we own.
- **Agent owns its own audit trail; OpenEMR's log is supplementary.** OpenEMR's log is conditional, mutable, and PHI-in-plaintext; HIPAA §164.312(b) requires authoritative. We engineer our own audit store exactly to the regulation.
- **Cite-or-refuse must be enforced by code, not by prompt.** Telling the LLM to cite isn't enforcement. A deterministic verifier runs after the LLM and checks every claim against a fresh database query. If anything doesn't match, the agent admits the gap rather than shipping a confident wrong answer.

### For the demo

> "OpenEMR is a 25-year EHR with a modernizing core. We can build on it — but three structural risks shape every architectural decision in `ARCHITECTURE.md`: the audit trail, the BAA chain, and the hot-path indexing on overnight events."

---

## §A. Deep Dive — The Audit-Trail Gap (Finding #1)

### Why it matters

HIPAA's Security Rule §164.312(b) — "audit controls" — requires a covered entity to *record and examine activity in information systems that contain ePHI*. In practice this means a regulator, a compliance officer, or a privacy attorney needs to be able to ask three questions and get a defensible answer:

1. **Who did what to which patient, and when?** Across every read, write, login, and tool invocation.
2. **Can we prove no one tampered with the record?** The audit log itself must be tamper-evident.
3. **Can we retrieve six years of history quickly when a breach investigation demands it?** §164.530(j) requires retention of compliance documentation for at least six years.

OpenEMR's stock audit configuration fails all three on the API/FHIR boundary the agent will use:

- **Logging is opt-in.** `ApiResponseLoggerListener.php:56` only fires when an admin sets `api_log_option > 0`. An admin can switch it off. There is no enforcement that it stays on.
- **When it does log, it logs the wrong thing.** It writes the full JSON response body into a single `log.comments` text column (`sql/database.sql:7758-7776`) — PHI in plaintext, not encrypted at the column level, in a table where every row is mixed with non-PHI rows. A regulator looking for "what did Maya see at 6:35 AM Tuesday" has to grep blobs.
- **The log is mutable.** Both `log` and `audit_master` are normal InnoDB tables. Any DBA with `UPDATE` rights can rewrite history. There is no append-only constraint at the schema level, no daily integrity hash, no WORM backup.
- **Forensic queries don't scale.** `log.patient_id` is the only useful index. "Who accessed Mr. Patel in the last 24 hours" is a table scan; "every action User X took" is a table scan. A six-year retention horizon makes scans impractical.

The combined effect: even if every other piece of OpenEMR were perfect, you could not honestly tell a HIPAA auditor that you are recording and examining activity on the agent's data path. **§164.312(b) cannot be satisfied by OpenEMR alone — full stop.** This is the textbook definition of a P0 in a regulated environment.

### Mitigation

The agent service maintains its own audit infrastructure, sized to the regulation:

- **Append-only Postgres table** in the agent's database (separate from OpenEMR). Schema enforces append-only with a write-only role and per-row immutability; updates and deletes are denied at the DB grant level, not just at the application level.
- **Logged on every request, before any tool call.** Gateway writes `{request_id, user_id, username, facility_id, encounter_id, jwt_jti, timestamp, intent, decision, decision_reason}` *before* the agent runtime sees the request. If the audit write fails the request returns 503 and never proceeds — there is no path where work happens but logging doesn't.
- **Indexed for forensics.** Composite indexes on `(user_id, timestamp)`, `(patient_id, timestamp)`, `(timestamp, decision)` so "who accessed whom and when" returns in milliseconds.
- **Six-year retention with monthly partitions.** Each month's partition is closed and copied to immutable WORM storage at month-end; live tables hold the most recent 90 days for fast access, archives hold the rest.
- **Tamper evidence.** Daily integrity hash chain — each day's audit rows are hashed, the hash is signed and stored in a separate location. A regulator can verify any range of rows was not modified after the fact.
- **PHI minimization.** The audit row records *what was asked and what categories of data were returned*, not the data itself. We log "Maya queried overnight delta for patient_id=X, returned 7 RN notes citing forms.id ∈ {…}" — not the note text. PHI stays in OpenEMR; the audit stays in our store. Privacy officer reads the audit, never reads PHI from it.
- **OpenEMR's native log remains supplementary.** We don't disable it; we don't depend on it. If OpenEMR's log is on it provides a second source. The agent's audit is the system of record.

The result: every question §164.312(b) can ask has a defensible answer. The audit trail is owned by code we wrote, not configuration in OpenEMR's admin UI.

> **Note: the audit store is *not* the same as the observability stack.** LangSmith / Langfuse / OpenTelemetry traces capture LLM behavior (prompts, responses, tool calls, latency) and are read by engineers debugging the agent. The audit store captures *who asked the agent what about which patient* — request metadata only, no PHI content — and is read by privacy officers and HIPAA auditors. The two have different schemas, different retention requirements, different access controls, and different mutability guarantees. The audit store is append-only with a tamper-evident hash chain; observability traces are mutable. The audit store retains 6 years; observability traces retain 30 days. Conflating them — using LangSmith as the HIPAA log — fails compliance because traces aren't immutable, aren't append-only, and contain PHI in plaintext (model prompts/responses). The custom audit store exists *because* LangSmith / Langfuse cannot be that thing.

---

## §B. Architectural Constraints in Detail

### Constraint 1 — The agent must run as a separate service, not as code inside OpenEMR.

*The question this answers:* Why not just write a PHP module that lives inside OpenEMR? It would be simpler — same database connection, same session, same deployment.

*The reason we said no:*

- **Legacy SQL surfaces are still active.** §1.4 documents 10 files in `/library` that still interpolate variables directly into SQL strings. They aren't injectable today (the inputs are internal), but a careless `require_once` from agent code pulls them into the agent's runtime — and the next refactor could break the safety. We can't audit "the agent never touches those files" if the agent runs in the same PHP process.
- **OpenEMR's ACL helper has no caller-side enforcement.** §1.3 — there's no compile-time gate that says "every retrieval went through `AclMain`." If the agent runs in-process, we'd be one missing check away from an authorization hole. Out-of-process forces every retrieval to cross an HTTP boundary we control.
- **Audit ownership is impossible in-process.** §5.1 — to own the audit trail (see §A above) the agent has to log *before and after* every external system call. In-process, "external system call" is a blurry line; out-of-process, it's an HTTP request the gateway intercepts.

*In one sentence to the reviewer:* "Building the agent inside OpenEMR's runtime would mean inheriting 25 years of legacy code paths we can't fully audit. Building it outside means a clean, observable boundary — every byte of PHI that crosses into the agent crosses a wire we own."

### Constraint 2 — The agent owns its own audit trail; OpenEMR's log is supplementary, not authoritative.

*The question this answers:* OpenEMR has an `audit_master` table and an API logger. Why don't we just use those?

*The reason we said no:*

- **OpenEMR's logging is conditional, mutable, and PHI-in-plaintext** (the expanded mitigation in §A covers this in detail).
- **HIPAA §164.312(b) doesn't accept "best effort" — it requires authoritative.** A regulator asking "show me all access for this patient over six years" needs an answer that is verifiably complete and tamper-evident. OpenEMR's stock setup cannot demonstrate either property.
- **Multi-vendor responsibility splits cleanly when each system owns its own audit.** OpenEMR audits OpenEMR-internal access. The agent audits agent-mediated access. Privacy officer reads both side by side; no system depends on the other being honest.

*In one sentence to the reviewer:* "When something goes wrong and we need to prove what the agent did and why, we need a log we can defend on three axes: tamper-evident, append-only, fast to query for six years. OpenEMR's log doesn't meet that bar, so we keep our own — engineered exactly to the regulation."

### Constraint 3 — Cite-or-refuse must be enforced by code that runs after the LLM, not by instructions inside the prompt.

*The question this answers:* Why isn't "always cite your sources" in the system prompt enough?

*The reason we said no:*

- **Prompts are aspirations; code is enforcement.** Telling an LLM to cite is statistically reliable, not deterministic. Under load, edge cases, or prompt-injection drift, models occasionally drop citations or invent timestamps. A patient-safety system can't tolerate "occasionally."
- **The dominant clinical-summarization failure is omission, not hallucination.** Per *Nature* (cited in `USERS.md`), omission rate is roughly 3.45%, hallucination rate is roughly 1.47%. The dangerous failure isn't the LLM making up a fall — it's the LLM missing a real one. Missing the 03:14 RN fall note means missing the neuro exam means missing the subdural. A prompt that says "be careful" doesn't catch this; a code-level count parity check (independent `SELECT COUNT(*)` reconciled against the LLM's cited count) does.
- **The data model itself increases omission risk.** §2.2 — `pnotes.date` is unindexed; the optimizer can return partial result sets under load. The agent's tool layer must enumerate by primary key and refuse on partial reads. That refusal is a property of the tool implementation, not of the LLM's behavior.

*In one sentence to the reviewer:* "Telling the LLM to always cite isn't the same as enforcing it. We run a separate piece of plain code after the LLM, before the user sees anything, that checks every claim against a fresh database query. If anything doesn't match, the agent admits the gap — 'I found N events but counted M; review source' — instead of shipping a confident wrong answer. The verifier is the keystone of the architecture."

---

## 1. Security

This section covers the surfaces where OpenEMR makes (or fails to make) a security decision the agent will inherit: authentication, authorization, default credentials, and the legacy SQL patterns the agent must avoid touching. Read it as a map of what we trust, what we re-check at the gateway, and what we refuse to call from agent code.

### 1.1 Password hashing is sound on the modern path (P2 — observation, not a fix)

OpenEMR is doing the right thing on passwords — modern algorithms, constant-time comparison, the works. The one wart is a legacy fallback that still validates ancient bcrypt hashes from before 2018, which we should rip out once nobody's using those old accounts. For our agent, this means we don't need to invent our own auth; we lean on OpenEMR's session and only worry about the service token we issue ourselves.

`src/Common/Auth/AuthHash.php:111-150` configures Bcrypt, Argon2i, Argon2id, or SHA512-crypt via `password_hash()` with cost parameters from `OEGlobalsBag`. Verification (`:187-234`) uses constant-time `hash_equals` for the SHA512 path and `password_verify` for the rest. A legacy bcrypt-with-21-char-salt fallback (`:216-222`) still exists for hashes created pre-5.0.0 and silently re-validates them; it should be excised once the install no longer carries those rows.

- **Agent integration implication:** Authentication for the agent must inherit the live OpenEMR session via `SessionWrapperFactory`, not re-implement password verification. Any service token issued to the agent service must be bcrypt-or-better at rest in our own store; do not mirror OpenEMR's legacy fallback.

### 1.2 Default credentials in development image (P0 if prod, P2 if scoped)

The default OpenEMR docker image ships with username `admin` and password `pass`, and those credentials are literally documented in the repo. If we deploy that image to a public URL without rotating, anyone reading the GitHub README can log in. This is the single highest-likelihood real-world breach for any team demoing OpenEMR — we have to gate it at deploy time and make sure the agent itself refuses to operate as the default admin account.

- `CLAUDE.md:34-36` (project root of fork) documents the canonical default `admin / pass`.
- The `docker/development-easy/` compose stack ships with this and is the *recommended* local setup.
- There is no automated rotation hook on first boot.
- A Railway demo deploy using this image untouched is internet-reachable with a credential published in the repo.

- **Agent integration implication:** Pre-deploy gate: rotate `admin` and disable Google sign-in defaults before exposing the URL. Document this in the deploy README. The agent must refuse to attach to a session whose user is `admin` unless an explicit dev-mode flag is set, so demo credentials cannot be used to query real PHI by accident.

### 1.3 ACL helper is the single point of truth — and is silently fail-closed (P1)

There's exactly one function in OpenEMR that decides who can see what. It's working — when in doubt, it says no — but it doesn't tell you *why* it said no, it can't distinguish "deny" from "no rule was ever configured," and it caches its answers for the lifetime of the worker process. So a permission revoked mid-shift might not actually take effect for a while. For the agent, this means we can trust a "no" from OpenEMR but we can never trust a "yes" without re-checking ourselves at the gateway.

`src/Common/Acl/AclMain.php:166-238` (`aclCheckCore`) is the canonical permission gate. The call path looks like this:

```
                                                  ┌─ no structured reason
Request ──► AclMain::aclCheckCore ──► GACL cache ─┼─ no expiry (process-lifetime)
                       │                          └─ no caller enforcement
                       ▼
                  DB lookup ──► boolean (true | false)
```

Important properties of the helper:

| Property | Where | Implication |
|---|---|---|
| Superuser bypass on `admin/super` | `:174` | One ACL row = god mode; surface in every audit row. |
| Empty-result = deny | `:182-184` | Fail-closed, but "deny" is indistinguishable from "no rule defined." |
| No structured reason returned | `:236-237` | Returns bool only; caller cannot route on *why*. |
| Static GACL object cached for process lifetime | `:128, 132-139` | Mid-shift revocation only takes effect on PHP-FPM recycle. |

Same file, `zhAclCheck()` (`:252-310`) builds raw SQL strings via concatenation of placeholder counts, then binds them — the structure is parameterized but the `IN (?,?,?)` list size is template-string-driven. This is *not* an injection (placeholders are bound), but it is an example of pattern-by-pattern safety that a junior contributor could break by accident.

- **Agent integration implication:** Wrap every agent tool call in a *structured* ACL probe that records `{user, section, value, allowed, reason}` to our own append-only audit before performing the read. Treat OpenEMR's `false` as authoritative for deny but assume a parallel re-check at the data layer; never let a `true` from a cached GACL object be the only gate.

### 1.4 Inline SQL string interpolation persists in legacy paths (P1)

Most of OpenEMR's modern code uses parameterized queries — the safe pattern. But the older `/library` directory still has files where SQL is built by string concatenation. They aren't actively exploitable today because the inputs come from internal code, not users, but they're the kind of pattern a future refactor can break without anyone noticing. The agent has to stay out of those legacy files entirely and only call modern service classes.

Concrete examples surfaced by searching for `sqlStatement(` calls with `$variables` interpolated directly into the SQL string:

- `library/options.inc.php:233` — `ORDER BY $order_by_sql` (column is a caller variable).
- `library/options.inc.php:3443` — `SELECT $sel FROM ...`.
- Other inline-SQL files matched (10 total): `library/patient.inc.php`, `library/translation.inc.php`, `library/standard_tables_capture.inc.php`, `library/sql.inc.php`, `library/amc.php`, `library/direct_message_check.inc.php`.

These are not classical injection (sources are typically internal code, not user input), but they are templates a refactor can break and PHPStan level 10 does not catch SQL-as-string. Modern paths via `OpenEMR\Common\Database\QueryUtils` (referenced in `src/Common/Acl/AclMain.php:121`) are parameterized; new code must use that. The risk surface is *legacy includes still active in production code paths*.

- **Agent integration implication:** The agent must never call legacy `library/*.inc.php` functions that interpolate SQL. All agent reads route through FHIR or `BaseService`-derived service classes (`CLAUDE.md` "Service Layer Pattern" section). Document this as a hard constraint in the agent harness.

### 1.5 Patient-portal session is a separate, thinner gate (P1)

Patients log into a different door than clinicians do, and that door has a much simpler lock — basically a session-cookie check, no MFA, no rate limit. By design it also tells the rest of the system "skip the clinician auth checks" because the user isn't a clinician. That's correct, but it means any bug that confuses a portal session for a clinician session is a privilege escalation. The agent must never be reachable from the portal at all.

`portal/verify_session.php:50-60` is the entire portal session check:

- Requires `pid` and `patient_portal_onsite_two` on the session, otherwise destroy and redirect.
- No MFA, no rate limit at this layer (depends on session middleware upstream).
- Sets `$ignoreAuth_onsite_portal = true` (`:62`), which downstream code uses to skip the *clinician* auth check.

This is correct in design (portal users are not clinicians) but means a bug that confuses portal vs. clinician sessions opens an authorization hole.

- **Agent integration implication:** The agent must never be reachable from the patient portal endpoint range (`/portal/*`). Enforce at the gateway: deny any agent request whose session originated from `getPortalSession()`. Treat portal sessions as a different identity class entirely.

### 1.6 Residual `mysqli_query` callsites in legacy code (P2)

A handful of old files still talk to the database the raw, pre-ORM way. They're in install scripts and vendored third-party code, so they're not in any normal request path — but a careless `require_once` could pull them in. We just need a CI check that fails the build if the agent module ever touches one of those files.

Files still containing raw `mysqli_query(` calls (6 occurrences across 4 files):

- `admin.php`
- `portal/patient/fwk/libs/verysimple/DB/DataDriver/MySQLi.php`
- `library/classes/Installer.class.php`
- `interface/eRxStore.php`

Most are in install/admin paths or third-party-vendored ORM code. None should ever be in an agent code path, but their presence means a careless include can pull them.

- **Agent integration implication:** Static analysis on the agent module should fail the build if any required-or-included file path resolves to one of those four files.

---

## 2. Performance

The agent's hot paths are *time-window* and *per-patient* reads against the inpatient list. The schema is not tuned for them. Read this section as a list of where the agent's pre-rounds query plan will degrade, and where silent omission becomes a clinical risk rather than a latency one.

### 2.1 `lists` table — flat, two non-uuid indexes, hot for problems/allergies/meds (P1)

Problems, allergies, and meds all live in one shared table called `lists`, and the database only has indexes on patient ID and row type — not on whether the entry is still active or when it ended. So when the agent asks "what are this patient's *active* allergies," the database has to scan every allergy row that patient has ever had. Fine at one patient, painful at fifty. We work around it by caching a snapshot per patient for the length of a pre-rounds session.

`sql/database.sql:7671-7715`.

- Existing indexes: `PRIMARY (id)`, `KEY pid (pid)`, `KEY type (type)`, `UNIQUE uuid (uuid)`.
- Missing: composite `(pid, type, enddate)` for "active problems for this patient," and any index on `activity`.
- Query shape: `pid IN (...) AND type='allergy' AND (enddate IS NULL OR enddate > NOW()) AND activity=1`. Only the `pid` and `type` predicates are indexed; the active/timeframe filter is a row scan per matched patient.

Practical impact for a 13-patient hospitalist service with ~50 list rows per patient: ~650 rows scanned per allergy/problem/med call, three calls per patient = a few thousand row reads per pre-rounds session. Fine at 13 patients, ugly at 50, and it shares the table with every other clinic's data on a multi-tenant box.

- **Agent integration implication:** Cache list snapshots per `(pid, type)` for the duration of one pre-rounds session (TTL ≤ 15 min). Invalidate on any FHIR write event we observe. Document in `ARCHITECTURE.md` cost section.

### 2.2 `pnotes` (clinician/RN notes) — only `pid` indexed; this is the overnight-delta hot path (P0 for the use case)

Clinical notes live in a table that's only indexed by patient — not by date. The hospitalist's whole overnight-delta question is "what happened on these patients between 6pm and 6am," which is a date-range query against this exact table. On a patient with a long inpatient stay, the database has to scan every note they've ever had. The dangerous part is that if the query times out, the agent silently gets a partial result — and that's exactly the omission failure mode that causes a missed neuro exam. The agent must enumerate notes by primary key and verify the count, never trust one bulk query.

`sql/database.sql:8670-8689`.

- Existing indexes: `PRIMARY (id)`, `KEY pid (pid)`.
- Missing: `date`, `(pid, date)`, `activity`, `assigned_to`.
- Query shape: "all RN-authored pnotes for these 13 PIDs where `date BETWEEN '2026-04-27 18:00' AND '2026-04-28 06:00'`."
- Plan: optimizer narrows by `pid IN (...)` then row-scans the date filter. Worst case (a 6-month inpatient stay, hundreds of rows per pid) the agent risks a partial set if the query is killed by `max_execution_time`.

This is the failure mode `kb/research/healthcare-personas-and-usecases.md:132` calls out: silent omission of an RN fall note → missed neuro exam → missed subdural hematoma.

- **Agent integration implication:** Hard requirement for the agent contract: *enumerate by primary key and verify count*. The `list_overnight_events` tool must (a) issue a `SELECT id FROM pnotes WHERE pid=? AND date BETWEEN ? AND ?` first, (b) record the count, (c) fetch each row by id, (d) refuse if any individual fetch fails. Never trust a single bulk SELECT to be complete. Add a recommended schema patch to the deploy doc: `KEY pnotes_pid_date (pid, date)`.

### 2.3 `procedure_report.date_report` — unindexed, used for "what came back overnight" (P1)

Same problem, different table. When the hospitalist asks "what labs or cultures came back since 6pm," the column that records when the result returned isn't indexed at all. The fix is to use the *order date* (which is indexed) to narrow down candidates and then filter the rest in code. Important to document this so a future engineer doesn't naively rewrite the query against the unindexed column.

`sql/database.sql:10467-10484`.

- Existing indexes: `PRIMARY (procedure_report_id)`, `KEY procedure_order_id (procedure_order_id)`, `UNIQUE uuid`.
- Missing: any index on `date_report`, `date_collected`, `report_status`, `review_status`.
- Query shape: "what microbiology came back since 18:00?" filters on `date_report` and `report_status` — both unindexed.
- Joined to `procedure_order` (which has `KEY datepid (date_ordered, patient_id)`, `:10408`), the plan is acceptable for ordered-but-not-back labs but degrades for "result returned in this window."

- **Agent integration implication:** Use `procedure_order.date_ordered` (indexed) to bound the candidate set, then filter `procedure_report` rows in application code. Document the shape of this query so a future engineer doesn't naively switch to `date_report` and silently slow the pre-rounds path.

### 2.4 `audit_master` and `log` tables — primary-key-only / single-column index (P1)

OpenEMR's own audit tables aren't indexed for the queries you'd actually run during an incident — "show me everything user X did in the last 24 hours" is a full table scan. Combined with the fact that the audit log dumps PHI into a comments column, this means OpenEMR's audit log is not viable as our system of record for HIPAA. We keep our own.

- `audit_master` (`sql/database.sql:149-162`) — `PRIMARY KEY (id)` only. No index on `pid`, `user_id`, `created_time`, `type`, `approval_status`. "Show me all audit events for this user in the last 24h" is a full scan.
- `log` (`sql/database.sql:7758-7776`) — only `KEY patient_id (patient_id)`. No index on `date`, `event`, `category`, `user`, `success`. Same problem, at the table the API logger writes into.
- `audit_details` (`sql/database.sql:171-180`) — `KEY audit_master_id` only. No index on `table_name` or `field_name`, so "which fields changed in this table this week" is a full scan.

Forensics queries for an after-the-fact incident review are O(n) on a table that grows with every API call.

- **Agent integration implication:** The agent's *own* audit store must be the system of record for who-did-what-when. Treat OpenEMR's `log` and `audit_master` as redundant secondary records; query our own store for breach-notification, access-review, and HIPAA §164.312(b) audit-control reports. Recommend a schema patch (`log_pid_date_idx`, `log_user_date_idx`) for the operator running this stack but do not block on it.

### 2.5 `form_encounter` is reasonably indexed; do not break it (P2 — note for the integrator)

Good news for once — the encounters table is properly indexed for the "today's census" query. Just a note to whoever builds against it: don't add a join that defeats the existing index path.

`sql/database.sql:2022-2062`. Indexes: `PRIMARY (id)`, `UNIQUE uuid`, `KEY pid_encounter (pid, encounter)`, `KEY encounter_date (date)`. The hospitalist "today's census" query — `WHERE date >= CURDATE() AND class_code='IMP'` — uses `encounter_date`. The agent should not introduce a join that breaks this index path.

---

## 3. Architecture

This section maps the surfaces the agent is allowed to touch and the ones it is forbidden to touch. The headline is that OpenEMR is four codebases in one repository, and the agent integrates with exactly two of them — the modern PSR-4 services and the Symfony event boundary.

### 3.1 Four parallel paradigms in one tree

OpenEMR is really four codebases stacked on top of each other — 25-year-old procedural PHP, modern PSR-4 services, Zend MVC modules, and Symfony event subscribers. They all coexist in one tree. The agent only integrates with the modern surface and the Symfony events; the legacy directories are off-limits, full stop. We enforce that with autoloader scoping and a static-analysis check, not just a code-review convention.

| Paradigm | Where | Invocation | Agent? |
|---|---|---|---|
| Legacy procedural | `/library/*.inc.php`, `/interface/**` | `require_once`, globals | No (forbidden) |
| Modern PSR-4 | `/src/**` (`OpenEMR\`) | DI, `BaseService`, `QueryUtils` | Yes (preferred) |
| Zend MVC modules | `/interface/modules/zend_modules/...` | Zend dispatcher | Read-only via FHIR |
| Symfony subscribers | `/src/RestControllers/Subscriber/*` | EventDispatcher | Yes (hooks) |

The agent integrates *only* through the modern PSR-4 surface (`/src`) and the Symfony event subscriber boundary (`/src/RestControllers/Subscriber/ApiResponseLoggerListener.php`). The legacy `/library` and `/interface` PHP is **read-by-FHIR-only**; the agent never `require_once`'s a legacy file.

- **Agent integration implication:** This is a build-level rule, not a runtime hope. The agent module's autoloader scope must exclude `/library` and `/interface`. Static-analysis check: any `use` or `require_once` that resolves outside `OpenEMR\` namespaced classes fails CI.

### 3.2 The REST and FHIR surface is the only stable contract

The only sane place to plug in is OpenEMR's REST and FHIR API. That gives us a documented contract, OAuth2 auth, and the audit-log hook automatically. The agent runs as an *external* service that calls FHIR over the network — never as a PHP module that gets loaded inside OpenEMR's process. That separation is what gives us a clean BAA boundary and keeps OpenEMR's internals out of our blast radius.

`_rest_routes.inc.php` (project root) is the master route map; `apis/` and `src/RestControllers/` host the controllers. The Symfony EventDispatcher fires `KernelEvents::TERMINATE` after every REST/FHIR response, which is where `ApiResponseLoggerListener` logs (`ApiResponseLoggerListener.php:32-37`).

The integration seam:

```
┌───────────────┐  OAuth2   ┌──────────────────────┐    ┌─────────────────┐    ┌──────────┐
│ Agent Service │──────────►│ /apis/{site}/fhir/*  │───►│ RestControllers │───►│ Services │───► DB
└───────────────┘           └──────────────────────┘    └─────────────────┘    └──────────┘
                                                                  │
                                                                  ▼
                                                       ┌──────────────────────┐
                                                       │ Symfony EventDispatcher │
                                                       └──────────────────────┘
                                                                  │
                                              ┌───────────────────┼───────────────────┐
                                              ▼                                       ▼
                                  ApiResponseLoggerListener                AuthorizationListener
                                       (conditional)
```

- **Agent integration implication:** The agent registers as an OAuth2 client (per `Documentation/api/AUTHENTICATION.md`); never as a PHP module that calls FHIR controllers in-process. This buys us the audit trail, the ACL recheck on `AuthorizationListener`, and a clean BAA boundary. Custom UI widget under `/interface/modules/custom_modules/` is fine because it only renders an iframe to the external agent service.

### 3.3 Data lives in MySQL with ADODB-surface legacy and Doctrine-surface modern

OpenEMR talks to MySQL two different ways — a modern Doctrine path for new code and a legacy compatibility path for old code. The agent itself never opens a database connection. Where we genuinely need a time-window query that FHIR can't express, only the gateway holds a read-only replica handle, and every query gets an ACL re-check and an audit row before it fires. The reason isn't really performance — it's that direct DB reads from agent code would bypass the event dispatcher where audit logging lives.

Per `CLAUDE.md` ("Technology Stack"): MySQL via Doctrine DBAL 4.x with an ADODB compatibility surface for legacy code. New code uses `DatabaseConnectionFactory`; legacy uses the `sqlStatement()` family in `library/sql.inc.php`.

The **agent process** never opens a MySQL connection. All reads go through either the FHIR R4 surface or, where time-window queries demand it, a MySQL **read replica accessed exclusively by the gateway** on a read-only DB user, with an ACL re-check per query and an audit row written before the query fires (see `ARCHITECTURE.md` §2 Layer 5 / Decision 25). The reason is not performance — it's that any direct DB read from the agent process would bypass the Symfony event dispatcher (where API audit logging lives) and risk pulling a legacy `library/*.inc.php` include.

- **Agent integration implication:** Hard constraint — only the gateway holds the read-replica handle; the agent only ever sees structured tool results. All writes route through FHIR. The replica is never reachable from agent code paths and never carries write credentials.

### 3.4 Custom-modules surface is real and active

OpenEMR has a working extension point for custom modules — eight third-party modules already ship in this fork. We use that surface for exactly one thing: a tiny module that renders an iframe pointed at our external agent service. No business logic in the module. That keeps OpenEMR's audit responsibility off our plate for the in-process side.

`interface/modules/custom_modules/` already contains 8 third-party modules:

- `oe-module-claimrev-connect`, `oe-module-comlink-telehealth`, `oe-module-dashboard-context`, `oe-module-dorn`
- `oe-module-ehi-exporter`, `oe-module-faxsms`, `oe-module-prior-authorizations`, `oe-module-weno`

Skeleton (`oe-module-custom-skeleton`) is referenced from research. Modules bootstrap via `config/module.config.php` and event subscriptions.

- **Agent integration implication:** The MVP UI widget is a *minimal* custom module that does one thing: render an iframe pointed at the external agent service, with the OpenEMR session token passed via signed JWT. No agent business logic in the module. This keeps the OpenEMR audit responsibility off our plate for the in-process side.

### 3.5 Coupling risk: globals and `$GLOBALS` still service-locate in legacy code

The legacy code uses PHP globals as a poor-man's service locator — variables that live everywhere and could be set by anyone. The modern path wraps that in a typed object instead. The agent module must use the modern wrapper exclusively, and we enforce it with a custom static-analysis rule that fails the build on any direct global access.

`CLAUDE.md` ("Legacy Code Is Not the Standard") admits the legacy paradigm uses `$GLOBALS` as a service locator, untyped arrays through layers, and scattered `$_SESSION` reads. The modern path uses `OEGlobalsBag` (`src/Core/OEGlobalsBag.php`), a Symfony ParameterBag wrapper. The agent integration must use `OEGlobalsBag::getInstance()` only, never `$GLOBALS` directly.

- **Agent integration implication:** Add a custom PHPStan rule (the project already runs custom rules — `tests/PHPStan/Rules/`) that fails any agent-module file using `$GLOBALS`, `$_SESSION` direct access, or `mysqli_query`.

---

## 4. Data Quality

This section catalogs the schema-level shapes that turn into agent failure modes — places where missing, ambiguous, or denormalized data becomes a clinical risk if the agent treats it naively. Per `kb/research/healthcare-personas-and-usecases.md:132`, omission rate (3.45%) > hallucination rate (1.47%) in clinical summarization, so false negatives — the things the agent fails to report — are the bigger risk.

### 4.1 `lists` is one table for problems, allergies, medications, surgeries, dental, IPPF — distinguished only by `type` (P1)

Problems, allergies, meds, surgeries — they all live in the same denormalized table, separated only by a type column. The columns that matter for an allergy (severity, reaction) are nullable and nothing forces them to be filled in. So an allergy row with an empty severity field is indistinguishable from "we never asked." If the agent collapses NULL into "no" — "no severe allergies on file" — and the data was simply never entered, that's how the patient gets penicillin. The agent must always report provenance and completeness, not just content.

`sql/database.sql:7671-7715`. The columns are a union of fields meaningful only to specific `type` values, with no FK constraint enforcing which fields a row of a given type must populate:

- Injury fields: `injury_part`, `injury_type`, `injury_grade`.
- Allergy fields: `severity_al`, `reaction`, `verification`, `external_allergyid`.
- Medication-routing fields: `erx_source`, `erx_uploaded`.
- Problem fields: `outcome`, `destination`.

Practical effect: an allergy row with empty `reaction` and `severity_al` is indistinguishable from "data not entered" vs. "no known reaction." A problem row with `enddate=NULL` and `activity=NULL` is ambiguous. Drift between sites is high.

- **Agent failure mode (hospitalist):** Agent says "no known drug allergies" because rows have empty `reaction`. Reality: data was never entered. Patient gets penicillin. Anaphylaxis.
- **Agent integration implication:** The agent's allergy/problem/med tools must report **provenance and completeness, not just content**. "1 allergy on file (penicillin, severity=NULL — data not entered)" is the right answer; "no severe allergies" is the wrong answer. Agent contract: never compress NULL into "no" without an explicit data-quality note.

### 4.2 `pnotes` `activity` and `deleted` are both flags with no FK to a reason (P2)

A clinical note can be hidden two different ways — soft-deleted, or marked inactive — and there's no single flag to check. If the agent gets that filter wrong it either shows a deleted note as a real overnight event, or hides an active note. The second one is the silent failure that costs you a neuro exam decision. We wrap pnotes reads in a service adapter so the LLM never sees those flags directly.

`sql/database.sql:8670-8689`. `activity tinyint(4)` and `deleted tinyint(4) DEFAULT 0` both gate visibility. Soft-deleted notes are recoverable but the agent must filter `deleted=0` *and* `activity!=0` (or per the controller's convention) consistently. There is no single flag to check.

- **Agent failure mode (hospitalist):** Agent shows a deleted note as overnight event, or hides an active note as "deleted." Both are bad; the second is the silent failure that kills the hospitalist's neuro-exam decision.
- **Agent integration implication:** Wrap pnotes reads in a service-layer adapter that codifies "visible note" — never let the LLM see `deleted` and decide what to do with it.

### 4.3 `patient_data` has 60+ free-text columns and weak demographics indexing (P2)

The patient demographics table has 60-plus free-text columns — language, religion, interpreter needs, eight numbered "user text" slots — and most of them have no modified-time. So when the agent reads "preferred language: Spanish," it has no idea whether that was set during registration in 2014 or yesterday. A resident structuring an encounter in Spanish based on stale data is its own kind of harm. We surface a last-modified timestamp alongside every demographic value so the user can judge staleness.

`sql/database.sql:8334-8420+`.

- Useful indexes: `idx_patient_name (lname, fname)`, `idx_patient_dob (DOB)`.
- Inconsistent free-text fields: `interpreter`, `interpreter_needed`, `language`, `religion`, `referrer`, `usertext1..8`, `userlist1..7`.
- Multiple "title" fields, two SSN fields elsewhere.
- Address is a single row — no historical address tracking.

- **Agent failure mode (resident):** Agent asserts "patient prefers Spanish" based on `language` field which was set at registration in 2014 and never updated. Resident structures the encounter in Spanish; patient is now most-comfortable in English.
- **Agent integration implication:** Demographics tools must surface *modified-time* alongside the value when it exists. For fields that lack a modified_time column, surface the patient_data row's `last_update` and let the user judge staleness.

### 4.4 No formal duplicate-record reconciliation (P1 for the agent)

OpenEMR has no master patient index — no automated way to detect that the same human got registered twice and ended up with two different patient IDs. The hospitalist asks for AM labs on patient 12345; the labs are filed under 67890 because registration created a duplicate record this morning. The agent confidently says "no AM labs." Omission again. Our patient-resolution tool has to actively search for siblings by name, DOB, and partial SSN before claiming "no data."

OpenEMR has manual patient-merge tools (`patient_data` carries a `pubpid`/`pid` distinction) but no automated `MASTER_PATIENT_INDEX` or canonical identity store. Two encounters for the same human can have two different `pid`s if registration created a second record.

- **Agent failure mode (hospitalist):** Census query returns one `pid`; the patient's labs are under another `pid`. Agent says "no AM labs"; reality: AM labs exist, on the duplicate. *Omission again.*
- **Agent integration implication:** The agent's "patient resolution" tool must accept a `pid` *and* an MRN-equivalent (`pubpid`, `external_id`) and explicitly *check for siblings by name+DOB+last4SSN before claiming "no data."* Refusal is the right answer when sibling-match confidence is mid.

### 4.5 Encounter `date_end`, `discharge_disposition`, and discharge readiness markers are nullable and not normalized (P1 for use case 4)

"Is this patient ready to discharge" is not a field in OpenEMR. The pieces that actually answer it — pending consults, pending cultures, transport, SNF bed, family meeting, med rec — are scattered across notes, orders, and free-text. So the agent cannot answer "discharge ready: yes/no." What it *can* do is enumerate every blocker on a fixed checklist and report each one's evidence or absence. The verdict stays with the hospitalist; we provide the audit trail behind it.

`sql/database.sql:2022-2062`. `date_end DATETIME DEFAULT NULL`, `discharge_disposition VARCHAR(100) NULL DEFAULT NULL`. These are the only structural signals for "is this patient discharged." Discharge *readiness* (the use case) requires multi-source dependency tracking — pending consults, pending cultures, SNF bed status — none of which has a single dedicated table; they live in `pnotes`, `procedure_order`, free-text in `form_clinical_notes`, and outside-the-record case-management notes.

- **Agent failure mode (hospitalist, use case 4):** Agent says "discharge ready" because `date_end` is set in the orders by mistake; reality, urine culture is still pending. Patient gets discharged with growing E. coli.
- **Agent integration implication:** Use case 4 cannot be implemented as "check field X." The `get_discharge_blockers` tool must enumerate a fixed checklist (consults, micro pending, transport, SNF bed, family meeting, med rec) and return *each one's evidence or absence* — not a synthesized "ready/not ready" verdict. The verdict belongs to the hospitalist.

---

## 5. Compliance / HIPAA

This section maps OpenEMR's stock behavior against the six rules called out in `Week 1/PROJECT.md` Q4. The throughline: OpenEMR's audit, retention, and BAA posture do not, by themselves, satisfy HIPAA — the agent service has to fill those gaps and own the system of record.

### 5.1 Audit logging is conditional and structurally weak (P0)

OpenEMR's API audit log only fires when an admin has flipped a global setting on, and when it *is* on, it dumps the entire JSON response — full PHI in plaintext — into a free-text comments column. The audit table also isn't append-only at the database level, so anyone with normal write access can rewrite history. None of this meets HIPAA's audit-control requirement on its own. We have to keep our own append-only audit store and treat OpenEMR's log as redundant.

The flow, with the failure points marked:

```
                                        ┌── conditional on api_log_option>0
                                        ▼          (admin can disable)
API request ──► Symfony EventDispatcher ──► ApiResponseLoggerListener
                                                       │
                                                       ▼
                                        log.comments  (PHI in plaintext,
                                                       no column encryption,
                                                       no append-only constraint)
```

The findings together:

- API logging is gated on `OEGlobalsBag::getInt('api_log_option') > 0` (`src/RestControllers/Subscriber/ApiResponseLoggerListener.php:56`). An admin can switch it off; nothing enforces that it stays on for HIPAA §164.312(b) audit controls.
- When logging *is* on, the listener writes the **full JSON response body** into `log.comments` (`ApiResponseLoggerListener.php:62-86`, `sql/database.sql:7758-7776`). That is PHI in plaintext in the audit log itself, with no encryption-at-rest at the column level.
- Neither `log` nor `audit_master` is append-only at the schema level. Both are InnoDB tables with normal `DELETE` and `UPDATE` privileges — a DBA or anyone with `DROP`/`UPDATE` rights can rewrite history.

Independent confirmation: searched for `EventAuditLogger::getInstance` or `newEvent(` in `_rest_routes.inc.php` — **zero hits**. The route file itself does no audit logging; it relies entirely on the post-response listener firing. If the listener throws or `api_log_option=0`, the call is invisible.

- **HIPAA citation:** §164.312(b) audit controls require recording and examining activity in systems containing ePHI. §164.530(j) requires retaining documentation 6 years. OpenEMR's stock setup violates both implicitly (loggable-but-not-required + no retention enforcement).
- **Agent integration implication:** The agent service maintains its own append-only audit table (write-only role, separate database user, daily integrity hash, ≥6-year retention) keyed by `(user_id, patient_id, tool_name, request_hash, response_hash, timestamp, decision)`. It logs **the request and the redacted-response-fingerprint, never the full PHI response body.** That table is the system of record for §164.312(b) for everything the agent touches. Per `PROJECT.md`: "PHI in plain log files" is on the explicit fail list — we must not replicate OpenEMR's mistake.

### 5.2 No automatic break-glass or emergency-access flow at the FHIR boundary (P1)

"Break-glass" is the emergency-access pattern — a clinician overrides normal permissions to see a patient they wouldn't otherwise have rights to, with extra logging. The plumbing for it exists in OpenEMR but isn't wired into the FHIR routes. Not a blocker for us this week because hospitalists work off attending-of-record relationships, not break-glass. It becomes a blocker the day someone wants to extend this to the ED.

`src/Common/Logging/BreakglassChecker.php` and `BreakglassCheckerInterface.php` exist (63 lines total in the checker) and there is an `Audit/` subdirectory under `Common/Logging/`. The infrastructure exists. There is no evidence in `_rest_routes.inc.php` that FHIR controllers route through it on an emergency-access path. This is a hospitalist non-issue (we are using attending-of-record relationship, not break-glass — see `kb/research/healthcare-personas-and-usecases.md:24,32`) but it is a future-ED-blocker.

- **HIPAA citation:** §164.312(a)(2)(ii) requires emergency access procedure.
- **Agent integration implication:** Document in `ARCHITECTURE.md` that break-glass is out of scope for week 1 (hospitalist only). When ED extension is built later, route through `BreakglassChecker` and require reason-code + audit escalation per Yale's published procedure (`kb/research/healthcare-personas-and-usecases.md:116`).

### 5.3 BAA boundary — LLM provider, observability vendor, hosting (P0 for any real-PHI deploy)

A BAA — Business Associate Agreement — is the contract HIPAA requires between us and any vendor that touches PHI. OpenEMR doesn't have one with Anthropic, with Railway, or with our observability vendor. *We* are the business associate that has to cover that gap, which means we need a signed BAA with each of those three before any real patient data flows. If even one is unconfirmed, the demo runs on synthetic data and we label it that way on the deck.

OpenEMR has no BAA with our LLM provider, by definition. The agent service is the BA covering that gap. Per `Week 1/PROJECT.md` open question 3-4:

| Vendor | BAA path | Status this week |
|---|---|---|
| Anthropic API | Zero-retention enterprise tier with BAA | Required before any real PHI |
| Railway hosting | HIPAA tier with BAA | Verify before deploy |
| Observability — week-1 demo: LangSmith; production target: self-hosted Langfuse | LangSmith Hobby for the synthetic-data demo (no PHI in traces, no vendor BAA needed); for any real-PHI deployment, migrate to self-hosted Langfuse inside the same Railway project so the BAA boundary is preserved without a third-party observability vendor | LangSmith fine for week-1 (synthetic only); Langfuse self-host is the documented production migration |

OpenEMR's audit gaps (5.1) plus our LLM provider's logging window mean *we* are responsible for the end-to-end audit of every PHI-touching call. No "the LLM provider has it" — they don't, by contract.

- **Agent integration implication:** Hard pre-deploy gate. If any of the three BAAs is unconfirmed, the deploy is synthetic-data only and labeled as such in the demo. Document in `ARCHITECTURE.md` cost section and on the deck (slide 12).

### 5.4 Transmission security — TLS termination location is the operator's call (P1)

The default OpenEMR development stack runs on plain HTTP. That's fine for local dev, not fine for anything else. We need to verify TLS 1.3 minimum on the public edge and make sure the agent-to-OpenEMR hop inside the network is also encrypted, not internal-network plaintext. The agent gateway should refuse any inbound request that isn't TLS.

OpenEMR ships with HTTPS support but the `docker/development-easy/` stack uses HTTP on port 8300 (`CLAUDE.md:38-40`). TLS termination on Railway happens at the edge by default. Verify that TLS 1.3 is the minimum and that the agent-to-FHIR hop is also TLS, not internal-network plaintext.

- **HIPAA citation:** §164.312(e)(1) transmission security.
- **Agent integration implication:** Agent gateway refuses any inbound request not over TLS 1.3. Outbound to OpenEMR FHIR endpoint over TLS only; mTLS if Railway supports it, otherwise OAuth2 bearer token over TLS with short expiry.

### 5.5 Minimum necessary — agent must self-restrict even though treatment is exempt (P1)

HIPAA technically exempts treatment from its "minimum necessary" rule, but that doesn't mean we should pull every record we can. The agent self-restricts: each tool reads one slice (meds tool returns meds, not problems), and sensitive segments — mental health, substance use, reproductive, HIV — are default-deny unless the clinician explicitly opts in for that encounter. This is policy we set, not policy OpenEMR enforces.

HIPAA §164.502(b) exempts treatment from minimum-necessary, but `kb/research/healthcare-personas-and-usecases.md:24, 116-118` calls out the agent posture: scope per-use-case, default-deny on sensitive segments (mental health, SUD per 42 CFR Part 2, repro, HIV) without explicit per-encounter affirmation. OpenEMR's `form_encounter.sensitivity` field (`sql/database.sql:2032`) is one signal, not a full segmentation.

- **Agent integration implication:** Agent tool registry caps each tool's read scope (e.g., `get_med_history` returns meds only, not problems-or-allergies). Sensitive-segment opt-in per-encounter, with refusal as the default. Documented in `USERS.md`.

### 5.6 Retention enforcement is not a schema property (P1)

HIPAA wants six years of audit retention. OpenEMR's audit tables have no retention policy and no partitioning to support enforcing one — it's expected to be done externally, on the honor system. Our own audit store handles this with monthly partitions and a write-once backup, with the retention policy written down as an attribute of the store rather than left implicit.

Neither `log` nor `audit_master` has a retention policy column or a partitioning scheme that supports a 6-year retention boundary (`sql/database.sql:149-162, 7758-7776`). Operators are expected to retain externally.

- **HIPAA citation:** §164.530(j) — six-year documentation retention.
- **Agent integration implication:** Agent's own audit store uses monthly partitions + immutable WORM storage backup; retention policy is a written attribute of the store, not a hope. Document in `ARCHITECTURE.md`.

### 5.7 Default credentials in the deploy image (revisited — P0)

Same finding as 1.2, but called out again here because under HIPAA it implicates two specific rules — password management and unique user identification. Worth repeating because it's the single most likely real-world breach for any team that demos OpenEMR on a public URL.

Already covered in §1.2. Repeated here because §164.308(a)(5)(ii)(D) (password management) and §164.312(a)(2)(i) (unique user identification) are both implicated when `admin/pass` is published. **This is the highest-likelihood real-world breach vector for any team that demos OpenEMR on a public URL.**

---

## Summary table — findings indexed by use case

| Finding | P | Hospitalist overnight delta (UC-1) | Discharge readiness (UC-2) | Resident audit (UC-3) |
|---|---|---|---|---|
| 1.2 Default creds | P0 | ✓ blocks | ✓ blocks | ✓ blocks |
| 1.3 ACL helper silent fail | P1 | ✓ | ✓ | ✓ |
| 1.5 Portal session conflation | P1 | ✓ | ✓ | ✓ |
| 2.2 `pnotes` no date index | P0 (UC) | ✓ direct | indirect | ✓ direct |
| 2.3 `procedure_report` no date index | P1 | ✓ direct | ✓ direct | ✓ |
| 4.1 `lists` flat denormalized | P1 | ✓ (allergies/meds) | ✓ (active meds) | ✓ |
| 4.4 Duplicate `pid` reconciliation | P1 | ✓ omission risk | ✓ | ✓ |
| 4.5 Discharge fields nullable | P1 | — | ✓ direct | indirect |
| 5.1 Audit logging conditional | P0 | ✓ | ✓ | ✓ |
| 5.3 BAA chain | P0 | ✓ | ✓ | ✓ |
| 5.7 Default creds (HIPAA framing) | P0 | ✓ | ✓ | ✓ |

---

## Open items that this audit cannot close

| # | Open item | Why it's open | Action |
|---|---|---|---|
| 1 | Live database row counts | Need a seeded instance to confirm whether missing indexes on `pnotes(date)` and `procedure_report(date_report)` trigger query-plan regressions at Synthea volume. | Rerun the perf section after Stage 1 (local Docker run with seed data). |
| 2 | `api_log_option` default | Listener honors the global, but the install-script default on first boot is unverified. | Trace `Installer.class.php`; document the default in the `ARCHITECTURE.md` deploy checklist. |
| 3 | PHPStan baseline drift | Modern code is level 10 clean *with* a baseline; baseline size is a proxy for residual legacy debt. | Run `composer phpstan` post-setup and capture the count for the architecture defense. |
| 4 | CVE list since 8.1.1-dev tag | Research file references CVE-2026-33917 and v5.0.1.4 SQLi; the fork's dev tag means we must pin to a release commit before any production deploy and re-audit. | Document the pin and re-audited CVE list in the `ARCHITECTURE.md` deployment section. |

---

**End of audit.** Findings above are file-anchored. Where a claim is from secondary research (not from the fork itself), it is cited to the kb research files, not asserted as a code-confirmed fact.
