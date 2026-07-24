+++
title = 'On-Demand Playbooks: Teaching Agents Specialized Workflows'
date = '2026-07-23T10:00:00+05:30'
draft = false
description = 'What agent skills are, how they differ from rules and tools, how SKILL.md works in Cursor, and concrete skill ideas mapped to LLMs, agents, context windows, tree shaking, and monorepos.'
tags = ['AI', 'Agents', 'Agent Skills', 'Cursor', 'Engineering', 'Productivity']
categories = ['AI', 'Engineering']
summary = 'Agent skills are reusable playbooks - not always-on rules and not raw tools. Here is how they work, how to author them, and skill ideas for every topic on this blog.'
+++

<img src="/images/posts/agent-skills/cover.svg" alt="Agent skills as packaged playbooks loaded on demand" width="960" height="400" />

*Rules tell the agent how to behave in general. Tools let it act. Skills teach it how to do a specific job well - only when that job shows up.*

If you have used a coding agent and found yourself pasting the same checklist ("audit tree shaking like this", "design the agent loop with these stop conditions"), you already want **agent skills**.

This post explains:

1. What an agent skill is  
2. How it differs from rules, tools, MCP, and prompts  
3. The anatomy of a skill (`SKILL.md`)  
4. How to write ones that actually get used  
5. **Concrete skill ideas for every post on this blog**

---

## What is an agent skill?

An **agent skill** is a packaged set of instructions (and optionally scripts, templates, and reference docs) that teaches an agent how to perform a **specialized workflow**.

Think of it as a playbook:

| Piece | Role |
|-------|------|
| **Name** | Stable id (`bundle-export-audit`) |
| **Description** | What it does + when to use it (discovery) |
| **Instructions** | Step-by-step procedure the agent should follow |
| **Extras** | Examples, reference docs, helper scripts |

In Cursor, a skill lives in a folder with a required `SKILL.md`:

```text
bundle-export-audit/
  SKILL.md          # required
  reference.md      # optional deep docs
  examples.md       # optional examples
  scripts/          # optional helpers
```

Personal skills live under `~/.cursor/skills/`. Project skills live under `.cursor/skills/` and travel with the repo.

The important mental model:

> Skills are **procedural knowledge** the agent can load **on demand**, instead of stuffing every niche workflow into the always-on system prompt.

That ties directly to the [context window](/posts/context-windows/) problem: you cannot afford to preload every checklist on every turn.

---

## Skills vs rules vs tools (and friends)

<img src="/images/posts/agent-skills/skills-vs-rules-tools.svg" alt="Comparison of rules, skills, and tools" width="960" height="460" />

| Concept | Job | Loaded when | Example |
|---------|-----|-------------|---------|
| **Rules** | Persistent preferences / standards | Often / always (or by file glob) | "Use TypeScript strict; no default exports" |
| **Skills** | Specialized workflows | When the task matches the description | "Audit a component library for tree shaking" |
| **Tools** | Executable actions | When the agent decides to act | `read_file`, shell, browser, DB query |
| **MCP / APIs** | External systems exposed as tools | Same as tools | Figma, Slack, Datadog |
| **Memory** | Durable facts about you / project | Injected selectively | "Prefers pnpm workspaces" |
| **One-off prompt** | Ad-hoc instructions for this chat | This turn only | "Refactor this function" |

### A useful decision test

- If it should apply **almost every time** in this repo -> **rule**  
- If it is a **repeatable multi-step procedure** for a specific kind of ask -> **skill**  
- If it needs to **touch the world** (files, network, UI) -> **tool**  
- If it is a **fact** ("we deploy Fridays") -> **memory** or docs, not a skill  

Skills often *use* tools. They do not replace them.

Skills also sit on top of the [agent loop](/posts/agent-loop/): the skill shapes *how* the agent reasons and which steps to take; the loop still reason -> act -> observe until done.

---

## Anatomy of `SKILL.md`

<img src="/images/posts/agent-skills/anatomy.svg" alt="Skill folder structure and progressive disclosure" width="960" height="420" />

Every skill starts with YAML frontmatter plus markdown instructions:

```markdown
---
name: bundle-export-audit
description: >-
  Audit JS/TS component libraries for tree-shaking readiness (ESM entrypoints,
  sideEffects, barrel files). Use when packaging libraries, debugging bundle
  size, or reviewing export graphs.
---

# Bundle export audit

## Workflow
1. Locate package entrypoints (`exports`, `module`, `main`).
2. Check `sideEffects` in package.json.
3. Inspect barrel files for eager re-exports.
4. Verify with a bundler report...
```

### Metadata that matters

| Field | Why it matters |
|-------|----------------|
| `name` | Stable id; lowercase + hyphens |
| `description` | **Discovery**. The agent uses this to decide whether the skill applies |
| `disable-model-invocation` | If true, only load when the user explicitly names the skill |

### Progressive disclosure

Do **not** dump a textbook into `SKILL.md`.

1. Keep `SKILL.md` short (aim under ~500 lines; shorter is better)  
2. Put deep reference material in `reference.md`  
3. Link it; let the agent open it only when needed  

This is context-window hygiene as a design pattern.

---

## How discovery works

Roughly:

1. The agent sees skill **descriptions** (cheap metadata).  
2. Your request comes in: "why is `Button` still in the production bundle?"  
3. Description match -> agent reads `SKILL.md`.  
4. It follows the workflow, calling tools as needed.  
5. If stuck, it may open `reference.md` or run a script.

So the description must include **what** and **when**:

- Good: "Audit component libraries for tree shaking. Use when debugging bundle size, reviewing `sideEffects`, or packaging ESM libraries."  
- Bad: "Helps with frontend."  

Write descriptions in third person. They are injected as capability text, not as a chat persona.

---

## Authoring principles that separate useful skills from junk

### 1. Assume the model is already smart

Do not explain what a monorepo is. Encode *your* team's procedure: which command, which folder layout, which failure modes.

### 2. Prefer checklists and templates over essays

Agents follow procedures better than philosophy.

### 3. Match freedom to fragility

| Fragility | Skill style |
|-----------|-------------|
| High (migrations, releases) | Exact commands / scripts |
| Medium (PR reviews) | Templates + must-check items |
| Low (brainstorming) | Light guidance; more freedom |

### 4. One skill, one job

"Do everything frontend" becomes a graveyard. Split: tree-shake audit, Storybook scaffold, visual regression triage.

### 5. Put project skills in the repo

If the workflow is team-specific, `.cursor/skills/` beats a private note in someone's head.

---

## Skill ideas for every post on this blog

<img src="/images/posts/agent-skills/use-cases.svg" alt="Skill ideas mapped to blog topics" width="960" height="520" />

Below is a practical catalog. Each idea names the skill, when it should trigger, and what the playbook should contain - grounded in topics already covered here.

### 1. From [How LLMs Work / run locally](/posts/understanding-llms/)

**Skill: `local-llm-ops`**

- **When:** user mentions Ollama, LM Studio, GGUF, local models, VRAM, quantization  
- **What it teaches:**
  - pick a model size for the machine (RAM/VRAM heuristic table)
  - pull / serve with Ollama vs LM Studio
  - smoke-test prompts and latency
  - common failures (context too large for local window, GPU offload, corrupt GGUF)
- **Why a skill:** the steps are boring but easy to get wrong; you do not want them in every chat's rules.

**Skill: `prompt-eval-harness`**

- **When:** comparing model answers or building a tiny eval set  
- **What it teaches:** fixed prompt suite, scoring rubric, how to log outputs without overfitting to one lucky reply.

---

### 2. From [AI Agents Explained](/posts/ai-agents/)

**Skill: `agent-design-checklist`**

- **When:** "design an agent", "add tool calling", "build a bot for X"  
- **Playbook:**
  1. Write the goal in observable terms  
  2. List minimum tools (action space)  
  3. Define done-criteria and budgets  
  4. Add human gates for irreversible actions  
  5. Sketch failure modes (looping, tool misuse)  
- **Output template:** a one-pager the team can implement.

**Skill: `tool-schema-review`**

- **When:** reviewing function/tool definitions  
- **Checks:** names, required args, examples, error shape, permission notes. Bad schemas create bad [agent loops](/posts/agent-loop/).

---

### 3. From [The Agent Loop](/posts/agent-loop/)

**Skill: `debug-agent-trajectory`**

- **When:** "agent is stuck", "keeps calling the same tool", "said done but tests fail"  
- **Playbook:**
  1. Dump / inspect the trajectory  
  2. Flag duplicate tool calls  
  3. Check whether observations were truncated or wrong  
  4. Verify stop conditions (budget vs success vs human)  
  5. Propose a concrete fix (cap, done-check, schema change)
- **Optional script:** parse a JSONL run log and print repeated `(tool, args)` pairs.

**Skill: `agent-runbook-from-workflow`**

- **When:** turning a manual human workflow into an agent  
- **Output:** goal, tools, loop exits, eval cases - the loop design checklist as a skill.

---

### 4. From [Context Windows Explained](/posts/context-windows/)

**Skill: `context-budget-hygiene`**

- **When:** long chats go soft, prompts feel bloated, "forgot earlier decision"  
- **Playbook:**
  1. Identify what must stay verbatim vs summarize  
  2. Produce a handoff summary for a fresh thread  
  3. Prefer retrieve-on-demand over paste-everything  
  4. Put critical constraints at edges / in rules  
- **Template:** goal, decisions, constraints, next step (the handoff block from the context post).

**Skill: `rag-chunk-review`**

- **When:** retrieval quality is bad  
- **Checks:** chunk size, overlap, missing files in ignore lists, noisy lockfiles/vendored code in the index.

This skill exists because skills themselves are a mitigation for context limits - teaching the agent to spend context wisely is recursive but useful.

---

### 5. From [Tree Shaking in Component Libraries](/posts/tree-shaking-component-libraries/)

**Skill: `bundle-export-audit`**

- **When:** library packaging, "why is unused component in the bundle", reviewing `package.json` exports  
- **Playbook:**
  1. Inspect `exports` / `module` / `main`  
  2. Confirm ESM vs CJS reality  
  3. Read `sideEffects`  
  4. Hunt barrel files that re-export everything  
  5. Build a minimal consumer app and compare bundle reports before/after  
- **Scripts:** small `esbuild`/`rollup` fixture that imports one button and fails CI if chart code appears.

**Skill: `barrel-file-refactor`**

- **When:** migrating away from deep barrels  
- **Steps:** map public API, split entrypoints, update docs/imports, verify tree shaking still holds.

---

### 6. From [Monorepos Explained](/posts/monorepos/)

**Skill: `monorepo-task-graph`**

- **When:** adding a package, slow CI, "what should Nx/Turbo run"  
- **Playbook:**
  1. Identify workspace tool (pnpm/yarn/npm)  
  2. Find task runner (Nx, Turborepo, Bazel, Rush...)  
  3. Map project graph / dependsOn  
  4. Propose `affected` command for the change  
  5. Call out cache inputs that are wrong (env files, unbounded globs)
- **Output:** exact commands for local + CI.

**Skill: `new-workspace-package`**

- **When:** scaffolding a new app/lib in the monorepo  
- **Steps:** naming, package.json, tsconfig references, project.json/turbo pipeline entry, ownership CODEOWNERS, minimal test target.

---

### Bonus: a skill for *this* blog itself

**Skill: `hugo-blog-post`**

- **When:** "write a blog post", "add diagrams for a post"  
- **Playbook tailored to this site:**
  - path: `content/posts/<slug>/index.md`
  - front matter fields you use (`title`, `date`, `tags`, `summary`...)
  - images under `static/images/posts/<slug>/` as ASCII-safe SVG (`Arial`, no fancy unicode dashes)
  - prefer `<img src="/images/...">` over fragile cover shortcodes
  - link related posts when relevant  
- **Why:** you have already paid the setup tax once; encode it so the next post stays consistent.

---

## Minimal example you can copy

```markdown
---
name: context-budget-hygiene
description: >-
  Compact long AI chats and redesign prompts for finite context windows.
  Use when conversations forget earlier decisions, prompts feel bloated,
  or the user asks for a handoff summary / context cleanup.
---

# Context budget hygiene

## Steps
1. Restate the goal in one sentence.
2. List decisions that must not be lost.
3. List constraints (stack, deadlines, "do not").
4. Drop tool noise and repeated failed attempts.
5. Emit a handoff block for a fresh thread.

## Handoff template
Goal:
Decisions:
Constraints:
Next step:
Open questions:
```

Save that as `~/.cursor/skills/context-budget-hygiene/SKILL.md` (or under `.cursor/skills/` in a repo) and you have a real skill.

---

## Common mistakes

1. **Skill that should be a rule** - "always use pnpm" belongs in rules.  
2. **Skill that should be a tool/script** - if the value is deterministic parsing, ship a script and have the skill call it.  
3. **Vague description** - skill never triggers.  
4. **Novel-length SKILL.md** - burns context and gets partially followed.  
5. **Ten skills that overlap** - agent picks the wrong one; merge or narrow triggers.  
6. **No done-criteria** - skill becomes vibes. Tie workflows to checks (bundle report, tests, `affected` dry-run).

---

## How this fits the bigger agent picture

From earlier posts on this blog:

- An [agent](/posts/ai-agents/) = model + tools + memory + orchestrator  
- The [agent loop](/posts/agent-loop/) is how progress happens  
- The [context window](/posts/context-windows/) is the budget for each turn  

**Skills** are how you install *specialist procedures* into that system without paying the full context cost on every request.

```text
User task
  -> match skill description
  -> load playbook
  -> enter agent loop (tools + observations)
  -> stop on real done-check
```

---

## Closing

Agent skills are not magic autonomy. They are **versioned playbooks**:

- discoverable via a good description  
- cheap until needed  
- precise enough to make the agent consistent  
- composable with rules (policy) and tools (action)

If you want a practical starting set from this blog's topics, create these six first:

1. `local-llm-ops`  
2. `agent-design-checklist`  
3. `debug-agent-trajectory`  
4. `context-budget-hygiene`  
5. `bundle-export-audit`  
6. `monorepo-task-graph`  

Then add `hugo-blog-post` if you write here often.

**Next step:** pick the workflow you have explained to an agent twice this week. That repetition is your first skill.
