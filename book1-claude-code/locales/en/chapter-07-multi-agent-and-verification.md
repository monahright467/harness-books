# Chapter 7 Multi-Agent Work and Verification: Managing Instability Through Division of Labor

## 7.1 Past a certain point, the question is not "can one agent do it" but "how should work be split"

Single-agent friction can be hidden by patience, but once tasks grow, research, implementation, and verification all crowd one context chain, competing for budget, attention, and narrative center. Multi-agent looks like an obvious answer, but it is not cheap: unconstrained parallelism just copies single-agent disorder. The real difficulty is isolating per-agent instability while recombining outputs coherently.

Claude Code source is clear-eyed here: it does not treat subagents as "another chat window." It treats them as managed execution processes with explicit cache boundaries, state boundaries, verification duties, and cleanup responsibilities.

## 7.2 The first principle of forked agents is cache safety

At the top of `src/utils/forkedAgent.ts`, a comment precisely states forked-agent utility duties:

1. share cache-critical params with parent to ensure prompt-cache hits
2. track usage across the whole query loop
3. record metrics
4. isolate mutable state to avoid disturbing parent loop

Among these, cache-critical sharing appears first. That is not accidental. It means fork is runtime-controlled branching. If it is branching, parameters that must remain parent-consistent are crucial; otherwise prompt-cache reuse fails and both cost and latency degrade.

`CacheSafeParams` explicitly includes:

- `systemPrompt`
- `userContext`
- `systemContext`
- `toolUseContext`
- `forkContextMessages`

Source also warns against casually changing `maxOutputTokens`, because thinking configuration may change and thinking config is part of cache keys.

This shows multi-agent is first a runtime economics problem. If each child agent re-burns parent context from scratch, what looks like parallel acceleration is parallel waste. Claude Code's first concern in fork paths is preserving cache discipline.

### Skeleton: forkAgent()

```
// skeleton: forkAgent()  (src/utils/forkedAgent.ts, src/utils/createSubagentContext.ts)
params = CacheSafeParams {
    systemPrompt, userContext, systemContext,
    toolUseContext, forkContextMessages        // must match parent to keep prompt-cache hits
}
ctx = createSubagentContext(parent):
    readFileState  := clone(parent.readFileState)              // isolated by default
    abortController := new ChildAbortController(parent)        // child controller linked to parent
    getAppState    := wrap(parent.getAppState, suppress_prompt)
    setAppState    := noop                                     // read-only by default
    nestedMemoryAttachmentTriggers := new Set()
    loadedNestedMemoryPaths         := new Set()
if opt_in.shareAbortController:  ctx.abortController := parent.abortController
if opt_in.shareSetAppState:      ctx.setAppState      := parent.setAppState
hooks.fire(SubagentStart, { agent_id, agent_type })
track_usage(parent, child)
defer hooks.fire(SubagentStop,  { agent_transcript_path })     // lifecycle must close
```

### Invariants

```
assert child.CacheSafeParams == parent.CacheSafeParams         # fork does not break prompt cache
assert child.mutable_state isolated unless opt_in.share_*       # sharing must be explicit
assert parent.abort ⇒ propagate(child.abort)                    # parent dies, child dies
assert SubagentStart fired ⇒ SubagentStop fired eventually      # lifecycle closes
assert verification_worker ≠ implementation_worker              # verify and implement are separate roles
```

## 7.3 State isolation: reduce contamination before sharing anything

The second key is `createSubagentContext()`: by default, all mutable state is isolated to avoid parent interference. Defaults clone `readFileState`, create child abort controllers, wrap `getAppState` to suppress permission prompts, make `setAppState` a no-op, and recreate sets like `nestedMemoryAttachmentTriggers` and `loadedNestedMemoryPaths`. Sharing happens only via explicit opt-in flags such as `shareSetAppState`, `shareSetResponseLength`, `shareAbortController`.

The reason matters: the main value of child agents is containing local chaos away from main thread. Research misreads, temporary file observations, exploratory reasoning branches, and in-flight tool decisions should not be blindly written back. Claude Code's stance is explicit: sharing requires consent, isolation is default ethics — closer to transactional database design than chat toys.

## 7.4 Coordinator mode: synthesis is the scarce capability

In `src/coordinator/coordinatorMode.ts`, coordinator expectations are disciplined:

- help users reach goals
- direct workers for research, implementation, and verification
- synthesize findings and communicate with users
- answer directly when delegation is unnecessary

The most important line in section 5 of the prompt is: **Always synthesize**. When workers return findings, coordinator must digest and convert them into concrete prompts; it must not forward raw findings and outsource understanding again.

This is the throat of multi-agent design. The scarce capability is not parallel output; it is recompressing distributed local knowledge into actionable, verifiable next steps. Without this layer, multi-agent degrades into polite task forwarding. Claude Code handles this at prompt level: research and synthesis are separated, coordinator owns interpretation, and follow-up prompts must mention concrete files, locations, and changes rather than abstract "based on above findings." Research can be distributed; understanding must reconverge.

## 7.5 Verification must be independent, or implementation completion will impersonate problem solved

`coordinatorMode.ts` also contains a section worth copying verbatim in spirit. It separates common task flow into:

- Research
- Synthesis
- Implementation
- Verification

And explicitly states verification proves effectiveness, not merely code existence:

- run tests with the feature enabled
- investigate errors instead of dismissing them as unrelated
- stay skeptical
- test independently, do not rubber-stamp

This shows Claude Code treats verification as second-layer quality gate, not a side effect of implementation: prompt explicitly stacks implementation self-check plus independent verification worker.

Why it matters: "I changed code" and "the change is correct" are separated by a wide river, and models are good at building paper bridges over it — changes, explanations, plausible test-like output, none of which guarantees real correctness. Independent verification prevents "can modify code" from impersonating "can deliver outcomes"; in multi-agent work this role separation is central.

## 7.6 Hooks and task lifecycle: spawning a subagent is only the beginning

In multi-agent systems, a commonly ignored fact is that spawn is start, not end.

`src/utils/hooks/hooksConfigManager.ts` defines `SubagentStart` and `SubagentStop` hooks. Start events include `agent_id` and `agent_type`. Stop events include `agent_transcript_path`, and even allow exit code 2 so stderr can be fed back and let subagent continue.

This means subagents are explicit lifecycle objects in Claude Code. Start can be observed, pre-stop can be intercepted, transcript paths are traceable. "Subagent end" is itself a managed event.

At another layer, `registerAsyncAgent()` in `src/tasks/LocalAgentTask/LocalAgentTask.tsx` shows each async agent has cleanup handlers. Parent aborts propagate to child abort controllers. On task completion, outputs can be evicted, state updated, and cleanup callbacks unregistered.

This resembles operating-system lifecycle control, not chat tabs:

- is the agent still running
- should child terminate when parent terminates
- should output artifacts be retained
- were cleanup handlers leaked

![Claude Code Multi-Agent Runtime Lifecycle](diagrams/diag-ch07-01-multi-agent-lifecycle.png)

Many multi-agent demos stop at "I can start another agent." Claude Code takes one more crucial step: it treats agents as runtime entities that can leak resources, leave state residue, and become orphans after parent death.

### Subagent lifecycle failure matrix

| Event order | Pre-state | Trigger | Next | Threshold |
|---|---|---|---|---|
| parent abort + child in-flight | child unfinished | parent abort | propagate `child.abort`, wait cleanup | no orphans allowed |
| child crash | child exits abnormally | `SubagentStop` exit ≠ 0 | evict output, fire stop hook | exit 2 feeds stderr back to continue |
| child timeout | child runs too long | `registerAsyncAgent` timeout | abort child, write synthetic result | — |
| cache key drift | `CacheSafeParams` mutated | fork | refuse fork or rebuild cache | must align fully |
| undeclared share | opt-in not enabled | child writes `setAppState` | no-op, parent untouched | isolation is default |
| stale memory conflict | memory disagrees with reality | old record read | trust reality; update/delete memory | verify first |
| leaked cleanup | task ends without unregister | finalize | force unregister + evict | no dangling handlers |

## 7.7 Verification applies to memory and recommendations, not only code

Multi-agent verification is not only a post-code-change activity. `src/memdir/memoryTypes.ts` warns explicitly: memory records can become stale; before giving recommendations from memory, verify current state; when memory conflicts with present reality, trust the current observed state and update or delete stale memory.

The broader fact: verification is the system's baseline habit against temporal and context drift. If a system verifies newly written code but not old memory, assumptions, or indexes, history will still steer it wrong. Verification is both a skill and an institutional discipline: you can delegate work, persist information, and let other agents run ahead, but before users act on outputs, someone must return to current reality and revalidate.

## 7.8 Multi-agent really solves uncertainty partitioning

Claude Code's multi-agent design centers on a plain target: partition uncertainty. Research workers explore in local contexts, implementation workers focus on edits, verification workers specialize in skepticism, and coordinator sits in the middle to converge and communicate.

The main benefit is clear responsibilities, so failures can be located: did research miss a key signal, did synthesis fail to digest findings, did implementation introduce defects, did verification undercheck? A single agent delivering "one thick soup" cannot be debugged by layer. Multi-agent's value is less about speed and more about placing different uncertainties in different containers, then recombining them through coordinator synthesis.

## 7.9 The seventh principle extractable from source

This chapter can be compressed into one sentence:

> Multi-agent systems depend on clear division of labor: research, implementation, verification, and synthesis must run under different constraint containers, then be stitched back into deliverable outcomes by a coordinator.

Claude Code source supports this jointly:

- `forkedAgent.ts` prioritizes cache-safe params, usage tracking, and state isolation, making fork a runtime-control problem first
- `createSubagentContext()` isolates mutable state by default and only shares via explicit opt-in
- `coordinatorMode.ts` requires synthesis instead of forwarding, so understanding cannot be outsourced
- verification is explicitly separated from implementation and must independently prove effectiveness
- `hooksConfigManager.ts` provides `SubagentStart` and `SubagentStop`, making subagents observable lifecycle objects
- `LocalAgentTask.tsx` handles parent abort propagation, cleanup, and output eviction

As portable principles: fork first for cache and state boundaries, then for "persona specialization"; isolate child agents by default and share only by explicit declaration; research can be delegated, synthesis cannot; verification must be role-separated from implementation; agent lifecycle must be observable, interruptible, and reclaimable; the real parallel value is clearer responsibilities, not raw speed.

Next chapter moves from runtime design to organization design: how `CLAUDE.md`, skills, approvals, hooks, and memory become reusable team institutions instead of private expert tricks.
