# Appendix A Checklists: Turning Principles into Executable Constraints

Previous chapters discussed principles. If principles cannot be turned into checklists, they often decay into judgments that sound right but do not hold up in practice. This appendix turns easy-to-say, hard-to-sustain judgments into directly usable lists.

These checklists do not guarantee automatic improvement. They mainly prevent common and repetitive mistakes from recurring. In engineering, many improvements come from repeating fewer avoidable errors.

## A.1 Agent runtime design checklist

If an AI coding agent is going into real engineering workflow, it should answer at least:

- Is there an explicit query loop instead of treating each turn as isolated Q&A?
- Is there a cross-turn state object recording recovery, budget, compaction, hooks, and turn counters?
- Is model output handled as event stream instead of final prose blob?
- Can interrupted tool calls be closed with synthetic results to keep execution ledger complete?
- Are completion, failure, recovery, and continuation modeled as distinct stop semantics?
- Is there context budgeting for long sessions, rather than ad hoc emergency handling only after overflow?

If two or three of these cannot be answered clearly, the system is likely still in "demo-capable" stage rather than "workflow-capable" stage.

## A.2 Prompt design checklist

System prompt should not be merely long. It should be layered with explicit duties.

At minimum, check:

- Are identity, behavior rules, tool constraints, and output discipline organized separately?
- Is prompt source precedence explicit (default, project, custom, append, agent-specific)?
- Are dangerous operations, unauthorized behavior, and verification discipline written as explicit rules rather than vague hints?
- Is prompt prevented from carrying duties that belong to runtime enforcement?
- Can team maintain it stably, rather than appending emergency text after every bug?

A practical test: if removing one section causes structural behavior change, that section is likely true control-plane logic. If behavior barely changes, it may be decoration.

## A.3 Tool and permission design checklist

Any system that lets models touch the world should ask first:

- Do tool calls pass through unified orchestration rather than direct model invocation?
- Does concurrency require explicit safety proof rather than default allowance?
- Is there semantic branching like `allow / deny / ask`?
- Are high-risk tools treated as special cases rather than identical to ordinary tools?
- Can interrupts, fallbacks, and sibling failures produce explicit closure semantics?
- Can execution preserve causal chain and avoid dangling `tool_use` blocks?

If Bash and ReadTool are governed almost identically, risk understanding is usually insufficient.

## A.4 Context governance checklist

Any long-session agent will eventually be disciplined by context limits. Early governance is cheaper.

At minimum, check:

- Are long-lived rules, persistent memory, session continuity, and temporary dialogue layered?
- Is there explicit separation between entrypoint files and body files to prevent index bloat?
- Are there token budgets for memory, session memory, and skill attachments?
- Is compact output space pre-reserved instead of waiting until the window is full?
- After compact, are work semantics restored (plans, skills, key files, tool state)?
- Is there recovery strategy when compact itself fails?

Systems with good context governance often look somewhat frugal. That frugality is usually a strength.

## A.5 Error recovery checklist

Recovery design has two common failures: no design, and dead-loop design.

At minimum, check:

- Are recoverable errors routed to recovery branches before being surfaced immediately?
- Are recovery paths layered from low-destructiveness to high-destructiveness?
- Are there guards preventing reactive-compact, stop-hook, and retry loops from biting each other?
- After `max_output_tokens`, is continuation prioritized over recap?
- Do automated recoveries include counters, retry caps, and circuit breakers?
- Are interrupts treated as failure states requiring semantic closure?

A recovery system that never stops can be as dangerous as a recovery system that never starts.

## A.6 Multi-agent checklist

Multi-agent design should organize uncertainty, not multiply confusion. Check:

- Does fork design preserve prompt-cache consistency through cache-safe params?
- Is mutable state isolated by default for child agents?
- Are research, implementation, verification, and synthesis roles explicitly separated?
- Does coordinator truly synthesize instead of only forwarding worker output?
- Is verification independent from implementation?
- Is agent lifecycle observable, interruptible, and reclaimable?
- Does parent abort propagate to children to prevent orphan tasks?

If a system claims multi-agent but all agents do similar work and nobody owns synthesis or verification, it is usually parallelized disorder.

## A.7 Team adoption checklist

The most common rollout misread is mistaking personal proficiency for institutional maturity.

Before rollout, verify:

- Is layered `CLAUDE.md` in place, with clear guidance on what belongs and what does not?
- Is verification definition standardized before scaling skill volume?
- Are approval boundaries tiered by consequence and environment sensitivity?
- Are key institutions attached to appropriate hook timing instead of stuffed into static docs?
- Are transcript, task output, and hook events retained as replay evidence?
- Is there maintenance policy for stale memory, obsolete rules, and invalid skills?

Teams that can sustainably use agents are usually teams where ordinary members can operate correctly within institutions, not teams that depend on a few experts.

## A.8 Review question set

To review an AI coding-agent proposal, ask directly:

- Which behaviors are constrained by prompt and which are enforced by runtime?
- Who blocks tool misuse, and at what layer?
- When is context compacted, and how are work semantics reconstructed afterward?
- How are prompt-too-long and max-output-tokens recovered differently?
- After interruption, how is transcript consistency maintained with tool results?
- In multi-agent flows, who owns synthesis and who owns verification?
- Does failure recovery include circuit breakers and anti-loop guards?
- How does the team audit what the agent did and why?

If answers frequently reduce to "we can add that later," runtime is likely not truly designed yet.

## A.9 Final short checklist

If everything above feels too long, keep these six: design permission before capability; rollback before autonomy; verification before delivery; context budgets before long dialogue; lifecycle before multi-agent; institutions before expecting team proficiency. Meeting them does not guarantee excellence; missing them usually means the system has just not failed yet.

## A.10 Implementation seeds (pseudocode stubs)

The skeletons from earlier chapters collapsed into reusable starting points. Full forms live in their chapters; this keeps only the minimal spine.

### A.10.1 queryLoop (skeleton, from Chapter 3)

```
state = { messages, toolUseContext, autoCompactTracking, turnCount, transition, ... }
while not done(state):
    govern_input(state)                 // memory / snip / collapse / autocompact
    events = stream_model(state)
    for e in events:
        if e.is(tool_use): schedule(e, state.toolUseContext)
        if e.is(api_error): return surface(e)
    if interrupted: drain_tools_with_synthetic_results(state); break
    state = advance(state, recover_if_needed(state))
assert state.turnCount monotonic ∧ every tool_use has tool_result
```

### A.10.2 permission decision (skeleton, from Chapter 4)

```
decision = hasPermissionsToUseTool(tool, input, ctx)
match decision:
    allow: exec(tool, input)
    deny:  reject(reason)
    ask:   route_to(coordinator | worker | classifier | interactive)
assert decision ∈ {allow, deny, ask}        # three-valued, never collapses
assert ask never auto-escalates to allow    # no unauthorized escalation
```

### A.10.3 forkAgent (skeleton, from Chapter 7)

```
params = CacheSafeParams { systemPrompt, userContext, systemContext, toolUseContext, forkContextMessages }
ctx    = createSubagentContext(parent)       // mutable state isolated by default
hooks.fire(SubagentStart, { agent_id, agent_type })
defer hooks.fire(SubagentStop, { agent_transcript_path })
assert parent.abort ⇒ propagate(child.abort)
```

### A.10.4 recoverFromError (skeleton, from Chapter 6)

```
on recoverable_error(e):
    if e.is(prompt_too_long):
        if stagedCollapse > 0: recoverFromOverflow()
        elif not hasAttemptedReactiveCompact: tryReactiveCompact()
        else: surface(e); skip_stop_hooks()
    if e.is(max_output_tokens):
        if cap < MAX: raise(maxOutputTokensOverride); retry()
        else: append(meta_continue_msg); retry()
assert consecutiveFailures < MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES
```
