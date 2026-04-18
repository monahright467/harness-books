# Chapter 6 Errors and Recovery: An Agent System That Keeps Working After Failure

## 6.1 The least trustworthy sentence in engineering is "under normal conditions"

Many design docs describe only the "normal path," as if a pretty happy-path flow could make errors secondary. Once agent systems enter real runtime, this breaks fast: models get truncated, requests go too long, hooks create loops, tools get interrupted, fallback paths trigger, and recovery logic itself fails.

Maturity cannot be judged by how human-like the system sounds when smooth; it must be judged by whether failures look like system behavior. Claude Code's strength here is not pretending errors are rare. Source repeatedly reflects a calm assumption: errors are part of the main path, and recovery must be predesigned runtime mechanism.

## 6.2 `prompt too long` is seasonal, not exceptional

For long-session agents, `prompt too long` is not an edge case. It is a season that eventually arrives. Treating it as rare exception is an invitation to be corrected by production.

Claude Code `query.ts` does not treat it as accidental. It can temporarily withhold such errors instead of surfacing immediately. During streaming, withheld logic identifies recoverable classes including:

- prompt too long
- media size error
- max output tokens

Meaning: some errors should be handed to recovery first, then surfaced only if recovery fails. Users usually care less about raw error type than whether work can continue.

For prompt-too-long specifically, Claude Code first tries cheaper, less destructive recovery. If context collapse is enabled, it first calls `recoverFromOverflow()` to flush staged collapse; only if insufficient does it call `reactiveCompact.tryReactiveCompact()`. Recovery is layered: clear known backlog first, then do heavier full compaction.

This sequence is highly practical. Good recovery does not hit every error with one giant hammer. It tries to preserve fine-grained context first and escalates only when required.

## 6.3 Reactive compact: recovery must not become dead-loop machinery

A common mistake in recovery systems is both naive and costly: once an error is "recoverable," keep retrying until recoverable turns into resource disaster.

Claude Code is explicitly defensive against this. Two places in `query.ts` show it.

First is `hasAttemptedReactiveCompact`. Once reactive compact has already been attempted, same-class failures are not blindly retried. If compact did not help, repeating compact often just replays the same failure in different posture.

Second is stop-hook dead-loop guards. Source comments are direct: if stop hooks run after unrecoverable prompt-too-long states, death spirals become possible:

error -> hook blocking -> retry -> error -> hook blocking

This is not literary writing; it is brutally honest engineering. The most dangerous errors are often branches where failure and recovery start reproducing each other.

So when prompt-too-long cannot recover, Claude Code surfaces the error directly and skips stop hooks. Continuing formal process there only ritualizes failure.

## 6.4 `max_output_tokens`: recovery should prioritize continuation

Many model products handle truncation with polite waste: "Sorry, I was cut off, let me summarize." It sounds nice and helps little.

Claude Code behavior after `src/query.ts:1185` is far more engineering-aligned. It first tries lower-cost recovery: if current cap is conservative, raise `maxOutputTokensOverride` and rerun the same request. No meta message, no courtesy preface, just one more chance to finish the task.

If higher cap still fails, it escalates to second layer: append a concise meta user message that says, in effect:

continue directly; no apology; no recap; if cut mid-sentence, continue from that half sentence; split remaining work into smaller chunks.

This instruction is highly instructive. Claude Code treats recovery as preserving task continuity, not preserving social polish. In long tasks this difference is huge. Each truncation recap burns additional budget and increases semantic drift. Eventually system spends turns recapping itself instead of doing the task.

For `max_output_tokens`, good recovery is usually continuation-first. Claude Code explicitly optimizes for that.

## 6.5 Auto-compact circuit breaker: recovery systems need governance too

Previous sections are about "how to recover one failure." `src/services/compact/autoCompact.ts` tackles another layer: what if recovery mechanism itself keeps failing?

Source answer is simple and correct: stop retrying forever.

`AutoCompactTrackingState` tracks `consecutiveFailures`. Once failure count exceeds threshold, even if `shouldAutoCompact` says compact is due, system skips compact directly. Source comments reference historical waste: sessions once burned large amounts of API calls on repeated autocompact failure, so circuit breaker was required.

Circuit breaking means admitting current recovery method is ineffective in current state. Mature systems must not only record success metrics; they must know when to back off under repeated failure. Recovery systems without brakes are like vehicles without brakes.

Harness Engineering principle here is hard and clear: any automated recovery must be countable, rate-limited, and breakable.

### Circuit-breaker invariants

```
assert withheld_error ∈ {prompt_too_long, media_size, max_output_tokens}  # recoverable set is fixed
assert hasAttemptedReactiveCompact ⇒ skip further reactive compact        # no self-loop
assert consecutiveFailures < MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES          # breaker engaged
assert compact aborted by user ⇏ summary_success                           # abort ≠ success
assert every withheld recoverable_error surfaces iff recovery exhausted    # withheld must exit
```

### Recovery-path failure matrix

| Event order | Pre-state | Trigger | Next | Threshold |
|---|---|---|---|---|
| PTL → collapse | `stagedCollapse > 0` | `prompt_too_long` | `recoverFromOverflow()` | — |
| PTL → compact | `stagedCollapse = 0` | `prompt_too_long` | `tryReactiveCompact()` | once per turn |
| PTL → surface | `hasAttemptedReactiveCompact` | `prompt_too_long` | surface directly; skip stop hooks | — |
| compact PTL | compact input too long | inner `prompt_too_long` | `truncateHeadForPTLRetry()` | drops early API rounds in chunks |
| MOT → cap↑ | cap < MAX | `max_output_tokens` | raise `maxOutputTokensOverride` | cap ∈ {conservative, max} |
| MOT → continue | cap = MAX | `max_output_tokens` | append meta user msg, continue | no recap, no apology |
| autocompact streak | `consecutiveFailures` ≥ threshold | next trigger | skip compact, surface | `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3` |
| user abort | streaming with pending `tool_use` | Esc | consume remaining + synthetic `tool_result` | ledger must close |

## 6.6 Compact itself can fail, so recovery action needs its own recovery

`compactConversation()` contains a very realistic moment: compact requests themselves can hit `prompt too long`.

There is black humor here. System sends summary requests to reduce context, then summary requests fail because context is too large. Many designs hide this scenario because it is embarrassing. Engineering systems prioritize survival over elegance.

Claude Code handles this with `truncateHeadForPTLRetry()` in `compact.ts`. When compact input is too large, it strips older API rounds in chunks from the head and retries compact, preventing users from being stuck in "cannot compact compact."

Trade-off is explicit: this fallback is lossy and may drop history, but it prioritizes not deadlocking users. Source comments describe it as last-resort escape hatch.

Value of this choice: it does not deny hard limits. When the system is choking, first priority is restoring breathing, then preserving high-fidelity history.

![Claude Code Recovery Decision Paths](diagrams/diag-ch06-01-recovery-decision-paths.png)

![Claude Code Compact Fallbacks](diagrams/diag-ch06-02-compact-fallbacks.png)

## 6.7 Abort semantics: interrupts are part of recovery

Many teams classify abort under UX only. Runtime-wise, interrupts are failure states requiring semantic closure.

Claude Code handles this at two layers. In `query.ts`, streaming interrupt consumes `StreamingToolExecutor.getRemainingResults()` and synthesizes tool results for issued-but-unfinished calls, preventing dangling commitments. In `compact.ts`, the compact abort controller is passed into the forked agent and `APIUserAbortError` is handled explicitly, preventing "compact cancelled by user Esc" from being counted as a successful summary.

Interruption is not merely "user stopped reading"; it is a state transition requiring correct closure. Recovery that handles exceptions but ignores interruption leaves semantically half-broken execution traces.

## 6.8 Error handling protects narrative consistency of execution

Claude Code's recovery philosophy has one often-overlooked goal: preserve narrative consistency of execution — whether the system can still explain what it attempted, why it failed, which recovery path was used, and whether it is now continuing, stopping, or rerouting.

Fields like `transition.reason`, `maxOutputTokensRecoveryCount`, `hasAttemptedReactiveCompact`, compact boundaries, and synthetic error messages in `query.ts` exist to keep this narrative unbroken — they are anti-amnesia mechanisms. Without narrative consistency, systems keep outputting text while internally decomposing: users see filler, ops sees hook-retry and compact-retry loops with unclear causality, and teams can no longer explain what the system actually went through. Recovery fixes not only errors but the system's self-explainability; once explainability breaks, the engineering object degrades into opaque magic.

## 6.9 The sixth principle extractable from source

This chapter can be compressed into one sentence:

> An agent system shows its reliability by maintaining explainable, bounded, and resumable execution order after failure.

Claude Code source supports this across key points:

- `query.ts` withholds recoverable errors for branch-level transformation before surfacing
- Prompt-too-long recovery layers collapse drain before reactive compact, ordered by cost and destructiveness
- `hasAttemptedReactiveCompact` and stop-hook guards prevent recovery self-loops
- `max_output_tokens` handling escalates cap first and prefers direct continuation over polite recap
- `autoCompact.ts` tracks consecutive failures and enforces circuit breaking
- `compact.ts` includes fallback for compaction's own prompt-too-long failure

As portable principles: layer recovery paths rather than using one heavy hammer; recovery logic must be loop-safe; automated recovery needs counters and circuit breakers; after truncation, continuation beats summary; interruptions are semantic failure states requiring closure; reliability is proven by whether the system can still explain itself after errors.

Next chapter turns to a harder class of problems: multi-agent and verification. When systems move from "one agent recovers itself" to "one agent delegates and another verifies," error and recovery stop being purely single-thread concerns and become organizational design.
