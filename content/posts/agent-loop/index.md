+++
title = 'The Agent Loop: How AI Agents Actually Make Progress'
date = '2026-07-24T16:20:00+05:30'
draft = false
description = 'A deep dive into the agent loop - reason, act, observe, repeat - including trajectories, stop conditions, failure modes, and how coding agents use it in practice.'
tags = ['AI', 'Agents', 'Agent Loop', 'Tool Calling', 'ReAct', 'Engineering']
categories = ['AI', 'Engineering']
summary = 'An agent is mostly a loop: the model proposes the next action, the runtime executes it, observations come back, and the cycle continues until a stop condition fires.'
+++

<img src="/images/posts/agent-loop/cover.svg" alt="Reason, act, observe, repeat agent loop" width="960" height="400" />

*A chatbot answers once. An agent keeps circling: decide, do, look, decide again.*

If you already read the [AI agents overview](/posts/ai-agents/), you met the loop in outline form. This post zooms all the way in.

The **agent loop** is the control system. Models provide judgment. Tools provide reach. The loop is what turns those into *progress over time*.

---

## The shortest useful definition

```text
while not done:
  decide next action from current context
  if action is "final answer": stop
  execute the tool
  append the observation to context
```

That is almost the whole architecture. Planners, multi-agent graphs, memory stores, and fancy frameworks are elaborations of this loop - not replacements for it.

---

## Why a loop exists at all

An LLM call is a **single shot**: tokens in, tokens out. Real work is not.

Fixing a CI failure needs several moves: read the log, open a file, patch it, run tests, maybe patch again. Research needs search, open, compare, cite. Booking a trip needs check availability, compare options, confirm.

A loop gives the system:

1. **Feedback** - actions change the world; observations bring that change back  
2. **Horizon** - hard problems become a sequence of easier ones  
3. **Recovery** - a wrong step can be corrected by the next step  

Without a loop you have autocomplete. With a loop you have a workflow.

---

## Anatomy of one iteration

<img src="/images/posts/agent-loop/one-iteration.svg" alt="Context in, LLM decision, runtime execution" width="960" height="520" />

### 1. Assemble context

The runtime builds a prompt from:

- system instructions / rules  
- the user goal  
- tool schemas (what actions exist)  
- the trajectory so far (prior actions + observations)  
- optional retrieval (docs, code chunks, memories)  

This is where [context windows](/posts/context-windows/) bite: every extra step burns budget.

### 2. Model reasons / decides

The model does **not** run tools. It emits one of:

- one or more **structured tool calls**  
- a **final answer** / stop signal  
- sometimes a clarifying question to the user  

In modern APIs this looks like function calling. In classic ReAct it looked like free-form:

```text
Thought: The stack trace points at jwt.ts.
Action: read_file
Action Input: src/auth/jwt.ts
```

Same idea. Different packaging.

### 3. Runtime acts

Your code (or Cursor's / ChatGPT's tool layer):

1. Parses the call  
2. Validates arguments and policy  
3. Executes the tool  
4. Captures stdout, return value, or error  

This boundary matters. If the model "hallucinates" a tool result in text, that is a bug in the design. Observations should come from the runtime.

### 4. Observe and append

The tool result is written back into the transcript as an **observation**. Next iteration, the model sees it.

Then the loop checks stop conditions and either continues or exits.

---

## Trajectory: the loop's working memory

<img src="/images/posts/agent-loop/trajectory.svg" alt="Growing trajectory across agent steps" width="960" height="440" />

A **trajectory** is the full sequence:

```text
goal
  -> step1 (action, observation)
  -> step2 (action, observation)
  -> ...
  -> final answer
```

The model does not secretly remember step 1 while doing step 7. The product **re-sends** the growing transcript (or a compacted version of it) on every turn.

That has three consequences:

1. **Debuggability** - you can replay what happened  
2. **Cost** - long runs get expensive  
3. **Fragility** - one bad observation can poison later decisions  

When people say an agent "got stuck," they often mean the trajectory entered a bad basin: repeated searches, the same failing command, or a wrong assumption never challenged.

---

## Stop conditions (the part beginners skip)

<img src="/images/posts/agent-loop/stop-conditions.svg" alt="Success, budget, human, and hard-fail stop conditions" width="960" height="480" />

A loop without exits is a money printer pointed at your API bill.

Common exits:

| Exit | Meaning | Example |
|------|---------|---------|
| **Success** | Done-criteria satisfied | Tests pass; ticket closed |
| **Budget** | Step / token / time / cost cap | Max 40 tool calls |
| **Human-in-the-loop** | Needs approval or clarification | Deploy to production |
| **Hard fail** | Policy block or unrecoverable error | Disallowed path; auth expired |

Important nuance: **do not trust the model saying "done."** Prefer an external check.

- Coding agents: run the test suite / typecheck  
- Research agents: require citations that resolve  
- Ops agents: verify the resource state via API  

The done-check is part of the loop design, not a polite afterthought.

---

## Loop shapes you will see in the wild

### ReAct (Reason + Act)

Interleave thoughts with actions:

`Thought -> Action -> Observation -> Thought -> ...`

Great for teaching and debugging. Verbose. Modern systems often keep the structure but hide the "Thought" text behind tool-calling APIs.

### Pure tool-calling loop

No free-form action syntax. The model returns JSON-like tool calls. Cleaner for products. Still the same loop underneath.

### Plan then execute

First produce a plan, then walk the steps. Helps on long tasks. Dangerous if the plan is wrong and never revised - better systems **re-plan** when observations disagree with the plan.

### Reflect / critique loops

After an answer, force a critique pass ("what could be wrong?") then revise. Trades latency and cost for quality.

### Multi-agent loops

Several specialized loops (researcher, coder, reviewer) coordinated by an orchestrator. Still loops. Just nested or parallel ones with a message bus.

---

## A concrete walkthrough

Goal: *Failing CI on auth middleware.*

| Step | Decision | Action | Observation |
|------|----------|--------|-------------|
| 1 | Need the error | `read_ci_log()` | `TypeError` in `jwt.ts:42` |
| 2 | Inspect code | `read_file("src/auth/jwt.ts")` | `userId` can be null |
| 3 | Fix + verify | `apply_patch(...)` then `run_tests()` | 12/12 passed |
| 4 | Stop | final answer | "Fixed null check; CI green" |

Notice what made this work:

- observations were real tool outputs  
- verification was a tool, not vibes  
- the loop stopped because tests passed, not because the model felt finished  

---

## Pseudocode you can actually implement

```python
def run_agent(goal, tools, limits):
    transcript = [system_prompt, goal, tool_schemas(tools)]
    steps = 0

    while steps < limits.max_steps:
        steps += 1
        decision = llm.decide(transcript)

        if decision.type == "final":
            return decision.answer

        if decision.type == "ask_user":
            return await_human(decision.question)

        if is_duplicate(decision, transcript):
            return fail("looping on the same tool call")

        if not policy.allows(decision):
            observation = "blocked by policy"
        else:
            observation = run_tool(decision.name, decision.args)

        transcript.append(decision)
        transcript.append(observation)

        if externally_done(goal, observation):
            return success(transcript)

    return fail("budget exceeded")
```

Everything "agentic" you see in demos is a dressed-up version of this.

---

## Failure modes that live inside the loop

| Failure | What it looks like | Mitigation |
|---------|--------------------|------------|
| **Spinning** | Same tool call with same args | Duplicate detection; step cap |
| **Thrashing** | Alternates between two failing strategies | Force a different tool; escalate to human |
| **Poisoned observation** | Truncated/wrong tool output misleads later steps | Structured tool results; size limits; retries |
| **Premature stop** | Declares done without evidence | External done-checks |
| **Never stop** | Keeps "almost finishing" | Hard budgets; wall-clock timeout |
| **Context blow-up** | Trajectory too long to reason over | Compact history; summarize tool noise; see context-window post |
| **Tool hallucination** | Invents a tool name or args shape | Strict schemas; reject unknown tools |

If you only add one safeguard, add **step caps + duplicate-call detection**. They catch a surprising amount of production pain.

---

## How this shows up in Cursor and ChatGPT

### Coding agents (Cursor-style)

The loop is obvious once you look:

1. Reason over the goal + open files + rules  
2. Act: search codebase, read file, edit, run terminal  
3. Observe: file contents, diffs, command output  
4. Repeat until tests pass or you interrupt  

The IDE is not "one big prompt with the whole repo." It is a loop that **fetches** what it needs each step - which is why indexing and tools matter as much as the model.

### ChatGPT with tools

Same skeleton, different action space: browse, code interpreter, file analysis, connectors. The chat UI hides the loop; under the hood each tool round-trip is another iteration, and long threads still hit context limits.

---

## Design checklist for a healthy loop

1. **Clear goal** - what does success look like in observable terms?  
2. **Small action space** - fewer tools, better schemas, safer defaults  
3. **Typed observations** - return structured data when you can  
4. **Hard budgets** - steps, tokens, time, money  
5. **External verification** - tests, queries, checksums  
6. **Human gates** - for irreversible actions  
7. **Logging** - store the full trajectory for every run  
8. **Evals** - measure task success, not just "the model sounded confident"  

---

## Closing

Strip away the branding and most AI agents are this:

> a model that proposes actions, a runtime that executes them, and a loop that feeds results back until an exit condition fires.

Once you see that, debugging gets easier. You stop asking "why is the AI dumb?" and start asking better questions:

- What was in context at step N?  
- Was the observation truthful and complete?  
- Did we have a real done-check?  
- Did we allow infinite retries?

Related reading on this blog:

- [AI Agents Explained](/posts/ai-agents/) - building blocks and terminology  
- [Context Windows Explained](/posts/context-windows/) - why long loops get amnesia  

**Next step:** pick one manual workflow you already do (triage a failing test, draft a release note from commits). Write the goal, 3-5 tools, a done-check, and a step budget. That checklist *is* your agent loop.
