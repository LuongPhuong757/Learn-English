# Feature Research

**Domain:** Personal scaffolded-AI-conversation language learning app (beginner English, Vietnamese L1, habit-first)
**Researched:** 2026-10-06
**Confidence:** MEDIUM-HIGH (pedagogy and habit research HIGH; SRS-transfer-to-conversation adaptation MEDIUM — no direct prior art found; competitor internals LOW — teardowns are marketing-adjacent)

---

## The Central Finding

Three independent research lines converge on the same design, and all three **contradict what mainstream language apps ship**:

1. **Prompts beat recasts.** Lyster & Saito's meta-analysis of classroom corrective feedback found smaller effect sizes for recasts than for prompts, and specifically that *lower-proficiency learners misread recasts as conversational continuation and miss the corrective intent entirely*. Nearly every AI conversation app corrects by silent recast. For this learner that is close to no correction at all. ([Lyster & Saito meta-analysis summary](https://caslsintercom.uoregon.edu/content/22384), [explicit CF meta-analysis](https://pops.uclan.ac.uk/index.php/jsltr/article/view/360/147))

2. **Text chat is not the lesser mode.** Satar & Özdener and replications: novice learners' speaking proficiency improved in *both* text-chat and voice-chat conditions, but anxiety decreased **only** in the text group. ([Open Research Online](https://oro.open.ac.uk/17665), [CALL-EJ](https://callej.org/index.php/journal/article/download/407/336/991)) PROJECT.md's "both modes reach the same core loop" is evidence-supported, not a compromise.

3. **Missing one day is statistically irrelevant to habit formation.** Lally et al.: missing a single opportunity reduced automaticity by 0.29 — "a very small decrease" with no long-term consequence. Missing a *week* derails. ([UCL study coverage](https://scienceblog.com/5-research-backed-ways-to-build-a-habit-that-actually-lasts/)) A streak that resets to zero after one miss encodes a model of habit that the research says is wrong — and streak loss is a documented *exit point* for an app's most loyal users. ([UX teardown](https://uxdesign.cc/3-reframing-streaks-on-duolingo-5-ideas-for-a-more-healthy-and-flexible-approach-to-language-8fd89545771e))

Everything below follows from these.

---

## 1. The Daily Loop

### Design rule that governs both paths

**The streak banks on the learner's first utterance, not on session completion.** The Core Value is literally "produces at least one English utterance." Make the data model say exactly that. At ~45-75 seconds in, the day is irreversibly counted; everything after is upside. This is BJ Fogg's "shrink the behavior until it is easy to begin" made into a database write. ([Tiny Habits / Behavior Design](https://scienceblog.com/5-research-backed-ways-to-build-a-habit-that-actually-lasts/))

**The app chooses the path; the learner is never asked.** "How much time do you have?" is a decision at the exact moment decisions kill habits. Auto-select the 5-minute path when: local time ≥ 21:00, OR the last 3 sessions were under 6 minutes, OR the learner taps the persistent "quick" affordance. Otherwise run the 20-minute path — which the learner can exit at any turn with no penalty, because the day was already banked.

### 5-minute path ("busy day" — the Minimum Viable Session)

| Time | What happens | Why |
|------|--------------|-----|
| 0:00–0:10 | Open → one button, pre-labeled with the already-chosen scenario. No menu, no mode picker (defaults to last used). | Zero decisions. |
| 0:10–0:45 | AI opens with ONE short question (≤12 words) continuing last session's scenario. 2 full frames shown below the input, each with a VN gloss. | Continuity removes re-orientation cost; frames remove the empty-box failure. |
| 0:45–1:15 | **Learner's first utterance → STREAK BANKED.** Visible tick animation. | Core Value satisfied at t=75s. |
| 1:15–3:30 | 4–6 exchange turns. Scaffold ladder active. Max 1 in-flow correction total. | The actual practice. |
| 3:30–4:30 | **One** due review item, injected as a conversational question whose natural answer requires the target form. Never a flashcard screen. | Production-mode review (see §3). |
| 4:30–5:00 | Close: the learner's own sentences from today, listed. Streak + lifetime-days. Nothing else. | Fixed, content-based reward (see §4). |

### 20-minute path

| Time | What happens | Why |
|------|--------------|-----|
| 0:00–0:30 | Resume card + at most 3 scenario options, one of which is "continue yesterday." | ≤3 options is a glance, not a decision. |
| 0:30–1:30 | **Pre-task priming:** scenario framing + 3–4 target frames with VN gloss and a tap-to-hear button on each. | TBLT pre-task phase; also the only listening the beginner can actually parse. |
| 1:30–2:00 | First utterance → **STREAK BANKED**. | Same rule. |
| 2:00–12:00 | **Task cycle:** 12–18 guided turns. Ladder escalates/fades per §2. ≤1 prompt-type correction per turn. | The spine. |
| 12:00–16:00 | **Task repetition block:** the AI re-runs 2–3 moments where the learner struggled, same task, scaffold one rung *lower* than the first attempt. | Task repetition has the strongest evidence base for fluency gain of any single classroom technique; it is also free to implement (re-prompt, no new content). |
| 16:00–19:00 | Review queue: up to 3 due items, injected conversationally. | Capped — see §3. |
| 19:00–20:00 | Close: "your sentences" transcript (learner utterances only), new review items silently added, streak confirmed. | |

**Shared vs divergent:** turns 1..n are identical machinery in both paths. The 5-minute path is the 20-minute path with the priming and repetition blocks omitted and the review cap set to 1. Do **not** build two loops.

### What competitors do (and where they fail this learner)

| App | Session structure | Gap for this learner |
|-----|-------------------|----------------------|
| **Speak** | Structured curriculum, scenario role-plays, in-conversation feedback on pronunciation/grammar/fluency | Fixed curriculum assumes a daily budget; rigid path conflicts with irregular time |
| **Praktika** | Avatar lessons with scaffolding + guided questions for beginners — closest prior art | Learning paths "rigid," limited feedback, **no vocabulary retention system** — exactly the error-capture gap this project fills |
| **TalkPal** | Roleplays, debates, photo description, 57+ languages | Open-ended modes = empty box; "sentences too simple" past beginner but no withdrawal mechanic |
| **Langua** | Conversations persist across sessions / remembers prior discussions | Memory without scaffolding still fails an absolute beginner |

Sources: [TalkPal review](https://languatalk.com/blog/talkpal-review/), [best AI language app comparison](https://languatalk.com/blog/whats-the-best-ai-for-language-learning/). **Confidence LOW** — these are competitor-adjacent blogs, not independent teardowns. Treat as directional only.

**The whitespace:** nobody combines *contingent scaffold withdrawal* with *conversational error capture fed back into the same conversation*. That's the product.

---

## 2. Scaffolding Mechanics (the central bet)

### The ladder — 5 levels, stored per learner **per function**

Not a global "level." Store `scaffold_level[user_id][function_tag]`. A learner can be L0 at greetings and L4 at past-tense narration simultaneously. This is what Krashen's i+1 actually means operationally, and a single global level is the reason generic apps feel both too easy and too hard at once.

| Level | Screen shows | Learner does |
|-------|--------------|--------------|
| **L4 Full frame + bank** | 2 complete sentences with one slot: `"I'd like a ___, please."` + 4 tappable word chips + VN gloss under each frame | Taps to assemble. Voice mode: tap "hear it," then repeat. |
| **L3 Frame only** | Same 2 frames. No word bank, no gloss. | Produces the full sentence, adapting the frame. |
| **L2 Starter** | First 2–3 words only: `"I'd like…"` | Completes it. |
| **L1 Cue** | No English. A Vietnamese function cue only: `Hỏi giá` ("ask the price") | Produces from intent. |
| **L0 Open** | Nothing. | Free response. |

Evidence base: sentence frames and substitution tables are the standard EAL/ESL scaffold for oral production ([Bell Foundation scaffolding guide](https://www.bell-foundation.org.uk/resources/great-ideas/scaffolding/)); a sentence-frame intervention moved mean speaking scores 32.69 → 55.75, p < 0.001 ([UIN SGD study](https://digilib.uinsgd.ac.id/143099/)). **Confidence MEDIUM** — single small study, but consistent with the broader scaffolding literature.

### Contingency rules — exactly what triggers movement

Scaffolding has three defining properties: **contingency** (adjust to what the learner showed), **fading**, and **transfer of responsibility**. The operative rule is: *when the learner succeeds, reduce support; when the learner fails, increase it* — and support should be **as minimal as necessary for the learner to self-correct** (Aljaafreh & Lantolf's regulatory scale, the canonical L2 implementation of this). ([Vanderbilt annotated bibliography](https://my.vanderbilt.edu/l2studies/?p=121), [levels of mediation](https://journals.equinoxpub.com/LST/article/view/32867))

The most recent operationalization — CoMeT, n=131 adults — implements precisely "support rises when a learner fails at a decision point and fades on take-up," and reports it surrendered the full answer in 1 session in 16 vs 1 in 6 for a question-only tutor, with *lower* learner frustration. ([arXiv 2609.22993](https://arxiv.org/abs/2609.22993)) **Confidence MEDIUM** — different domain (Python), but it is the only quantified contingent-fading implementation found.

**FADE one level when:**
`frames_bypassed_streak >= 3` — i.e. 3 consecutive on-task turns at the current level where `frame_tapped == false` AND no unresolved error in the target form of that function.

**Instant fade:** learner taps the always-present "I've got this" control. Honor it for the rest of the session; persist only if they don't escalate within 3 turns.

**ESCALATE one level when any of:**
- (a) no input for >20s (text) or >8s with no speech energy (voice)
- (b) learner taps "help"
- (c) two consecutive turns that are L1 Vietnamese, single-word, or off-task
- (d) the same target form fails repair twice in one session

**Arbitration:** never move more than one level per turn. Escalation wins ties. Escalation is **silent** — the frames simply reappear, with no "incorrect" copy anywhere. A beginner reads "you got it wrong" as "I can't do this."

### What the AI prompt does — structured output, not prose

The model receives: current `scaffold_level`, `function_tag`, the learner's last 3 turns, `due_review_items[]`, and the VN error taxonomy (§8). It must return JSON:

```json
{
  "say": "What would you like to drink?",
  "frames": ["I'd like a ___, please.", "Can I have a ___?"],
  "word_bank": ["coffee", "tea", "water", "juice"],
  "vi_cue": "Gọi đồ uống",
  "function_tag": "order_drink",
  "errors": [
    { "learner_said": "I want coffee", "target": "I'd like a coffee",
      "type": "article_omission", "severity": 1, "blocks_meaning": false }
  ]
}
```

- `frames` is `[]` at L0/L1; `word_bank` is non-empty only at L4; `vi_cue` only at L1 and L4.
- **`say` NEVER contains a correction.** This single separation is what makes "correct without breaking flow" implementable — corrections live in `errors` and are rendered by the client, so a model failure degrades to no correction rather than to a broken conversation.
- Cap `say` at 20 words at L2+, 12 words at L3/L4. Long AI turns are noise at beginner level and eat the session.

### Correction policy (contradicts common app design)

Per turn, select the **single** highest-severity error and render it as a **prompt, not a recast**:

- Text mode: show the learner's own sentence with the error span underlined and tappable → tapping reveals the fix. The learner must act.
- Voice mode: AI says the learner's phrase up to the error with rising intonation — `"Yesterday I go…?"` — eliciting self-repair.
- If the learner self-repairs → grade the item `Easy`. If they need the reveal → `Good`. If they repeat the error → `Again`.

All other errors in the turn are logged silently and never shown. Most apps silently recast everything; the meta-analytic evidence says that is the weakest option and is specifically lost on low-proficiency learners.

---

## 3. Error Capture and Spaced Review

### Does standard SRS transfer? Partially. Three mismatches, three fixes.

**Mismatch 1 — items vs instances.** SRS needs stable item identity; a conversational error is an instance. **Fix:** normalize instance → **error pattern** keyed by `type` from a closed taxonomy, not by sentence text. ~30–60 patterns cover an entire beginner's error space. That is a tractable deck. Without this the "deck" is unbounded and ungroupable.

**Mismatch 2 — who grades.** SRS assumes learner self-report of recall. Here the grade comes from an LLM's judgment of a production attempt — noisier. **Fix:** three grades only — `Again` (repeated), `Good` (correct with scaffold), `Easy` (correct unscaffolded). Never expose a four-button grading UI; a beginner cannot distinguish Hard from Good, and that decision cost compounds daily.

**Mismatch 3 — the review modality is the disease.** The learner's diagnosed failure is "knows words but can't produce them." Recognition-mode flashcards are exactly that pathology. The literature agrees: flashcard practice "has not transferred to the situation of trying to utter a sentence," production uses different machinery than recognition, and SRS is "a supplement to deliberate practice of actual language use," with explicit doubts about grammar. ([Scott Young](https://www.scotthyoung.com/blog/2021/04/26/do-flashcards-work/), [Nihongo Pera Pera](https://nihongoperapera.com/flashcards-insufficient.html)) **Fix:** the review *event* is a conversational elicitation injected into the scenario — never a separate deck screen.

### Algorithm: FSRS v6 via `ts-fsrs`. Not SM-2. Not Leitner.

On the 700M-review Anki benchmark, FSRS-5 beat SM-2 for 97.4%+ of users: log-loss 0.291 vs 0.354, retention RMSE **5.3% vs 16.2%**, and **15–40% fewer reviews at equal retention**. ([Anki FAQ](https://faqs.ankiweb.net/what-spaced-repetition-algorithm), [SM-2 vs FSRS differences](https://l-m-sherlock.notion.site/What-are-the-main-differences-between-SM-2-and-FSRS-135c250163a180ca9029ea460136254c), [comparison](https://www.diane.app/en/guides/fsrs-vs-sm2))

"Fewer reviews at equal retention" is the operative property: in a 5-minute session, review time is in direct competition with conversation time. Leitner is rejected outright — its only advantage is being executable with physical boxes. `ts-fsrs` (npm, FSRS v6, TypeScript ESM, Node ≥16) is the reference JS implementation with `repeat()`/`next()` APIs. ([ts-fsrs](https://npmjs.com/package/ts-fsrs))

**Honest caveat:** FSRS parameters are trained on flashcard review logs. With ~50 items and ~5 users there will *never* be enough history for per-user optimization. **Use default FSRS-6 parameters and do not build the optimizer.** Confidence that FSRS's measured advantage fully survives transfer to production-mode conversational items: **MEDIUM**. Confidence it is no worse than SM-2: **HIGH**. Either way the scheduler is a swappable module — keep it behind an interface.

### Adaptation specifics (all P1 if review ships)

- **Maturation gate:** do not create a review item until a pattern occurs **3 times across ≥2 distinct sessions.** Single instances are slips, typos, and ASR noise. *This threshold is the single highest-leverage tuning knob in the product* — set it too low and the queue fills with garbage and the learner stops trusting it.
- **Hard cap: 3 due items per 20-min session, 1 per 5-min session.** An overflowing queue is the canonical Anki abandonment mode and here it would eat the conversation. Overflow carries forward; it does **not** lapse.
- **Injection, not appending.** Pass `due_items` into the AI prompt with the instruction to construct a turn whose natural answer requires that form.
- **"Not elicited" ≠ "forgot."** If the AI fails to create an opportunity within 2 turns, skip the item and reschedule unchanged. Never grade `Again` for the model's failure.
- **Retirement:** a pattern that goes 3 consecutive sessions without recurrence in free production is retired and shown in the progress view. This gives the review system a visible win condition.

---

## 4. Streak and Habit Mechanics

### What the evidence actually says

| Finding | Source | Implication |
|---------|--------|-------------|
| Loss aversion 2–2.5×; streaks drive 20–40% DAU lift | [behavioral/gamification summary](https://mwm.ai/glossary/daily-streak) | The mechanic works. Keep a streak. |
| Streak loss is "a convenient exit point for its most loyal users"; most feel **relieved** when a long streak ends | [UX teardown](https://uxdesign.cc/3-reframing-streaks-on-duolingo-5-ideas-for-a-more-healthy-and-flexible-approach-to-language-8fd89545771e) | The streak is a liability at precisely the moment it is most valuable. Loss must be softened. |
| Missing **one** day: automaticity −0.29 (negligible). Missing a **week**: derails. | [Lally et al. coverage](https://scienceblog.com/5-research-backed-ways-to-build-a-habit-that-actually-lasts/) | Zeroing a streak after one miss is a **false signal**. Model the science, not the folk theory. |
| Weekend Amulet: +4% week-later return, 5% fewer lost streaks. Streak Wager: +14% D7. | [Duolingo blog](https://blog.duolingo.com/how-streaks-keep-duolingo-learners-committed-to-their-language-goals) | The measurable wins are **loss-mitigation**, not loss-amplification. |
| Bingers abandon more than pacers | same | Do not offer "double up to catch up." |
| Streak gamification fosters **app dependency** rather than autonomous habit | [IJMAT case study](https://www.wr-publishing.org/index.php/ijmat/article/view/908) | With no monetization, there is no reason to buy engagement with a slot machine. |

### Concrete mechanics

1. **Bank on first utterance.** (§1)
2. **Two numbers; the streak is the smaller one.** Primary = **"Days active: 47"** — lifetime, monotonic, can never decrease. Secondary = current streak. The honest number is the big one, and it keeps rising after a miss, which is the entire point.
3. **Free, automatic, retroactive repair — 2 per calendar month.** Missed yesterday? On open: *"Yesterday is covered. Let's keep going."* Silently restore. **Do not sell it. Do not make it a currency. Do not require pre-purchase.** Duolingo's freeze-as-gem-purchase is a monetization artifact; copy the mechanic, drop the price. Cap at 2/month so it doesn't degrade into every-other-day.
4. **Gap > 3 days → no loss screen.** Show *"Welcome back — 47 days total."* then immediately run the 5-minute path. Never show a zeroed counter or a broken flame.
5. **No variable rewards. No points, gems, XP, leagues, chests.** Variable-ratio reinforcement is what makes apps compulsive rather than habitual, and the user explicitly rejected gamified-streaks-as-currency. The reinforcer here is **fixed and content-based**: at every close, show the learner their own sentences. It's honest, it's free, and it's the only reward that is literally the Core Value.
6. **Notifications: one per day, learner-set fixed time, one "snooze to tonight."** Duolingo's multi-armed-bandit timing optimization produced **+0.5% DAU** ([Normcore Tech teardown](https://vicki.substack.com/p/duo-the-push-and-the-bandits)) — a rounding error that cost an ML system. At 5 users this is unambiguously not worth building. **Defer timing ML permanently.**
7. **Busy-day is a mode the app chooses.** (§1) Never a question.

### What works without social accountability

The user declined social features, which removes the strongest external accountability lever. Three intrinsic substitutes, in evidence-strength order:

- **Self-comparison across time** — show a transcript from 3 weeks ago next to today's. For an absolute beginner this delta is dramatic and arrives fast.
- **The content artifact** — "your sentences" is a growing record the learner authored. Production artifacts are a stronger retention object than abstract counters.
- **Scaffold position as visible competence** — "Ordering food: you now do this without help" (§6).

---

## 5. Voice vs Text

**Architect voice as an I/O adapter over the shared turn engine, not as a parallel loop.** Shared: turn engine, scaffold ladder, error capture, review injection, scenario state, streak banking — all of it. The mode flag changes rendering, input capture, and exactly two behaviors below.

| Need | Text only | Voice only |
|------|-----------|------------|
| Frame interaction | Tappable frame chips + word-bank assembly UI | Tap-to-hear model audio on **every** frame before production |
| L1 support | Inline VN gloss | VN gloss as text under the audio |
| Composition | Edit-before-send — **this is where the anxiety reduction comes from** | — |
| Input capture | — | **Push-to-talk, not VAD.** VAD cuts off beginners who pause mid-sentence; this is a concrete, common, fatal failure. |
| Transparency | — | Always-visible ASR transcript + one-tap "that's not what I said" |
| Pacing | — | Configurable AI speech rate, **0.8× default** |
| Escape hatch | — | One-tap drop to text mid-session, conversation preserved |

### The divergence that actually matters: error capture must differ by mode

ASR word error rate on non-native accented English runs **~20% higher** than General American on average, and for some L1 backgrounds open-source baselines hit **38–55% WER vs 2–4%** on clean native speech. ([arXiv 2503.06924](https://arxiv.org/abs/2503.06924v2), [Nepali-accented eval dataset](https://www.nepjol.info/index.php/east/article/view/98619))

Therefore, in v1:
- **Never create a review item from a pronunciation error detected in voice mode.** You cannot separate learner error from ASR error, and a false correction from a "teacher" destroys trust fast.
- Grammar/word-choice errors extracted from the transcript **are** acceptable — a transcript that is grammatically wrong usually reflects real learner output.
- Pronunciation scoring: **deferred entirely.** (§7)

### The counterintuitive call

Text is the **lower-anxiety on-ramp that still produces speaking gains** (§Central Finding). Combined with the Core Value's own wording — "spoken **or typed**" — this means **text mode alone satisfies the Core Value**, and voice can ship in v1.x without the product failing. This is a deliberate disagreement with PROJECT.md's Active requirement list; see §MVP for the mitigation (ship the adapter boundary in v1 so voice is additive, not a re-architecture).

---

## 6. Progress Signals Beyond the Streak

### Honest and motivating

| Signal | Why honest | Priority |
|--------|-----------|----------|
| **Days active (lifetime, monotonic)** | Can never go down. Matches the habit science. | P1 |
| **"Your sentences"** — every utterance the learner produced, newest first, dated | Zero computation, highest emotional payload, and it is *literally identical* to the Core Value | P1 |
| **Scaffold position per function** — "Ordering food: no help needed. Talking about yesterday: still using frames." | The only metric that measures the thing the product claims to do | P1 |
| **Utterances produced (count)** | Trivially true, directly on-goal | P1 |
| **Error patterns retired** (3 clean sessions) | Gives the review system a visible win condition | P2 |
| **Longest unscaffolded turn (words)** | Crude but monotonic fluency proxy; moves fast for a beginner | P2 |
| **Then-vs-now transcript pair** | Self-comparison; the strongest intrinsic motivator available without social | P2 |

### Motivating but misleading — do not ship

| Metric | Why it's a trap |
|--------|-----------------|
| **Words learned / vocab count** | Counts exposure, not production. **This is the exact metric that produced "knows words but can't produce them."** Shipping it would re-create the failure the product exists to fix. |
| **Accuracy % / error rate** | Goes **down** as the learner improves, because fading the scaffold increases errors *by design*. An improving learner would watch a worsening number. Actively harmful. |
| **CEFR level estimate** | Not defensible from minutes/day of chat. Learners anchor hard on a level number and then optimize for it. |
| **Pronunciation score** | Partly measurement noise (§5). "You scored 61%" on a contrast the learner cannot yet *hear* is demotivating and unactionable. |
| **Time studied** | Rewards duration. Undermines the busy-day path — a 5-min day must be worth the same as a 40-min day. |
| **XP / points** | Currency without a store is meaningless; currency with a store is the gamification the user rejected. |

---

## 7. Anti-Features

| Anti-feature | Why it's requested | Why it harms **this** learner | Instead |
|---|---|---|---|
| **Open chat box, "Let's talk!"** | Feels like the real thing | The documented day-3 killer; an absolute beginner produces nothing | Always-present frames at L2+ |
| **Multiple-choice / tap-the-tiles exercises** | Duolingo's core loop; feels productive, easy to build | Recognition, not production — **it is the diagnosed cause of this learner's failure** | Every interaction produces a full utterance |
| **Hearts / lives / failure states** | Stakes feel motivating | Can cap a day at **zero** utterances. Directly contradicts the Core Value. | No failure state exists. Ever. |
| **Streak as purchasable currency, leagues, gems** | Proven DAU driver | Explicitly rejected by the user; fosters app dependency over habit | Free auto-repair (§4) |
| **Locked lesson tree / fixed curriculum** | Feels structured | Assumes a daily budget the learner doesn't have; generates "I'm behind" guilt | Lateral scenario selection, nothing gated |
| **Pronunciation scoring** | Core to Speak/ELSA marketing | ASR WER on accented speech makes the score partly noise (§5) | Model audio + self-comparison recording (v2) |
| **Separate flashcard deck screen** | Standard SRS UX | Splits the loop in two, and the second half is what gets skipped on busy days | Review injected into conversation (§3) |
| **Long AI replies / native-speed audio** | Feels "authentic" | Unparseable at beginner level; eats the session; rejected in PROJECT.md | Cap AI turns at 12–20 words; 0.8× speech |
| **Onboarding placement test** | "Personalizes the experience" | A barrier in front of the first utterance | Start everyone at L4 turn one; the ladder finds the level in ~2 sessions |
| **Daily goal selector (casual/regular/intense)** | Feels like autonomy | A decision at the door | The app picks (§1) |
| **Explicit grammar lessons / rule explanations** | Learners ask for them | Declarative rules don't transfer to real-time production; consumes the whole 5-min session | VN cue at L1 + prompt-type correction |

---

## Feature Dependencies

```
[Conversation turn engine + structured JSON output]
    ├──required by──> [Scaffold ladder L4..L0]
    │                     └──required by──> [Progress: scaffold position]
    ├──required by──> [Streak banking on first utterance]
    │                     └──required by──> [Busy-day auto-selection]
    │                     └──required by──> [Lifetime days + auto-repair]
    ├──required by──> [Utterance persistence] ──> ["Your sentences" view]
    │                                         ──> [Then-vs-now transcript pair]
    ├──required by──> [Error capture]
    │                     ├──requires──> [Seeded VN-L1 error taxonomy (closed list)]
    │                     └──requires──> [Pattern maturation gate: 3 occurrences / 2 sessions]
    │                           └──required by──> [FSRS scheduler (ts-fsrs)]
    │                                 └──required by──> [Conversational review injection]
    │                                       └──requires──> [Per-session review cap]
    └──required by──> [Voice I/O adapter] ──> [Push-to-talk, ASR transcript, model audio]

[Per-user isolation + rate limiting] ──gates──> [any multi-user use]

[Pronunciation scoring] ──CONFLICTS──> [Error capture reliability]   (ASR noise)
[Unbounded review queue] ──CONFLICTS──> [5-minute path]              (review eats conversation)
[Global single difficulty level] ──CONFLICTS──> [Per-function scaffold ladder]
```

### Dependency notes

- **Structured JSON output is the keystone.** The scaffold ladder, error capture, and review injection all read from the same response object. If the model returns prose, all three are unbuildable. Lock the schema in the same phase as the turn engine.
- **Streak banking depends only on the turn engine.** It is the cheapest possible proof of the Core Value and should land in the earliest shippable phase, before scaffolding is tuned.
- **Review injection cannot ship before the maturation gate and the per-session cap.** Shipping review without both produces a garbage queue that eats the busy-day session — the two things that killed the learner's previous attempts.
- **The voice adapter boundary must exist in v1 even if voice doesn't.** Define the turn engine's input as `{text, source: 'typed'|'spoken', confidence?}` from day one. That makes voice additive rather than a re-architecture, which is what PROJECT.md's local-first/deploy-later constraint requires anyway.
- **Pronunciation scoring and error capture must not share a phase.** They have opposite tolerance for ASR noise.

---

## MVP Definition

### Launch With (v1) — the Core Value fails without these

- [ ] **Conversation turn engine with locked structured-output schema** — everything else reads from it
- [ ] **Text mode, end to end** — satisfies "spoken **or typed**"; lower-anxiety on-ramp with equal proficiency gains
- [ ] **Scaffold ladder L4→L0 with the §2 contingency rules, per-function state** — the central bet; without it, empty box, abandonment by day three
- [ ] **Streak banked on first utterance + lifetime "days active" counter** — the Core Value made into a database write
- [ ] **Busy-day auto-selection (app chooses, never asks)** — irregular time is the stated #1 risk
- [ ] **"Your sentences" session-close view** — the only honest reward, and it's nearly free
- [ ] **Prompt-type in-flow correction, ≤1 per turn, never in `say`** — recasts are lost on this proficiency level
- [ ] **Error capture into the seeded VN-L1 taxonomy — capture only, no review yet** — builds the dataset the review system needs; shipping review on week-one data would produce noise
- [ ] **Per-user data isolation + per-user API rate limiting** — hard constraint from PROJECT.md; gates any friend using it
- [ ] **Seeded VN-L1 error taxonomy (~25–35 closed-list patterns)** — a data file, not a feature, but review is ungroupable without it

### Add After Validation (v1.x) — should-have

- [ ] **Voice mode** (push-to-talk, model audio on frames, visible ASR transcript, 0.8× rate, mid-session drop-to-text) — *trigger: text loop sustains 14 consecutive days*. Flagged tension: PROJECT.md lists this as an Active v1 requirement; recommend demoting it with the adapter boundary shipped in v1 as mitigation.
- [ ] **FSRS review injection** (ts-fsrs defaults, 3-grade, maturation gate, per-session cap, "not elicited ≠ forgot") — *trigger: ≥20 matured error patterns exist*
- [ ] **Free auto streak repair, 2/month, retroactive** — *trigger: first missed day occurs*
- [ ] **Scaffold-position progress view** — *trigger: any function reaches L1 or L0*
- [ ] **Task-repetition focus block in the 20-min path** — *trigger: sessions routinely exceed 12 min*
- [ ] **One fixed-time daily reminder with snooze** — *trigger: deployed to a URL / phone access exists*
- [ ] **Error patterns retired + longest-unscaffolded-turn metrics**
- [ ] **Then-vs-now transcript comparison** — *trigger: 21+ days of history*

### Future Consideration (v2+) — defer

- [ ] **Pronunciation scoring / phoneme feedback** — ASR noise on accented speech makes it partly unreliable and demotivating at beginner level
- [ ] **FSRS per-user parameter optimization** — mathematically impossible at ~50 items / 5 users
- [ ] **Notification timing ML / bandits** — Duolingo got +0.5% DAU from this; nonsensical at 5 users
- [ ] **CEFR level estimation** — not defensible from the available signal
- [ ] **Self-recording playback vs model audio** — good idea, but needs voice mode mature first
- [ ] **Scenario authoring UI** — hand-write 10–15 scenarios as data; a UI for 5 users is pure overhead
- [ ] **Offline mode** — the loop requires an API call by definition
- [ ] **Learner-supplied content** — explicitly rejected in PROJECT.md; also fails exactly on the lazy days the product is designed for

---

## Feature Prioritization Matrix

| Feature | User Value | Impl. Cost | Priority |
|---------|-----------|------------|----------|
| Turn engine + structured output schema | HIGH | MEDIUM | P1 |
| Scaffold ladder + contingency rules | HIGH | MEDIUM | P1 |
| Streak banked on first utterance | HIGH | LOW | P1 |
| Busy-day auto-selection | HIGH | LOW | P1 |
| Text mode UI (frames, word bank, gloss) | HIGH | MEDIUM | P1 |
| "Your sentences" view | HIGH | LOW | P1 |
| Prompt-type correction rendering | HIGH | LOW | P1 |
| Error capture + VN taxonomy | MEDIUM (now) / HIGH (later) | MEDIUM | P1 |
| Per-user isolation + rate limit | HIGH (constraint) | LOW | P1 |
| Voice mode adapter boundary (no voice yet) | — (enabling) | LOW | P1 |
| FSRS review injection | HIGH | MEDIUM | P2 |
| Voice mode implementation | HIGH | HIGH | P2 |
| Auto streak repair | MEDIUM | LOW | P2 |
| Scaffold-position progress view | MEDIUM | LOW | P2 |
| Task-repetition block | MEDIUM | LOW | P2 |
| Daily reminder | MEDIUM | LOW | P2 |
| Then-vs-now comparison | MEDIUM | LOW | P2 |
| Pronunciation scoring | LOW (net negative now) | HIGH | P3 |
| FSRS optimizer / notification ML / CEFR | LOW | HIGH | P3 |

---

## 8. Vietnamese L1 — Target It, But Only as a Prior

**Should the app target Vietnamese-specific difficulties? Yes — invisibly, as the error taxonomy's vocabulary. Never as a curriculum.** Explicit "Vietnamese speakers struggle with final consonants" lessons are declarative grammar instruction wearing a hat (see anti-features).

### Predictable difficulties (evidence-backed)

| Area | Specific | Root cause in Vietnamese | Severity for a *communication* goal |
|------|----------|--------------------------|-------------------------------------|
| **Final consonants** | /s/, /z/, /t/, /v/, /ks/, /dʒ/ dropped or softened | Vietnamese syllables rarely end in these | **HIGH** — kills plurals, past tense, 3rd person |
| **Consonant clusters** | `asked`, `texts`, `strength` | Vietnamese has no clusters | MEDIUM |
| **Articles** | a / an / the omitted or misused | **Vietnamese has no articles at all** | **LOW** — rarely blocks meaning; deliberately deprioritize |
| **Tense marking** | past/perfect endings dropped; complex tenses avoided entirely | Vietnamese marks time with a preverbal particle/adverb, not inflection | **HIGH** — blocks meaning, and *avoidance* is invisible to naive error detection |
| **Plural -s** | dropped | Vietnamese doesn't inflect for number | MEDIUM-HIGH (compounds with final-consonant issue) |
| **S-V agreement** | `he go` | No agreement in Vietnamese | MEDIUM |
| **/l/ vs /r/, long vowels** | minimal-pair confusion | Phoneme inventory gap | MEDIUM |

Sources: [TEFL Academy: common mistakes of Vietnamese learners](https://www.theteflacademy.com/blog/common-mistakes-of-vietnamese-learners-of-english/), [final consonants & clusters study, Dong Nai University](https://vjol.info.vn/NNDS/article/view/20281), [llexi L1 error reference](https://llexi.com/l1-errors/common-english-errors-vietnamese-speakers.html). **Confidence HIGH** — consistent across independent sources.

### How it becomes code

1. **A closed list of ~25–35 `error.type` values** ships as a data file. The AI's `errors[].type` must be drawn from it — free-text types make the deck ungroupable.
2. **Severity weights encode the goal.** `article_omission` = severity 1 (never prompted in-flow, logged only). `past_tense_omission`, `plural_s_omission`, `final_consonant_deletion` = severity 2–3 (eligible for the single in-flow prompt). This is where "communication, not correctness" becomes an actual number in a config file.
3. **Avoidance detection** (should-have, v1.x): track *target forms the scenario invited but the learner never attempted*. Vietnamese learners famously **avoid** complex tenses rather than producing them wrong — a pure error-capture system is blind to this. Count `function_tag` invitations vs attempts.
4. **VN gloss at L4 and L1 only.** L1 use is a documented contributor to successful scaffolding in language classrooms — it is a rung on the ladder, not cheating. It disappears automatically at L3+ as the ladder fades.

---

## Confidence Assessment

| Area | Confidence | Why |
|------|-----------|-----|
| Scaffolding pedagogy (frames, contingency, fading) | **HIGH** | Converging: Vygotsky/ZPD, Aljaafreh & Lantolf regulatory scale, Bell Foundation practice, CoMeT quantification |
| Correction policy (prompts > recasts) | **HIGH** | Lyster & Saito meta-analysis, with an explicit low-proficiency-learner finding |
| Streak/habit mechanics | **HIGH** | Lally automaticity data + Duolingo's own published A/B numbers + independent teardowns |
| Vietnamese L1 error inventory | **HIGH** | Consistent across independent VN academic and TEFL sources |
| Text-vs-voice anxiety finding | **MEDIUM-HIGH** | Satar & Özdener plus replications; small Ns, consistent direction |
| FSRS superiority **as an algorithm** | **HIGH** | 700M-review benchmark |
| FSRS **transfer to conversational error items** | **MEDIUM** | No prior art found. The adaptation (pattern normalization, 3-grade, maturation gate, elicitation-based review) is reasoned from first principles, not measured. Keep the scheduler behind an interface. |
| Specific numeric thresholds (3 bypassed turns, 20s silence, 3-occurrence maturation, 3-item cap) | **LOW** | Defensible starting points, **not** research-derived. Treat every one as a config constant and expect to tune. |
| Competitor session internals | **LOW** | Only competitor-adjacent blogs were accessible; no independent teardowns found |

## Gaps

- **No prior art for SRS over conversational errors.** The §3 adaptation is this research's main original synthesis and is the biggest open risk in the product.
- **Avoidance detection is under-specified.** It is the right idea for a Vietnamese L1 learner but neither the literature search nor competitor review produced an implementable method. Needs phase-specific research.
- **No evidence found on habit retention for non-social, non-gamified learning apps specifically.** Nearly all retention data comes from gamified/social products. The three intrinsic substitutes in §4 are reasoned, not measured. Flag the roadmap phase covering progress signals for deeper research.
- **ASR vendor choice is unresolved here** (belongs in STACK.md), but §5's conclusion — no pronunciation review items in v1 — holds regardless of vendor.

## Sources

- Lyster & Saito oral CF meta-analysis — https://caslsintercom.uoregon.edu/content/22384 · https://pops.uclan.ac.uk/index.php/jsltr/article/view/360/147
- Aljaafreh & Lantolf regulatory scale / ZPD mediation — https://my.vanderbilt.edu/l2studies/?p=121 · https://journals.equinoxpub.com/LST/article/view/32867
- Bell Foundation scaffolding & substitution tables — https://www.bell-foundation.org.uk/resources/great-ideas/scaffolding/
- Sentence frames and EFL speaking performance — https://digilib.uinsgd.ac.id/143099/
- CoMeT: contingent escalation and fading (n=131) — https://arxiv.org/abs/2609.22993
- Satar & Özdener, text vs voice CMC: proficiency and anxiety — https://oro.open.ac.uk/17665 · https://callej.org/index.php/journal/article/download/407/336/991
- FSRS vs SM-2 benchmark — https://faqs.ankiweb.net/what-spaced-repetition-algorithm · https://l-m-sherlock.notion.site/What-are-the-main-differences-between-SM-2-and-FSRS-135c250163a180ca9029ea460136254c · https://www.diane.app/en/guides/fsrs-vs-sm2
- ts-fsrs implementation — https://npmjs.com/package/ts-fsrs
- Flashcard transfer limits — https://www.scotthyoung.com/blog/2021/04/26/do-flashcards-work/ · https://nihongoperapera.com/flashcards-insufficient.html
- Lally et al. habit automaticity / missing a day — https://scienceblog.com/5-research-backed-ways-to-build-a-habit-that-actually-lasts/
- Duolingo streak mechanics and A/B results — https://blog.duolingo.com/how-streaks-keep-duolingo-learners-committed-to-their-language-goals
- Streak loss as exit point — https://uxdesign.cc/3-reframing-streaks-on-duolingo-5-ideas-for-a-more-healthy-and-flexible-approach-to-language-8fd89545771e
- Gamification → app dependency vs habit — https://www.wr-publishing.org/index.php/ijmat/article/view/908
- Duolingo push notification bandits (+0.5% DAU) — https://vicki.substack.com/p/duo-the-push-and-the-bandits
- ASR accuracy on non-native English — https://arxiv.org/abs/2503.06924v2 · https://www.nepjol.info/index.php/east/article/view/98619
- Vietnamese L1 English errors — https://www.theteflacademy.com/blog/common-mistakes-of-vietnamese-learners-of-english/ · https://vjol.info.vn/NNDS/article/view/20281 · https://llexi.com/l1-errors/common-english-errors-vietnamese-speakers.html
- Competitor landscape (LOW confidence, vendor-adjacent) — https://languatalk.com/blog/talkpal-review/ · https://languatalk.com/blog/whats-the-best-ai-for-language-learning/

---
*Feature research for: scaffolded AI conversation, beginner English, Vietnamese L1, habit-first*
*Researched: 2026-10-06*
