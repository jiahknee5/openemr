# USERS.md — AgentForge Clinical Co-Pilot

**Source-of-truth document for `ARCHITECTURE.md`. Every architectural decision in the project must trace back to a user, a workflow moment, or a use case described here.**

**Anchored research:** `kb/research/healthcare-personas-and-usecases.md` (six personas, ten use cases, prioritization rationale, authorization model, failure modes) and `kb/research/openemr-codebase-walkthrough.md` (OpenEMR data model, FHIR R4 surface, phpGACL).

---

## Executive Summary

**We built for one user — a daytime hospitalist on a 13-patient morning — because the moment is bounded, recurring, and demonstrably worth ~12 minutes of attending time per shift.**

### The user, in one table

| | |
|---|---|
| **Who** | Dr. Maya Patel, MD — daytime hospitalist, 3rd year out of residency |
| **Where** | 350-bed academic medical center, general medicine teaching service |
| **When** | Tuesday 6:30 AM, pre-rounds, 13-patient service (10 carryovers + 3 overnight admits) |
| **Pressure** | Rounds at 8 AM. Three new admits overnight. Two PGY-2 residents to teach. Daycare pickup at 5:30 PM. |
| **North-star metric** | **Rounds finished by noon** |
| **Worst case** | A missed overnight event she cannot defend at the next morning's M&M |

### Three use cases — the 90-second moments

| | Use case | Time | Why an agent (not a dashboard) |
|---|---|---|---|
| **UC-1** | Overnight delta per patient — ranked by clinical salience | 6:35 AM | Synthesizes 6 tables (`forms`, `form_clinical_notes`, `lists`, `prescriptions`, `procedure_result`, `audit_master`); ranks "fall with head strike" above "slept well" by *reading the note* |
| **UC-2** | Discharge readiness sweep — ten stable patients | 7:20 AM | Walks a multi-source dependency graph (consults, micro, placement, transport, DME, family, follow-up) — no single table answers it |
| **UC-3** | Resident pre-presentation audit — A&P draft vs. chart | 7:50 AM | Reads two free-text artifacts (resident draft + chart), flags omissions; omission-detection is the work |

### The quantified value

| | Without agent | With agent |
|---|---|---|
| Tab-opens before rounds | **~78** (6 tabs × 13 patients) | 1 query |
| Time before any thinking | **~13 minutes** of context-switch | **~80 seconds** total |
| Net | — | **~12 minutes back per morning per hospitalist** |

At a 350-bed academic hospital with ~25 hospitalists, that recovers ~$2.5M/year of attending time against a ~$3,500/year cost envelope (`ARCHITECTURE.md` §8).

### Why not the other personas

| Persona | Why not (week 1) |
|---|---|
| **Primary care physician** | Ambient scribes (Abridge, Suki, DAX) already own the documentation market with signed BAAs |
| **Emergency physician** | Best cognitive-load payoff (~4,000 clicks/shift) but worst safety profile — break-glass, anchoring bias, sentinel-event risk |
| **Specialist (cards consult)** | Too narrow to generalize; value lives in the upstream referral packet, not the EHR |
| **Nurse / RN** | Scope-of-practice landmines; OpenEMR's nurse ACLs too coarse to segregate physician notes |
| **Resident** | Folds in as a **secondary persona** on the same UX with a supervision overlay (UC-3) — no separate build |

### For the demo

> "We picked a hospitalist on a 13-patient morning because the moment is bounded, recurring, and demonstrably valuable. Every decision in `ARCHITECTURE.md` traces back to her workflow."

---

## 1. The User

**Dr. Maya Patel, MD.** Daytime hospitalist, third year out of residency, general medicine teaching service at a 350-bed academic medical center. Tuesday 2026-04-28, day three of a seven-on block. Service this morning: **13 patients** — 10 carryovers, 3 admitted overnight by the nocturnist (78F sepsis-from-UTI; 54M chest pain ruling out ACS on telemetry; 31F DKA). Two PGY-2 residents and one sub-intern present on rounds, 8:00 AM, workroom 4 East. Paid by service-day, not RVU. North-star metric: **rounds finished by noon** — afternoon clinic admin and a 5:30 PM daycare pickup.

Not "physicians." Not "hospitalists in general." A specific narrow user with a specific time pressure, a specific list, a specific worst-case: a missed overnight event she cannot defend at the next morning's M&M.

The agent is built for her. Everyone else is downstream.

---

## 2. The Workflow — Pre-rounds, 6:30–8:00 AM

Pre-rounds is the single most EHR-tethered moment in the hospitalist day. Per [s10.ai](https://s10.ai/blog/ai-scribe-for-hospital-rounds) and [The Hospitalist](https://www.the-hospitalist.org/hospitalist/article/38229/critical-care/to-preround-or-not/), hospitalists document 15–25 patients in 3–4 hours of rounds and pre-rounding stretches "hours on heavy lists." For broader-EHR context: ED counterparts execute ~4,000 mouse clicks per 10-hour shift ([emDocs](https://www.emdocs.net/cognitiveload/)); PCPs spend 36 min/visit in the EHR ([AMA](https://www.ama-assn.org/practice-management/digital-health/primary-care-visits-run-half-hour-time-ehr-36-minutes)). Hospitalist pre-rounds is the same shape: scattered tabs, repeated context-switches, no batch synthesis.

A typical Tuesday:

| Time | Action | EHR friction |
|---|---|---|
| 6:25 | Park, badge in, walk to workroom 4 East | — |
| 6:30 | Boot workstation, open OpenEMR, pull up service list | First load: ~12s |
| 6:32 | Print paper census (still on paper — every hospitalist does this) | Print queue, walk to printer |
| 6:35 | Open patient #1 encounter | Encounter chart loads |
| 6:36 | Click into vitals flowsheet — scan last 24h | Tab #2 |
| 6:37 | Click into MAR — confirm AM meds, look for held/discontinued | Tab #3 |
| 6:38 | Click into nursing notes since 6 PM yesterday | Tab #4 |
| 6:39 | Click into AM labs (if back) and microbiology | Tab #5 |
| 6:40 | Scan consult notes for any new from cards/ID/nephro | Tab #6 |
| 6:41 | Write three lines in pocket carbon copy or paper census | — |
| 6:42 | Repeat for patient #2 |  |

That is **~6 tabs per patient × 13 patients = 78 tab-opens** before rounds, plus the context-switch cost between patients. On a heavy-list morning, work spills past 8:00 AM, rounds end after noon, the resident misses didactics, Maya misses pickup.

OpenEMR's data model makes the friction concrete. A complete pre-rounds delta on one patient touches at minimum: `form_encounter` (admission), `forms` + `form_clinical_notes` (overnight nursing/cross-cover notes), `lists` (problems, allergies, meds), `prescriptions` (active meds, HOLD/DC events), `procedure_result` (AM labs, micro), and `audit_master` (timestamps). Six tables. The agent's job: cross those tables once per patient, ranked by acuity, with citations.

---

## 3. The 90-Second Moments

Three moments in Maya's morning where the agent earns its keep. Each is a place where a conversational, multi-step, cited synthesis is qualitatively different from a dashboard.

### Moment A — 6:35 AM, "First chart open"

- **30s before:** Census printed, coffee in hand. She doesn't yet know who changed overnight.
- **Need:** Ranked per-patient list of overnight events with clinical salience scoring. Triages her *own attention* before opening any chart.
- **What she does:** Reorders rounding sequence. Sees the fall first. Tells covering resident to repeat the lactate before 8.
- **Failure:** "No overnight events" when patient #7 had a 03:14 RN-documented fall with head strike. She skips the neuro exam. Subdural missed. Canonical hospitalist worst case (research §5).

### Moment B — 7:20 AM, "Discharge readiness sweep"

- **30s before:** Pre-rounded on the three sickest. Ten stable patients remaining. Every empty bed at noon is one for the ED upstairs.
- **Need:** Per-patient blockers — pending consult, awaiting culture sensitivity, no SNF, no transport, no DME, family meeting, no follow-up. A dependency graph, not a list.
- **What she does:** One-page printout to case manager at 8:30 — "Patel can go if cards signs off, Williams needs SNF." Case manager works phones during rounds.
- **Failure:** Agent says "no blockers" on a patient with a 04:00 positive blood culture the discharge tool missed because the lab arrived as `procedure_result` but the tool only checked `forms`. Premature discharge → 30-day readmit → CMS penalty.

### Moment C — 7:50 AM, "Resident pre-presentation"

- **30s before:** PGY-2 finished his draft A&P on the DKA admit, about to present. Anxious. 90 min in the chart.
- **Need:** Audit of the resident's draft against the chart — did he mention the AM K+ 2.9? Did he note that home metformin was held? Is his fluid plan consistent with 8h I&O?
- **What she does:** Reads on her phone before he speaks. Clean → lets him present uninterrupted. Omission → nudges instead of catching cold.
- **Failure:** Agent generates "continue lisinopril" when lisinopril was DC'd yesterday for AKI. Resident pastes. Maya cosigns under pressure. Patient re-dosed. Canonical resident worst case (research §5).

Each of these is a *closed loop* in under two minutes — a question, an answer, an action. A dashboard cannot close the loop because a dashboard does not synthesize. A sorted list cannot close it because the sort key is clinical salience, which requires reading the events. A better chart view cannot close it because the question crosses tables. The agent shape is the shape of the question.

---

## 4. The Three Use Cases

### UC-1 — Overnight Delta Per Patient (research use case #3)

**Scenario.** Maya, 6:35 AM, opens the agent sidebar from her service list. She types nothing. The agent has already pre-computed for her: for each of 13 patients, what changed between 6:00 PM yesterday and 6:00 AM today, ranked by acuity.

**What the agent does, step by step.**
1. Reads service list (filtered to attending = `users.id` for Patel, status = inpatient, encounter open).
2. For each patient, calls `list_overnight_events(patient_id, window=[18:00 yesterday, 06:00 today])`.
3. Tool aggregates: `forms` joined with `form_clinical_notes` for nursing/cross-cover notes; `procedure_result` for labs/micro returned in window; `prescriptions` events for HOLD/DC/new-start; `form_encounter` updates for status changes; vitals deltas from the flowsheet form rows.
4. Each event is returned with a citation tuple: `(table, row_id, timestamp, author_user_id)`.
5. Verifier layer checks: every claim has a citation; every citation timestamp is in the window; the count of cited nursing notes matches an independent `SELECT COUNT(*)` on `forms WHERE form_name LIKE 'Nursing%' AND date BETWEEN ...`. Mismatch = the agent reports "I may have missed N notes — please review" rather than fabricating.
6. Ranks by acuity using a deterministic rubric (any RN note tagged "fall" or "rapid response" or "code" → top; any vital sign 2σ off baseline → next; any new positive culture or critical lab → next; otherwise stable).

**Tool calls (PROJECT.md Q1).** `list_overnight_events`, `get_labs_window`, `get_med_history` (only if a med change is detected and Maya drills in).

**Output shape.** Structured. One card per patient. Top of card: room/name/MRN/one-line acuity tag. Body: bullet list of cited events. Each bullet has a hover/click that opens the source artifact in OpenEMR. No prose paragraphs. No "summary." Just a cited delta.

**Why an agent.**
- **Vs. dashboard.** A dashboard sorts by column; it cannot rank by clinical salience. Putting "fall with head strike" above "patient slept well" requires reading the note. Reading the note is the agent's job.
- **Vs. sorted list.** ~150 event rows across 13 patients. Maya has no time to read 150 rows. Compression — *which events matter* — is the value.
- **Vs. better chart view.** Still requires opening 13 charts. The 90-second moment is gone.
- **Vs. Epic SmartList.** A SmartList is canned SQL. It cannot say "this note describes a fall, this one doesn't" without pre-tagging the notes, which RN documentation does not provide.
- **Quantified swap.** 78 tab-opens × ~10s = **13 minutes of context-switch** before any thinking. With agent: one query, ~6s/card, **~80s total. Net ~12 min back per morning.**

---

### UC-2 — Discharge Readiness Check (research use case #4)

**Scenario.** Maya, 7:20 AM. She has triaged the sick patients. She wants to know which of her ten stable patients can go home today, and what is blocking the rest.

**What the agent does.**
1. For each stable patient, calls `get_discharge_blockers(patient_id)`.
2. Tool walks a dependency graph: pending consults (`form_encounter` rows with `pc_status = pending` for consult specialty), awaiting culture sensitivities (`procedure_order` with `procedure_result` not yet final), placement status (free-text in `forms` care-management note — agent must read it), DME orders (`prescriptions` with `is_dme = 1` and no fill record), family meeting flag, follow-up appointment scheduled (`openemr_postcalendar_events` joined on patient).
3. Returns per patient: blocker list with citation, plus a green/yellow/red readiness tag.
4. Verifier checks: every "no blocker" answer must enumerate the categories it checked. The agent cannot say "ready for discharge" without listing the seven categories it looked at. **A "no blocker" answer with an incomplete category list is suppressed and reported as a refusal.**

**Tool calls.** `get_discharge_blockers`, `get_consult_status`, `get_labs_window` (for pending culture sensitivities).

**Output shape.** Structured table: patient · readiness tag · blockers (list) · who-owns-it (case mgr / consultant / family / hospitalist). Maya prints this for the case manager.

**Why an agent.**
- **Vs. dashboard.** Discharge dashboards exist in every modern EHR; none work because blockers live in different tables and the dependency graph is institution-specific. The agent encodes the graph as a tool; a dashboard would need a cross-functional IT project.
- **Vs. sorted list.** "Sorted by length of stay" tells you who has been there longest, not who is ready to leave. Different question.
- **Vs. Epic SmartList.** Single-table queries cannot integrate consult + micro + SNF + transport into one verdict. The integration is the work.
- **Quantified swap.** Without agent: case manager starts cold at 9 AM, interrupts rounds. With agent: printout in hand at 8:30, phones working during rounds, **discharges pended by 10 AM vs noon.** ~2 hr/bed × ~3 discharges/day = real ED throughput.

---

### UC-3 — Resident Pre-Presentation Completeness Audit (research use case #8)

**Scenario.** PGY-2 resident has finished his draft A&P at 7:50 AM. He pastes it into the agent's "audit my draft" textarea. The agent compares his draft against the chart and flags omissions and inconsistencies.

**What the agent does.**
1. Resident pastes draft text. Agent treats it as an unstructured artifact.
2. Calls `list_overnight_events` (same tool as UC-1) and `get_med_history(patient_id, window=24h)` to assemble ground truth.
3. For each clinical claim in the draft (extracted via structured prompt), checks: does the chart support this? Is there a more recent event the draft missed?
4. For each medication mentioned, walks `prescriptions` with `audit_master` events in the last 24h to flag any HOLD/DC the draft did not acknowledge. **This is the canonical resident verifier — see §6.**
5. Returns a side-by-side: resident's claim · chart support (cited or refuted) · omitted-but-relevant events.

**Tool calls.** `list_overnight_events`, `get_med_history`, `get_labs_window`.

**Output shape.** Two-column diff. Left: resident's draft, paragraph by paragraph. Right: chart-grounded annotations. **Watermarked "DRAFT — attending review required" across every output.** Output is not copy-pasteable as a single block — each annotation can be copied individually, but the watermarked frame cannot. This is a deliberate friction.

**Why an agent.**
- **Vs. dashboard.** Dashboards don't read free text. The draft and chart are both free text. Wrong shape.
- **Vs. sorted list.** That's what the resident already has and is failing to integrate. Integration *into his own draft* is the value.
- **Vs. better chart view.** He's been in the chart 90 minutes already.
- **Vs. dot-phrase.** Auto-populates standard sections but cannot detect what the resident *forgot*. Omission-detection is the work.
- **Quantified swap.** Without agent: attending catches it live on rounds, resident is publicly corrected, ~3 min of rounds lost on chart-checking. With agent: resident catches it before presenting, attending teaches *reasoning* not chart-trawling. Audit log preserved for program director review.

---

## 5. Why Not the Other Personas

**Primary care physician.** Volume is huge and pain is real (36 min/visit in EHR), but the dominant pain is *documentation* — and ambient scribes (Abridge, Suki, DAX) already own that market with signed BAAs. A PCP demo collides head-on with billion-dollar incumbents. The "between rooms" co-pilot opportunity is real but requires longitudinal trust — a fabricated history-of-MI surviving in the chart is a sentinel risk, and that's a multi-quarter problem, not a one-week one.

**Emergency physician.** Highest cognitive-load payoff (~4,000 clicks/shift, [emDocs](https://www.emdocs.net/cognitiveload/)) but worst safety profile for a one-week build. Unscheduled patients, no prior relationship, mandatory break-glass per chart, anchoring-bias risk under time pressure. One bad chest-pain answer is a sentinel event. Break-glass (research §4) is where regulatory complexity lives: reason codes, escalated audit, privacy-officer notification ([Yale HIPAA](https://hipaa.yale.edu/security/break-glass-procedure-granting-emergency-access-critical-ephi-systems)). Defer to week 2+.

**Specialist (cardiology consult).** Too narrow to generalize, and value depends on the *referral packet* — upstream of the EHR. Low demo ROI.

**Nurse / RN.** Scope-of-practice landmines (no doses, no differentials, no prognosis), and OpenEMR's nurse ACLs are coarse. Segregating "physician personal notes" from the nursing view needs schema-level tagging that doesn't exist, or a parallel permission store — explicitly rejected (§8).

---

## 6. Secondary Persona — Resident

The resident is **the same workflow as the hospitalist with supervision overhead** — not a separate persona requiring a separate UX. The PGY-2 inherits Maya's sidebar verbatim, with three additive constraints at the gateway:

1. **Attending-of-record link required.** Resident session token must resolve to an active supervision relationship in `gacl_aro` linked to a current attending. No link, no agent — prevents off-service chart-trawling.
2. **Watermarked output.** Every response watermarked "DRAFT — attending review required." UI frame plus copy-operation header. No one-click-paste into a co-signed note. Friction is deliberate.
3. **Attending sidebar.** Every resident query visible to attending-of-record in a separate inbox: question, response, one-click "reviewed." Cosign is informed, not blind.

UC-3 is the resident-facing surface. UC-1 and UC-2 are shared. The supervision overlay is enforced at gateway and renderer, not at the tool layer — no re-architecture.

---

## 7. Failure Modes by Use Case

Per [npj Digital Medicine 2025](https://www.nature.com/articles/s41746-025-01670-7), in clinical LLM summarization the **omission rate (3.45%) exceeds the hallucination rate (1.47%)**. False negatives are the bigger risk. Every verifier below is biased toward catching omission, not toward suppressing hallucination.

| Use case | Worst-case failure | Verifier mitigation |
|---|---|---|
| UC-1 Overnight delta | "No overnight events for Mr. Patel in 412" when an RN documented a fall at 03:14 with head strike. Hospitalist skips neuro exam. Missed subdural hematoma. | (a) Mandatory enumeration of every RN-authored note in the window with `(forms.id, forms.date)` citations — agent must list, not summarize-then-conclude. (b) Independent count query: `SELECT COUNT(*) FROM forms WHERE pid = ? AND date BETWEEN ?` reconciled against the agent's enumerated count — mismatch triggers refusal. (c) Any note that failed to parse causes the agent to refuse with "I may have missed notes," not to silently omit. |
| UC-2 Discharge readiness | "No blockers, ready for discharge" on a patient with a positive blood culture from 04:00 that the discharge tool didn't see because the lab interface delivered it as `procedure_result` but the tool only checked `forms`. False clear → premature discharge → 30-day readmission. | (a) The "ready for discharge" answer must enumerate every category the agent checked (consults, micro, placement, transport, DME, family, follow-up) — incomplete category list = answer suppressed. (b) Any pending culture sensitivity (`procedure_order` finalized = false) blocks the green tag categorically. (c) Refusal on any category that returned a tool error, never silent default-to-clear. |
| UC-3 Resident audit | Agent generates "continue lisinopril" when lisinopril was discontinued yesterday for AKI. Resident pastes into co-signed note, attending cosigns under time pressure, patient re-dosed. | (a) Every medication mentioned in the audit must cite its most recent `prescriptions` row by `(prescriptions.id, audit_master.timestamp)`. (b) Cross-check against the last 24h of `audit_master` events on the prescriptions table — any HOLD/DC must be surfaced adjacent to the med name in red. (c) Watermarked DRAFT output that cannot be one-click-pasted. (d) Attending sidebar shows what the agent told the resident, so cosign is informed. |

The cross-cutting verifier is **cite-or-refuse**: if no grounded source exists for a claim, the agent must refuse. Refusal is a successful response, not an error. This is the grounding contract that makes the rest of the system defensible.

---

## 8. Authorization Model — User-Facing Implications

OpenEMR's `gacl_aro` / `gacl_aco` / `gacl_map` is the single source of truth. The agent does not have a parallel permission store. What this means for Maya in practice:

- **What she sees.** Her 13 service patients — the same list as the OpenEMR census. Reassignment drops them from her view at the next gateway re-check. Full encounter, vitals, MAR, notes, labs, micro, consults — everything her ACL grants in OpenEMR.
- **What is blocked.** Patients not on her service. Sensitive segments (mental health, SUD per 42 CFR Part 2, repro health, HIV) *omitted by default* even on her own patients — agent says "additional consent required to surface this category"; she taps per-encounter affirmation if needed. Minimum-necessary self-restriction even though HIPAA exempts treatment ([HHS](https://www.hhs.gov/hipaa/for-professionals/faq/minimum-necessary/index.html)). Defensibility posture, not regulatory.
- **Audit footprint.** Every retrieval logged in append-only audit: tool called, record read, what was surfaced, what was suppressed. Not just answers — retrievals. Privacy officer reads the log, not Maya.
- **Break-glass.** Maya is a **low break-glass case** — attending-of-record, treatment relationship presumptive, no break-glass for canonical pre-rounds. Reserved for ED (week-2+) and covering-hospitalist edge cases. Path documented, not built, in week 1.

**Principle: if Maya cannot open the chart in OpenEMR, she cannot get a word about it from the agent.** No parallel permission store. No agent-side cache surviving an ACL change. Gateway re-checks every retrieval against the OpenEMR session. Cost: ~80–150ms/hop. Benefit: agent inherits every institutional control and breaks only the way OpenEMR breaks.

End-to-end auth flow detailed in `ARCHITECTURE.md` §6.

---

## 9. Summary

User: **Dr. Maya Patel, daytime hospitalist, 13 patients, 6:30 AM Tuesday, rounds at 8.** Workflow: **pre-rounds** — the most EHR-tethered, batchable, demonstrable moment in inpatient medicine. Headline use case: **overnight delta** — ranked, cited, per-patient summary of 6 PM–6 AM events, replacing 78 tab-opens with one query. The two supporting use cases (discharge readiness, resident audit) share the same tool surface and verifier stack, so week 1 covers all three without architectural fan-out. Other personas are documented as week-2+ extensions. Every decision in `ARCHITECTURE.md` traces back to a moment, use case, or failure mode here. If a piece of the architecture doesn't trace back, it doesn't get built this week.
