# Requirements: LearnEnglish

**Defined:** 2026-10-06
**Core Value:** Every single day the learner opens the app and successfully produces at least one English utterance — spoken or typed.

Requirements are derived from `.planning/research/SUMMARY.md` and the owner's scope decisions recorded in `PROJECT.md § Key Decisions`. Where research and intuition conflicted, the resolution is noted inline.

## v1 Requirements

### Foundation — Irreversible Contracts

Everything here is cheap now and unbackfillable later. Four research documents independently placed this set first.

- [ ] **FND-01**: Every user-scoped table carries `user_id NOT NULL` from migration 001, with scoping enforced at the data-access layer rather than in route handlers
- [ ] **FND-02**: `users.id` is an opaque uuid (never email) and every user row carries an IANA timezone name (never a UTC offset)
- [ ] **FND-03**: Practice events are append-only with a client-computed `local_date` denormalized at write time; every streak and counter is derived on read, never stored as a mutable integer
- [ ] **FND-04**: No provider API key is reachable from the browser — enforced by a build-time server-only boundary, not by convention
- [ ] **FND-05**: Environment config is validated at boot and throws on a missing or malformed value
- [ ] **FND-06**: Every write carries a client-generated idempotency key so a retry cannot double-count an utterance or a streak day
- [ ] **FND-07**: The error taxonomy is a closed, checked-in set of `structure_key` values covering common Vietnamese-L1 English errors, defined before the first error is ever recorded
- [ ] **FND-08**: The conversation turn protocol carries `scaffold_options`, `frame`, and `expected_target` from the first turn ever written
- [ ] **FND-09**: The turn engine's input is `{text, source: 'typed'|'spoken', confidence?}` behind a `lib/speech` port, so spoken input is later additive rather than a rewrite
- [ ] **FND-10**: An eval harness of ~20 fixed learner turns asserts vocabulary coverage, sentence length, correction count, and assessor honesty

### Cost Control

- [ ] **COST-01**: Every AI provider call originates server-side and passes through exactly one budget choke point before spending
- [ ] **COST-02**: Spend is metered in money (`cost_micros`), not request count, and logged from the first request
- [ ] **COST-03**: Exceeding budget degrades the session through tiers (full → cheaper model → winding down → zero-cost activity) and never returns an error — a denied day is a broken streak
- [ ] **COST-04**: The budget window resets at the user's local midnight, the same boundary the streak uses
- [ ] **COST-05**: Retries are capped at 2 and debit the user's budget, so a client retry loop cannot run up an unbounded bill
- [ ] **COST-06**: A global kill-switch and a provider-account-level spend cap exist as backstops independent of the application's own limiter

### Conversation Loop

- [ ] **CONV-01**: A learner who types nothing can complete a full session by tapping only — the scaffold is the input mechanism, not a hint panel beside a text box
- [ ] **CONV-02**: The AI partner's reply and the scaffold frames come from one structured call, with the reply ordered first so speakable text arrives before UI chrome
- [ ] **CONV-03**: Only the last 3–4 turns plus a compact state summary are sent to the model, bounding drift, cost, and latency with one mechanism
- [ ] **CONV-04**: Model output is checked against a vocabulary allowlist and a deterministic validator, with capped regeneration and a hand-authored safe reply as the final fallback
- [ ] **CONV-05**: Error assessment runs as a separate call with no persona and no conversation history, fired in parallel the instant the utterance arrives
- [ ] **CONV-06**: The assessor call cannot fail a turn — it is fire-and-forget
- [ ] **CONV-07**: At most one correction per turn is surfaced in flow, selected by recurrence; it lives in a separate field from the AI's reply so a model failure degrades to no correction rather than a broken conversation
- [ ] **CONV-08**: Corrections elicit learner self-repair rather than silently modelling the right answer — low-proficiency learners read a recast as conversation continuing, not as correction
- [ ] **CONV-09**: Conversation state lives entirely in database rows; no server-side in-memory session state, so a dropped connection, a closed tab, or a redeploy resumes cleanly
- [ ] **CONV-10**: Scaffold level is computed in application code, never by a model call, and stored as a float that the UI buckets

### Listening

- [ ] **LISN-01**: The AI partner's turn is spoken aloud with server-side TTS on every turn
- [ ] **LISN-02**: Speech rate defaults to a slowed setting appropriate for a beginner, and the rate is itself a withdrawable scaffold
- [ ] **LISN-03**: Scaffold-frame audio is pre-generated and cached by content hash, since it is static per scenario
- [ ] **LISN-04**: Audio playback unlocks on the first tap of the session, satisfying iOS autoplay restrictions

### Habit

- [ ] **HBT-01**: The streak is credited on the learner's first produced utterance, server-side, roughly 45–75 seconds into a session — not on session completion
- [ ] **HBT-02**: A busy-day path is selected by the app rather than asked about, reachable in one tap, completing in under 60 seconds
- [ ] **HBT-03**: A zero-cost activity exists that qualifies for the streak while making no model calls — it serves the busy-day path, the budget-exhausted tier, and degraded connectivity as one component
- [ ] **HBT-04**: A missed day is never rendered as a loss or a reset; progress is shown as a non-decreasing count of days practiced
- [ ] **HBT-05**: Each session closes by showing the learner their own sentences from that session, dated

### Scaffolding Content

- [ ] **CONT-01**: Scenario 1's scaffold curriculum is hand-authored — which structures, in what order, with which ~800 words — and is a gating deliverable, not a task inside an engineering phase
- [ ] **CONT-02**: UI chrome and all explanations are in Vietnamese; English appears only in the target content

### Error Capture

- [ ] **ERR-01**: Errors are captured into the closed taxonomy from day one, silently, with no learner-visible review until enough patterns have matured
- [ ] **ERR-02**: An error instance is normalized to a `structure_key` rather than stored as raw text
- [ ] **ERR-03**: A pattern only becomes a review candidate after recurring at least 2–3 times across at least 2 separate sessions
- [ ] **ERR-04**: Review outcomes are appended to a log shaped so a scheduling algorithm can be adopted later from real history — this record is unbackfillable
- [ ] **ERR-05**: No unconfirmed transcript ever becomes a review item

### Access

- [ ] **ACC-01**: Every model-touching route requires authentication
- [ ] **ACC-02**: Public signup is disabled; accounts are seeded by the owner
- [ ] **ACC-03**: Each user sees only their own history, errors, and streak
- [ ] **ACC-04**: The app is deployed to a container host with managed Postgres, where `DATABASE_URL` is the only difference from local
- [ ] **ACC-05**: An HTTPS tunnel for phone testing is established before the phase that needs a secure context, not at the end

## v1.x Requirements

Deferred but planned. Tracked as post-v1 phases 6–8 in `ROADMAP.md`, visible but not part of the v1 commitment.

### Speaking

- **SPK-01**: Learner records a spoken response using push-to-talk, never automatic silence detection
- **SPK-02**: Recognition runs server-side behind the existing port, with `expected_target` passed as a decoding hint
- **SPK-03**: The transcript is always shown with a one-tap "that's not what I said" that discards the utterance from error capture and offers a text fallback
- **SPK-04**: Streak credit is granted the moment audio of sufficient duration is captured — before recognition returns and regardless of what it returns
- **SPK-05**: Recognition output never produces a verdict on correctness; when similarity is ambiguous, the app says nothing about accuracy and moves the conversation on
- **SPK-06**: The learner can drop to text mid-session in one tap
- **SPK-07**: A "that's not what I said" tap rate above ~15% means voice mode is not ready and is treated as a release gate

### Review & Withdrawal

- **REV-01**: A capped ~20-item priority watchlist scored by recency × recurrence × failure count, with displacement when full
- **REV-02**: Review happens inside conversation by biasing scenario and frame selection, never as a flashcard deck screen
- **REV-03**: No due dates, no review debt, and no visible "N due" count anywhere in the UI
- **REV-04**: Patterns auto-retire after 3 clean productions, or after 5 failures with a note in the recap that a human should look at it
- **REV-05**: Scaffold withdrawal is contingent and per language function, so a learner can be unsupported at greetings and fully supported at past-tense narration simultaneously
- **REV-06**: Withdrawal is asymmetric — demotion is fast and undamped, promotion is slow and confidence-weighted
- **REV-07**: A Vietnamese end-of-session recap gives three bullets: what you said, what's standard, why
- **REV-08**: Streak repair is automatic, retroactive, free, roughly twice a month, and never a purchasable currency
- **REV-09**: A scaffold-position progress view shows standing per scenario in plain language

### Content Scale-Out

- **SCALE-01**: Scenarios 2–10 authored
- **SCALE-02**: A then-vs-now transcript pair showing the learner's own early and recent utterances
- **SCALE-03**: Retired-pattern count and longest unscaffolded turn as progress signals
- **SCALE-04**: At most one daily notification, learner-set, deep-linked to a sub-60-second activity, and only after 30 days of real use proves it is needed

## Out of Scope

| Feature | Reason |
|---------|--------|
| Pronunciation scoring, phoneme feedback | ~22.9% false-reject rate aimed at a learner with a four-for-four abandonment record. Excluded from every version, not just v1 |
| Realtime speech-to-speech API | 8–50x cost, open-tab billing risk, and it destroys the clean transcript that error capture and review are built on |
| FSRS / SM-2 / any interval algorithm | Resolved against in research — its benchmark advantage needs preconditions that all fail here, and it fixes none of the four actual problems. The capped watchlist replaces it |
| Words-learned, accuracy-%, time-studied, XP, hearts, leagues | "Words learned" is the exact metric that produced "knows words but can't produce them"; accuracy-% falls as the learner improves, because fading the scaffold increases errors by design |
| Auto-VAD turn endpointing | Tuned to 500–800ms silence for native speakers; a beginner pauses 2–4s mid-word-retrieval and gets cut off |
| Serverless hosting | Breaks the single always-on process that the budget counter and streaming response assume |
| CRDT / sync engine (ElectricSQL, Automerge, Yjs) | "Local-first" here means localhost-first. Five users reading only their own rows on one device — the largest over-engineering risk in the project |
| Scheduler, queue, Redis, object storage | Streaks, due-ness, and budget windows are all computed on read. Nothing needs to run at midnight |
| Social features, shared leaderboards, friend streaks | Owner explicitly chose private isolated data |
| Per-user API keys | Too high a barrier for friends in a habit app |
| Exam prep (IELTS/TOEIC) | Goal is conversational ability, not a score with a deadline |
| Learner-supplied content, scenario-authoring UI | Fails on lazy days when nothing gets pasted; beginner level cannot absorb native-speed material |
| CEFR level estimation, onboarding placement test | Adds a judgment the learner did not ask for at the moment they are most likely to leave |
| Offline AI conversation | The zero-cost activity covers degraded connectivity without it |
| Monetization, payments, public signup | Personal tool for owner plus friends |
| Native mobile apps | Web only; mobile browser is sufficient reach |

## Traceability

Mapped during roadmap creation (`.planning/ROADMAP.md`, 2026-10-06). Every v1 requirement appears in exactly one phase.

### v1

| Requirement | Phase | Status |
|-------------|-------|--------|
| FND-01 | Phase 1 — Foundation & Irreversible Contracts | Pending |
| FND-02 | Phase 1 — Foundation & Irreversible Contracts | Pending |
| FND-03 | Phase 1 — Foundation & Irreversible Contracts | Pending |
| FND-04 | Phase 1 — Foundation & Irreversible Contracts | Pending |
| FND-05 | Phase 1 — Foundation & Irreversible Contracts | Pending |
| FND-06 | Phase 1 — Foundation & Irreversible Contracts | Pending |
| FND-07 | Phase 1 — Foundation & Irreversible Contracts | Pending |
| FND-08 | Phase 1 — Foundation & Irreversible Contracts | Pending |
| FND-09 | Phase 1 — Foundation & Irreversible Contracts | Pending |
| FND-10 | Phase 1 — Foundation & Irreversible Contracts | Pending |
| COST-01 | Phase 1 — Foundation & Irreversible Contracts | Pending |
| COST-02 | Phase 1 — Foundation & Irreversible Contracts | Pending |
| COST-03 | Phase 1 — Foundation & Irreversible Contracts | Pending |
| COST-04 | Phase 1 — Foundation & Irreversible Contracts | Pending |
| COST-05 | Phase 1 — Foundation & Irreversible Contracts | Pending |
| COST-06 | Phase 1 — Foundation & Irreversible Contracts | Pending |
| CONT-01 | Phase 2 — Scaffold Curriculum, Scenario 1 | Pending |
| CONT-02 | Phase 2 — Scaffold Curriculum, Scenario 1 | Pending |
| CONV-01 | Phase 3 — The Text Loop, One Scenario | Pending |
| CONV-02 | Phase 3 — The Text Loop, One Scenario | Pending |
| CONV-03 | Phase 3 — The Text Loop, One Scenario | Pending |
| CONV-04 | Phase 3 — The Text Loop, One Scenario | Pending |
| CONV-05 | Phase 3 — The Text Loop, One Scenario | Pending |
| CONV-06 | Phase 3 — The Text Loop, One Scenario | Pending |
| CONV-07 | Phase 3 — The Text Loop, One Scenario | Pending |
| CONV-08 | Phase 3 — The Text Loop, One Scenario | Pending |
| CONV-09 | Phase 3 — The Text Loop, One Scenario | Pending |
| CONV-10 | Phase 3 — The Text Loop, One Scenario | Pending |
| HBT-01 | Phase 3 — The Text Loop, One Scenario | Pending |
| HBT-02 | Phase 3 — The Text Loop, One Scenario | Pending |
| HBT-04 | Phase 3 — The Text Loop, One Scenario | Pending |
| HBT-05 | Phase 3 — The Text Loop, One Scenario | Pending |
| LISN-01 | Phase 4 — Listening, The Partner Speaks | Pending |
| LISN-02 | Phase 4 — Listening, The Partner Speaks | Pending |
| LISN-03 | Phase 4 — Listening, The Partner Speaks | Pending |
| LISN-04 | Phase 4 — Listening, The Partner Speaks | Pending |
| ACC-05 | Phase 4 — Listening, The Partner Speaks | Pending |
| ACC-01 | Phase 5 — Make It Real | Pending |
| ACC-02 | Phase 5 — Make It Real | Pending |
| ACC-03 | Phase 5 — Make It Real | Pending |
| ACC-04 | Phase 5 — Make It Real | Pending |
| ERR-01 | Phase 5 — Make It Real | Pending |
| ERR-02 | Phase 5 — Make It Real | Pending |
| ERR-03 | Phase 5 — Make It Real | Pending |
| ERR-04 | Phase 5 — Make It Real | Pending |
| ERR-05 | Phase 5 — Make It Real | Pending |
| HBT-03 | Phase 5 — Make It Real | Pending |

**Notes on two placements that moved from `research/SUMMARY.md`'s ordering:**
- **ACC-05** (HTTPS tunnel) sits in Phase 4, not Phase 5. Its own wording is "established before the phase that needs a secure context, not at the end" — with TTS in v1, Phase 4 is the first phase to touch a real phone, so the tunnel is built there.
- **HBT-03** (zero-cost activity) sits in Phase 5, per SUMMARY's correction of ARCHITECTURE: its content source is swappable, the component is not, so it is not blocked behind a review system that intentionally has no data yet.

### v1.x

| Requirement | Phase | Status |
|-------------|-------|--------|
| REV-01 … REV-09 | Phase 6 — Watchlist Review & Scaffold Withdrawal | Deferred (v1.x) |
| SPK-01 … SPK-07 | Phase 7 — Speaking, Push-to-Talk Input | Deferred (v1.x, gated on Spike 1) |
| SCALE-01 … SCALE-04 | Phase 8 — Content Scale-Out & Long-Tail | Deferred (v1.x) |

**Coverage:**
- v1 requirements: 47 total
- Mapped to phases: 47 ✓
- Unmapped: 0 ✓
- Duplicated across phases: 0 ✓
- v1.x requirements: 20 total, all mapped to post-v1 phases 6–8

---
*Requirements defined: 2026-10-06*
*Last updated: 2026-10-06 after roadmap creation — traceability and coverage filled*
