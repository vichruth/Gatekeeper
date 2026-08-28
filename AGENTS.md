# AGENTS.md

Instructions for AI coding agents (Antigravity, Claude Code, Cursor) working in this repo.
**Read this fully before writing any code.**

---

## 0. What this project is

**Gatekeeper** — an agentic pipeline that turns internal employee complaints into shipped internal tools, with human approval gates at the points where judgment actually matters.

It started life as a take-home design document. The design was worth building, so this repo builds it. **That origin is stated openly in the README — do not hide or rephrase it.**

**Current state: design is complete (D1–D12). No application code exists yet.** There is no `backend/`, no `frontend/`, no `tests/`, no `docs/`. Your job is to build them, in the phase order in §10.

### Where things live — read this before hunting for files

| What | Where | Note |
|---|---|---|
| The full design (D1–D12) | `design_desicions.md` **in the working directory** | **Gitignored** — present on disk, absent from a fresh clone. This is the authority on *why*. Read it before Phase 2. |
| Architecture diagram | `pipeline-diagram.md` (Mermaid source) | Also gitignored |
| Security posture + known gaps | `SECURITY.md` | Also gitignored |
| Session history / decisions made | `PROJECT_LOG.md` | Also gitignored |
| `docs/DESIGN.md` | **Does not exist yet — you create it in Phase 1** | See §3 |

The private docs are gitignored deliberately: they contain the original internship framing and should not be published. `docs/DESIGN.md` is the public-safe rewrite you produce.

### The n8n JSON files are NOT the implementation

`n8n-pipeline-full.json`, `n8n-pipeline-sketch.json` and `n8n-skeleton.json` are committed, and they are **reference sketches of the design only**. They were used to pressure-test the hand-offs on a canvas. They are deliberately not runnable — stubbed inputs, uncredentialed LLM nodes, NoOp placeholders.

**Do not build on them. Do not use n8n.** §2 fixes the orchestration as plain Python. The JSONs are useful to read for the node-by-node reasoning in their `notes` fields, nothing more.

---

## 1. Non-negotiable rules

1. **Scope discipline over completeness.** A working 60% beats a broken 100%. Several designed features are deliberately NOT built (see §7). Do not build them. Do not "helpfully" add them.
2. **Zero paid services.** Everything must run on free tiers or locally. No AWS, no paid DB, no paid LLM API. If a task seems to need a paid service, stub it and flag it instead.
3. **Never auto-approve a human gate.** The gates are the entire point of the system. An agent bypassing them defeats the design.
4. **Structured output + validation at every LLM boundary.** Never trust an LLM response without schema validation. See §5.
5. **No secrets in the repo.** API keys come from `.env` only. `.env` is already gitignored — verify before your first commit that no key appears in any tracked file.
6. **Honesty in docs.** If something is stubbed, say "stubbed." If a number is a guess, say so. Do not write README claims the code doesn't support. **The README's status table currently says everything is "Not started" — update it at the end of each phase to match reality, and never ahead of it.**
7. **Nothing is silently dropped.** This is the design's central promise, and it is easy to violate accidentally. Every record that cannot proceed gets a status and a reason — never a filter that makes it vanish. See §5's uncertainty rule.

---

## 2. Tech stack (fixed — do not substitute)

| Layer | Choice | Why |
|---|---|---|
| Backend | **Python 3.11+ / FastAPI** | Fast to write, auto-generated API docs |
| Database | **SQLite** | Zero setup, zero cost, file-based. Postgres is overkill here |
| ORM | **SQLModel** | Pydantic + SQLAlchemy in one, less boilerplate |
| Validation | **Pydantic v2** | Schema enforcement on LLM output |
| LLM | **Groq free tier** (default), Gemini fallback | Free, fast. Provider swappable via env var |
| Embeddings | **`sentence-transformers` locally** | See the blocker note in §11 — do not assume the LLM provider offers embeddings |
| Frontend | **Next.js 14 (App Router) + Tailwind** | Deploys free on Vercel |
| Orchestration | **Plain Python** | NOT n8n — self-hosting costs money and plain Python is more readable to a reviewer |
| Testing | **pytest** | Deterministic logic must be tested (see §8) |

### `.env.example` — create this in Phase 1, committed

```
LLM_PROVIDER=groq          # "groq" | "gemini"
GROQ_API_KEY=
GEMINI_API_KEY=            # only needed if LLM_PROVIDER=gemini
DATABASE_URL=sqlite:///./gatekeeper.db
```

Do not hardcode a model name in more than one place — put it in `llm/client.py` as a constant, and **check the provider's current model list rather than trusting a name from memory**; model IDs are deprecated frequently.

---

## 3. Repository structure to create

```
Gatekeeper/
├── AGENTS.md                  # this file
├── README.md                  # exists — keep its status table honest
├── LICENSE                    # exists (MIT)
├── .env.example               # committed
├── .env                       # gitignored, NEVER commit
├── .gitignore                 # exists
├── requirements.txt
├── docs/
│   └── DESIGN.md              # YOU WRITE THIS — see below
├── backend/
│   ├── main.py                # FastAPI app + routes
│   ├── models.py              # SQLModel tables
│   ├── schemas.py             # Pydantic schemas for LLM I/O
│   ├── db.py                  # engine + session
│   ├── agents/
│   │   ├── scout.py           # normalise + cluster
│   │   ├── scorer.py          # threshold + scoring + ranking
│   │   └── definer.py         # PRD generation
│   ├── llm/
│   │   ├── client.py          # provider-swappable LLM wrapper
│   │   └── prompts.py         # all prompts, one place
│   ├── validation.py          # schema + semantic validation
│   └── seed.py                # demo data loader
├── frontend/                  # Next.js approval dashboard
└── tests/
    ├── test_scorer.py
    └── test_validation.py
```

### `docs/DESIGN.md` — a Phase 1 deliverable

The README links to it, so it must exist. Write it by adapting `design_desicions.md` (gitignored, on disk) into a **public-safe** version:

- Keep: all twelve decisions D1–D12, the failure-mode table, the seven test scenarios, the assumptions list.
- Strip: the hiring-company name, any reference to a rubric, evaluation criteria, an interviewer, or "what to say in the document." That framing is private and reads oddly in a public repo.
- Embed the diagram by pasting the Mermaid block from `pipeline-diagram.md` directly — **GitHub renders Mermaid natively, so no PNG export is needed.**

---

## 4. The pipeline to build

```
Ingest → Scout → Scorer → [GATE 1] → Definer → Validate → [GATE 2] → (stub) → [GATE 3] → Done
```

### Data model — one `UseCase` record carries everything

```python
class UseCase(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    raw_text: str                    # original complaint
    source: str                      # "qcm" | "form"
    role_pool: str                   # e.g. "compliance", team the reporter belongs to
    role_pool_size: int              # denominator for the 25% threshold
    reporter_id: str
    hours_lost_per_week: float | None
    severity_self_reported: int | None   # 1-5
    normalised_title: str | None     # Scout output
    cluster_id: str | None           # Scout output
    impact_score: float | None       # Scorer output
    sentiment_score: float | None    # Scorer output
    threshold_met: bool | None
    route: str | None                # "high_signal" | "low_signal_high_conviction"
    prd: str | None                  # Definer output (JSON string)
    confidence: float | None         # every agent writes this
    unresolved_assumptions: str | None   # JSON list[str]
    status: str = "ingested"         # see status enum below
    created_at: datetime
    updated_at: datetime

class DecisionLog(SQLModel, table=True):
    """APPEND-ONLY. Never UPDATE or DELETE a row here."""
    id: int | None = Field(default=None, primary_key=True)
    usecase_id: int
    gate: str                        # "gate_1" | "gate_2" | "gate_3"
    actor: str                       # who decided
    action: str                      # "approve" | "reject" | "edit"
    changes: str | None              # what they changed
    decision_ms: int | None          # how long they took — see §6
    timestamp: datetime
```

**Status values:** `ingested` → `normalised` → `scored` → `gate_1_pending` → `gate_1_approved` / `rejected` → `prd_drafted` → `gate_2_pending` → `gate_2_approved` → `gate_3_pending` → `shipped` / `halted` / `escalated` / `persisted`

Use timezone-aware UTC datetimes throughout. Time decay maths breaks subtly on naive local times.

### Agent responsibilities

**Scout (`agents/scout.py`)**
- Normalises raw complaints from both sources into one shape. LLM-assisted extraction of a clean title + the affected role.
- **Clusters** semantically similar complaints so two teams describing the same problem in different words have their pools **combined before scoring**. Embeddings + cosine similarity, threshold ~0.85.
- **When a cluster spans multiple role pools, the denominator is the sum of the distinct pools' sizes**, and the numerator is the count of distinct reporters across the cluster. This case is the whole reason clustering exists — see test scenario 7.
- **Scout NEVER discards or filters anything.** A silent filter is where good ideas die. Normalise and cluster only.

**Scorer (`agents/scorer.py`)** — mostly deterministic Python, this is the judgment layer
- Threshold: `(weighted_reporters_in_cluster / role_pool_size) >= 0.25`, with time decay — weight each report by `0.5 ** (days_old / 90)` (90-day half-life).
- **Two separate scores, never collapsed into one number:**
  - `impact_score` — from hours lost, recurrence, automatability, blast radius
  - `sentiment_score` — from self-reported severity + LLM tone read
- **Sentiment is a weak signal.** Cross-check against `hours_lost_per_week`. Never let sentiment alone promote a candidate.
- **Two routes, both landing at Gate 1:** `high_signal` (threshold met) and `low_signal_high_conviction` (below threshold but concrete hours + detailed description). Concretely: treat "concrete" as `hours_lost_per_week >= 4` **and** a description of at least ~100 characters. A threshold may deprioritise an idea; it must never silently kill one.
- **Cap the shortlist at 5/run.** Items past the cap get status `persisted` and re-enter the next run's ranking — they are not deleted. **The cap must change the record's routing, not just set a flag** — otherwise capped items still reach Gate 1 and the cap is decorative.
- Guard the zero case: a candidate reporting 0 hours must not divide by zero or collapse a tolerance window to zero width.

**Definer (`agents/definer.py`)**
- Takes the ONE candidate approved at Gate 1 → produces a PRD (problem, users, scope, requirements, success criteria) as structured JSON.
- Runs only after Gate 1 approval. **Never generate a PRD before Gate 1** — a polished PRD anchors the reviewer and biases the "is this worth doing" decision.

---

## 5. Validation (`validation.py`) — build this carefully

Three layers, cheapest first:

**Layer 1 — schema validation.** Parse LLM output into a Pydantic model. On failure: retry ONCE with the validation error fed back into the prompt. On second failure: set status `escalated`, never pass bad data downstream.

> **Bound the retry with an explicit counter that actually persists across the retry.** A retry counter that resets each iteration produces an infinite loop, and this is an easy mistake to make. Three outcomes exist here — pass, retry, escalate — so do not model it as a boolean.

**Layer 2 — semantic validation.** This is the important one and most implementations skip it.

> Shape validation is not enough. A tensor of shape `[3368, 2]` passes a check that only asserts "is this 2D" while being completely wrong. Same risk here: a PRD can be perfectly-formed JSON and still describe a different problem than the one reported.

Concretely, check that:
- the PRD's stated user group matches the `role_pool` that actually reported it
- any claimed time saving is within a sane multiple of `hours_lost_per_week`
- the PRD references terms present in the original `raw_text`

Fail → escalate with the specific mismatch named.

**Layer 3 — self-critique.** Before a human sees the PRD, one LLM call scores it against a checklist: are specific users named, are success criteria measurable, is scope bounded. Attach the result; don't block on it.

**Uncertainty rule:** every agent emits a `confidence` float and an `unresolved_assumptions: list[str]`. Below 0.6 confidence → status `escalated`, with the *specific question* attached — not "I'm stuck" but "could not determine whether this affects the whole team or one squad."

---

## 6. The three gates — who reviews, and how each behaves

The reviewer differs per gate, and that is a design decision, not an implementation detail. The dashboard must make it obvious who is being asked.

| Gate | Reviewer | Question | Action offered | Default on timeout |
|---|---|---|---|---|
| **1** | AI implementation lead | *Is this worth our time?* | Pick from the shortlist, or reject | May proceed (low risk) |
| **2** | **The person who raised the problem** (`reporter_id`) | *Did we understand it correctly?* | Edit and approve | May proceed (low risk) |
| **3** | AI implementation lead | *Is it safe to put this live?* | One-click approve to ship | **HALT — never auto-approves** |

Rules:

- **Gate 3 defaults to halt.** This asymmetry is deliberate: a gate that defaults to proceed is not a control. Do not "fix" it for consistency.
- **Gate 2's reviewer is the reporter, not the lead.** Only they know whether the PRD describes their actual pain. The UI at Gate 2 should show the original complaint next to the generated PRD.
- **Every gate action writes an append-only `DecisionLog` row.** Never UPDATE, never DELETE. At Gate 2, record the field-level diff in `changes` — repeated edits to the same field are evidence the Definer prompt is systematically wrong.
- Record `decision_ms` on every gate action. The rubber-stamp detector is not being built (§7), but the *data it would need* is free to capture now and impossible to reconstruct later.

---

### API surface

```
POST /ingest/form          # single complaint submission
POST /ingest/qcm           # batch of meeting notes
POST /pipeline/run         # weekly batch: scout -> score -> rank -> queue for gate 1
GET  /queue/gate/{n}       # pending items at gate n (1|2|3)
POST /gate/{n}/{id}/approve
POST /gate/{n}/{id}/reject
POST /gate/{n}/{id}/edit
GET  /usecase/{id}         # full record + decision log
GET  /stats                # counts by status
```

`GET /stats` should return counts by status, plus **gate rejection rate** — it is cheap to compute from the decision log and it is the one metric that tells you whether the gates are real. Too low means rubber-stamping; too high means discovery is broken. The headline metric from the design (% of shipped tools still in use after 30 days) is **not** computable here, since the 30-day review is not built — do not fake it.

---

## 7. Deliberately NOT built — do not implement

Stub these as documented no-ops. The README says so explicitly. Building them is out of scope and will not be merged.

| Feature | Why not |
|---|---|
| **Designer agent** (wireframe generation) | Needs the internal platform's capability; out of scope |
| **Builder agent** (app generation) | Same |
| **Rubber-stamp detector** | Designed, not built. Documented in DESIGN.md |
| **30-day post-ship review** | Designed, not built |
| **Real auth / access control** | Demo uses a hardcoded reviewer identity. Flagged as a known gap |
| **Retention policy** | Known gap, documented |

Gates 2 and 3 exist as real endpoints, but the work *between* them (Designer/Builder) is a stub that immediately marks the record ready for Gate 3.

---

## 8. Tests required

Deterministic logic must be tested. LLM calls should not be tested for output content.

- `test_scorer.py` — threshold maths, time decay, the two-route split, shortlist capping, the below-threshold escape path
- `test_validation.py` — schema failure → retry → escalate; semantic mismatch detection

**Use the design's own seven test scenarios as fixtures.** They are listed in `design_desicions.md` and each one exists to prove a specific decision was right. Scenario 7 (two teams, same problem, different wording) is the one the original design *failed* before clustering was added — it must pass.

Target: the scoring module is fully covered. It is the part that encodes actual judgment, and it is the part a reviewer will read first.

---

## 9. Seed data (`seed.py`) — this matters more than it looks

Good seed data makes the system demonstrable in 30 seconds; bad seed data makes it look like it does nothing.

Load ~15 realistic complaints across 3–4 role pools of **differing sizes** (a 4-person team and a 40-person team — the size difference is the point of normalising by pool). The set must include:

- Several clearly **above** the 25% threshold
- Some **below** threshold but with concrete hours-lost data → exercises the `low_signal_high_conviction` route
- **At least two describing the same underlying problem in different words**, across different teams → exercises clustering (scenario 7)
- One **angry but trivial** complaint (high sentiment, ~10 min/week) and one **calm but serious** one (low sentiment, 6 hrs/week) → proves why two scores beat one
- One **judgment-shaped** task ("reviewing vendor questionnaires is exhausting") → should score low on automatability
- Enough volume that the 5-item cap actually triggers and something lands in `persisted`

---

## 10. Build order (follow this sequence)

Do not start a phase before the previous one runs. Commit at the end of each phase.

**Phase 1 — spine.** `.gitignore` + `.env.example` FIRST, then `requirements.txt` (pinned), models, db, ingestion endpoints, Scout normalise (no clustering, no LLM). Seed data per §9. Also write `docs/DESIGN.md` (§3).
*Done when:* `uvicorn` starts, `/docs` loads, seed populates the DB, both ingest endpoints persist data.

**Phase 2 — scoring.** Full Scorer with tests. Nearly all deterministic Python — no LLM in the scoring maths. Add `POST /pipeline/run`.
*Done when:* `pytest` passes, and a seeded run produces a ranked shortlist with items in both routes plus at least one `persisted`.

**Phase 3 — LLM layer.** `llm/client.py` (provider-swappable, retry with exponential backoff on rate limits), `llm/prompts.py`, Scout clustering via embeddings, Definer, all three validation layers, confidence/uncertainty handling.
*Done when:* a PRD generates end-to-end, a deliberately-broken PRD retries once then escalates, and scenario 7's two complaints land in one cluster.

**Phase 4 — gates + dashboard.** Gate endpoints, decision log writes, Next.js approval queue UI, README status table updated.
*Done when:* all three gates are clickable in the UI, every action appears in the decision log, and Gate 3 does not auto-approve.

**Final pass.** Verify nothing from §7 got built. Full test suite green. `.env` untracked and no key in any tracked file. README table matches reality exactly. Every LLM call goes through `llm/client.py`; every prompt lives in `llm/prompts.py`.

---

## 11. Known blockers and open decisions — resolve these deliberately

These are real and will be hit. Do not improvise past them silently.

**1. The LLM provider may not offer embeddings.** Groq serves chat completions; it is not an embeddings provider. Phase 3's clustering needs embeddings, so:
- **Recommended:** `sentence-transformers` with `all-MiniLM-L6-v2` locally — free, no API key, works offline, ~80MB download.
- Alternative: Gemini's embedding model, but that couples clustering to a provider the rest of the code treats as a fallback.
- Verify the current state of the provider's API before choosing; do not trust this note as permanently accurate.

**2. `/pipeline/run` re-run behaviour is undefined.** Running it twice must not double-process or duplicate shortlist entries. Decide idempotency explicitly — most likely: only pick up records in `normalised`/`persisted` status.

**3. Cross-pool clustering denominator.** Specified in §4 (sum the distinct pools). Confirm it behaves sensibly when a cluster spans a 4-person and a 40-person team — the small team's signal should not be drowned.

**4. `prd` is stored as a JSON string.** Define the Pydantic schema for it in `schemas.py` and parse on read. Do not let unvalidated free-form JSON accumulate in that column.

**5. Confidence threshold is 0.6 in this spec.** The n8n sketch used 0.5. This document wins; keep one constant, in one place.

**6. `docs/DESIGN.md` does not exist and the README links to it.** Until Phase 1 creates it, that link is dead. This is the single most visible gap in the repo right now.

---

## 12. Style

- Type hints everywhere. `black` formatting, 100-char lines.
- Prompts live in `llm/prompts.py`, never inline in agent code.
- Every LLM call goes through `llm/client.py`. No direct SDK calls in agent files.
- Docstrings explain *why*, not *what*. The design reasoning matters more than the mechanics — the scorer and validation modules especially, since that reasoning is the valuable part of this project.
- Errors are explicit. No bare `except:`. Nothing is silently dropped — if something can't proceed, it gets a status and a reason.
