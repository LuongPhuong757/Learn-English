# Pitfalls Research

**Domain:** AI-scaffolded conversational language learning (beginner L1-Vietnamese → spoken English), solo-built personal web app, voice + text, shared API key
**Researched:** 2026-10-06
**Confidence:** HIGH on STT error rates, LLM level drift, retention benchmarks, cost and latency numbers (all externally sourced with figures). MEDIUM on the specific interaction of these with an absolute beginner (little published work on true A0 learners with LLM partners — most studies use A2–B1 university students). LOW on nothing claimed as fact below.

---

## Read This First: Three Findings That Should Change Scope

Before the pitfall list, three results are severe enough that mitigating them is not enough — the product definition should move.

**1. STT cannot be allowed to judge this learner. Not "should be careful" — cannot.**
On the L2-ARCTIC corpus, Vietnamese-L1 speakers had a mean Match Error Rate of **0.143 vs 0.007 for US-English speakers — a ~20× gap**, with Vietnamese male speakers at **0.181** ([Automatic Speech Recognition for Non-Native English, arXiv 2503.06924](https://arxiv.org/pdf/2503.06924)). Critically, L2-ARCTIC speakers are *not beginners* — they are fluent L2 users reading prepared sentences. A "mất gốc" learner producing a halting five-word utterance is well outside that distribution; plan for 25–40% WER in the first months. At 25% WER, an 8-word sentence contains ~2 misrecognised words — **nearly every utterance will contain an STT error**. Any feature that compares STT output to a target and tells the learner they were wrong will be wrong most of the time, aimed at the one user who has already abandoned three products.
→ **Scope change: cut pronunciation scoring and STT-based correctness judging from v1 and v2.** Voice at beginner level is listen-and-repeat, where STT is used only as a loose "did sound come out" gate, never as a verdict.

**2. Realtime speech-to-speech is the wrong v1 architecture on cost, latency, and control simultaneously.**
Realtime voice APIs run roughly **$0.06–0.11/min flagship, $0.02–0.05/min mini**, with one developer billed **~$6 for a 75-second session** and runaway sessions reaching **$0.46/min** ([Forasoft, OpenAI Realtime API pricing](https://www.forasoft.com/article/openai-realtime-api-pricing); [OpenAI community: Realtime API extremely expensive](https://community.openai.com/t/realtime-api-extremely-expensive/966825)). Twenty minutes a day × 30 days × 5 users at $0.08/min is **~$240/month** on a personally funded key. A staged push-to-talk pipeline (batch STT → text LLM → streaming TTS) costs roughly **$1–3/user/month** for the same session count. Realtime also hands you the LLM's turn-taking and gives you no clean transcript to attach scaffolding, correction, or error-capture to.
→ **Scope change: push-to-talk, staged pipeline, no realtime API in v1.**

**3. The empty input box must not exist at beginner level — which means v1 "conversation" is a constrained-choice protocol, not free chat.**
Controlled-generation research found that **prompting alone produced beginner-comprehensible output only 39.4% of the time, vs 83.3% with constrained generation** ([Toward Beginner-Friendly LLMs for Language Learning, arXiv 2506.04072](https://arxiv.org/html/2506.04072v2)). Separately, CEFR level distinctions in prompted tutors **collapse into each other by turn 8–9 of a dialogue** ([Alignment Drift in CEFR-prompted LLMs, arXiv 2505.08351](https://arxiv.org/html/2505.08351v2)). A free-chat architecture with a system prompt saying "respond at A1" is a known-failing design.
→ **Scope change: the beginner turn is "pick one of 3 options" or "fill one blank in a given frame". Free text input unlocks by level, it is not the default.**

---

## Critical Pitfalls

### Pitfall 1: The blank-input-box freeze (highest-probability killer)

**What goes wrong:** The app presents an AI partner and a text/mic input. The learner — who by self-report "knows words but can't produce them" — has nothing to type. They stare, feel the familiar shame, close the tab. This is the exact abandonment pattern PROJECT.md already names, and it recurs even when the UI *does* have scaffolding, because the scaffold is rendered as advice ("you could say…") next to an empty box rather than as the input mechanism itself.

**Why it happens:** Developers build the chat UI first because it is the fun part and because *they* can carry the conversation when testing. The scaffolding gets bolted on as a hint panel. A hint panel still requires the learner to type.

**How to avoid (architectural):** Make the scaffold the input, not a hint beside it.
- The turn payload from the server must always carry `scaffold_options: [{display, target_utterance}]` and `frame: {template, blank_slot, candidate_fillers[]}`. This is a protocol-level decision; a hint panel cannot be retrofitted into it without redoing the turn contract.
- At beginner level, render 2–4 tappable complete sentences plus one frame with a single swappable slot. Production = select + substitute one word. That is a real utterance and it counts.
- The free-text box is feature-flagged off by level. It appears when the learner has produced N unscaffolded-eligible turns, not on day one.
- Ship a "say it for me" button: tapping a frame plays TTS first, then the learner repeats. For a true beginner, repetition precedes production.

**Warning signs:** In testing, the developer never uses the scaffold options because they can type faster. The learner's first session has >15s median time-to-first-input. Any session with zero utterances produced.

**Phase to address:** Phase 1 — it *is* the product.

---

### Pitfall 2: LLM level drift (certain to occur, prompting will not fix it)

**What goes wrong:** Turns 1–3 are perfect A1. By turn 6 the model is using "actually", "would you mind", relative clauses, and 15-word sentences. The learner stops understanding, stops responding, leaves. Measured directly: readability distributions for A1/B1/C1-prompted tutors **converge to near-complete overlap by turns 8–9** across models ([arXiv 2505.08351](https://arxiv.org/html/2505.08351v2)). The paper's conclusion is blunt: "prompt engineering in and of itself may not be enough to fully constrain LLM behavior."

**Why it happens:** The system prompt's influence decays relative to the growing conversational context, and the in-context examples the model is anchoring on are its own increasingly complex prior turns.

**How to avoid — ranked by leverage, with honest costs:**

| Mitigation | Effectiveness | Cost | Architectural? |
|---|---|---|---|
| **Context windowing**: send only last 3–4 turns + a compact structured state summary, never the whole transcript | HIGH — removes the drift feedback loop at its source; simultaneously cuts token cost and latency | ~1 extra cheap summarisation call per 5 turns | **YES — phase 1.** Retrofitting a windowing layer into code that passes `messages[]` straight through touches every call site |
| **Vocabulary allowlist + post-hoc validator**: constrain to NGSL top ~800 + scenario words; tokenise the output, check every content word, regenerate on violation | HIGH — this is the mechanism behind 39.4% → 83.3% | +1 call on ~10–25% of turns; +400–800ms on those turns; one-time cost to build the word list | **YES — phase 1.** The generate→validate→regenerate loop is a pipeline shape, not a feature |
| **Hard sentence-length cap** (≤8 words/sentence, ≤2 sentences at A1), validated deterministically before sending | MEDIUM-HIGH, and it is nearly free | ~0 | Yes, same validator |
| **Few-shot anchoring**: 3–4 fixed exemplar turns at target level in *every* request | MEDIUM — slows drift, does not stop it | ~200 input tokens/turn (cacheable) | Cheap to add later |
| **Periodic re-grounding**: re-inject the level instruction as a system turn every N turns | LOW-MEDIUM — helps, measurably less than windowing | ~0 | Later |
| **Readability formulas (Flesch-Kincaid, etc.) as the gate** | **DO NOT** — these formulas are unreliable on 6-word conversational turns; they were validated on paragraphs of prose | — | — |

Use vocabulary-list coverage + word count as the gate. Not readability scores.

**Warning signs:** Log the out-of-vocabulary word count and mean sentence length per assistant turn from day one and chart them against turn index. Drift is visible in that chart before it is visible in the learner's behaviour.

**Phase to address:** Phase 1 (windowing + validator), Phase 2 (few-shot tuning).

---

### Pitfall 3: The encouraging-partner / honest-assessor conflict (the sycophancy trap)

**What goes wrong:** The same model call is asked to be a warm, confidence-building conversation partner *and* to judge whether the learner's utterance was correct. It will not judge honestly. State-of-the-art models agree with users' wrong answers **more than 24% of the time, with some models at 58–60%** ([Understanding Sycophancy in LLMs, arXiv 2602.01002](https://www.alphaxiv.org/overview/2602.01002)). Preference post-training *amplifies* this: in one Q&A setup, false positives rose from **46.7% to 70.2% after RLHF** ([ibid.](https://arxiv.org/pdf/2602.01002)). The product consequence is specific and severe: the learner practises daily for three months, is told "Great job!" every time, and discovers in a real conversation that nobody understands them. That is a worse abandonment than quitting on day three, because it costs three months of belief.

**Why it happens:** One prompt, one call, because it's simpler and cheaper.

**How to avoid (architectural, phase 1):**
- **Two separate calls with separate prompts.** The *partner* call generates the reply and never evaluates. The *assessor* call receives only `{learner_utterance, expected_target, scenario_context}` — no persona, no "be encouraging", no conversation history — and returns strict JSON: `{intelligible: bool, errors: [{type, span, correction}], one_thing_to_fix: string|null}`.
- The assessor's prompt must not contain the word "student", "learner", or "encourage". Frame it as a text-annotation task, not a teaching task.
- The partner's warmth is applied to the *delivery* of the assessor's verdict, never to its content.
- Keep an honest private metric the learner never sees: % of utterances the assessor marked fully correct, trended weekly. If that line is flat at 95% from week one, the assessor is sycophantic, not the learner perfect.

This split cannot be bolted on later without rewriting the turn pipeline and re-deriving every stored error record.

**Warning signs:** Assessor returns `errors: []` on a deliberately broken test input ("I yesterday go school"). Build that as a fixture test on day one.

**Phase to address:** Phase 1.

---

### Pitfall 4: Over-correction destroys flow; under-correction fossilises errors (both are real, the fix is asymmetric)

**What goes wrong:** Correct everything and the beginner experiences every turn as a failure and quits. Correct nothing and errors fossilise — teachers' reluctance to correct directly is a documented fossilisation pathway ([Lyster & Saito feedback literature summary](https://e-journal.usd.ac.id/index.php/LLT/article/download/250/216)).

**The non-obvious part:** the SLA literature does *not* support the obvious "use gentle recasts" compromise. **Only ~30% of learners respond to recasts**, and meta-analysis finds large effect sizes for *prompts* (elicitation, clarification requests) which outperform recasts in within-group contrasts ([Lyster & Saito 2010, via the above](https://e-journal.usd.ac.id/index.php/LLT/article/download/250/216)). But prompts — "Can you try that again?" — are precisely the move that triggers freeze in a beginner. The research-optimal technique and the retention-optimal technique point in opposite directions at A0.

**How to avoid:**
- **Budget: at most one correction per turn**, selected by *recurrence* (appears in the error watchlist) not by severity. Silently log the rest.
- **Deliver it as an in-flow recast** ("Oh, you went to school yesterday! What did you do there?") so flow is never broken, accepting the ~30% uptake.
- **Recover the missing explicitness in a Vietnamese-language end-of-session recap**, not mid-conversation. Three bullets max: what you said, what's standard, why. Vietnamese L1 explanation is the cheapest high-value asset in the whole product and is routinely skipped because the dev is building an "English app".
- **Escalate only on recurrence:** 1st occurrence = silent log; 2nd = in-flow recast; 3rd = next session's scenario is *chosen* to elicit that structure with a scaffold frame that makes the correct form easy. This turns correction into content selection rather than interruption. It is also the SRS mechanism (see Pitfall 7), which is why they should be one system.
- **Never correct an utterance that came from STT without the learner having confirmed the transcript** (see Pitfall 5).

**Warning signs:** More than one correction marker rendered per turn. Learner's turn count per session dropping over time.

**Phase to address:** Phase 1 (one-per-turn budget + silent logging), Phase 2 (recap, escalation ladder).

---

### Pitfall 5: STT misrecognition reported as learner error (trust-destroying, likelihood ~certain)

**What goes wrong:** Learner says "I want to buy a ticket." Whisper hears "I want to buy a chicken." App says: *you made a mistake*. The learner knows they said ticket. Every subsequent correction is now suspect, including the true ones. This single failure converts the app from a coach into an adversary, and for a user with three prior abandonments it is terminal.

**Likelihood for THIS project:** Very high. Vietnamese-L1 MER **0.143** on *read speech by fluent L2 speakers* vs **0.007** for US English ([arXiv 2503.06924](https://arxiv.org/pdf/2503.06924)); beginner spontaneous speech will be materially worse. And pronunciation-assessment systems are no better: a representative mispronunciation-detection system reports a **false-reject rate of 22.9%** — roughly 1 in 4 *correctly* pronounced items flagged as wrong ([mispronunciation detection survey figures](https://signal.ejournal.org.cn/en/article/doi/10.16798/j.issn.1003-0530.2020.06.020)); even native speech rarely gets phoneme error rate below 15% ([arXiv 2209.06265](https://arxiv.org/pdf/2209.06265)).

**How to avoid (architectural for the first three, phase 1):**
1. **Decouple "produced an utterance" from "was it accurate."** The streak credit is granted the moment audio of sufficient duration is captured — *before* STT returns, and regardless of what it returns. Core Value must never depend on a recogniser. This has to be in the event model from the first schema.
2. **Always show the transcript with a one-tap "that's not what I said."** That button must (a) discard the utterance from error capture entirely, (b) offer a text-entry fallback, (c) be logged — if it's pressed >15% of the time, voice mode is not ready.
3. **Never feed an unconfirmed transcript into the error watchlist.** Require 2 independent occurrences before an error becomes a review item (see Pitfall 7) — this alone filters most STT noise.
4. **Bias the recogniser toward the expected answer.** Because the turn is scaffolded, you *know* the target utterance. Pass it as an STT prompt/hint (Whisper's `prompt` parameter, or provider keyword boosting) and score by normalised similarity to the target with a *generous* threshold, not by exact transcription. This is the single biggest accuracy win available and it is only possible because the architecture carries `expected_target` in the turn payload — another reason that field belongs in the protocol from day one.
5. **Asymmetric thresholds.** Be fast to say "yes, that worked" and extremely slow to say "that wasn't right." When similarity is ambiguous, say nothing about accuracy and move the conversation on. False praise is recoverable; false blame is not.
6. **No phoneme-level pronunciation scores. Ever, in this product.** The false-reject rate makes it a motivation weapon aimed at the one person it's built for.

**Warning signs:** "That's not what I said" tap rate >15%. Any corrections whose source is an unconfirmed transcript. Voice sessions shorter than text sessions.

**Phase to address:** Phase 1 (1–3 as protocol/event-model decisions), Phase 3 (4, when voice ships).

---

### Pitfall 6: Habit mechanics that backfire — especially when retention *is* the Core Value

**What goes wrong (the baseline):** Education apps have **Day-1 retention ~14–15% and Day-30 ~2–3%**, among the lowest of any category, versus ~25%/~6% for consumer apps generally; the steepest fall is D1→D7 ([Passion.io retention benchmarks](https://passion.io/blog/mobile-app-retention-benchmarks-for-creators-course-coaching-apps)). This app's bet is against a 2–3% base rate.

**Specific failure modes and what the evidence says:**

| Mechanic | Evidence | Verdict |
|---|---|---|
| Visible streak counter | Highlighting a streak increases repeat behaviour (Journal of Consumer Research) — **but** once broken, users are less likely to continue, and **highlighting the broken streak makes continuation even less likely** ([The Decision Lab, Streak Creep](https://thedecisionlab.com/insights/consumer-insights/streak-creep-the-perils-of-too-much-gamification)) | Use — with grace built in, and **never render a "streak lost" state** |
| Streak freeze / slack | UPenn & UCLA work shows goal "slack" outmotivates rigid rules; Duolingo's two-freeze change raised relative active learners **+0.38%** ([Duolingo research blog](https://blog.duolingo.com/duolingo-streak-research/)) | Use — and be *more* generous than Duolingo, since this user has no streak equity to protect |
| Daily push notification | 1 push/week → **10% disable notifications**; 3–6/week → **40% disable**; 25% of opt-outs cite "too many"; **80% will disable or uninstall** over notification volume ([Braze](https://www.braze.com/blog/opt-out-of-push-notifications-why-users-do-it/); [Mobile Marketing Magazine](https://mobilemarketingmagazine.com/?p=90128)) | **Max one per day, learner-chosen time, deep-linked to a ≤60s activity, and never a second nag.** A "you're about to lose your streak!" follow-up is the single worst message this app could send |
| Heavy gamification (points, levels, leaderboards) | Documented to sap intrinsic motivation — "Duolingo burnout", users optimising the streak over the learning ([Decision Lab, ibid.](https://thedecisionlab.com/insights/consumer-insights/streak-creep-the-perils-of-too-much-gamification)) | Out of scope already (no social). Keep it that way. Resist adding XP |
| The "too long session" trap | PROJECT.md already identifies this | Mitigate by making the *default* session short, not by offering a short option. The busy-day path should be the normal path |

**The danger unique to this project:** when retention *is* the Core Value, there is a constant pull to make the streak easier to satisfy until opening the app counts. That produces a 90-day streak and zero English. The opposite pull — making the streak "meaningful" by requiring a full session — breaks it on a 5-minute day.

**Resolution (architectural, phase 1):**
- The streak unit is **exactly one produced utterance.** Not opening the app. Not a full session. One. Server-decides, from a logged `utterance_produced` event.
- **Model practice as an append-only event log with timestamp + IANA timezone, and derive all counters.** Never store a mutable `current_streak` integer. Every forgiveness mechanic, timezone fix, retroactive repair, and "days practiced" view is then a query change instead of a migration. Getting this wrong in phase 1 is a schema rewrite later.
- **Display a non-decreasing primary number** — "days practiced: 47" — with the current run as a secondary. A number that can never go down cannot trigger streak-break abandonment.
- Automatic grace: a missed day is silently absorbed (up to ~2/week) with no "you used a freeze!" interstitial. Mention it only if asked.
- Track a second, private, non-gameable metric (utterances produced, distinct frames used, assessor pass rate) so the owner can distinguish habit from theatre.

**Warning signs:** Streak length rising while utterances-per-session falls. Any session with 0 utterances that still credited a day. Notification tap-through below ~10%.

**Phase to address:** Phase 1 (event model, one-utterance rule, non-decreasing display). Phase 4+ (notifications — and only after 30 days of self-use proves they're needed).

---

### Pitfall 7: SRS on conversational errors — worse than flashcard SRS in four specific ways

**Standard SRS failure modes:** review debt avalanche (every new card is a loan against future time; review load grows faster than card addition, and a two-week break produces a queue the user will not face), ease hell (SM-2 permanently depresses ease on repeatedly-failed cards so they dominate the queue), and leeches consuming disproportionate time ([Anki Burnout](https://www.neonlingo.com/blog/anki-burnout); [Chris Krycho on Anki](https://v5.chriskrycho.com/journal/anki-and-spaced-repetition/)).

**Why conversational errors make each one worse:**
1. **Item generation is automatic and unbounded.** A flashcard deck grows when *you* add a card. An error-capture system adds items every session without consent. Given a beginner, an error rate near 100% of utterances, and 10 turns/session, you manufacture review debt faster than any human deck.
2. **Items are poisoned by STT.** ~1 in 4 "errors" from voice mode may be recogniser artifacts (Pitfall 5). Flashcards don't have this failure.
3. **Errors are not atomic.** "I go school yesterday" is three overlapping issues (tense, preposition, article). Scheduling them as three items triples the debt and reviews the same sentence three times. Scheduling it as one item makes it un-gradeable.
4. **There is no good prompt/answer pair.** "What's the past tense of go?" is exactly the decontextualised drilling PROJECT.md rejects as the cause of the problem.

**What must be designed against (phase 1 decisions, simple implementation):**
- **No due dates. No debt. No "N cards due" number anywhere in the UI.** That number is the abandonment trigger. Instead: a **priority-ranked watchlist** that the next session *samples from*. Overdue items simply rank higher; they never accumulate as a visible obligation. A 10-day absence produces zero backlog.
- **Hard cap the active watchlist at ~15–20 items.** When full, a new error displaces the lowest-priority one. Capacity, not queue.
- **Require ≥2 independent occurrences** before an error becomes an item. Dedupes STT noise and transient slips in one rule.
- **Auto-retire aggressively.** 3 clean productions → retire. 5 failures → **retire anyway** and surface it in the Vietnamese recap as "this one needs a human/a lesson." Do not fight leeches; this is a personal app, not a medical-school deck.
- **Review happens inside conversation, never as a deck.** The watchlist's job is to *bias scenario and frame selection* so the target structure is elicited naturally. This is also Pitfall 4's escalation ladder — build them as one system, not two.
- **Do not implement SM-2, FSRS, or any ease-factor algorithm in v1.** They exist to optimise retention over thousands of items; you have twenty. A priority score of `recency × recurrence × failure_count` is sufficient and cannot enter ease hell because there is no ease factor.

**Warning signs:** Any screen showing a count of pending reviews. Watchlist size growing monotonically. Same error surfacing in more than 3 consecutive sessions.

**Phase to address:** Phase 2 (watchlist capture + capping). Explicitly **not** phase 1 — the error log can accumulate silently from phase 1 and be used later.

---

### Pitfall 8: Cost blowup on the shared key

**The concrete ways the owner gets a surprise bill, ranked by likelihood here:**

| Mechanism | Realistic magnitude | Guardrail |
|---|---|---|
| **Open tab in voice/realtime mode.** Session stays live, audio input bills continuously, nobody is talking | $0.46/min runaway observed; **$6 for 75 seconds** reported ([OpenAI community](https://community.openai.com/t/realtime-api-extremely-expensive/966825)) | Avoid realtime entirely in v1. If voice is staged + push-to-talk, an idle tab costs **$0**. This is the dominant argument for the staged pipeline |
| **Context accumulation.** Sending the full transcript each turn → tokens grow quadratically with turn count. A 30-turn session re-sending everything costs ~15× a windowed one | 10–20× on long sessions | Context windowing (Pitfall 2) — same mitigation, three benefits |
| **Retry storm.** A failing call retried inside a React effect or a client-side loop; one bad deploy and it runs all night | Unbounded. This is the classic solo-dev bill | Max 2 retries, exponential backoff, **retries debit the user's budget**, and a circuit breaker that hard-stops the route after N consecutive failures |
| **Key exfiltration → token jacking.** See Pitfall 10 | **$82,314 in two days** on a stolen Gemini key vs $180/month normal; **$600,000** at METR; one IR case near **$1M** ([hackmag](https://hackmag.com/news/gemini-api-key); [ITPro/METR](https://www.itpro.com/security/cyber-attacks/hackers-ran-up-a-usd600-000-ai-bill-after-swiping-api-keys-says-metr-and-nobody-realized-for-weeks); [Gridinsoft](https://blog.gridinsoft.com/ai-token-jacking-stolen-api-keys/)) | Pitfall 10 |
| **Unauthenticated `/api/chat`.** A public route with a server-side key is a free LLM proxy. These get found by scanners | Same order as above | Auth on every model-touching route, phase 1 |
| **Regeneration loop.** The vocabulary validator rejects, you regenerate, it rejects again, forever | 5–10× on affected turns | Cap at 2 regenerations, then fall back to a **hand-authored safe reply** from the scenario's static fallback set |

**Architectural guardrails (phase 1, all of them):**
- **One server-side model gateway module. Every model call goes through it.** No provider SDK calls anywhere else in the codebase. This is the choke point for budgeting, logging, caching, retries, and provider swaps, and scattering SDK calls across route handlers is the thing you cannot undo cheaply.
- **Check the per-user budget *before* the call, not after.** Store `tokens_used_today` / `cost_cents_today` per user; atomic decrement; refuse with a friendly "you've practised a lot today — come back tomorrow" message. Post-hoc accounting discovers overspend after the money is gone.
- **Global daily kill-switch** independent of per-user limits, for the retry-storm case.
- **Set a hard spend limit at the provider account level.** This is the only backstop that survives a bug in your own limiter.
- **Log cost per request from request #1.** You cannot diagnose a bill you never attributed.
- **Tiered models:** cheap model for scaffold generation and assessment, expensive model only where quality is visible. Prompt caching where available (cached input can be **~80× cheaper**).

**Target to design against:** ≤$5/user/month. If the design can't hit that, the design is wrong, not the budget.

**Phase to address:** Phase 1. All of it. Retrofitting a gateway after 20 call sites exist is a week you won't spend.

---

### Pitfall 9: Latency — and the counter-intuitive thing about beginners

**The numbers:** human turn-taking gaps average **~200ms**; **>500ms** reads as unnatural; **~800ms** from end-of-utterance to first audio is the usable budget for a natural-feeling voice agent; **>1000ms feels broken**. Industry median for production voice agents is **1.4–1.7s** ([twig.so 800ms rule](https://www.twig.so/blog/voice-ai-agents-latency-budget-800ms); [tianpan.co latency budget](https://tianpan.co/blog/2026-04-09-voice-ai-production-300ms-latency-budget); [aiweekly on p99 vs perception](https://aiweekly.co/node/1785)).

**Where the budget goes** ([Deepgram voice agent architecture](https://deepgram.com/learn/voice-agent-architecture-stt-llm-tts-pipeline-design); [futureagi sub-500ms guide](https://futureagi.com/blog/sub-500ms-voice-ai-guide-2026/)):
network 30–50ms · STT first partial 100–150ms · LLM TTFT 200–300ms with prefix caching · TTS first audio 80–150ms streaming · orchestration 50–100ms. With streaming, perceived gap ≈ `max(STT_partial, LLM_TTFT) + TTS_first_audio + orchestration`. Streaming STT saves 100–200ms; streaming TTS saves 200–400ms.

**The counter-intuitive part, and the real pitfall for this product:** those thresholds come from *fluent adults in real-time dialogue*. A beginner using scaffolded turn-taking does not need 200ms — a deliberate ~1s pause reads as the app thinking, and is fine if the UI says so. **The latency risk here is not slowness, it is turn-end detection.** Voice-agent VAD is tuned for native speakers, typically ending the turn after 500–800ms of silence. A "mất gốc" learner pauses 2–4 seconds mid-sentence while retrieving a word. **Auto-VAD will cut them off mid-utterance**, repeatedly, and that is an order of magnitude more damaging than a 1.5s reply delay — it is Pitfall 5's trust failure delivered by a different mechanism.

**Mitigations:**
- **Push-to-talk, not VAD.** Hold-to-speak, or tap-to-start / tap-to-stop. The learner decides when their turn ends. This also removes the open-tab cost risk and the barge-in complexity. For this user it is strictly better than realtime, not a compromise.
- **Budget: perceived response ≤1.5s, hard ceiling 5s with an explicit fallback** ("let me try that again") rather than an indefinite spinner.
- **Pre-generate and cache TTS for all scaffold frames.** They are static per scenario. Playback is then ~0ms and the learner's most frequent audio is instant.
- **Stream TTS from the first sentence** rather than waiting for the full LLM response.
- **Pre-fetch the likely next turn.** Because the beginner's reply is drawn from 2–4 known options, you can speculatively generate the response to the most likely one during the learner's thinking time. Constrained choice buys you latency for free — another dividend of Pitfall 1's design.
- **Show state honestly.** A visible "listening / thinking / speaking" indicator converts unexplained delay into expected delay. Perceived latency ≠ measured latency ([aiweekly](https://aiweekly.co/node/1785)).

**Phase to address:** Phase 3 (voice). Push-to-talk-vs-VAD is a product decision to make in phase 1 so the voice phase isn't scoped around realtime.

---

### Pitfall 10: Security and privacy for a shared-key, few-user app

**The realistic threat model — not OWASP boilerplate, the four things that actually happen to apps like this:**

1. **Key exfiltration (highest severity).** Truffle Security found **2,863 Google Cloud API keys embedded in client-side website code**, many of which silently granted Gemini access ([ThaiCERT summary](https://www.thaicert.or.th/?p=12238)). Stolen keys reach gray-market resale proxies within minutes. Consequences as above: $82k/2 days, $600k, ~$1M.
   - **Must not:** put the key in any `NEXT_PUBLIC_*` / `VITE_*` variable; commit `.env`; call the provider SDK from browser code; paste the key into a client-side "quick test".
   - **Must:** key server-side only; `.env*` in `.gitignore` before the first commit; a secret scanner in CI (gitleaks); provider-level spend cap; rotate on any suspicion.
2. **The open LLM proxy (most likely to actually happen).** For "just a handful of friends" it is tempting to skip auth. An unauthenticated route that calls a paid model is a free-inference endpoint; scanners find these. **Every model-touching route requires a session, from the first deploy.** Use a managed magic-link provider — a day of work, not a week. Do not roll your own.
3. **IDOR / missing per-user scoping.** PROJECT.md requires isolated data. The failure is a query like `getHistory(conversationId)` with no `WHERE user_id = ?`. **Scope by user at the data-access layer, not in route handlers** — one unscoped query is all it takes. If the stack supports row-level security, use it.
4. **Prompt injection — low severity here, be honest about it.** No tools, no secrets in context, single trusted user per session. The realistic outcome is the learner steering the partner off-curriculum. Mitigation: output validation already in place for level control (Pitfall 2) catches most of it. Do not spend phase-1 effort here.

**Privacy — voice recordings and conversation logs of a private individual:**
- **Beginner conversation content is personal.** A2 scenarios are "my family", "my job", "where I live". The transcript is a diary.
- **Must not:** persist raw audio beyond the transcription request; send audio or transcripts to a provider whose default terms allow training on it (verify the no-training / zero-retention setting explicitly and record the date you verified it); ship Sentry/analytics with default request-body capture on (it will log transcripts to a third party); use one shared database row-space without user scoping.
- **Must:** delete audio immediately after STT returns, store transcript only; a one-click "delete all my data" per user; and **tell the friends in plain Vietnamese, before they start, that their conversations are sent to a third-party AI provider and stored on the owner's server.** For a personal app among friends, informed consent is the whole compliance story — and the only one that matters if a friendship is on the line.
- Consider: text-only mode stores strictly less. For a user who may be self-conscious about their voice, that is also a feature.

**Phase to address:** Phase 1 (key handling, auth, data scoping, audio-retention policy as a written rule). Phase 3 (audio pipeline must honour the rule).

---

## Scope Traps for a Solo Developer

Honest sizing of the parts that look small.

| Looks like | Actually is | Realistic multiplier |
|---|---|---|
| **"Add a mic button"** | getUserMedia + HTTPS requirement, iOS Safari autoplay policy blocking TTS without a user gesture, AudioWorklet vs deprecated APIs, format conversion, mobile backgrounding killing streams, permission-denied recovery, push-to-talk UX on touch | **5–10×.** This is the classic 3-week sink |
| **"A handful of friends, no real auth needed"** | Needed anyway because the key is shared and routes are public — but it is a 1-day job *if* you use a managed magic-link provider and a 2-week job if you build it | 1× if bought, 10× if built |
| **"The AI generates the lessons"** | **It does not.** Scaffold frames need a deliberate progression: which structures, in what order, with which 800 words, across which scenarios. 10 scenarios × ~8 turns of hand-checked frames is the actual product and it is content work, not code | **This is the #1 under-estimated item in the project** |
| **"Withdraw scaffolding as they improve"** | A learner-state model: frames mastered, active errors, level, confidence per structure — plus the policy that reads it. A schema and a rules engine, not an `if` | 4× |
| **"Spaced repetition"** | Unbounded tuning with no stopping condition. Cut to the capped watchlist (Pitfall 7) | Cut it |
| **"Streak tracking"** | Timezones, what counts as a day, DST, travel, retroactive repair, grace accounting. Mundane and it eats a week and it is visible when wrong | 3×, and unavoidable |
| **"Prompt tuning"** | Without an eval harness — 20 fixed learner turns replayed through the prompt, asserting vocab coverage, sentence length, correction count, assessor honesty — every prompt change is blind, and regressions are invisible until the learner hits them. Built later means never built | Build a minimal one in phase 1; ~half a day, pays back in week two |

**The most likely way this project dies half-built — stated plainly:**

Week 1: the chat shell and the LLM call come together fast and feel magical, because the developer is testing it and the developer can hold a conversation in English. Weeks 2–4: voice mode consumes everything — Safari, permissions, VAD, latency. Week 5: the developer is tired, ships it to the learner. The learner opens an unscaffolded chat that has drifted to B1 by turn 6, freezes at the empty box, and quits on day three — the identical failure to Duolingo, arrived at by a more expensive route. The scaffold content, which was the entire thesis, was never authored.

**The second most likely death:** the streak and progress UI get polished because they are fun and visible, while correction, error capture, and review stay stubs. The learner gets a 30-day streak and no English, discovers this, and the trust loss is worse than quitting early.

**Prescription for the roadmap:** Phase 1 ships **text only, one scenario, fully scaffolded, no voice, no SRS, no notifications, no auth beyond a name** — and the owner uses it for **7 consecutive days** before any further feature is built. If the loop does not survive 7 days of the owner's own use in text, voice will not rescue it; it will only make the corpse more expensive. That 7-day gate should be an explicit roadmap checkpoint, not a hope.

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|---|---|---|---|
| Call the provider SDK directly in route handlers | Ships in minutes | No cost metering, no budget enforcement, no caching, no provider swap; refactoring 20 call sites later | **Never.** Gateway module on day one |
| Pass the full `messages[]` transcript each turn | Trivially correct conversation | Level drift (Pitfall 2) + quadratic cost + rising latency, all at once | **Never** past ~6 turns. Window from the start |
| One LLM call that chats *and* assesses | Half the cost and latency | Sycophantic assessment, poisoned error data, a learner who believes a lie for 3 months | **Never.** Split in phase 1 |
| Mutable `current_streak` integer column | Simple | Every grace/timezone/repair feature becomes a migration + backfill | **Never.** Append-only events, derive counters |
| No auth, "it's just friends" | Saves a day | Open LLM proxy, cross-user data leaks | Local-only dev. **Never on a public URL** |
| Hand-wave the scaffold content, let the LLM improvise frames | Feels like progress | The product has no curriculum and the thesis is untested | **Never for scenario #1.** Acceptable for scenarios 5–10 once the pattern is proven |
| Skip the eval harness | Half a day saved | Every prompt change is blind; silent regressions | Only until the first prompt regression, which arrives in week 2 |
| Store raw audio "for debugging" | Easier to diagnose STT issues | Privacy exposure, storage growth, a promise to friends broken | Local dev with your own voice only, behind an explicit flag, never deployed |
| Hard-code the one scenario's content in a component | Fast | Scenario #2 forces the data model you should have had | Acceptable **once**, with the schema sketched before scenario #2 |

---

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|---|---|---|
| LLM provider | Trusting the system prompt to hold level over a long conversation | Window the context, validate output against a vocabulary list, regenerate on violation; see Pitfall 2 |
| LLM provider | Streaming the partner's reply straight to the UI with no validation gate | Validate before display at beginner level; accept the latency cost, or stream only after the first validated sentence |
| STT (Whisper/Deepgram/AssemblyAI) | Treating the transcript as ground truth | Pass `expected_target` as a decoding hint, score by similarity with a generous threshold, always show the transcript with a "that's not what I said" escape |
| STT | Auto-VAD endpointing at 500–800ms of silence | Push-to-talk. Beginners pause 2–4s mid-sentence |
| TTS | Generating scaffold-frame audio on every request | Pre-generate and cache per scenario; frames are static |
| TTS on iOS Safari | Playing audio without a prior user gesture — silently blocked | Unlock the audio context on the first tap of the session; test on a real iPhone, not the simulator |
| Realtime voice API | Adopting it for "natural conversation" | 20–50× the cost of the staged pipeline, no transcript control, no scaffold insertion point. Not for v1 |
| Auth provider | Rolling your own because "it's only 5 users" | Managed magic link; a day of work |
| Error tracking (Sentry et al.) | Default request-body capture shipping transcripts to a third party | Scrub request bodies and transcript fields before enabling, or don't enable it |
| Hosting (Vercel/Fly/Render) | Key in env vars *and* a stray committed `.env.local` | Secret scanner in CI; `.gitignore` before commit #1; provider-level spend cap as the backstop |

---

## Performance Traps

Scale here is 5 users, so most classic traps are irrelevant. These are the ones that bite anyway.

| Trap | Symptoms | Prevention | When It Breaks |
|---|---|---|---|
| Unwindowed context | Later turns slower and pricier than earlier ones; level drift | Last 3–4 turns + state summary | Turn ~8 of the **first** session |
| Synchronous validate-then-regenerate on every turn | Occasional 2–3s stalls with no explanation | Cap regenerations at 2, fall back to a hand-authored safe reply, show a "thinking" state | ~10–25% of turns, immediately |
| Full-history load on session start | Slow open; worsens monthly | Paginate; load the last session only | ~3 months of daily use |
| No DB index on `(user_id, created_at)` | Fine at 5 users, invisible until it isn't | Add it when the table is created | Never at this scale — add it anyway, it's one line |
| TTS generated per request | Latency spike on every scaffold render + recurring cost | Cache per frame | Day one |
| Error watchlist unbounded | Review selection slows; the list becomes meaningless | Hard cap ~20 with displacement | Week 2 of real use |

---

## UX Pitfalls

| Pitfall | User Impact | Better Approach |
|---|---|---|
| Empty input box at beginner level | Freeze → abandonment by day 3 (the project's named central risk) | Scaffold *is* the input: tappable options + one-slot frames |
| Corrections rendered as red error markers | Every turn feels like failure | One in-flow recast per turn; explicit teaching deferred to a Vietnamese recap |
| English-only UI chrome and explanations | A "mất gốc" learner can't read the app that's teaching them English | **Vietnamese UI and Vietnamese explanations; English only in the target content.** Cheap, high-leverage, routinely skipped |
| "Streak lost 🔥💔" screen | Evidence says highlighting the break *reduces* continuation | Never render it. Show cumulative days practiced |
| "You have 23 reviews due" | Direct abandonment trigger | No counts. Sample from a priority watchlist |
| Spinner with no state | Perceived latency ≫ real latency | Explicit listening / thinking / speaking states |
| Session that "ends" at a fixed length | Breaks on a 5-minute day; the named abandonment trigger | Every turn is a complete, credited unit. The learner leaves whenever; it already counted |
| Pronunciation score | ~1-in-4 false rejections aimed at a fragile learner | Don't ship it |
| Voice as the default mode | Intimidating at A0, and where STT error is worst | Text default; voice offered as listen-and-repeat |
| Onboarding that asks for level/goals | D1 drop-off is the steepest; every pre-practice screen costs users | Zero-question entry. First screen is turn one of a conversation. Infer level from behaviour |

---

## "Looks Done But Isn't" Checklist

- [ ] **Scaffolded conversation:** often missing — the scaffold is a *hint panel* beside a text box rather than the input mechanism. Verify a user who types nothing can still complete a full session by tapping only.
- [ ] **Level control:** often missing — validated only on turns 1–3. Verify by logging OOV-word count and sentence length at **turn 10+** of a real session.
- [ ] **Correction:** often missing — no per-turn budget, so a bad utterance produces 4 corrections. Verify max one correction marker per turn on an intentionally broken input.
- [ ] **Assessment honesty:** often missing — the assessor shares the partner's encouraging prompt. Verify it returns a non-empty `errors` array for the fixture "I yesterday go school very much".
- [ ] **Voice mode:** often missing — no "that's not what I said" escape, and streak credit depends on STT succeeding. Verify a deliberately garbled utterance still credits the day.
- [ ] **Streak:** often missing — timezone handling, and a "streak lost" state nobody meant to build. Verify by changing the device timezone and by simulating a 3-day gap.
- [ ] **Busy-day path:** often missing — it exists but is 3 taps deep. Verify it is reachable in **one** tap from the entry screen and completes in under 60 seconds.
- [ ] **Per-user isolation:** often missing — one unscoped query. Verify by logging in as user B and attempting to fetch user A's conversation id directly.
- [ ] **Cost controls:** often missing — budget checked *after* the call. Verify a user at their limit is refused *before* any provider request is made, and that the provider account itself has a hard spend cap.
- [ ] **Key hygiene:** often missing — `.env` committed in the very first commit, before `.gitignore`. Verify with `git log -p` and a secret scanner.
- [ ] **Audio retention:** often missing — the "delete immediately" rule is written down but a temp file survives. Verify no audio exists on disk or in object storage after a voice turn.
- [ ] **Error watchlist:** often missing — an unbounded table with no cap and no retirement. Verify the cap holds after 50 simulated errors.

---

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---|---|---|
| Level drift discovered in production | **LOW** | Add windowing + validator at the gateway; no schema change if the gateway exists |
| Scattered SDK calls, no gateway | **HIGH** | Touch every call site; all cost history is unrecoverable. Prevent instead |
| Sycophantic assessor already shipped | **HIGH** | Split the calls *and* re-derive the error watchlist, whose stored data is now untrustworthy |
| Mutable streak integer | **MEDIUM-HIGH** | Migration + backfill from whatever logs exist; the pre-migration history is approximate forever |
| STT falsely blamed the learner | **VERY HIGH — may be unrecoverable** | Apologise in Vietnamese, disable accuracy feedback in voice mode entirely, rebuild trust over weeks. This is why the asymmetric threshold is non-negotiable |
| Learner quit on day 3 | **HIGH** | Re-engaging an abandoner is harder than the first acquisition, and this learner has a four-for-four abandonment record |
| Key leaked | **MEDIUM** financially *if caught fast*, catastrophic if not | Rotate immediately; provider spend cap limits the blast radius; audit usage. The cap is the whole defence |
| Review debt accumulated | **LOW** if there are no due dates | With a priority watchlist there is nothing to recover from — which is the point of choosing it |
| Voice mode consumed 3 weeks | **MEDIUM** | Ship text-only, use it daily, return to voice later. Prevented by the phase-1 gate |

---

## Pitfall-to-Phase Mapping

### Architectural — must be in Phase 1, cannot be bolted on

| Decision | Pitfall(s) | Why it can't wait | Verification |
|---|---|---|---|
| Turn protocol carries `scaffold_options`, `frame`, `expected_target` | 1, 5, 9 | Changing the turn contract later invalidates the UI, the STT biasing, and every stored turn | A tap-only session completes end to end |
| Single server-side model gateway (budget check *before* call, cost logging, retry cap, caching) | 8, 10 | 20 scattered call sites is a week of refactor and zero recoverable cost history | A user at budget is refused with no provider request issued |
| Partner call ≠ assessor call; assessor returns strict JSON with no persona | 3, 4 | Shared prompts poison the stored error data, not just the UX | Assessor flags the known-broken fixture |
| Context windowing (last 3–4 turns + state summary) | 2, 8, 9 | Fixes drift, cost, and latency with one change; retrofitting touches every call | OOV count and sentence length flat from turn 1 to turn 12 |
| Vocabulary allowlist + deterministic output validator + capped regeneration | 2 | The generate→validate→regenerate loop is a pipeline shape | 0 out-of-list content words across a 15-turn transcript |
| Append-only practice-event log with timestamp + timezone; all counters derived | 6 | Every grace/timezone/repair feature is otherwise a migration | Timezone change and a 3-day gap both behave correctly |
| Streak credit = one produced utterance, granted server-side before STT returns | 5, 6 | Core Value must not depend on a recogniser | Garbled voice input still credits the day |
| Per-user scoping enforced at the data-access layer | 10 | One unscoped query is a cross-user leak | User B cannot fetch user A's conversation by id |
| Audio never persisted; transcript-only storage, written as a rule now | 10 | Honours the promise to friends; prevents the "temp file" leak | No audio artifacts after a voice turn |
| Auth on every model-touching route before first deploy | 8, 10 | An open route is a free LLM proxy | Unauthenticated request to `/api/*` returns 401 |
| Minimal eval harness (20 fixed learner turns, assertions on level/correction/honesty) | 2, 3, 4 | Built later means never built | Harness runs in CI and fails on a deliberately bad prompt |

### Later refinement — safe to defer

| Item | Phase | Note |
|---|---|---|
| Error watchlist capture + cap + retirement | 2 | Errors can be logged silently from phase 1 and used later |
| Vietnamese end-of-session recap | 2 | High value, low risk, no architectural dependency |
| Correction escalation ladder (silent → recast → scenario selection) | 2 | Needs the watchlist first |
| Voice mode (push-to-talk, staged pipeline, STT biasing, confirm-transcript) | 3 | Gate behind 7 days of real text-mode use |
| TTS caching and latency tuning | 3 | Pure optimisation |
| Scenarios 2–10 | 3–4 | Content work; the model is proven on scenario 1 |
| Automatic scaffold withdrawal | 4 | Start with a manual level setting |
| Notifications | 4+ | Only after daily use is already happening. One per day, max |
| Deployment, multi-user auth hardening | 2–3 | Local-first per PROJECT.md, but the gateway/auth/scoping shape must already be right |
| Streak grace mechanics UI | 2 | The *data model* is phase 1; the UI is not |

### Explicitly recommended OUT of scope

| Item | Reason |
|---|---|
| Realtime speech-to-speech API | 20–50× cost, no transcript control, no scaffold insertion point, open-tab billing risk |
| Phoneme-level pronunciation scoring | ~22.9% false-reject rate aimed at a learner with a four-for-four abandonment record |
| Auto-VAD turn endpointing | Cuts off a hesitating beginner mid-sentence; push-to-talk is strictly better here |
| SM-2 / FSRS / any ease-factor SRS | Ease hell and review debt for a 20-item watchlist you can schedule with a 3-term priority score |
| A visible "reviews due" count | Direct abandonment trigger |
| XP, levels, badges, leaderboards | Intrinsic-motivation erosion ("streak creep"); social already out of scope |
| Free-text input at beginner level | The named central risk of the whole project |

---

## Sources

**Speech recognition on non-native speech**
- [Automatic Speech Recognition for Non-Native English: Accuracy and Disfluency Handling (arXiv 2503.06924)](https://arxiv.org/pdf/2503.06924) — L2-ARCTIC, per-L1 error rates; Vietnamese MER 0.143 (male 0.181) vs US English 0.007 — HIGH
- [Evaluating OpenAI's Whisper ASR: Performance Across Diverse Accents and Speaker Traits (Cambridge Engage)](https://www.cambridge.org/engage/coe/article-details/65490ca0a8b423585a102952) — accent/L1/L2-proficiency effects on WER — HIGH
- [Automated detection of pronunciation errors in non-native English speech employing deep learning (arXiv 2209.06265)](https://arxiv.org/pdf/2209.06265) — native-speech phoneme error rate floor ~15% — HIGH
- [Mispronunciation detection with extended recognition network constraints](https://signal.ejournal.org.cn/en/article/doi/10.16798/j.issn.1003-0530.2020.06.020) — false accept 29.2%, false reject 22.9% — MEDIUM

**LLM proficiency-level control**
- [Alignment Drift in CEFR-prompted LLMs for Interactive Spanish Tutoring (arXiv 2505.08351)](https://arxiv.org/html/2505.08351v2) — level distinctions collapse by turns 8–9; prompting insufficient — HIGH
- [Toward Beginner-Friendly LLMs for Language Learning (arXiv 2506.04072)](https://arxiv.org/html/2506.04072v2) — beginner comprehensibility 39.4% (prompting) → 83.3% (controlled generation) — HIGH
- [From Tarzan to Tolkien: Controlling Language Proficiency Level of LLMs (arXiv 2406.03030)](https://arxiv.org/html/2406.03030v1) — CaLM, fine-tuned control beating GPT-4 at lower cost — MEDIUM

**Sycophancy / feedback honesty**
- [Understanding Sycophancy in LLMs (arXiv 2602.01002 overview)](https://www.alphaxiv.org/overview/2602.01002) — models agree with wrong answers 24%–60%; RLHF false positives 46.7% → 70.2% — HIGH
- [Sycophancy in Large Language Models: Causes and Mitigations (arXiv 2411.15287)](https://arxiv.org/pdf/2411.15287) — mechanism — HIGH

**Corrective feedback / SLA**
- [Corrective feedback review incl. Lyster & Saito meta-analysis figures](https://e-journal.usd.ac.id/index.php/LLT/article/download/250/216) — recast uptake ~30%, prompts > recasts, fossilisation from non-correction — MEDIUM
- [The Effectiveness of Explicit Corrective Feedback in the L2 Classroom](https://pops.lancashire.ac.uk/index.php/jsltr/article/view/360) — small effect for explicit feedback, long-term unconfirmed — MEDIUM
- [CALICO Research Brief 8: chatbots in language learning](https://calico.org/wp-content/uploads/2023/06/Research-Brief-8-Summer-2023.pdf) — limitations with lower-proficiency learners — MEDIUM

**Retention and habit mechanics**
- [Mobile App Retention Benchmarks (Passion.io)](https://passion.io/blog/mobile-app-retention-benchmarks-for-creators-course-coaching-apps) — Education D1 ~14–15%, D30 ~2–3%; steepest drop D1→D7 — MEDIUM (industry data, methodology varies)
- [Streak Creep: The perils of too much gamification (The Decision Lab)](https://thedecisionlab.com/insights/consumer-insights/streak-creep-the-perils-of-too-much-gamification) — broken-streak abandonment, harm of highlighting the break, intrinsic-motivation erosion — MEDIUM
- [Duolingo streak research](https://blog.duolingo.com/duolingo-streak-research/) and [how streaks keep learners committed](https://making.duolingo.com/how-streaks-keep-duolingo-learners-committed-to-their-language-goals) — streak freeze +0.38% active learners; "slack" research — MEDIUM (vendor-published)
- [Braze: What makes users opt out of push notifications](https://www.braze.com/blog/opt-out-of-push-notifications-why-users-do-it/) — 25% opt out from volume; 1/wk → 10% disable, 3–6/wk → 40% — MEDIUM
- [80% of users will disable an app due to push notifications](https://mobilemarketingmagazine.com/?p=90128) — MEDIUM

**Spaced repetition failure modes**
- [Anki Burnout: Why Your Review Pile Keeps Growing](https://www.neonlingo.com/blog/anki-burnout) — review debt mechanics, ease hell, recovery patterns — MEDIUM (practitioner)
- [Anki and Spaced Repetition (Chris Krycho)](https://v5.chriskrycho.com/journal/anki-and-spaced-repetition/) — leeches, sustainability — MEDIUM (practitioner)

**Cost**
- [OpenAI Realtime API pricing: the API is cheap, your voice-agent bill won't be (Forasoft)](https://www.forasoft.com/article/openai-realtime-api-pricing) — $0.06–0.11/min flagship, $0.02–0.05 mini, $0.46/min runaway, caching ~80× — MEDIUM-HIGH
- [OpenAI community: Realtime API extremely expensive](https://community.openai.com/t/realtime-api-extremely-expensive/966825) and [VAD and token accumulation](https://community.openai.com/t/realtime-api-pricing-vad-and-token-accumulation-a-killer/979545) — ~$6 for 75s — MEDIUM (practitioner reports)

**Latency**
- [Latency Budgets for Voice AI Agents: The 800ms Rule](https://www.twig.so/blog/voice-ai-agents-latency-budget-800ms) — 200ms human gap, 800ms budget — MEDIUM-HIGH
- [Voice AI in Production: Engineering the 300ms Latency Budget](https://tianpan.co/blog/2026-04-09-voice-ai-production-300ms-latency-budget) — per-stage breakdown — MEDIUM
- [Voice Agent Architecture: STT→LLM→TTS Pipeline Design (Deepgram)](https://deepgram.com/learn/voice-agent-architecture-stt-llm-tts-pipeline-design) — streaming savings — MEDIUM-HIGH
- [Sub-500ms Voice AI Guide](https://futureagi.com/blog/sub-500ms-voice-ai-guide-2026/) — stage budgets — MEDIUM
- [Voice agent p99 metrics miss actual user perception](https://aiweekly.co/node/1785) — 1.4–1.7s production median; perceived ≠ measured — MEDIUM

**Security**
- [AI Token Jacking: Stolen API Keys Fueled Nearly $1M Bills (Gridinsoft)](https://blog.gridinsoft.com/ai-token-jacking-stolen-api-keys/) — HIGH
- [Hackers ran up a $600,000 AI bill after swiping API keys — METR (ITPro)](https://www.itpro.com/security/cyber-attacks/hackers-ran-up-a-usd600-000-ai-bill-after-swiping-api-keys-says-metr-and-nobody-realized-for-weeks) — HIGH
- [Stolen API Key Costs Developer $82,000 in Gemini Usage (HackMag)](https://hackmag.com/news/gemini-api-key) — $82,314 in 2 days vs $180/mo baseline — MEDIUM-HIGH
- [Thousands of Publicly Exposed Google Cloud API Keys (ThaiCERT / Truffle Security)](https://www.thaicert.or.th/?p=12238) — 2,863 keys in client-side code — MEDIUM-HIGH

---
*Pitfalls research for: AI-scaffolded conversational English learning for an absolute-beginner Vietnamese L1 learner, solo-built, shared-key*
*Researched: 2026-10-06*
