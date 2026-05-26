# Reliability

Most of what this doc is about is *what changed because of what broke*. The features are table stakes.

---

## The three sprints that earned the right to ship features again

After the first big feature push, dogfooding surfaced a class of failures that no amount of further features would have fixed. I gated all new feature work behind a **three-sprint hardening plan**.

Numbers at gate-clear (2026-05-23):

- **80 issues triaged, 39 fixed, all 15 Highs closed.**
- **Cloud Readiness prerequisite met.** Sprint 4 was unblocked — though Sprint 4 itself was then parked behind the commercial-viability gate (see [`roadmap.md`](roadmap.md)).

The tracker has grown since: 108 issues / 65 Fixed / 21 Highs closed as of 2026-05-26, after Study v2 QA, the Curator output pipeline fixes, the eval suite rebuild QA, and the Hermes Bridge QA all closed in the same window.

---

## The five most instructive incidents

These are the kind of failure modes you only learn by running real agents on a real machine. Each one took a class of bug down with it.

### 1. Laptop-killing 30+GB memory crash (Lark import wedge)

A diagnostic helper called `_check_feishu` did an actual `import lark_oapi` just to *test availability*. `lark_oapi` is ~86 MB, ~10,000 modules; importing it compiles bytecode for the whole Lark API. Under memory pressure a 2-second check became a 15-minute hang while the machine swapped, wedging agent init. Compounded by a concurrent dispatcher firing all agents with no cap.

**Fix:** `importlib.util.find_spec()` (no import, ~50ms) plus `JOB_HUNT_MAX_CONCURRENT_AGENTS = 2` with overflow work queued and drained as slots free.

**Lesson:** capability *probes* are not free. Anything that does I/O or compilation to answer "is this available?" needs a non-loading path.

### 2. Wedged agent ran 15 minutes (parent-side timeout, starved event loop)

The only timeout was a parent-process timer; under memory pressure the parent's event loop itself starved, so the kill never fired.

**Fix:** an opt-in in-process daemon-thread watchdog in `cli.py` that reads `HERMES_HARD_TIMEOUT_SEC`, sleeps, and calls `os._exit(124)` — a direct syscall, immune to event-loop starvation. Defaulted to 40 minutes.

**Lesson:** out-of-process supervision is only as healthy as the supervisor. Defence-in-depth means an in-process backstop even when "obviously the parent will kill it."

(Companion upstream PR filed: [NousResearch/hermes-agent#32511](https://github.com/NousResearch/hermes-agent/pull/32511). When merged, Fluid's local patch becomes unnecessary.)

### 3. Recovery cascade: 3 failures became ~40 blocked issues

A `stranded_issue_recovery` task could itself become stranded and spawn its own recovery task — geometric fan-out.

**Fix:** recovery is now non-recursive. Any recovery-origin issue **escalates in place** instead of spawning a child, at all three gate sites. Audited and bulk-cancelled the 40 stale issues.

**Lesson:** any retry/recovery mechanism needs an origin-aware kill switch, not just a count. The cascading failure mode here is what *makes* the original failure visible — three failures producing forty issues is the signal, not the fault.

### 4. Match score mismatch (the same number disagreed with itself)

Three stages each produce a `fitScore`, but `opportunities.match_score` was only updated at *research-stage approval*. So the circle in the UI went stale as deeper stages re-assessed.

**Fix:** the bridge now promotes the most-advanced stage's fitScore (`prepare > research`) to `match_score` on every sync — *"latest agent score wins."* With a lenient regex fallback for prepare outputs whose embedded cover letter breaks strict JSON parsing.

**Lesson:** when multiple agents produce the same datum, write down which one wins. Don't leave it implicit in the update logic.

### 5. Server silently on the wrong (empty) database

`pnpm dev` runs from `server/`, so `resolve(process.cwd(), ".env")` looked in `server/` and missed the repo-root `.env` — and the server fell back to embedded Postgres. Spent an embarrassing amount of time debugging "missing data" that was just the wrong DB.

**Fix:** `findEnvFileUpwards()` walks parent directories. Removed the orphaned embedded instance entirely.

**Lesson:** failure modes that masquerade as feature regressions are the worst kind. Make config resolution explicit and fail loudly when ambiguous.

---

## The Sprint 1–3 backlog (the unglamorous fixes)

These don't have war stories the way the five above do, but the cluster as a whole was the actual gate.

- **Atomic `issueNumber` via a counter helper** (ISS-032 / ISS-054) — replaces every `MAX()+1` inline query, which was the silent race risk on concurrent issue creation.
- **Optimistic-concurrency claim on each automation tick** (ISS-031) — prevents the same automation from firing twice when two ticks race.
- **Per-company FIFO lock on `rebuildProfileFacts`** (ISS-036) — prevents concurrent fact-rebuild races.
- **Profile-leak fix on `agentService.remove`** (Study integration A3) — deleted-agent `~/.hermes/profiles/` directories were leaking.
- **SSRF guard on Study's URL import** (ISS-015) — scheme + IP-range + redirect + size + timeout check before fetching user-supplied URLs.
- **Recovery hygiene** (ISS-050 / 051 / 052 / 034) — handles all four origin kinds and terminates terminated-cap agents instead of looping on them.
- **Per-session study turn FIFO lock + DB `UNIQUE (session_id, turn_index)` constraint** (ISS-007) — prevents concurrent turn writes from creating duplicate-index rows.

Every fix was a small, scoped commit with an issue ID — the backlog itself is the artifact. The Issue_Tracking system tracks it: `Issue_Tracking/issues/ISS-NNN.md` is the source, `ISSUE_LOG.md` and `SESSION_LOG.md` and `known-bugs.json` are auto-regenerated from it by `pnpm bugs:build` (which the pre-commit hook fires).

---

## The rolling hardening backlog (after Sprint 3)

Correctness Mediums on shipped features that don't block cloud but should land before cloud ships to users. These are worked opportunistically, in parallel with feature work:

- **Study engine** — ISS-009 through ISS-014.
- **Job Hunt agent pipeline** — ISS-005, 006, 029, 033, 037–039.
- **Meetings** — ISS-045, 047, 048.
- **Recent Study v2 QA** — ISS-081–087 (filed and all fixed in S-13/14/15); ISS-088/089/091/092/093 (Curator pipeline; all filed and fixed in S-17 except ISS-091 which is disk-fill on the embedded Postgres backup scheduler, no code fix yet).
- **Recent eval suite QA** — ISS-094–100 (all 7 filed and fixed in S-18).
- **Recent Hermes Bridge QA** — ISS-101–108 (all 8 filed and fixed in S-19, the same session that shipped the Bridge).

The pattern: ship → QA pass in the same session → file the issues → fix them in the same session if Low/Medium, file-only if blocked. ISS-101..108 is the cleanest example — 8 Mediums/Lows in the Bridge subsystem, all filed, fixed, and closed in S-19.

---

## The latest reliability layer: Hermes Bridge + Hermes Health

The 2026-05-26 ship. Owns the seam between Fluid and the local Hermes CLI — the dependency Fluid silently degrades against today.

**What it actually does:**

- **Boot-time marker checks** with auto-heal (shells out to the documented `apply_hermes_patches.py`) and fail-loud + documented escape hatch (`FLUID_SKIP_HERMES_CHECK=1`) for the edge cases.
- **Boot-time smoke test** — single attempt, Nemotron `:free`, 1-hour skip window, degrade-don't-block on failure. Subprocess gets `SIGTERM`'d on timeout (it doesn't just leak — ISS-103).
- **Per-call health envelope** — every Hermes call writes a row with `parseConfidence` × `costSource` × `errorClass` × `fallbackHit`. This is the data the **Hermes Health** page renders.
- **Two-tier cost tracking** — verbose-regex first; `tiktoken` fallback for OpenAI-family; non-OpenAI marked `unverified` rather than estimated. Every existing `cost_events` row got tagged with its source tier in migration `0108`, so the Health page can show drift.
- **Five `error_class` values** that tell a story (`timeout`, `crash`, `quota_exhausted`, `model_unavailable`, `parse_failed`, `unknown`) rather than a binary "something broke."

**Why this is in the reliability doc rather than the features doc:** every previous incident in this doc was a class of bug that took a layer of supervision down with it. The Hermes Bridge is the first proactive supervision layer that exists *before* the bug shows up. The Health page is what makes the supervision visible.

The full spec is in `building_documents/product/hermes_bridge.md` in the private repo — sanitized excerpt of the architecture decisions is in [`decisions.md`](decisions.md) and [`roadmap.md`](roadmap.md) under the latest ship.

---

## Posture going forward

- Every ship runs its own QA pass in the same session, files findings, fixes Lows/Mediums in-session, defers blocked items with an ID.
- Every feature gets a reliability layer if it touches a dependency Fluid trusts implicitly (the Bridge being the worked example).
- Every doc gets reconciled against the codebase after the ship — drift between docs and code is a tracked failure mode, not a tolerated one.

This is the posture I'd put on a roadmap slide if you asked me what "shipping AI agent products responsibly" looks like in practice. It's not glamorous. It's the work.
