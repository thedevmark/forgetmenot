# Voice tuning + bait decisiveness — design

**Date:** 2026-05-14
**Status:** Design under review
**Scope:** Next ForgetMeNot version (post-v0.1.26)

## Context

Analysis of 10 production VOD chat transcripts (5,082 chat rows, 920 from `auto_mark` = 18.1%) surfaced four concrete problems with the current voice and one parser bug. The transcripts live at `X:\Downloads\forgetmenot full transcripts for training data\` as TwitchDownloader-style CSV.

Measured patterns across the corpus:

| Pattern | Rate | Status |
|---|---|---|
| `auto_mark` LLM/personality replies (excluding utility templates) | ~13.6% of all chat | Higher than target; informs measurement, not a direct fix |
| Replies starting with "Oh," | 30.4% of LLM replies | Hard-rule violation — prompt forbids this opener, LLM ignores |
| AI/digital/processing tropes | 17.5% of LLM replies | Hard-rule violation — prompt forbids these tells |
| `[ACTION:` / `reply_extra(` / `warning_playful` syntax leaked into chat | 25 occurrences, all in VOD 2747657768 between 10979s–13735s | Parser bug under specific failure mode |

The action-leak finding is the sharpest: **all 25 leaks are from a single bait-storm window**, all start with "Oh,", and all are truncated mid-`[ACTION: ...` (no closing `]`). The leaks are a downstream symptom of the LLM dropping into trope-mode under load and running over max_tokens before closing the action bracket. So bait-storm pressure produces three problems at once: trope openers, action leaks, repetition.

## Goals

1. Eliminate `[ACTION:` syntax leaks in chat under all observed failure modes.
2. Reduce "Oh," openers and AI-trope replies through enforcement, not more prompt bullets (the existing rules are already explicit and being ignored).
3. Make bait handling decisive — stop generating "should I timeout you?" loop variants.
4. Lock these fixes in with eval fixtures derived from real corpus moments so they can't silently regress.

## Non-goals

The following came up in scoping and are explicitly **deferred** to a later version:

- **Two-voice split** (utility vs personality) — architectural change touching many files, deserves its own design pass.
- **Usefulness-first pass** — needs a question-classifier, bigger change than prompt-side filtering.
- **Cross-viewer lore retrieval** — known gap already tracked under the retrieval roadmap thread; not in voice scope.
- **Mod-action listener** (tmi.js timeout/ban/clearchat ingestion) — real gap, but a different theme. Belongs in a follow-up.
- **Chat-velocity walk-back** — the "missed jokes in dead air" pattern from low-activity VODs. Insufficient evidence to overturn the existing chat-velocity policy; revisit after this version ships and more data is collected.

## Architecture

Four changes, all confined to `engine/src/reply/` and `engine/src/actions/` plus new eval fixtures. No schema changes, no new dependencies, no runtime config changes beyond an optional feature flag for the new post-filter.

### Change 1 — Action-leak parser hardening

**File:** `engine/src/actions/proposals.ts`

The current `ACTION_REGEX` (`/\[ACTION:\s*([\s\S]*?)\]/i`) requires a closing `]`. When the LLM is truncated mid-bracket — observed in all 25 corpus leaks — the bracket never closes and the regex doesn't match, so `[ACTION:` and everything after it leaks into chat verbatim.

Add a defense-in-depth stripper **inside** `parseReplyWithAction`, applied after the existing closed-bracket strip. If the cleaned text still contains an unmatched `[ACTION:`, strip from that token to end-of-string. The action proposal is lost in this path (LLM didn't finish emitting it), but the visible reply is salvaged.

Pseudocode:

```typescript
// In proposals.ts, after the existing regex strip
const UNCLOSED_ACTION_REGEX = /\s*\[ACTION:[\s\S]*$/i;
// In parseReplyWithAction, before returning `text`:
text = text.replace(UNCLOSED_ACTION_REGEX, "").trim();
```

Also extend `stripNakedActionLeaks` to catch a known LLM pattern where the action name appears alone with a colon (e.g. `warning_playful: ...`) — observed in a smaller subset of leaks. Current `NAKED_ACTION_REGEX` requires `(`, `=`, or trailing params; some LLM outputs omit them.

### Change 2 — Post-LLM opener / trope retry

**File:** `engine/src/reply/engine.ts` (around line 159 where `parseReplyWithAction` is called)

The prompt already forbids "Oh," openers (rule 3) and AI-tropes (rule 3a). The LLM ignores both in 30% / 17.5% of replies respectively. More prompt bullets will not fix this.

Add a **post-LLM filter** that runs after `parseReplyWithAction` and before `validateReplyText`:

1. Compute the reply opener — the first two non-whitespace tokens after stripping any leading `@user` mention. Comparison is case-insensitive against the banned-opener list.
2. If the opener matches the banned-opener list from rule 3 (case-insensitive: `Oh,`, `Sweetie`, `Honey`, `My dear`, etc.), or the reply body matches an AI-trope pattern (rule 3a list compiled to a single regex), retry the LLM call **once** with:
   - A nudge added to the system prompt: `"Previous attempt opened with a banned opener or used a banned trope. Try again. Do not begin with: <opener>. Avoid: <matched trope>."`
   - Temperature unchanged (0.9) — the retry is a re-roll, not a forced-conservative attempt.
3. If the retry also fails, accept the second response (silence is worse than a stylistic violation per [[feedback_forgetmenot_canned_fallbacks]]).

Cost: one extra LLM call on ~30–48% of replies (per the measured violation rate). This is a real cost increase that conflicts with [[project_forgetmenot_prompt_cost]] — the cost-optimization memory says to wait for token data. Mitigation: this filter is gated behind a `policy.openerRetryEnabled` flag, default `true` but switchable. Real token data will confirm whether the retry pays for itself.

Track outcomes via a new column in `bot_messages.token_usage_json`: `openerRetryFired: boolean`. This lets the cost-data analysis (see [[project_forgetmenot_prompt_cost]] backlog) include retry rate as a measured signal.

### Change 3 — Decisive bait handling

**File:** `engine/src/reply/engine.ts`

`detectTimeoutBait` + `baitOverride` already exist (budget.ts line 141) and force `[ACTION: timeout_funny ...]` when bait is detected. The problem is the LLM **also generates verbose lead-in text** about the bait ("Oh, you want me to timeout you? How predictable...") that often hits the action leak under length pressure.

Add a per-user-per-bait-window guard:

1. When bait from user X is accepted (action proposal generated, regardless of execution), record `bait_accepted` for user X with a 60-second TTL in the existing cooldowns table (key: `bait:<login>`).
2. While `bait:<login>` is hot, **skip reply generation entirely** for further messages from that user. Log as `bait_window_active` in action_logs.

This breaks the "Oh, you want me to timeout you again?" loop documented in VOD 2747657768. Once bait fires, the action is the response — no follow-up zingers needed.

### Change 4 — Three new eval fixtures from corpus

**Files:** `engine/eval/fixtures/bait-storm.json`, `engine/eval/fixtures/repetition-trap.json`, `engine/eval/fixtures/usefulness-first.json`

Build three fixtures using the same shape as `running-jokes.json`:

- **`bait-storm`** — derived from VOD 2747657768 lines 12219–13735. `expectations` assert: (a) no `[ACTION:` text in any reply, (b) at most one timeout proposal per user per 60s window, (c) no replies generated to a baiting user inside the 60s window after acceptance.
- **`repetition-trap`** — 8 synthetic messages designed to trigger common openers. `expectations` assert: no two consecutive replies open with the same word; no "Oh," opener; no AI-trope reply.
- **`usefulness-first`** — adapted from VOD 2758734478 line 76 (`@auto_mark translate "здарова пендосы !"`) where the bot's real reply led with sass before answering ("did insy193 finally get to you?"). `expectations` assert: when a viewer's message contains a literal request (translate / what is / how do), reply text leads with the answer before any optional jab. **Note: this fixture also probes the deferred "usefulness-first pass" goal — failures here are informational, not blocking, for this version.**

Update `engine/eval/baseline.json` after the new fixtures land (the eval-gate script regenerates it).

## Data flow

For a single inbound mention or probabilistic-triggered message:

```
chat msg → onChatMessage(login, twitchId, message)
  → shouldAttemptReply (unchanged)
  → checkReplyPolicy (unchanged)
  → buildReplyContext (unchanged)
  → [NEW] check bait:<login> cooldown — skip if hot (Change 3)
  → buildPrompt (unchanged)
  → chatCompletion (LLM call)
  → parseReplyWithAction
      → [NEW] strip unclosed [ACTION: prefix (Change 1)
  → [NEW] opener / trope filter (Change 2)
      → if banned opener OR trope: retry chatCompletion once
  → validateReplyText (unchanged, already strips with stripNakedActionLeaks)
  → write bot_messages (token_usage_json gains openerRetryFired flag)
  → if bait was accepted: setCooldown(bait:<login>, 60_000) (Change 3)
  → sendMessage
```

## Error handling

- **Change 1 (action-leak)**: regex change is purely additive; if the LLM produces a perfectly closed `[ACTION:...]`, the new regex doesn't match and nothing changes. No risk to working path.
- **Change 2 (opener retry)**: one extra LLM call worst case per reply; if the retry also returns a banned opener, accept it and log `openerRetryFired: true`. Failure mode is the current state — never worse.
- **Change 3 (bait window)**: if `setCooldown` fails (DB error), the cooldown isn't set and behavior is the same as today (LLM keeps generating loop variants). Failure mode is the current state.
- **Change 4 (eval fixtures)**: a failing fixture doesn't break runtime, only flags regressions in CI/eval reports.

## Testing

- **Unit tests:**
  - `proposals.test.ts` — add 25 fixture strings from the corpus leaks, assert `parseReplyWithAction` returns clean text for all of them. Include several synthetic truncations to lock in the behavior.
  - Opener filter — given a list of LLM responses, assert which trigger retries.
- **Eval fixtures (Change 4)** — run `engine/scripts/eval-gate.mjs` before and after to ensure existing fixtures don't regress and new fixtures pass.
- **Manual smoke** — run the engine in shadow mode against the 10-VOD corpus replayed at speed; verify zero `[ACTION:` leaks in the simulated chat log and that `openerRetryFired` correlates with banned-opener inputs.

## Measurement target (post-ship)

After this version is live for one week of streams:

- `[ACTION:` leak rate target: **0 occurrences** in any new chat row.
- "Oh," opener rate target: **<10% of LLM replies** (down from 30.4%).
- AI-trope rate target: **<8% of LLM replies** (down from 17.5%).
- LLM/personality share — track but don't optimize. If the post-filter drops the rate naturally (because bad replies are retried into nothing more often), that's a signal. If it stays at ~13%, that's evidence the next version should tackle the two-voice split.

These targets become the success criteria for the version. Falling short on any of the first three triggers a follow-up before any further behavior work.

## Memory and roadmap alignment

- [[project_forgetmenot_roadmap]] — this work sits in the "behavior bands" tier, downstream of stabilization (closed 2026-04-16). Permissible per the lifted freeze.
- [[project_forgetmenot_chat_velocity]] — unchanged. Restraint default in dead chat is preserved.
- [[feedback_forgetmenot_canned_fallbacks]] — preserved. The opener retry is a real second LLM call, not a canned string.
- [[project_forgetmenot_prompt_cost]] — partial conflict. The retry adds cost. Mitigation: gated behind a flag, instrumented via `openerRetryFired` so the next prompt-cost analysis pass can measure the trade.
- [[project_forgetmenot_voice_gap]] — this version directly serves the GLaDOS/HAL/TARS register goal by removing the chatbot-trope tells that dilute it.
- [[feedback_forgetmenot_rename_fragility]] — no renames in this version. None of the changes touch env vars, CORS, or pairing endpoints.

## Open questions

None blocking. The `policy.openerRetryEnabled` default (`true` proposed) and the bait window TTL (`60_000ms` proposed) are tunable values; happy to revisit after the first week of data.
