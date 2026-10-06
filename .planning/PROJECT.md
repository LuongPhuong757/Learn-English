# LearnEnglish

## What This Is

A personal web app for learning English, built around scaffolded AI conversation. The user is a beginner ("mất gốc") whose goal is spoken communication, and whose real obstacle is not lack of material but lack of a loop short enough to return to every single day. Each session the learner picks voice or text depending on their situation; at beginner level the AI supplies sentence frames to adapt rather than demanding free production, and the scaffolding is progressively withdrawn as the learner's output improves. Built for the owner and a handful of friends — each with their own private data, no social features.

## Core Value

Every single day the learner opens the app and successfully produces at least one English utterance — spoken or typed. If everything else fails, this must not.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] Learner holds a guided conversation with an AI partner in a chosen scenario
- [ ] Learner chooses voice or text mode per session; both reach the same core loop
- [ ] At beginner level the AI supplies sentence frames to adapt rather than demanding free production
- [ ] Scaffolding is progressively withdrawn as the learner's output improves
- [ ] AI corrects errors in-conversation without breaking conversational flow
- [ ] Recurring errors are captured and resurfaced for spaced review in later sessions
- [ ] A "busy day" path exists: the app decides the shortest high-value activity and it still counts toward the streak
- [ ] Session length adapts to available time rather than assuming a fixed daily budget
- [ ] Daily streak is tracked and visible as the primary progress signal
- [ ] Each user has private, isolated data (own history, own errors, own streak)
- [ ] Runs locally first; deployable to the web without re-architecture
- [ ] Shared API key is rate-limited per user to bound cost

### Out of Scope

- Social features (shared leaderboards, seeing friends' streaks) — user explicitly chose private, isolated data; adds work without serving the core loop
- Per-user API keys — owner pays via one shared key; asking friends to obtain keys is too high a barrier for a habit app
- Certification exam prep (IELTS/TOEIC) — goal is conversational ability, not a score with a deadline
- Learner-supplied content (pasting YouTube links/articles to mine for vocabulary) — considered and rejected as the v1 spine; it fails on lazy days when nothing gets pasted, and beginner level can't absorb native-speed native material
- Standalone vocabulary drilling detached from conversation — the user already reports "knows words but can't produce them"; isolated word study is the cause, not the cure
- Monetization, payments, public signup — personal tool for owner plus friends
- Native mobile apps — web only; mobile browser is sufficient reach

## Context

**Learner profile:** Beginner level across all four skills (vocabulary retention, listening to native-speed speech, speaking fluency, natural writing). Self-reports as "mất gốc." Destination is being able to communicate in English, not a test score.

**Failure history:** No fixed system or tool to date — studies when motivated, with no measurable progress. Has abandoned previous attempts. This is the central risk the product must design against.

**The design tension that shaped this project:** Free English content is abundant; what is missing is a loop short enough to sustain. The learner chose AI conversation as the content source, which matches the destination (speaking) but is a trap at beginner level — an open prompt with an empty input box produces nothing and triggers abandonment by day three. Abandoning conversation is equally wrong: decontextualized vocabulary study is precisely why the learner "knows words but can't produce them." The resolution is conversation at the center with scaffolding: the AI supplies usable sentence frames early, and withdraws them as production improves.

**Availability:** Study time is irregular — some days five minutes, some days an hour. The daily loop must scale down without breaking the habit signal, because a broken streak on a busy day is a known abandonment trigger.

**Users:** Owner plus a small number of friends. Each has isolated data. No social or comparative features requested — explicitly declined.

## Constraints

- **Deployment**: Local-first, web deployment later — v1 runs on the owner's machine for speed of iteration, but must not be architected in a way that blocks deploying to a URL, since daily habit realistically requires phone access
- **Cost**: Single shared API key paid by the owner — per-user rate limiting is required to bound spend, since the core loop calls an AI model every session
- **Modality**: Both voice and text input must be supported — the learner selects per session based on their surroundings
- **Audience size**: Handful of users — no scaling, no multi-tenancy complexity, but real per-user data isolation is required
- **Tech stack**: Not yet decided — deferred to the research phase

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| AI conversation as the product spine, not vocabulary drilling | Destination is spoken communication; decontextualized word study is the diagnosed cause of "knows words but can't produce them" | — Pending |
| Scaffolded production (AI supplies frames) at beginner level, withdrawn over time | Unscaffolded conversation at "mất gốc" level yields an empty input box and abandonment by day three | — Pending |
| Habit retention, not skill coverage, is the Core Value | Learner is weak in all four skills but named daily consistency as the goal; covering all four thinly per session is what makes sessions long and causes quitting | — Pending |
| Busy-day path where the app chooses the shortest high-value activity | Study time is irregular; a streak broken on a busy day is a known abandonment trigger | — Pending |
| Both voice and text input, learner-selected per session | Voice matches the destination but is intimidating for a beginner; text lowers the barrier on hard days | — Pending |
| Private per-user data, no social features | Explicitly chosen over a shared streak board | — Pending |
| Local-first, deployable later | Faster v1 iteration; deployment deferred but not architecturally foreclosed | — Pending |
| Shared owner-funded API key with per-user rate limits | Per-user key acquisition is too high a barrier for friends in a habit app | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-10-06 after initialization*
