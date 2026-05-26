# Roadmap

The most PM-relevant doc in this repo. What's shipped, what's shipping now, **what's parked and why**, what's queued. The parked list is the point — anyone can talk about what they shipped; saying no in writing is harder.

This is a sanitized excerpt of the internal `building_documents/roadmap/README.md`, which lives next to the code and is reconciled against the codebase after every ship.

---

## Shipped

### Job Hunt (full redesign — 2026-05-18)

Replaces the ported v1 with a Paperclip-native, keyboard-first workflow.

- **3 agents, 5 pipeline stages** — Scout (discover + apply) → Researcher → Writer (prepare + outreach) → Apply. Approval-gated at every stage.
- **Hard-rule score caps** — Scout's match rubric refuses to score past structural mismatch. See [`decisions.md`](decisions.md) §2.
- **Triage UI** — bulk pursue/skip; per-card quick-actions; floating compare-2-to-4 bar; saved views (localStorage); funnel analytics.
- **Detail page** — structured renderers per agent stage (research dossier, resume bullets with per-bullet copy, cover letter with inline edit/revert, interactive apply checklist); live elapsed timer when an agent is working; inline match-score and rationale override.
- **Interview tracker** — schedule + prep + debrief + outcome + sentiment + `.ics` export.
- **Recurring searches** — daily / weekdays / weekly / custom cadence cron.
- **Gmail draft integration** — outreach email, follow-up email, both pre-filled with opp-aware context.
- **A11y baseline** — every card is keyboard-activatable; MatchScore has descriptive aria-label; notes status is `aria-live="polite"`.

### Study Module v2 (2026-05-23)

v1 shipped the execution shell (4 agents, FSRS, sync/async paths) on top of an empty curriculum layer. v2 rebuilt the content layer, added a fifth agent (Curator), and made Day 0 actually usable.

- **5-agent team** — Tutor (sync) · Quiz Master (sync) · Study Planner (async) · Note Taker (async) · Curator (async, v2).
- **Curriculum-aware prompts** — Tutor and Quiz Master see the topic tree, Curator-written content, adjacent-topic mastery, and the top-3 cited sources per topic. Day 0 isn't a blank chat anymore.
- **Per-agent models** — Tutor: Nemotron 3 Super 120B `:free`; Quiz Master: Arcee Trinity Large Thinking `:free`; Study Planner: DeepSeek V4 Flash `:free`; Curator + Note Taker: MiniMax-M2.7. All via OpenRouter free tier.
- **FSRS-6 spaced repetition** with per-student weight optimization (Python subprocess driven by a study automation).
- **Sync sessions over an awaited Hermes turn** through a concurrency semaphore; async content import and Meeting-2.0-backed Deep Dive through the heartbeat pipeline.
- **Guided on-ramp** — "Start here" banner on a fresh course, recommended-next-topic ranker (exam-weight × inverse-mastery × prereq-gated), Today's plan card grid, "Refresh this topic" CTA per leaf.
- **GMAT Focus Edition** flagship course — full Quant depth, Verbal + DI shallower at v1.

### Hermes Bridge + Hermes Health (2026-05-26 — most recent ship)

The "Fluid watches its own substrate" subsystem. Owns the seam between Fluid and the local Hermes CLI — the dependency Fluid silently degrades against today.

- **Boot-time substrate check** — marker-based verification of out-of-repo Hermes patches (the `cli.py` wall-clock-timeout watchdog and the `~/.hermes/config.yaml` fallback config) with auto-heal on absence, fail-loud with documented escape hatch on unreachable.
- **One-shot smoke test** — Nemotron `:free`, "reply with the single word: ok", 1-hour skip window, degrade-don't-block on failure. Subprocess gets `SIGTERM`'d on timeout (it doesn't just leak).
- **Per-call telemetry** — every Hermes call writes one row to `hermes_health_events` tagged with `parseConfidence` (verbose-regex hit quality), `costSource` (which tier produced the token counts), `errorClass` (timeout / crash / quota / model-unavailable / parse-failed / unknown), and `fallbackHit`.
- **Two-tier cost tracking** — verbose-regex first; `tiktoken` fallback for OpenAI-family models; non-OpenAI marked `unverified` rather than estimated dishonestly.
- **Hermes Health page** — top-level sidebar entry next to Study. Renders substrate status, agent runtime table (calls / avg latency / fallback count / errors per agent), cost-source integrity stacked bar (with drift threshold called out inline), recent incidents.

QA pass filed and fixed 8 issues in the same ship session — observe()-never-throws guarantee made real, fallback false-positives on unknown models eliminated, smoke-test subprocess no longer leaked, classifyError no longer drops `exitCode: null` + non-empty `errorMessage` calls.

### Reliability hardening (Sprints 1–3 cleared 2026-05-23)

Three sprints, gated before any new feature work. Detail in [`reliability.md`](reliability.md). Tracker numbers at gate clear: 80 issues triaged, 39 fixed, **all 15 High-severity defects closed**. Tracker now at 108 issues / 65 Fixed / 21 Highs closed (Study v2 QA, Curator pipeline fixes, eval suite rebuild, and Hermes Bridge QA all closed in the same window).

### Eval suite redesign (2026-05-26)

Agent-first landing grid (9 cards: 6 covered, 3 honestly marked "Coverage planned"). Per-agent detail page with persona snapshot, dimension rationales, seed-case panel with side-by-side failure view, per-dimension trend chart. Reuses existing `eval_scores.rationale` storage and the existing persona endpoint — no schema changes, no new chart library.

### Quietly also shipped

- **Truth-grounding verifier** — see [`decisions.md`](decisions.md) §4.
- **Cross-opportunity pattern aggregator** — see [`decisions.md`](decisions.md) §5.
- **Career-context document upload** — optional longform PDF/md/txt extracted into the profile fact base (migrations 0100–0102); resume tailoring as a write-back action.
- **Agent-controlled browser panel** — Electron `WebContentsView` driven by CDP on port 9223; collapsible split-view for user takeover on login walls or attention alerts.
- **Module-agnostic Home** — `useStudyHome` + `useJobHuntHome` hooks, shared activity feed, agent status dots, due-card badge.
- **Issue tracker** — 108 file-based issues with frontmatter (severity / status / area / sessionLogged / sessionFixed), 19 sessions, auto-regenerated `ISSUE_LOG.md` / `SESSION_LOG.md` / `known-bugs.json` via a pre-commit hook. The in-app **Known Bugs** tab serves this directly.
- **Branding** — full Paperclip → Fluid rebrand (icons, brand assets); Electron desktop app shipped; Chrome extension for capturing job listings.

---

## Shipping now

### Developer mode (in progress)

A single `localStorage["fluid.devMode"]` toggle in Settings that adds a collapsible Developer section to the sidebar listing every hidden Paperclip operator route (Issues, Routines, Goals, Agents list, Companies, Approvals, Costs, Activity, etc.). No DB change, no backend, no rebrand — routes already work; this just unhides them in nav for one user (me).

This resolves the parked "what to do about hidden operator pages" question with the lowest-effort answer that doesn't lie about scope. The other answer ("rebrand and elevate") is parked until commercial viability — see below.

### The commercial-viability gate

Sprint 4 polish and Cloud Readiness are **parked indefinitely** until Fluid demonstrates commercial viability for *something* — Study v2, Job Hunt, or a future module.

The gate is qualitative: *"I'll know it when I see it."* Paying users, strong unprompted pull from a friend/colleague, or similar. Until that signal exists, engineering-readiness isn't the bottleneck — market signal is.

What stays active under the gate:

- The **rolling hardening backlog** (correctness Mediums across Study, Job Hunt, Meetings) — bug fixes on shipped features, not polish.
- Anything that **directly tests commercial viability** — manual product validation, qualitative user feedback, sharpening existing surfaces to be demo-worthy.
- **Developer-only tooling** that speeds up the above (e.g. Developer mode above; Hermes Bridge + Health) — not user-facing, doesn't violate the gate, just makes QA / debugging / agent config faster.

This is the call I'd push back on the hardest if I were on the team — but I'd also expect the PM to have written it down. It's here.

---

## Parked (with reasons)

The decisions about what *not* to build, on the record, with the trigger to un-park written next to each.

### Cloud Readiness — engineering gate met, product gate not met
**Status:** all the prerequisites shipped (Sprint 3 closed authz, SSRF, race fixes; `better-auth` exists; `Dockerfile` + `docker/` compose variants present from Paperclip upstream).
**Why parked:** no demand signal. Building cloud agent execution + managed DB + multi-tenancy UI before there's a user who wants those things would be wrong-order work.
**Trigger to un-park:** the commercial-viability gate above.

### Sprint 4 polish — six items
**Status:** scoped and triaged.
**Why parked:** each is a real defect against shipped scope, but none of them is what's stopping users from coming back. They're polish on a product nobody's yet pulled at hard enough to need polish.
**The items:**
1. **Fluid-native onboarding wizard** — first-run still reuses Paperclip's "Create a Company" framing. Replace with a Fluid framing (upload resume → set preferences → auto-provision agents) so the user never sees "company."
2. **Resolve the hidden operator pages** — ~40 routes are hidden from the sidebar but resolve by direct URL. Decide: gate behind a power-user toggle, move under Settings, or remove. (Developer mode above is the partial answer; the full call is parked.)
3. **Cloud-side saved searches** — `SavedSearches.tsx` persists to `localStorage` only; needs a table + API for cross-device persistence.
4. **Per-location loops in Scout searches** — Scout currently builds portal URLs from `locations[0]` only.
5. **`Stepper` reusable primitive** — `TeamSetupWizard` hand-rolls its step indicator; extract to a generic component.
6. **macOS code signing + notarization** — `@electron-forge/maker-dmg` is configured but `osxSign` and notarization aren't; release CI runs on `ubuntu-latest`. Needs an Apple Developer identity, notarization creds, and a macOS runner.

**Trigger to un-park:** commercial-viability gate, *and* the specific item moves from "polish" to "this is what's blocking the next user."

### Multi-adapter unlock — 10 of 11 adapters intentionally unreachable
**Status:** all 11 adapters (`claude_local`, `codex_local`, `cursor`, `gemini_local`, …) are wired in the registry; agent creation hard-forces `hermes_local`.
**Why parked:** holding the model layer constant keeps the eval-harness signal in the prompt change, not in provider differences. Per-agent model selection is the kind of thing to light up *once* the eval harness has enough longitudinal data to detect regression vs. swap-driven drift.
**Trigger to un-park:** eval baselines stable across at least 5 nightly suite runs per agent.

### Multi-tenancy UI — infra exists, no consumer
**Status:** `companyMemberships` table exists; better-auth wired; multi-tenant data model already in place from Paperclip upstream.
**Why parked:** local-trusted single-user is the actual deployment mode today. Exposing a workspace switcher when there's nothing to switch to is a worse UX than not having it.
**Trigger to un-park:** Cloud Readiness un-parks, *and* there are users to put in separate tenants.

### Persona versioning / A-B prompt tooling
**Status:** spec'd at a high level; not designed in detail.
**Why parked:** no observed prompt regression in production to motivate it. The eval harness already records the prompt version used on each run, which is the substrate this would build on.
**Trigger to un-park:** an eval-detected regression that's hard to root-cause without a versioned prompt history.

### Notifications (email, push) / Billing / Analytics
**Status:** no work.
**Why parked:** all three are commercial-cloud concerns that pre-suppose cloud-side deployment + paying users. None of those things exist yet.
**Trigger to un-park:** Cloud Readiness un-parks first.

---

## Future (planned, pending prerequisites)

Things that are clearly the right next step *after* the gates above clear.

- **Onboarding wizard rewrite** — Fluid-native framing (overlaps Sprint 4 item #1 above; will land together once un-parked).
- **Cloud agent execution** — containerized Hermes, managed Postgres (RDS / Supabase / Neon).
- **Persona governance** — versioning, A/B routing, eval-driven prompt selection.
- **More tokenizers** — Hermes Bridge ships v1 with `tiktoken` only; OpenAI-family models get exact estimates, everything else gets `unverified`. Multi-tokenizer support is a v1.1 if the `unverified` slice on the Health page grows above 10%.
- **Scheduled nightly eval suite runs + regression alerting** — the harness records the data; the alerting layer isn't built.
- **New rubrics for the 3 "Coverage planned" agents** (Scout, Note Taker, Study Planner) — currently honestly marked uncovered on the Eval page rather than faked.

---

## How this roadmap stays honest

After every ship I run a doc-sync agent that diffs `building_documents/` against the codebase — Current State, technical refs, architecture diagrams, this roadmap. Drift between docs and code is itself a tracked failure mode. The internal version of this file has been audit-reconciled five times in the last six weeks.

The `Issue_Tracking/` system is the other half: 108 issues, 19 sessions, every fix lands with an ID, the rollup files are auto-generated by `pnpm bugs:build` and the pre-commit hook keeps them in sync. *"I shipped X"* is cheap; *"I shipped X, here's the bug it created, here's the session that closed it, here's the doc that got updated as a result"* is the loop.
