<div align="center">

# 🚦 Gatekeeper
From raw signal to shipped tool — with humans in the loop where judgment actually matters

### Turning internal complaints into shipped tools — with humans in the loop where judgment actually matters

[![Python](https://img.shields.io/badge/Python-3.11+-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://python.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=for-the-badge&logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com)
[![Next.js](https://img.shields.io/badge/Next.js-000000?style=for-the-badge&logo=next.js&logoColor=white)](https://nextjs.org)
[![SQLite](https://img.shields.io/badge/SQLite-003B57?style=for-the-badge&logo=sqlite&logoColor=white)](https://sqlite.org)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=for-the-badge)](LICENSE)

**Runs entirely on free tiers. No paid API, no cloud bill, no vendor lock-in.**

**🚧 Status: design complete (D1–D12), build in progress.** See the table below for what's actually running vs. what's designed.

</div>

---

## 📖 The story behind this

This began as a take-home design exercise: *design an autonomous system that takes an internal use case from discovery → PRD → mock-up → shipped app, pausing for a human only where judgment genuinely adds value.*

I didn't get the role. But the design was good, and designs that only exist as PDFs are worth nothing — so I'm building it.

---

## 🎯 The problem

Companies don't lack ideas for internal tooling. People describe painful workflows constantly — in meetings, tickets, and offhand complaints.

**What's actually scarce is the translation.** Turning a raw complaint into something precise enough to build, and then building it. That's where the value leaks out.

This system closes that gap — autonomously by default, with a human stepping in at exactly three points.

---

## 🔁 How it works

```
  📥 INGEST          🔍 SCOUT           📊 SCORER          🚦 GATE 1
  QCM notes    →    normalise    →    threshold +   →   "Is this worth
  + form            + cluster          2 scores          our time?"
                                                              ↓
  🚦 GATE 3        🎨 (stubbed)       🚦 GATE 2         ✍️ DEFINER
  "Safe to     ←    designer &   ←   "Did we get   ←    drafts the
   ship?"           builder           it right?"          PRD
```

### The three gates — and why they sit where they do

> **Each gate asks a different question, and the questions get cheaper to answer as the artifact gets more expensive to produce.**

| Gate | Who reviews | Question | Why here specifically |
|:---:|---|---|---|
| **1️⃣** | AI implementation lead | *Is this worth our time?* | **Before** the PRD exists. A polished PRD anchors the reviewer — it makes a mediocre idea look considered |
| **2️⃣** | The person who raised it | *Did we understand it correctly?* | They're the only one who knows their real pain. Cheapest place to catch a misread |
| **3️⃣** | AI implementation lead | *Is it safe to ship?* | Blast radius, not quality. **Defaults to halt** on timeout |

Remove any one and a whole class of error goes uncaught: Gate 1 catches the *wrong* problem, Gate 2 the *misunderstood* problem, Gate 3 the *unsafe deployment*.

---

## 💡 Design decisions worth explaining

<table>
<tr><td width="50%" valign="top">

**🎚️ Threshold is 25% of a role-pool, not a headcount**

"5+ people must complain" silently kills a 4-person team drowning in a task, while a mild annoyance across 200 people clears it easily. Normalising by pool size fixes that. Reports decay with a 90-day half-life.

</td><td width="50%" valign="top">

**📊 Two scores, never collapsed into one**

Impact and sentiment stay separate. A calm complaint about 6 hours/week should outrank a furious one about 10 minutes. Sentiment is self-reported and gameable, so it's cross-checked against reported hours — never a headline number.

</td></tr>
<tr><td width="50%" valign="top">

**🚪 Two entry routes, one gate**

High-signal (threshold met) and low-signal-high-conviction (below threshold, but concrete evidence) both reach a human. **A threshold may deprioritise an idea. It must never silently kill one.**

</td><td width="50%" valign="top">

**🔍 Semantic validation, not just schema validation**

A PRD can be perfectly-formed JSON and still describe the wrong problem. So validation also checks *meaning* — does the stated user group match who reported it, do claimed savings match reported hours.

</td></tr>
</table>

**🤐 The system may be uncertain. It may never hide it.** Every agent emits a confidence score and its unresolved assumptions. Below threshold, it escalates with the *specific question* attached — not "I'm stuck," but "couldn't determine whether this affects the whole team or one squad."

---

## 🛠️ Stack

| Layer | Tech |
|---|---|
| Backend | FastAPI · SQLModel · Pydantic v2 |
| Database | SQLite |
| LLM | Groq (free tier) · Gemini fallback — swappable via one env var |
| Frontend | Next.js 14 · Tailwind |
| Tests | pytest |

---

## 🚀 Quick start

*Target usage once the build lands — see the status table above for what exists today.*

```bash
git clone https://github.com/vichruth/Gatekeeper
cd Gatekeeper

# backend
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env          # add your free Groq key
python -m backend.seed        # load demo complaints
uvicorn backend.main:app --reload

# frontend (separate terminal)
cd frontend && npm install && npm run dev
```

Backend at `localhost:8000` (docs at `/docs`), dashboard at `localhost:3000`.

Get a free Groq key at [console.groq.com](https://console.groq.com) — no card required.

---

## ✅ What's built vs. designed

Being explicit about this, because a README that overclaims is worse than one that's modest.

| | Component | Status |
|:---:|---|---|
| ⬜ | Ingestion (form + meeting notes) | Not started |
| ⬜ | Scout — normalise + semantic clustering | Not started |
| ⬜ | Scorer — threshold, decay, dual scoring, routing | Not started |
| ⬜ | Definer — PRD generation | Not started |
| ⬜ | Schema + semantic validation, bounded retry | Not started |
| ⬜ | All three human gates + append-only decision log | Not started |
| ⬜ | Approval dashboard | Not started |
| 🟡 | Designer / Builder agents | **Planned as stubs** — needs a platform capability this repo doesn't have |
| 🟡 | Rubber-stamp detector | Designed, not built |
| 🟡 | 30-day post-ship review | Designed, not built |
| ❌ | Auth / access control | Known gap — demo will use a fixed reviewer |
| ❌ | Data retention policy | Known gap |

---

## 🧭 Design doc

The full reasoning — all twelve decisions, the failure modes, the seven test scenarios it was stress-tested against (including one it originally **failed**) — lands in `docs/DESIGN.md` as part of the Phase 1 build (see `AGENTS.md`). Not written yet — this repo currently has the design and the build spec, not the code.

---

<div align="center">

**MIT Licensed** · Built by [Vichruth M](https://github.com/vichruth)

*If the design reasoning is useful to you, take it.*

</div>
