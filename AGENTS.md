# AGENTS.md

Instructions for AI coding agents (Antigravity, Claude Code, Cursor) working in this repo.
**Read this fully before writing any code.**

---

## 0. What this project is

**Signal to Shipped** — an agentic pipeline that turns internal employee complaints into shipped internal tools, with human approval gates at the points where judgment actually matters.

It started life as a take-home design document. The design was worth building, so this repo builds it. **That origin is stated openly in the README — do not hide or rephrase it.**

**Current state:** design is complete (D1–D12, see `docs/DESIGN.md`). Code is not written. Your job is to build it.

---

## 1. Non-negotiable rules

1. **Scope discipline over completeness.** A working 60% beats a broken 100%. Several designed features are deliberately NOT built (see §6). Do not build them. Do not "helpfully" add them.
2. **Zero paid services.** Everything must run on free tiers or locally. No AWS, no paid DB, no paid LLM API. If a task seems to need a paid service, stub it and flag it instead.
3. **Never auto-approve a human gate.** The gates are the entire point of the system. An agent bypassing them defeats the design.
4. **Structured output + validation at every LLM boundary.** Never trust an LLM response without schema validation. See §5.
5. **No secrets in the repo.** API keys come from `.env` only. `.env` must be gitignored from commit #1, before any code is written.
6. **Honesty in docs.** If something is stubbed, say "stubbed." If a number is a guess, say so. Do not write README claims the code doesn't support.

---

## 2. Tech stack (fixed — do not substitute)

| Layer | Choice | Why |
|---|---|---|
| Backend | **Python 3.11+ / FastAPI** | Fast to write, auto-generated API docs |
| Database | **SQLite** | Zero setup, zero cost, file-based. Postgres is overkill here |
| ORM | **SQLModel** | Pydantic + SQLAlchemy in one, less boilerplate |
| Validation | **Pydantic v2** | Schema enforcement on LLM output |
| LLM | **Groq free tier** (default), Gemini fallback | Free, fast. Provider swappable via env var |
| Frontend | **Next.js 14 (App Router) + Tailwind** | Deploys free on Vercel |
| Orchestration | **Plain Python** | NOT n8n — self-hosting costs money and plain Python is more readable to a reviewer |
| Testing | **pytest** | Deterministic logic must be tested (see §7) |

---

## 3. Repository structure to create

```
signal-to-shipped/
├── AGENTS.md                  # this file
├── README.md
├── .env.example               # committed
├── .env                       # gitignored, NEVER commit
├── .gitignore
├── requirements.txt
├── docs/
│   ├── DESIGN.md              # the D1-D12 design doc
│   └── diagram.png
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
    timestamp: datetime
```

**Status values:** `ingested` → `normalised` → `scored` → `gate_1_pending` → `gate_1_approved` / `rejected` → `prd_drafted` → `gate_2_pending` → `gate_2_approved` → `gate_3_pending` → `shipped` / `halted` / `escalated` / `persisted`

### Agent responsibilities

**Scout (`agents/scout.py`)**
- Normalises raw complaints from both sources into one shape. LLM-assisted extraction of a clean title + the affected role.
- **Clusters** semantically similar complaints so two teams describing the same problem in different words have their pools **combined before scoring**. Use embeddings + cosine similarity, threshold ~0.85.
- **Scout NEVER discards or filters anything.** A silent filter is where good ideas die. Normalise and cluster only.

**Scorer (`agents/scorer.py`)** — mostly deterministic Python, this is the judgment layer
- Threshold: `(reporters_in_cluster / role_pool_size) >= 0.25`, with time decay — weight each report by `0.5 ** (days_old / 90)` (90-day half-life).
- **Two separate scores, never collapsed into one number:**
  - `impact_score` — from hours lost, recurrence, automatability, blast radius
  - `sentiment_score` — from self-reported severity + LLM tone read
- **Sentiment is a weak signal.** Cross-check against `hours_lost_per_week`. Never let sentiment alone promote a candidate.
- **Two routes, both landing at Gate 1:** `high_signal` (threshold met) and `low_signal_high_conviction` (below threshold but concrete hours + detailed description). A threshold may deprioritise an idea; it must never silently kill one.
- **Cap the shortlist at 5/week.** Uncapped items get status `persisted` and re-enter next week's ranking — they are not deleted.

**Definer (`agents/definer.py`)**
- Takes the ONE candidate approved at Gate 1 → produces a PRD (problem, users, scope, requirements, success criteria) as structured JSON.
- Runs only after Gate 1 approval. **Never generate a PRD before Gate 1** — a polished PRD anchors the reviewer and biases the "is this worth doing" decision.

---

## 5. Validation (`validation.py`) — build this carefully

Three layers, cheapest first:

**Layer 1 — schema validation.** Parse LLM output into a Pydantic model. On failure: retry ONCE with the validation error fed back into the prompt. On second failure: set status `escalated`, never pass bad data downstream.

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

## 6. Deliberately NOT built — do not implement

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

## 7. Tests required

Deterministic logic must be tested. LLM calls should not be tested for output content.

- `test_scorer.py` — threshold maths, time decay, the two-route split, shortlist capping, the below-threshold escape path
- `test_validation.py` — schema failure → retry → escalate; semantic mismatch detection

Target: the scoring module is fully covered. It is the part that encodes actual judgment.

---

## 8. Build order (follow this sequence)

**Phase 1 — spine.** Repo skeleton, `.gitignore` + `.env.example` FIRST, models, db, ingestion endpoints, Scout normalise (no clustering yet). Seed with demo data. No LLM.

**Phase 2 — scoring.** Full Scorer with tests. Still no LLM for scoring. This phase should be nearly all deterministic Python.

**Phase 3 — LLM layer.** LLM client wrapper (provider-swappable via `LLM_PROVIDER` env var), Scout clustering via embeddings, Definer, all three validation layers.

**Phase 4 — gates + dashboard.** Gate endpoints, decision log writes, Next.js approval queue UI, README.

Commit at the end of each phase with a clear message. Do not start a phase before the previous one runs.

---

## 9. API surface

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

---

## 10. Style

- Type hints everywhere. `black` formatting, 100-char lines.
- Prompts live in `llm/prompts.py`, never inline in agent code.
- Every LLM call goes through `llm/client.py`. No direct SDK calls in agent files.
- Docstrings explain *why*, not *what*. The design reasoning matters more than the mechanics.
- Errors are explicit. No bare `except:`. Nothing is silently dropped — if something can't proceed, it gets a status and a reason.