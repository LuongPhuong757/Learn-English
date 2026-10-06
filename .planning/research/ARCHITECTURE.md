# Architecture Research

**Domain:** Single-tenant-per-user AI conversation habit app (web, voice + text, shared provider key)
**Researched:** 2026-10-06
**Confidence:** MEDIUM (see Confidence & Provenance at the end — the built-in web tools classify LOW by provider, but the load-bearing auth/latency claims were read directly from OpenAI's official docs)

---

## 0. Terminology Correction (read this first)

PROJECT.md says "local-first." Read in context — *"v1 runs on the owner's machine for speed of iteration, but must not be architected in a way that blocks deploying to a URL"* — this means **localhost-first**, not local-first in the CRDT/sync-engine sense.

**This distinction is the single largest over-engineering risk in the project.** If someone reads "local-first" as the Ink & Switch meaning, they will reach for ElectricSQL / Automerge / Yjs / RxDB and build a bidirectional sync engine for 5 users who each only ever read their own rows on one device at a time. Do not.

What is actually required: a conventional server app whose *deploy target is an environment variable*, plus a narrow offline path for one specific activity (§8).

---

## 1. Standard Architecture

### System Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│  BROWSER (phone or Mac) — never sees the provider API key            │
├──────────────────────────────────────────────────────────────────────┤
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌──────────────────┐   │
│  │ Today      │ │ Session    │ │ Review     │ │ speech adapter   │   │
│  │ screen     │ │ (voice/txt)│ │ (ZCA)      │ │ listen()/speak() │   │
│  └─────┬──────┘ └─────┬──────┘ └─────┬──────┘ └────────┬─────────┘   │
│        │              │              │                 │             │
│  ┌─────┴──────────────┴──────────────┴─────┐  ┌────────┴─────────┐   │
│  │ IndexedDB: outbox + today's ZCA cache   │  │ Service Worker   │   │
│  └─────────────────────────────────────────┘  └──────────────────┘   │
└────────────────────────────┬─────────────────────────────────────────┘
                             │ HTTPS, session cookie, idempotency key
┌────────────────────────────┴─────────────────────────────────────────┐
│  SERVER (one Next.js process — route handlers)                       │
├──────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │ auth → assertBudget() ← THE CHOKE POINT, every AI path        │    │
│  └───────────────────────────┬──────────────────────────────────┘    │
│  ┌──────────────┐ ┌──────────┴───────┐ ┌──────────┐ ┌────────────┐   │
│  │ scaffold/    │ │ ai/turn.ts       │ │ srs/     │ │ activity/  │   │
│  │ level.ts     │ │ (1 stream call)  │ │ fsrs     │ │ choose.ts  │   │
│  │ (DB, no LLM) │ │ ai/errors.ts     │ │ due-on-  │ │ zero-cost  │   │
│  │              │ │ (parallel call)  │ │ read     │ │ .ts        │   │
│  └──────────────┘ └──────────────────┘ └──────────┘ └────────────┘   │
│  ┌──────────────┐ ┌──────────────────┐ ┌────────────────────────┐    │
│  │ streak/      │ │ budget/record    │ │ BlobStore (iface)      │    │
│  │ compute-on-  │ │ cost_micros      │ │ LocalDisk | S3         │    │
│  │ read         │ │                  │ │                        │    │
│  └──────────────┘ └──────────────────┘ └────────────────────────┘    │
├──────────────────────────────────────────────────────────────────────┤
│  POSTGRES (docker-compose locally · managed in prod · DATABASE_URL)  │
│  users · scaffold_state · sessions · turns · captured_errors ·       │
│  review_items · review_logs · streaks(cache) · usage_counters        │
└────────────────────────────┬─────────────────────────────────────────┘
                             │ server-side only, real API key
                    ┌────────┴────────┐
                    │ LLM provider    │   (+ optional: STT, TTS)
                    └─────────────────┘
```

**Zero scheduled jobs. Zero queues. Zero Redis. Zero object storage in v1.** All deliberate — see §9.

### Component Responsibilities

| Component | Responsibility | Implementation |
|-----------|----------------|----------------|
| `server/env.ts` | Validate every env var with zod at module load; throw at boot | ~30 lines. The only place `process.env` is read. |
| `server/budget/assert.ts` | Single function every AI path calls before spending | `SELECT…FOR UPDATE` on `usage_counters`, returns a **tier**, never throws a 429 |
| `server/scaffold/level.ts` | Compute "how much support right now" from DB rows | Pure function + one query. **No model call.** |
| `server/ai/turn.ts` | One streaming structured call → reply + frames | Vercel AI SDK `streamText` with structured output |
| `server/ai/errors.ts` | Parallel, non-blocking error analysis of learner's utterance | Cheap small model, constrained to the closed `structure_key` enum |
| `server/srs/` | FSRS schedule read/write | `ts-fsrs`; due computed at read time |
| `server/streak/compute.ts` | Streak as a pure function of `sessions.local_date` | No job. Cached in `streaks`, recomputable. |
| `server/activity/zero-cost.ts` | **Keystone.** A streak-qualifying activity with zero model calls | Serves busy-day + budget-exhausted + offline |
| `lib/speech/` | `listen()` / `speak()` adapters | WebSpeech impl v1; server-STT impl swappable |
| `lib/datetime.ts` | **The only file where timezone math happens** | IANA tz in, `local_date` out |
| `lib/offline/outbox.ts` | IndexedDB queue of unsynced actions with client-generated ids | ~80 lines, no library |

---

## 2. Overall Shape — and what the shared key forces

### Decision: one Next.js (App Router) TypeScript app. One repo, one process, one deployable.

**Why not split frontend/backend:** two deployables, a CORS surface, a second auth hop, and duplicated types — for 5 users and one developer. The split buys independent scaling you will never need.

**Why not a pure SPA + separate API:** the SPA has no server-rendered boundary, so "this module must never reach the browser" becomes a convention instead of a build-enforced rule. With Next.js, `import 'server-only'` at the top of every file under `src/server/` makes the build **fail** if a client component ever transitively imports it. That is the mechanical enforcement of the key constraint. Use it.

**Why not server-rendered-only (no client interactivity):** voice capture, streaming token rendering, and the offline outbox all require client JS. The App Router gives you both without choosing.

### What the shared owner-funded key forces

Three hard consequences, in order of how much they shape the design:

1. **Every provider call originates from the server process.** No `NEXT_PUBLIC_*` key, ever. No client-side `fetch('https://api.openai.com')` with a real key. This is non-negotiable and it is what rules out the whole class of "serverless frontend calls the AI directly" architectures.

2. **There must be exactly one function all spending passes through.** Because the key is shared, per-user accounting is only possible if no call site can bypass accounting. If you add `assertBudget()` after you have six call sites, you will miss one. **Build the choke point before the first model call ships.**

3. **If you ever want true realtime voice, you cannot keep the key server-side *and* keep the audio server-side.** OpenAI's Realtime API solves this with ephemeral client secrets: your server POSTs to `https://api.openai.com/v1/realtime/client_secrets` with the real key and returns a short-lived token; the browser then POSTs its SDP offer to `https://api.openai.com/v1/realtime/calls` with that ephemeral token and holds a WebRTC peer connection straight to OpenAI. *"You should only use standard OpenAI API keys on the server, not in the browser."* ([OpenAI Realtime WebRTC guide](https://developers.openai.com/api/docs/guides/realtime-webrtc.md)) The real key never leaves your server — **but your server also never sees the audio or the token count in real time.** That trade is the subject of §6.

### Request path — one conversation turn, end to end (text or Web-Speech voice)

```
 1. [browser]  learner speaks or types          → transcript string
                 voice: lib/speech/webspeech.ts SpeechRecognition → final transcript
                 text:  <textarea> submit
 2. [browser]  POST /api/sessions/{id}/turns
                 body: { text, clientTurnId: uuid, localDate: "2026-10-06", frameUsed: bool }
                 headers: cookie=session; Idempotency-Key: <clientTurnId>
 3. [server]   auth → resolve userId from session cookie               ~1 ms
 4. [server]   assertBudget(userId, estimate) → tier: FULL|REDUCED|ZERO ~3 ms
                 ZERO → respond 200 with {redirect: "zero-cost"}; NEVER 429
 5. [server]   INSERT turns (role='learner') ON CONFLICT (session_id, client_turn_id)
                 DO NOTHING   ← idempotency; a retried offline sync is free     ~3 ms
 6. [server]   scaffold/level.ts: SELECT scaffold_state → effective level       ~3 ms
                 ── fires in parallel, awaited by nobody ──────────────────┐
 7a.[server]   ai/errors.ts (cheap model, constrained to structure_key enum)│ 600-1500 ms
                 → UPSERT captured_errors, UPSERT review_items              │ (invisible)
                 ─────────────────────────────────────────────────────────┘
 7b.[server]   ai/turn.ts: ONE streamText call. Prompt carries:
                 scenario goal, last N turns, effective scaffold level,
                 top 1-3 recurring structure_keys for this user.
                 Structured output with `reply` as the FIRST field.
               → first token                                            400-800 ms
 8. [server]   stream SSE/text chunks back as they arrive                 (passthrough)
 9. [browser]  render tokens; as soon as `reply` closes, speak() starts        +50 ms
10. [browser]  frames field arrives after reply → render frame chips
11. [server]   on stream end: INSERT turns (role='ai'); budget/record actual
                 usage into usage_counters; mark session.produced_utterance=true
```

**Perceived time to first word out of the speaker: ~0.5–0.9 s.** The error analysis (7a) is 600–1500 ms but runs concurrently and is never on the critical path.

---

## 3. The Conversation Engine

### The three candidates, assessed

| | (a) Single stateful prompt | (b) Structured turn pipeline (3 sequential calls) | (c) Framework agent loop (LangGraph etc.) |
|---|---|---|---|
| Model calls / turn | 1 | 3 | 3–8 |
| Time to first word | 400–800 ms | ~2.5–4 s | 2–6 s |
| Cost / turn | 1× | ~2.5× | 3–6× |
| Controllability of scaffold level | **Poor** — model drifts, forgets frames | Good | Good |
| Error detection quality | Mediocre (competes with reply for attention) | Good | Good |
| Debuggability when it misbehaves | Reread a prompt | Inspect 3 outputs | Trace a graph |
| Dependency weight | zero | zero | heavy |
| Verdict for **this** project | latency ✅ control ❌ | control ✅ latency ❌ | **over-engineering** |

Voice is where this is decided. Natural turn-taking needs sub-800 ms and feels broken past ~1.5 s ([AI Engineer / voice agent guidance](https://ai.engineer/talks/building-effective-voice-agents); OpenAI Realtime targets sub-500 ms). A beginner who waits 3 seconds after every sentence quits. Pipeline (b) is disqualified on latency alone. Agent loop (c) adds a graph runtime, a state serializer, and a dependency with its own release cadence to solve a problem you do not have at 5 users — call it out in review if it appears.

### Recommended: (a′) — hybrid, and the key move is that scaffold level is *not* an LLM decision

```
              ┌────────────────────────────────────────────┐
 learner      │  scaffold/level.ts  — PURE CODE, 3 ms      │
 utterance ──▶│  reads scaffold_state, returns level 0.0-3.0│
      │       └────────────────────┬───────────────────────┘
      │                            │ injected as a prompt constraint
      │       ┌────────────────────▼───────────────────────┐
      ├──────▶│  ai/turn.ts — ONE streaming structured call │──▶ reply (streamed)
      │       │  schema: { reply, frames[], recastUsed }    │──▶ frames (after)
      │       └────────────────────────────────────────────┘
      │
      └──────▶┌────────────────────────────────────────────┐
   (parallel) │  ai/errors.ts — cheap model, non-blocking   │──▶ captured_errors
              │  output constrained to closed structure_key │──▶ review_items
              └────────────────────────────────────────────┘
```

Three design moves, each of which buys back controllability without buying latency:

**(i) Scaffold level is computed in code, not asked of the model.** This is the single most important choice in the engine. The thing you most need to be stable, testable, and auditable — *how much help this person gets right now* — is a deterministic function of DB rows. Asking a model to decide it every turn makes it non-reproducible and un-unit-testable. Costs 3 ms.

**(ii) One call produces reply AND frames, with `reply` first in the schema.** Vercel AI SDK `streamText` with structured output (`Output.object()`) streams partial objects, so ordering the schema `{ reply, frames, recastUsed }` means the speakable text arrives before the UI chrome. ([AI SDK docs](https://vercel.com/docs/ai-sdk)) Two calls would double TTFT for no benefit.

**(iii) Error analysis fires in parallel the instant the utterance arrives — never after the reply.** The learner's text is fully available at step 5; nothing about error detection needs the AI's reply. Running it sequentially after the reply (the obvious implementation) adds a full round trip to the critical path for zero information gain.

**Inline correction (the recast) stays in the main call,** fed by the DB: the prompt receives this user's top 1–3 active `structure_key`s so the AI can naturally re-say the learner's sentence correctly inside its reply. That costs nothing and keeps conversational flow — which is the stated requirement ("corrects errors without breaking conversational flow").

**Busy-day / zero-budget degradation:** skip (iii) entirely and batch error analysis once at session end over the full transcript. One call instead of N.

```ts
// server/ai/turn.ts — shape, not full implementation
const level = await getEffectiveScaffoldLevel(userId, scenarioId); // 0.0–3.0, pure DB
const recurring = await getTopActiveStructures(userId, 3);         // pure DB

void analyzeErrors({ userId, turnId, text: utterance });            // fire-and-forget

return streamText({
  model: provider(tier === 'REDUCED' ? MODEL_CHEAP : MODEL_MAIN),
  system: buildSystemPrompt({ scenario, level, recurring, learnerL1: 'vi' }),
  messages: recentTurns,                     // from DB, not from memory
  experimental_output: Output.object({ schema: TurnSchema }), // reply FIRST
});
```

---

## 4. Scaffolding State Model

### The four candidate granularities

| Granularity | Cold start | Drives UI? | Problem |
|---|---|---|---|
| Single global level | easy | yes | Too coarse — someone who can order coffee is still lost in a job interview; one number drags them backwards everywhere |
| Per-skill (speaking/listening/…) | easy | awkward | The app has one production loop; "listening level" has nowhere to go |
| Per-grammar-structure | **brutal** — 200 unknowns on day 1 | no | This is a *mastery* model, not a *support* model. It already exists as SRS. |
| Per-scenario | **natural** — new scenario = max support | yes | Alone, it never learns the learner is globally improving |

### Recommended: two-tier — slow global baseline + fast per-scenario delta

```
effective_level = clamp(global.level + scenario.delta, 0.0, 3.0)
```

Per-grammar-structure data is **not** duplicated here. It lives in `review_items` and feeds *content* (which errors get recast, which cards are due) — not *support level*. Keeping those two concerns separate is what stops this table from turning into a second SRS.

```sql
CREATE TABLE scaffold_state (
  user_id    uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  scope      text NOT NULL,          -- 'global' | 'scenario'   (open for 'skill' later)
  scope_key  text NOT NULL DEFAULT '', -- '' for global; scenario_id for scenario
  level      real NOT NULL DEFAULT 0.0,  -- 0.0 .. 3.0, FLOAT not enum (see below)
  confidence real NOT NULL DEFAULT 0.0,  -- 0..1, grows with observations; damps swings
  observations int NOT NULL DEFAULT 0,
  updated_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (user_id, scope, scope_key)
);
```

The generic `(scope, scope_key)` shape means adding a third tier later is **inserting rows, not migrating a table**. That is cheap insurance; take it.

### The levels, and how they move

| Level | Name | What the learner sees |
|---|---|---|
| 0.0–0.7 | FULL | AI offers 2–3 complete sentences; learner taps or repeats one |
| 0.8–1.5 | FRAME | `"I'd like a ___, please."` + 3 word chips |
| 1.6–2.3 | HINT | Open turn + 3 vocabulary chips, no structure given |
| 2.4–3.0 | FREE | Open turn; frames hidden behind a "help" button |

**Promotion/demotion signal must be *production*, not *correctness*.** The Core Value is "produces at least one utterance," not "produces a correct utterance." Grading on correctness at L0 punishes exactly the learner this app exists for.

```
per turn:  produced = turn.text.length > 0
           unassisted = produced && !turn.frame_used
after each session:
  if unassisted_rate >= 0.7 over last 3 sessions in scope → level += 0.25 * confidence
  if produced_rate   <  0.5  in this session              → level -= 0.5   (IMMEDIATE, no damping)
  global.level = EWMA(α=0.15) over scenario effective levels   -- deliberately slow
```

Demotion is faster than promotion and ignores confidence. A learner stuck at too high a level produces nothing, which is a Core Value failure; a learner at too low a level is merely bored. Asymmetry is correct here.

### The trade-off, stated plainly

Two tiers cost you a join and a second update path, versus one global integer. You buy: new scenarios start supported without resetting the learner globally, and a bad day in one scenario does not tank the baseline. You do **not** buy fine-grained grammar targeting — that is SRS's job and duplicating it here would be the mistake.

### Hard to reverse — and the cheap fix

Storing `level` as an **enum/int of 4 buckets** is hard to reverse: every historical `turns.scaffold_level` becomes uninterpretable the day you want 6 buckets. **Store a float 0.0–3.0 and bucket in the UI.** Re-bucketing then becomes a CSS-layer change over intact history. This costs nothing today.

---

## 5. Data Model

Postgres. DDL below is the real thing, not a sketch.

```sql
-- ─── identity ────────────────────────────────────────────────────────────
CREATE TABLE users (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),  -- OPAQUE. never email.
  auth_subject  text UNIQUE,              -- provider's sub; NULL in dev mode
  email         text UNIQUE,
  display_name  text NOT NULL,
  timezone      text NOT NULL DEFAULT 'Asia/Ho_Chi_Minh',  -- IANA NAME, never an offset
  daily_budget_micros bigint NOT NULL DEFAULT 300000,      -- $0.30/day
  retain_audio  boolean NOT NULL DEFAULT false,
  created_at    timestamptz NOT NULL DEFAULT now()
);

-- ─── content (seeded from repo, not user-authored) ───────────────────────
CREATE TABLE scenarios (
  id text PRIMARY KEY,                    -- 'cafe-order', 'intro-yourself'
  title text NOT NULL, title_vi text NOT NULL,
  goal_prompt text NOT NULL,              -- injected into system prompt
  seed_frames jsonb NOT NULL DEFAULT '[]',
  est_minutes int NOT NULL DEFAULT 5,
  busy_day_eligible boolean NOT NULL DEFAULT false
);

-- ─── the loop ────────────────────────────────────────────────────────────
CREATE TABLE sessions (
  id uuid PRIMARY KEY,                    -- CLIENT-GENERATED (offline-safe)
  user_id uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  scenario_id text REFERENCES scenarios(id),
  kind text NOT NULL CHECK (kind IN ('conversation','busy_day','review')),
  modality text NOT NULL CHECK (modality IN ('voice','text')),
  local_date date NOT NULL,               -- DENORMALIZED from user tz AT WRITE TIME
  time_budget_minutes int,
  scaffold_level_start real NOT NULL,
  produced_utterance boolean NOT NULL DEFAULT false,  -- ← the Core Value flag
  turn_count int NOT NULL DEFAULT 0,
  started_at timestamptz NOT NULL DEFAULT now(),
  ended_at timestamptz
);
CREATE INDEX ON sessions (user_id, local_date DESC);

CREATE TABLE turns (
  id uuid PRIMARY KEY,
  session_id uuid NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
  client_turn_id uuid NOT NULL,           -- idempotency key from the browser
  idx int NOT NULL,
  role text NOT NULL CHECK (role IN ('learner','ai')),
  text text NOT NULL,                     -- ALWAYS text, both modalities
  audio_url text,                         -- NULL by default (see §7); column exists day 1
  scaffold_level real,
  frame_offered jsonb,
  frame_used boolean NOT NULL DEFAULT false,
  latency_ms int,
  created_at timestamptz NOT NULL DEFAULT now(),
  UNIQUE (session_id, client_turn_id),    -- ← makes retried syncs free
  UNIQUE (session_id, idx)
);

-- ─── errors & review ─────────────────────────────────────────────────────
CREATE TABLE captured_errors (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  turn_id uuid REFERENCES turns(id) ON DELETE SET NULL,
  structure_key text NOT NULL,            -- ← CLOSED ENUM. see hard-to-reverse #2
  learner_text text NOT NULL,
  corrected_text text NOT NULL,
  explanation_vi text,
  created_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ON captured_errors (user_id, structure_key);

CREATE TABLE review_items (                -- ONE ROW PER STRUCTURE, not per error
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  structure_key text NOT NULL,
  -- FSRS state
  due_at timestamptz NOT NULL,
  stability real NOT NULL DEFAULT 0,      -- memory half-life in days
  difficulty real NOT NULL DEFAULT 0,     -- 1..10
  reps int NOT NULL DEFAULT 0,
  lapses int NOT NULL DEFAULT 0,
  state text NOT NULL DEFAULT 'New',      -- New|Learning|Review|Relearning
  last_review_at timestamptz,
  occurrence_count int NOT NULL DEFAULT 1,
  status text NOT NULL DEFAULT 'active' CHECK (status IN ('active','retired')),
  UNIQUE (user_id, structure_key)
);
CREATE INDEX ON review_items (user_id, due_at) WHERE status = 'active';

CREATE TABLE review_logs (                 -- append-only; UNBACKFILLABLE if skipped
  id bigserial PRIMARY KEY,
  review_item_id uuid NOT NULL REFERENCES review_items(id) ON DELETE CASCADE,
  rating int NOT NULL,                     -- 1 Again 2 Hard 3 Good 4 Easy
  reviewed_at timestamptz NOT NULL DEFAULT now(),
  elapsed_days real, scheduled_days real,
  stability_before real, difficulty_before real
);

-- ─── derived caches ──────────────────────────────────────────────────────
CREATE TABLE streaks (                     -- CACHE ONLY. sessions is the truth.
  user_id uuid PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  current_len int NOT NULL DEFAULT 0,
  longest_len int NOT NULL DEFAULT 0,
  last_active_local_date date,
  updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE usage_counters (
  user_id uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  window_key text NOT NULL,                -- 'day:2026-10-06' | 'month:2026-10'
  requests int NOT NULL DEFAULT 0,
  tokens_in bigint NOT NULL DEFAULT 0,
  tokens_out bigint NOT NULL DEFAULT 0,
  audio_seconds int NOT NULL DEFAULT 0,
  cost_micros bigint NOT NULL DEFAULT 0,   -- MONEY is the unit, not requests
  PRIMARY KEY (user_id, window_key)
);
```

### Relationships

```
users ─1:N─ sessions ─1:N─ turns ─0:N─ captured_errors
  │                                        │
  │                                 (structure_key, text FK-by-convention)
  │                                        ▼
  ├─1:N─ review_items ─1:N─ review_logs
  ├─1:N─ scaffold_state   (scope='global' 1 row, scope='scenario' N rows)
  ├─1:1─ streaks          (cache, derivable from sessions.local_date)
  └─1:N─ usage_counters   (one row per (user, window_key))

scenarios ─1:N─ sessions
```

### Hard-to-reverse decisions, ranked by pain

| # | Decision | Why it cannot be undone cheaply |
|---|---|---|
| 1 | **Postgres vs SQLite** | Dialect leaks into queries, migrations, upsert semantics, date handling. Reversing means a data move *and* an audit of every query. |
| 2 | **`structure_key` as a closed enum vs free text** | If the model free-generates keys you get `past_tense_error`, `wrong_past_tense`, `verb_tense_past` as three cards. SRS never consolidates. Fixing retroactively means LLM-relabelling all history and merging FSRS state — the merged stability/difficulty is fiction. **Ship a `content/structure-keys.ts` with ~40 keys + `other` before the first error is recorded.** |
| 3 | **`users.timezone` as IANA name, not UTC offset** | An offset is wrong twice a year under DST, and old rows carry no recoverable truth about which offset was in effect. |
| 4 | **`sessions.local_date` denormalized at write time** | Recomputing later requires knowing the user's tz *at that moment*, which you didn't store. Streak history becomes unverifiable. |
| 5 | **`review_logs` appended from day one** | Unbackfillable. Without it you can never run FSRS parameter optimization against this learner. Costs one INSERT. |
| 6 | **`users.id` opaque uuid, not email as PK** | Switching auth providers (dev login → Google/magic link) rewrites every FK if identity is keyed on email. |
| 7 | **Client-generated `session.id` / `client_turn_id`** | Retrofitting idempotency after duplicates exist means a reconciliation script and judgement calls about which duplicate was real. |
| 8 | **Audio retained vs discarded** | Discard → no pronunciation feature ever without new data. Retain → you inherit object storage, signed URLs, retention policy, and a privacy obligation toward friends. Decide explicitly, not by accident. §7 recommends discard-with-a-door-open. |
| 9 | **`scaffold_state.level` float vs enum** | §4. Float is free insurance. |

---

## 6. Local-first → Deployed Migration Path

**Target state on day one: the only difference between `pnpm dev` on the Mac and production is the contents of the environment.**

| Concern | Day-one rule | The specific trap if you skip it |
|---|---|---|
| **Database** | Postgres 16 via `docker-compose.yml` locally; managed Postgres in prod. Single `DATABASE_URL`. Drizzle + checked-in SQL migrations. | SQLite file. Then: Vercel's filesystem is read-only/ephemeral, and on Fly *"anything written to a Machine's root disk is ephemeral: when the Machine is redeployed, the root file system is rebuilt… deleting any data"* ([Fly.io](https://fly.io/docs/js/prisma/sqlite/)). You either pin yourself to one container with a mounted volume forever, or you do a dialect migration under pressure. |
| **Database (if you insist on SQLite)** | Then commit *now* to a container-with-volume host (Fly/Railway), forbid serverless, and keep to the portable column subset: `text`/`integer`/`timestamp` only, no `jsonb`, no arrays, query builder only, additive migrations only ([Drizzle portability guidance](https://instagit.com/BuilderIO/agent-native/using-drizzle-orm-agent-native-portable-schema.md)). | You use `jsonb` for `frame_offered` (you will — it's the natural type) and the "portable schema" promise quietly dies. |
| **Auth** | Real session cookie + real `users` row from day one. Dev convenience lives behind `AUTH_MODE=dev` (a "pick a user" screen), **not** behind a hardcoded id. Every query carries `where user_id = ?` from the first query written. | `const USER_ID = 1`. Six weeks later you retrofit tenant filtering into 60 call sites and miss three — and "private, isolated data per user" is a named requirement. |
| **Secrets** | `src/server/env.ts` reads `process.env` once, validates with zod, throws at boot. `.env.local` gitignored. No var is ever `NEXT_PUBLIC_*` if it is a secret. | A key in a committed config file; or a shared `config.ts` imported by a client component and silently bundled into the browser JS. |
| **Server/client wall** | `import 'server-only'` as line 1 of every file under `src/server/`. | Next.js will happily bundle a module into the client bundle if a client component imports it; the build does not warn **unless** `server-only` is present. This is the enforcement mechanism for the entire §2 constraint. |
| **Files / audio** | A `BlobStore` interface with `put(key, bytes) → url` and two impls (`LocalDiskStore`, `S3Store`). Callers only ever see URLs. Wire `LocalDiskStore` now even though §7 says store nothing. | `fs.writeFileSync('./uploads/…')` scattered across handlers. Container restart eats it; and the refactor touches every call site. |
| **URLs** | `APP_URL` env var. Browser fetches are relative (`/api/…`). | `http://localhost:3000` baked into a redirect, an OAuth callback, or the realtime-token fetch. |
| **Process timezone** | Set `TZ=UTC` in dev *and* prod. All local-date math goes through `lib/datetime.ts` with the user's IANA tz. | Your Mac is `Asia/Ho_Chi_Minh`; a Vercel/Fly box is UTC. `new Date().toISOString().slice(0,10)` gives a *different day* on the two machines. Streaks silently break on deploy day. This is the most likely real bug in this project. |
| **Background jobs** | None (§9). | `node-cron` in-process: runs zero times on serverless, runs N times on N instances. Both failure modes are silent. |
| **Secure context for mic** | `getUserMedia` / `MediaRecorder` / `SpeechRecognition` require a secure context. `localhost` counts; **`http://192.168.1.x:3000` does not.** Use `cloudflared tunnel` / `ngrok` for phone testing from day one. | You try voice mode on your phone over the LAN, the mic never prompts, you spend an afternoon debugging the wrong layer. |
| **Rate-limit counter location** | Postgres, not process memory. | In-memory `Map` resets on redeploy and is per-instance. At 5 users Postgres is free; there is no reason to take the risk. |

**Deployment target recommendation:** a container host with a managed Postgres (Fly.io, Railway, Render) rather than Vercel + Neon — not because Vercel is worse, but because a long-lived process keeps the door open for streaming/WebRTC token flows and removes the serverless-timeout class of problems from a streaming app. Either works; decide before writing the first migration, because it determines #1 above.

---

## 7. Rate Limiting and Cost Control

### Where the counter lives

Postgres `usage_counters`, keyed `(user_id, 'day:' || userLocalDate)`. One `INSERT … ON CONFLICT DO UPDATE … RETURNING` per call. At 5 users this is sub-millisecond; **Redis here is over-engineering.**

**Critical alignment:** the budget window must reset at the **user's local midnight — the same boundary as the streak.** If the budget window is UTC and the streak window is local, a Vietnamese learner opening the app at 8 a.m. finds a "new day" streak but a budget that doesn't reset until 7 a.m. the *next* day. That is a confusing bug that will be blamed on the model.

### The unit is money, not requests

`cost_micros`. Requests are a useless unit here because the spread is enormous:

| Activity | Rough cost |
|---|---|
| One text turn (small model + parallel error call) | **~$0.002–0.01** |
| 10-min session via Web Speech STT + text LLM + browser TTS | **~$0.05–0.15** |
| 10-min session via server STT (`gpt-realtime-whisper` $0.017/min) + text LLM + server TTS | ~$0.25–0.45 |
| 10-min session on flagship Realtime (`gpt-realtime-2.1`, $32/$64 per 1M audio in/out, ~$0.06–0.11/min) | **~$0.60–1.10** |
| 10-min session on `gpt-realtime-2.1-mini` ($10/$20 per 1M, ~$0.02–0.05/min) | ~$0.20–0.50 |

([pricing figures](https://www.forasoft.com/article/openai-realtime-api-pricing), [mini pricing](https://www.eesel.ai/blog/gpt-realtime-mini-pricing))

**This table changes a decision.** At a sane $0.30/user/day budget (5 users ≈ $45/month ceiling), *one* flagship Realtime session exceeds a full day's budget. The cheap path — browser STT + text LLM + browser TTS — fits ~3 sessions a day inside the same budget. That is the economic argument for §8's recommendation, independent of the latency argument.

### Enforcement: one choke point, pre-authorize then reconcile

```ts
// server/budget/assert.ts — the ONLY gate. Called before every provider call.
export async function assertBudget(userId: string, estMicros: number): Promise<Tier> {
  const { spent, budget } = await readOrCreateCounter(userId, dayKey(userId));
  const pct = (spent + estMicros) / budget;
  if (pct < 0.70) return 'FULL';      // best model, chosen modality
  if (pct < 0.90) return 'REDUCED';   // cheap model, warn in UI
  if (pct < 1.00) return 'WINDING_DOWN'; // finish current turn, then steer to review
  return 'ZERO';                      // no model calls at all
}
```

For the **Realtime WebRTC path specifically** (if ever built), exact metering is impossible because the server is not in the audio path. Handle it by **pre-authorizing instead of post-metering**: mint the ephemeral client secret with a hard `max_output_tokens` and a session minute cap, debit the *worst-case* cost of that session up front, refund the difference when the client reports actual usage over the data channel at session end, and refuse to mint another token once the day's budget is spent. Set `OpenAI-Safety-Identifier` to the hashed user id on the server-side mint — *"the Realtime API binds the identifier to the resulting ephemeral token"* ([OpenAI](https://developers.openai.com/api/docs/guides/realtime-webrtc.md)) — so abuse stays attributable. Client-reported usage is advisory; the pre-authorization is the real control. For 5 trusted friends this is entirely adequate; building exact mid-session metering would be over-engineering.

### What happens at the limit — the Core Value interaction

**Never return 429. Never block the day.** The Core Value says: *"Every single day the learner opens the app and successfully produces at least one English utterance. If everything else fails, this must not."* A rate limiter that hard-fails is a direct violation of the product's one non-negotiable.

So the limiter **does not deny — it downgrades**:

| Tier | What the learner experiences | Model cost | Streak? |
|---|---|---|---|
| FULL | Full conversation in the chosen modality | full | ✅ |
| REDUCED | Same loop, cheaper model, banner: *"~5 minutes of conversation left today"* | ~⅓ | ✅ |
| WINDING_DOWN | AI brings the scene to a natural close, then offers review | tapering | ✅ |
| **ZERO** | **Zero-Cost Activity**: due `review_items` rendered with stored frames and stored expected answers. The learner speaks or types a response; validation is string/fuzzy match in code. `produced_utterance = true`. | **$0.00** | ✅ |

### The structural finding

> **There must exist at least one streak-qualifying activity that makes zero model calls.**

That single component satisfies **three otherwise separate requirements**:

1. **Busy-day path** — "the app decides the shortest high-value activity" (requirement) → due reviews, 90 seconds, zero cost.
2. **Budget exhausted** — the ZERO tier above.
3. **Offline** — §8; no network needed because the content is already in IndexedDB.

Collapsing three requirements into one component is the most valuable architectural result in this document, and it has a direct roadmap consequence: **the Zero-Cost Activity is not a late nice-to-have, it is a mid-early foundational component.** Build it right after SRS, before voice.

---

## 8. Audio Pipeline

### Three architectures, measured

| | Latency to first AI audio | Cost | Storage obligation | Engine count |
|---|---|---|---|---|
| **A. Web Speech API in + SpeechSynthesis out** | **~0.5–0.9 s** (recognition ~0–300 ms after speech end, LLM TTFT 400–800 ms, TTS starts immediately) | **$0** for STT/TTS | none — no audio file ever exists | **one** (voice is a shell over text) |
| **B. MediaRecorder → upload → server STT → LLM → server TTS** | 1.5–3 s with *streaming* STT (Deepgram Nova-3 TTFT ~150 ms US / 250–350 ms global); **3–7 s with batch Whisper** | ~$0.017–0.04/min | optional | one |
| **C. Realtime WebRTC speech-to-speech** | **~0.5 s**, true barge-in/interruption | $0.02–0.11/min | none by default | **two** (separate metering + state model) |

([STT latency benchmarks](https://gradium.ai/content/stt-api-benchmark-2026-latency-accuracy); [Deepgram vs Whisper](https://www.deepgram.com/learn/whisper-vs-deepgram))

**The disqualifying fact for B-with-Whisper:** the OpenAI Whisper API is batch-oriented at 1–5 s and is not designed for real-time streaming. If you go server-side STT, it must be a *streaming* provider, or voice mode is dead on arrival.

### Recommendation: A for v1, B as the escape hatch, C only if barge-in proves necessary

**v1 = Web Speech API in, SpeechSynthesis out.** Zero cost, zero infrastructure, zero storage, ~1 s round trip, and — decisively — **voice becomes a thin shell over the text engine**, so there is one conversation engine, one scaffold path, one error path, one SRS path.

**The honest risk, and it is the biggest open question in this document:** Safari's `webkitSpeechRecognition` *"still uses an older model"*, needs network, and prompts about sending audio to Apple; Apple's newer SpeechAnalyzer has no Web Speech surface ([addpipe](https://blog.addpipe.com/apple-speechanalyzer-api/)). Accuracy on **Vietnamese-accented beginner English** is unmeasured. If recognition mangles a beginner's utterance, the app corrects an error they didn't make — which destroys trust faster than any latency problem.

> **Spike this before the voice phase is planned.** 20 minutes: record 15 utterances from the actual learner, run them through Chrome/Android and Safari/iOS Web Speech, count word error rate. A bad result moves voice to option B (+$0.017/min, +400–800 ms) and changes the phase's scope and cost model.

**The architectural requirement that makes this reversible:** both paths sit behind one adapter, and `turns.text` is the only thing the rest of the system sees.

```ts
// lib/speech/index.ts — swapping providers is a config change, not a rewrite
export interface SpeechAdapter {
  listen(opts: { lang: string; onPartial?: (t: string) => void }): Promise<string>;
  speak(text: string, opts?: { rate?: number }): Promise<void>;
  cancel(): void;
}
export const speech: SpeechAdapter =
  process.env.NEXT_PUBLIC_SPEECH_MODE === 'server' ? serverSpeech : webSpeech;
```

### Storage: discard by default, door left open

Keep **text only**. Rationale:

- The only v1 feature that needs the audio bytes is pronunciation scoring — explicitly out of scope.
- Retaining voice recordings of friends creates a real privacy obligation with no offsetting benefit.
- Audio storage is what forces object storage + signed URLs + lifecycle policies into existence — a whole subsystem, for nothing.

But keep `turns.audio_url` (nullable), `users.retain_audio` (default false), and the `BlobStore` interface from day one. Enabling retention later is then a flag plus a 30-day TTL, not a migration plus a refactor.

**Latency note for option B if you take it:** 5 s of webm/opus mono at 32 kbps ≈ 20 kB — upload is 150–400 ms on 4G, which is not the problem. The problem is always the STT provider's model, not the bytes on the wire. Optimize the provider, not the encoding.

---

## 9. Session Resumption and Offline Behavior

### Design rule that makes resumption free

> **No server-side in-memory conversation state. Ever.**

Conversation state is the `turns` rows. The client holds only `session_id`. This single rule gives you: resumption after a dropped connection, resumption after closing the tab, resumption on a different device, and safe multi-instance deployment — all with no additional machinery. The alternative (a `Map<sessionId, Conversation>` on the server) breaks all four.

```
GET /api/sessions/active
  → { session, turns[], effectiveLevel }   -- client rehydrates and continues
```

### The degraded-mode ladder

```
online, budget OK      → full AI conversation
online, budget spent   → Zero-Cost Activity (server-rendered from DB)
flaky connection       → Zero-Cost Activity from IndexedDB; writes go to outbox
fully offline          → Zero-Cost Activity from IndexedDB; writes go to outbox
offline + cache empty  → "Open the app once on wifi" + a single stored fallback phrase
```

**Offline AI conversation is out of scope.** No on-device model. Say so and stop.

### Mechanics

**PWA shell.** Manifest + a service worker caching the app shell and static assets. Worth it not for offline AI but because *"Add to Home Screen"* is what makes a daily habit app feel like an app on a phone — and that is the Core Value's delivery vehicle. Also a prerequisite if push notifications are ever added on iOS.

**Prefetch on every online load:** write today's due `review_items` (+ frames, + expected answers) into IndexedDB. This is what makes the offline ZCA possible, and it costs one extra query.

**Outbox with idempotency:**

```ts
// lib/offline/outbox.ts
type PendingAction = {
  clientId: string;   // uuid v4, generated on the device
  type: 'session.start' | 'turn.create' | 'session.end' | 'review.grade';
  payload: unknown;
  localDate: string;  // "2026-10-06" — COMPUTED ON THE CLIENT
  queuedAt: number;
};
// on reconnect: POST /api/sync { actions: PendingAction[] }
// server: every write is ON CONFLICT (…, client_id) DO NOTHING
```

**The one offline trap that matters, stated as a rule:**

> `local_date` is computed **on the client, at the moment of the activity**, and sent with the payload. The server never derives it from request arrival time.

Why: a learner completes a review offline at 11 p.m. and the phone syncs at 9 a.m. the next morning. Server-derived dating puts that session on the wrong day and **breaks the streak** — which is precisely the failure the Core Value forbids. The server validates the client date for sanity (within ±36 h of now in the user's tz) but trusts it.

**Grace window.** Independently of offline, count activity within **3 hours after local midnight** toward the previous day. This removes the "I finished at 00:04 and lost 40 days" abandonment event, which is the single most-cited streak-app complaint, at the cost of one `interval` in a query ([habit-tracker design discussion](https://dev.to/charlie_brinicombe/how-to-build-a-streaks-feature-4fck)).

---

## 10. Background Work

### Recommendation: zero scheduled jobs in v1. Everything lazy, computed on read.

| Work | Scheduler? | How to do it instead |
|---|---|---|
| **SRS due computation** | **No** | `WHERE due_at <= now() AND status='active'`. FSRS computes the next `due_at` at grade time and writes it to the row. Nothing needs to "run." |
| **Streak rollover at midnight** | **No** | Streak is a pure function of `sessions.local_date` for that user. Count back consecutive local dates from today (or yesterday, within the grace window). Cache in `streaks`, invalidate on session completion. |
| **Usage counter reset** | **No** | The `window_key` *is* the date. A new day reads a non-existent row = zero spent. This is why the generic `(user_id, window_key)` design beats a mutable `today_tokens` column — it eliminates a job. |
| **Daily content preparation** | **No** | "Today's activity" is a cheap query + rules (time budget, due reviews, last scenario, scaffold level). Pre-generating conversations wastes tokens on days the user doesn't show up and serves stale content when they do. |
| **Streak-at-risk push notification** | **Yes — and this is the only legitimate one** | Defer to a later milestone. When added: one hourly job selecting users whose local hour is 20 and who have no session today. Note: **web push on iOS requires the PWA be installed to the home screen.** |

### "Whose timezone does midnight rollover use?"

**Nobody's — because nothing runs at midnight.** This is the whole point of computing on read. The question only arises if you build a cron, and the cron is what creates the problem (you would need 38 fan-out runs, one per IANA offset, or a per-user schedule). Lazy computation makes the timezone question evaporate.

The only timezone rule that survives: all date math goes through `lib/datetime.ts`, which takes the user's IANA name and returns a `local_date`. That file is the sole place `Intl.DateTimeFormat` / `date-fns-tz` is imported. One file to audit, one file to test.

---

## 11. Recommended Project Structure

```
src/
├── app/
│   ├── (app)/
│   │   ├── page.tsx                  # Today — the one screen that matters
│   │   ├── session/[id]/page.tsx     # conversation (voice or text)
│   │   └── review/page.tsx           # Zero-Cost Activity surface
│   ├── api/
│   │   ├── sessions/route.ts
│   │   ├── sessions/[id]/turns/route.ts   # THE HOT PATH — streams
│   │   ├── sessions/active/route.ts       # resumption
│   │   ├── sync/route.ts                  # idempotent offline outbox drain
│   │   └── realtime/token/route.ts        # LATER/OPTIONAL: mints ephemeral secret
│   └── layout.tsx
├── server/                           # EVERY file: `import 'server-only'` on line 1
│   ├── env.ts                        # zod-validated; throws at boot
│   ├── db/{schema.ts, client.ts, migrations/}
│   ├── ai/{provider.ts, turn.ts, errors.ts, prompts/}
│   ├── budget/{assert.ts, record.ts, tiers.ts}
│   ├── scaffold/{level.ts, policy.ts}
│   ├── srs/{fsrs.ts, due.ts}
│   ├── streak/compute.ts
│   └── activity/{choose.ts, zero-cost.ts}
├── lib/
│   ├── speech/{index.ts, webspeech.ts, server-stt.ts}   # swappable adapter
│   ├── offline/{outbox.ts, cache.ts, sw.ts}
│   └── datetime.ts                   # THE ONLY place timezone math happens
└── content/
    ├── scenarios/*.ts                # seeded into DB on boot
    └── structure-keys.ts             # THE CLOSED ERROR TAXONOMY (~40 + 'other')
```

### Structure rationale

- **`src/server/` as a hard wall** is the mechanical enforcement of "the API key never reaches the browser." Not a convention — a build failure.
- **`lib/speech/` as an adapter** is what makes "swap Web Speech for Deepgram" a config change rather than a rewrite. This directly serves the local→deployed constraint.
- **`lib/datetime.ts` as the single tz location** is what prevents the streak-breaks-on-deploy bug (§6).
- **`content/structure-keys.ts` in the repo, not the DB,** because it is code the prompt depends on; keeping it versioned with the prompt keeps them from drifting.

---

## 12. Anti-Patterns (specific to this project)

### AP1: Reading "local-first" as a sync engine
**What people do:** reach for ElectricSQL / Automerge / RxDB / Yjs.
**Why wrong:** 5 users, each reading only their own rows, usually on one device. You inherit conflict resolution, schema versioning across clients, and a sync server — to solve a problem that does not exist.
**Instead:** conventional server + a narrow IndexedDB outbox for one activity (§9).

### AP2: Letting the model decide the scaffold level
**What people do:** put "assess the learner's level and adjust support" in the system prompt.
**Why wrong:** non-reproducible, un-unit-testable, drifts within a session, and silently regresses when the prompt is edited. The learner feels the app "forgot" them.
**Instead:** `scaffold/level.ts`, pure code over `scaffold_state`. 3 ms, deterministic, has tests.

### AP3: Sequential generate → detect → scaffold pipeline
**What people do:** three model calls per turn because it looks clean.
**Why wrong:** 2.5–4 s to first word. Voice mode dies. The beginner this app exists for quits.
**Instead:** one streaming call with `reply` first in the schema; error analysis fired in parallel at utterance arrival (§3).

### AP4: Returning 429 when the budget is exhausted
**What people do:** the standard rate-limit response.
**Why wrong:** directly violates the Core Value. A blocked day is a broken streak is an abandoned app.
**Instead:** tiered degradation ending in a $0 activity that still counts (§7).

### AP5: Free-text error labels from the model
**What people do:** `errorType: string` straight out of the model.
**Why wrong:** 200 near-synonymous keys; SRS never consolidates; the learner reviews the same mistake under four names. Retroactively unmergeable.
**Instead:** a closed enum of ~40 `structure_key`s passed into the prompt as the allowed set, plus `other`.

### AP6: Server-derived dates
**What people do:** `new Date().toISOString().slice(0,10)` on the server.
**Why wrong:** wrong timezone, wrong on offline sync, wrong after deploy (your Mac is not UTC).
**Instead:** client-computed `localDate`, server-validated, stored denormalized on `sessions`.

### AP7: Adding a scheduler before you need one
**What people do:** Inngest / Trigger.dev / BullMQ + Redis for "SRS and streaks."
**Why wrong:** neither needs it (§10). You add a second runtime, a second deploy artifact, and a second failure mode to this project's operational surface — for a cron that could have been a `WHERE` clause.
**Instead:** compute on read. Revisit only for push notifications.

### AP8: Starting with Realtime WebRTC because it's the coolest
**What people do:** build voice on `gpt-realtime` first.
**Why wrong:** it is a *second* conversation engine with a different state model and a metering model your budget system cannot fully observe — and at $0.60–1.10 per 10-minute session on the flagship, one session exceeds a sane daily budget (§7).
**Instead:** Web Speech first; Realtime only if barge-in turns out to be essential, and then on the mini model with pre-authorized session budgets.

---

## 13. Dependency Ordering for the Roadmapper

### Foundational — must exist before anything that touches a model or a date

| ID | Component | Depends on | Why it cannot be deferred |
|----|-----------|-----------|---------------------------|
| **F1** | Postgres + Drizzle + migrations; `users` (opaque uuid pk, IANA tz); `server-only` wall; zod-validated `env.ts`; real session cookie (dev-mode login behind a flag) | — | Every query that ships without `where user_id = ?` is a retrofit; the `server-only` wall is unauditable once 40 modules exist |
| **F2** | `lib/datetime.ts` + the `sessions.local_date` convention + client-computed dates + the 3 h grace window | F1 | Dates written without it are unrecoverable (§5 #3, #4) |
| **F3** | `usage_counters` + `assertBudget()` choke point + tier enum | F1, F2 | Retrofit cost scales with the number of call sites, and you will miss one |
| **F4** | `content/structure-keys.ts` — the closed error taxonomy | — | Errors recorded with free-text keys cannot be consolidated afterward (§5 #2) |
| **F5** | Idempotency contract: client-generated `session.id` / `client_turn_id`; `ON CONFLICT DO NOTHING` on every write endpoint | F1 | Retrofitting means reconciling duplicate rows by judgement |

*F1–F5 are cheap individually and ruinous to add late. They are one phase, not five.*

### Core loop — sequential

| ID | Component | Depends on |
|----|-----------|-----------|
| **C1** | Text conversation turn: streaming route handler, one structured call, `turns` written, `produced_utterance` set | F1, F2, F3, F5 |
| **C2** | Scaffold state + promotion/demotion policy (`scaffold_state`, `level.ts`, `policy.ts`) | C1 |
| **C3** | Parallel error capture → `captured_errors` | C1, F4 |
| **C4** | SRS: `review_items` + `review_logs` + `ts-fsrs` + due-on-read | C3 |
| **C5** | **Zero-Cost Activity** (`activity/zero-cost.ts`) — the keystone | C4 |
| **C6** | Streak compute-on-read + Today screen + activity chooser (`activity/choose.ts`) | C1, C5, F2 |

> **C5 is the keystone.** Busy-day path, rate-limit exhaustion, and offline mode all route into it. Scheduling it late turns three later phases into blocked phases. It is also the only thing standing between a budget overrun and a Core Value violation.

### Independent — parallelizable waves

| Component | Can start after | Notes |
|---|---|---|
| **Voice adapter** (`lib/speech/`, Web Speech impl) | C1 (needs only the text turn interface) | ⚠ **Run the accuracy spike before this phase is planned** — a bad Web Speech WER result changes scope, cost model, and latency budget (§8) |
| **PWA shell + service worker + outbox** | F5 (idempotency is the contract) | SW/manifest plumbing is independent of content; outbox content needs C5 |
| **Scenario content authoring** (`content/scenarios/`) | — | Fully independent; a non-engineering task |
| **Today screen visual design** | C6 data shape | Independent of engine work |

### Optional / last

| Component | Why last |
|---|---|
| Realtime WebRTC voice (`/api/realtime/token`) | Second engine, pre-authorized metering, 5–10× cost. Only if barge-in proves necessary. Needs F3 extended with a pre-auth/refund path. |
| Push notifications + the one hourly job | The only legitimate scheduler. Needs installed PWA on iOS. |
| Real auth provider (magic link / Google) | Dev-mode login is sufficient on localhost; F1's opaque `users.id` makes this a drop-in |
| Audio retention + `S3Store` | §7 recommends discard; the `BlobStore` interface in F1 makes this a flag |

### Expensive to defer vs cheap to defer

**Expensive (do now):** `assertBudget` choke point · idempotency keys · `structure_key` taxonomy · `review_logs` append · `server-only` wall · IANA tz + denormalized `local_date` · `BlobStore` interface · scaffold `level` as float · opaque `users.id`.

**Cheap (do NOT do now — doing so is over-engineering here):** real auth provider · object storage · any scheduler/queue/Redis · Realtime WebRTC · vector search or semantic error clustering · multi-provider LLM abstraction beyond what the AI SDK gives free · any sync engine · observability stack beyond `console.log` + the provider's own dashboard.

---

## 14. Scaling Considerations

| Scale | Adjustment |
|---|---|
| **0–10 users (this project, forever)** | Everything above. One process, one Postgres, zero jobs, zero cache layer. |
| 10–1k | Nothing changes except the Postgres instance size. The budget counter is the only contended row and it is per-user. |
| 1k+ | You are building a different product; revisit from scratch. |

**First bottleneck if it ever appears:** provider spend, not compute. The architecture's only real scaling control is §7's tiering, which already exists.

**Explicit over-engineering warning:** this table's second and third rows should not influence a single decision in v1. If a design choice is justified by them, it is the wrong choice.

---

## 15. Integration Points

### External Services

| Service | Integration pattern | Gotchas |
|---|---|---|
| LLM provider | Vercel AI SDK `streamText` + structured output, server-side only, behind `server/ai/provider.ts` | Order schema fields so `reply` streams first; model id in env, not in code |
| Browser Web Speech API | `lib/speech/webspeech.ts` | Secure context required (localhost yes, LAN IP no); Safari uses an older model, needs network, prompts about sending data to Apple; **accuracy on accented beginner speech is unverified** |
| Browser SpeechSynthesis | `lib/speech/webspeech.ts` `speak()` | Voice list loads async — wait for `voiceschanged`; iOS requires a user gesture to unlock audio |
| Streaming STT (optional, option B) | `lib/speech/server-stt.ts` behind the same adapter | Must be a *streaming* provider; batch Whisper at 1–5 s kills voice mode |
| OpenAI Realtime (optional, last) | Server mints at `POST /v1/realtime/client_secrets`; browser POSTs SDP to `/v1/realtime/calls` | Server is out of the audio path → pre-authorize, don't post-meter; set `OpenAI-Safety-Identifier` server-side |
| Postgres | Drizzle, single `DATABASE_URL` | Set process `TZ=UTC` in both environments |

### Internal Boundaries

| Boundary | Communication | Note |
|---|---|---|
| Browser ↔ server | HTTPS + session cookie + `Idempotency-Key` header | No shared types across a network boundary other than zod-validated payloads |
| `app/api/**` ↔ `server/**` | Direct import; `server/**` is `server-only` | This boundary is the security perimeter — keep it thin and obvious |
| `server/ai/turn.ts` ↔ `server/scaffold/` | Direct call, synchronous, DB-backed | Scaffold never calls the model |
| `server/ai/errors.ts` ↔ everything | Fire-and-forget; failures logged, never surfaced | Must never be able to fail a turn |
| UI ↔ `lib/speech` | Adapter interface only | The swap point for all three audio architectures |

---

## Confidence & Provenance

The GSD confidence classifier (`gsd query classify-confidence`) returns **LOW** for the `websearch` and `webfetch` providers, which were the only providers enabled in this project's config (Exa, Brave, Tavily, Firecrawl, Ref, Perplexity, Jina all `false`). Reported honestly rather than inflated:

| Claim area | Provider | Seam tier | Practical note |
|---|---|---|---|
| Realtime ephemeral-token flow, endpoints, `gpt-realtime-2.1`, safety identifier | webfetch | LOW | Read **directly from `developers.openai.com` official docs** — treat as high in practice |
| STT latency numbers (Deepgram ~150 ms TTFT, Whisper 1–5 s batch) | websearch | LOW | Consistent across three independent sources; directionally reliable, exact figures not re-measured |
| Realtime pricing ($32/$64 per 1M audio; mini $10/$20) | websearch | LOW | **Verify against `openai.com/api/pricing` before the budget phase.** Decisions here are order-of-magnitude, which the figures support regardless. |
| Drizzle SQLite↔Postgres portable-subset rules | websearch | LOW | Community guidance, not Drizzle's own docs |
| Fly.io ephemeral root filesystem | websearch | LOW | From Fly's own docs via search; high in practice |
| Streak/timezone design patterns | websearch | LOW | Design folklore, well-corroborated |
| LLM TTFT 400–800 ms | — | **ESTIMATE** | Not independently measured here. Typical for frontier small models. Measure during C1. |
| Web Speech accuracy on Vietnamese-accented beginner English | — | **UNKNOWN** | **The single biggest open question. Spike before planning the voice phase.** |

### Gaps the roadmapper should flag for phase-level research
1. **Web Speech WER on the actual learner's speech** — gates the entire voice phase's design and cost model.
2. **Current Realtime/model pricing** — verify at the budget phase; the $0.30/day default depends on it.
3. **Prompt design for scaffolded recasts** — the engine shape is settled here; the prompt content is a separate research problem and deserves its own AI-spec phase.
4. **FSRS applied to grammar structures rather than flashcards** — FSRS is validated for discrete recall items; applying it to "produces this structure correctly in conversation" is an adaptation, not a known-good use. Worth a short investigation at C4.

## Sources

- [OpenAI — Realtime API with WebRTC (official docs)](https://developers.openai.com/api/docs/guides/realtime-webrtc.md) — ephemeral client secrets, `/v1/realtime/client_secrets`, `/v1/realtime/calls`, `gpt-realtime-2.1`, `OpenAI-Safety-Identifier`
- [AI Engineer — Building Effective Voice Agents](https://ai.engineer/talks/building-effective-voice-agents) — sub-800 ms turn-taking threshold
- [Gradium — STT API Benchmark 2026: Latency and Accuracy](https://gradium.ai/content/stt-api-benchmark-2026-latency-accuracy) — Deepgram Nova-3 TTFT ~150 ms
- [Deepgram — Whisper vs Deepgram](https://www.deepgram.com/learn/whisper-vs-deepgram) — Whisper batch 1–5 s, not designed for streaming
- [Forasoft — OpenAI Realtime API pricing](https://www.forasoft.com/article/openai-realtime-api-pricing) and [eesel — GPT Realtime Mini pricing 2026](https://www.eesel.ai/blog/gpt-realtime-mini-pricing) — per-minute cost figures
- [Fly.io — SQLite / Prisma deployment](https://fly.io/docs/js/prisma/sqlite/) — ephemeral root filesystem, volumes required, single-host
- [Drizzle portable schema patterns](https://instagit.com/BuilderIO/agent-native/using-drizzle-orm-agent-native-portable-schema.md) — the SQLite↔Postgres portable column subset
- [ts-fsrs](https://npmjs.com/package/ts-fsrs) and [Spaced repetition recap — domenic.me](https://domenic.me/fsrs/) — stability/difficulty/retrievability, card state fields
- [addpipe — A Quick Look at Apple's SpeechAnalyzer API](https://blog.addpipe.com/apple-speechanalyzer-api/) — Safari's Web Speech uses an older model, no SpeechAnalyzer web surface
- [AssemblyAI — Speech recognition with the Web Speech API](https://www.assemblyai.com/blog/speech-recognition-javascript-web-speech-api) — browser SpeechRecognition mechanics
- [Vercel AI SDK docs](https://vercel.com/docs/ai-sdk) — `streamText`, `Output.object()` partial-object streaming
- [dev.to — How to build a streaks feature](https://dev.to/charlie_brinicombe/how-to-build-a-streaks-feature-4fck) and [techinterview.org — Design a Mobile Habit-Tracking App](https://www.techinterview.org/post/3233475167/design-mobile-habit-tracking-app/) — local-day boundaries, DST, grace periods

---
*Architecture research for: scaffolded AI conversation habit app (voice + text, shared provider key, localhost-first)*
*Researched: 2026-10-06*
