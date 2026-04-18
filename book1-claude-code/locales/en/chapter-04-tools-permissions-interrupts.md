# Chapter 4 Tools, Permissions, and Interrupts: Why Agents Cannot Touch the World Directly

## 4.1 Once models call tools, the nature of risk changes

A text-only model mostly costs communication time when wrong. Once it starts calling tools, risk changes in kind: tools are not opinions but actions, and actions touch the real world. A wrong explanation stays at understanding level; a wrong command deletes files, kills processes, mangles Git history. Capability growth comes with consequence growth.

So the central question is simple: who constrains these tools? Claude Code's answer is to turn tools into managed execution interfaces, not let models reach out directly.

## 4.2 Tool orchestration is part of behavioral constitution

`runTools()` at `src/services/tools/toolOrchestration.ts:19` does something representative: it does not directly execute the `tool_use` list — it batches by concurrency safety. `partitionToolCalls()` at `:91` reads each tool's `inputSchema` and calls `isConcurrencySafe()`; safe calls go into parallel batches, unsafe calls split into serial units.

This looks like performance tuning but is really consistency design. Once concurrency is allowed, an old question returns: who decides context evolution, and in what order? In parallel paths (`:31`–`:63`), Claude Code does not let the fastest tool mutate context first; it buffers `contextModifier` callbacks and replays them in original block order. Execution is parallel while semantic context evolution stays deterministic.

Classic engineering conservatism: concurrency may improve throughput but must not break causality. Mature agent systems do not idolize parallelism — they treat it as an exception that must prove safety, not a default freedom.

## 4.3 A lot happens before a tool actually runs

Many people assume that once `tool_use` appears, execution naturally follows. Robust systems do not work that way. After `src/services/tools/toolExecution.ts:30`, `runToolUse()` is already wrapped with permission logic, hooks, telemetry, and synthetic error materialization — pre-checks, in-flight events, post-execution correction, and failure compensation.

Tools here are not ordinary internal library functions. Internal functions assume stable callers who own consequence; tool interfaces sit between models and the external world, so runtime cannot assume stable caller judgment. That is why so many wrappers exist: caller is the most unstable variable. Tools should not be modeled as "extensions of model capability" but as external capabilities whose risk must be runtime-managed. Once you accept that, permission, hooks, interrupts, and synthetic results become basics, not burdens.

## 4.4 Permission comes before capability

Claude Code's permission entry point is after `src/hooks/useCanUseTool.tsx:27`. The very existence of `CanUseToolFn` says something important: tool allowance is not decided by model intent alone; it goes through an authorization chain.

Inside `useCanUseTool()`, runtime does not auto-run just because the model requested a tool. It first calls `hasPermissionsToUseTool(...)`, see `useCanUseTool.tsx:37`. Results are split into `allow`, `deny`, or `ask`. This may look ordinary, but it is fundamental. Mature authorization systems need more than yes/no; they need a third state where system itself should not decide on behalf of user.

After `useCanUseTool.tsx:64`, branches continue:

- `deny`: reject directly
- `ask`: route to coordinator, swarm worker, classifier, or interactive approval
- `allow`: execute

This structurally rejects a common dangerous idea: if the model understood user intent, it has authority to execute. It does not. Intent understanding is not authorization, and certainly not persistent authorization. Systems must split "can do" from "may do."

From this angle, permission clarifies agent role: model can propose actions, but runtime, rules, and user decide release. Capability and authority are deliberately separated.

### Skeleton: the permission chain

```
// skeleton: useCanUseTool()  (src/hooks/useCanUseTool.tsx:27)
decision = hasPermissionsToUseTool(tool, input, ctx)
match decision:
    case allow:  return exec(tool, input)
    case deny:   return reject(reason)
    case ask:    return route_to(coordinator | swarm_worker | classifier | interactive_prompt)

assert decision ∈ {allow, deny, ask}                    # three-valued, no boolean shortcuts
assert ask never auto-escalates to allow                # no unauthorized escalation
assert deny is sticky for this tool_use_id              # no silent retry to allow
```

![Claude Code Permission Decision Layers](diagrams/diag-ch04-01-permission-decision-layers.png)

## 4.5 Permission results are runtime semantics, not booleans

After `src/utils/permissions/PermissionResult.ts:23`, Claude Code even defines explicit describers for permission semantics: `allow`, `deny`, and `ask`. This detail matters. Permission is not an internal bool; it is a runtime object with independent meaning.

Why this matters: a permission system must let runtime clearly express why a step did not continue. When an agent says "I need confirmation," it is declaring responsibility boundaries. Once boundaries are explicit, refusal, approval, cache rules, temporary grant, and persistent grant all have a place.

Put simply, if an agent cannot distinguish "I can do this," "I cannot do this," and "I must ask," it should not touch a terminal. Terminals do not fill missing semantics. Terminals only execute.

## 4.6 StreamingToolExecutor proves interrupt is first-class semantics

Once tools are parallel and streaming, interrupt handling becomes immediately complex. Runtime now faces a queue with states like queued, executing, completed, yielded, not one single action.

After `src/services/tools/StreamingToolExecutor.ts:34`, Claude Code explicitly makes this a dedicated streaming executor. Most important is how it handles interruption and discard.

At `StreamingToolExecutor.ts:64` to `:70`, runtime can discard the current tool set during streaming fallback. At `:153` to `:205`, it generates synthetic error messages for different causes:

- sibling error
- user interrupted
- streaming fallback

After `:210`, it further distinguishes interruption causes:

- cancelled due to sibling tool failure
- cancelled due to user interrupt
- dropped due to fallback

At `:233` and below, each tool can define `interruptBehavior`, deciding whether to `cancel` or `block` when user interjects.

This design matters because Claude Code does not treat interruption as "a special execution error." It treats interruption as semantics equal in importance to execution itself. Runtime must know not only whether a tool may start, but how it ends when interrupted, how results are closed, and whether new messages can interleave.

That is a core Harness Engineering trait: design start and stop both. Execution systems without stop semantics eventually depend on user-side hard interruption to finish design.

### Concurrency & interrupt failure matrix

| Event order | Pre-state | Trigger | Next |
|---|---|---|---|
| one tool fails in parallel batch | siblings still executing | sibling error | keep others; replay `contextModifier` in block order |
| user interjects + `interruptBehavior=cancel` | tool not yet done | user interrupt | cancel, emit `user interrupted` synthetic result |
| user interjects + `interruptBehavior=block` | tool not yet done | user interrupt | finish tool, block new message until done |
| streaming fallback | batch queued/executing | fallback | discard batch, yield fallback synthetic results in order |
| abort with pending `tool_use` | any | abort signal | synthesize `tool_result`, close the ledger |
| Bash compound command exceeds subcommand cap | classifier rejects | safety check | `deny`; do not route to `ask` |

![Claude Code Tool Execution Lifecycle](diagrams/diag-ch04-02-tool-execution-lifecycle.png)

## 4.7 Why Bash is always more suspect than other tools

In Claude Code's tool world, Bash is a risk amplifier, not a normal tool. It is too general — a file-read tool will not casually kill processes, a grep tool will not secretly push commits, but Bash can do almost everything. Claude Code's distrust is explicit.

First layer is prompt guidance in `src/tools/BashTool/prompt.ts:42` and below: detailed rules for git, PRs, dangerous commands, hooks, force push, and interactive flags — disciplined verbosity where consequences are large. Second layer is permission and safety classification. `src/tools/BashTool/bashPermissions.ts:1` handles shell semantics, command prefixes, redirection, wrappers, safe env vars, classifier routing, and rule matching; after `:95` a subcommand-count cap prevents compound commands from escaping checks.

Bash is a dangerous channel requiring special governance, not a generic command interface. A reusable judgment: high-risk capability deserves special governance, not generic capability treatment; treating Bash as ordinary is usually design laziness.

## 4.8 Tool systems protect users and protect runtime itself

Permissions, scheduling, and interrupts look like user protection mechanisms, but they also protect runtime consistency itself. If an agent system allows unresolved problems such as incomplete `tool_result`, out-of-order context mutation, unbounded parallel side effects, and ambiguous interrupt semantics, the first thing to break is internal consistency.

This is tightly coupled across `query.ts` and tool execution layers. Chapter 3 discussed how query loop synthesizes missing tool results on interrupt. This chapter shows StreamingToolExecutor has its own discarded/errored/sibling abort/interrupt behavior mechanics. Together they preserve a traceable causal chain for:

- what was executed
- what was unfinished
- why it stopped

That is another core harness meaning: preserving order for the system itself. Many constraints look like misoperation prevention on the surface, but at deeper level they prevent runtime from collapsing into unexplainable state fragments.

## 4.9 The fourth principle extractable from source

This chapter can be compressed to one line:

> Tools are managed execution interfaces; permission is an organ of the agent system.

Claude Code source supports this jointly:

- `toolOrchestration.ts` batches before execution, so scheduling comes before impulse
- `toolExecution.ts` wraps hooks, permission, telemetry, and synthetic errors around execution, so calls are never bare
- `useCanUseTool.tsx` splits authorization into `allow / deny / ask`, making authority a first-class semantic branch
- `StreamingToolExecutor.ts` defines interruption, fallback, and sibling-failure semantics, making stopping as important as starting
- `BashTool/prompt.ts` and `bashPermissions.ts` apply high-pressure special governance to Bash, proving high-risk capability must carry denser constraints

As portable engineering principles: model proposes, runtime authorizes; preserve causal order even under parallelism; interrupt is first-class, not a generic exception; treat Bash-class tools as explicit exceptions; a tool system protects both users and runtime.

Next chapter addresses another common illusion in this system: "more context is always better." Claude Code implementation says the opposite. Experienced systems treat context as a resource, not a warehouse. We now turn to how memory, `CLAUDE.md`, and compact form one context-governance regime.
