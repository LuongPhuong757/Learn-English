# Project Research Summary

**Project:** LearnEnglish
**Domain:** Personal scaffolded-AI-conversation language learning web app (beginner English, Vietnamese L1, voice + text, habit-first, shared owner-funded API key, ~5 users)
**Researched:** 2026-10-06
**Confidence:** MEDIUM-HIGH overall (see Confidence Assessment — the provenance caveat is real and stated plainly)

## Executive Summary

This is a **habit product with a pedagogy engine inside it**, not a chat app. Four independent research lines converge on one shape: a single always-on server process that owns the provider key, a *single* conversation turn engine whose input is `{text, source: 'typed'|'spoken', confidence?}`, a scaffold that **is the input mechanism** rather than a hint panel beside an empty box, and an append-only event log from which every streak, counter and review priority is derived. Voice, review scheduling, progress UI and deployment are all adapters or readers over that core. The research is unusually unanimous on the load-bearing parts; the disagreements are at the edges and are resolved below.

The recommended approach: **Postgres + Drizzle on both sides, one long-lived Node process, provider calls server-side only behind one budget choke point, text mode first, one hand-authored scenario, no SRS algorithm, no realtime speech API, no pronunciation scoring.** Error capture runs from day one but produces no learner-visible review until a capped watchlist exists. The speech layer ships as a port in v1 with a server-side STT provider as its default implementation; which vendor is a measured decision, not a researched one.

The three risks that actually kill this, in order. **(1) The scaffold curriculum never gets written.** PITFALLS names this the #1 under-estimated item: the AI does *not* generate the lessons — deciding which structures, in what order, with which ~800 words, across which scenarios is content work, and it is the entire product thesis. **(2) Voice mode eats the schedule** (5–10x the apparent size: iOS Safari, permissions, formats, autoplay, push-to-talk UX), and the typical outcome is an unscaffolded chat shipped in week 5 that drifts to B1 by turn 6 and loses the learner on day three — the Duolingo failure reached by a more expensive route. **(3) Speech recognition falsely blames the learner.** Vietnamese-L1 speakers score mean MER **0.143** (male 0.181) vs **0.007** for US English on L2-ARCTIC — roughly 20x — and those are *fluent* L2 speakers reading prepared sentences; a "mất gốc" beginner should be planned for at 25–40% WER. The mitigation is structural, not incremental: STT never judges correctness, the streak credits before STT returns, and the transcript is always shown and correctable before the AI responds.

---

## Resolved Conflicts

Four genuine contradictions existed across the research set. Three are settled here. The fourth belongs to the project owner and is deliberately left open.

### CONFLICT 1 — Database: SQLite vs Postgres -> **Postgres on both sides. RESOLVED.**

**Decision:** Postgres 16 locally (docker-compose, or Postgres.app / `brew services` if a daemon-free Mac is preferred) and managed Postgres in production, behind a single `DATABASE_URL`. Drizzle ORM with checked-in SQL migrations. `user_id NOT NULL` on every user-scoped table from migration 001.

**Rationale.** The two documents are not actually opposed on the thing STACK cared most about: *both reject serverless*, and both land on a long-lived process on a container host (Fly/Railway/Render/VPS). Once that is agreed, SQLite's strongest argument — "serverless would kill a file-backed DB" — is an argument against a host nobody is proposing, and what remains is a narrower trade. On that narrower trade Postgres wins on two concrete points.

First, **the `jsonb` need is real, and it is not one column.** `frame_offered`, `scenarios.seed_frames`, and the turn protocol's `scaffold_options` payload are all natively JSON-shaped, and ARCHITECTURE's own §6 is explicit that choosing SQLite means committing to a portable column subset with *no* jsonb and no arrays — a discipline that one careless column silently breaks, converting "deployable without re-architecture" back into a dialect migration under pressure. The budget choke point also wants an atomic read-modify-write returning a tier, and the `review_items (user_id, due_at) WHERE status='active'` partial index is exactly the kind of thing that gets written without noticing it isn't portable.

Second, **"copy the file to deploy" was never the real migration path.** The moment a friend uses the deployed URL, the Mac database and the production database diverge permanently; the deploy step is a schema migration either way, and with Postgres it is literally an env var.

**What the rejected option costs.** SQLite would have given zero-daemon local dev, trivial file-copy backups, synchronous sub-millisecond reads, and $0 of hosted database spend. Giving it up costs a one-time ~15-minute local Postgres setup, a daemon the owner must remember to start, and $0–5/month in production (free tiers cover 5 users comfortably). Everything else STACK argued for survives unchanged — Drizzle, checked-in migrations, per-user keys from day one, and the hard prohibition on serverless hosts. The escape hatch now runs the other way and is cheaper: if local Postgres ever becomes friction, Drizzle makes the *local* driver swappable (PGlite) without touching production.

### CONFLICT 2 — Speech-to-text provider -> **Server-side STT is the v1 default; vendor deferred to a measured spike. RESOLVED (on the right axis).**

**Decision.** The architectural question — *where does recognition run* — is settled now: **server-side, behind `lib/speech`'s port, with `expected_target` passed as a decoding hint.** Browser Web Speech is demoted to a dev-convenience / degraded fallback implementation of the same port, never the default and never a source of error-capture input. The *vendor* question (Deepgram Nova-3 with keyterm prompting vs OpenAI `gpt-4o-transcribe` with `prompt`) is explicitly deferred to the measurement spike below, and the voice phase must not be planned before that spike runs.

**Rationale — and why cost is the wrong axis.** ARCHITECTURE proposed "the cost table in §7 is the tiebreaker." That tiebreaker answers a question STACK never asked. STACK's objection was vocabulary biasing and browser support. On the biasing point, **PITFALLS independently reaches the same conclusion from a different direction**: because the turn is scaffolded, you already know the target utterance, so passing it to the recogniser as a hint and scoring by generous normalised similarity is "the single biggest accuracy win available." Two documents, researched separately, name recogniser biasing toward the known expected target as the decisive mechanism — and **the browser Web Speech API structurally cannot do it.** There is no hint, prompt, keyterm or vocabulary parameter; Chrome ships the audio to Google's recogniser and returns a string. That is not a cost difference, it is a capability the architecture depends on. Add that Web Speech has no Firefox or Edge support (~88% global coverage), that Safari's implementation "still uses an older model" per ARCHITECTURE's own §8, and that it produces no server-side audio artifact — and the browser path cannot be the default for the one user who matters.

**On cost specifically, ARCHITECTURE's table measured the wrong unit.** Its $0.25–0.45/session figure for server STT assumed ~$0.017/min across a 10-minute *session*. Under push-to-talk — which all four documents independently require for a beginner who pauses 2–4 seconds mid-sentence — you bill only captured speech, which STACK measures at ~3 min/session. At Deepgram batch rates that is ~$0.014/session, or about **$2/month for all five users**. The cost gap ARCHITECTURE was weighing largely evaporates once VAD is off the table.

**What the rejected option costs.** Rejecting Web-Speech-as-default costs ~$2/month, ~300–600 ms of round trip on a complete clip (invisible beside the LLM turn — and PITFALLS §9 is explicit that a beginner does not need the 200 ms human-turn-taking budget; a deliberate ~1 s pause reads as the app thinking), and the `MediaRecorder` format-detection work the browser path avoids (webm/opus vs mp4/aac on iOS 14.5–18.3). All three are bounded and known.

**Why the vendor stays open.** ARCHITECTURE §13 calls WER on Vietnamese-accented beginner English "the single biggest open question," STACK's Gaps section says no benchmark exists for Vietnamese-accented English specifically, and PITFALLS' 20x figure is for fluent readers, not beginners. Nobody has the number. Deepgram's keyterm advantage rests on a vendor case study; OpenAI's `prompt` is weaker biasing but one fewer vendor. Choosing between them from the literature would be guessing with a citation attached. The port makes it an env var; the spike makes it a measurement.

**Reconciled with the correctness rule (unanimous, non-negotiable).** STT output is *never* a verdict. Streak credit is granted server-side the moment audio of sufficient duration is captured, **before STT returns and regardless of what it returns**. The transcript is always rendered with a one-tap "that's not what I said" that discards the utterance from error capture, offers a text fallback, and is itself logged — above ~15% tap rate, voice mode is not ready. No unconfirmed transcript ever becomes a review item. Similarity thresholds are asymmetric: fast to say "that worked," extremely slow to say "that wasn't right"; when ambiguous, say nothing about accuracy and move the conversation on. No phoneme-level pronunciation score ever, in any version.

### CONFLICT 3 — Spaced repetition: FSRS vs no algorithm -> **No FSRS in v1. PITFALLS is right. RESOLVED.**

**Decision.** Ship a **capped ~20-item priority watchlist** with a transparent score (`recency x recurrence x failure_count`), no due dates, no debt, and no "N due" count anywhere in the UI. New errors displace the lowest-priority item when the cap is full. Review happens *inside* the conversation by biasing scenario and frame selection so the target structure is elicited naturally — never as a deck screen. **Keep all four of FEATURES' adaptations, which are what actually made its proposal good and which are algorithm-independent:** normalisation of error instances to a closed `structure_key` taxonomy, the maturation gate (>=2–3 occurrences across >=2 sessions before an item exists), three grades only, and conversational elicitation with "not elicited != forgot." Auto-retire aggressively: 3 clean productions -> retire; 5 failures -> retire anyway and surface it in the Vietnamese recap as needing a human.

**Rationale, and why the proposed synthesis is not coherent.** "FSRS schedules which items enter a capped watchlist without exposing due dates" is mechanically possible — you would use `due_at` as a sort key — but it is a compromise that inherits the costs of both sides and banks none of FSRS's benefit. FSRS's measured 700M-review advantage (log-loss 0.291 vs 0.354) comes from *self-reported recall of atomic flashcards over many reviews*. Every precondition fails here: grades come from an LLM assessor's judgment of production in conversation (noisy — and FEATURES concedes parameters can never be optimised at ~50 items x 5 users); errors are not atomic ("I go school yesterday" is three overlapping issues); and elicitation may not occur at all. More decisively, **FSRS does not fix the four problems PITFALLS actually raised** — unbounded auto-generated items, STT-poisoned items, non-atomicity, and the absence of a clean prompt/answer pair. Those are fixed by the taxonomy, the maturation gate and the cap, all of which are FEATURES' own contributions and none of which need interval math. What FSRS adds on top is a state machine (New/Learning/Review/Relearning), a lapse/ease model whose failure mode PITFALLS documents as "ease hell," a dependency, and a stability/difficulty pair you cannot validate at this scale — to produce a sort order over twenty rows.

**What the rejected option costs — and the one thing to take from it anyway.** Dropping FSRS forfeits the benchmarked "15–40% fewer reviews at equal retention." At a cap of 1–3 elicitations per session that efficiency is worth approximately zero sessions. The genuine loss is optionality, and ARCHITECTURE identifies exactly how to buy it back for one INSERT: **append a `review_logs`-shaped record from day one** (item, outcome, timestamp, elapsed since last, what was scheduled). That data is unbackfillable, and with it FSRS can be adopted later from real history if the simple scorer visibly mis-prioritises. Both sides agree the scheduler sits behind an interface — carry that forward verbatim. The interface is the insurance; the algorithm is not needed yet.

### CONFLICT 4 — Voice mode in v1 -> **NOT RESOLVED HERE. Owner's decision.**

PROJECT.md lists "Learner chooses voice or text mode per session; both reach the same core loop" as an **Active requirement**, chosen explicitly during requirements gathering. Two research documents argue against it for v1. This is a scope decision, not a technical one, and the owner is being asked directly. The evidence, crisply:

**For demoting voice to v1.x (FEATURES, PITFALLS):**
- Satar & Özdener and replications: novice learners' speaking proficiency improved in *both* text-chat and voice-chat conditions, but anxiety decreased **only** in the text group. Text is not the lesser mode.
- The Core Value's own wording is "spoken **or typed**" — text mode alone satisfies it.
- "Add a mic button" is a **5–10x underestimate**: getUserMedia, HTTPS/secure-context, iOS autoplay blocking TTS without a gesture, format detection, mobile backgrounding killing streams, permission-denied recovery, push-to-talk UX. PITFALLS names voice the most likely cause of project death — the three weeks it eats are the three weeks the scaffold curriculum doesn't get written.
- STT error rates are worst exactly where the learner is weakest (25–40% projected WER).
- PITFALLS goes furthest: Phase 1 text-only, one scenario, no voice, and a **7-consecutive-day owner self-use gate** before anything else is built.

**For keeping voice in v1 (PROJECT.md, STACK):**
- The destination is *spoken* communication; text-only risks optimising for a proxy.
- The learner explicitly chose per-session modality because the choice is situational (surroundings, time of day) — removing it removes a named reason the loop is returnable-to.
- STACK's pipeline (STT -> LLM -> TTS behind ports) makes voice additive rather than a second engine, and prices the whole stack at ~$3.50/month.
- A beginner has no internal pronunciation model to correct against; model audio on every frame is itself the listening lesson.

**What holds either way — build this regardless of the decision.** All four documents agree on the boundary: the turn engine's input is `{text, source: 'typed'|'spoken', confidence?}` and `turns.text` is the only thing the rest of the system ever sees. `lib/speech` exposes `listen()` / `speak()` with swappable implementations. `turns.audio_url` (nullable), `users.retain_audio` (default false) and a `BlobStore` interface exist from day one even though nothing is stored. The scaffold protocol carries `expected_target` from the first turn ever written, because that field is what makes STT biasing possible later. **If the owner keeps voice in v1, it still ships after the text loop is proven, and it still waits on the STT spike.** The ordering below is unchanged by the outcome; only the phase's position moves.

---

## Points of Unanimous Agreement (highest-confidence content in the research set — do not lose these)

1. **Every provider call originates from the server process.** No key in any `NEXT_PUBLIC_*` / `VITE_*` var, no client-side SDK call, `.gitignore` + secret scanner before commit #1, and a hard spend cap at the provider account level as the backstop that survives a bug in your own limiter.
2. **Exactly one budget choke point, checked *before* the first model call ever ships.** Retrofit cost scales with call-site count and you will miss one. The unit is money (`cost_micros`), not requests. It **downgrades, never 429s** — a blocked day is a broken streak is an abandoned app.
3. **The partner call is separate from the assessor call.** Separate prompts; the assessor gets `{learner_utterance, expected_target, scenario_context}` only, no persona, strict JSON out. Warmth is applied to the *delivery* of the verdict, never to its content.
4. **Per-user data scoping from migration 001**, enforced at the data-access layer rather than in route handlers. One unscoped query is a cross-user leak, and retrofitting means auditing every query.
5. **The streak is credited on the learner's first produced utterance** (~45–75 s in), not on session completion — and server-side before STT returns. The Core Value must never depend on a recogniser.
6. **No pronunciation scoring in v1** (or v2) — ~22.9% false-reject rate aimed at a learner with a four-for-four abandonment record.
7. **No realtime speech-to-speech API in v1** — 8–50x cost, open-tab billing risk, destroys the clean transcript the error log and review system are built on, and forces a second implementation for text mode.
8. **Audio is not persisted by default.** Delete immediately after transcription; transcript only. Keep `turns.audio_url`, `users.retain_audio` and the `BlobStore` interface so enabling it later is a flag, not a migration.

Also unanimous and worth carrying: the closed `structure_key` taxonomy before the first error is recorded · IANA timezone names with client-computed `local_date` denormalised at write time · append-only events with derived counters, never a mutable `current_streak` · no scheduler, queue, Redis or sync engine · capped AI turn length · push-to-talk rather than auto-VAD · the scaffold is the input, not a hint · no visible "reviews due" count · Vietnamese UI chrome with English only in the target content.

---

## Key Findings

### Recommended Stack

One always-on Node process, never serverless — the single decision everything else assumes, and the only unanimous stack claim. The framework is **not** load-bearing (SvelteKit 2 + `adapter-node` if starting fresh, Next.js 15 `output: standalone` if React is already known); what *is* load-bearing is a container host (Fly/Railway/Render/VPS), never Vercel/Netlify/CF Workers serverless. Next.js's `import 'server-only'` wall is a genuine advantage worth weighing: it makes "this module must never reach the browser" a build failure rather than a convention, which is the mechanical enforcement of the shared-key constraint.

**Core technologies:**
- **Node 22 LTS, one process** — holds the DB connection, the streaming response and the budget counter; survives the move to a URL untouched
- **Postgres 16 + Drizzle ORM + checked-in SQL migrations** — Conflict 1; `DATABASE_URL` is the only local/prod difference
- **Tiered LLM calls** — a mid-tier model for the conversation turn (the scaffolding system prompt is long and rule-dense; instruction adherence under that load is what you are buying), a stronger model once per session for error analysis (highest leverage per dollar — its output compounds into every future session), a cheap/fast model for the busy-day path. Prompt caching on the system prompt + scaffold library cuts the conversation line ~60–70%; keep volatile content *after* the last cache breakpoint or the line roughly triples
- **Server-side STT behind a port** — Conflict 2; vendor pending spike; push-to-talk, complete-clip upload, no streaming in v1
- **TTS with a style/`instructions` parameter, server-side, cached by `hash(text+voice+instructions)`** — "speak slowly and clearly, as if to a beginner" is a product feature, and speaking rate is itself a withdrawable scaffold. Scaffold-frame audio is static per scenario: pre-generate it. Browser `SpeechSynthesis` stays a degraded fallback only (iOS exposes only low-quality voices, so the learner would hear a different "teacher" per device)
- **Managed magic-link / email+password auth with signup disabled** — a day of work bought, not a fortnight built; Lucia is deprecated. Identity (`users.id` as an opaque uuid, never email) exists from migration 001; the credential mechanism can be dev-mode until the first deploy
- **`usage_counters` in the database, not Redis** — a `SUM()` over ~150 rows/month is the entire implementation
- **No scheduler, no queue, no Redis, no object storage, no sync engine** — all four documents independently warn against these

**Monthly cost at 5 users x 1 session/day:** ~$12–18 total (~$3/user). Cost is not the binding constraint — pedagogical quality is. Design against <=$5/user/month; if the design can't hit that, the design is wrong, not the budget.

### Expected Features

**Must have (v1 — the Core Value fails without these):**
- **Conversation turn engine with a locked structured-output schema** — the keystone; scaffold, error capture and review injection all read from the same object. If the model returns prose, all three are unbuildable
- **The scaffold IS the input, not a hint panel** — the turn payload always carries `scaffold_options[{display, target_utterance}]`, `frame{template, blank_slot, candidate_fillers[]}` and `expected_target`. Verification: *a learner who types nothing can complete a full session by tapping only*
- **Text mode end to end, one hand-authored scenario, fully scaffolded**
- **Streak banked on the first utterance**, server-side, before STT returns
- **Busy-day path the app chooses, never asks** — "how much time do you have?" is a decision at the exact moment decisions kill habits; reachable in one tap, completes in under 60 s
- **One in-flow correction per turn, selected by recurrence, never inside the AI's spoken reply** — corrections live in a separate `errors` field so a model failure degrades to no correction rather than a broken conversation
- **Error capture into a closed Vietnamese-L1 taxonomy (~25–40 `structure_key`s + `other`), silent — no learner-visible review yet**
- **"Your sentences" session close** — the learner's own utterances, dated. Zero computation, highest emotional payload, literally identical to the Core Value
- **Vietnamese UI chrome and Vietnamese explanations; English only in the target content** — cheap, high-leverage, routinely skipped because the dev is building "an English app"
- **Per-user isolation + per-user budget enforcement**

**Should have (v1.x — differentiators):**
- **Contingent scaffold withdrawal, per function rather than global** — a learner can be unscaffolded at greetings and fully supported at past-tense narration simultaneously. This is the whitespace: nobody combines contingent withdrawal with conversational error capture fed back into the same conversation
- **Capped watchlist review injected conversationally** (Conflict 3) — trigger: ~20 matured patterns exist
- **Vietnamese end-of-session recap** — three bullets: what you said, what's standard, why. Recovers the explicitness that in-flow recasts lack
- **Voice mode** (push-to-talk, model audio on every frame, visible transcript, 0.8x speech rate, one-tap drop to text) — position per Conflict 4
- **Free automatic retroactive streak repair, ~2/month, never sold, never a currency**
- **Scaffold-position progress view** — "Ordering food: no help needed" is the only metric that measures what the product claims to do
- **Then-vs-now transcript pair** — the strongest intrinsic motivator available without social features

**Defer (v2+) / never:**
- Pronunciation scoring and phoneme feedback; realtime speech-to-speech; FSRS per-user optimisation; notification-timing ML; CEFR level estimation
- Words-learned / accuracy-% / time-studied metrics — the first is the exact metric that produced "knows words but can't produce them"; the second goes *down* as the learner improves, because fading the scaffold increases errors by design
- Hearts, lives, XP, leagues, failure states, a visible "reviews due" count, onboarding placement tests, open "Let's talk!" chat boxes
- Offline AI conversation, learner-supplied content, scenario-authoring UI

### Architecture Approach

A conventional server app whose *deploy target is an environment variable*. ARCHITECTURE's terminology correction is load-bearing and should be repeated in the roadmap: PROJECT.md's "local-first" means **localhost-first**, not local-first in the CRDT/sync-engine sense. Reading it the other way leads to ElectricSQL/Automerge/Yjs for five users who each read only their own rows on one device — the single largest over-engineering risk in the project.

**Major components:**
1. **The budget choke point** (`server/budget/assert.ts`) — every AI path calls it *before* spending. Returns a tier (FULL -> REDUCED -> WINDING_DOWN -> ZERO), **never a 429**.
2. **The turn call** (`server/ai/turn.ts`) — one streaming structured call producing reply and frames, with `reply` ordered first in the schema so speakable text arrives before UI chrome. One call, not a three-call pipeline (2.5–4 s to first word kills the loop) and not an agent framework (over-engineering at 5 users).
3. **The assessor** (`server/ai/errors.ts`) — a separate call fired in parallel the instant the utterance arrives, never after the reply, never with a persona. Fire-and-forget; must never be able to fail a turn.
4. **Scaffold level as pure code** (`server/scaffold/level.ts`) — no model call, ~3 ms. The thing you most need stable, testable and auditable — how much help this person gets right now — must not be a model decision. Store level as a float, bucket in the UI.
5. **The Zero-Cost Activity** — a streak-qualifying activity that makes zero model calls. The most valuable architectural result in the research set: it satisfies the busy-day requirement, the budget-exhausted tier and offline, as one component.
6. **`lib/speech` adapter + `lib/datetime.ts`** — the only swap point for all audio architectures, and the only file where timezone math happens.
7. **Append-only practice-event log** with timestamp + IANA timezone; all counters derived.

Conversation state lives in `turns` rows; **no server-side in-memory conversation state, ever.** That one rule gives resumption after a dropped connection, after closing the tab, on another device, and safe redeploys, with no additional machinery. Nothing runs on a schedule: streaks, due-ness and budget windows are all computed on read, which is also what makes the "whose midnight?" question evaporate.

### Critical Pitfalls

1. **The blank input box at beginner level** — the project's named central risk, and it recurs *even when scaffolding exists*, because the scaffold gets built as a hint panel beside a text box. A hint panel still requires typing. Avoid by making the scaffold the input at the protocol level; free-text unlocks by level, it is not the default.
2. **LLM level drift — certain to occur, and prompting will not fix it.** A1/B1/C1-prompted tutors converge to near-complete readability overlap by **turns 8–9**. Mitigate with context windowing (last 3–4 turns + a compact state summary — this fixes drift, quadratic cost and latency with one change), plus a vocabulary allowlist and a deterministic output validator with capped regeneration (controlled generation moved beginner-comprehensible output from **39.4% to 83.3%**). Do **not** gate on readability formulas — they were validated on paragraphs, not six-word turns.
3. **The encouraging-partner / honest-assessor conflict.** One call asked to be warm *and* to judge will not judge honestly: models agree with users' wrong answers >24% of the time, some at 58–60%, and RLHF pushed false positives from 46.7% to 70.2% in one setup. The consequence is specific — the learner practises daily for three months, hears "Great job!" every time, and discovers in a real conversation that nobody understands them. That is worse than quitting on day three. **Split the calls in Phase 1**; the assessor's prompt must not contain "student," "learner," or "encourage." Bolting this on later means rewriting the pipeline *and* re-deriving every stored error record.
4. **STT misrecognition reported as learner error** — Conflict 2's correctness rules. Recovery cost is rated VERY HIGH and possibly unrecoverable.
5. **Cost blowup on the shared key** — ranked by likelihood here: an open tab in a realtime session (avoided entirely by push-to-talk — an idle tab costs $0), unwindowed context accumulation (10–20x on long sessions), a retry storm in a client effect (unbounded; the classic solo-dev bill), key exfiltration ($82k in two days on one stolen key; $600k at METR), and an unauthenticated model-touching route, which scanners find.
6. **Auto-VAD turn endpointing** — tuned for native speakers at 500–800 ms of silence; a "mất gốc" learner pauses 2–4 seconds mid-word-retrieval and gets cut off repeatedly. Push-to-talk is strictly better here, not a compromise. Decide this in Phase 1 so the voice phase isn't scoped around realtime.

### The Scope Warning (preserved — do not soften)

> **The most likely way this project dies half-built.** Week 1: the chat shell and the LLM call come together fast and feel magical, *because the developer is testing it and the developer can hold a conversation in English*. Weeks 2–4: voice mode consumes everything — Safari, permissions, VAD, latency. Week 5: the developer is tired and ships it to the learner. The learner opens an unscaffolded chat that has drifted to B1 by turn 6, freezes at the empty box, and quits on day three — the identical failure to Duolingo, arrived at by a more expensive route. **The scaffold content, which was the entire thesis, was never authored.**
>
> **The second most likely death:** the streak and progress UI get polished because they are fun and visible, while correction, error capture and review stay stubs. The learner gets a 30-day streak and no English, discovers this, and the trust loss is worse than quitting early.

"The AI generates the lessons" is false and is the **#1 under-estimated item in the project.** Scaffold frames need a deliberate progression — which structures, in what order, with which ~800 words, across which scenarios. Ten scenarios x ~8 turns of hand-checked frames *is* the product, and it is content work, not code. **Treat it as a named, parallel, non-engineering workstream that starts in Phase 1 and is a gating deliverable for Phase 2, not a task buried inside an engineering phase.** Hand-waving it for scenario #1 is never acceptable; letting the LLM improvise frames is acceptable only for scenarios 5–10 once the pattern is proven.

---

## Implications for Roadmap

### Reconciled Phase Ordering

All four documents proposed orderings under different names (STACK's load-bearing list, FEATURES' dependency graph, ARCHITECTURE's F1–F5 / C1–C6, PITFALLS' architectural-vs-refinement split). This is the single reconciled ordering; disagreements and who wins are noted inline.

#### Phase 1: Foundation & Irreversible Contracts
**Rationale:** Every item here is cheap now and ruinous later — each is either unbackfillable data or a retrofit across every call site. ARCHITECTURE is right that F1–F5 are one phase, not five. PITFALLS' architectural list is the same set seen from the pitfall side; merge them.
**Delivers:** Postgres + Drizzle + migration 001 with `user_id NOT NULL` everywhere and per-user scoping enforced **at the data-access layer** · opaque uuid `users.id` + IANA timezone (never an offset) · build-enforced server-only boundary · zod-validated `env.ts` that throws at boot · `.gitignore` + secret scanner before commit #1 · **one server-side model gateway with `assertBudget()` before every call, cost logged from request #1, max 2 retries that debit the user's budget, a global kill-switch, and a provider-account spend cap** · `lib/datetime.ts` + client-computed `local_date` + the 3-hour post-midnight grace window · append-only practice-event log · client-generated `session.id` / `client_turn_id` with `ON CONFLICT DO NOTHING` · `content/structure-keys.ts` (the closed error taxonomy) · the turn protocol carrying `scaffold_options`, `frame`, `expected_target` · the `{text, source, confidence?}` input boundary and the `lib/speech` port · `BlobStore` interface wired to local disk · **a minimal eval harness** (20 fixed learner turns, assertions on vocabulary coverage, sentence length, correction count, assessor honesty) — half a day, and built later means never built.
**Avoids:** Pitfalls 1, 3, 5, 6, 8, 10 — all of them at protocol level.
**In parallel, non-engineering:** scenario #1 scaffold content authoring begins now.

> **Ordering disagreement resolved:** STACK orders "Auth before any conversation feature." ARCHITECTURE and PITFALLS both allow dev-mode login on localhost with a real `users` row. ARCHITECTURE/PITFALLS win — STACK conflates *identity* (needed in migration 001, non-negotiable) with *authentication* (needed before the first deploy). Real credentials land in Phase 3.

#### Phase 2: The Text Loop, One Scenario — and the 7-Day Gate
**Rationale:** This is the product thesis and it must be proven before anything is built on top of it. Text exercises the identical loop with zero audio risk; shipping voice first means a loop bug and a microphone bug look identical.
**Delivers:** streaming turn route with one structured call (`reply` first) · context windowing + vocabulary allowlist + deterministic validator with capped regeneration and a hand-authored safe-reply fallback · partner call separated from assessor call, assessor firing in parallel on utterance arrival · one in-flow correction per turn, the rest logged silently · **streak banked on first utterance** · "your sentences" close · busy-day path the app selects, one tap, under 60 s · Vietnamese UI chrome · a fixed/manual scaffold level (the ladder *protocol* is live, the automatic withdrawal *policy* is not).
**Gate:** **the owner uses it for 7 consecutive days before any further feature is built.** An explicit roadmap checkpoint, not a hope. If the loop does not survive 7 days of the owner's own use in text, voice will not rescue it — it will only make the corpse more expensive.

> **Ordering disagreement resolved:** ARCHITECTURE puts scaffold state (C2) after the text turn (C1); PITFALLS says the scaffold must be in Phase 1. Both win, on different objects: the **protocol** (payload shape, `expected_target`, tap-only completion) is Phase 1, because changing the turn contract later invalidates the UI, the STT biasing and every stored turn; the **policy** (automatic promotion/demotion) is Phase 4, starting from a manual level setting.

#### Phase 3: Make It Real — Deploy, Auth, Error Capture, Vietnamese Recap
**Rationale:** Daily habit realistically requires phone access, and the HTTPS/secure-context work is a prerequisite for voice anyway. Error capture writes the dataset the watchlist needs; it must run for weeks before review ships, or review ships on week-one noise.
**Delivers:** managed auth on every model-touching route · deploy to a container host + managed Postgres (`DATABASE_URL` is the only change) · HTTPS tunnel for phone testing established *now*, not at the end · error capture into the closed taxonomy, silent · append-only review-outcome log (the unbackfillable one) · Vietnamese end-of-session recap · the Zero-Cost Activity, content-sourced from hand-authored scenario frames.

> **Ordering disagreement resolved:** ARCHITECTURE puts the Zero-Cost Activity at C5, *after* SRS. That dependency is an artifact of assuming its content source is due review items. The content source is swappable; the component is not. Build the ZCA here from stored frames and upgrade its content source in Phase 4 — otherwise the busy-day path (a v1 requirement in PROJECT.md and the named mitigation for irregular study time) is blocked behind a review system that intentionally has no data yet.

#### Phase 4: Watchlist Review + Scaffold Withdrawal
**Rationale:** Both need accumulated data from Phase 3. The watchlist and the correction escalation ladder (silent log -> in-flow recast -> next scenario *chosen* to elicit that structure) are **one system, not two** — build them together.
**Delivers:** capped ~20-item priority watchlist with displacement and aggressive retirement, behind a scheduler interface · review as conversational elicitation that biases scenario and frame selection, never a deck · contingent scaffold withdrawal per function with asymmetric movement (demotion fast and undamped, promotion slow and confidence-weighted — a learner stuck too high produces nothing, which is a Core Value failure; a learner too low is merely bored) · scaffold-position progress view · free automatic streak repair.

#### Phase 5: Voice (position subject to the owner's Conflict 4 decision)
**Rationale:** Highest risk-to-value ratio in the project and the documented schedule-eater. Everything it needs — the adapter boundary, `expected_target`, the push-to-talk decision, transcript-confirmation rules, streak-before-STT — already exists from Phase 1, so this phase is genuinely additive.
**Delivers:** push-to-talk capture with `MediaRecorder` format detection · server-side STT behind the existing port with `expected_target` biasing · always-visible transcript with "that's not what I said" · pre-generated cached TTS for all scaffold frames · 0.8x default speech rate · one-tap drop to text mid-session · iOS audio unlock on the first tap of the session.
**Blocked on:** the STT measurement spike. Do not plan this phase before it runs.

#### Phase 6: Content Scale-Out & Long-Tail
Scenarios 2–10 (content workstream continuing) · then-vs-now transcript comparison · retired-patterns and longest-unscaffolded-turn metrics · one fixed-time daily notification, maximum, learner-set, deep-linked to a <=60 s activity, never a second nag, and only after 30 days of real daily use proves it is needed.

### Phase Ordering Rationale

- **Unbackfillable before everything.** Timezone-correct dates, append-only events, idempotency keys, the closed taxonomy and the review-outcome log cannot be reconstructed later. Four documents say this independently.
- **The choke point before the first model call.** Retrofit cost scales with call-site count and you will miss one. One function now; a week of refactor and zero recoverable cost history later.
- **Protocol before policy.** The turn contract, the speech input shape and `expected_target` are Phase 1; the scaffold-withdrawal policy and the review scorer are later. Changing a protocol invalidates stored data; changing a policy is a code edit.
- **Prove the loop in text, with a human gate, before adding modalities.** This single ordering decision is the research's main defence against the documented death mode.
- **Error capture precedes review by weeks, deliberately.** Shipping review on week-one data produces a garbage queue, and the two things that killed the learner's previous attempts were a garbage queue and a session that didn't fit a busy day.
- **Content authoring is a parallel track with its own deliverables**, not a task hidden inside an engineering phase.

### Spikes That Must Run Before Specific Phases Are Planned

| # | Spike | Before | Why |
|---|-------|--------|-----|
| 1 | **Vietnamese-accented-English STT measurement.** Record 20 real utterances from the actual learner (not the developer). Run Deepgram Nova-3 with and without keyterms, OpenAI `gpt-4o-transcribe` with and without `prompt`, and Chrome/Safari Web Speech as a control. Metric: WER **and** normalised similarity-to-`expected_target` at a generous threshold — the latter is what the product actually uses. | Phase 5 (voice) | **Two documents independently call this the single biggest unknown.** It settles the vendor, the cost model, the latency budget, and whether voice is viable at all. Half a day. |
| 2 | **Level-drift / validator calibration.** Replay a 15-turn transcript; log out-of-vocabulary content-word count and mean sentence length against turn index; verify they are flat from turn 1 to turn 12. | Phase 2 | Drift is certain and is visible in that chart before it is visible in the learner's behaviour. Doubles as the eval harness's first assertion. |
| 3 | **Assessor-honesty fixture.** "I yesterday go school very much" must return a non-empty `errors` array. | Phase 1 (build as a test) | A sycophantic assessor shipped is a HIGH-cost recovery — it poisons every stored error record, not just the UX. |
| 4 | **Provider pricing re-verification** against official pricing pages. | Phase 1 (budget work) | ARCHITECTURE's pricing figures are websearch-sourced and explicitly flagged as not re-measured; the daily-budget default depends on them. |
| 5 | **Measured LLM TTFT + prompt-cache hit rate** on the real system prompt. | During Phase 2 | ARCHITECTURE labels 400–800 ms TTFT an ESTIMATE; STACK notes that injecting volatile content *before* the last cache breakpoint roughly triples the conversation cost line. Both are measurements, not research. |
| 6 | **iOS secure-context + audio-unlock smoke test on a real iPhone** (not the simulator), over a tunnel. | Inside Phase 5, first task | `http://192.168.x.x` is not a secure context; you physically cannot test the mic on a phone against the dev server without a tunnel. Three documents flag this independently. |
| 7 | **Prompt design for scaffolded correction** — a research problem in its own right, distinct from the engine shape. | Phase 2 / Phase 4 | ARCHITECTURE recommends this get its own AI-spec treatment. |

### Research Flags

**Phases likely needing deeper research during planning:**
- **Phase 2** — the scaffolded-correction prompt is unsolved content design, and controlled generation (vocabulary allowlist + validator loop) has a known mechanism but no reference implementation for this use.
- **Phase 4 (withdrawal)** — STACK is explicit that no settled engineering pattern exists for "measure production quality, decrement scaffold level." This is product design, not stack selection. Avoidance detection (tracking target forms the scenario invited but the learner never attempted — Vietnamese L1 learners famously *avoid* complex tenses rather than producing them wrong) is under-specified in all four documents and needs phase-level research.
- **Phase 4 (review)** — no prior art exists for spaced review over conversational errors. FEATURES' adaptation is its main original synthesis and is reasoned, not measured.
- **Phase 5 (voice)** — gated on Spike 1; scope and cost model both move on the result.

**Phases with standard patterns (skip research-phase):**
- **Phase 1** — schema, migrations, auth, rate limiting, timezone handling and deployment are all well-trodden. The risk here is forgetting an item, not not knowing how. Use the Phase 1 list above as a checklist.
- **Phase 3 deploy/auth** — managed provider, one day.
- **Phase 6** — content work and straightforward read-only views.

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | **MEDIUM-HIGH** | HIGH on the process/persistence shape and on vendor pricing (read from official pages). MEDIUM on framework minor versions — pin at scaffold time. MEDIUM-HIGH on STT accuracy claims, and that is generous: the benchmarks are accented English in aggregate, not Vietnamese specifically. |
| Features | **MEDIUM-HIGH** | HIGH on the pedagogy (Lyster & Saito on prompts-vs-recasts, Aljaafreh & Lantolf's regulatory scale, Lally on habit automaticity, the Vietnamese L1 error inventory, FSRS-vs-SM-2 as an algorithm). MEDIUM on SRS transfer to conversational items (no prior art). **LOW on every specific numeric threshold** — 3 bypassed turns, 20 s silence, 3-occurrence maturation, 3-item cap are defensible starting points, not research-derived. Treat each as a config constant and expect to tune. LOW on competitor internals (vendor-adjacent blogs only). |
| Architecture | **MEDIUM — with a provenance caveat, stated plainly** | **This project's config has every MCP search provider disabled (Exa, Brave, Tavily, Firecrawl, Ref, Perplexity, Jina all false), so ARCHITECTURE.md's provenance is websearch/webfetch only, which the project's own classifier rates LOW.** In practice the auth/endpoint claims were read directly from official OpenAI docs and are reliable; the STT-latency and realtime-pricing numbers are corroborated across three sources but **not re-measured** (Spike 4); the Drizzle portability rules are community guidance, not Drizzle's own docs; LLM TTFT is an explicit ESTIMATE. The *reasoning* in that document — choke point, parallel assessor, compute-on-read, no in-memory state, the Zero-Cost Activity — does not depend on those numbers and is high quality. Do not present the whole document as equally settled. |
| Pitfalls | **HIGH** | Externally sourced with figures throughout: L2-ARCTIC per-L1 error rates, CEFR drift at turns 8–9, controlled generation 39.4% -> 83.3%, sycophancy 24–60%, mispronunciation false-reject 22.9%, education-app D30 retention 2–3%, documented key-theft bills. MEDIUM only on how these interact with a *true A0* learner — most published work uses A2–B1 university students, so the beginner numbers are extrapolations in the pessimistic direction. |

**Overall confidence: MEDIUM-HIGH.** The load-bearing decisions are corroborated by two or more documents reasoning independently. The soft spots are concentrated in three places and all three are named spikes.

### Gaps to Address

- **ASR accuracy on this specific learner's speech — the biggest unknown in the project.** Nobody has the number; no benchmark for Vietnamese-accented *beginner* English exists. -> Spike 1, before Phase 5 is planned. Handle by keeping the provider behind a port so the answer is an env var.
- **Does scaffold-target biasing materially reduce error rate for this learner?** The mechanism is sound and two documents endorse it, but the supporting figures are a vendor case study. -> Measured as part of Spike 1 (with-keyterms vs without is the comparison that matters).
- **No prior art for review over conversational errors.** -> Mitigated by choosing the simple scorer (Conflict 3) and logging the data FSRS would need. Revisit at Phase 4 with real history.
- **Avoidance detection is under-specified** — the right idea for a Vietnamese L1 learner, but neither the literature nor the competitor review produced an implementable method. -> Flag the Phase 4 withdrawal work for dedicated research.
- **No evidence on habit retention for non-social, non-gamified learning apps specifically.** Nearly all retention data comes from gamified/social products, so the three intrinsic substitutes (self-comparison across time, the content artifact, scaffold position as visible competence) are reasoned, not measured. -> Instrument them and watch; the owner's own 7-day gate is the first real data point.
- **Pricing and latency figures are corroborated, not re-measured.** -> Spikes 4 and 5, before the budget default and the latency budget are fixed.
- **Framework minor versions are MEDIUM confidence** in a monthly-moving ecosystem. -> Pin at scaffold time; the architecture claims are what matter.

## Sources

### Primary (HIGH confidence)
- [OpenAI — Realtime API with WebRTC, official docs](https://developers.openai.com/api/docs/guides/realtime-webrtc.md) — ephemeral client secrets, safety identifier, "only use standard API keys on the server"
- [OpenAI API pricing](https://developers.openai.com/api/docs/pricing) · [Deepgram pricing](https://deepgram.com/pricing) · [Claude model pricing](https://www.anthropic.com/pricing) — per-minute and per-token rates
- [ASR for Non-Native English (arXiv 2503.06924)](https://arxiv.org/pdf/2503.06924) — L2-ARCTIC Vietnamese MER 0.143 (male 0.181) vs US English 0.007
- [Alignment Drift in CEFR-prompted LLMs (arXiv 2505.08351)](https://arxiv.org/html/2505.08351v2) — level distinctions collapse by turns 8–9
- [Toward Beginner-Friendly LLMs for Language Learning (arXiv 2506.04072)](https://arxiv.org/html/2506.04072v2) — 39.4% -> 83.3% via controlled generation
- [Understanding Sycophancy in LLMs (arXiv 2602.01002)](https://www.alphaxiv.org/overview/2602.01002) — 24–60% agreement with wrong answers; RLHF false positives 46.7% -> 70.2%
- [Lyster & Saito oral CF meta-analysis](https://caslsintercom.uoregon.edu/content/22384) — prompts > recasts; low-proficiency learners miss recasts
- [Anki FAQ: spaced repetition algorithm](https://faqs.ankiweb.net/what-spaced-repetition-algorithm) — FSRS vs SM-2, 700M-review benchmark
- [caniuse: Speech Recognition API](https://caniuse.com/speech-recognition) · [WebKit MediaRecorder](https://webkit.org/blog/11353/mediarecorder-api/) · [WebKit bug 290497](https://bugs.webkit.org/show_bug.cgi?id=290497) — browser support and iOS voice limits
- [AI token jacking / stolen key bills (Gridinsoft)](https://blog.gridinsoft.com/ai-token-jacking-stolen-api-keys/) · [METR $600k via ITPro](https://www.itpro.com/security/cyber-attacks/hackers-ran-up-a-usd600-000-ai-bill-after-swiping-api-keys-says-metr-and-nobody-realized-for-weeks)

### Secondary (MEDIUM confidence)
- [Deepgram: Introducing Nova-3](https://deepgram.com/learn/introducing-nova-3-speech-to-text-api) — keyterm prompting, vendor-reported WER reduction
- [Satar & Özdener, text vs voice CMC](https://oro.open.ac.uk/17665) — proficiency gains in both modes, anxiety reduction only in text
- [Aljaafreh & Lantolf regulatory scale](https://my.vanderbilt.edu/l2studies/?p=121) · [Bell Foundation scaffolding](https://www.bell-foundation.org.uk/resources/great-ideas/scaffolding/) · [CoMeT, n=131 (arXiv 2609.22993)](https://arxiv.org/abs/2609.22993) — contingency and fading
- [Lally et al. via ScienceBlog](https://scienceblog.com/5-research-backed-ways-to-build-a-habit-that-actually-lasts/) — missing one day is about -0.29 automaticity; missing a week derails
- [Streak Creep (The Decision Lab)](https://thedecisionlab.com/insights/consumer-insights/streak-creep-the-perils-of-too-much-gamification) · [Duolingo streak research](https://blog.duolingo.com/duolingo-streak-research/) · [Braze push opt-out data](https://www.braze.com/blog/opt-out-of-push-notifications-why-users-do-it/)
- [Mispronunciation detection false-reject 22.9%](https://signal.ejournal.org.cn/en/article/doi/10.16798/j.issn.1003-0530.2020.06.020) · [native phoneme error floor ~15% (arXiv 2209.06265)](https://arxiv.org/pdf/2209.06265)
- [OpenAI Realtime pricing analysis (Forasoft)](https://www.forasoft.com/article/openai-realtime-api-pricing) · [Realtime API extremely expensive (OpenAI community)](https://community.openai.com/t/realtime-api-extremely-expensive/966825) — $0.46/min runaway, ~$6 for 75 s
- [800 ms latency rule (twig.so)](https://www.twig.so/blog/voice-ai-agents-latency-budget-800ms) · [Deepgram voice agent architecture](https://deepgram.com/learn/voice-agent-architecture-stt-llm-tts-pipeline-design)
- [Fly.io ephemeral root filesystem](https://fly.io/docs/js/prisma/sqlite/) · [Turso/libSQL 2026](https://noqta.tn/en/blog/turso-libsql-distributed-sqlite-edge-database-2026)
- [Vietnamese L1 English errors — TEFL Academy](https://www.theteflacademy.com/blog/common-mistakes-of-vietnamese-learners-of-english/) · [Dong Nai University study](https://vjol.info.vn/NNDS/article/view/20281)
- [ts-fsrs](https://npmjs.com/package/ts-fsrs) · [Anki Burnout](https://www.neonlingo.com/blog/anki-burnout) — review debt, ease hell
- [Better Auth vs Lucia vs NextAuth 2026](https://www.pkgpulse.com/guides/better-auth-vs-lucia-vs-nextauth-2026) — Lucia deprecation

### Tertiary (LOW confidence — needs validation)
- [Drizzle portable-schema guidance](https://instagit.com/BuilderIO/agent-native/using-drizzle-orm-agent-native-portable-schema.md) — community, not official; underpins Conflict 1's portability argument
- [Nova-3 accent analysis](https://convertaudiototext.com/blog/deepgram-nova-3-explained) · [Deepgram vs Whisper comparison](https://www.callmissed.com/blog/deepgram-nova-vs-whisper-large-v3-turbo) — accented-audio claims, directional only
- [Education-app retention benchmarks (Passion.io)](https://passion.io/blog/mobile-app-retention-benchmarks-for-creators-course-coaching-apps) — D1 14–15%, D30 2–3%; methodology varies
- [TalkPal review](https://languatalk.com/blog/talkpal-review/) · [best AI language app comparison](https://languatalk.com/blog/whats-the-best-ai-for-language-learning/) — competitor-adjacent blogs, not independent teardowns
- [Streak feature design (dev.to)](https://dev.to/charlie_brinicombe/how-to-build-a-streaks-feature-4fck) — design folklore, well-corroborated

---
*Research completed: 2026-10-06*
*Ready for roadmap: yes — with one open scope question (Conflict 4, voice in v1) awaiting the project owner*
