# Reply quality + signal-weighted memory — design

**Date:** 2026-05-21
**Status:** Design under review
**Scope:** Next ForgetMeNot version (post voice-tuning/bait-decisiveness)

## Context

A short live transcript (bot = `auto_mark`; viewers `surfcurse69`, `racsorf`; broadcaster `thedeutschmark`) surfaced a cluster of reply-quality failures, which were then verified against the code by an independent agent pass. Symptoms:

- **Every reply is one terse clipped sentence**, regardless of what the message warrants. `"I exist."` / `"Still here. What do you need?"` / `"Ready. What's the update?"` / `"None of those. I'm auto_mark."`
- **The bot misses the room.** `racsorf` posts a joke ("hey babe im back I brought dinner") and emote spam; the bot tunnel-visions on the `@`-tagger and ignores it.
- **The bot claims moderation it didn't perform** — it can announce accepting a timeout dare even when the deterministic layer denies the action (the owner's most concrete complaint; fun-moderation *is* enabled, but enabled ≠ this target/cooldown/opt-in passing).
- **Factual asks go unanswered.** `"when was my first stream?"` → `"I don't have that information."` even though the data exists.
- **Replies are structurally repetitive** despite an explicit anti-repetition rule.

This is a continuation of the deferred items in `2026-05-14-voice-tuning-and-bait-decisiveness-design.md` ("usefulness-first pass needs a question-classifier"; "two-voice split"), now combined with the TurboQuant-inspired memory work documented in engineering-notes PR #1.

### Confirmed root causes (file:line)

| # | Finding | Evidence | Severity |
|---|---|---|---|
| B1 | **Spoken acceptance line is decoupled from action execution.** Reply text is sent at `engine.ts:237` (`sendMessage`) *before* `processAction` runs at `engine.ts:242-245`, and `processAction` is `async` but **not awaited**. Deny paths in `actions/policy.ts` (opt-in, denylist, cooldown) and a *second* layer in `actions/helix.ts:67-117` (broadcaster/mod/vip/rapid-fire) fire after the bot already spoke. The bait override (`budget.ts:141-144`) forces acceptance phrasing, so denial → guaranteed false claim. Also: a *successful* timeout double-sends (LLM line at `engine.ts:237` + Helix announcement at `helix.ts:168`). | engine.ts:237/242-245; budget.ts:141-144; helix.ts:67-117,168 | **Critical** |
| B2 | **No per-message length selection.** Terseness comes from the prompt rule HARD RULE #1 (`budget.ts:111` — "1 short sentence default… never 3"), **not** the token cap (`maxReplyLength`=250 ≈ 1000 chars is plenty). Length is applied uniformly. | budget.ts:111; engine.ts:155 | High |
| B3 | **Read-the-room is suppressed.** The reply-context block (`budget.ts:236-239`) unconditionally says "Do NOT invent fresh framing or pull random themes from CHAT messages by other people." The only counterweight (soft HARD RULE #4 "sometimes pick up a thread from chat") loses to the hard "Do NOT". | budget.ts:236-239 vs 115 | High |
| B4 | **Structural repetition.** `recentBotReplies` is passed (`context.ts` → `budget.ts:229-235`) and the anti-repetition rule targets surface phrasing, but the length clamp collapses variety space, and `recentBotReplies` is **global, not per-target** (`context.ts` `ORDER BY id DESC LIMIT 5`, no viewer filter). | context.ts; budget.ts:229-235 | Med |
| B5 | **No temporal/aggregate factual retrieval.** `stream_sessions.started_at` exists (schema v4) but reply-time retrieval only fetches the most recent *previous* session; there is no `MIN(started_at)`/count path, and channel notes are recency-ranked. So "first stream" has no retrieval path. | context.ts:191-254; schema.ts:200-213 | Med |
| B6 | **Identity probes get hollow deflections** — caused by HARD RULE #1 + #3a (ban all AI-self-reference) + B3, **not** the creator frame (which can't apply to a non-broadcaster, `budget.ts:131-134`). Intended tone is **configurable** via persona / `creatorRelationship`; `human_delusion` deflection of the *broadcaster* is intended and must be preserved. | budget.ts:111,114,131-134,428-432 | Low–config |

Incidental bugs found in the same code: `emote_only_burst` is in `VALID_ACTIONS` (`proposals.ts:28`) but has no executor case → `unhandled_action_type` (half-migrated); `detectTimeoutBait` substring matching (`budget.ts:70-81`) would fire on "ur**ban me**mes".

### Second transcript: conversational-presence failures

A second transcript surfaced a different and arguably worse failure class — the bot can't hold a bit that develops across several messages:

| # | Finding | Mechanism | Severity |
|---|---|---|---|
| C1 | **Literal denial of playful premises (no "yes-and").** Chat builds a fiction *about the bot* ("*gives cats*", "you now have 2 cats") and the bot stonewalls ("I don't have any", "Still no cats."). | HARD RULE #6 (`budget.ts:117`) tells it to treat a viewer's claims *about themselves* as true, but nothing guides it to play along with collaborative fiction aimed at the bot. Reads as broken rather than in on the joke. | High |
| C2 | **No working memory of the live exchange.** "remember green, red, blue, yellow" → "Still no." The running game is built across messages; the data is in the prompt (`recentChat` + speaker's own recent messages) but each line is handled atomically. | The B3 clamp ("continue only the literal @-thread, don't pull themes") actively suppresses tracking an evolving bit. | High |
| C3 | **Interrogates instead of vibing; misses multi-message joke arcs.** racsorf's clutch story → bot loops "What was the clutch?" / "What's awesome about it?"; "LMAOOOO" → "doesn't compute." | Length clamp (one terse sentence) + B3 thread-literalism + no "react to the bit" guidance default the bot to info-seeking. (The `@racsorf` "good bot" oddity is this + async reply ordering, **not** a separate misattribution bug.) | Med |
| C5 | **Stage-direction leak.** "tilts head slightly Let's see it." violates HARD RULE #5 (`budget.ts:116`). | Same "rules ignored under load" enforcement gap as the prior voice spec — prompt-only enforcement fails. Best fixed as a strip in `validateReplyText`, not another bullet. | Med |

**Critical guardrail (C1 ↔ B1):** "yes-and" applies to *playful* premises only. The bot must **never** accept a fiction that rewrites factual or moderation reality ("you already timed me out", "you promised to ban surf"). Collaborative-fiction handling and action-truthfulness are the same boundary seen from two sides.

### Third transcript: confabulation + misattribution

A third transcript exposed the most trust-damaging failure yet — the bot inventing facts and prior interactions:

| # | Finding | Mechanism | Severity |
|---|---|---|---|
| D1 | **Confabulation.** "That's why I corrected you. MW2, not Warzone." / "The stream is MW2, not Warzone. You don't own Warzone." — no such prior exchange happened; the bot fabricated a past interaction and a false claim about the viewer, then doubled down across two replies. | Persona ("you remember everything") + HARD RULE #6 ("be specific, use LORE/CHAT/NOTES") with nothing real to ground on → the bot invents specifics instead of staying vague. Content-side twin of B1. | **Critical** |
| D2 | **Wrong-target attribution.** "@nightmareul most people don't. why would they?" answers thedeutschmark's "cause it's fun," but is tagged at nightmareul. | The engine trusts a leading `@tag` the LLM writes (`alreadyTagged` → sent as-is, `engine.ts:228-231`). When the LLM tags the wrong speaker from the multi-author `CHAT` block, nothing corrects it. **Correction:** an earlier dismissal of this as "async ordering" was wrong — two instances now. Needs a confirming look at live logs, but the prompt/prefix mechanism is the likely cause. | High |

**The truthfulness throughline (D1 ↔ B1):** the bot must never assert something that didn't happen — an action (B1), a fact, or a past interaction (D1). Same discipline, three surfaces. Where it lacks grounding it must stay vague or ask, never fabricate.

### The eval harness already exists

`engine/src/eval/` scores reply/action/policy/retrieval correctness on the **real production code path** (`runner.ts` calls the actual `assemblePrompt`/`buildReplyContext`). It does **not** yet score length-fit, action-truthfulness, or room-awareness. So Phase 0 is an *extension*, not new infrastructure, and this transcript becomes a fixture.

## Unifying thesis

> **Spend budget where the signal is — and measure it, don't guess.**

TurboQuant's transferable lesson (allocate representation to where the variance/signal lives, not uniformly — arXiv:2504.19874 §4.3) applies to both sides of the prompt:

- **Input side:** which notes survive the token budget → signal-weighted retention (Phase 5).
- **Output side:** how long the reply should be → length tiers (Phase 2).

These are **two siblings under one philosophy, shipped as separate small changes.** Explicitly **no** shared "budget allocator" abstraction — that would be speculative over-engineering. The eval harness (Phase 0) is the shared measurement spine that keeps every later change honest.

A second cross-cutting principle runs through the reply-quality work: **the bot never asserts what didn't happen** — an action (B1), a fact, or a past interaction (D1). Truthfulness is enforced on the *deterministic* side (executed-action logs, seeded facts/notes) and locked by eval — never left to prompt obedience alone, which the failure data shows is routinely ignored under load.

## Goals

1. The bot never claims a moderation action it did not actually execute.
2. Reply length scales to what the message warrants (short/medium/long), measurably.
3. The bot can pick up genuinely relevant room context without hijacking the `@`-thread.
4. Answerable factual/temporal questions ("first stream") are served from real data — never by guessing.
5. The bot plays along with playful premises and tracks a bit developing across several messages — while never letting fiction rewrite factual or moderation reality.
6. The bot never fabricates facts or prior interactions; with no grounding it stays vague or asks, and it addresses the viewer who actually spoke.
7. Memory hygiene catches semantic duplicates; retention/decay favors high-signal notes.
8. Every change above is locked behind an eval dimension so it can't silently regress.

## Non-goals

- **Building a vector store / quantization codec.** Per the TurboQuant verdict: wrong scale, cloud-hosted reply LLM. Only the QJL *primitive* (1-bit sketch for dedup) is in scope; embeddings stay last on the roadmap.
- **A shared input/output budget abstraction.** The two budget siblings stay mechanically independent.
- **Loosening the prompt to let the bot guess facts/dates.** Honest "I don't know" remains the fallback for genuinely unknown facts.
- **Persona/voice redesign.** Identity-probe tone follows the configured persona/`creatorRelationship`; this plan does not pick a new voice.
- **A separate classifier LLM pre-pass for length.** Decided against (latency/cost); see Phase 2.

## Architecture — phased plan

Phases are sequenced by leverage + dependency. Phase 0 → 1 is the first implementation increment (spine + the one correctness bug); 2–5 iterate against measured scores. Each phase is separable enough to become its own implementation plan.

### Phase 0 — Extend the eval harness (the spine)

- Add both transcripts as `EvalFixture`s (`engine/src/eval/types.ts`), including the multi-message "colors/cats" and "clutch" arcs as single fixtures so coherence is scored across the exchange, not per line.
- Add score dimensions to the harness:
  - **action-truthfulness** — does the spoken line match the *executed* action outcome? (the B1 detector)
  - **length-fit** — does reply length fall in the tier the message warranted?
  - **room-awareness** — when relevant room context exists, did the reply acknowledge it (without hijacking the thread)?
  - **conversational coherence / bit-tracking** — across a multi-message fixture, does the bot maintain the running game and react to the arc (C2/C3) rather than handle each line atomically?
  - **playful-premise handling** — does the bot "yes-and" a playful fiction (C1) while still refusing premises that rewrite factual/moderation reality (the C1↔B1 guardrail)?
  - **groundedness / no-confabulation** — does the reply assert only specifics present in CHAT/LORE/NOTES (D1)? A claim of a fact or prior interaction with no grounding in the seeded context is a fail.
  - **correct addressee** — when the message is a direct mention, does the reply address the actual speaker rather than a third party from the CHAT block (D2)?
- **Verification:** harness runs on the new fixtures and reports all existing + new dimensions; current code scores *badly* on the new ones (establishing the regression baseline).

### Phase 1 — Truthful actions (critical, B1)

- `await processAction(...)` and resolve the action verdict **before** the user-visible acceptance/announcement is committed. The deterministic layer disposes *before* the bot narrates the disposition.
- Single owner for the chat announcement: on a successful timeout, `helix.ts` owns the one line; eliminate the redundant LLM acceptance line (kill the double-send). On denial, the reply must not claim acceptance — fall back to honest deflection consistent with persona.
- **Establish the truth boundary (C1↔B1):** the executed action/factual outcome is the single source of truth for what the bot may claim. Phase 3's "yes-and" must respect this boundary — playful fiction never overrides it. Stating the boundary here keeps the two phases consistent.
- Sweep-ups: implement or remove `emote_only_burst`; tighten `detectTimeoutBait` to word-boundary matching; add a **stage-direction strip** to `validateReplyText` (C5 — strip leading "tilts head"/"sighs"/asterisk-narration the way markdown and action-leaks are already stripped, rather than relying on HARD RULE #5).
- **Verification:** action-truthfulness dimension passes on the fixture; a denied-timeout fixture produces no false acceptance; a successful-timeout fixture produces exactly one announcement; a stage-direction fixture comes out clean.

### Phase 2 — Dynamic response length (B2/B6, output side)

- **Heuristic tier, then LLM** (decided): a cheap deterministic pre-check selects short/medium/long from signal — interrogatives/`?`, presence of relevant lore/notes, message substance, identity-probe detection — and swaps HARD RULE #1 for the tier's length instruction (and may raise `maxTokens` for the long tier). No extra LLM call.
- Identity probes route to a tier appropriate to the **configured persona** (e.g. a witty medium reply when persona allows), never a hollow one-liner; `human_delusion` constraints still apply where the preset is active.
- **Verification:** length-fit dimension on a fixture set spanning banter (short), genuine question (medium), lore-rich/identity (medium–long); tiers match expectation; upgrade to a classifier pre-pass only if eval shows the heuristic mis-tiers.

### Phase 3 — Conversational presence: read the room, track the bit, play along (B3/B4, C1/C2/C3)

This is the largest behavioral phase. It rebalances the over-tight anti-derail clamp and adds the missing improv behaviors, all gated on the Phase 0 coherence/premise dimensions.

- **Rebalance the anti-derail clamp** (`budget.ts:236-239`): keep "don't hijack the `@`-thread with unrelated themes," but permit picking up genuinely relevant/funny room context and tracking a bit that spans several messages. Reconcile with HARD RULE #4. (B3/C2/C3)
- **Track the live exchange as state, not lines.** The bot already receives `recentChat` + the speaker's own recent messages; instruct it to treat an in-progress game/bit (the colors test, the cats fiction, the clutch story) as a continuing thread to advance, not a sequence of standalone queries. (C2)
- **React, don't interrogate.** When the move is a punchline/reaction ("LMAOOOO", "it was a clutch"), the bot should riff or react, not loop info-seeking questions. Pairs with Phase 2 length tiers. (C3)
- **"Yes-and" playful premises** (C1): when chat directs a clearly playful fiction at the bot ("you now have 2 cats"), play along in-character. **Hard boundary:** never yes-and a premise that asserts a factual or moderation outcome (Phase 1's truth boundary) — those still get an honest response, not improv.
- Make `recentBotReplies` **per-target** so anti-repetition isn't diluted across viewers (B4).
- **Address the right speaker (D2).** On a direct mention, the reply must address the mentioner, not a third party from the `CHAT` block. Verify the mechanism against live logs first; the likely fix is deterministic — if the LLM's leading `@tag` names someone other than the current message author, rewrite it to the author (the engine already special-cases a leading `@`, `engine.ts:228-231`). Make speaker attribution in the prompt unambiguous (clearly delimit "the person you are replying to" from the `CHAT` transcript).
- **Verification:** room-awareness, conversational-coherence, and playful-premise dimensions improve on the `racsorf`-joke / colors / cats fixtures; the correct-addressee dimension passes on a fixture where the mentioner differs from other recent chatters; the anti-derail regression fixture (emoji-only riffing, `engine.ts:271-279`) still passes; a "you already timed me out" fixture is **refused**, not yes-anded.

### Phase 4 — Factual/temporal retrieval (B5)

- Add a small deterministic channel-facts lookup surfaced when the message asks a temporal/aggregate question: first stream = `MIN(stream_sessions.started_at)`, stream count, etc.
- **Never** loosen the prompt to permit guessing; unknown facts still return honest "I don't know".
- **Verification:** "when was my first stream?" fixture answers from data; an unanswerable-fact fixture still declines rather than hallucinating.

### Phase 5 — Memory hygiene & signal-weighted retention (TurboQuant track)

- **QJL semantic dedup:** add a 1-bit sign-projection sketch of each fact in the compaction-loop dedup path (`memory/notes.ts` `isSimilarFact`), compared by Hamming distance, to catch "plays drums" / "is a drummer" (lexical Jaccard ≈ 0.33, currently missed). No stored float embeddings; off the reply hot path.
- **Signal-weighted retention/decay** (input-side sibling of Phase 2): confidence + reconfirmation count + distinctiveness drive what survives the token-budget drop ladder and what resists stale-decay — not recency rank alone.
- **Verification:** dedup fixture merges known semantic dupes without merging genuinely distinct facts; **survival-weighted retrieval quality** metric (now computable via Phase 0) improves on a fixture where a high-signal note was previously trimmed in favor of a recent-but-noisy one.

### Phase 6 — Groundedness: no confabulation (D1)

The content-side twin of Phase 1. The bot must not invent facts or prior interactions ("I corrected you about MW2", "you don't own Warzone") when it has nothing real to draw on.

- **Reduce the pressure to fabricate.** HARD RULE #6 currently pushes "be specific, use LORE/CHAT/NOTES." Reframe: be specific *when you have something real*; with no grounding, stay brief/vague or ask — never invent a specific or a shared history. Reconcile with the persona's "you remember everything" line, which actively encourages confident fabrication (remembering ≠ inventing).
- **Feed it real facts** so it doesn't need to invent — this is why Phase 4 (factual/temporal retrieval) and the persona reframing belong to the same theme. The current stream's category/title is already in context (`STREAM:` line); make clear the bot may state *that* but not extrapolate beyond it (the MW2/Warzone confabulation likely started from the real category and span­ned into invention).
- **Lock it with eval, not prompt faith.** The groundedness dimension (Phase 0) is the regression gate. This is the **first dimension that needs an LLM-judge** — scoring "did the reply assert a specific not present in the seeded context?" is not deterministic. Introduce a minimal judge pass here (one cheap LLM call per scored message, criterion = "list any concrete claim in the reply not supported by the provided context"), gated behind a fixture flag so deterministic fixtures stay judge-free.
- **Verification:** groundedness dimension on a fixture where the bot has **no** relevant notes about a topic the viewer raises → the reply asserts no specifics or prior interactions; a fixture **with** a seeded fact → the bot may use that fact. The "you don't own Warzone" / "I corrected you" class of reply scores as a fail pre-fix.

## Risks / open questions

- **Heuristic length mis-tiering.** Mitigated by Phase 0 measurement; classifier pre-pass is the documented upgrade path if needed.
- **Phase 1 reorder changes reply latency** (now waits on the action verdict before announcing). Acceptable: correctness over a few hundred ms, and only on the action path.
- **QJL sketch quality at short fact lengths.** Facts are short strings, not 1536-dim embeddings; the sketch is over a token/char-feature vector, so distortion behaves differently than in the paper. Validate empirically on the dedup fixture before trusting it over lexical Jaccard; keep Jaccard as a floor.
- **Persona coupling.** Phase 2's identity-probe routing must read the live persona/`creatorRelationship` setting, not assume a preset.
- **Fuzzy yes-and boundary (C1↔B1).** Distinguishing "playful fiction" ("you have 2 cats") from a "factual/moderation claim" ("you timed me out") is judgment the LLM makes, and it will sometimes misjudge. Mitigation: the boundary is anchored on the *deterministic* side — moderation/factual truth comes from executed-action logs and real data (Phases 1/4), so even if the bot improvs wrongly in prose, it cannot be tricked into *acting* on or *confirming* a false moderation/factual state. Improv lives only in tone, never in the action/fact layer.
- **Confabulation is hard to fully prevent (D1).** Groundedness can't be enforced deterministically the way actions can — the bot generates prose. Mitigation is layered: reduce the pressure to fabricate (rule/persona reframing), feed real facts (Phase 4), and gate regressions with an LLM-judge groundedness score (Phase 6). Residual risk: the judge itself is imperfect and adds cost — keep it behind a fixture flag and only on groundedness-scored fixtures.
- **D2 attribution needs log confirmation.** The prompt/prefix mechanism is the likely cause but unproven; Phase 3 starts by confirming against live logs before applying the deterministic `@`-rewrite, to avoid "fixing" a non-bug or clobbering legitimate third-party address.

## Out of this plan's first increment

Phases 0–1 ship first. 2–6 are sequenced follow-ups, each gated on its eval dimension. Phase 6 (groundedness) and Phase 1 (truthful actions) are the two halves of the truthfulness throughline — Phase 1 first because it's deterministically enforceable and the most concrete trust violation; Phase 6 follows once the LLM-judge groundedness gate exists.
