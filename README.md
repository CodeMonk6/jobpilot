# Job Scout

**An AI agent that runs your job hunt for you.** Describe what you want; it discovers matching live jobs across dozens of public ATSes and aggregators, ranks them against your profile with a role-knowledge-graph and multi-signal scorer, and tailors your resume per role. Your loop collapses to **describe → review → submit**.

![Next.js](https://img.shields.io/badge/Next.js_14-000000?logo=nextdotjs&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?logo=typescript&logoColor=white)
![Tailwind CSS](https://img.shields.io/badge/Tailwind_CSS-06B6D4?logo=tailwindcss&logoColor=white)
![Supabase](https://img.shields.io/badge/Supabase-3FCF8E?logo=supabase&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/Postgres_+_pgvector-4169E1?logo=postgresql&logoColor=white)
![Google Gemini](https://img.shields.io/badge/Google_Gemini-8E75B2?logo=googlegemini&logoColor=white)
![Turborepo](https://img.shields.io/badge/Turborepo-EF4444?logo=turborepo&logoColor=white)
![Vercel](https://img.shields.io/badge/Vercel-000000?logo=vercel&logoColor=white)

---

## What it does

A serious job search means 100–200 applications, each demanding a tailored resume, a 30-field application form, a separate ATS login (Workday, Greenhouse, Lever, iCIMS each with their own), and a fresh prep cycle when an interview comes back. The grind is so high that the loudest signal in hiring isn't fit — it's who has the time and stamina to apply.

Job Scout collapses that loop so a candidate's signal is their actual experience, not their willingness to fill forms:

1. **Talk to it.** *"Senior AI product manager, Series B startup, remote, $180K+, want to break into healthcare."* An intake agent asks follow-ups when it needs them and hard-caps the conversation so it never spirals.
2. **Upload your resume** (or skip if you're pivoting). The PDF is parsed by a multimodal model into a structured profile — work history, skills, education — and embedded into a vector index for semantic match.
3. **It pulls live jobs** from three source tiers, fanned out in parallel, with the first ranked results landing fast and deeper sweeps available on demand.
4. **You browse a ranked feed** with multi-chip filters (role, location, industry, source, salary floor) that auto-populate from your intake. Each row shows a match score, source, posted date, and a one-sentence "why this matches."
5. **One click → a tailored resume** that reframes your existing experience to mirror the job — never inventing new history — downloadable as a clean one-page PDF.

### Three non-negotiables

| Rule | Why |
|---|---|
| **Never fabricate resume content** | A post-generation validator compares output against the source profile; a new employer, title, date, or degree fails the check and the result is rejected. Tailoring means reframing truthful content, never inventing it. |
| **Submit to the original ATS, not a proxy** | Applications are the candidate's legal representations; they must land where the employer expects them. |
| **The user clicks the final Submit, always** | A legal safeguard and a manual sanity check that is never automated. |

---

## How it works

A central **Coordinator** routes work to specialized agents, each with a narrow domain and strict typed I/O (schema-validated in and out):

```
                    ┌─ Intake Agent      understand candidate goals
                    ├─ Resume Tailor     per-job rewriting + anti-fabrication
   Coordinator ─────┼─ Search Agent      fetch + dedupe + rank live jobs
                    ├─ Apply Agent       auto-fill ATS via headless browser
                    └─ Interview Coach   voice-mode mock interviews
```

Each agent is independently testable with mocked model responses, fails predictably, and uses the right model for its task — a cheap, fast model for classification, a stronger one for tailoring.

### Where the live jobs come from

The Search Agent fans out across three tiers, parallelized and lazily deepened so the first results arrive quickly:

- **Tier 1 — global aggregators.** Server-side keyword filtering across major public job APIs. Fast; this is what the initial search hits.
- **Tier 2 — direct company ATS APIs.** Structured JSON pulled straight from dozens of curated company instances on the leading ATS platforms.
- **Tier 3 — HTML scrapers.** The slow, failure-prone long tail of career pages that nonetheless surface unique listings.

The user clicks **Load more** to deepen from Tier 1 into Tier 2 and then Tier 3 — controlling how far the search reaches instead of waiting on slow sources by default.

### How relevance actually works

Matching runs as a three-layer pipeline on every search:

**Layer 1 — Intent extraction.** A typed query is parsed into a structured intent (titles, seniority, industries, locations, work mode, salary). A resume-only search instead distills the resume into a small role family plus a "core profile essence" used as the query.

**Layer 2 — Role-graph expansion.** The intent passes through a static role knowledge graph — **21 role families across ~320 canonical titles**, each with a career ladder and adjacent specializations. Typing "Marketing Manager" implicitly searches Brand Manager, Growth Marketing Manager, Marketing Director, VP Marketing, and more, so jobs in the family rank together regardless of literal string overlap. This step is deterministic, free, and runs without any model call at search time.

**Layer 3 — Multi-signal scoring.** Candidate jobs that survive dedup and hard filters are scored as a weighted blend of vector cosine similarity, a structured signal (title, industry, location overlap, with a freshness bump only when there is already a positive signal), and an enrichment signal (role-family match, seniority, industry, salary floor, years required, visa policy). Embeddings are computed only for the top candidates by structured score, saving the large majority of embedding cost. The finalists then run through a single rerank pass that reorders by genuine fit and writes the per-result "why this matches" line.

---

## Highlights

- **Role knowledge graph** — a hand-built taxonomy of 21 families and ~320 titles that turns a single job title into its whole career neighborhood, making matching intelligent without an expensive model call on the hot path.
- **Anti-fabrication validator** — tailored resumes are programmatically checked against the source profile; anything that invents new history is rejected before it reaches the user.
- **Tiered, user-controlled fanout** — fast first results from filtering aggregators, with progressively deeper (and slower) source tiers gated behind explicit "Load more" so the user decides the latency/coverage trade-off.
- **Cost-aware model routing** — classification and parsing run on a cheap fast model; only tailoring and heavy reasoning escalate, with embeddings restricted to top candidates.
- **Strict typed agent boundaries** — every agent speaks schema-validated I/O, so failures are predictable and each agent is unit-testable with mocked model responses.
- **Production data model** — Postgres with vector embeddings, row-level security scoped per user, and auto-provisioning on signup; schema and migrations are version-controlled.
- **One-page PDF + spreadsheet export** — tailored resumes stream as a clean LETTER-format PDF, and filtered result sets export to a workbook.

---

## Tech stack

| Layer | Tech |
|---|---|
| **Frontend** | Next.js 14 (App Router, server components, strict TypeScript), Tailwind CSS, shadcn/ui |
| **Auth + database** | Supabase Postgres with `pgvector` for semantic embeddings; row-level security scoped to the authenticated user; auto-provisioning trigger on signup |
| **LLM** | Google Gemini family — a fast model for intake, parsing, ranking, and tailoring, with a stronger model available behind a feature flag for heavy reasoning |
| **Embeddings** | Gemini embeddings indexed in pgvector for query-to-job semantic match |
| **Job sources** | A pluggable adapter layer spanning filtering aggregators, direct ATS APIs, and HTML scrapers across three tiers |
| **Documents** | React-based PDF rendering for one-page resumes; workbook export for filtered results |
| **Automation (roadmap)** | Headless browser + Computer Use for ATS auto-fill, with stealth browsing, CAPTCHA solving, and encrypted per-site credential management |
| **Monorepo** | Turborepo + pnpm, packaged into clearly owned workspaces (agents, sources, ATS adapters, db, shared schemas, UI) |
| **Infra** | Deployed on Vercel; CI runs typecheck, lint, and tests across the workspace |

### The hard problem: ATS submission

Workday, iCIMS, Taleo and others actively block bots, so straight headless automation fails. The roadmap tackles this in layers: stealth browsing with residential proxies and randomized fingerprints; CAPTCHA solving; per-site email aliases with encrypted password storage and programmatic email-confirmation polling; and a human-handoff fallback that surfaces the live browser session when a blocker (typically phone/SMS verification) needs a real person. The final Submit is always the user's click.

---

## Status

Phases 1–4 are **live**: conversational intake, resume parsing, the tiered live-search engine with role-graph ranking and rerank, per-job resume tailoring with PDF download, spreadsheet export, and an applications history view. Active work targets the ATS auto-fill agent and a voice-mode interview coach. Each phase ships as a discrete, deployable unit reviewed at its boundary.
