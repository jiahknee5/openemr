# ARCHITECTURE.md — AgentForge Clinical Co-Pilot

**Anchored to:** `USERS.md` (target user, three use cases, failure modes), `AUDIT.md` (security/perf/architecture/data-quality/HIPAA findings), `kb/research/openemr-codebase-walkthrough.md` (data model, FHIR R4 surface, integration paths).

> Every architectural decision in this document traces back to a specific finding in `AUDIT.md` or a specific use case in `USERS.md`. If a component does not have such a trace, it is out of scope.

---

## How to read this document

- Each section opens with a 1-2 sentence non-technical framing, then drops into the technical specifics and diagrams. The two paragraphs are meant to read as one.
- Numbered cross-references (`§3`, `§6 Step 5`, etc.) point to specific sections so a peer can verify a claim without searching.
- The trace tables in §4 and the failure-mode matrix in §10 show how user needs from `USERS.md` map to architecture choices, and which layer catches each failure mode.

---

## Executive Summary

**Five layers, five tools, three-orders-of-magnitude payback (see §8), structurally cannot ship an unverified claim. The verifier is the keystone — every other decision serves it.**

### The five-layer architecture

```
Client widget  →  Gateway  →  Agent runtime  →  Verifier  →  Data plane (FHIR)
   refuse          deny w/      refuse on cap    degrade or       401/403
   banner          reason                          refuse         propagate
```

Each layer fails in a different direction so a single failure cannot ship a wrong answer.

### Key decisions

Each row names the choice we made *and* the alternatives we rejected, with the reason for the rejection. Full rationale per decision in the Appendix at the end of this document (`§A.1`–`§A.9`).

| Choice | Decision | Why this — and why not the alternatives |
|---|---|---|
| **Integration path** | External Python service + thin OpenEMR PHP widget | **Not pure in-process custom module:** `AUDIT.md` §1.4 — legacy SQL surfaces in `/library` are one `require_once` away from the agent's runtime. **Not iframe-only with no module:** loses identity-forwarding seam. (See §A.1) |
| **LLM provider** | Anthropic Claude — Sonnet 4.6 (~85% of calls) + Opus 4.7 escalation (~15%) | **Not OpenAI GPT-4/5:** no zero-retention enterprise BAA tier comparable to Anthropic's. **Not self-hosted Llama/Qwen:** tool-use reliability not yet at parity for clinical synthesis. **Not multi-provider router:** cache prefix can't be shared, doubles cost and complexity. (See §A.3) |
| **Agent framework** | LangGraph state machine + Anthropic SDK direct | **Not LangChain framework:** abstraction layer not earned for a 5-tool surface; couples our code to vendor release cadence. **Not CrewAI / AutoGen:** multi-agent overhead with no use case requiring it. **Not raw SDK with custom loop:** LangGraph's explicit state machine is the cleanest seam for verifier integration. (See §A.2) |
| **Verifier** | Deterministic post-LLM Python (5 checks) | **Not LLM-as-judge at runtime:** shifts the safety property from code to a stochastic process; +1500ms latency; fails hospital security review ("show me the code that guarantees this"). **Not self-critique by same LLM:** correlated blind spots. **Not prompt-engineering-only:** empirical ceiling ~92–95% citation faithfulness; the gap from there to 100% is the entire reason the system is deployable. (See §A.4) |
| **Tools** | **5 maximum** — `list_overnight_events`, `get_labs_window`, `get_med_history`, `get_consult_status`, `get_discharge_blockers` | **Not 10+ rich tools:** orchestrator confusion threshold ~7 (Ash's lecture) — the model confuses near-neighbors and skips required calls. Composition over expansion. (See §A.9) |
| **Data egress** | FHIR R4 read-only via OAuth2 / SMART-on-FHIR | **Not direct DB read (MySQL replica):** forces through legacy `library/*.inc.php` SQL interpolation paths (AUDIT.md §1.4). **Not custom REST shim around OpenEMR:** re-implements OpenEMR's auth surface, doubles maintenance. **Not in-process `QueryUtils` calls:** same legacy-include risk as integration path. (See §A.5) |
| **Deployment** | Railway (single project, agent + OpenEMR + MariaDB) | **Not AWS / GCP / Azure full stack:** time-to-deploy too long for a one-week demo. **Not Render / Fly.io / Heroku:** no current path to a HIPAA-tier BAA; Railway has one. **Not Vercel:** stateful agent service doesn't fit the serverless model. (See §A.6) |
| **Observability** | LangSmith (week-1 demo, synthetic data only) + OpenTelemetry + Sentry; **production target is self-hosted Langfuse** inside the Railway BAA boundary | **Not single-vendor (LangSmith for production):** third-party PHI traces are BAA-fragile. **Not Datadog only:** not LLM-aware. **Not OpenTelemetry alone:** not LLM-aware (tool calls and prompts don't survive the standard span model). **Not the custom audit table alone:** different audience (engineers vs. privacy officer), different schema, different mutability — they cannot be the same thing. (See §A.7) |
| **Eval methodology** | Held-out clinical scenarios with deterministic ground truth + verifier-pass-rate as primary metric + LLM-judge for narrative quality (eval-time only, never runtime) | **Not runtime LLM-judge:** same auditability and latency objections as the verifier alternative. **Not MedQA-only / academic benchmark:** doesn't measure omission, which is our P0 failure mode. **Not manual rubric scoring only:** doesn't scale to weekly regressions on every code change. **Not "no formal eval":** the omission rate (3.45% per *Nature*) has to be measurable to be defendable. (See §A.8) |

### Cost envelope

| Metric | Value |
|---|---|
| **Cost (order-of-magnitude estimate, see §8 caveat)** | Per hospitalist per morning ≈ tens of cents; per-year ≈ low thousands of dollars per hospital. Recovered attending time at the same scale ≈ low millions. **Three-orders-of-magnitude payback.** |
| Latency target | **<3s p50, <6s p95** |

### The five tools

| Tool | Used by | Returns |
|---|---|---|
| `list_overnight_events` | UC-1, UC-3 | RN notes, vitals deltas, micro, med events with citations `(table, row_id, timestamp, author_user_id)` |
| `get_labs_window` | UC-1, UC-2, UC-3 | `procedure_result` rows in time window |
| `get_med_history` | UC-1, UC-3 | `prescriptions` events with HOLD/DC/start, joined to `audit_master` |
| `get_consult_status` | UC-2 | Open consult requests + responses |
| `get_discharge_blockers` | UC-2 | Structured blocker list across 7 categories (consults, micro, placement, transport, DME, family, follow-up) |

### Out of scope (explicit)

Write-back to OpenEMR · break-glass mode · patient-portal integration · cross-session memory · PCP/ED/Specialist/Nurse personas · multi-tenant isolation · SMART-on-FHIR launch (week-2).

### For the demo

> "Five layers, five tools, three-orders-of-magnitude payback (cost vs. recovered time, see §8), structurally cannot ship an unverified claim. The verifier is the keystone — every other decision serves it."

---

## 1. The Five-Layer Trust Boundary

The system is built as five layers stacked on top of each other, and a request has to pass through all of them before the user sees an answer. Each layer has one job, and more importantly, each layer is designed to fail in a different direction so that no single bug, outage, or mistake can ship a wrong clinical answer past the user. If the gateway breaks, you get a clear 503. If the agent gets confused, you get a refusal. If the verifier finds a gap, you get a flagged-incomplete answer instead of a fluent lie. Read the diagram top-to-bottom — that is the actual path of a request.

### The five layers, in plain English

Imagine Maya at 6:35 AM in front of her workstation. She clicks the AgentForge sidebar inside the patient's chart in OpenEMR. What happens next is a chain of five short steps, each handled by a different piece of the system. We split the work this way on purpose: each piece has one job, and each piece is designed to fail in a different direction so that no single mistake can ship a wrong clinical answer to her screen.

**Layer 1 — The widget on Maya's screen.** This is the only piece Maya actually sees. It is a small panel that lives inside OpenEMR's existing chart view, and it deliberately does not contain any AI. Its job is to grab Maya's hospital login session, package it up, and forward the question to the next layer. We kept it dumb so that we could change the agent without redeploying OpenEMR, and so that we couldn't accidentally pull any of OpenEMR's older code into the agent's runtime. If the widget cannot reach the agent, it shows Maya a banner that says "AgentForge unavailable — open the chart directly." It never gives a half-answer.

**Layer 2 — The gateway, which is the bouncer.** Every request from the widget hits this layer first. Before any AI runs, the gateway does five small things in order: it checks Maya's identity (is the session valid?), it confirms with OpenEMR that Maya is allowed to see this specific patient, it writes a row to our own append-only audit log, it checks she hasn't already used today's AI budget, and only then does it hand the request off. If any of those fail, the gateway returns a structured error — "session expired," "not on this patient's care team," "audit log unavailable" — and the request stops. We own the audit log ourselves because OpenEMR's built-in log is conditional, mutable, and stores PHI in plaintext (see AUDIT.md §A); none of those properties pass a HIPAA review.

**Layer 3 — The agent runtime, where the LLM actually thinks.** This is the piece that uses Claude. It runs as a state machine: it reads the question, plans which tools to call (out of the five tools listed in §3), calls them, and assembles a draft answer. Sonnet 4.6 handles routine cases; Opus 4.7 takes over when the synthesis is genuinely hard. Importantly, the runtime is on a leash. It can iterate at most five times, and on the fifth try it returns a flagged-incomplete summary instead of pretending it finished. The LLM never makes the final call about what Maya sees on her screen — that decision belongs to the next layer.

**Layer 4 — The verifier, which is the safety net.** This is the most important layer in the whole system. After the LLM has written its draft answer, but before Maya sees it, a separate piece of plain Python code re-checks every claim against the actual data. It runs five checks in order: every claim must cite a real record, every timestamp must fall inside the requested time window, the count of cited rows must match an independent database count we run ourselves, every medication mentioned must reference its most recent prescription event, and resident drafts must be wrapped in a non-copyable "DRAFT" frame. If any check fails, the agent admits the gap with a phrase like "I found seven events but counted nine; please review the source" — it never strips a claim silently. This layer is what makes the system safe to put in front of a clinician. Cite-or-refuse is enforced in code here, not by polite instructions to the LLM in the prompt.

**Layer 5 — The data plane, which is OpenEMR's existing FHIR API.** This is where the agent actually reads patient data from. We do not give the agent a database connection; we give it OAuth2 credentials to OpenEMR's existing read-only FHIR endpoints (the same standardized API any modern clinical system exposes). Layer 2 calls it once for ACL checks; Layer 3's tools call it for the actual patient data. There are no write endpoints in scope this week — the agent cannot edit OpenEMR. If FHIR returns a 401 or 403 (authentication or permission failure), the error propagates back up to Layer 2, the gateway writes an audit row about it, and Layer 1 shows Maya the banner.

**Two side systems write the audit and the observability data in parallel.** They are not part of the request path; they are downstream consumers of every layer's activity. The **audit store** (a custom append-only Postgres table, six-year retention, tamper-evident) is the HIPAA system of record — it answers the regulator's question "who saw what patient when?" The **observability stack** (LangSmith for the week-1 demo, self-hosted Langfuse for production, plus OpenTelemetry and Sentry — see Layer 3 detail in §2) is the engineer's debug view — it answers "why did this query take four seconds and refuse?" These are different things solving different problems and we will not conflate them.

### How to read the diagram

The main column (Layers 1–4, top to bottom) is the **path of one request** — Maya's browser at the top, Maya's screen at the bottom, and the request flows through the layers in between. The right-hand boxes are **what each layer does when it fails** — every layer has a different failure direction, which is the whole point. The left-hand boxes are the **two side systems each layer writes to** (audit store, observability stack). They are parallel, not in series. Layer 5 (data plane) is shown separately at the bottom because it is *called by* Layers 2 and 3 rather than sitting in the request path.

```
                       ┌───────────────────────────────────┐
                       │  Maya (browser, OpenEMR encounter)│
                       └─────────────────┬─────────────────┘
                                         │  HTTPS + session cookie
                                         ▼
   ╔═══════════════════════════════ TRUST BOUNDARY ═══════════════════════════════╗
   ║                                                                              ║
   ║   ┌────────────────────────────────────────────────┐                         ║
   ║   │ LAYER 1 · CLIENT WIDGET                         │   on widget error      ║
   ║   │ ─────────────────────────────────────────────── │   ┌───────────────┐    ║
   ║   │ Where: /interface/modules/custom_modules/       │──▶│ Banner:       │    ║
   ║   │        oe-module-agentforge/  (PHP module)      │   │ "AgentForge   │    ║
   ║   │ Owns:  forward identity, render structured      │   │  unavailable  │    ║
   ║   │        response. NO LLM, NO business logic.     │   │  — open chart │    ║
   ║   │ Calls: POST /agent/run + signed JWT identity    │   │  directly"    │    ║
   ║   └─────────────────────────┬──────────────────────┘   └───────────────┘    ║
   ║                             │ POST + X-OpenEMR-Identity (60s JWT)            ║
   ║                             ▼                                                ║
   ║                                                                              ║
   ║   ┌────────────────────────────────────────────────┐                         ║
   ║   │ LAYER 2 · GATEWAY (FastAPI on Railway)          │   on auth/ACL fail     ║
   ║   │ ─────────────────────────────────────────────── │   ┌───────────────┐    ║
   ║   │ Six steps, every request, in order:             │──▶│ 401/403 with  │    ║
   ║   │  (1) verify JWT (sig, exp, jti replay cache)    │   │ structured    │    ║
   ║   │  (2) resolve user via FHIR /Practitioner/{id}   │   │ reason —      │    ║
   ║   │  (3) ACL probe — round-trips OpenEMR AclMain    │   │ never opaque  │    ║
   ║   │  (4) WRITE audit row ────────┐                  │   │ 500           │    ║
   ║   │  (5) rate-limit + token cap  │                  │   └───────────────┘    ║
   ║   │  (6) hand request to Layer 3 │                  │                        ║
   ║   │                              │                  │   on audit-write fail  ║
   ║   │ Owns: auth, ACL recheck,     │                  │   ┌───────────────┐    ║
   ║   │       audit, scope, budget   │                  │──▶│ 503 — request │    ║
   ║   └────────────┬─────────────────┼──────────────────┘   │ NEVER         │    ║
   ║                │ verified         │ writes               │ proceeds      │    ║
   ║                │                  ▼                      │ without log   │    ║
   ║                │           ┌──────────────────────┐      └───────────────┘    ║
   ║                │           │ AUDIT STORE           │                          ║
   ║                │           │ ──────────────────── │                          ║
   ║                │           │ Postgres,             │   HIPAA system of       ║
   ║                │           │ append-only           │   record for            ║
   ║                │           │ (write-only DB role,  │   §164.312(b).          ║
   ║                │           │  per-row immutable),  │                         ║
   ║                │           │ 6-yr WORM retention,  │   Schema: who-asked-    ║
   ║                │           │ tamper-evident        │   what-when, no PHI     ║
   ║                │           │ daily hash chain.     │   content.              ║
   ║                │           │                       │                         ║
   ║                │           │ Detail: AUDIT.md §A.  │                         ║
   ║                │           └──────────────────────┘                          ║
   ║                ▼                                                              ║
   ║                                                                              ║
   ║   ┌────────────────────────────────────────────────┐                         ║
   ║   │ LAYER 3 · AGENT RUNTIME (LangGraph + Claude)    │   on iteration cap     ║
   ║   │ ─────────────────────────────────────────────── │   ┌───────────────┐    ║
   ║   │ Models: Sonnet 4.6 default (~85% of calls)      │──▶│ Refusal or    │    ║
   ║   │         Opus 4.7 escalation (~15% — UC-1 rank,  │   │ "checked X,   │    ║
   ║   │         UC-3 cross-text reasoning)              │   │ found N, did  │    ║
   ║   │ Adaptive thinking ON, effort=high               │   │ not complete  │    ║
   ║   │ Prompt caching on system block (~85% hit rate)  │   │ synthesis"    │    ║
   ║   │ Max 5 iterations · 5 tools (see §3)             │   └───────────────┘    ║
   ║   │                              ┌──────┐           │                        ║
   ║   │ Owns: synthesis ONLY.        │spans │           │                        ║
   ║   │       NOT ranking (rubric),  │      │           │                        ║
   ║   │       NOT verification.      ▼      │           │                        ║
   ║   └─────────┬────────────┬──────────────┴──────────┘                         ║
   ║             │            │                                                   ║
   ║             │            ▼                                                   ║
   ║             │      ┌────────────────────────┐                                ║
   ║             │      │ OBSERVABILITY STACK     │                               ║
   ║             │      │ ────────────────────── │                                ║
   ║             │      │ LangSmith (week-1 demo, │   For ENGINEERS only.         ║
   ║             │      │   synthetic data only)  │   NOT the audit store.        ║
   ║             │      │ → Langfuse self-hosted  │                               ║
   ║             │      │   (production target,   │   Different schema, 30-day    ║
   ║             │      │    BAA-internal)        │   retention, mutable.         ║
   ║             │      │ + OpenTelemetry (system │                               ║
   ║             │      │   spans, vendor-neutral)│   See AUDIT.md §A             ║
   ║             │      │ + Sentry (uncaught      │   audit-vs-observability      ║
   ║             │      │   exceptions, crashes)  │   callout.                    ║
   ║             │      └────────────────────────┘                                ║
   ║             ▼                                                                ║
   ║                       draft answer + tool-result citations                   ║
   ║                                                                              ║
   ║   ┌────────────────────────────────────────────────┐                         ║
   ║   │ LAYER 4 · VERIFIER (deterministic, post-LLM)    │   on any check fail    ║
   ║   │ ─────────────────────────────────────────────── │   ┌───────────────┐    ║
   ║   │ Plain Python pipeline, five checks in order:    │──▶│ Degrade to    │    ║
   ║   │  (1) citation completeness (claim → row_id)     │   │ "found N but  │    ║
   ║   │  (2) window integrity (ts ∈ requested window)   │   │ counted M;    │    ║
   ║   │  (3) count parity (independent SELECT COUNT(*)  │   │ review source"│    ║
   ║   │      vs LLM's cited count)                      │   │               │    ║
   ║   │  (4) med-rec freshness (last 24h cited)         │   │ NEVER strip   │    ║
   ║   │  (5) DRAFT watermark (UC-3 resident output)     │   │ silently;     │    ║
   ║   │                                                  │   │ ALWAYS tell   │    ║
   ║   │ Owns: cite-or-refuse contract. The keystone.    │   │ the user what │    ║
   ║   │       This is what makes the system safe.       │   │ was dropped.  │    ║
   ║   └─────────────────────────┬──────────────────────┘   └───────────────┘    ║
   ║                             │                                                ║
   ╚═════════════════════════════│════════════════════════════════════════════════╝
                                 │  verified answer  (or flagged-incomplete)
                                 ▼
                       ┌───────────────────────────────────┐
                       │  Maya (sees ranked, cited delta — │
                       │  every bullet click-throughs to   │
                       │  the source artifact in OpenEMR)  │
                       └───────────────────────────────────┘


   Called by Layers 2 (ACL probe) and 3 (tool fetches) — NOT in the request column above:

   ┌──────────────────────────────────────────────────────────────────────────┐
   │ LAYER 5 · DATA PLANE (read-only, OAuth2 / SMART-on-FHIR)                  │
   │ ───────────────────────────────────────────────────────────────────────── │
   │ Endpoint:  /apis/{site}/fhir/*  on the OpenEMR Railway service            │
   │ Scopes:    patient/Patient.read, Encounter.read, Observation.read,        │
   │            Condition.read, MedicationRequest.read, AllergyIntolerance.    │
   │            read, DocumentReference.read, user/Practitioner.read           │
   │ Owns:      data egress only. NO write paths week 1; agent NEVER mutates.  │
   │ Fails:     401/403 propagate to the gateway; NEVER a silent empty array.  │
   └──────────────────────────────────────────────────────────────────────────┘
                                    ▲
                                    │  FHIR resources (read-only)
                                    │
                       ┌────────────┴───────────────┐
                       │ OPENEMR (Apache + MariaDB)  │
                       │ system of record            │
                       └─────────────────────────────┘
```

**Each layer fails in a different direction so a single failure cannot ship a wrong answer.**

| Layer | Failure mode | What the user sees |
|---|---|---|
| 1 — Client | widget cannot reach gateway | Banner: "AgentForge unavailable — open chart directly" |
| 2 — Gateway | identity / ACL / audit fail | 401, 403, or 503 with structured reason |
| 3 — Agent runtime | model timeout, iteration cap, tool error | Refusal or "checked X, did not complete" |
| 4 — Verifier | citation, window, count, med-rec, or watermark check fails | "Found N but counted M; review source" |
| 5 — Data plane | OpenEMR FHIR returns 401/403 or schema mismatch | Propagated as Layer 2 audit-row + Layer 1 banner |

---

## 2. Layer-by-Layer Detail

This section walks each of the five layers in turn, naming the specific fail-safe behavior, the AUDIT finding or USERS use case it traces to, and the hard constraints that hold it in place.

### Layer 1 — Client Widget

This is the part the clinician actually sees — a small card that lives on the side of OpenEMR's existing encounter screen. It does not run any AI itself; it just collects the user's session, asks the agent service for an answer, and renders the structured response. We deliberately kept it dumb so we could iterate on the agent without redeploying OpenEMR, and so we wouldn't accidentally pull in any of OpenEMR's legacy code.

**Where:** OpenEMR custom module under `/interface/modules/custom_modules/oe-module-agentforge/`. Skeleton from [oe-module-custom-skeleton](https://github.com/adunsulag/oe-module-custom-skeleton). Drops a sidebar card on the encounter view via existing card-system hooks. **Zero schema changes** to OpenEMR. Posts to the agent service over HTTPS. Renders one card per patient for UC-1 and UC-2; renders a two-column diff for UC-3.

**Why a module + external service, not a pure custom module that runs the LLM in-process:** `AUDIT.md` §1.4 — legacy `library/*.inc.php` files still interpolate SQL variables; an in-process include risks pulling them. The module is a thin shell whose only job is to forward the user's identity and render the agent's structured response.

**Fail-safe:** if `/agent/run` returns non-200 or `status != "ok"`, the widget shows a banner: "AgentForge unavailable — open the chart directly." Never silent.

**Trace:** `USERS.md` §3 (90-second moments require an in-flow widget, not a separate tab).

### Layer 2 — Gateway

The gateway is the bouncer. Every request from the OpenEMR widget hits it first, and it does three things before letting the agent see anything: confirms who's asking, re-checks that the user has permission for that specific patient, and writes an audit row to our own append-only table. If any of those fail, the request stops here with a structured reason — never a silent denial or an opaque 500. We own the audit log ourselves because OpenEMR's built-in audit logging is conditional on a global flag and dumps full PHI JSON into a single column when it is on.

**Stack:** FastAPI (already deployed on Railway as the `agent` service: `https://agent-production-66ad.up.railway.app`).

**Responsibilities, in order, on every request:**

1. **Identity** — extract OpenEMR session ID from forwarded cookie. Resolve to `users.id` via FHIR `/Practitioner/{id}` lookup. (Demo mode: `X-Agent-Token` shared secret. Production: SMART-on-FHIR launch with token introspection.)
2. **ACL recheck** — re-evaluate `AclMain::aclCheckCore` semantics via the agent's own probe call (a FHIR query that round-trips through OpenEMR's auth). Treat OpenEMR's `false` as authoritative deny; treat `true` as a *necessary but not sufficient* signal — the gateway also enforces its own scope (must be on the user's service-of-record).
3. **Audit BEFORE compute** — write `{request_id, user_id, patient_pids, tool_intent, ts}` to the agent's own append-only Postgres `audit_log` table. Failure to write = refuse the request. **OpenEMR's audit table is not the system of record** (`AUDIT.md` summary).
4. **Rate-limit + budget** — per-user token cap per session (10K tokens) and per-day cap (200K). Stops a runaway loop from billing the BAA.

**Fail-safe behavior:**

- Identity fail → 401 with `{"reason": "session_invalid"}`.
- ACL fail → 403 with `{"reason": "no_treatment_relationship", "patient_pid": N}`.
- Audit-write fail → 503 with `{"reason": "audit_unavailable"}`. Gateway does **not** proceed to the agent.

**Trace:**

- `AUDIT.md` §5.1 (audit logging structurally weak at API boundary) → gateway owns audit.
- `AUDIT.md` §1.3 (ACL is silent fail-closed; no structured reason) → gateway returns structured reason.
- `AUDIT.md` §1.5 (portal session is a different identity class) → gateway rejects portal-origin sessions.
- `USERS.md` §8 #5 (resident must have attending-of-record link) → gateway enforces.

### Layer 3 — Agent Runtime

This is where the LLM actually thinks. A LangGraph state machine plans the work, picks tools, calls them, and assembles a draft answer. Sonnet 4.6 handles the routine cases; Opus 4.7 takes over when the synthesis is genuinely hard (a busy patient with 40-plus overnight events, or a resident draft that needs cross-text reasoning). Importantly, the runtime is on a leash — five iterations max, and on cap it returns a flagged-incomplete summary instead of pretending it finished. The LLM never makes the final call about what the user sees; that's the verifier's job.

**Stack:** Python 3.12, FastAPI host, **LangGraph** orchestrator, Anthropic SDK direct (per home `CLAUDE.md` "prefer direct SDK"). Observability for the week-1 demo is **LangSmith** (`wrap_anthropic` + `@traceable` from the `langsmith` SDK; project `agentforge-hospitalist`) — acceptable here because the demo runs synthetic data only and no PHI flows through traces. The **production target is self-hosted Langfuse** inside the same Railway project so the BAA boundary is preserved without a third-party observability vendor (see Decision 27 in §9); the Langfuse SDK is a near-drop-in for the `wrap_anthropic` / `@traceable` pattern, so the migration is a config swap rather than a rewrite. Around that LLM-span layer we run two more pieces with non-overlapping jobs. **OpenTelemetry (OTel)** is the open, vendor-neutral standard for collecting traces, metrics, and logs from applications: we instrument the FastAPI process once against OTel and that same instrumentation can fan out to LangSmith today and to self-hosted Langfuse later, which is exactly the property that makes the migration above a config change rather than a rewrite — without OTel we'd be locked into whichever observability vendor we picked first. **Sentry** is an error-tracking service that captures unhandled exceptions in production with stack traces and request context; we use it because it covers a different failure mode from the trace layer. LangSmith / Langfuse traces *successful but suspect* LLM behavior — slow responses, low cache hit rate, refusal patterns, tool-call loops — while Sentry catches *outright crashes*: an unhandled exception in the FastAPI handler, a 500 from a downstream FHIR service, a malformed schema in a tool result. Both have to be in the stack because a clinical agent that is silently throwing 500s is just as undeployable as one that is silently producing low-quality answers, and neither tool sees the other's failure mode.

**Models:**

- **Default:** `claude-sonnet-4-6` for routine tool composition. Best $/intelligence ratio.
- **Escalation:** `claude-opus-4-7` for UC-1 ranking when >40 events in window, and for UC-3 draft-vs-chart cross-text reasoning.
- **Adaptive thinking** on by default (per `claude-api` skill default for Opus 4.7). `output_config.effort = "high"`.
- **Prompt caching** on the frozen system block (cite-or-refuse charter, the five tool descriptions). Per-request volatile content (patient list, time window) goes in the user message after the cache breakpoint. Expected cache hit rate: ≥85% within a single morning's session.

**Orchestration:**

- LangGraph state machine: `intent → plan → tool-call(s) → verify → respond | refuse`.
- Max 5 iterations per query. On cap, **summarize-with-refusal** ("I checked X tools and found N events; I did not complete the synthesis — please review the chart").
- Memory: append-only scratchpad per session (lecture pattern). No cross-session state in week 1; the per-shift session is the unit.

**Tool surface (max 5):** see §3.

**Fail-safe:** model timeout → refusal. Tool error → refusal. Iteration cap → flagged-incomplete summary. Never a fluent answer with hidden gaps.

**Trace:**

- `USERS.md` §4 UC-1 step 6 (deterministic acuity rubric) → ranking is *not* an LLM-only decision; rubric is code, LLM provides text.
- `AUDIT.md` §2.2 + §2.3 (data model not tuned for time-window reads) → tool implementations enumerate-by-PK before fetch.

### Layer 4 — Verifier

The verifier is the safety layer, and it is the single most important component in the whole system. After the LLM has written its answer but before the user sees it, a separate piece of plain code goes back to the data and re-checks every claim: does it cite a real row, is the timestamp inside the requested window, does the count match an independent `SELECT COUNT(*)` we ran ourselves. If anything doesn't match, the agent admits the gap — "I found N events but counted M; review source" — instead of shipping a fluent answer with a hidden hole. Cite-or-refuse is a property of code here, not a property of the prompt, which is why the system is safe to put in front of a clinician on day one.

Deterministic, runs after the LLM, before the user sees anything. The verifier is the keystone of the architecture; it is what makes "cite-or-refuse" a property of the system rather than a property of the prompt.

```
   LLM draft response ──▶ ┌──────────────────────┐
                          │ 1. Citation check    │ ── fail ──▶ strip claim
                          └──────────┬───────────┘
                                     ▼
                          ┌──────────────────────┐
                          │ 2. Window integrity  │ ── fail ──▶ strip claim
                          └──────────┬───────────┘
                                     ▼
                          ┌──────────────────────┐
                          │ 3. Count parity      │ ── fail ──▶ degrade
                          └──────────┬───────────┘                │
                                     ▼                            │
                          ┌──────────────────────┐                │
                          │ 4. Med-rec freshness │ ── fail ──▶ degrade
                          └──────────┬───────────┘                │
                                     ▼                            │
                          ┌──────────────────────┐                │
                          │ 5. DRAFT watermark   │ ── fail ──▶ refuse
                          └──────────┬───────────┘                │
                                     ▼                            ▼
                              user (verified)            user (flagged-incomplete
                                                                or refusal)
```

**Checks, in order:**

1. **Citation completeness.** Every clinical claim in the response must reference a citation tuple `(table, row_id, timestamp, author_user_id)` returned by some tool call in this turn. Orphan claim = strip claim, add "I dropped a claim that lacked a record-anchored citation" footer.
2. **Window integrity.** Every cited timestamp ∈ the requested window. Out-of-window = strip claim. (Catches the "ruled out PE 3 years ago" anchor-bias case from `USERS.md`.)
3. **Count parity.** For UC-1 (overnight delta) and UC-2 (discharge blockers): the number of cited rows of type T must equal an **independent** `SELECT COUNT(*)` issued by the verifier itself over the same `(pid, window, table)` predicate. Mismatch = degrade to "I found N events but counted M; review source." This is the structural defense against omission (`USERS.md` §7 hospitalist worst case).
4. **Med-reconciliation freshness.** Any medication name in the response must cite the most recent `prescriptions` event for that med in the last 24h. If a HOLD/DC exists and is not surfaced, add it adjacent. (`USERS.md` §7 resident worst case — re-dosed lisinopril on AKI.)
5. **DRAFT watermark for resident output.** UC-3 outputs are wrapped in a non-copyable frame. Annotations are individually copyable; the frame is not.

**Fail-safe:** verifier finds a problem → degrade. Never strip silently — always tell the user *what* was dropped or unverified.

**Trace:**

- `USERS.md` §7 hospitalist failure mode (silent omission of fall note) → check (3) count parity.
- `USERS.md` §7 resident failure mode (DC'd lisinopril) → check (4) med freshness.
- `USERS.md` §7 resident DRAFT watermark → check (5).
- `AUDIT.md` §2.2 (`pnotes` indexed only on `pid`) → independent `SELECT COUNT(*)` is the structural mitigation; we cannot trust a single bulk query to be complete.

### Layer 5 — Data Plane (FHIR R4 + MySQL read replica + Redis)

This is how the agent actually reads patient data. We use FHIR for the clean, standards-shaped resources (problems, meds, labs, encounters), a read-only MySQL replica for the time-window queries that FHIR is too slow at, and a small Redis cache to kill duplicate fetches inside one pre-rounds session. The hard rule: the agent process itself never opens a database connection. Only the gateway can talk to the replica, and it re-checks ACL on every query, so the agent only ever sees structured tool results. The agent has zero write paths in week 1 — OpenEMR remains the system of record.

> The agent never opens a database connection.

**Three-tier hybrid (per Decision 25 in §9):**

1. **FHIR R4 read-only** at `/apis/{site}/fhir/*` for clean resources (Patient, Encounter, Observation, Condition, MedicationRequest, AllergyIntolerance, DiagnosticReport, DocumentReference, Practitioner). OAuth2 with SMART-on-FHIR launch. Read scopes only — no `*.write`, no `system/*.*` in week 1.
2. **MySQL read replica** for time-window queries (`pnotes` overnight window, `procedure_report.date_report`, vitals trends) where FHIR's per-resource shape is too chatty. **The agent process never opens a DB connection** — only the gateway holds the read replica handle, on a read-only DB user, and re-checks OpenEMR ACL on every replica query before returning rows to the agent. Writes always go through FHIR.
3. **Redis cache** with 15-min TTL on per-`(patient_id, tool_name)` retrievals — kills duplicate fetches inside one pre-rounds session and underpins the prompt-caching savings in §8.

**Why hybrid not FHIR-only:** FHIR per-resource latency on a 13-patient batch overnight delta runs ~600ms+; the replica path lands ~80ms p50. The agent's gateway is the only consumer of the replica, so the legacy `library/*.inc.php` SQL-interpolation surface (`AUDIT.md` §1.4) is never reachable from agent code.

**Why not direct DB from the agent:** any direct DB read from the agent process bypasses the gateway's ACL re-check and audit-write, and risks pulling a legacy include. **Hard constraint: only the gateway speaks to the replica; the agent only sees structured tool results.**

**Caveat from AUDIT:** FHIR endpoints inherit OpenEMR's logging gap (`ApiResponseLoggerListener.php:56` conditional). The gateway compensates by writing its own audit row before any FHIR or replica call, so we have a record even if OpenEMR doesn't.

**Trace:**

- `AUDIT.md` §1.4 → no legacy include path; replica access is gateway-only.
- `AUDIT.md` §2.2 (`pnotes` index gap on `date`) → replica path enables enumerate-by-PK + count parity within the latency budget.
- `AUDIT.md` §5.1 → gateway owns audit.

---

## 3. Tool Surface (Five Tools, Composed)

These five tools are the only way the agent reads patient data. We capped the count at five on purpose: research and our own observation say that once you give an LLM more than about seven tools, it starts confusing near-neighbors and skipping calls. Every use case in `USERS.md` is built by composing a subset of these five — overnight delta uses three, discharge readiness uses three, the resident audit uses three. New use cases earn a tool only if they genuinely cannot be expressed by composition.

Per Ash's lecture: orchestrator confusion threshold is ~7 tools. We constrain to five, one per question shape. Every use case in `USERS.md` is a composition of a subset.

| Tool | What it answers (plain English) | Inputs | Returns (with citations) | Used by |
|---|---|---|---|---|
| `list_overnight_events` | "What changed for this patient between yesterday evening and this morning?" Pulls every clinically-relevant event from the 6 PM–6 AM window: nursing notes, vital sign deltas, new lab/culture results, medication changes, status updates. Each event comes back with a citation back to the exact record so the hospitalist can click through to the source. *This is the engine of the morning's overnight delta — the most-used tool.* | `patient_id`, `window=[start, end]` | RN notes, vitals 2σ deltas, new culture results, med events. Each w/ `(table, row_id, ts, author)` | UC-1, UC-3 |
| `get_labs_window` | "What labs came back during this window?" Pulls lab and microbiology results within a time range, with status (pending / final), value, and reference range. Used both for the overnight question (UC-1) and the discharge-readiness check (UC-2, where we need to know if a culture sensitivity is back yet). | `patient_id`, `window` | `procedure_result` rows incl. status, value, ref-range | UC-1, UC-2, UC-3 |
| `get_med_history` | "What medication events happened recently?" Pulls prescription events — new starts, holds, discontinuations — with timestamps and the user who initiated each. Critical for catching the "continue lisinopril when it was discontinued yesterday" failure mode in the resident audit (UC-3). | `patient_id`, `window` | `prescriptions` events with HOLD/DC/start markers | UC-1, UC-3 |
| `get_consult_status` | "Which consult requests are pending and which have responses?" Pulls open consult orders and any returned consult notes. Used in the discharge-readiness check (UC-2) to identify when a specialty consultant is the blocker. | `patient_id` | open consult requests + responses | UC-2 |
| `get_discharge_blockers` | "What is preventing this patient from going home today?" The only tool that runs a multi-table dependency walk. Returns a structured blocker list across seven categories (pending consults, pending micro, placement, transport, DME, family meeting, follow-up). Used exclusively in UC-2. | `patient_id` | structured blocker list across 7 categories (`USERS.md` §4 UC-2) | UC-2 |

> **Are we confident in these five?** Defensibly yes for the three target use cases (UC-1, UC-2, UC-3) — every step in each use case can be expressed as a composition of these five tools, and we kept the count at five specifically to stay below the orchestrator confusion threshold (~7 tools per Ash's lecture). What we *haven't* validated yet is whether the tools' input/output shapes survive contact with real data — for example, `list_overnight_events` aggregates five different source types and returns them as a single ranked list, which may be too coarse if the LLM struggles to disambiguate event categories during synthesis. Implementation will tell us. **The set is the right size; the boundaries between specific tools may shift after we run the first real pre-rounds session against seeded data.** Adding tools is cheap; removing them is expensive (it breaks LLM behavior trained against the previous set), so we'll iterate with restraint.

**Request lifecycle for a single tool call:**

```
   Tool invoked
        │
        ▼
   Enumerate by PK  ──▶ SELECT id ... (record count)
        │
        ▼
   Verify count     ──▶ refuse on partial
        │
        ▼
   Fetch rows       ──▶ SELECT body WHERE id IN (...)
        │
        ▼
   Return with citations  (table, row_id, ts, author_user_id)
```

**Implementation contract for every tool (enforced by the agent harness):**

- **Enumerate before fetch.** Issue `SELECT id ...` first; record the count. Only then fetch row bodies. Refuse on partial.
- **Citations are first-class.** Every returned event has `(table, row_id, timestamp, author_user_id)`. No narrative summaries from the tool.
- **No cross-patient queries.** One `patient_id` per call. Forces the verifier's count-parity check to be tractable.

**Trace:** `AUDIT.md` §2.2 (`pnotes` index gap) → enumerate-before-fetch is *required*, not best-practice.

---

## 4. Use Case → Architecture Trace

This section shows how the three real workflows from `USERS.md` actually flow through the five layers — which tools they call, which ACL gate applies, what the verifier specifically checks for that case. The second table is the failure-mode coverage matrix: for every way the system could ship a wrong clinical answer, it names the layer that catches it.

> If we cannot point to a layer, the failure mode is not covered.

Every use case in `USERS.md` decomposes into the five layers as follows.

| Use case | Client | Gateway | Agent | Verifier | Data plane |
|---|---|---|---|---|---|
| **UC-1 Overnight Delta** | Sidebar card grid | ACL: attending-of-record | `list_overnight_events` × 13, then rank by acuity rubric | Count parity per pnotes/labs | FHIR `Observation`, `DocumentReference`, `MedicationRequest` |
| **UC-2 Discharge Readiness** | Printable table | ACL: attending-of-record | `get_consult_status`, `get_discharge_blockers`, `get_labs_window` | "No blocker" requires enumerated category list | FHIR `ServiceRequest`, `Procedure`, `CarePlan` |
| **UC-3 Resident Audit** | Two-col diff w/ DRAFT watermark | ACL: resident + attending-of-record link | `list_overnight_events`, `get_med_history`, `get_labs_window` | Med-reconciliation freshness; DRAFT frame | FHIR `MedicationRequest`, `MedicationStatement`, `Observation` |

**Failure-mode coverage matrix (`USERS.md` §7):**

| Failure mode | Layer that catches it |
|---|---|
| Silent omission of RN fall note | Layer 4 verifier — count parity vs. independent `SELECT COUNT(*)` |
| "Continue lisinopril" after DC | Layer 4 verifier — med-reconciliation freshness check |
| Resident pastes agent output as note | Layer 4 verifier — non-copyable DRAFT frame |
| Wrong-patient data (parallel charts) | Layer 2 gateway — every tool call binds to one `patient_id` from the request |
| Stale problem list as current | Layer 5 data plane — FHIR `Condition.clinicalStatus` filter; surface inactive separately |
| Out-of-window "ruled out PE 3y ago" | Layer 4 verifier — window integrity check |

---

## 5. Authorization Model (HIPAA + ACL)

The agent does not invent its own permission system. It inherits whatever access the user already has in OpenEMR, and the gateway re-checks that access on every single tool call as defense in depth. On top of that we layer treatment-relationship gates (you have to actually be on the patient's care team), sensitive-segment opt-ins (mental health, SUD, repro, HIV are omitted by default and the user is told), and a supervision link for residents. The bottom row of the table is the HIPAA Security Rule walk-through — for each rule, here is the literal code or config that satisfies it.

**Inheritance from OpenEMR phpGACL.** No parallel permission store. The gateway re-checks at every call (defense in depth).

**Layered checks:**

1. **Role × resource matrix** — agent inherits user's OpenEMR ACL. (`USERS.md` §8 #1.)
2. **Treatment-relationship gate** — for routine queries, attending-of-record / consulting team / attached provider / assigned RN. (`USERS.md` §8 #2.)
3. **Break-glass mode** — explicit user action with reason code, audit escalated, privacy officer notified. **Out of scope for week 1** (hospitalist is the low-break-glass case).
4. **Sensitive-segment opt-in** — mental health, SUD (42 CFR Part 2), repro health, HIV require explicit per-encounter affirmation. **Default: agent omits** and tells the user.
5. **Supervision link for residents** — resident queries require linked attending-of-record. DRAFT watermark.
6. **Audit every retrieval, not just every answer** — what tool, what record, what surfaced, what suppressed.
7. **Refusal as primitive** — no source = refuse. Refusal is a successful response.

**HIPAA pass (`PROJECT.md` Q4):**

| # | Rule | Citation | Implementation |
|---|---|---|---|
| 1 | BAA with every PHI vendor | 45 CFR §164.504(e) | Anthropic API zero-retention; Railway BAA tier; week-1 demo uses LangSmith on synthetic data only (no PHI in traces); production target is self-hosted Langfuse inside the same Railway project so the BAA boundary is preserved without a third-party observability vendor |
| 2 | Minimum necessary | §164.502(b) | Per-use-case scope cap |
| 3 | Access controls | §164.312(a) | OpenEMR ACL inherit + gateway recheck + auto-logoff |
| 4 | Audit controls | §164.312(b) + §164.530(j) | Append-only Postgres `audit_log`; ≥6yr retention; immutable |
| 5 | Integrity + transmission | §164.312(c),(e) | TLS 1.3 in transit; AES-256 at rest; cite-or-refuse for integrity |
| 6 | Breach notification | §164.400-414 | Documented IR plan; 60-day HHS path |

**Common audit failures avoided** (per `PROJECT.md`): no PHI in plain logs; LLM with BAA; no tokens in URLs; default `admin/pass` rotated pre-deploy (`AUDIT.md` §1.2); audit log immutable; ACL not over-broad; LLM transcripts ≤30d retention.

---

## 6. Doctor Authentication Flow

Maya opens her browser at 6:30 AM, types her hospital password into OpenEMR, and lands in her service-list view. She clicks the AgentForge sidebar inside an open encounter and within a second sees a ranked overnight delta. What she does not see is a chain of identity hand-offs running underneath: her OpenEMR session is signed into a short-lived token by the OpenEMR module, that token is verified by the agent gateway, the gateway calls back to OpenEMR's FHIR `Practitioner` endpoint to confirm she is real and active, an audit row is written before any tool runs, and only then does the agent get to think. Each hand-off is designed to fail safe — a missing signature, an expired session, a sensitive-segment patient, a resident with no attending link, all return a structured refusal instead of leaking access.

```
 Browser           OpenEMR            OpenEMR-Module        Agent-Gateway        FHIR
   │                  │                      │                    │                │
   │ 1. POST /login   │                      │                    │                │
   │   (user/pass)    │                      │                    │                │
   │─────────────────▶│                      │                    │                │
   │                  │ bcrypt/Argon2 verify │                    │                │
   │                  │ session cookie       │                    │                │
   │   Set-Cookie     │  HttpOnly+Secure     │                    │                │
   │◀─────────────────│  SameSite=Lax        │                    │                │
   │                  │                      │                    │                │
   │ 2. GET /encounter│                      │                    │                │
   │─────────────────▶│                      │                    │                │
   │                  │ render encounter +   │                    │                │
   │                  │ mount module iframe  │                    │                │
   │                  │─────────────────────▶│                    │                │
   │                  │                      │ read $_SESSION     │                │
   │                  │                      │ sign 60s JWT (HMAC)│                │
   │                  │                      │ {user_id,facility, │                │
   │                  │                      │  encounter,exp,jti}│                │
   │   iframe + JWT   │                      │                    │                │
   │◀─────────────────────────────────────────                    │                │
   │                  │                      │                    │                │
   │ 3. POST /agent/run                      │                    │                │
   │    X-OpenEMR-Identity: <jwt>            │                    │                │
   │    X-Agent-Token: <demo-secret>         │                    │                │
   │─────────────────────────────────────────────────────────────▶│                │
   │                  │                      │                    │ verify HMAC    │
   │                  │                      │                    │ check exp+jti  │
   │                  │                      │                    │ (replay cache) │
   │                  │                      │                    │                │
   │                  │                      │                    │ 4. GET         │
   │                  │                      │                    │ /Practitioner/ │
   │                  │                      │                    │ {user_id}      │
   │                  │                      │                    │───────────────▶│
   │                  │                      │                    │ active=true    │
   │                  │                      │                    │◀───────────────│
   │                  │                      │                    │                │
   │                  │                      │                    │ ACL probe:     │
   │                  │                      │                    │ /Patient?_count│
   │                  │                      │                    │ (round-trip    │
   │                  │                      │                    │  AclMain)      │
   │                  │                      │                    │───────────────▶│
   │                  │                      │                    │ allowed/deny   │
   │                  │                      │                    │◀───────────────│
   │                  │                      │                    │                │
   │                  │                      │                    │ write audit row│
   │                  │                      │                    │ BEFORE compute │
   │                  │                      │                    │                │
   │                  │                      │                    │ sensitive seg? │
   │                  │                      │                    │   → 403 opt-in │
   │                  │                      │                    │ resident?      │
   │                  │                      │                    │   → require    │
   │                  │                      │                    │     attending  │
   │                  │                      │                    │                │
   │   200 OK + result                                            │                │
   │◀─────────────────────────────────────────────────────────────│                │
```

1. **Step 1 — User authenticates to OpenEMR.** Doctor logs into OpenEMR via the existing OIDC/OAuth2 stack. OpenEMR validates against bcrypt/Argon2 (`AUDIT.md` §1.1). Session cookie set with HttpOnly, Secure, SameSite=Lax. Auto-logoff after 15 min idle per HIPAA §164.312(a)(2)(iii).
2. **Step 2 — Widget loads inside OpenEMR.** The custom module's iframe/sidebar renders inside the authenticated OpenEMR encounter view. The browser already has the OpenEMR session cookie scoped to the OpenEMR origin.
3. **Step 3 — Widget signs an identity assertion.** Module backend (PHP, runs inside OpenEMR's PHP-FPM) reads the current `$_SESSION` to get `users.id`, `users.username`, and active facility/encounter context. It then creates a short-lived (60s) JWT signed with a per-deploy HMAC secret, payload: `{user_id, username, facility_id, encounter_id, exp, jti}`. JWT is included in the iframe URL or a `postMessage` to the widget's frontend.
4. **Step 4 — Widget calls agent service.** Frontend POSTs to `https://agent-...railway.app/agent/run` with header `X-OpenEMR-Identity: <jwt>` plus the existing `X-Agent-Token` shared secret (week-1 demo). In production: SMART-on-FHIR launch passes a real OAuth2 access token instead of the demo token.
5. **Step 5 — Gateway verifies identity.** Agent gateway (`Layer 2` in §2): (a) verifies the demo `X-Agent-Token`, (b) verifies the JWT signature with the shared HMAC, (c) checks `exp` and replay protection on `jti`, (d) extracts `user_id`, (e) calls back to OpenEMR FHIR `/apis/{site}/fhir/Practitioner/{user_id}` with a service-account OAuth2 token to confirm the practitioner exists and is active.
6. **Step 6 — ACL probe.** Gateway issues an explicit FHIR query that *should* succeed only if the user has clinical access (e.g., `/Patient?_count=0` with the user's session) — round-trips through OpenEMR's `AclMain` so we get the canonical answer, not a cached one. (`AUDIT.md` §1.3 — never trust a cached `true`.)
7. **Step 7 — Audit row written.** Before any tool call, gateway writes `{request_id, user_id, username, facility_id, encounter_id, jwt_jti, ts, intent}` to the agent's append-only audit table. Failure to write = 503, request never proceeds.
8. **Step 8 — Step-up for sensitive segments.** If the request touches a sensitive-segment patient (mental health, SUD per 42 CFR Part 2, repro, HIV — `USERS.md` §8 #4), the gateway returns 403 with `{"reason": "sensitive_segment_optin_required", "categories": [...]}`. The widget renders a per-encounter consent modal; on affirmative, widget reissues the request with `X-Sensitive-Optin: <category-list>` header. Gateway records this in the audit row.
9. **Step 9 — Resident supervision link.** If the user is a resident (role check via FHIR `Practitioner.qualification`), gateway also requires an `X-Attending-Of-Record: <attending-user-id>` header signed in the JWT. No link, no agent. Documented in `USERS.md` §6.
10. **Step 10 — Auto-logoff propagation.** When OpenEMR's session expires, the next agent request fails identity verification (Step 5e — FHIR call returns 401). Gateway returns 401 with `{"reason": "session_invalid"}` and the widget redirects to OpenEMR's login.

**Trace:**

- `AUDIT.md` §1.1 (password hashing) → Step 1 inherits OpenEMR's bcrypt/Argon2.
- `AUDIT.md` §1.3 (silent fail-closed ACL) → Step 6 round-trips for canonical answer instead of caching.
- `AUDIT.md` §5.1 (audit gap) → Step 7 writes to agent-owned audit table before any compute.
- `USERS.md` §6 (resident supervision overlay) → Step 9 enforced at gateway, not at tool layer.
- `USERS.md` §8 #4 (sensitive-segment opt-in) → Step 8 default-omit + structured refusal.

---

## 7. Deployment Topology (As Deployed on Railway)

This is what is actually running, right now, in production. There are three Railway services: OpenEMR itself (the EHR we're co-piloting), MariaDB (its database), and our agent (FastAPI + LangGraph + Anthropic SDK). The clinician's browser only ever talks to OpenEMR over HTTPS; the agent is reached server-side through the OpenEMR widget so the session cookie can be forwarded safely. One operational gotcha worth knowing: OpenEMR's auto-installer is not safe to run in parallel, so its service is pinned to a single instance — multiple replicas race on CREATE TABLE and deadlock the install.

Project: **`agentforge-agent`** (Railway, `jiahknee5@gmail.com`).

```
┌─────────────────────┐  HTTPS  ┌─────────────────────────┐
│ Browser (clinician) │────────▶│ openemr (web edge)      │
└─────────────────────┘         │ openemr/openemr:latest  │
                                │ Apache + PHP 8.4        │
                                │ TARGET_PORT=80          │
                                └────────────┬────────────┘
                                             │ private
                                  ┌──────────▼────────────┐
                                  │ mysql (MariaDB 10.11) │
                                  │ MARIADB_ROOT_HOST=%   │
                                  │ Volume: /var/lib/mysql│
                                  └───────────────────────┘

┌─────────────────────┐  HTTPS  ┌─────────────────────────┐
│ OpenEMR widget      │────────▶│ agent (FastAPI)         │
│ (server-side proxy) │         │ Python 3.12             │
└─────────────────────┘         │ X-Agent-Token gate      │
                                │ LangSmith (demo) →      │
                                │   Langfuse (prod target)│
                                │ Anthropic SDK direct    │
                                └────────────┬────────────┘
                                             │ private FHIR
                                             │ /apis/{site}/fhir/*
                                             ▼
                                      (openemr service)
```

**Currently live (week-1 demo posture):**

- OpenEMR: `https://openemr-production-8348.up.railway.app` — admin/pass, **rotate before any clinical use**.
- Agent: `https://agent-production-66ad.up.railway.app` — `X-Agent-Token` header.
- MariaDB: `mysql.railway.internal:3306` — private only.
- LangSmith project: `agentforge-hospitalist` (week-1 demo, no PHI in traces — synthetic data only; configured via `LANGSMITH_API_KEY` + `LANGSMITH_PROJECT` env vars in `Week 1/agent/app/observability.py`). LLM spans auto-flow for every `prerounds_briefing` workflow via `wrap_anthropic` and `@traceable`. System traces go to OpenTelemetry; unhandled exceptions to Sentry. **Production target:** swap LangSmith for self-hosted Langfuse inside this same Railway project before any real-PHI deployment, so observability stays inside the BAA boundary; the `@traceable` and SDK-wrap patterns translate near-1:1 to the Langfuse SDK.

**Scaling note that bit us in week-1 deploy and is now constrained:** OpenEMR service is pinned to **1 instance** in `us-east4-eqdc4a`. The OpenEMR auto-installer is not idempotent under concurrent launches; multiple replicas race on `CREATE TABLE` and deadlock. Documented in deploy README.

---

## 8. Cost Analysis

> **Caveat — these numbers are first-pass estimates pending real-world measurement.** Token counts are derived from prompt sizes and observed tool-result payloads on synthetic data; cache hit rate (~85%) is assumed not measured; the Sonnet/Opus split (85/15) is intuition not data. Expect the per-morning cost to land within ±50% of the figure below once we run a real pre-rounds session against seeded data, and to settle to a stable number only after a few weeks of actual hospitalist use. Use these numbers for *order-of-magnitude defense* (is this $0.40 or $40 per morning? — clearly the former) but do not put them in a budget proposal until we've measured.

A full pre-rounds session for one hospitalist covering 13 patients across all three use cases costs roughly 25 to 40 cents in API spend. Scaled to a 350-bed academic hospital with ~25 hospitalists running this every weekday, that's about $3,500 a year. The clinical-time payback is roughly twelve minutes recovered per hospitalist per morning, which works out to about $2.5M of attending time per year — three orders of magnitude over the cost. The latency budget is tight (under 3 seconds median, under 6 seconds at p95) and the verifier is the hard part: it runs after the LLM stream so the user sees a brief "checking sources..." state. The infra cost for the week-1 demo posture is under $90/month all-in.

Per `PROJECT.md`: AI cost analysis must live in this document.

### 8.1 Per-session cost model (one hospitalist, one morning)

The walk-through below builds the per-morning number from first principles. Each step names its inputs, the operation, and the output so a reviewer can audit any step in isolation without trusting the next.

#### Step 1 — What's being billed

Anthropic bills three quantities per API call, and they're priced very differently:

- **Input tokens** — the prompt the model reads (system prompt + user message + tool results). At Sonnet 4.6: **$3.00 per million tokens**.
- **Output tokens** — the tokens the model generates. At Sonnet 4.6: **$15.00 per million tokens** — *5× the input rate.*
- **Cache reads** — input tokens that hit a previously-cached prefix (e.g., the unchanging system prompt across calls in the same session). Billed at **~10% of fresh input** ($0.30/M at Sonnet 4.6).

The cost story is dominated by (a) keeping output tokens small — tools return citations not narrative, and (b) keeping the big invariant prefix in cache so we pay 10% on it instead of 100% across the morning's repeat calls.

#### Step 2 — Per-call token shape (one UC-1 query)

A single UC-1 "what changed overnight for these 13 patients" call decomposes as:

| Component | Tokens | Cache state | Why this size |
|---|---:|---|---|
| System prompt (cite-or-refuse charter + 5 tool descriptions) | ~3,500 | **Cached** (invariant within a session) | Frozen at session start; identical across every call this morning |
| Patient context + tool results (13 patients × ~600 tokens overnight events each) | ~7,800 | **Fresh** | Different every call; cannot be cached |
| Verifier prompts + count-parity scratch | ~2,000 | **Fresh** | Per-call counts and citations |
| **Per-call input total** | **~13,500** | (3,500 cached + 10,000 fresh) | |
| **Per-call output** | **~1,200** | n/a | Ranked summary with citations, kept terse |

#### Step 3 — Calls per morning

A pre-rounds session for one hospitalist (covering 13 patients, with 2 residents under them) breaks down as:

| Call type | How many | Why this many |
|---|---:|---|
| UC-1 Overnight Delta | **1** | One batch covers all 13 patients |
| UC-2 Discharge Readiness | **1** | One batch over the discharge-candidate subset |
| UC-3 Resident Audit | **2** | Two residents → two draft-vs-chart audits |
| Verifier secondary calls | **~10** | Count-parity re-queries + edge cases across the above |
| **Total calls** | **~14** | |

Aggregating Step 2's per-call shape across these 14 calls:

| Use case | Calls | Input tokens | Output tokens |
|---|---:|---:|---:|
| UC-1 Overnight Delta | 1 | 13,500 | 1,200 |
| UC-2 Discharge Readiness | 1 | 9,000 | 800 |
| UC-3 Resident Audit | 2 | 4,000 × 2 = 8,000 | 600 × 2 = 1,200 |
| Verifier secondary calls | 10 | 4,000 (aggregate) | 200 (aggregate) |
| **Morning total** | **~14** | **~34,500** | **~3,400** |

#### Step 4 — Subtotal at Sonnet 4.6 rates (with cache)

Apply the cache-hit discount. The ~3,500-token system block is in cache for ~85% of the morning's 14 calls (the first call seeds the cache; the remaining 13 hit it). So:

- **Cache-hit portion of input:** 3,500 tokens × 14 calls × 0.85 hit rate = ~41,650 tokens billed at the cache rate.
- **Fresh portion of input:** the remaining input — fresh per-patient bodies (~31,000 tokens across the morning) plus the cache-miss occurrences of the system block (~7,350 tokens) = ~38,350 fresh tokens.

Now multiply each bucket by its rate:

| Bucket | Tokens | Rate ($/M) | Subtotal |
|---|---:|---:|---:|
| Cache reads | 41,650 | $0.30 | 41,650 / 1,000,000 × $0.30 = **$0.0125** |
| Fresh input | 38,350 | $3.00 | 38,350 / 1,000,000 × $3.00 = **$0.115** |
| Output | 3,400 | $15.00 | 3,400 / 1,000,000 × $15.00 = **$0.051** |
| **Sonnet-only subtotal** | | | **~$0.18 / morning / hospitalist** |

(The simpler approximation that ignores per-call cache mechanics — "effective billed input ≈ 35,500 tokens × $3/M + 3,400 tokens × $15/M" — lands at ~$0.16, within rounding of the explicit calculation above. Both are within the ±50% caveat band at the top of §8.)

#### Step 5 — Opus 4.7 escalation (~15% of calls)

About 15% of the morning's calls — the genuinely hard synthesis cases (a 40+ event UC-1 patient, a UC-3 cross-text reasoning audit) — escalate to Opus 4.7 at **$15/M input, $75/M output**, roughly 5× the Sonnet rate.

- Escalated calls per morning: ~14 × 0.15 ≈ **2 calls**.
- Marginal token volume on those 2 calls: ~5,000 fresh input + ~700 output (the higher-effort answers run a bit longer).
- Marginal Opus cost: 5,000 / 1M × $15 + 700 / 1M × $75 = $0.075 + $0.053 ≈ **+$0.13 / morning**.

(Earlier drafts cited "+$0.10/morning" using $5/M + $25/M Opus rates from a prior list-price snapshot; the current $15/M + $75/M public rate is used here. Both land within the caveat band.)

#### Step 6 — Final per-morning per-hospitalist number

Sum Steps 4 and 5:

- **Sonnet baseline:** ~$0.18
- **Opus escalation:** ~+$0.13
- **Per-morning per-hospitalist: ~$0.25–$0.40** (the band reflects the caveat at §8 and the cache-rate / escalation-fraction sensitivities).

Scaling out:

- 350-bed academic hospital × 25 hospitalists × 250 weekdays × ~$0.40 ≈ **$2,500–$3,500 / year in API spend**.
- Recovered attending time at the same scale: 12 minutes saved/morning × 25 hospitalists × 250 weekdays × $200/hr blended attending rate ≈ **~$2.5M / year**.
- **Three-orders-of-magnitude payback** — and that's the ratio that's robust to the ±50% caveat at the top of §8, which is why we lead the deck with the ratio not the absolute number.

### 8.2 Latency budget

| Step | Budget (p50) | Budget (p95) |
|---|---:|---:|
| Gateway auth + audit write | 80ms | 200ms |
| FHIR queries (parallel × 13 patients) | 600ms | 1,400ms |
| LLM (Sonnet 4.6 w/ adaptive thinking, ~1.2K out streaming) | 1,200ms | 2,800ms |
| Verifier | 300ms | 800ms |
| Network/edge | 100ms | 300ms |
| **Total** | **~2,280ms** | **~5,500ms** |

**Target: <3s p50, <6s p95.** Streaming the LLM response masks much of the LLM latency for perceived UX; the hard constraint is the verifier (which is post-stream and visible as a brief "checking sources..." state). The 300ms / 800ms numbers are the *per-call* verifier budget; on a busy 13-patient UC-1 batch the verifier's secondary count-parity queries can compound to **800–1,500ms p95** end-to-end (the figure cited on the defense deck slide 6 — "+800–1,500 ms latency per response").

### 8.3 Infrastructure cost

| Component | Tier | Monthly |
|---|---|---:|
| Railway (3 services + volumes) | Hobby/Pro | ~$25 |
| Anthropic API | Pay-as-you-go | $10–50 |
| LangSmith Hobby (week-1 demo) / Langfuse self-hosted (production target) | LangSmith Hobby tier today; Langfuse on the same Railway project at production | ~$0 (week 1) / ~$10 (prod target) |
| **Total** | | **~$45–85/mo for week-1 demo posture** |

Production HIPAA tier on Railway adds ~$200/mo. The week-1 demo uses LangSmith Hobby (~$0) on synthetic data only; the production migration moves observability to self-hosted Langfuse inside the same Railway project so there is no separate vendor BAA to negotiate. Anthropic API zero-retention is a configuration toggle, not a price increase.

---

## 9. Trade-offs and Decisions

Every architecture has alternatives we considered and rejected, and this section names them honestly. For each of the eight major decisions on the deck — integration path, orchestration framework, model choice, verification approach, data access, deployment platform, observability, and eval methodology — the table lists what we picked, what we passed on, why the pick is defensible, and the falsifier (the specific signal that would make us change our mind). The falsifier matters: it shows the decision is held with a known threshold, not a religion. The cross-cutting choices at the bottom (tool count, memory model, week-1 auth, caching strategy) are the smaller calls that don't get their own slide but still need to be defended.

The deck (`Week 1/deck/21-28-*.html`) defends eight architectural decisions. They are summarised here in the order they appear, with the rejected option and the falsifier (the signal that would change the decision). For full comparison tables and sources see the deck slides.

| # | Decision | Chose | Rejected | Why defendable / falsifier |
|---|---|---|---|---|
| 21 | OpenEMR integration path | Hybrid: thin custom module (UI shell) + external Python agent service over OAuth2 | Full in-process custom module (D); sidebar widget only (A); external FHIR consumer with no module (B) | `AUDIT.md` §1.4 forbids legacy SQL-interpolation includes; module is thin, agent iterates independently. **Falsifier:** hospital network policy bans external services → fall back to in-process Composer sidecar |
| 22 | Agent orchestration framework | LangGraph (explicit graph + typed state) + Anthropic SDK direct for tool-calls inside nodes | CrewAI (emergent), AutoGen (research-grade), pure custom | Clinical determinism needs reproducible execution; typed edges give time-travel debugging; graph spec is YAML-portable. **Falsifier:** LangGraph adds >200ms/node → port spec to custom executor |
| 23 | LLM provider & model strategy | Anthropic Claude Sonnet 4.6 default (~85% of calls), Opus 4.7 escalation (~15%) on multi-source synthesis and resident audit; BAA + zero-retention; prompt-cached | OpenAI (BAA fragile by churn), Gemini, self-hosted Llama 3.3 70B, any consumer endpoint without BAA | Refusal quality + citation discipline measurable on our eval; switchable behind a thin LLM client. **Falsifier:** hospital BAA is OpenAI-only → port to GPT-4o, eval set tells us if quality drops |
| 24 | Verification approach | Two-layer: deterministic citation parser + rule-based domain constraint engine (RxNorm/openFDA, dose, allergy, stale-data) | LLM-as-judge at runtime; "trust the LLM"; prompt-engineering-only | Post-LLM determinism is auditable; ~800ms/$0 vs. verifier-LLM's 3000ms/$0.02. LLM-judge is reserved for the eval tier (Decision 28), not runtime. **Falsifier:** citation suppression causes >5% false-refusal on eval → add a verifier-LLM tier with stricter latency budget |
| 25 | Data access layer | 3-tier hybrid: FHIR R4 (clean resources) + MySQL read replica via gateway (time-window queries) + Redis 15-min TTL cache | FHIR-only (slow for batch overnight delta); direct MySQL from agent (`AUDIT.md` §1.4 + bypasses ACL); vector DB over notes (deferred) | Replica is **gateway-only**, on a read-only DB user, with ACL re-check on every query — `AUDIT.md` §1.4 hard constraint preserved. Hits 80ms p50 vs. 600ms+ FHIR-only. **Falsifier:** institution runs OpenEMR on managed Aurora with no replica → fall back to FHIR-only with per-patient pre-fetch + larger Redis |
| 26 | Deployment platform | Railway for week 1 (BAA on Pro+, Docker-portable); AWS ECS planned for the 10K-user production tier | Vercel (frontend-biased, bad for stateful PHP); Fly.io (strong runner-up); AWS now (2–3 days vs. 10 min) | Speed wins week 1; Docker keeps us portable so AWS migration is a Terraform exercise. **Falsifier:** Railway's BAA isn't available at submission time → switch to Fly.io with no code changes |
| 27 | Observability stack | Three tools, three audiences: OpenTelemetry (system traces), LLM-span tool for LLM debugging, custom append-only Postgres audit table (HIPAA), Sentry (unhandled exceptions). **Week-1 demo uses LangSmith** for the LLM-span layer (synthetic data only, no PHI in traces); **production target is self-hosted Langfuse** inside the same Railway project so PHI stays inside the BAA boundary without a third-party observability vendor | LangSmith as the *production* answer (BAA-fragile as a third-party vendor for PHI traces); Braintrust (eval-focused, not HIPAA audit); OpenTelemetry alone (not LLM-aware); custom audit table alone (not observability) | SRE, LLM debugging, and HIPAA audit have different schemas; one tool can't serve all three. LangSmith is fine for the week-1 demo because no real PHI is in scope; for production, self-hosting Langfuse keeps PHI inside the Railway BAA boundary. **Falsifier:** Langfuse self-host adds >5% latency → drop it to async-only ingestion, or stay on LangSmith with a vendor BAA if one is offered |
| 28 | Eval methodology | Three-tier graded: rule-based on every PR (citation, auth, schema, refusal), LLM-judge on every PR with Opus 4.7 (omission, hallucination, tone), human spot-check on every release (5 hardest cases, clinician-rubric); Synthea synthetic patients | Rule-based only (misses semantic omission); LLM-judge only (LLM-on-LLM bias on auth/citation); production-only (HIPAA exposure, can't ship); human-only (doesn't scale) | Rules catch deterministic, LLM catches semantic, human catches "I-can't-articulate-why-it's-wrong"; ~$1/run × 10 PRs/day = $10/day, well under $450 Ramp budget. **Falsifier:** LLM-judge variance >10% across runs → switch to a fine-tuned smaller model with deterministic temperature 0 |

**Cross-cutting choices (not decision slides but called out for the defense):**

| Choice | Chose | Rejected | Why |
|---|---|---|---|
| Tool count | 5 max | 10+ rich tools | Ash's lecture: orchestrator confusion threshold ~7. Composition over expansion |
| Memory | Per-session scratchpad | Cross-session | Hospitalist shift is bounded; cross-session privacy + retention complexity not earned in week 1 |
| Auth (week 1) | Shared `X-Agent-Token` | SMART-on-FHIR launch | SMART setup blocks the demo; documented as week-2 work |
| Caching | System block + tool descriptions | Per-patient response cache | Per-patient data is volatile; system is stable. Cache where the bytes are stable |

### 9.1 Decision rationale (defense-deck depth)

The table above is the scannable index. The subsections below are the prose version of each decision in the user's own voice — what we chose, what we rejected, why the choice survives a hostile reviewer, and the falsifier that would actually move us off it. Each one is the answer to the corresponding 5-min defense Q&A.

#### Integration shape — hybrid module + external service

**What we chose.** A thin OpenEMR custom module under `/interface/modules/custom_modules/oe-module-agentforge/` that does nothing but forward identity and render a structured response, plus an external Python FastAPI service that runs the agent in its own process.

**What we rejected.** A pure custom module that runs the LLM in-process inside OpenEMR's PHP-FPM (couples deploy cadence, drags in legacy `library/*.inc.php` SQL-interpolation files per `AUDIT.md` §1.4). Also rejected: an agent-only standalone tool with no module at all (loses the "lives in the workflow" thesis — clinicians won't context-switch to a separate URL during pre-rounds).

**Why this is defendable.** The deploy cadences are different: the agent will iterate weekly on prompts, tools, and verifier rules; OpenEMR will not. Coupling them slows both. The legacy include surface is a real, documented risk — `AUDIT.md` §1.4 cites specific files where SQL variables are still string-interpolated, and an in-process agent can pull those files transitively. By keeping the module thin and the agent external, the worst legacy code path is never reachable from agent code. A reviewer pushing back on "why two services?" is really asking "why not pure-PHP?" and the answer is: the PHP ecosystem for tool-calling LLMs is thin, the deploy cadence mismatch hurts both sides, and the audit blast radius from the legacy include surface is something we can avoid for free by separating the processes.

**What would change our mind.** OpenEMR shipping first-class FastAPI tool-calling primitives (consolidate into one process). Or a hospital network policy that bans external services entirely — in that case we fall back to an in-process Composer sidecar inside OpenEMR's container, accepting the slower iteration cycle.

#### Orchestration — LangGraph + Anthropic SDK direct

**What we chose.** LangGraph for the explicit graph + typed state machine (`intent → plan → tool-call(s) → verify → respond | refuse`), with Anthropic SDK called directly inside each node. No LangChain wrapper around the model call.

**What we rejected.** LangChain end-to-end (its Anthropic adapter lags behind the SDK on adaptive thinking, prompt caching, and tool schema evolution). CrewAI (emergent multi-agent — the failure modes are not auditable, which is exactly what clinical AI cannot afford). AutoGen (research-grade, not production-shaped). Pure custom executor (we would re-build LangGraph badly).

**Why this is defendable.** Clinical determinism needs reproducible execution: every state transition has to be inspectable in the trace, every tool call has to be re-runnable from a logged input, every refusal has to map to a node. LangGraph's typed edges give us time-travel debugging and a graph spec that's portable as YAML — we can swap executors without rewriting the agent. The reason we go around LangChain on the model call itself is that Anthropic ships features (adaptive thinking, prompt caching breakpoints, the structured `thinking` block) that LangChain's adapter consistently lags on by weeks; for a one-week build, that lag is the difference between a working prompt-cache hit rate and a 4× cost overrun. A reviewer asking "why not LangChain?" is asking why we're paying the integration cost twice; the answer is that we're not — LangGraph alone gives us the orchestration value, and the SDK gives us the model features the day they ship.

**What would change our mind.** LangGraph itself becoming a latency drag (>200ms/node overhead would kill our 2.3s p50 budget); in that case we port the YAML spec to a custom executor with the same node interface. Or LangChain catching up such that the SDK passthrough no longer matters.

#### Verifier — deterministic post-LLM, not self-critique

**What we chose.** A separate, deterministic Python pipeline that runs after the LLM has produced its draft response, performing five checks (citation completeness, window integrity, count parity, med-rec freshness, DRAFT watermark) before any output reaches the user. Cite-or-refuse is a property of code, not the prompt.

**What we rejected.** LLM-as-judge at runtime ("ask another LLM whether this answer is good"). Self-critique by the same LLM ("now check your work"). Prompt-engineering-only ("be careful and cite everything").

**Why this is defendable.** The LLM is the source of the failure modes we are trying to catch — hallucinated citations, omitted events, out-of-window claims. Asking an LLM to police those failures shifts the failure mode from one LLM to two correlated LLMs, and it makes the safety property non-deterministic and non-auditable. A hospital security review will ask "show me the code that guarantees this property"; with LLM-as-judge the answer is a prompt and a probability distribution, and that does not pass review. With deterministic verification the answer is a Python module with unit tests and a CI gate. Empirically, prompt engineering caps at roughly 92–95% citation faithfulness on clinical synthesis (Singhal et al. *Nature* 2023; npj Digital Medicine 2024); the gap from there to 100% is the entire reason the system is deployable. The +800–1,500ms latency cost is the price we pay for that property and it is worth it for a use case where the user is going to look at the output for 90 seconds anyway.

**What would change our mind.** A published model that hits ≥99.5% citation faithfulness on our eval set without external verification, sustained across a model generation. Or our own eval showing the verifier suppresses >5% of correct claims (false-refusal) — at that point we'd add a verifier-LLM tier as a second-chance review, but only inside a tighter latency budget.

#### Tool count — 5 max, not 10+

**What we chose.** Exactly five tools (`list_overnight_events`, `get_labs_window`, `get_med_history`, `get_consult_status`, `get_discharge_blockers`), one per question shape, all use cases built by composition.

**What we rejected.** A richer surface of 10+ tools (e.g., separate tools for each lab type, separate tools for each note type, separate tools for each consult status). Or a single "ask anything" tool with a free-text query.

**Why this is defendable.** The orchestrator-confusion threshold for tool-calling LLMs is empirically around 7 — Ash's lecture and our own observation match: past that count, models start confusing near-neighbors and skipping calls. Five gives us headroom and forces composition: every use case in `USERS.md` is a subset of these five, and a new use case earns a new tool only if it genuinely cannot be expressed by composition. The cost of expansion is also real on the verifier side — every new tool is a new citation shape the verifier has to know about, and a new failure surface for prompt injection. Composition keeps the verifier's contract small. A reviewer pushing back on "why so few?" is usually imagining a richer demo; the answer is that the demo is built and the three use cases all work with five, so the burden of proof is on the sixth tool to justify itself.

**What would change our mind.** A use case that genuinely cannot be expressed by composition of the existing five and is high-value enough to ship — the most likely candidate is a structured imaging/radiology tool, which is genuinely a different shape from the labs/notes/meds axis. At that point we add it and accept the verifier-contract growth.

#### Memory — per-session scratchpad, not cross-session

**What we chose.** An append-only scratchpad scoped to the pre-rounds session (one shift = one session). No cross-session state.

**What we rejected.** Persistent cross-session memory ("the agent remembers what you asked it last week"). Per-patient longitudinal memory carried across days.

**Why this is defendable.** The hospitalist shift is the natural unit of the task — pre-rounds at 6:30 AM is a complete bounded workflow that ends when rounds start. Cross-session memory adds three problems we don't earn back: HIPAA retention semantics (every memory write is now PHI in our store, with its own audit and minimum-necessary obligations), staleness risk (yesterday's overnight delta is actively misleading today), and prompt-injection persistence (a poisoned chart text yesterday becomes a poisoned context tomorrow). For UC-1, UC-2, and UC-3 the within-session scratchpad is sufficient; nothing in `USERS.md` requires more. A reviewer asking "why no memory?" is usually asking the wrong question — the right one is "why would memory help here?" and we don't have a clinical answer.

**What would change our mind.** A use case where genuine cross-shift continuity is required — most likely a panel-management or longitudinal-discharge-followup workflow. If we add it, memory becomes its own audited subsystem with explicit retention rules, not a flag we flip.

#### Model default — Sonnet 4.6 + Opus 4.7 escalation, not Opus always

**What we chose.** Claude Sonnet 4.6 as the default for ~85% of calls; Opus 4.7 as a targeted escalation for the synthesis-heavy ~15% (UC-1 ranking when >40 events in window, and UC-3 draft-vs-chart cross-text reasoning). Both under Anthropic's signed BAA with zero-retention.

**What we rejected.** Opus 4.7 for everything (cost runs ~5× higher with no measurable quality lift on routine tool composition). Sonnet 4.6 for everything (drops on UC-3's cross-text reasoning in our eval). OpenAI (BAA available but historical data-use policy churn raises compliance pushback). A self-hosted Llama 3.3 70B (no audit-grade vendor relationship and tool-calling reliability under 70B is empirically thin for clinical use).

**Why this is defendable.** The Sonnet/Opus split is not a hedge — it is a measured cost/quality decision driven by what we see on the eval. Routine tool composition (call three tools, return structured rows with citations) is a strict-tool-following task and Sonnet handles it reliably; the synthesis-heavy minority is where Opus's reasoning depth measurably moves the citation-faithfulness number. Defaulting to Opus for everything would burn the cost envelope without changing the verifier's accept/reject decision, because the verifier is the actual quality gate. A reviewer asking "why not Opus everywhere?" is asking us to spend more for a property we already get; the answer is that we measured it. A reviewer asking "why Anthropic over OpenAI?" is asking about vendor lock-in; the answer is the BAA terms (zero-retention is a config toggle, not a price tier), the historical data-use policy stability, and that the model client is thin enough to swap to GPT-4o behind it if the BAA picture changes.

**What would change our mind.** A hospital with an OpenAI-only BAA mandate (we port to GPT-4o, the eval set tells us if quality drops). An on-prem requirement (we move to a local 70B with tool-calling tuning, accepting the deployment burden). Or our own eval showing Sonnet's gap on UC-3 closes — at that point we drop the escalation and run Sonnet-only.

#### Data egress — FHIR R4 read-only, not direct DB

**What we chose.** A three-tier hybrid where FHIR R4 read-only endpoints handle clean resources (Patient, Observation, MedicationRequest, etc.) and a MySQL read replica — accessed only by the gateway, never the agent — handles time-window queries that FHIR is too slow at. A 15-minute Redis cache sits in front of both. No write paths in week 1.

**What we rejected.** FHIR-only (per-resource latency on a 13-patient overnight delta runs ~600ms+ — kills the 2.3s p50 budget). Direct MySQL from the agent process (bypasses gateway ACL re-check and audit-write; risks pulling legacy `library/*.inc.php` includes per `AUDIT.md` §1.4). Vector DB over notes (deferred — FHIR's structured surface is sufficient for the three use cases).

**Why this is defendable.** The hard constraint is that the agent process never opens a database connection. Every byte the agent sees is a structured tool result that has already passed the gateway's ACL recheck and been written to the audit log. The replica path lets us hit 80ms p50 on the queries FHIR is bad at, while preserving the ACL-and-audit envelope because only the gateway holds the replica handle, on a read-only DB user. A reviewer pushing back on "why a replica at all, isn't FHIR the standard?" is right that FHIR is the contract — and the agent's contract is FHIR-shaped tool results regardless of which physical store served them. The replica is a latency optimization that is invisible above the gateway. A reviewer pushing back on "why not vector search?" is imagining a free-text RAG architecture that we explicitly don't need: the use cases are time-windowed structured queries, not similarity search over note text.

**What would change our mind.** An institution that runs OpenEMR on managed Aurora with no replica available — we fall back to FHIR-only with per-patient pre-fetch and a larger Redis cache, accepting a slower p95. Or a use case that genuinely requires note semantic search (e.g., "find every note that mentions difficulty breathing in the last week"), at which point a small vector index over `pnotes.body` becomes its own audited tool.

#### Auth — shared `X-Agent-Token` for week 1, not SMART-on-FHIR

**What we chose.** A pre-shared `X-Agent-Token` header between the OpenEMR module and the agent gateway for the week-1 demo, with a documented migration path to SMART-on-FHIR launch + OAuth2 introspection in week 2.

**What we rejected.** Building the full SMART-on-FHIR launch in week 1 (blocks the demo for 2–3 days of OAuth-client setup that doesn't change the agent's behavior). Skipping auth entirely (non-starter). A custom signed-cookie scheme (re-implements OAuth badly).

**Why this is defendable.** The week-1 deliverable is the agent thesis — five layers, five tools, verifier-led safety, BAA-bound infra. SMART-on-FHIR is a deployment-credentials problem, not an agent-design problem; it changes one header on the gateway and the migration is a config swap because the rest of the auth stack (FHIR `Practitioner` lookup, ACL recheck, audit write) already runs the same way. A reviewer pushing back on "you don't have real auth" is right that the demo is not production-deployable, but the demo isn't the deployment target — the demo is the thesis under synthetic data, and shared-token + synthetic-only is the standard-of-practice posture for that. We document the production migration explicitly so there's no ambiguity about what's deferred vs. broken.

**What would change our mind.** Going to a real-PHI environment (the shared token must be replaced with SMART-on-FHIR before any real data flows — not week-2 work, blocker work).

#### Caching — system block + tool descriptions, not per-patient

**What we chose.** Anthropic prompt caching applied to the frozen system block (the cite-or-refuse charter and the five tool descriptions, ~3,500 tokens), with per-request volatile content (patient list, time window) placed after the cache breakpoint. Expected hit rate ≥85% within a single morning's session.

**What we rejected.** Per-patient response caching (cache the whole answer for `(patient_id, query_shape)`). No caching at all (leaves ~$0.10/morning on the table per hospitalist).

**Why this is defendable.** The cache rule is "cache where the bytes are stable, not where they are hot." The system block is stable across every request in a morning; the per-patient body is volatile by definition (the entire point of the agent is that patient state changes). Caching a per-patient response would be either incorrect (serving stale data) or trivially invalidated (every new tool result invalidates it), so the caching surface is small by design. A reviewer asking "why not cache more aggressively?" is asking us to trade correctness for cost, and in clinical AI that trade is the wrong direction. The system-block cache pays back ~85% of the input-token bill for free with no correctness risk.

**What would change our mind.** A measurably higher hit rate on a smaller cache (e.g., if we found that 70% of UC-1 queries actually share a patient sublist, we'd consider caching the per-patient retrieval intermediates, not the response). Or a per-shift "reading materials" precompute that lives behind the cache breakpoint as stable bytes.

#### Observability stack — LangSmith + OTel + Sentry now, Langfuse self-hosted later

**What we chose.** Three tools with non-overlapping jobs: an LLM-span layer (LangSmith for the week-1 synthetic-only demo, with self-hosted Langfuse as the production target inside the same Railway BAA boundary), OpenTelemetry for system traces and metrics, and Sentry for unhandled exceptions. A custom append-only Postgres `audit_log` table is the HIPAA-grade record of record — that lives in §5, not the observability stack.

**What we rejected.** A single-vendor stack (LangSmith or Datadog for everything — couples LLM-debugging cadence to vendor-BAA negotiations and locks us in by instrumentation). LangSmith as the *production* answer (third-party PHI traces are a BAA-fragile posture). Braintrust (eval-focused, not HIPAA-shaped audit). OpenTelemetry alone (not LLM-aware — tool calls and prompts don't survive the standard span model cleanly). Custom audit table alone (not observability — can't slice by trace, can't graph latency).

**Why this is defendable.** The three audiences for observability — SRE (is the service up and fast?), LLM debugging (why did this answer take 4s and refuse?), HIPAA audit (who saw what record when?) — have genuinely different schemas, different retention requirements, and different access controls. Forcing all three into one tool means one of them is wrong: the SRE view becomes too noisy, the LLM view loses prompt-level detail, or the audit view ends up co-mingled with debug data. OpenTelemetry is the seam that makes this not a vendor-lock problem: instrument once against OTel and the same spans flow to LangSmith today and self-hosted Langfuse tomorrow without touching application code. Sentry exists for a different failure mode than tracing — LangSmith / Langfuse traces *successful but suspect* LLM behavior (slow responses, low cache hit rate, refusal patterns); Sentry catches *outright crashes* (unhandled exception, 500 from FHIR, malformed tool result). Both are needed because a clinical agent silently throwing 500s is just as undeployable as one silently shipping low-quality answers. A reviewer asking "why three tools?" is asking us to defend complexity; the answer is that the three failure surfaces don't overlap, and a single tool that tried to cover all three would be the worst of three things instead of the best of one.

**What would change our mind.** Self-hosted Langfuse adds >5% latency to the request path — at that point we move it to async-only ingestion (fire-and-forget after response), or stay on LangSmith if Anthropic / LangChain offers a real BAA tier covering PHI traces. Or a single vendor ships a genuinely HIPAA-shaped offering with LLM-level span detail and SRE metrics under one BAA — we'd consolidate to it and keep OTel as the abstraction so the migration is a config change.

---

## 10. Risks and Mitigations

This is the worry list — every realistic way the system could hurt a patient, leak data, or fail in deployment, paired with the specific control that catches it. The two catastrophic risks are silent omission of an overnight nursing note and an out-of-date med getting pasted into a real chart; both are caught by the verifier. The high-severity ones are mostly about HIPAA posture and the legacy OpenEMR audit gap, which is exactly why the gateway owns its own audit log. If a peer asks "what keeps you up at night about this," this table is the answer.

| Risk | Severity | Mitigation | Trace |
|---|---|---|---|
| Silent omission of overnight RN note | **Catastrophic** | Verifier count parity (`§4 Layer 4 check 3`); enumerate-by-PK in tool (`§3 contract 1`) | `USERS.md` §7, `AUDIT.md` §2.2 |
| Out-of-date med list pasted to note | **Catastrophic** | DRAFT watermark + non-copyable frame; med-freshness verifier check 4 | `USERS.md` §7 |
| Wrong-patient data | **High** | One `patient_id` per tool call; gateway binds to request | `kb/research/healthcare-personas-and-usecases.md` Hospitalist failure |
| OpenEMR audit log gap (the `log` table is conditional + indexes weak) | **High** | Agent owns its own append-only audit; OpenEMR audit is supplementary not authoritative | `AUDIT.md` §5.1 |
| Default `admin/pass` left enabled | **High (deploy gate)** | Rotation script in deploy README; agent refuses session if user == `admin` outside dev mode | `AUDIT.md` §1.2 |
| LLM provider retains transcripts beyond BAA | **High** | Anthropic zero-retention configuration; week-1 demo uses LangSmith on synthetic data only (no PHI in traces); production target is self-hosted Langfuse inside the BAA boundary; documented in IR plan | `PROJECT.md` Q4 |
| JWT replay between widgets | **High** | `jti` cache in agent gateway with 60s TTL; replayed `jti` returns 401 | `§6` Step 5(c) |
| Module HMAC secret leaked | **High** | Secret rotated quarterly + per-deploy; never in code; lives in Railway env var | `§6` Step 3 |
| FHIR Practitioner lookup fails (transient) | **High** | Fail-closed: deny request + circuit breaker returns 503 not 200; never default to allow | `§6` Step 5(e) |
| Sensitive-segment opt-in skipped | **High** | Gateway hard-codes the segment list; module cannot bypass; opt-in headers signed in audit row | `§6` Step 8, `USERS.md` §8 #4 |
| Cache invalidation between patients (poison) | **Med** | Cache key includes `patient_id`; patient context after cache breakpoint | `claude-api` skill prompt-caching guidance |
| Verifier latency exceeds 1500ms | **Med** | Run count-parity queries in parallel; degrade message if budget exceeded | `PROJECT.md` Open Q §5 |
| Multi-replica install race (bit us already) | **Med (deploy)** | OpenEMR pinned to 1 instance until install idempotency is fixed upstream | `§7` |
| Resident treats agent output as authoritative | **Med (training)** | DRAFT watermark + attending sidebar showing what agent told the resident | `USERS.md` §7 |
| Opus 4.7 thinking blocks omitted by default in stream | **Low** | Use `thinking: {type: "adaptive", display: "summarized"}` only on UC-3 explanations where reasoning is shown to the user | `claude-api` skill |

---

## 11. What Is Out of Scope (Week 1)

Saying no out loud is part of the architecture. This list is the set of things we deliberately did not build in week 1 — write-back to OpenEMR, break-glass mode, the patient portal, cross-session memory, the full sensitive-segment opt-in flow, the other clinician personas, multi-tenant isolation, and a real SMART-on-FHIR launch. Each of these is a real piece of work, and naming them here means we can defend the scope without arguing about whether we forgot.

Stated explicitly so the architecture defense can be defended without scope drift:

- **Write-back to OpenEMR.** No agent-authored notes, orders, or chart entries.
- **Break-glass mode.** Hospitalist case is the low-break-glass case; ED is not in scope.
- **Patient-portal integration.** `AUDIT.md` §1.5 — separate identity class; not the right user class for week 1.
- **Cross-session memory.** Per-shift only.
- **Sensitive-segment opt-in flow.** Default-omit + tell the user. Full opt-in UX is week 3+.
- **PCP / ED / Specialist / Nurse personas.** Documented secondary; `USERS.md` §5.
- **Multi-tenant isolation.** Single-site demo only. Multi-tenant is a Railway project-per-tenant pattern, deferred.
- **SMART-on-FHIR launch.** Demo uses shared token; production launch deferred to week 2.
- **Local model inference.** Anthropic + BAA is the week-1 default; a local Llama-3 / Qwen tier is a cost optimisation, not a thesis-bearing change. Revisit week 6+.
- **Real PHI.** Synthetic Synthea data only this week. Demo-only environment until a hospital BAA is in hand.
- **Voice / mobile UI.** The sidebar widget inside the existing OpenEMR encounter view is where clinicians already are; voice and mobile are explicitly out of scope.

---

## 12. Defensible Thesis

This is the one paragraph the user should be able to deliver from memory. The point of the system is not that it generates fluent prose — fluent prose is easy and dangerous. The point is that it structurally cannot ship a clinical claim that isn't anchored to a real record. Two-layer verification, BAA-bound infrastructure, ACL inheritance with gateway recheck, and a workflow-anchored use case that gives the hospitalist back an hour. Every decision in the document traces to a specific failure we'd rather catch than ship.

Production clinical AI lives at the 85→100% gap. We close it with **two-layer verification (citation + domain constraints)**, **BAA-bound infrastructure**, **ACL-inherited authorization with gateway recheck**, and **a workflow-anchored use case that gives the hospitalist back an hour**. Every architectural decision in this document traces to a specific failure we'd rather catch than ship — `AUDIT.md` finding or `USERS.md` failure mode.

> The agent's value is not that it generates prose; it is that it structurally cannot ship an unverified claim past the user.

That property is what makes it deployable in a clinical setting on a one-week build.

---

## Appendix — Key Decisions in Detail

§9.1 is the scannable, four-part summary of each decision in the user's voice. This appendix is the long form: a one-line punch, a plain-English description for a non-technical reader, a comparison table of alternatives, the rationale behind the chosen option, the reason each alternative was rejected, the falsifier that would move us off the choice, and the trace back to specific `AUDIT.md` findings or `USERS.md` use cases. The reader who wants the headline should use §9.1; the reader who wants to interrogate a specific decision should come here.

The eight subsections below match the eight defense-deck decision slides (`Week 1/deck/21-28-*.html`). A ninth subsection covers three secondary decisions that appear in §9.1 but do not have their own deck slide.

### A.1 Integration path

**The decision.** Hybrid: a thin OpenEMR custom module forwards identity and renders responses; an external Python/FastAPI agent service runs the LLM, tools, and verifier in its own process.

**In plain English.** The agent has two homes. A small piece lives inside OpenEMR — just enough to put a chat panel next to the chart and to tell the outside service "this is Dr. Smith, looking at patient 73." The actual brain — the language model, the lookups, the safety checker — lives in a separate Python service that the OpenEMR module talks to over the network. We do this so the brain can be rebuilt every week without touching the EHR, and so the messy parts of OpenEMR's old code can never be reached by the LLM.

**Options considered.**

| Option | What it is | Pros | Cons |
|---|---|---|---|
| **Hybrid: thin module + external Python agent** ✓ | OpenEMR custom module is a UI shell + auth proxy; agent service is a separate Python process | Independent deploy cadence; agent never touches legacy SQL surface; failure isolation; real LLM SDKs available in Python | Two deploy targets; cross-process auth handshake to design |
| Sidebar widget only (option A on slide 21) | Templates inside `/interface/patient_file/` calling the agent inline | Fastest UX; lowest infra | Modifies core templates; brittle on UI refactors; demoware |
| External agent only, no module (option B on slide 21) | Standalone web app, OAuth2 to FHIR, no OpenEMR UI integration | Cleanest boundary; zero OpenEMR code | Loses the "lives in the workflow" thesis — clinicians won't context-switch during pre-rounds |
| Full custom module, agent in-process (option D on slide 21) | Python-via-PHP-bridge or pure-PHP LLM client running inside OpenEMR's PHP-FPM | Single deploy; single process | Couples deploy cadence; PHP LLM ecosystem is thin; pulls legacy `library/*.inc.php` SQL-interpolation files into the agent's transitive surface (`AUDIT.md` §1.4) |

**Why we chose hybrid.** The deploy cadences are different — the agent will iterate weekly on prompts, tools, and verifier rules; OpenEMR will not — and coupling them slows both. The legacy include surface is a real, documented risk: `AUDIT.md` §1.4 cites specific files where SQL is still string-interpolated, and an in-process agent can pull those files transitively through autoload. By keeping the module thin and the agent external, the worst legacy code path is never reachable from agent code. The "external only" variant loses the workflow-anchoring thesis from `USERS.md` §3 (the 90-second moments are pre-rounds, inside the chart) — a separate URL is not a workflow.

**Why we rejected the alternatives.**
- **Sidebar-only (A):** every UI refactor in OpenEMR core breaks our touch-points; the demo would survive but a hospital deployment would not. It also concentrates all logic in PHP, which has the same legacy-surface problem as option D.
- **External-only (B):** the user behaviour problem is decisive — clinicians do not open a second browser tab during pre-rounds. The architecture is technically clean but the use case is dead.
- **Full in-process (D):** beyond the legacy surface in `AUDIT.md` §1.4, the BAA blast radius is wrong — PHI traveling through the LLM client now lives in the same process memory as the rest of OpenEMR's session state, expanding what an exception trace, error log, or memory dump exposes. `AUDIT.md` §5.1 already flags OpenEMR's audit log as structurally weak, and binding the agent into that process inherits that weakness.

**What would change our mind.** OpenEMR shipping first-class FastAPI tool-calling primitives in core — at that point the deploy-cadence and ecosystem arguments collapse and we consolidate. Or a hospital network policy that forbids external services entirely; in that case we fall back to an in-process Composer sidecar, eat the slower iteration, and document the audit risk explicitly.

**Trace.** `AUDIT.md` §1.4 (legacy SQL string interpolation in `library/*.inc.php`) + §5.1 (audit log structural weakness) + Constraint 1 (the agent must run as a separate service, not as code inside OpenEMR) → external Python service is required, not optional. `USERS.md` §3 (90-second moments inside the chart) → a UI shell inside OpenEMR is required, not optional. The intersection is hybrid.

### A.2 Agent framework

**The decision.** LangGraph for the orchestrator (explicit state machine, typed edges) with the Anthropic SDK called directly inside each node. No LangChain wrapper around the model call.

**In plain English.** The agent's job is a five-step routine: figure out what the user asked, plan which tools to call, run the tools, check the answer, then either reply or refuse. We build that routine as an explicit flowchart (LangGraph) so we can see every step in a trace and replay any failure. Inside each step we talk to Claude using Anthropic's own library directly, instead of going through a wrapper, because the wrapper consistently lags Anthropic's new features by weeks and we need those features (prompt caching, structured thinking) to hit our cost and latency targets.

**Options considered.**

| Option | What it is | Pros | Cons |
|---|---|---|---|
| **LangGraph + Anthropic SDK direct** ✓ | Explicit graph + typed Pydantic state; SDK called inside nodes | Deterministic, replayable; time-travel debugging; YAML-portable spec; full SDK feature access (caching, thinking) | Two libraries to maintain; LangGraph 1.0 churn risk |
| LangChain end-to-end | Chains/agents abstraction wrapping both orchestration and the model call | Single library; large ecosystem | Anthropic adapter lags SDK on adaptive thinking, prompt-cache breakpoints, tool-schema evolution; harder to audit a chain than a graph |
| Raw Anthropic SDK + custom loop | No framework — just a Python state machine we write | Zero lock-in; smallest dependency surface | We re-build LangGraph badly; no time-travel debugging without writing it ourselves |
| CrewAI | Multi-agent roles with emergent delegation | Good for creative/exploratory tasks | Emergent behaviour is the wrong shape for clinical determinism — failure modes are not auditable |
| AutoGen | Conversational group-chat orchestration | Strong for research workflows | Implicit state through prompts; hard to inspect mid-flow; research-grade, not production-shaped |

**Why we chose LangGraph + SDK direct.** Clinical determinism is non-negotiable: every state transition has to be inspectable in the trace, every tool call has to be re-runnable from a logged input, every refusal has to map to a node so the verifier's contract is debuggable. LangGraph's typed edges give us that property and the graph spec is portable as YAML — if LangGraph 1.0 breaks compatibility we can port the spec to a custom executor without rewriting the agent. The reason we go around LangChain on the model call itself is empirical: Anthropic ships features (adaptive thinking, prompt-caching breakpoints, the structured `thinking` block) that LangChain's adapter lags on by weeks, and our cost envelope (§8.1) depends on a high cache hit rate that we cannot configure through the wrapper. Home `CLAUDE.md` and the `claude-api` skill both call this out: "prefer direct SDK." We follow it.

**Why we rejected the alternatives.**
- **LangChain end-to-end:** we'd pay the integration cost twice — once for orchestration value LangGraph already gives us, and once for an LLM adapter that consistently trails the SDK. The cost-envelope hit (no cache breakpoints) is the disqualifier, not a preference.
- **Raw SDK + custom loop:** we would re-implement the parts of LangGraph that matter (typed state, replay, time-travel debug) and build them worse on a one-week budget.
- **CrewAI:** emergent multi-agent delegation produces failure modes that cannot be explained on an audit row. A clinical deployment cannot ship that.
- **AutoGen:** the same problem as CrewAI plus a research-shaped API surface that doesn't compose with our verifier contract.

**What would change our mind.** LangGraph adding more than ~200ms/node of overhead — at that point our 2.3s p50 budget breaks and we port the YAML spec to a custom executor with the same node interface. Or LangChain catching up such that the adapter lag disappears across two consecutive Anthropic feature releases — at that point the SDK-direct argument weakens and a single-library stack wins on simplicity.

**Trace.** Home `CLAUDE.md` + `claude-api` skill ("prefer direct SDK"); `USERS.md` §7 failure modes (silent omission, fabricated citation) require deterministic, replayable execution; `AUDIT.md` Constraint 3 (cite-or-refuse must be enforced by code that runs after the LLM, not by prompt instructions) — LangGraph's explicit verifier node is the implementation of that constraint.

### A.3 LLM provider

**The decision.** Anthropic Claude under signed BAA with zero-retention: Sonnet 4.6 as the default for ~85% of calls, Opus 4.7 as a targeted escalation for the synthesis-heavy ~15% (UC-1 ranking when the overnight window has >40 events; UC-3 cross-text reasoning).

**In plain English.** We use Claude. Specifically, we use the cheaper, faster Claude (Sonnet) for the routine work — call three tools, return rows with citations — and we step up to the smarter, slower Claude (Opus) only when the question is genuinely synthesis-heavy. We picked Anthropic over OpenAI and Google because Anthropic's HIPAA contract terms are the most stable, the model's "I don't know" behaviour is more reliable, and the model client is thin enough that we could swap to OpenAI in a day if a hospital BAA forced us to.

**Options considered.**

| Option | What it is | Pros | Cons |
|---|---|---|---|
| **Anthropic Claude (Sonnet 4.6 + Opus 4.7 escalation)** ✓ | Two-model mix under enterprise BAA, zero-retention enabled | Stable BAA terms; strong refusal behaviour; reliable tool-calling; cost-tunable via model split + prompt caching | Single-vendor exposure (mitigated by thin client) |
| OpenAI (GPT-4o / GPT-5) | Enterprise BAA available; comparable cost | Mature ecosystem; competitive quality | Historical data-use policy churn raises compliance pushback; tool-use less reliable on long contexts in our eval |
| Google Gemini 2.5 (Vertex) | Cost-attractive, configurable retention | Lowest input price; long context | Less mature for agent/tool-use workflows; smaller third-party tooling ecosystem |
| Self-hosted open weights (Llama 3.3 70B / Qwen) | Local inference on Anthropic-class hardware | No vendor BAA needed; data never leaves perimeter | Tool-calling reliability under 70B is empirically thin for clinical use; deployment burden cannibalizes a one-week build; no audit-grade vendor relationship |
| Multi-provider router | Dynamic fallback across vendors | Vendor-neutral; resilience | Doubles BAA negotiation cost; routing logic adds a failure mode and a latency hop |
| Consumer endpoint (no BAA) | Public Anthropic / OpenAI API without enterprise terms | Lowest setup cost | HIPAA blocker — disqualifying |

**Why we chose Anthropic.** The Sonnet/Opus split is a measured cost-quality decision driven by what we see on the eval, not a hedge. Routine tool composition is a strict-tool-following task and Sonnet 4.6 handles it reliably; the synthesis-heavy minority is where Opus 4.7's reasoning depth measurably moves citation faithfulness on UC-3's draft-vs-chart cross-text reasoning. Defaulting to Opus everywhere would burn the cost envelope (§8.1) without changing the verifier's accept/reject decision, because the verifier is the actual quality gate. Across vendors, Anthropic wins on three measurable axes: BAA terms (zero-retention is a config toggle, not a price tier), refusal quality (the "I don't know" behaviour holds up on our eval set in a way that matters for cite-or-refuse), and prompt-caching pricing (the system-block cache is what makes our $0.03–0.09/session math work). The model client is thin enough that GPT-4o is a one-day port if a hospital's BAA mandate changes.

**Why we rejected the alternatives.**
- **OpenAI:** the BAA exists, but the data-use policy has churned multiple times and that churn shows up as compliance-officer pushback in real deployments; the tool-use reliability gap on long contexts is a measurable second reason, not the primary one.
- **Gemini:** the cost is attractive but the agent-shaped tooling is less mature, and our cost envelope is already met by Sonnet + caching, so we don't earn the immaturity tax.
- **Self-hosted open weights:** under-70B tool-calling reliability is empirically thin for clinical workflows; the deployment burden alone (vLLM cluster, GPU procurement, tool-use tuning) does not fit a week-1 budget; and there is no audit-grade vendor relationship to point a hospital security review at.
- **Multi-provider router:** every additional vendor doubles BAA negotiation; the routing logic is a new failure surface; for a one-week build the optionality is not earned.
- **Consumer endpoint:** disqualifying — no BAA means no real PHI, ever.

**What would change our mind.** A hospital with an OpenAI-only BAA mandate (we port to GPT-4o, the eval set tells us if quality drops). An on-prem requirement (we move to a local 70B+ model with tool-calling tuning, accepting the deployment burden as a multi-week project). Or our own eval showing Sonnet 4.6 closes the gap on UC-3 cross-text reasoning — at that point we drop the Opus escalation and run Sonnet-only.

**Trace.** `AUDIT.md` §5.3 (BAA boundary covers LLM provider, observability vendor, hosting) → BAA is required; consumer endpoints disqualifying. `USERS.md` §7 (silent omission, fabricated citation as catastrophic failure modes) → refusal quality is a primary axis, not a tiebreaker. §8.1 cost envelope → Opus-everywhere is over budget; the split is measured.

### A.4 Verification approach

**The decision.** A separate, deterministic Python verifier runs after the LLM produces its draft and performs five checks (citation completeness, window integrity, count parity, med-rec freshness, DRAFT watermark) before any output reaches the user. Cite-or-refuse is a property of code, not of the prompt.

**In plain English.** When the language model finishes writing its answer, we don't show it to the doctor yet. First we hand it to a small piece of regular Python code that opens the answer, finds every claim, checks that each claim points to a real record we actually fetched, checks the time window matches, checks we didn't drop any patients, checks that anything called "current" is actually current, and stamps it as a draft. If any check fails, the user sees a refusal — "I can't confirm this; here's what's missing" — instead of a polished sentence we can't back. The check is code, not a prompt to another LLM, because hospital security reviews want to see the lines of code that guarantee the property and run unit tests against them.

**Options considered.**

| Option | What it is | Pros | Cons |
|---|---|---|---|
| **Deterministic post-LLM Python verifier (5 checks)** ✓ | Rule-based parser + domain-constraint engine running after generation, before user sees output | Auditable; deterministic; cheap (~800ms / $0); has unit tests; passes hospital security review | Extra latency; verifier rules need maintenance as tools evolve |
| LLM-as-judge at runtime | A second LLM call critiques the first LLM's output before it ships | Catches semantic drift; flexible | Shifts the failure mode from one LLM to two correlated LLMs; non-deterministic; non-auditable; +1500–3000ms / +$0.02; "show me the code that guarantees this" answer is "a prompt," which fails security review |
| Self-critique by same LLM | Same model is asked to "now check your work" before responding | Cheap; no second model | Same model's blind spots are the same on the second pass; empirically marginal; still non-deterministic |
| Prompt engineering only | "Be careful, cite everything, refuse if unsure" written into the system prompt | Zero added latency; zero added cost | Empirically caps at ~92–95% citation faithfulness on clinical synthesis (Singhal et al. *Nature* 2023; npj Digital Medicine 2024); the gap from there to 100% is the entire reason the system is deployable |

**Why we chose the deterministic verifier.** The LLM is the source of the failure modes we are trying to catch — hallucinated citations, omitted overnight events, out-of-window claims, stale med lists. Asking an LLM to police those failures shifts the safety property from "code that runs" to "behavior of a stochastic process," and that does not pass a hospital security review. A reviewer asks "show me the code that guarantees this property"; with deterministic verification the answer is `verifier/citation.py`, `verifier/window.py`, `verifier/count_parity.py`, `verifier/med_freshness.py`, `verifier/watermark.py` — five Python modules with unit tests and a CI gate (slide 19). With LLM-as-judge the answer is a prompt and a probability distribution. Empirically the floor matters too: prompt engineering caps at roughly 92–95% citation faithfulness on clinical synthesis, and the gap from there to 100% is the whole reason a clinical deployment is possible at all (the verifier's job is to refuse on the 5–8% the LLM gets wrong, not to make the LLM right). The +800–1,500ms latency is the price of that property and it is worth it for a use case where the user is going to look at the output for 90 seconds anyway. The LLM-judge belongs in the eval tier (A.8), not the runtime path — at eval time we can afford 3s and $0.02; at runtime we cannot, and even if we could the auditability problem doesn't go away.

**Why we rejected the alternatives.**
- **LLM-as-judge at runtime:** the auditability failure is decisive — a hospital security review is not satisfied by "we ask another model to check it." The latency hit (+1500–3000ms breaks the 2.3s p50) and the cost hit (+$0.02/req doubles routine cost) are secondary. The fundamental issue is that the safety property is no longer a property of code.
- **Self-critique by the same LLM:** the same model's blind spots are correlated across passes — if Sonnet hallucinated a citation on pass one, Sonnet on pass two is the wrong reviewer. Empirically the lift is marginal and the auditability problem is unchanged.
- **Prompt engineering only:** the empirical ceiling (~92–95%) is the ceiling. We are deploying into a setting where a 5% omission rate is catastrophic — `USERS.md` §7 lists silent omission of an overnight nursing note as a catastrophic failure mode; the rate matters because the user, by definition, doesn't see what was omitted. A reviewer who pushes back with "well, prompt better" is unintentionally arguing that the deployment is not viable, because no amount of prompt tuning closes the 92→100% gap; only post-LLM code does.

**What would change our mind.** A published model that hits ≥99.5% citation faithfulness on our eval set without external verification, sustained across a model generation (not a single benchmark spike) — at that point the verifier becomes belt-and-suspenders rather than load-bearing. Or our own eval showing the verifier suppresses >5% of correct claims (false-refusal rate) — at that point we add a verifier-LLM tier as a second-chance review for borderline cases, but only inside a tighter latency budget and behind the deterministic checks, not replacing them.

**Trace.** `AUDIT.md` Constraint 3 (cite-or-refuse must be enforced by code that runs after the LLM, not by instructions inside the prompt) — this is the literal sentence the verifier implements. `AUDIT.md` §2.2 (`pnotes` indexed only on `pid`; the overnight-delta hot path) and `USERS.md` §7 (silent omission of an RN note as catastrophic) → count parity (verifier check 3) is required because the data layer cannot guarantee no rows were dropped. *Nature* 2023 / npj Digital Medicine 2024 → 92–95% prompt-engineering ceiling is the empirical reason the verifier is load-bearing. §10 risk table rows 1 ("Silent omission of overnight RN note") and 2 ("Out-of-date med list pasted to note") → both are caught by verifier checks 3 and 4 respectively.

### A.5 Data access layer

**The decision.** Three-tier hybrid: FHIR R4 read-only for clean resources (Patient, Observation, MedicationRequest, AllergyIntolerance, etc.), a MySQL read replica accessed only by the gateway for time-window queries that FHIR is too slow at, and a 15-minute Redis cache in front of both. No write paths in week 1.

**In plain English.** The agent never opens a database connection itself. Every piece of data it sees has come through a controlled doorway — the gateway — that has already checked "is this user allowed to see this patient?" and written down the access. For most reads we use the modern FHIR API (the standard interface for healthcare data). For a couple of queries that FHIR is genuinely slow at — like "give me everything that happened to thirteen patients between 8 PM and 6 AM" — the gateway hits a read-only copy of the database directly, but only the gateway, and only with permissions checked again. A short cache sits in front so the agent doesn't ask the same question twice in one pre-rounds session.

**Options considered.**

| Option | What it is | Pros | Cons |
|---|---|---|---|
| **3-tier hybrid (FHIR + MySQL read replica via gateway + Redis)** ✓ | FHIR for clean resources; replica only for time-window batch; Redis 15-min TTL | Hits 80ms p50 on the slow queries; ACL re-checked on every replica query; FHIR contract preserved above the gateway; agent process never opens a DB connection | Two read paths to maintain; replica needs read-only DB user; Aurora-managed installs may not have a replica |
| FHIR R4 only | Every read goes through the FHIR API | Cleanest boundary; standard contract | Per-resource latency on a 13-patient overnight delta runs ~600ms+; kills the 2.3s p50 budget for UC-1 |
| Direct MySQL from the agent process | Agent code holds a DB connection and writes its own SQL | Fastest on paper | Bypasses gateway ACL re-check and audit-write; risks pulling legacy `library/*.inc.php` SQL-interpolation includes per `AUDIT.md` §1.4 — disqualifying |
| Custom REST shim around OpenEMR | Build our own REST endpoints inside OpenEMR | Tunable to our exact use case | Re-implements FHIR badly; touches OpenEMR core; adds a maintenance surface for marginal benefit |
| `\OpenEMR\Common\Database\QueryUtils` from in-process module | Use OpenEMR's parameterized query helper from within OpenEMR | Inherits OpenEMR's security model | Requires the agent to run in-process (option D from A.1) — already rejected |
| Vector DB over notes | Embedding index over `pnotes.body` for similarity search | Powerful for free-text retrieval | Tricky chunk-level ACL; the three week-1 use cases are time-windowed structured queries, not similarity search |

**Why we chose the hybrid.** The hard constraint is that the agent process never opens a database connection — every byte the agent sees is a structured tool result that has already passed the gateway's ACL recheck and been written to the audit log. The replica path lets us hit 80ms p50 on the queries FHIR is bad at (the overnight-delta batch over 13 patients), while preserving the ACL-and-audit envelope because only the gateway holds the replica handle, on a read-only DB user. The agent's contract is FHIR-shaped tool results regardless of which physical store served them — the replica is a latency optimization that is invisible above the gateway. `AUDIT.md` §1.4 is the hard reason we can't let agent code touch MySQL directly: legacy `library/*.inc.php` files still string-interpolate SQL, and any path that pulls them in transitively is a SQL-injection blast radius we don't need to inherit. FHIR is the supported, parameterized boundary, and the replica preserves that boundary while solving the latency problem.

**Why we rejected the alternatives.**
- **FHIR-only:** the latency math is the disqualifier — 600ms+ per resource × 13 patients × multiple resource types blows the 2.3s p50 budget and degrades the UC-1 batch into something a hospitalist won't use. We tried it; it doesn't work for the batch shape.
- **Direct MySQL from the agent:** two failures stacked. First, it bypasses the gateway's ACL recheck and audit-write, which is the entire point of the gateway (`AUDIT.md` §5.1: OpenEMR's audit log is structurally weak, so the agent owns its own). Second, it pulls legacy SQL-interpolation files transitively (`AUDIT.md` §1.4). Either alone is disqualifying.
- **Custom REST shim:** we'd be re-implementing FHIR with worse semantics and adding maintenance burden inside OpenEMR core for a marginal latency win — the replica gets us the latency win without touching OpenEMR.
- **`QueryUtils` from in-process module:** depends on the agent running in-process, which A.1 already rejected for the legacy-surface and deploy-cadence reasons.
- **Vector DB over notes:** the use cases (`USERS.md` §4) are time-windowed structured queries; similarity search is a different shape and we don't need it for UC-1/2/3. We'd be carrying a vector index, an embedding model, and a chunk-level ACL story for capability we don't use.

**What would change our mind.** An institution that runs OpenEMR on managed Aurora with no replica available — we fall back to FHIR-only with per-patient pre-fetch and a larger Redis cache, accepting a slower p95 and rewriting the UC-1 batch tool to fan out over patients. Or a use case that genuinely requires note semantic search (e.g., "find every note that mentions difficulty breathing in the last week"), at which point a small vector index over `pnotes.body` becomes its own audited tool with its own chunk-level ACL story.

**Trace.** `AUDIT.md` §1.4 (legacy SQL interpolation in `library/*.inc.php`) → agent code cannot touch MySQL directly. `AUDIT.md` §2.2 (`pnotes` indexed only on `pid` — the overnight-delta hot path) → FHIR-only is too slow for UC-1; replica with proper indexes is required. `AUDIT.md` §5.1 (OpenEMR audit log structurally weak) + Constraint 2 (the agent owns its own audit trail) → only the gateway holds the replica handle, on a read-only DB user, with audit-write on every query. `USERS.md` UC-1 (overnight delta over 13 patients in 6:30–8:00 AM window) → batch latency budget is real, not theoretical.

### A.6 Deployment platform

**The decision.** Railway for week 1 (already deployed; BAA available on Pro+; Docker-portable). AWS ECS as the production target at the 10K-user tier, ported behind a Terraform module.

**In plain English.** We need a live URL by Tuesday, and we need a HIPAA contract eventually. Railway gets us a live URL in ten minutes and offers HIPAA terms when we need them. AWS would give us a live URL in two-to-three days. We took the ten-minute option because the demo gate is what matters this week, and we kept everything in Docker so the AWS migration later is a Terraform exercise, not a rewrite.

**Options considered.**

| Option | What it is | Pros | Cons |
|---|---|---|---|
| **Railway (week 1) + AWS ECS (production)** ✓ | Railway services from `/docker/docker-compose.yml`; AWS as planned migration | 10-min time-to-live; BAA on Pro+; already wired (railway skill + project linked); Docker-portable | Railway-specific networking conventions; not where the 10K-user tier ends up |
| AWS now (ECS / RDS / ALB) | Full AWS stack from day 1 | Production-grade; mature BAA; multi-region | 2–3 days of setup; $300+/mo idle; over-built for week-1 demo |
| GCP / Azure equivalent | Equivalent to AWS on a different cloud | Comparable; institutional preference for some hospitals | Same setup cost; no week-1 advantage |
| Render | PaaS comparable to Railway | Mature; good docs | Slower deploy cadence; less agent-friendly tooling; BAA story less clear |
| Fly.io | Edge-deployed PaaS with built-in multi-region | Strong runner-up; BAA available | ~30 min time-to-live; we don't need edge deployment for a single demo region |
| Heroku | Classic PaaS | Familiar | Deprecated free tier; cost-disadvantaged vs. Railway/Fly; weaker container story |
| Vercel + separate DB host | Frontend-biased PaaS plus managed Postgres elsewhere | Best-in-class for Next.js | Frontend-biased — bad fit for stateful PHP; agent service would need a second host anyway |

**Why we chose Railway.** The week-1 deliverable is a deployed URL the cohort can hit. Time-to-live dominates every other consideration when the gate is "submit a working URL by the deadline." Railway gets us there in ten minutes with services for OpenEMR (from `/docker/docker-compose.yml`), the agent (Python FastAPI), MySQL, and Redis as add-ons. The home `CLAUDE.md` confirms we already built a `railway` skill and have a live project — we're not picking the platform, we're confirming the choice we've been running on. Cost discipline matters: $20–60/mo is well inside the $450 Ramp budget, where a $300+ AWS idle cost would not be. Most importantly, everything is in Docker, so the migration to AWS ECS at the 10K-user tier is a Terraform exercise — there is no Railway-specific code to unwind. BAA is the production-tier consideration, not the demo-tier consideration: week 1 is synthetic Synthea data only, so the BAA conversation is "Railway Pro+ when we add real PHI."

**Why we rejected the alternatives.**
- **AWS now:** the 2–3 day setup cost lands inside the demo deadline; the $300+ idle cost lands outside the Ramp budget; we get production-grade infra we don't need for a synthetic-data demo. The right time for AWS is when we have a hospital BAA in hand and a 10K-user load profile, not week 1.
- **GCP / Azure:** same setup cost as AWS; only a win if a specific hospital's institutional preference forces it, which is not our week-1 constraint.
- **Render:** mature but agent-friendly tooling is weaker; the railway skill we already built is the deciding factor — we'd be re-paying setup cost for no gain.
- **Fly.io:** genuine runner-up; the only reason it lost is that we already had a Railway project running. If Railway's BAA disappeared, Fly is the move.
- **Heroku:** the cost curve and free-tier deprecation make it dominated by Railway and Fly on every axis.
- **Vercel:** wrong tool for the job — frontend-biased PaaS, bad fit for stateful PHP-FPM, and the agent service would need a second host anyway, so we'd be running two platforms for no benefit.

**What would change our mind.** Railway's BAA is unavailable when we go to a real-PHI environment — we switch to Fly.io with no application-code changes (Docker portability is the whole point) and accept the longer deploy cadence. Or we hit the 10K-user load profile and Railway's pricing crosses AWS — we execute the planned ECS migration via the Terraform module. Or a partner hospital's institutional preference is GCP/Azure — we port via Terraform variants of the same module.

**Trace.** §7 deployment topology (Railway services already wired); home `CLAUDE.md` (railway skill at `.claude/skills/railway/`); `AUDIT.md` §5.3 (BAA boundary — hosting is in scope); §8.3 cost ($20–60/mo inside the Ramp budget); §11 (real PHI is out of scope for week 1, so BAA-tier hosting is a production-migration concern, not a demo blocker).

### A.7 Observability stack

**The decision.** Three tools for three audiences: OpenTelemetry for system traces and metrics, an LLM-span tool (LangSmith for the week-1 synthetic-data demo, self-hosted Langfuse as the production target inside the BAA boundary), and Sentry for unhandled exceptions. The HIPAA-grade audit-of-record is a separate append-only Postgres `audit_log` table that lives in §5, not in the observability stack.

**In plain English.** Three different people need to see three different things. The on-call SRE asks "is the service up and fast?" — that's OpenTelemetry. The agent team asks "why did this answer take four seconds and why did it refuse?" — that's the LLM-span tool. The compliance officer asks "who looked at patient 73's chart at 6:42 AM?" — that's the audit table. None of these three views can be one tool, because they have different schemas, retention needs, and access controls. We use a third-party LLM-tracing tool for the week-1 synthetic demo because no real patient data is in scope, and we move to a self-hosted version for production so the patient data never leaves our boundary.

**Options considered.**

| Option | What it is | Pros | Cons |
|---|---|---|---|
| **OTel + LangSmith (demo) → Langfuse self-hosted (prod) + Sentry; audit table separate** ✓ | Three-tool stack with the LLM layer migratable; OTel is the abstraction seam | Each audience gets the right view; self-hosted Langfuse keeps PHI inside BAA; OTel makes the LangSmith→Langfuse swap a config change | More moving parts; self-host of Langfuse adds infra cost (~$10/mo) |
| Single-vendor (LangSmith only, or Datadog only) | One tool covers everything | Simpler operations | Couples LLM-debugging cadence to vendor BAA negotiations; one tool can't cover SRE, LLM, and HIPAA audit well |
| Braintrust | Eval-focused LLM observability | Strong for offline eval | Not HIPAA-shaped audit; eval companion only |
| OpenTelemetry alone | Generic distributed tracing | Vendor-neutral; free | Not LLM-aware — tool calls and prompts don't survive the standard span model cleanly |
| Custom audit table alone | Append-only Postgres rows | Regulator-grade; HIPAA-shaped | Can't slice by trace; can't graph latency; not actually observability |
| LangSmith as the production answer | LangSmith for everything LLM, in production with real PHI | Easy; native LLM-aware | BAA-fragile as a third-party vendor for PHI traces; production posture risk |

**Why we chose the three-tool stack.** The three audiences for observability — SRE (is the service up and fast?), LLM debugging (why did this answer take 4s and refuse?), HIPAA audit (who saw what record when?) — have genuinely different schemas, different retention requirements (90 days for SRE, 30 for debugging, ≥6 years for HIPAA), and different access controls (SRE can see traces; only compliance can see the audit table). Forcing all three into one tool means one of them is wrong: the SRE view becomes too noisy, the LLM view loses prompt-level detail, or the audit view co-mingles with debug data. OpenTelemetry is the seam that makes this not a vendor-lock problem — instrument once against OTel and the same spans flow to LangSmith today, self-hosted Langfuse tomorrow, without touching application code. The LangSmith-now / Langfuse-later split is deliberate: week 1 is synthetic data, so a third-party trace store is fine; production carries PHI, so the LLM-span store has to be inside the BAA boundary, and self-hosted Langfuse on the same Railway project is the cleanest answer to "keep PHI inside the perimeter without negotiating yet another vendor BAA." Sentry exists for a different failure mode than tracing — LangSmith / Langfuse traces *successful but suspect* LLM behaviour (slow responses, low cache hit rate, refusal patterns); Sentry catches *outright crashes* (unhandled exception, 500 from FHIR, malformed tool result). A clinical agent silently throwing 500s is just as undeployable as one silently shipping low-quality answers.

**Why we rejected the alternatives.**
- **Single-vendor (LangSmith only / Datadog only):** the schema mismatch is the disqualifier. One tool that tries to be SRE + LLM + HIPAA audit ends up being the worst of three things instead of the best of one. Vendor lock-in via instrumentation makes the eventual migration painful enough that we never do it.
- **Braintrust:** eval-focused, not HIPAA-shaped audit. Good for the eval tier (A.8) as a companion, not the runtime observability stack.
- **OpenTelemetry alone:** the standard span model doesn't carry LLM-shaped data well — tool calls, prompts, token counts, cache breakpoints get smashed into generic attributes. Necessary, not sufficient.
- **Custom audit table alone:** can't slice by trace, can't graph latency, can't correlate with the LLM-span layer. The audit table is regulator-grade evidence; it is not a debugging tool.
- **LangSmith as the production answer:** the BAA-fragility is structural. LangSmith is a third-party vendor receiving PHI traces, and the BAA picture across LLM-observability vendors has been consistently fragile. Defending "we send PHI to a third party for debugging" to a hospital security review is a fight we don't need to have when self-hosted Langfuse exists.

**What would change our mind.** Self-hosted Langfuse adds >5% latency to the request path — at that point we move it to async-only ingestion (fire-and-forget after response), or stay on LangSmith if Anthropic / LangChain offers a real BAA tier covering PHI traces under enterprise terms. Or a single vendor ships a genuinely HIPAA-shaped offering with LLM-level span detail and SRE metrics under one BAA — we consolidate to it and keep OTel as the abstraction so the migration is a config change, not a rewrite.

**Trace.** `AUDIT.md` §5.1 (OpenEMR's audit log is structurally weak) + Constraint 2 (the agent owns its own audit trail; OpenEMR's log is supplementary, not authoritative) → custom append-only Postgres audit table is required, separately from observability. `AUDIT.md` §5.3 (BAA boundary covers observability vendor) → LangSmith is fine for synthetic-only week 1; Langfuse self-hosted is required for PHI. §10 risk row "LLM provider retains transcripts beyond BAA" → zero-retention + self-hosted observability is the mitigation.

### A.8 Eval methodology

**The decision.** Three-tier graded eval: rule-based checks on every PR (citation, auth, schema, refusal logic), LLM-judge on every PR using Opus 4.7 as a different model from the generator (omission, hallucination, tone), human spot-check on every release (5 hardest cases, clinician rubric). Synthea-generated synthetic patients with hand-crafted overnight-event ground truth. The verifier-pass-rate is the headline metric.

**In plain English.** Before we ship any change, we run three checks. First, a deterministic rule-based check on all 50 cases — does every claim have a citation, does the auth boundary hold, does the response match the schema. Second, a smarter language model (Opus, a different model from the one we use to generate the answer) reads every output and flags omissions, hallucinations, and tone problems. Third, a human (a clinician for production; the user plus a friend in medicine for week 1) hand-reviews the five hardest cases against a rubric. The rule check catches the things you can write a regex for. The LLM check catches the things only a smart reader notices. The human check catches the things you can't articulate but you know are wrong.

**Options considered.**

| Option | What it is | Pros | Cons |
|---|---|---|---|
| **Three-tier (rules + LLM-judge + human spot-check) on Synthea** ✓ | Layered eval with different models at each tier | Each tier catches what the others miss; cost-bounded (~$1/run × 10 PRs/day); CI-blockable | More tiers to maintain; LLM-judge variance to monitor |
| Rule-based only | Regex/schema checks alone | Fast; free; deterministic | Misses semantic omission, paraphrase drift; the most dangerous failure mode (silent omission) is invisible to rules |
| LLM-judge only | A second LLM grades the first LLM's output | Catches semantic drift | LLM-on-LLM bias on auth and citation; non-deterministic; can't gate on it alone |
| Production-only (no synthetic) | Eval against real production traffic | Real distribution | Can't test before ship; HIPAA exposure on every eval cycle — disqualifying for clinical |
| Real-time LLM-judge in production | Run LLM-judge on every live response, not just at eval time | Catches drift in production | Doubles cost on every request; the auditability problem from A.4 returns; we already chose the deterministic verifier instead |
| Manual rubric scoring only | Human reads every output | Catches everything | Doesn't scale past ~50 cases; goes stale fast |
| Benchmark against MedQA / MedAlign alone | Public clinical NLP benchmarks | Comparable to the literature | Wrong distribution — those benchmarks measure clinical knowledge, not workflow-anchored cite-or-refuse on overnight deltas |
| No formal eval | Ship and watch logs | Zero overhead | Disqualifying — the omission rate (3.45% per Singhal et al.) is invisible without a synthetic ground truth to compare against |

**Why we chose the three-tier approach.** The failure-mode coverage is the argument: rules catch deterministic structural failures (missing citation, wrong schema, auth boundary violation) where the LLM-judge would be expensive overkill; the LLM-judge catches semantic failures (silent omission, hallucinated paraphrase, wrong tone) that rules genuinely cannot see; humans catch the "I-can't-articulate-why-it's-wrong" cases that even the LLM-judge misses. Singhal et al.'s 3.45% omission rate (*Nature* 2023) is the empirical threshold we care about — silent omission is by definition invisible to the user, so the eval has to expose it. Synthea-generated patients with hand-crafted overnight-event annotations let us build deterministic ground truth (the overnight-event timestamps are the ground truth) and the verifier-pass-rate becomes a binary metric we can graph over time. The LLM-judge runs at eval time — not runtime — because at eval time we can afford 3s and $0.02 per case; at runtime we cannot, and we don't need to (the deterministic verifier from A.4 is the runtime gate). Cost is bounded: ~$1/run × 10 PRs/day = $10/day eval budget, well under the $450 Ramp envelope. The eval blocks PRs in CI on tier-1 (rules) and tier-2 (LLM-judge) regressions; tier-3 (human) blocks releases.

**Why we rejected the alternatives.**
- **Rule-based only:** the catastrophic failure mode (`USERS.md` §7) is silent omission, and silent omission is exactly what rules can't see — by definition the omitted record never appears in the output, so a regex over the output finds nothing wrong. Necessary, not sufficient.
- **LLM-judge only:** LLM-on-LLM bias is real on the auth-and-citation axis (the judge will agree with the generator's "this looks cited"), and the variance across runs makes it a poor sole gate. We need it as the second tier, not the only tier.
- **Production-only:** can't test before ship is the operational disqualifier; HIPAA exposure on every eval cycle is the compliance disqualifier. Both alone are decisive.
- **Real-time LLM-judge in production:** rejected for the same reason A.4 rejected runtime LLM-as-judge — it shifts the safety property from code to a stochastic process. The deterministic verifier is the runtime gate; LLM-judge stays at eval time.
- **Manual rubric only:** doesn't scale past ~50 cases without a paid clinician panel, and goes stale every time the prompt or tools change.
- **MedQA / MedAlign alone:** wrong distribution — those benchmarks measure clinical knowledge (which is the LLM's pre-training problem), not workflow-anchored cite-or-refuse on overnight deltas (which is our deployment problem).
- **No formal eval:** the omission rate is invisible without a ground-truth set to compare against; "ship and watch logs" cannot detect a 3.45% silent-omission rate because the omitted records, by definition, leave no log entry.

**What would change our mind.** LLM-judge variance exceeding 10% across runs on the same eval set — at that point we switch to a fine-tuned smaller judge model with deterministic temperature 0, or fall back to a larger human panel for the semantic tier. Or a hospital partner brings real (de-identified, IRB-approved) clinical scenarios that beat Synthea on ecological validity — at that point Synthea becomes a smoke-test tier and the partner data becomes the eval-of-record.

**Trace.** `USERS.md` §7 (silent omission of overnight RN note as catastrophic failure mode) → omission detection has to be a measurable metric, not a vibe. Singhal et al. *Nature* 2023 (3.45% omission rate on clinical synthesis) → the rate is the threshold; the eval has to expose it. `AUDIT.md` §5.3 (BAA boundary, real PHI is out of scope for week 1) → synthetic data (Synthea) is required, not optional. §10 risk table rows 1, 2, and "Resident treats agent output as authoritative" → all three are tested by tier-3 human spot-check on the rubric.

### A.9 Secondary decisions

These three decisions appear in §9.1 but do not have their own deck slide. They get a briefer treatment because the reasoning is more bounded and less contested.

**A.9.1 Tool count — five maximum.**

The decision: exactly five tools (`list_overnight_events`, `get_labs_window`, `get_med_history`, `get_consult_status`, `get_discharge_blockers`), one per question shape. New use cases are built by composition, not expansion.

The reason: the orchestrator-confusion threshold for tool-calling LLMs is empirically around seven (Ash's lecture; matches our own observation). Past that count, models start confusing near-neighbours and skipping calls. Five gives us headroom and forces composition — every UC in `USERS.md` §4 is a subset of these five. Expansion has a verifier-side cost too: every new tool is a new citation shape the verifier has to know about, and a new failure surface for prompt injection (§10 row "Verifier latency exceeds 1500ms").

Rejected: 10+ rich tools (one per lab type, one per note type, one per consult status — burns the orchestrator's working memory for marginal expressivity). A single "ask anything" tool with free-text query (kills auditability — there's no schema for the verifier to bind to). What would change our mind: a use case that genuinely cannot be expressed by composition of the five — most likely a structured imaging/radiology tool, which is genuinely a different shape from the labs/notes/meds axis.

Trace: `USERS.md` §4 (UC-1, UC-2, UC-3 all expressible as compositions of these five); §3 contract on tool surface.

**A.9.2 Memory pattern — per-session scratchpad, no cross-session.**

The decision: an append-only scratchpad scoped to the pre-rounds session (one shift = one session). No cross-session state. No per-patient longitudinal memory carried across days.

The reason: the hospitalist shift is the natural unit of the task. Pre-rounds at 6:30 AM is a complete bounded workflow that ends when rounds start (`USERS.md` §2). Cross-session memory adds three problems we don't earn back in week 1: HIPAA retention semantics (every memory write is now PHI in our store, with its own audit and minimum-necessary obligations per `AUDIT.md` §5.5–5.6), staleness risk (yesterday's overnight delta is actively misleading today), and prompt-injection persistence (a poisoned chart text yesterday becomes a poisoned context tomorrow).

Rejected: persistent cross-session memory ("the agent remembers what you asked it last week"); per-patient longitudinal memory across days. What would change our mind: a use case where genuine cross-shift continuity is required — most likely a panel-management or longitudinal discharge-followup workflow. If we add it, memory becomes its own audited subsystem with explicit retention rules, not a flag we flip.

Trace: `USERS.md` §2 (pre-rounds is a bounded shift); `AUDIT.md` §5.5 (minimum necessary) + §5.6 (retention enforcement is not a schema property — every persistent store is a new compliance surface).

**A.9.3 Auth posture — `X-Agent-Token` for week 1, SMART-on-FHIR for production.**

The decision: a pre-shared `X-Agent-Token` header between the OpenEMR module and the agent gateway for the week-1 synthetic-data demo, with a documented migration path to SMART-on-FHIR launch + OAuth2 introspection in week 2.

The reason: the week-1 deliverable is the agent thesis — five layers, five tools, verifier-led safety, BAA-bound infra. SMART-on-FHIR is a deployment-credentials problem, not an agent-design problem; it changes one header on the gateway and the migration is a config swap because the rest of the auth stack (FHIR `Practitioner` lookup, ACL recheck, audit write — see §6) already runs the same way. Building the full SMART launch in week 1 burns 2–3 days of OAuth-client setup that doesn't change the agent's behaviour or move any of the eval metrics.

Rejected: building the full SMART launch in week 1 (blocks the demo); skipping auth entirely (non-starter); a custom signed-cookie scheme (re-implements OAuth badly). What would change our mind: going to a real-PHI environment — at that point the shared token must be replaced with SMART-on-FHIR before any real data flows. Not week-2 work; blocker work.

Trace: §6 (the rest of the auth stack — FHIR Practitioner lookup, ACL recheck, audit write — already runs in a way that makes the migration a config swap); `AUDIT.md` §1.5 (patient-portal session is a thinner gate — auth posture matters per identity class); §11 (real PHI is out of scope for week 1, so SMART-on-FHIR is a production-migration concern).
