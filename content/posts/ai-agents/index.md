+++
title = 'AI Agents Explained: How They Work and the Terminology That Matters'
date = '2026-07-22T22:50:00+05:30'
draft = false
description = 'A detailed guide to AI agents — how the plan-act-observe loop works, core building blocks, and the terminology engineers actually need.'
tags = ['AI', 'Agents', 'LLM', 'Tool Calling', 'Engineering']
categories = ['AI', 'Engineering']
summary = 'AI agents are LLMs plus tools, memory, and a control loop. Learn how they work and the key terms behind modern agent systems.'
+++

<img src="/images/posts/ai-agents/cover.svg" alt="Chatbot vs AI agent comparison" width="960" height="400" />

*A chatbot answers once. An agent keeps acting until the goal is done — or until it hits a limit.*

Chatbots complete a prompt. **AI agents** pursue a goal.

That difference sounds small. In practice, it changes everything: the system can search the web, run commands, call APIs, edit files, ask clarifying questions, and decide what to do next. This is why products like coding agents, research agents, and customer-support bots feel closer to "software that works" than "software that talks."

This post explains how agents work under the hood, then builds a vocabulary you can use when designing, evaluating, or debugging them.

**TL;DR:** An AI agent is usually an LLM wrapped in a loop that can call **tools**, store **state/memory**, and stop when a **goal**, **budget**, or **policy** says so. The hard part is not the prompt — it is control, reliability, and safety.

---

## Chatbot vs Assistant vs Agent

These words get used interchangeably. They shouldn't.

| Term | What it does | Typical shape |
|------|--------------|---------------|
| **Chatbot** | Answers in natural language | Prompt in, text out |
| **Assistant** | Helps with tasks, often with light tools | Chat + optional tool calls |
| **Agent** | Works toward a goal over multiple steps | Plan → act → observe → repeat |

A useful test:

> If the system can take multiple actions over time, use intermediate results, and decide when it is finished — you are in agent territory.

<img src="/images/posts/ai-agents/building-blocks.svg" alt="Model, tools, memory, and orchestrator building blocks" width="960" height="380" />

*Four building blocks: model, tools, memory, orchestrator.*

---

## The Core Loop: How Agents Actually Work

Most production agents share the same control loop:

1. **Perceive** — read the user goal, current context, and available tools  
2. **Reason** — decide the next action (or decide to stop)  
3. **Act** — call a tool / API / function  
4. **Observe** — put the tool result back into context  
5. **Repeat** until done, failed, or budget exceeded  

<img src="/images/posts/ai-agents/agent-loop.svg" alt="Perceive reason act observe agent loop" width="960" height="420" />

*The agent loop is the heart of the system.*

### What the LLM is doing

The model is still a next-token predictor. Agent frameworks do not give the model "true autonomy." They give it:

- a **system prompt** that defines role and rules  
- a **tool schema** describing what actions exist  
- a **transcript** of previous thoughts, actions, and observations  
- a **runtime** that executes tool calls and returns results  

The model proposes actions. The runtime makes them real.

### A tiny example

Goal: "Summarize today's top AI news for backend engineers."

Possible loop:

1. Thought: I need recent sources.  
2. Action: `web_search("AI news backend infrastructure")`  
3. Observation: list of articles  
4. Action: `fetch_url(article_1)`  
5. Observation: article text  
6. Action: `fetch_url(article_2)`  
7. Final answer: structured summary  

That multi-step behavior is the agent.

---

## Anatomy of an Agent System

### 1. Model (the policy)

The LLM chooses what to do next. Stronger models usually plan better, follow tool schemas more reliably, and fail less often — but they are still probabilistic.

Important properties:

- **Instruction following** — does it respect constraints?  
- **Tool-calling accuracy** — does it emit valid arguments?  
- **Long-context quality** — can it use earlier observations?  
- **Latency / cost** — agents multiply tokens because they loop  

### 2. Tools (the hands)

Tools are functions the agent can call. Examples:

- search / browse  
- database query  
- HTTP request  
- shell command  
- code execution  
- file read/write  
- calendar / email / ticket APIs  

In modern APIs this is often called **function calling** or **tool use**: you register JSON schemas, the model returns a structured call, your code runs it, then you send the result back.

### 3. Memory (the notebook)

Without memory, every step is amnesia with a growing chat log.

Common layers:

| Memory type | What it stores | Example |
|-------------|----------------|---------|
| **Working memory** | Current transcript / scratchpad | Last 20 tool results |
| **Episodic memory** | Past runs / sessions | "Last deploy failed on migration" |
| **Semantic memory** | Facts / docs via retrieval | RAG over your wiki |
| **Entity memory** | Structured records | User prefs, project metadata |

### 4. Orchestrator (the runtime)

This is the non-model software around the loop:

- parse tool calls  
- execute tools with permissions  
- enforce max steps / timeouts  
- handle retries and errors  
- decide when to ask a human  
- log traces for debugging  

If the model is the brain, the orchestrator is the operating system.

---

## Important Patterns

### ReAct (Reason + Act)

ReAct interleaves reasoning traces with actions:

`Thought → Action → Observation → Thought → ... → Final Answer`

<img src="/images/posts/ai-agents/react-loop.svg" alt="ReAct thought action observation loop" width="960" height="360" />

*ReAct made the agent loop explicit. Tool-calling APIs made it product-ready.*

### Tool calling / function calling

Instead of free-form "Action: search[...]" text, the model emits structured calls:

```json
{
  "name": "get_weather",
  "arguments": { "city": "Bengaluru" }
}
```

This is more reliable for production systems.

### Planning then execution

Some systems first create a plan ("1. gather context 2. draft 3. verify"), then execute steps. Useful for longer tasks; risky if the plan is wrong and never revised.

### Reflection / self-critique

After producing an answer or action, the agent critiques itself and revises. Helps quality; increases cost and latency.

### Multi-agent systems

Multiple specialized agents collaborate:

- researcher  
- coder  
- reviewer  
- manager / router  

This can improve modularity, but also adds coordination failure modes.

---

## Terminology Cheat Sheet

Use this as a practical glossary.

### Goal / objective
The outcome the agent should achieve ("open a PR that fixes flaky tests").

### Task / subtask
A smaller unit of work inside the goal.

### Environment
Everything the agent can observe or affect: files, APIs, browsers, terminals, tickets.

### Action space
The set of allowed tools/actions. Smaller action spaces are usually safer and easier to control.

### Observation
What comes back after an action (tool output, error, page content).

### State
The agent's current internal situation: transcript, variables, scratchpad, flags.

### Policy
The strategy that maps state → next action. In LLM agents, the model + prompt act as the policy.

### Trajectory / trace
The full sequence of thoughts, actions, and observations. Essential for debugging.

### Horizon
How many steps ahead the agent can usefully plan. Long horizons are hard.

### Budget / limits
Max steps, max tokens, max dollars, max runtime. Always set these.

### Stop condition
When the loop ends: goal complete, tool returns final answer, human takeover, or limit hit.

### Hallucination
Confidently invented facts. Agents can also hallucinate **tool results** if the runtime is buggy — or invent tool names that do not exist.

### Grounding
Connecting answers to retrieved evidence or real tool outputs.

### RAG (Retrieval-Augmented Generation)
Fetch relevant documents and put them into context before generation. Often used as agent memory/search.

### Tool schema
Machine-readable description of a tool's name, args, and meaning.

### Scaffolding
The prompts, parsers, retries, and glue code around the model. Most "agent magic" is scaffolding.

### Autonomy level
How much the agent can do without human approval. From "suggest only" to "execute in production."

### Human-in-the-loop (HITL)
A human must approve sensitive actions (payments, deploys, deletes).

### Guardrails
Filters and policies that block unsafe prompts, outputs, or tool calls.

### Eval / evaluation
Tests for agent quality: success rate, cost, latency, safety, and recovery from errors.

### Deterministic vs stochastic steps
Tool calls can be deterministic; model decisions are stochastic unless temperature is zero and the stack is pinned.

---

## A Minimal Mental Model in Code

Conceptual pseudocode:

```text
context = [system_prompt, user_goal, tool_schemas]

for step in 1..max_steps:
    decision = llm(context)

    if decision.type == "final_answer":
        return decision.content

    if decision.type == "tool_call":
        result = run_tool(decision.name, decision.args)  # your runtime
        context.append(decision)
        context.append(observation(result))
        continue

    handle_invalid_decision(decision)
```

Everything sophisticated — planners, multi-agent graphs, memory stores — is an elaboration of this loop.

---

## Why Agents Fail (and What to Design For)

Agents fail differently from chatbots.

| Failure | What it looks like | Mitigation |
|---------|--------------------|------------|
| **Looping** | Repeats the same tool call | Detect duplicates, cap steps |
| **Tool misuse** | Wrong args / wrong tool | Better schemas, validation, examples |
| **Context overflow** | Forgets early constraints | Summarize, retrieve, compress |
| **Cascading errors** | One bad observation derails the run | Retries, verification tools, checkpoints |
| **Over-autonomy** | Does irreversible things | Permissions, HITL, sandboxes |
| **False success** | Says "done" when not done | Acceptance tests, graders, evals |

A robust agent is less about a clever prompt and more about **verification**: tests, typed tool outputs, and clear done-criteria.

---

## Practical Design Checklist

When you build an agent, define these explicitly:

1. **Goal** — what "done" means in measurable terms  
2. **Tools** — minimum viable action space  
3. **Permissions** — what it must never do alone  
4. **Memory** — what must persist across steps/sessions  
5. **Budgets** — steps, tokens, time, money  
6. **Observability** — traces you can replay  
7. **Evals** — golden tasks you run on every change  

If you cannot write those seven down, you do not have an agent architecture yet — you have a demo.

---

## Where Agents Shine (and Where They Don't)

**Strong fit**

- multi-step research with sources  
- code changes with tests as feedback  
- ops workflows with clear APIs  
- internal copilots over company tools  

**Weak fit**

- tasks needing perfect legal/medical certainty with no verifier  
- ultra-low-latency single responses  
- poorly specified goals ("make it better")  
- environments with no reliable feedback signal  

Agents need feedback. No feedback, no real control loop.

---

## Connection to Everyday Developer Tools

Coding agents (in IDEs or CLIs) are the same pattern:

- tools: read file, edit file, run tests, search repo  
- memory: conversation + codebase retrieval  
- loop: change code → run checks → read errors → fix  

That is why they feel powerful — and why they can also create [cognitive debt](/posts/the-cognitive-debt-of-ai-assisted-development/) if you stop inspecting their work.

---

## Conclusion

An AI agent is not a magical new model species. It is a **system design**:

> LLM + tools + memory + orchestration loop + limits

Once you see the loop, the terminology clicks into place: actions, observations, trajectories, budgets, guardrails, evals.

If you want to go deeper next, natural follow-ups are:

- building a minimal tool-calling agent from scratch  
- evaluating agents with task success metrics  
- multi-agent patterns and when they are worth the complexity  

**Next step:** Take one workflow you do manually (for example, "triage failing CI"), write the goal + tools + done-criteria, and you will already have the blueprint of an agent.
