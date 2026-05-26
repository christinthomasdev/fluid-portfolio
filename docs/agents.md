# Agents

This is the section most "AI agents" projects skip. The agents are the surface area where everything else lives — eval, truth-grounding, supervision — so the persona-level decisions are the product decisions.

---

## How a persona actually gets built

Every agent runs on a dedicated **Hermes profile** on disk at `~/.hermes/profiles/fluid-{role}-{id}/` with its own `SOUL.md`, config, model cache, sessions, and state DB. The `SOUL.md` the agent reads at wake-up is a composition of three layers, written by `job-hunt-persona.ts` or `study-persona.ts`:

1. **Base persona** — role-specific, hardcoded. Defines scope, how-you-work, tools, accuracy rules, tone, and explicit *"what you do NOT do."*
2. **Task-completion block** (Job Hunt only, ~13 lines, appended to every persona). Forces the agent to set a terminal issue status (`done` / `blocked` / `in_review`) as the last action of every run. Added to fix **ISS-003**: agents reliably posted output comments but unreliably set status, so successful runs that left the issue in `in_progress` got classified `successful_run_missing_state` and triggered the recovery cascade. The fix wasn't a code change — it was making the disposition non-optional in the persona.
3. **User context block** — injected from `jobHuntConfig` or the student's course on team setup or profile change. Carries name, current role, years, key skills, target roles, locations, salary range, plus the first 3,000 chars of the resume (capped to keep `SOUL.md` from bloating).

The composition is regenerated whenever the user changes their profile or search criteria — agents never have a stale picture of the person they're working for.

---

## Models — current state and the honest framing

**Job Hunt agents (3) all run on `MiniMax-M2.7` via the `minimax` provider.** Study v2 agents are differentiated: Tutor on Nemotron 3 Super 120B `:free`, Quiz Master on Arcee Trinity Large Thinking `:free`, Study Planner on DeepSeek V4 Flash `:free`, Curator + Note Taker on MiniMax-M2.7. All via OpenRouter free tier where applicable.

Two reasons Job Hunt stayed single-model:

1. **Hermes is a local CLI runtime.** MiniMax-M2.7 is the workable local default. Mixing providers means provisioning credentials per profile and accepting that some agents run cloud, some local — useful eventually, not load-bearing on Job Hunt today.
2. **A single model makes the eval harness apples-to-apples.** When I'm tuning prompts or rubrics, holding the model constant keeps the signal in the change, not in provider differences.

Study v2 differentiated because the agents are doing meaningfully different work — Tutor needs strong instructable conversation; Quiz Master needs strict-format reliability; Planner needs reasoning chains; Note Taker + Curator are bulk content jobs. The 5 split is "right tool for the job" first, "single-model purity" second.

Per-agent model selection on Job Hunt is the kind of thing I'd light up once the eval harness has enough longitudinal data to detect regression vs. swap-driven drift. The infrastructure already supports it — `adapterConfig` JSON carries `{ model, provider }` per agent and the eval harness records the model used on each run.

---

## Job Hunt agents (3)

Three agents cover five pipeline stages. **Scout** owns both ends of the funnel (discover + apply) because both require the browser; **Writer** owns prepare + outreach because both are content generation grounded in the same fact base.

### Scout — opportunity finder and applicant
- **Inputs:** search criteria (target roles, locations, salary range, must-include / deal-breaker keywords), an opportunity URL for back-fill, or an application URL for submission.
- **Outputs:** structured opportunity records (title, company, location, salary, requirements, JD); back-filled JD on stranded entries; submitted-application confirmations.
- **Tools:** `browser_navigate`, `browser_snapshot`, `browser_click`, `browser_type` — Electron's embedded Chromium driven by CDP on port 9223. No headless scraping.
- **Three accuracy mechanisms:**
  1. **Hard-rule score caps in the rubric** (see [`decisions.md`](decisions.md) §2) — model cannot rate past a structural deal-breaker no matter how good the prose.
  2. **Login walls trigger user intervention, not bypass.** Persona explicitly says *"expect login walls — request user intervention"* and *"never bypass security measures."* The browser collapses to a split view so the user takes over for one step, then hands back.
  3. **Match-rationale ≤ 150 chars, one line.** Forces the cap reason to be human-scannable in the triage queue.

### Researcher — company analyst
- **Inputs:** an opportunity record plus the user's profile.
- **Outputs:** a 5-section dossier — Company Overview / Tech & Product / Key People / Culture & Reviews / Fit Assessment — plus a `fitScore` (0–100) and a 4-bucket recommendation (**Strong Fit / Good Fit / Moderate Fit / Weak Fit**).
- **Tools:** `web_search`, `browser_navigate + browser_snapshot` (for Glassdoor, LinkedIn), `web_extract`.
- **Three accuracy mechanisms:**
  1. **Fixed 5-section framework.** Output isn't free-form — the same headers in the same order on every company, so a stale or weak section is visually obvious in review.
  2. **"If you can't find data, say so — don't guess"** as an explicit persona rule. Combined with *"Fit assessment must reference specific user skills/experience."* Forces the model to ground claims in the user's actual profile, not pattern-matched generalities.
  3. **4-bucket recommendation, not a numeric score.** Discrete buckets make the recommendation argument-grade rather than a vibes number. The numeric `fitScore` is for sorting; the bucket is for deciding.

### Writer — career content specialist
- **Inputs:** the opportunity record + Researcher's dossier + the user's resume and fact base.
- **Outputs:** tailored resume bullets, cover letter draft, LinkedIn connect message (≤ 280 chars), follow-up message, email outreach (3–4 sentences), per-stage `fitScore` with breakdown.
- **Tools:** none external — text generation grounded in injected context.
- **Three accuracy mechanisms:**
  1. **"NEVER fabricate experience or skills"** is the first rule, and it's enforced *after the fact* by the truth-grounding verifier that walks every concrete claim against the cite-keyed fact base. The persona sets the intent; the verifier proves it. See [`decisions.md`](decisions.md) §4.
  2. **The `acknowledged_gap` claim category** (added 2026-05-21) lets the agent declare honest gaps without tripping the verifier — so *"no direct fintech experience, but adjacent payments work at Razorpay"* is candor, not fabrication.
  3. **Forbidden-phrase list and hard length caps** (*"NEVER use 'I am writing to express my interest'"*, *"250–400 words max"*, *"LinkedIn: max 280 chars"*). Persona-level templates that catch the obvious AI-prose tells before the human has to read for them.

---

## Study agents (5)

Five agents — four student-facing and one background worker. The split between **synchronous** agents (Tutor, Quiz Master in chat) and **asynchronous** agents (Study Planner for Deep Dive synthesis, Note Taker for card generation, Curator for content build-out) is structural — sync agents prioritise latency, async agents prioritise output discipline.

### Tutor — patient teaching specialist (sync)
- **Inputs:** student's current topic, the topic's curriculum context (v2 — was missing in v1), the student's learning-style preference, conversation history.
- **Outputs:** explanations in chat, follow-up checks for understanding, scaffolded next-step suggestions.
- **Tools:** none — pure conversational reasoning.
- **Three accuracy mechanisms:**
  1. **Explicit "Accuracy — non-negotiable" section** in the persona: *"If you are not confident a fact is correct, say so explicitly. Never present a guess as settled fact."* This is the bit most agent products skip — most personas optimize for helpfulness; this one explicitly *licenses uncertainty*.
  2. **No answer-dumping rule.** *"Do not just give answers to homework or exam questions. Teach the method."* Persona-enforced Socratic constraint, not a runtime check.
  3. **Feynman fallback.** *"If you cannot explain it simply, say so and work it out with the student rather than hiding behind jargon."* Failure mode named in the persona means the agent can announce it instead of silently degrading.

### Quiz Master — retrieval-practice specialist (sync)
- **Inputs:** topic to quiz on, conversation history, optionally a stuck-on-question signal.
- **Outputs:** mixed-format quiz (MCQ, true/false, short-answer) as a strict JSON envelope; short-answer grading verdicts.
- **Tools:** none.
- **Three accuracy mechanisms:**
  1. **Strict JSON envelope with documented schema** — `{ "questions": [ { type, prompt, choices, answerIndex | answer | modelAnswer, rubric, explanation } ] }`. MCQ and true/false are graded *by the system*, not by the model, which forecloses the "model thinks its own answer was right" failure mode.
  2. **"Double-check `answerIndex` before emitting. A wrong answer key destroys trust."** Persona-level imperative because a wrong answer key in a quiz is uniquely corrosive: the student trusts the verdict, gets the question right, and is told they're wrong.
  3. **Hints, not answers.** When the student is stuck, the persona explicitly forbids revealing the answer until the student has committed to one. Same Socratic discipline as Tutor, but enforced at quiz time.

### Study Planner — learning-schedule specialist (async)
- **Inputs:** topic mastery telemetry, exam date, student's available time, and in Deep Dive mode the other agents' independent perspectives.
- **Outputs:** sequenced study plans; Deep Dive synthesis as a strict JSON envelope `{ plan: [ { topic, reviewOn, technique, rationale } ], summary }`.
- **Tools:** none external; reads from `study_topics`, `study_cards`, `study_reviews`, mastery analytics.
- **Three accuracy mechanisms:**
  1. **Principle-grounded planning rules** in the persona: spaced practice, interleaving, work-backward-from-exam, weakest-topics-earliest. The model is told *what learning science to apply*, not asked to invent a plan.
  2. **"Resolve disagreements between agents explicitly in the summary"** for Deep Dive — forces the synthesis to be argued, not averaged. The summary is the artifact the student reads; ambiguity here would be the failure mode.
  3. **Sequenced, not just listed.** Persona explicitly: *"Sequence the plan; do not just list topics."* A bulleted topic dump would technically satisfy the schema; the persona forecloses it.

### Note Taker — study-materials specialist (async)
- **Inputs:** raw content (lecture transcripts, readings, PDFs, study-session outputs).
- **Outputs:** flashcards (basic, cloze, MCQ) and short summaries — strict JSON envelope with per-card `tags` driving topic-level mastery tracking.
- **Tools:** none external; consumes uploaded content via the content-import pipeline.
- **Three accuracy mechanisms:**
  1. **"One idea per card. If a card needs 'and', split it."** Persona enforces the most-violated flashcard rule in spaced repetition — compound cards crater retention, and the model defaults to compound cards unless told otherwise.
  2. **"Front asks for active recall — a real question, not a topic heading."** Persona-level rule that forces the testable form rather than the lecture-note form.
  3. **"If source material is ambiguous, write conservatively or skip."** Licensed skip option means the model doesn't manufacture a confident card from uncertain content — same accuracy-over-coverage trade as Tutor's uncertainty clause.

### Curator — knowledge-base librarian (async, v2)
- **Inputs:** a course topic and the topic tree it sits in; the student's uploaded materials; permitted open-web sources.
- **Outputs:** per-topic knowledge base — learning objectives, 500–2,000 word body, worked examples, source list with trust scores, starter flashcards — as a strict JSON envelope.
- **Tools:** content-import pipeline, web sources (officially-licensed and open-content prioritised).
- **Three accuracy mechanisms:**
  1. **"MACHINE-PARSED, STRICT FORMAT REQUIRED" output discipline** — the most aggressive output spec of any agent, with **worked INCORRECT counter-examples** (*"DO NOT post the topic body as standalone prose before the JSON"*; *"DO NOT post a 'DONE: …' summary"*; *"DO NOT split across comments"*). This wording wasn't preemptive; it was added after the agent broke the contract in real runs and silently failed downstream parsing — ISS-092, four bugs filed in one session.
  2. **Trust-scored sources (0..1).** Every source row carries `trust`: official / publisher 0.9+, OCW / Khan 0.8, Wikipedia 0.6, blog summaries 0.3–0.5. *"Be honest; downstream agents weight by trust."* Forces source-quality into the data model rather than relying on the Curator's pick alone.
  3. **Fair-use rule in the persona, not just at the runtime layer.** *"Summarise and cite copyrighted material; never redistribute it verbatim. User uploads are the only source from which you may store full text."* Legal constraints written into the persona so the model can self-enforce, not just have its output filtered after the fact.

---

## Evaluation harness

Agents are non-deterministic, so the only way to manage them is to test them like a product, not a script.

- **Two modes** — *prompt mode* runs each agent against a fixed prompt to isolate reasoning quality (fast, cheap, deterministic enough to grade); *pipeline mode* runs through the live runtime to catch integration regressions.
- **Graded vs. guardrail dimensions** — graded dimensions score quality on a rubric; guardrail dimensions are pass/fail safety checks. A guardrail failure fails the run regardless of how good the graded score was.
- **N-iteration averaging** — every test case repeats N times to measure consistency, not a lucky sample. A flaky agent that wins one in three is still a bad agent.
- **Per-agent rubrics** — each agent is judged against criteria specific to its role. Scout, Researcher, Writer, Tutor, Quiz Master, Curator are all registered eval stages on the same harness. Scout, Note Taker, Study Planner are honestly marked *"Coverage planned"* on the UI rather than faked.
- **Suite reports with rollups** — pass/fail breakdown, score distributions, per-dimension drilldown. This is the artifact I'd hand to someone reviewing a model swap.

The Eval page itself was rebuilt agent-first in 2026-05-26: 9-card landing grid (6 covered, 3 honestly marked uncovered), per-agent detail page with persona snapshot pulled verbatim from `SOUL.md`, dimension rationales, seed-case panel with side-by-side failure view, per-dimension trend chart. No new schema, no new chart library — reuses what was already there. QA pass closed 7 issues in the same ship.

Building the harness *after* the first features (in hindsight, the wrong order) is one of the things I'd do differently — see [`decisions.md`](decisions.md) "What I'd do differently."
