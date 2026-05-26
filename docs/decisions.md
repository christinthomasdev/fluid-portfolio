# Decisions

Six product decisions defined Fluid. They're not all the decisions I made — they're the ones that, with hindsight, *determined the rest*. A wrong call on any of them and the rest of the work would have been compensating for it.

A note on what's *not* in this doc: the calls about what *not* to build are in [`roadmap.md`](roadmap.md). Saying no in writing is the PM-iest thing in this repo and gets its own page.

---

## How I decide what makes the list

Three axes. Anything that scores on at least one stays on the active list; anything that scores on none gets parked, on the record, with the trigger for un-parking written next to it.

1. **Does it deepen an existing module's viability?** Not "another feature" — does the user *come back tomorrow* because of this? Curator (the v2 fifth Study agent) scored here: without it, Day 0 of a new course is a blank chat. The "Match Score" override loop scored here: it's what made Job Hunt stop feeling generic.
2. **Does it remove a known landmine?** The three reliability sprints scored almost exclusively here. So does the Hermes Bridge — a quiet dependency that degrades silently is a worse problem than a missing feature.
3. **Does it produce a defensible interview moment?** Honest framing: this is a portfolio. The Hermes Health page (substrate status, per-call telemetry, cost-source integrity) was greenlit partly because *"Fluid watches its own substrate and tells you when something's off"* reads convincingly to a non-engineer.

This frame is what kept the scope from sprawling. Every "we should also build…" idea got asked which axis it lived on, and most of them turned out to live on none.

---

## 1. Approval gates instead of full autonomy

The agents are perfectly capable of running an opportunity end-to-end without me. I deliberately broke the pipeline into reviewable artifacts at every stage. The asymmetry is the point: a generic cover letter sent to a company I'd never work for costs me a real foot in a real door; a 30-second review costs almost nothing. Every stage emits a versioned artifact — research dossier, tailored resume bullets, outreach draft, application checklist — and I approve before the next stage starts.

This is what most "autonomous agent" demos get wrong. Autonomy is not the goal. Reviewable, interruptible, reversible work *is*.

The corollary: every artifact needs a UI primitive that makes review take ≤ 30 seconds. That's why each agent's output is rendered as a structured card (not a freeform comment) with copy-to-clipboard, inline edit, and one-line summary at the top. The cost of the review *is* what makes the gate work.

---

## 2. Match-scoring with explicit hard-rule caps

Early Scout outputs were too forgiving — a 78/100 on a role that required 10+ years when I have 6. I rewrote the rubric so the model can't be polite past structural mismatch:

- Deal-breaker keyword present → score caps at **30**
- Below minimum salary floor → caps at **50**
- Missing all "must-include" themes (AI / Growth / Platform) → caps at **40**
- Seniority mismatch → caps at **60**

And the cap must be articulated in the rationale, in one line, ≤ 150 chars. The triage workflow went from "open every JD" to "scan score + one-line cap, decide the borderlines in 90 seconds." This is the unsexy work of making a model's output decision-grade.

Subtler lesson: the rubric isn't just a prompt. It's a *contract* with the UI — the score circle is colour-banded by cap zone, the rationale is the badge under it, and a manual override updates both. The rubric and the UI evolved together, not the rubric first then the UI second.

---

## 3. Roles and boundaries instead of one super-agent

The first version had a single "CEO" agent that handled everything. Given a task to enrich job descriptions from LinkedIn URLs, it tried to do it itself instead of delegating to the browser-capable agent. Adding *organizational* structure — not a bigger model — fixed it: each agent has an explicit scope, an explicit toolset, an explicit list of what it should NOT do, and (for the orchestrator) a delegation rule book. Clear lanes in agents are the same problem as clear lanes in a real team.

A direct consequence: when I added the Study module, "what roles, with what boundaries" was the design question I already knew how to answer. Study landed five agents in a focused week because the role-definition pattern was reusable.

---

## 4. Truth-grounding as a deterministic guardrail

Every concrete claim a Job Hunt writer agent emits — a metric, a scope, an achievement — must trace to a **cite-keyed fact** in a profile fact base extracted from my real resume plus an uploaded long-form career doc. A mechanical verifier walks the output, matches each claim against the fact base, and flags anything that doesn't trace. Deterministic by design: fast, explainable, impossible to fool with confident prose.

Two things I learned the hard way and fixed:

- The bridge originally deduped outputs by `(opportunity, stage, agent)`, which meant a re-run of the same stage silently kept the *older* output. The dedup is now **time-aware**: an output only counts as "this issue's" if its `createdAt >= issue.startedAt`. Spent half a day on a verifier showing 0 of N matches before I found this.
- Honest gaps were being flagged as fabrications. Added a third claim category — **`acknowledged_gap`** — so "I don't have direct fintech experience, but…" is recognized as candor, not a lie. Surface area for the model to be honest matters as much as surface area to catch dishonesty.

The deterministic guardrail is also what makes the eval rubric for Writer tractable: a `fabrication_caught` guardrail dimension is a clean pass/fail, not a judgement call.

---

## 5. Cross-opportunity learning loop

Most AI products are static — each interaction is independent. Fluid captures every score override, rationale rewrite, star rating, and close reason as a timeline event. A pattern aggregator computes lift-based keyword discriminators (*"5-star opps share AI/ML, B2B SaaS, Senior PM"*; *"consulting roles close-rejected ~90% of the time"*; *"RTO-required roles I downgrade to <40"*) and produces a structured user fingerprint that gets injected into Scout's match-scoring prompt and every downstream agent's task prompt.

The *"Patterns Fluid has learned"* view is the strongest demo moment in the product, because it's the moment the agents stop feeling generic — they're reading my own decisions back at me.

This is the decision that converts a personal tool into something demonstrable: the agents *learn from the user without being retrained*. That's a different kind of "AI product" than "GPT wrapper that remembers chat history."

---

## 6. Module abstraction proven by Study

The Study module shipped in a focused week and was the test of whether the platform thesis was real or marketing. Shared with Job Hunt: UI shell, sidebar, Home, agent runtime, profile/fact infrastructure, eval harness, meeting synthesizer, reliability primitives, on-disk Hermes profile management. Module-specific: schema, prompts, personas, pages, FSRS engine, automations.

The Job Hunt-only Home was rewritten into a module-agnostic shell (`useStudyHome` + `useJobHuntHome`, shared activity feed) so neither module is privileged. Study v2 went further — added a fifth "Curator" agent that ingests user material and open-web sources into a per-topic knowledge base, plus a guided on-ramp UI for students starting from zero. The same shell absorbed it cleanly.

The discipline that made this work: every shared primitive had to be refactored *the same week* the second consumer arrived. No "I'll generalize this later" — by the time Study needed it, Home wasn't a Job Hunt page anymore, it was a slot. That's the only way platform claims survive the second module.

---

## What I'd do differently

- **Personas first, not pipeline first.** I built the schema, routes, and UI before deeply thinking about what each agent should actually *be*. The persona design determines everything downstream — prompt structure, tool surface, escalation rules. Doing it last meant later persona changes invalidated earlier work.
- **Evaluation before features.** I had no systematic way to grade agent output for the first half of the build. "It feels good" is not a measurement. The harness should have come on day one — see [`agents.md`](agents.md) for what shipped.
- **Fewer features, deeper quality.** Fluid has Chat, Meetings, Job Hunt, Study, an Electron app, a Chrome extension, a full design system. If I started over I'd build only Job Hunt with exceptional agent quality and ship the second module after. The pull came from the "platform thesis" goal — but the thesis would have been just as well proven from a tighter first module.

These are the framing decisions a working PM would push back on if they were on my team. Naming them is part of the work.
