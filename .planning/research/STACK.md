# Stack Research

**Domain:** Personal AI language-learning web app (scaffolded conversation, voice + text, SRS, streaks)
**Researched:** 2026-10-06
**Confidence:** HIGH on architecture shape and persistence; HIGH on pricing (official vendor pages); MEDIUM on exact framework minor versions (pin at scaffold time); MEDIUM-HIGH on STT accuracy claims for Vietnamese-accented English (benchmark literature is about accented English generally, not Vietnamese specifically — see Gaps).

---

## The One Decision That Decides Everything Else

**A single always-on Node process with a local SQLite file on disk — never serverless.**

PROJECT.md says "deployable to the web without re-architecture." The only way to break that promise is to pick a runtime shape that can't hold a file. Vercel/Netlify/Cloudflare Workers functions have ephemeral or no filesystem, so SQLite-on-disk dies on deploy and you are forced into a managed Postgres migration — a rewrite of the data layer at exactly the wrong moment. Choose the process shape in Phase 1 and every other decision in this document becomes reversible.

Everything below assumes: Node 22 LTS, one process, one SQLite file, deployed later to Fly.io (machine + persistent volume) or a €4–6/mo Hetzner/VPS box.

---

## Recommended Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| **Node.js** | 22 LTS | Server runtime | Single long-lived process that can hold a SQLite file handle, a WebSocket, and an in-memory rate-limit cache. LTS through 2027. |
| **SvelteKit** | 2.x (Svelte 5) | Full-stack web framework | One codebase for UI + API routes + server actions; `adapter-node` emits a plain `node build` artifact that runs identically on the Mac and on a VPS. Smaller surface area than Next.js for a solo dev, and no React Server Component mental overhead. **Not load-bearing** — see escape hatch below. |
| **TypeScript** | 5.x | Language | The error-log / SRS / scaffold-state schemas are the product; types catch drift between the LLM's structured output and the DB. |
| **SQLite** (`better-sqlite3`) | 11.x / Node 22 `node:sqlite` | Primary datastore | Synchronous, zero-daemon, single file. At 5 users the entire DB is a few MB. Copy the file to deploy. Survives the move to a URL untouched. |
| **Drizzle ORM** | 0.4x | Schema, migrations, queries | SQL-first (you can read the generated SQL), real migration files, and — critically — a `libsql` driver, so swapping local SQLite → Turso later is a connection-string change, not a rewrite. |
| **Claude Sonnet 5.5** | `claude-sonnet-5-5` | Conversation turn | The tier where a long scaffolding spec in the system prompt is followed reliably. At 150 sessions/mo this costs ~$10 — cost is not the binding constraint, pedagogical quality is. |
| **Claude Opus 5.5** | `claude-opus-5-5` | Post-session error analysis + SRS item extraction | Runs once per session on a ~2.5K-token transcript. ~$0.026/session, ~$4/mo. This output compounds into every future session — spend here, not on the chat turn. |
| **Claude Haiku 4.5** | `claude-haiku-4-5` | Busy-day micro-activity, on-task classification, cheap reruns | $1/$5 per MTok. Used for the sub-60-second path where latency beats nuance. |
| **Deepgram Nova-3** | `nova-3` (streaming or pre-recorded REST) | Speech-to-text | **Keyterm prompting** (up to 100 phrases per request) is the decisive feature: you already know the target sentence frame the learner is adapting, so you can bias the recogniser toward the exact vocabulary expected. That converts an open-vocabulary accented-speech problem into a nearly constrained one. $0.0043/min batch, $0.0048/min streaming. |
| **OpenAI `gpt-4o-mini-tts`** | current | Text-to-speech | ~$0.015/min, and it accepts a free-text `instructions` parameter for speaking style — you can literally ask for "slow, clearly articulated, beginner-friendly pacing." No other TTS at this price lets you tune delivery, and delivery *is* the listening lesson. |
| **Better Auth** | 1.x | Auth + sessions | Self-hosted, Drizzle + SQLite adapter, httpOnly cookie sessions, email+password with signup disabled. See Auth section for why this is the minimum that isn't insecure. |

### Supporting Libraries

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `@anthropic-ai/sdk` | latest | Claude client | Every LLM call. Use prompt caching (`cache_control: {type:"ephemeral"}`) on the system prompt + scaffold library — cache reads are $0.20/MTok on Sonnet 5.5 vs $2.00 input, cutting conversation cost ~60–70%. |
| `@deepgram/sdk` | 3.x | STT client | Behind an `STT` port, not called directly from route handlers. |
| `openai` | 5.x | TTS client (and STT fallback) | Behind a `TTS` port. |
| `zod` | 4.x | Schema validation | Validate every LLM structured output before it touches the DB. An error-log row written from a hallucinated shape poisons SRS forever. |
| `drizzle-kit` | 0.3x | Migration generation | `drizzle-kit generate` + checked-in SQL migrations. Non-negotiable for the local→deploy move. |
| `tailwindcss` | 4.x | Styling | Solo dev, no design system needed, the UI is ~6 screens. |
| `date-fns` or `Temporal` | — | Streak date math | Streaks are timezone bugs waiting to happen. Store the user's IANA timezone and compute "day" in it, not UTC. |
| `ts-fsrs` | 5.x | Spaced repetition scheduling | FSRS is the current standard and strictly better-calibrated than hand-rolled SM-2. Don't write your own interval math. |
| `litestream` | 0.3.x (binary, not npm) | Continuous SQLite backup | Add at deploy time, not before. Streams WAL to S3/R2 for ~$0.10/mo. |

### Development Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| `vite` (bundled with SvelteKit) | Dev server, HMR | Nothing to configure. |
| **`cloudflared tunnel` or `tailscale serve`** | HTTPS to the dev box from a phone | **Load-bearing for the voice phase.** `getUserMedia` requires a secure context. `localhost` is secure, `http://192.168.x.x` is not — so you physically cannot test microphone capture on an iPhone against the dev server without a tunnel. Set this up the day you start voice work, not at deploy. |
| `drizzle-kit studio` | DB browsing | Faster than writing admin UI for 5 users. |
| `vitest` | Tests | Test the rate limiter and the SRS scheduler. Don't test the LLM. |

## Installation

```bash
# Scaffold
npm create svelte@latest learnenglish   # SvelteKit 2 + TS + adapter-node

# Core
npm install better-sqlite3 drizzle-orm zod ts-fsrs
npm install @anthropic-ai/sdk @deepgram/sdk openai
npm install better-auth

# Dev
npm install -D drizzle-kit @types/better-sqlite3 vitest tailwindcss @tailwindcss/vite
```

---

## Per-Question Answers

### 1. Web framework / runtime

**Recommendation: SvelteKit 2 (Svelte 5) on Node 22 LTS with `adapter-node`.** Cost: $0.

Why: a solo developer needs the fewest moving parts between "it works on my Mac" and "it works at a URL." `adapter-node` produces a single `build/index.js` you run with `node`. No edge runtime, no serverless cold-start, no RSC/client-boundary debugging.

**Explicit escape hatch — the framework is NOT load-bearing.** If you already know React, use **Next.js 15 with `output: "standalone"` and the Node runtime**, and nothing else in this document changes. What *is* load-bearing is that you do not deploy to Vercel's serverless functions, because SQLite-on-disk will not survive there. If you pick Next.js, pick a container host (Fly.io, Railway, VPS) at the same time.

**Against the popular choice:** the default 2026 reflex is "Next.js on Vercel." For this project that is wrong — it silently converts "deployable later" into "forced Postgres migration later," which is exactly the re-architecture PROJECT.md forbids.

### 2. Speech input (STT)

**Recommendation: Deepgram Nova-3 with keyterm prompting, called server-side from a complete uploaded clip.** Cost: $0.0043/min pre-recorded, $0.0048/min streaming (Pay-As-You-Go); $200 free signup credit. ([pricing](https://deepgram.com/pricing))

**Against the obvious choice — do not use the browser Web Speech API.** It is free and it is the wrong tool here, for four compounding reasons:

1. **Support is not universal.** Firefox: not supported, disabled by default across all versions. Edge: no support. Safari iOS: partial from 14.5+. Chrome Android: partial. Global coverage ~88%, and the 12% includes the Firefox users you can't control. ([caniuse](https://caniuse.com/speech-recognition))
2. **Mobile Safari needs workarounds** — the API must be driven from a real user gesture inside Safari itself, and does not work reliably inside PWAs or WebViews. ([taming the Web Speech API](https://webreflection.medium.com/taming-the-web-speech-api-ef64f5a245e1))
3. **Accuracy is a black box you cannot tune.** Chrome ships the audio to Google's cloud recogniser; you get no vocabulary biasing, no confidence thresholds, no model choice. For a Vietnamese beginner, misrecognition is *the* abandonment risk — the app confidently showing "I goat to school" destroys the one thing the Core Value protects.
4. **It splits your pipeline.** Browser STT returns text only in the browser; you still need server-side audio for any pronunciation feedback later.

**Accuracy on heavily-accented non-native English — the honest picture.** Published comparisons put Whisper at roughly 8–12% WER on accented English versus Deepgram Nova-2 at 12–18%, i.e. Whisper-family models have historically been more robust to accent because of their 680k-hour noisy training set. ([comparison](https://www.callmissed.com/blog/deepgram-nova-vs-whisper-large-v3-turbo)) Nova-3 closed much of that gap — Deepgram reports a 47.4% batch / 54.3% streaming WER reduction (median 5.26% / 6.84%) and claims it outperforms Whisper across seven languages. ([Nova-3 launch](https://deepgram.com/learn/introducing-nova-3-speech-to-text-api)) Independent coverage still notes Nova-3 "loses ground on noisy or heavily accented audio." ([analysis](https://convertaudiototext.com/blog/deepgram-nova-3-explained))

So raw WER is roughly a wash. **Keyterm prompting breaks the tie.** Nova-3 accepts up to 100 keyterms per request (500 tokens multilingual) with no retraining; one reported deployment went from 10% recognition of critical domain terms to a 625% improvement. In this app you *always* know the expected answer space — the scaffold frame you just handed the learner. Feed its content words as keyterms on every request. That is a structural accuracy advantage no general-purpose model gives you, and the Web Speech API cannot offer at all.

**Fallback / A-B alternative: OpenAI `gpt-4o-transcribe` at $0.006/min, or `gpt-4o-mini-transcribe` at $0.003/min** ([OpenAI pricing](https://developers.openai.com/api/docs/pricing)). These also accept a `prompt` for steering, and keep you on one vendor. Put both behind the same `STT` port and switch with an env var.

**Latency:** a 5-second clip through Deepgram pre-recorded REST returns in roughly 300–600ms. That is invisible next to the LLM turn.

**Mandatory product constraint, not optional:** always render the transcript and always let the learner tap to correct or re-record before the AI responds. Never let a misrecognition become a logged "error" in the SRS. This single UI rule converts STT accuracy from an existential risk into an annoyance.

**Cost at 5 users:** ~3 min of learner speech per session × 150 sessions = 450 min/mo = **~$2/month.**

### 3. Speech output (TTS)

**Recommendation: OpenAI `gpt-4o-mini-tts`, server-side, with aggressive caching of generated audio.** Cost: $12/1M audio output tokens ≈ **$0.015/min**. ([OpenAI pricing](https://developers.openai.com/api/docs/pricing))

**Against the obvious choice — do not use browser `SpeechSynthesis` as the primary.** It is free and it fails the one user who matters:
- iOS Safari's exposed voices are the low-quality "Eloquence"-class set; `getVoices()` does not return the high-quality system voices, and recent iOS versions removed some that were previously reachable. ([Apple developer forum](https://developer.apple.com/forums/thread/723503), [WebKit bug 290497](https://bugs.webkit.org/show_bug.cgi?id=290497))
- The voice list differs per device and OS version, so the learner's pronunciation model is non-deterministic — they hear a different "teacher" on their phone than on the laptop.

**Does voice quality actually matter for a beginner?** Yes, and more than for an advanced learner, for a specific reason: a beginner has no internal model to correct against. An advanced learner hears a robotic voice and mentally normalises it; a "mất gốc" learner encodes whatever they hear as the target. Mispronounced or flat-prosody TTS actively teaches wrong listening. This is the one place where paying beats free.

**Why `gpt-4o-mini-tts` specifically over ElevenLabs:** ElevenLabs Flash v2.5 is faster (~75ms generation, ~197ms median time-to-first-audio in streaming benchmarks) and is the right answer for a real-time voice agent, but costs $50/1M characters ≈ $0.045–0.09/min — 3–6× more — and the latency advantage is irrelevant here because the AI's reply is pre-composed text, not a live duplex stream. ([latency benchmarks](https://gradium.ai/content/best-low-latency-tts-apis-2026), [TTS API comparison](https://techsy.io/en/blog/best-tts-apis-developers)) Deepgram Aura-2 at $0.030/1k chars is 2× OpenAI with no style control.

The decisive feature is the `instructions` parameter: "Speak slowly and clearly, as if to a beginner English learner. Separate words distinctly." You can also lower speaking rate as a withdrawable scaffold — pace up as the learner improves, in the same way sentence frames withdraw. That is a product feature, not a config knob.

**Cache every generated clip by `hash(text + voice + instructions)` in a local directory.** Scaffold frames, prompts, and corrections repeat heavily across sessions and across all 5 users. Real cost will land well under the estimate.

Keep browser `SpeechSynthesis` wired as an offline/degraded fallback only.

**Cost at 5 users:** ~1,000 characters (~70s) of AI speech per session × 150 = ~$2.70/mo before caching, likely **~$1.50/month** after.

### 4. LLM provider and model tier

**Three tiers, because the call sites have genuinely different economics.**

| Call site | Model | Price (in/out per MTok) | Frequency | Rationale |
|---|---|---|---|---|
| Conversation turn + inline scaffold | `claude-sonnet-5-5` | $2.00 / $10.00 | ~12×/session | Reliable instruction-following on a long scaffolding spec; low enough latency for a chat UI. Cache the system prompt (reads $0.20/MTok). |
| Post-session error analysis → error log + SRS items | `claude-opus-5-5` | $4.00 / $20.00 | 1×/session | Runs offline on a ~2.5K-token transcript. Its output feeds every future session. Highest leverage per dollar in the whole system. |
| Busy-day micro-activity, on-task classification | `claude-haiku-4-5` | $1.00 / $5.00 | occasional | Latency over nuance; the 60-second path must feel instant. |

**Monthly estimate — 5 users × 1 session/day × 30 days = 150 sessions.**

| Component | Per session | Monthly (150) |
|---|---|---|
| Conversation (Sonnet 5.5, ~25K in / 2K out, with caching) | ~$0.025–0.07 | $4 – $10 |
| Error analysis (Opus 5.5, 2.5K in / 800 out) | ~$0.026 | ~$4 |
| Busy-day / classification (Haiku 4.5) | ~$0.003 | ~$0.50 |
| STT (Deepgram Nova-3, ~3 min) | ~$0.014 | ~$2 |
| TTS (gpt-4o-mini-tts, ~70s, cached) | ~$0.010 | ~$1.50 |
| **Total** | **~$0.08–0.12** | **~$12–18/month (~$3/user)** |

**The decision this changes:** at this volume, the gap between the cheapest viable models and the best ones is about $10/month. **Cost is not the binding constraint — pedagogical quality is.** Do not optimise the conversation turn down to a nano-tier model to save $8. Spend the per-user rate limit on generosity and cap runaway loops instead.

**Cross-vendor alternatives, for honesty:** `gpt-5-mini` at $0.25/$2.00 and Gemini 3 Flash at $0.50/$3.00 are materially cheaper per token than Sonnet 5.5 ([pricing comparison](https://anotherwrapper.com/llm-pricing/gemini-3-flash-preview)). They are legitimate choices and would cut the conversation line to ~$2/mo. Keep the LLM behind a port so you can A/B them on real transcripts. The recommendation is Claude because the system prompt here is unusually long and rule-heavy (scaffold level, withdrawal policy, correct-without-breaking-flow) and instruction adherence under that load is what you are buying.

**Against the obvious choice — do not build on a realtime speech-to-speech API.** "It's a voice conversation app, use the Realtime API" is the reflex, and it's wrong three ways:
1. **Cost.** OpenAI `gpt-realtime` runs roughly $0.06–0.11/min realistically, worst case $0.46/min uncached — a 10-minute session is $0.60–$1.10, i.e. **$90–165/month** for 5 users, 8× the entire pipeline budget. Gemini Live is far cheaper (~$0.005/min in, ~$0.018/min out ≈ $0.09/session) and would be affordable, but shares the problems below. ([OpenAI Realtime pricing analysis](https://www.forasoft.com/article/openai-realtime-api-pricing), [Gemini Live pricing](https://tokenkarma.app/blog/gemini-live-api-pricing-voice-agents-2026/))
2. **You lose the artifact the product is built on.** The error log and SRS need a clean, inspectable text record of exactly what the learner produced. Speech-to-speech hides that inside the model.
3. **It breaks a stated requirement.** PROJECT.md requires "voice OR text, both reach the same core loop." A realtime audio model gives you one loop for voice and forces a second, different implementation for text. The STT → LLM → TTS pipeline gives you *one* loop with swappable I/O adapters — which is literally the requirement.

### 5. Data persistence

**Recommendation: SQLite (one file) + Drizzle ORM.** Cost: $0 local; ~$5/mo for a VPS/Fly machine with a volume when deployed; ~$0.10/mo for Litestream backups to R2.

Tables, all keyed by `user_id` from the very first migration: `users`, `sessions` (auth), `conversations`, `messages`, `error_log`, `srs_items` (FSRS state: due, stability, difficulty, reps, lapses), `daily_activity` (streak), `usage_ledger` (rate limiting).

**Why not Postgres:** it's the "grown-up" answer and it's wrong at this size. It adds a daemon to run and back up on the Mac, a connection pool to tune, and a second environment to keep in sync — for zero benefit at 5 users and maybe 50MB of data. Reach for it only if you ever need concurrent writes from multiple processes, which this app will never have.

**Why not a hosted BaaS (Supabase/Firebase):** both make the *local-first* half harder, not easier — you're either online-dependent during development or running a Docker compose stack to emulate. Supabase's RLS is genuinely nice for per-user isolation, but you can get the same guarantee with a `WHERE user_id = ?` discipline enforced by a repository layer, and you avoid a vendor in the critical path of a habit app.

**The deploy-later escape hatch:** Drizzle's `libsql` driver means that if you ever *do* need a managed/serverless-compatible store, switching to **Turso/libSQL** is a driver + connection-string change, not a data-model rewrite. libSQL embedded replicas (local file that syncs to a cloud primary) are production-mature as of 2026 and are the exact shape this project would want if it grew. ([Turso/libSQL 2026](https://noqta.tn/en/blog/turso-libsql-distributed-sqlite-edge-database-2026)) Don't adopt it now — just don't foreclose it, which Drizzle handles for you.

**Non-negotiable from day one:** checked-in migration files (`drizzle-kit generate`), and `user_id NOT NULL` with an index on every user-scoped table. Adding isolation later means a data migration plus touching every query.

### 6. Auth

**Recommendation: Better Auth 1.x, email + password, `emailAndPassword.disableSignUp: true`, accounts seeded by the owner with a CLI script.** Cost: $0.

This is genuinely the least work that is not insecure:
- No email provider to configure (no magic links, no verification, no password reset — the owner resets via script for 5 people).
- No OAuth app registration, no redirect URIs, no consent screen.
- Sessions in SQLite, httpOnly + Secure + SameSite cookies, handled by the library.
- Drizzle + SQLite adapter already exists, so it shares your one DB file.
- Scales to real signup/OAuth/passkeys later by flipping config, if friends grow to strangers.

**What's actually insecure and should be rejected despite being less work:** HTTP Basic Auth or a single shared password gives you no `user_id`, so per-user isolation, per-user rate limits, and per-user streaks all become impossible — it fails three PROJECT.md requirements, not just the security one. A hardcoded user list in env vars with hand-rolled cookie signing is the other tempting shortcut; don't — session fixation and timing-safe comparison are easy to get wrong and Better Auth is ~20 lines of setup.

**Against stale 2024-era advice: Lucia is deprecated.** Its maintainer converted it from a library into a learn-to-build-your-own-auth resource, and Auth.js/NextAuth has folded into Better Auth. Any tutorial recommending Lucia v3 is stale. ([comparison](https://www.pkgpulse.com/guides/better-auth-vs-lucia-vs-nextauth-2026), [self-hosted auth 2026](https://dev.to/noorix1/self-hosted-nodejs-authentication-in-2026-9ao))

**Per-user rate limiting rides on auth, and belongs in the same layer.** A `usage_ledger` table (user_id, date, provider, tokens_in, tokens_out, cost_cents) plus a server-side check before every provider call. ~40 lines of SQL. Do **not** bring in Redis or a rate-limit library — at 5 users, a SQL `SUM()` over the current month is the whole implementation. Enforce a monthly dollar cap *and* a per-session turn cap (the latter is what actually protects you from a runaway loop).

### 7. Audio handling in the browser

**Recommendation: `getUserMedia` + `MediaRecorder`, record the complete utterance, upload the blob, transcribe server-side. No streaming in v1.**

**Against the obvious choice — don't build streaming STT first.** Streaming over a WebSocket is the "real" architecture and it's wrong for v1: a beginner's utterance is 2–8 seconds, and a complete-clip round trip is ~300–600ms, imperceptible next to the LLM turn. Streaming costs you a persistent connection, reconnect logic, partial-result UI, and a second failure mode — for a latency win that changes nothing the learner perceives. Keep the `STT` port shaped so streaming can be added later without touching the conversation loop.

**Format detection is mandatory — there is no single MIME type that works everywhere.**

| Browser | Works |
|---|---|
| Chrome 126+ / Firefox | `audio/webm;codecs=opus` (48kHz) |
| Safari iOS 14.5–18.3 | `audio/mp4` (AAC) — **no webm** |
| Safari iOS 18.4+ | webm + Opus now supported, but still detect |

Loop `MediaRecorder.isTypeSupported()` over a preference list and send the container type to the server; Deepgram and OpenAI both accept webm/opus and mp4/aac. ([WebKit MediaRecorder](https://webkit.org/blog/11353/mediarecorder-api/), [cross-browser recording notes](https://blog.addpipe.com/recording-audio-in-the-browser-using-pure-html5-and-minimal-javascript.md))

**Four iOS gotchas that will silently break the app and are hard to debug:**
1. **Secure context required.** `getUserMedia` needs HTTPS. `localhost` counts; `http://192.168.1.x` does not. You cannot test voice on your phone against the dev server without a tunnel (`cloudflared tunnel` / `tailscale serve`). Budget this into the voice phase.
2. **First audio playback needs a user gesture.** If the AI's TTS reply is the first `<audio>` play in the page's life, iOS blocks it with no error. Prime a muted `<audio>` element inside the "Start session" click handler, then reuse that element for all playback.
3. **Stop the tracks.** `stream.getTracks().forEach(t => t.stop())` after recording, or the orange mic indicator stays lit and users assume they're being recorded.
4. **Permission prompt timing.** Request mic access on an explicit "Speak" button, never on page load — iOS denials are sticky and a denied prompt on day one kills the voice path permanently for that user.

---

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| SvelteKit 2 + `adapter-node` | Next.js 15 `output: standalone` | You already know React well. Nothing else in this doc changes — but you must still pick a container host, not Vercel serverless. |
| SQLite file | Turso / libSQL | Only if you later need serverless deploy or multi-region. Drizzle makes it a driver swap. |
| SQLite file | Postgres (Neon/Supabase) | Only if concurrent writers from multiple processes become real. Not at 5 users. |
| Deepgram Nova-3 | OpenAI `gpt-4o-transcribe` ($0.006/min) | You want one vendor and one API key. Accuracy is comparable; you lose keyterm prompting (the `prompt` param is weaker biasing). |
| Deepgram Nova-3 | Self-hosted `whisper.cpp` on the Mac | Zero marginal cost and full privacy, M-series Macs run large-v3-turbo fine. **Stops working the moment you deploy to a small VPS** — which is the architectural trap this project is explicitly trying to avoid. Use only if you decide local-only is permanent. |
| `gpt-4o-mini-tts` | ElevenLabs Flash v2.5 | You move to true realtime duplex conversation and need <200ms TTFA. Accept 3–6× cost. |
| `claude-sonnet-5-5` | `gpt-5-mini` ($0.25/$2.00) or Gemini 3 Flash ($0.50/$3.00) | You want to cut the LLM line from ~$10 to ~$2/mo. Test on real transcripts first — the scaffolding system prompt is rule-dense. |
| Better Auth email+password | Better Auth + Google OAuth | Friends complain about passwords. One config block, requires a Google Cloud OAuth app. |
| Upload-complete-clip STT | Deepgram streaming WebSocket | Only after measuring that users complain about the pause. |

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| **Vercel / Netlify / CF Workers serverless** | No durable filesystem → SQLite dies → forced Postgres migration. This is the exact re-architecture PROJECT.md forbids. | Fly.io machine + volume, Railway, or a €4–6 VPS. |
| **Web Speech API (`SpeechRecognition`) as primary STT** | No Firefox/Edge support; flaky in iOS Safari; untunable accuracy on Vietnamese-accented English; no server-side audio artifact. | Deepgram Nova-3 with keyterm prompting. |
| **Browser `SpeechSynthesis` as primary TTS** | iOS exposes only low-quality Eloquence-class voices and `getVoices()` is unreliable; the learner hears a different teacher per device. | `gpt-4o-mini-tts` with cached clips; keep SpeechSynthesis as degraded fallback. |
| **Realtime speech-to-speech API as the core loop** | 8× cost on OpenAI; destroys the text transcript the error log/SRS depend on; forces a second implementation for text mode. | STT → LLM → TTS pipeline with swappable I/O adapters. |
| **Lucia v3** | Deprecated — now a learning resource, not a maintained library. | Better Auth 1.x. |
| **Redis / `rate-limiter-flexible` for rate limiting** | A whole daemon for a `SUM()` over 150 rows/month. | A `usage_ledger` table and a SQL query. |
| **Prisma** | Heavy engine binary, migration shadow-DB friction with SQLite, and no clean libSQL path. | Drizzle. |
| **Hand-rolled SM-2 intervals** | Worse-calibrated than FSRS and you'll spend a week on it. | `ts-fsrs`. |
| **Storing streak dates in UTC** | Off-by-one-day streak breaks are an abandonment trigger per PROJECT.md. | Store the user's IANA timezone; compute day boundaries in it. |

## Stack Patterns by Variant

**If you decide local-only is permanent (no deploy):**
- Swap Deepgram → `whisper.cpp` large-v3-turbo via a local HTTP wrapper; swap TTS → a local Kokoro/Piper model.
- Marginal cost drops to just the LLM. But you give up phone access, which PROJECT.md says the habit realistically needs.

**If friends grow beyond ~20 users:**
- Keep SQLite. Add Litestream. You are nowhere near a write-concurrency problem.
- Move the rate limiter from "monthly dollar cap" to a token bucket, still in SQL.

**If voice turns out to be the dominant mode (watch this after 2 weeks):**
- Add Deepgram streaming behind the existing `STT` port.
- Consider Gemini Live (~$0.09/session) for a separate "free talk" mode — but keep the scaffolded loop on the pipeline so the error log stays intact.

## Version Compatibility

| Package A | Compatible With | Notes |
|-----------|-----------------|-------|
| `better-sqlite3` 11.x | Node 22 LTS | Native module — needs a rebuild on Node major upgrades and will not run on Deno/Bun edge runtimes. Node 22's built-in `node:sqlite` is an alternative with no native-build step but a smaller API surface. |
| `drizzle-orm` 0.4x | `better-sqlite3` and `@libsql/client` | Same schema definitions work against both — this is what makes the Turso escape hatch cheap. |
| `better-auth` 1.x | `drizzle-orm` + SQLite | Use the Drizzle adapter so auth tables live in the same file as app data; one backup, one migration chain. |
| SvelteKit 2 | Svelte 5, Vite 6/7, Node 22 | `adapter-node` only. Do not use `adapter-vercel`/`adapter-cloudflare`. |
| Tailwind 4 | Vite plugin (`@tailwindcss/vite`) | v4 dropped the PostCSS-config flow; use the Vite plugin. |

**Pin all versions at scaffold time** — the minor versions above are MEDIUM confidence and this ecosystem moves monthly. The *architecture* claims are HIGH confidence and are what matter.

---

## For the Roadmapper: Load-Bearing vs Deferrable

### Load-bearing — must be settled in the foundation phase

| Decision | Why it can't be deferred |
|---|---|
| **Single always-on Node process + local SQLite file (not serverless)** | Choosing a serverless host later forces a full data-layer rewrite. Everything else assumes this. |
| **Schema with `user_id NOT NULL` on every user-scoped table, from migration 001** | Retrofitting per-user isolation means a data migration plus auditing every query. PROJECT.md requires real isolation. |
| **Auth (Better Auth) before the first conversation feature** | Without a `user_id` there is no isolation, no per-user rate limit, and no per-user streak — three requirements at once. |
| **Server-side AI proxy with a `usage_ledger` rate limiter, in front of every provider call** | The shared-key cost constraint is global. Retrofitting the limiter means touching every call site; building it first means it's one function. |
| **Provider ports: `LLM`, `STT`, `TTS` interfaces** | One afternoon now. Without them, swapping STT vendors (which you *will* do once you measure real Vietnamese-accented accuracy) is a refactor of the conversation loop. |

### Deferrable — safe to decide later behind the ports

- Which STT vendor (Deepgram vs OpenAI) — env var swap.
- Which TTS vendor and voice — env var swap.
- Exact model tier per call site — config value, tune after reading real transcripts.
- Streaming STT — strict upgrade from batch, no schema change.
- Turso/libSQL migration — Drizzle driver swap.
- UI framework details, styling, component library.
- Litestream backups, actual deployment — but the *shape* must be fixed in Phase 1.

### Stack choices that force phase ordering

1. **Auth + schema → before any conversation feature.** Can't retrofit isolation.
2. **AI proxy + rate limiter → before the first LLM call.** Can't retrofit cost control.
3. **Text mode → before voice mode.** Voice stacks three independent risks on top of the core loop (`getUserMedia` permissions, iOS gesture/format quirks, STT accuracy on accented speech). Text exercises the identical loop with zero audio risk. PROJECT.md says both modes reach the same core loop — build the loop in text, then add audio as I/O adapters. Shipping voice first means a loop bug and a microphone bug look identical.
4. **Post-session error analysis → before the SRS phase.** The SRS reads `error_log`; the analysis pass writes it. Scheduling SRS first leaves it with no input.
5. **HTTPS tunnel setup belongs in the voice phase, not the deploy phase.** You cannot test microphone capture on an iPhone without a secure context. If the roadmap puts tunneling in the final deploy phase, the voice phase will be untestable on the target device.
6. **Streak + timezone handling → early, and with tests.** It's a cross-cutting concern that touches session write, busy-day path, and progress UI. Retrofitting timezone-correct day boundaries means rewriting all three.

### Phases that will need deeper, dedicated research

- **Voice phase** — real measured WER on *this learner's* speech. The published accented-English numbers are a proxy, not an answer (see Gaps).
- **Scaffolding withdrawal phase** — no settled engineering pattern exists for "measure production quality, decrement scaffold level." This is product design, not stack selection.

### Phases that are standard and won't need research

- Auth, schema/migrations, streak tracking, rate limiting, deployment — all well-trodden.

---

## Gaps / Lower-Confidence Areas

- **No benchmark found for ASR WER on Vietnamese-accented English specifically.** All figures cited are accented/non-native English in aggregate. Vietnamese L1 has known patterns (final-consonant deletion, /θ/→/t/, tone-vs-stress interference) that plausibly hurt more than average. **Mitigation: the first voice spike should record 20 real utterances from the owner and compare Deepgram Nova-3 (with and without keyterms) against `gpt-4o-transcribe` on actual audio.** Budget this as a spike, not an assumption.
- **Framework minor versions** are MEDIUM confidence; pin at scaffold time.
- **Prompt-cache hit rate** on the conversation turn is estimated, not measured. If the system prompt changes per-turn (e.g. scaffold level injected at the top), caching breaks and the LLM line roughly triples. Keep volatile content *after* the last cache breakpoint.

## Sources

- [caniuse: Speech Recognition API](https://caniuse.com/speech-recognition) — browser support matrix, HIGH
- [Taming the Web Speech API](https://webreflection.medium.com/taming-the-web-speech-api-ef64f5a245e1) — mobile Safari workarounds, MEDIUM
- [Deepgram pricing (official)](https://deepgram.com/pricing) — Nova-3 and Aura-2 per-minute/per-char rates, HIGH
- [Deepgram: Introducing Nova-3](https://deepgram.com/learn/introducing-nova-3-speech-to-text-api) — keyterm prompting, WER claims (vendor-reported), MEDIUM
- [Nova-3 pricing & accent analysis](https://convertaudiototext.com/blog/deepgram-nova-3-explained) — independent note on accented-audio weakness, MEDIUM
- [Deepgram Nova vs Whisper large-v3-turbo](https://www.callmissed.com/blog/deepgram-nova-vs-whisper-large-v3-turbo) — accented-English WER comparison, MEDIUM
- [OpenAI API pricing (official)](https://developers.openai.com/api/docs/pricing) — transcription and TTS per-minute rates, HIGH
- [Claude model pricing](https://www.anthropic.com/pricing) — via bundled `claude-api` reference, cached 2026-09-25, HIGH
- [Gemini 3 Flash / GPT-5 mini pricing comparison](https://anotherwrapper.com/llm-pricing/gemini-3-flash-preview) — cross-vendor token rates, MEDIUM
- [OpenAI Realtime API pricing analysis](https://www.forasoft.com/article/openai-realtime-api-pricing) — per-minute realtime costs, MEDIUM
- [Gemini Live API pricing](https://tokenkarma.app/blog/gemini-live-api-pricing-voice-agents-2026/) — realtime audio rates, MEDIUM
- [Low-latency TTS benchmarks 2026](https://gradium.ai/content/best-low-latency-tts-apis-2026) — ElevenLabs TTFA measurements, MEDIUM
- [Best TTS APIs for developers](https://techsy.io/en/blog/best-tts-apis-developers) — TTS cost comparison, MEDIUM
- [WebKit: MediaRecorder API](https://webkit.org/blog/11353/mediarecorder-api/) — Safari support, HIGH
- [Recording audio in the browser](https://blog.addpipe.com/recording-audio-in-the-browser-using-pure-html5-and-minimal-javascript.md) — per-browser codec/container matrix, MEDIUM-HIGH
- [Apple Developer Forums: SpeechSynthesis voices](https://developer.apple.com/forums/thread/723503) + [WebKit bug 290497](https://bugs.webkit.org/show_bug.cgi?id=290497) — iOS voice quality limits, HIGH
- [Turso & libSQL in 2026](https://noqta.tn/en/blog/turso-libsql-distributed-sqlite-edge-database-2026) — embedded replica maturity, MEDIUM
- [Better Auth vs Lucia vs NextAuth 2026](https://www.pkgpulse.com/guides/better-auth-vs-lucia-vs-nextauth-2026) + [Self-hosted Node auth 2026](https://dev.to/noorix1/self-hosted-nodejs-authentication-in-2026-9ao) — Lucia deprecation, Better Auth consolidation, MEDIUM-HIGH
- [SvelteKit vs Next.js for solo developers](https://solodevstack.com/blog/nextjs-vs-sveltekit-solo-developers) — ecosystem-size tradeoff, MEDIUM

---
*Stack research for: personal AI language-learning web app (scaffolded conversation, voice + text, SRS, streaks)*
*Researched: 2026-10-06*
