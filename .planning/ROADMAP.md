# Roadmap: LearnEnglish

## Overview

This roadmap is built backward from one sentence: **every single day the learner opens the app and successfully produces at least one English utterance.** Every phase below is judged on whether it moves that, and the phases are ordered so that the things which cannot be fixed later happen first.

The shape is: lay the unbackfillable contracts and the budget choke point before a single model call exists (Phase 1); author the scaffold curriculum as real written material on a parallel non-engineering track (Phase 2); prove the text loop end to end with one scenario and then stop and let the owner use it for seven consecutive days (Phase 3); give the owner English in their ears (Phase 4); put it on a URL, behind auth, with silent error capture running (Phase 5). Review, withdrawal, speaking and content scale-out are v1.x and sit below the line.

Two things in this roadmap are not engineering phases and must not be read as optional: the **scaffold content track** (Phase 2) and the **7-day self-use gate** (between Phase 3 and Phase 4). The research is unambiguous that the project dies at exactly those two points.

### Departures from `research/SUMMARY.md`

SUMMARY's reconciled ordering is carried forward intact except where the owner's Conflict 4 decision forced a change. Stated plainly:

1. **SUMMARY's "Phase 5: Voice" is split in two.** Text-to-speech (LISN-01…04) moves forward into v1 as Phase 4; speech-to-text (SPK-01…07) stays as a later standalone phase, blocked on Spike 1. Rationale: TTS treats the listening weakness the owner named ("người bản xứ nói nhanh không kịp"), carries none of the ~20x Vietnamese-L1 recognition risk, costs ~$1.50/month, and needs no microphone, no `getUserMedia`, and no secure context beyond ordinary HTTPS. Its only real constraint is the iOS first-tap audio unlock.
2. **Spike 6 (iOS secure-context + audio-unlock on a real iPhone) moves from the voice phase to Phase 4**, and **ACC-05 (the HTTPS tunnel) moves with it.** ACC-05's own wording is "established before the phase that needs a secure context, not at the end" — with TTS in v1, Phase 4 is now that phase, not the speaking phase.
3. **The content track is promoted to a numbered top-level phase (Phase 2)** rather than a parallel annotation. It starts with Phase 1, runs alongside it, and gates Phase 3. SUMMARY asked for a named parallel non-engineering workstream with its own deliverables; a top-level phase number is the only way that survives contact with a planner.
4. **The Vietnamese end-of-session recap is v1.x, not v1.** SUMMARY placed it in its Phase 3; `REQUIREMENTS.md` classifies it as REV-07 under v1.x. The requirements document wins on scope. It lands in Phase 6 with the rest of the review system.

Everything else — unbackfillable data contracts before everything, the budget choke point before the first model call, protocol before policy, prove the loop in text with a human gate before adding modalities, error capture preceding review by weeks deliberately — is SUMMARY's reconciled ordering, which four research documents reached independently.

---

## The Scope Warning

Preserved verbatim from `research/SUMMARY.md § The Scope Warning`. It is the most actionable finding in the research set and is reproduced here unsoftened because this is the document that gets read before each phase.

> **The most likely way this project dies half-built.** Week 1: the chat shell and the LLM call come together fast and feel magical, *because the developer is testing it and the developer can hold a conversation in English*. Weeks 2–4: voice mode consumes everything — Safari, permissions, VAD, latency. Week 5: the developer is tired and ships it to the learner. The learner opens an unscaffolded chat that has drifted to B1 by turn 6, freezes at the empty box, and quits on day three — the identical failure to Duolingo, arrived at by a more expensive route. **The scaffold content, which was the entire thesis, was never authored.**
>
> **The second most likely death:** the streak and progress UI get polished because they are fun and visible, while correction, error capture and review stay stubs. The learner gets a 30-day streak and no English, discovers this, and the trust loss is worse than quitting early.

"The AI generates the lessons" is false and is the **#1 under-estimated item in the project.** Phase 2 exists because of this paragraph.

---

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

### v1 — the committed scope

- [ ] **Phase 1: Foundation & Irreversible Contracts** - Every contract that cannot be backfilled later, plus the single budget choke point, in place before the first model call exists
- [ ] **Phase 2: Scaffold Curriculum — Scenario 1** - Hand-authored content track, parallel to Phase 1, gating Phase 3: the structures, order, ~800 words and Vietnamese copy that *are* the product
- [ ] **Phase 3: The Text Loop, One Scenario** - A learner who types nothing completes a real English conversation by tapping, and banks a streak — then the 7-day gate
- [ ] **Phase 4: Listening — The Partner Speaks** - The owner hears slowed, clear English on every turn, on a real phone
- [ ] **Phase 5: Make It Real — Deploy, Auth, Silent Error Capture** - Friends reach it at a URL, see only their own data, and every error is being recorded weeks before any review screen exists

### v1.x — planned, deferred, not committed

- [ ] **Phase 6: Watchlist Review & Scaffold Withdrawal** - Accumulated errors come back inside conversation, and the scaffold starts withdrawing itself
- [ ] **Phase 7: Speaking — Push-to-Talk Input** - The learner speaks instead of typing, behind the adapter boundary built in Phase 1
- [ ] **Phase 8: Content Scale-Out & Long-Tail** - Scenarios 2–10 and the intrinsic progress signals

---

## Checkpoints

### ⛔ The 7-Day Self-Use Gate — between Phase 3 and Phase 4

**This is a hard checkpoint, not a suggestion.** The owner uses the Phase 3 text loop for **7 consecutive days** before any further feature is built. No Phase 4 work — not planning, not spikes, not "just the audio unlock" — starts until it resolves.

| Outcome | Meaning | Consequence |
|---------|---------|-------------|
| **PASS** — 7 consecutive days, each with at least one produced utterance | The loop is returnable-to. The product thesis survived contact with its only real user. | Phase 4 is unblocked. Proceed. |
| **FAIL** — any missed day inside the window | The loop is not yet returnable-to. This product fails at retention, not at features. | **Stop. Do not build Phase 4, 5 or anything else.** Diagnose why the owner did not return — session too long, scaffold too thin, friction at open, the scenario is boring — fix it as a Phase 3 revision, and restart the 7-day count from zero. |

Rationale (PITFALLS, carried through SUMMARY): if the loop does not survive 7 days of the owner's own use in text, voice will not rescue it — it will only make the corpse more expensive. This is the cheapest real evidence available in the whole project and it is bought early on purpose.

**Note on timing.** The owner's stated intent is to hear English "from the first week, not the last phase." The gate makes that week two rather than week one, because the gate is strict and nothing is built during it. Phase 4 is placed immediately after the gate — it is the very first thing that happens on day 8 — which is as early as a strict gate allows. If the owner later decides week one matters more than gate purity, LISN-01 alone (speak the partner's reply aloud, browser-side fallback voice, no caching, no rate control) has no dependency beyond the Phase 1 turn protocol and can be pulled into Phase 3; LISN-02, 03 and 04 cannot and would stay in Phase 4.

### The Phase 2 content gate — before Phase 3

Phase 3 does not start until Scenario 1's scaffold curriculum is written and readable by a human. Hand-waving the frames for scenario #1 is never acceptable. Letting the LLM improvise frames is acceptable only for scenarios 5–10, once the pattern is proven (Phase 8).

---

## Spikes

Every spike below blocks the phase named in its row. **Planning may not start on a phase whose blocking spike has not run.** Carried from `research/SUMMARY.md § Spikes That Must Run Before Specific Phases Are Planned`, with Spike 6 re-pointed per Departure 2.

| # | Spike | Blocks | Why | Size |
|---|-------|--------|-----|------|
| 1 | **Vietnamese-accented-English STT measurement.** 20 real utterances from the actual learner (not the developer). Deepgram Nova-3 with and without keyterms, OpenAI `gpt-4o-transcribe` with and without `prompt`, Chrome/Safari Web Speech as control. Metric: WER **and** normalised similarity-to-`expected_target` at a generous threshold — the latter is what the product actually uses. | **Phase 7** | Two documents independently call this the single biggest unknown. It settles the vendor, the cost model, the latency budget, and whether voice is viable at all. | ~½ day |
| 2 | **Level-drift / validator calibration.** Replay a 15-turn transcript; log out-of-vocabulary content-word count and mean sentence length against turn index; verify flat from turn 1 to turn 12. | **Phase 3** | Drift is certain and is visible in that chart before it is visible in the learner's behaviour. Doubles as the eval harness's first assertion. | ~½ day |
| 3 | **Assessor-honesty fixture.** `"I yesterday go school very much"` must return a non-empty `errors` array. | **Phase 1** (built as a test) | A sycophantic assessor shipped is a HIGH-cost recovery — it poisons every stored error record, not just the UX. | Hours |
| 4 | **Provider pricing re-verification** against official pricing pages. | **Phase 1** (budget work) | ARCHITECTURE's pricing figures are websearch-sourced and explicitly flagged as not re-measured; the daily-budget default depends on them. | Hours |
| 5 | **Measured LLM TTFT + prompt-cache hit rate** on the real system prompt. | **During Phase 3** | 400–800 ms TTFT is an ESTIMATE; injecting volatile content before the last cache breakpoint roughly triples the conversation cost line. Both are measurements, not research. | Hours |
| 6 | **iOS secure-context + audio-unlock smoke test on a real iPhone** (not the simulator), over a tunnel. | **Phase 4, first task** *(moved from the voice phase — see Departure 2)* | `http://192.168.x.x` is not a secure context. Three documents flag this independently. TTS is now the first thing that touches a real phone. | Hours |
| 7 | **Prompt design for scaffolded correction** — a research problem in its own right, distinct from the engine shape. | **Phase 3**, revisited for **Phase 6** | ARCHITECTURE recommends this get its own AI-spec treatment. | ~1 day |

---

## Research Flags

Carried from `research/SUMMARY.md § Research Flags`, renumbered to this roadmap's phases.

**Needs deeper research before planning — knowledge risk:**
- **Phase 3** — the scaffolded-correction prompt is unsolved content design (Spike 7), and controlled generation (vocabulary allowlist + deterministic validator loop) has a known mechanism but no reference implementation for this use.
- **Phase 6 (withdrawal)** — no settled engineering pattern exists for "measure production quality, decrement scaffold level." This is product design, not stack selection. **Avoidance detection** — tracking target forms the scenario invited but the learner never attempted, since Vietnamese L1 learners famously *avoid* complex tenses rather than producing them wrong — is under-specified in all four research documents and needs phase-level research.
- **Phase 6 (review)** — no prior art exists for spaced review over conversational errors. FEATURES' adaptation is its main original synthesis and is reasoned, not measured.
- **Phase 7 (speaking)** — gated on Spike 1; scope and cost model both move on the result.

**Standard patterns — skip the research phase. The risk here is forgetting an item, not not knowing how:**
- **Phase 1** — schema, migrations, rate limiting, timezone handling. Use the phase's requirement list as a checklist.
- **Phase 5 (deploy/auth)** — managed provider, roughly one day.
- **Phase 8** — content work and straightforward read-only views.

**Phase 2** is neither: it is not knowledge risk and not checklist risk. It is the risk that it never gets written.

---

## Phase Details

### Phase 1: Foundation & Irreversible Contracts
**Goal**: A user exists, a practice event is recorded against the right local date, and a model call can be made and paid for — with every contract that cannot be backfilled later already locked in. Nothing here is visible to a learner; everything here is ruinous to add afterwards.
**Depends on**: Nothing (first phase)
**Requirements**: FND-01, FND-02, FND-03, FND-04, FND-05, FND-06, FND-07, FND-08, FND-09, FND-10, COST-01, COST-02, COST-03, COST-04, COST-05, COST-06
**Blocking spikes**: Spike 3 (assessor-honesty fixture, built as a test), Spike 4 (provider pricing re-verification, before the daily-budget default is fixed)
**Research**: Not required — standard patterns, checklist risk
**Success Criteria** (what must be TRUE):
  1. The owner can run one command against a clean database and end up with a seeded user row carrying an opaque uuid and an IANA timezone name, and a practice event written against the correct `local_date` for that timezone — verified by changing the user's timezone and watching the date boundary move.
  2. Replaying the same write twice with the same idempotency key produces one row, not two — so no retry can ever inflate a streak or double-count an utterance.
  3. A deliberately planted `process.env.OPENAI_API_KEY` reference in a client component fails the build rather than shipping, and booting with a missing or malformed env var throws at startup rather than at the first request.
  4. A scripted burst of model calls past the daily budget returns a degraded tier (FULL → REDUCED → WINDING_DOWN → ZERO) and never an error or a 429, with every call's `cost_micros` visible in the spend log, and flipping the kill-switch stops spending immediately.
  5. The eval harness runs 20 fixed learner turns and fails loudly when the assessor agrees that `"I yesterday go school very much"` is fine.
**Plans**: TBD

**Notes for planning:**
- ARCHITECTURE's F1–F5 are one phase, not five; PITFALLS' architectural list is the same set seen from the pitfall side. Merge them.
- **Identity is not authentication.** `users.id` as an opaque uuid is non-negotiable in migration 001; real credentials land in Phase 5. Dev-mode login against a real `users` row is correct here.
- Per-user scoping is enforced **at the data-access layer**, not in route handlers. One unscoped query is a cross-user leak and retrofitting means auditing every query.
- The turn protocol carries `scaffold_options`, `frame` and `expected_target` **from the first turn ever written** — `expected_target` is the field that makes STT biasing possible in Phase 7, and it must exist before any turn is stored.
- `lib/speech` port, `{text, source, confidence?}` input shape, `turns.audio_url` (nullable), `users.retain_audio` (default false) and a `BlobStore` interface wired to local disk all exist here even though nothing uses them yet.
- `content/structure-keys.ts` — the closed Vietnamese-L1 error taxonomy, ~25–40 keys plus `other` — is defined before the first error is ever recorded.
- "Local-first" in `PROJECT.md` means **localhost-first**, not local-first in the CRDT sense. No ElectricSQL, Automerge or Yjs. Five users each reading their own rows on one device is the single largest over-engineering risk in the project.
- `.gitignore` + a secret scanner before commit #1. A provider-account-level spend cap is a backstop that survives a bug in our own limiter.

---

### Phase 2: Scaffold Curriculum — Scenario 1
**Goal**: Scenario 1 exists as written material a human can read, teach from, and disagree with — the structures, their order, the ~800-word vocabulary, the frames for every turn, and the Vietnamese copy that wraps them. This is the product thesis in a document, and it is content work, not code.
**Depends on**: Nothing — **runs in parallel with Phase 1 from day one.** It is a non-engineering track with its own deliverables, deliberately not a task inside an engineering phase. It **gates Phase 3**.
**Requirements**: CONT-01, CONT-02
**Blocking spikes**: None
**Research**: Not required — this is authoring, not investigation
**Success Criteria** (what must be TRUE):
  1. A reader who is not the owner can open Scenario 1's document, read the ordered list of structures and the ~800-word vocabulary, and say what the learner is expected to produce on turn 1 and on turn 8.
  2. Every turn in Scenario 1 has hand-checked frames with their blank slots and candidate fillers written out, such that a learner could complete the whole scenario by choosing among them without composing a sentence of their own.
  3. Every Vietnamese string the learner will ever see — chrome, instructions, explanations, error wording, the busy-day prompt — exists in one checked-in copy deck, with English confined to the target content only.
  4. Each structure in the curriculum maps to at least one `structure_key` in the Phase 1 error taxonomy, so a learner's failure at it is recordable.
**Plans**: TBD

**Notes for planning:**
- This is the **#1 under-estimated item in the project** and the single most likely cause of project death. Ten scenarios × ~8 turns of hand-checked frames *is* the product.
- Hand-waving the frames for scenario #1 is never acceptable. LLM-improvised frames are acceptable only for scenarios 5–10, once the pattern is proven (Phase 8).
- Vietnamese chrome and Vietnamese explanations are cheap, high-leverage, and routinely skipped because the developer is building "an English app." Writing the copy deck here is what prevents that.
- Progress on this track is measured in written turns, not in days spent. If Phase 1 finishes and this has not, **Phase 3 waits.**

---

### Phase 3: The Text Loop, One Scenario
**Goal**: The owner opens the app, holds a real English conversation in Scenario 1 without typing a single character, gets at most one correction that makes them repair their own sentence, sees their own sentences at the end, and has a streak that was banked about a minute in. This is the product thesis proven or disproven.
**Depends on**: Phase 1 (contracts, budget choke point, turn protocol), Phase 2 (Scenario 1 content — hard gate)
**Requirements**: CONV-01, CONV-02, CONV-03, CONV-04, CONV-05, CONV-06, CONV-07, CONV-08, CONV-09, CONV-10, HBT-01, HBT-02, HBT-04, HBT-05
**Blocking spikes**: Spike 2 (level-drift / validator calibration) and Spike 7 (scaffolded-correction prompt design) before planning; Spike 5 (TTFT + prompt-cache hit rate) measured during the phase
**Research**: **Required.** Scaffolded-correction prompting is unsolved content design; controlled generation has a known mechanism but no reference implementation for this use.
**UI hint**: yes
**Success Criteria** (what must be TRUE):
  1. A learner who types nothing completes a full Scenario 1 session by tapping only — the scaffold is the input mechanism, with no text box to freeze at.
  2. By turn 12 the AI partner's vocabulary and sentence length are flat against turn 1 — no drift to B1 — because out-of-allowlist output is caught by the validator, regenerated within a cap, and falls back to a hand-authored safe reply rather than reaching the learner.
  3. The streak is credited roughly 45–75 seconds in, on the learner's first produced utterance, server-side — and killing the model provider mid-session afterwards does not take the streak away.
  4. When the learner makes a recurring error, they get at most one correction, phrased as a prompt that makes them say it again themselves rather than a recast they can read as the conversation simply continuing — and when the assessor call fails entirely, the conversation continues with no correction rather than breaking.
  5. On a day with no time, the app offers a path it chose itself — never asking "how much time do you have?" — reachable in one tap and done in under 60 seconds, which still counts.
  6. Closing the tab mid-conversation and reopening it resumes the same conversation on the same or another device, and the session ends by showing the learner their own sentences, dated.
**Plans**: TBD

**Notes for planning:**
- **Protocol is Phase 1; policy is Phase 6.** The scaffold ladder's *protocol* is already live; the *automatic withdrawal policy* is not. Scaffold level here is a **fixed or manually set float** computed in application code — never by a model call.
- One structured streaming call for the partner turn with `reply` ordered **first** in the schema, so speakable text arrives before UI chrome. Not a three-call pipeline (2.5–4 s to first word kills the loop) and not an agent framework.
- The assessor is a **separate call** with no persona and no conversation history, fired in parallel the instant the utterance arrives, fire-and-forget, and structurally incapable of failing a turn. Its prompt must not contain "student," "learner," or "encourage."
- Context windowing to the last 3–4 turns plus a compact state summary fixes drift, quadratic cost and latency with one mechanism.
- HBT-04: a missed day is never a loss or a reset. Progress is a non-decreasing count of days practiced.
- **This phase ends at the 7-day gate.** Build nothing else until it resolves.

---

### Phase 4: Listening — The Partner Speaks
**Goal**: The owner hears English spoken aloud on every turn of every session, slowed to a beginner's pace, on their actual iPhone over a real HTTPS URL — directly treating the listening weakness they named, with none of the recognition risk.
**Depends on**: Phase 3 **and a PASS on the 7-day self-use gate**
**Requirements**: LISN-01, LISN-02, LISN-03, LISN-04, ACC-05
**Blocking spikes**: Spike 6 (iOS secure-context + audio-unlock smoke test on a real iPhone over a tunnel) — **first task of the phase**
**Research**: Not required — the mechanism is known; the risk is the iOS surface
**UI hint**: yes
**Success Criteria** (what must be TRUE):
  1. The owner holds a full session on a real iPhone, over HTTPS, and hears every one of the partner's turns spoken aloud without once pressing a play button after the first tap of the session.
  2. The speech is noticeably slower and clearer than a native pace, and the rate is stored as a scaffold value that can be raised later rather than a hard-coded constant.
  3. Scaffold-frame audio plays back with no perceptible generation delay on the second and subsequent sessions, because it is pre-generated and cached by content hash.
  4. Phone testing works against the dev server through an established HTTPS tunnel — set up here, at the start, not improvised at the end of some later phase.
**Plans**: TBD

**Notes for planning:**
- This phase exists because of the owner's Conflict 4 decision: **listen first, speak later.** TTS ships; STT does not. See Departures 1 and 2 at the top of this document.
- ~$1.50/month. Cost is not the constraint here; the iOS autoplay surface is.
- `http://192.168.x.x` is not a secure context — run Spike 6 before anything else in this phase, on a physical device, not the simulator.
- Server-side TTS with a style/`instructions` parameter ("speak slowly and clearly, as if to a beginner"), cached by `hash(text+voice+instructions)`. Browser `SpeechSynthesis` is a degraded fallback only — iOS exposes only low-quality voices, so the learner would otherwise hear a different "teacher" on every device.
- Scaffold-frame audio is static per scenario. Pre-generate it from the Phase 2 content.

---

### Phase 5: Make It Real — Deploy, Auth, Silent Error Capture
**Goal**: The owner and a handful of friends each reach the app at a URL on their own phone, sign in, and see only their own history, errors and streak — while behind the scenes every error each of them makes is being normalised and recorded for weeks before any review screen exists.
**Depends on**: Phase 4
**Requirements**: ACC-01, ACC-02, ACC-03, ACC-04, ERR-01, ERR-02, ERR-03, ERR-04, ERR-05, HBT-03
**Blocking spikes**: None
**Research**: Not required for deploy/auth — managed provider, roughly one day
**UI hint**: yes
**Success Criteria** (what must be TRUE):
  1. A friend opens a URL on their own phone, signs in with an account the owner created for them, completes a session, and sees their own streak and their own sentences — and cannot reach the owner's, verified by trying.
  2. Public signup does not exist, and every model-touching route refuses an unauthenticated request.
  3. Deploying is a `DATABASE_URL` change and a container push — the same migrations, the same code, nothing forked for production.
  4. After a week of real use, querying the database shows errors normalised to `structure_key` values from the closed taxonomy, with nothing learner-visible anywhere in the UI, and a review-outcome log accumulating rows that could not be reconstructed later.
  5. On a day with no budget left, no signal, or no time, the learner can still complete a streak-qualifying activity that makes zero model calls — one component serving the busy-day path, the exhausted-budget tier and degraded connectivity at once.
**Plans**: TBD

**Notes for planning:**
- **Error capture precedes review by weeks, deliberately.** Shipping review on week-one data produces a garbage queue, and a garbage queue is one of the two things that killed the learner's previous attempts.
- ERR-03's maturation gate (≥2–3 occurrences across ≥2 separate sessions) and ERR-05 (no unconfirmed transcript ever becomes a review item) are enforced at write time here, not later at read time.
- ERR-04's append-only review-outcome log — item, outcome, timestamp, elapsed since last, what was scheduled — is the **unbackfillable** insurance policy that makes adopting FSRS in some later version possible from real history. One INSERT. Do not skip it because no review system reads it yet.
- The Zero-Cost Activity (HBT-03) is built here from stored Phase 2 frames. SUMMARY corrects ARCHITECTURE on this: its content source is swappable, the component is not, and blocking it behind a review system that intentionally has no data yet would block the busy-day path.
- Managed magic-link or email+password auth with signup disabled — a day of work bought, not a fortnight built. Lucia is deprecated.
- Container host with managed Postgres. **Never serverless** — it breaks the single always-on process the budget counter and the streaming response both assume.

---

### Phase 6: Watchlist Review & Scaffold Withdrawal — *v1.x*
**Goal**: Errors the learner actually repeats come back inside the conversation, chosen by a scorer the owner can read, and the scaffold starts withdrawing itself per language function — so a learner can be unsupported at greetings and fully supported at past-tense narration on the same day.
**Depends on**: Phase 5 **plus several weeks of accumulated real error data**. Not elapsed engineering time — elapsed usage.
**Requirements**: REV-01, REV-02, REV-03, REV-04, REV-05, REV-06, REV-07, REV-08, REV-09
**Blocking spikes**: Spike 7 revisited (scaffolded-correction prompt, now for elicitation)
**Research**: **Required.** No prior art for spaced review over conversational errors; no settled pattern for "measure production quality, decrement scaffold level"; avoidance detection under-specified in all four research documents.
**UI hint**: yes
**Success Criteria** (what must be TRUE):
  1. A structure the learner has got wrong repeatedly reappears as something the next scenario invites them to say — never as a flashcard deck, and never announced as a review.
  2. Nowhere in the interface is there a due date, a review debt, or an "N due" count.
  3. After three clean productions a pattern stops coming back; after five failures it stops coming back anyway and appears in the Vietnamese recap as something a human should look at.
  4. One bad week drops the learner's scaffold down immediately at the function they struggled with, while climbing back up takes repeated clean production — demotion fast and undamped, promotion slow and confidence-weighted.
  5. The learner can see, in plain Vietnamese, where they stand per scenario ("Ordering food: no help needed") and gets a three-bullet end-of-session recap: what you said, what's standard, why.
  6. A missed day repairs itself, retroactively and free, roughly twice a month, with nothing to buy and nothing to claim.
**Plans**: TBD

**Notes for planning:**
- The watchlist and the correction escalation ladder (silent log → in-flow prompt → next scenario *chosen* to elicit that structure) are **one system, not two.** Build them together.
- Capped ~20 items, scored `recency × recurrence × failure_count`, displacement when full, behind a scheduler interface. **No FSRS, no SM-2, no interval math** — resolved in research; every precondition for FSRS's measured advantage fails here.
- A learner stuck too high produces nothing, which is a Core Value failure; a learner too low is merely bored. That asymmetry is why withdrawal is asymmetric.
- Every numeric threshold in this phase (3 clean productions, 5 failures, ~20-item cap, 2–3 occurrence maturation) is a defensible starting point, **not research-derived**. Treat each as a config constant and expect to tune.

---

### Phase 7: Speaking — Push-to-Talk Input — *v1.x*
**Goal**: The learner says their turn out loud instead of typing it, and a misrecognition never costs them their streak and never gets reported back to them as their mistake.
**Depends on**: Phase 6 — and genuinely additive on Phase 1, since the `lib/speech` port, the `{text, source, confidence?}` input shape and `expected_target` have existed since migration 001
**Requirements**: SPK-01, SPK-02, SPK-03, SPK-04, SPK-05, SPK-06, SPK-07
**Blocking spikes**: **Spike 1 (Vietnamese-accented-English STT measurement) — do not plan this phase before it runs.** Scope, vendor and cost model all move on the result.
**Research**: **Required**, and gated on Spike 1
**UI hint**: yes
**Success Criteria** (what must be TRUE):
  1. The learner holds a button, speaks, pauses for four seconds mid-sentence while retrieving a word, and is not cut off — because there is no automatic silence detection anywhere.
  2. The streak is banked the moment enough audio is captured, before recognition returns and regardless of what it returns.
  3. The transcript is always shown before the AI replies, with a one-tap "that's not what I said" that discards the utterance from error capture, offers typing instead, and is itself logged.
  4. When recognition is ambiguous, the app says nothing about whether the learner was right and moves the conversation on — it never issues a verdict on correctness from a transcript.
  5. The learner can drop back to typing mid-session in one tap.
**Plans**: TBD

**Notes for planning:**
- **Release gate:** a "that's not what I said" tap rate above ~15% means voice mode is not ready. Ship nothing past that number.
- Vietnamese-L1 speakers score mean MER 0.143 (male 0.181) against 0.007 for US English on L2-ARCTIC — roughly 20x — and those are *fluent* L2 speakers reading prepared sentences. Plan for 25–40% WER on a "mất gốc" beginner.
- Server-side recognition behind the existing port with `expected_target` passed as a decoding hint. Browser Web Speech structurally cannot do that biasing — no hint, prompt or keyterm parameter — so it is a dev-convenience fallback only, never the default and never a source of error-capture input.
- Push-to-talk, complete-clip upload, no streaming. `MediaRecorder` format detection across webm/opus and iOS mp4/aac.
- No pronunciation scoring and no phoneme feedback, in this phase or any other version.

---

### Phase 8: Content Scale-Out & Long-Tail — *v1.x*
**Goal**: The learner has somewhere to go after Scenario 1, and can see for themselves — in their own words — that they are better than they were.
**Depends on**: Phase 6 (so the scaffold pattern is proven before it is multiplied)
**Requirements**: SCALE-01, SCALE-02, SCALE-03, SCALE-04
**Blocking spikes**: None
**Research**: Not required — content work and straightforward read-only views
**UI hint**: yes
**Success Criteria** (what must be TRUE):
  1. The learner can choose among ten scenarios, each with authored frames, without the earlier ones running dry.
  2. The learner sees a pair of their own transcripts — one from their first week, one from this week — side by side, with nothing computed or scored about them.
  3. Progress is expressed as retired patterns and longest unscaffolded turn, and nowhere as words learned, accuracy percentage, time studied or XP.
  4. At most one daily notification exists, at a time the learner chose, deep-linked to a sub-60-second activity — and it only ships at all after 30 days of real use has shown it is needed.
**Plans**: TBD

**Notes for planning:**
- Scenarios 2–10 are the content workstream continuing. LLM-improvised frames become acceptable here, for scenarios 5–10 only, once the hand-authored pattern is proven.
- Never a second nag. Never a streak-loss warning.

---

## Progress

### v1

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundation & Irreversible Contracts | 0/TBD | Not started | - |
| 2. Scaffold Curriculum — Scenario 1 | 0/TBD | Not started | - |
| ⛔ 7-Day Self-Use Gate | — | Not reached | - |
| 3. The Text Loop, One Scenario | 0/TBD | Not started | - |
| 4. Listening — The Partner Speaks | 0/TBD | Not started | - |
| 5. Make It Real — Deploy, Auth, Error Capture | 0/TBD | Not started | - |

*(The gate row sits between Phase 3 and Phase 4 in execution order; it is listed after Phase 2 only because the table is sorted by phase number.)*

### v1.x

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 6. Watchlist Review & Scaffold Withdrawal | 0/TBD | Not started | - |
| 7. Speaking — Push-to-Talk Input | 0/TBD | Not started | - |
| 8. Content Scale-Out & Long-Tail | 0/TBD | Not started | - |

---

## Coverage

All **47** v1 requirements are mapped to exactly one phase each. All **20** v1.x requirements are mapped to the three post-v1 phases.

| Phase | Requirements | Count |
|-------|--------------|-------|
| 1 | FND-01…10, COST-01…06 | 16 |
| 2 | CONT-01, CONT-02 | 2 |
| 3 | CONV-01…10, HBT-01, HBT-02, HBT-04, HBT-05 | 14 |
| 4 | LISN-01…04, ACC-05 | 5 |
| 5 | ACC-01…04, ERR-01…05, HBT-03 | 10 |
| **v1 total** | | **47** |
| 6 | REV-01…09 | 9 |
| 7 | SPK-01…07 | 7 |
| 8 | SCALE-01…04 | 4 |
| **v1.x total** | | **20** |

No orphaned requirements. No requirement appears in two phases.

---
*Roadmap created: 2026-10-06*
*Derived from `.planning/PROJECT.md`, `.planning/REQUIREMENTS.md`, and `.planning/research/SUMMARY.md` (reconciled ordering), adjusted for the owner's Conflict 4 decision: listen first, speak later.*
