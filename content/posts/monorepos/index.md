+++
title = 'One Repo, Many Apps: Workspaces, Task Graphs, and Tools like Nx'
date = '2026-07-21T14:00:00+05:30'
draft = false
description = 'A practical guide to monorepos — why teams adopt them, how workspaces and task graphs work, and how tools like pnpm, Nx, Turborepo, Bazel, and Rush fit together.'
tags = ['Monorepo', 'Nx', 'Turborepo', 'pnpm', 'DevTools', 'Engineering']
categories = ['Engineering', 'Tooling']
summary = 'Monorepos are not just “one big Git repo.” They are workspaces, task graphs, caching, and ownership. Here is the background and the modern tool landscape.'
+++

<img src="/images/posts/monorepos/cover.svg" alt="Polyrepo versus monorepo comparison" width="960" height="400" />

*A monorepo puts many projects in one version-control root. The hard part is not Git — it is dependency linking, task orchestration, and CI scale.*

“Should we move to a monorepo?” is one of those engineering questions that sounds binary and is not. A monorepo can make cross-cutting changes easy and CI painful — or the opposite — depending on tooling and discipline.

This post covers the **background** (why monorepos exist), the **core ideas** (workspaces, graphs, caching, ownership), and the **tool landscape** — from package workspaces to Nx, Turborepo, Bazel, Rush, and friends.

**TL;DR:** A monorepo is useful when many packages change together and shared standards matter. Start with a workspace manager (`pnpm`/`yarn`/`npm`), then add a task runner (`Nx`/`Turborepo`/…) when builds and tests become the bottleneck.

---

## Background: Why Monorepos Showed Up

### The polyrepo default

For years, the default was **one service / one library = one repository**:

- clear ownership boundaries  
- independent versioning and release cadence  
- smaller clones and simpler CI matrices for tiny teams  

That works until product work regularly spans boundaries:

- change an API contract and two frontends  
- update a design-system button used by five apps  
- fix a shared auth bug across mobile + web  

In a polyrepo world, that becomes:

1. PR in library repo  
2. publish a version  
3. bump dependents  
4. open more PRs  
5. hope integration tests catch skew  

That coordination tax is the original monorepo pitch.

### What large companies proved

Google’s famous monorepo story (and similar approaches at Meta and others) showed that **one codebase + strong tooling** can scale to enormous engineering orgs — with proprietary build systems, code search, and ownership systems.

Most product teams do not need Google’s internals. They need a lighter version:

- one Git repo  
- many packages/apps  
- local linking without constant publishing  
- CI that only rebuilds what changed  

That lighter version is what JS/TS monorepo tools optimize for today.

### Important clarification

A monorepo is **not**:

- “no modularity”  
- “everyone can edit everything without review”  
- “one deployable artifact”  

A healthy monorepo still has packages, APIs, ownership, and release boundaries. It just colocates them under one VCS root.

---

## Polyrepo vs Monorepo (Honest Tradeoffs)

| Dimension | Polyrepo | Monorepo |
|-----------|----------|----------|
| Cross-package change | Multi-repo choreography | One atomic PR |
| Dependency versions | Explicit publishes | Workspace links (or internal versions) |
| Tooling consistency | Easy to drift | Easier to standardize |
| CI complexity | Many small pipelines | One pipeline that must be smart |
| Access control | Repo-level is simple | Needs CODEOWNERS / directory perms |
| Clone / IDE size | Smaller per repo | Can get large |
| Independence | High | Coupled unless you enforce boundaries |

**Choose monorepo when:** shared libraries + multi-app product + frequent coordinated changes.

**Stay polyrepo when:** teams/products are truly independent, release cadences diverge hard, or compliance requires strict repo isolation.

Many orgs run a hybrid: a monorepo for the product platform, polyrepos for isolated experiments or external SDKs.

---

## Anatomy of a Modern JS/TS Monorepo

<img src="/images/posts/monorepos/layers.svg" alt="VCS, workspaces, task runners, and remote CI layers" width="960" height="360" />

*Think in layers. Do not buy a “monorepo framework” before you need the upper layers.*

### Typical layout

```text
repo/
  apps/
    web/                 # Next.js app
    api/                 # Nest/Express service
    admin/
  packages/
    ui/                  # shared component library
    config-eslint/
    tsconfig/
    database/
  package.json           # workspaces root
  pnpm-workspace.yaml
  nx.json | turbo.json   # optional task runner
```

### Core concepts

**Workspace package**  
A folder with its own `package.json`, linked locally by the package manager.

**Package graph**  
Who depends on whom (`web` → `ui` → `tokens`).

**Task graph**  
What commands depend on what (`test:web` depends on `build:ui`).

**Affected set**  
Projects impacted by a Git change (directly or transitively).

**Local cache / remote cache**  
Reuse task outputs when inputs (files, env, deps) are unchanged.

**Boundary rules**  
Lint or tooling rules that prevent `api` from importing `web`, or apps from reaching into another app’s internals.

---

## Layer 1: Git Is Necessary, Not Sufficient

Putting folders in one repo gives you atomic commits. It does **not** give you:

- dependency linking  
- incremental builds  
- smart CI  

Without more tooling, people reinvent scripts like “build everything every time,” which is how monorepos get a bad reputation.

---

## Layer 2: Workspaces (pnpm / Yarn / npm)

Workspaces are the foundation.

### What they solve

- install once at the root  
- link local packages without publishing to npm  
- declare `"ui": "workspace:*"` style dependencies  

### Tool notes

| Tool | Strengths | Watch-outs |
|------|-----------|------------|
| **pnpm workspaces** | Strict isolation by default, fast, popular for monorepos | Learn `pnpm-workspace.yaml` + shamefully-hoist edge cases |
| **Yarn Berry (v2+) workspaces** | Mature workspace features, Plug’n’Play option | PnP can surprise tooling that expects `node_modules` |
| **Yarn Classic workspaces** | Still common in older repos | Legacy; prefer Berry or pnpm for new work |
| **npm workspaces** | Built-in, simple | Less dominant for large monorepos than pnpm/Yarn |

**Practical recommendation:** for a new JS/TS monorepo, **pnpm workspaces** is a strong default.

Workspaces alone are enough for small/medium repos. You add a task runner when “build/test everything” becomes expensive.

---

## Layer 3: Task Runners and Monorepo Frameworks

This is where **Nx**, **Turborepo**, and similar tools live.

<img src="/images/posts/monorepos/task-graph.svg" alt="Task graph with lint build test and caching" width="960" height="340" />

*The value is scheduling + caching + affected analysis on a task graph.*

### Nx

**What it is:** a full monorepo toolkit — project graph, task pipeline, generators, optional plugins for React/Next/Angular/Node, module boundary linting, and remote caching.

**Why teams pick it:**

- strong **project/task graph** understanding  
- **affected** commands for CI  
- plugins/generators that standardize app/package scaffolding  
- enforcement of dependency constraints  
- local + remote computation caching  

**Tradeoffs:**

- more concepts and config than “just Turbo”  
- best ROI when you adopt its conventions, not only its cache  

Nx fits well when the monorepo is a long-term platform with many package types and you want guardrails, not only faster tasks.

### Turborepo (Vercel)

**What it is:** a fast task runner focused on pipelines and caching (local/remote), sitting on top of your existing workspaces.

**Why teams pick it:**

- low conceptual overhead  
- excellent “make CI faster” payoff  
- works with your current package scripts  
- remote cache pairs naturally with Vercel ecosystems (but is not limited to that)  

**Tradeoffs:**

- less opinionated about generators, architecture rules, and IDE graph UX than Nx  
- you still design package boundaries yourself  

Turborepo fits well when you already have a workspace monorepo and need speed without a full framework migration.

### Lage (Microsoft)

Pipeline/task orchestration with caching, used in some large Microsoft JS repos. Less mindshare than Nx/Turbo for greenfield startups, still relevant in enterprise JS shops.

### Lerna (historical + modernized)

Lerna popularized JS monorepos (especially publishing many packages). Today, **workspaces** handle linking; modern Lerna often pairs with Nx for task running/publishing workflows. If you see older tutorials saying “use Lerna for everything,” treat them as historical.

### Rush (Microsoft)

Opinionated enterprise monorepo manager: strict install model, phased builds, version policies, focused on very large multi-package orgs. Heavier than Turbo; more specialized than “add a cache.”

### Moon

Newer monorepo task runner with language-agnostic ambitions and strong task graph/caching story. Worth evaluating if you want an alternative to Nx/Turbo.

### Bazel (and Buck2)

**What they are:** general-purpose hermetic build systems (Google’s Bazel being the most widely discussed). Excellent for multi-language monorepos and reproducibility.

**Tradeoffs for typical product JS teams:**

- steep learning curve  
- rules/toolchains need investment  
- huge payoff at extreme scale or polyglot builds  

Use Bazel when hermetic, multi-language builds are the real requirement — not because “monorepo” was mentioned in a meeting.

### Pants / Please / other polyglot systems

Same category as Bazel-like builds: powerful, specialized, usually overkill for a Next.js + Node API shop.

---

## Comparison Cheat Sheet

| Tool | Primary job | Best for |
|------|-------------|----------|
| **pnpm/Yarn/npm workspaces** | Install + link packages | Every JS monorepo foundation |
| **Turborepo** | Task pipeline + cache | Fast CI on existing workspaces |
| **Nx** | Graph + tasks + generators + boundaries | Platform monorepos with structure |
| **Lage** | Task orchestration + cache | Large JS repos, MS-style pipelines |
| **Rush** | Enterprise monorepo governance | Strict versioning/install policies |
| **Lerna** | Multi-package publish workflows | Legacy repos / publish UX (often with Nx) |
| **Moon** | Task runner alternative | Teams evaluating beyond Turbo/Nx |
| **Bazel** | Hermetic polyglot build | Very large / multi-language orgs |

A useful mental model:

> Workspaces answer “how do packages see each other?”  
> Nx/Turbo/Bazel answer “how do we build/test efficiently and safely?”

---

## CI: Where Monorepos Win or Lose

### Naive CI (painful)

On every PR:

- install all deps  
- build all packages  
- test all apps  

This is correct and expensive.

### Smart CI (the point of task tooling)

On every PR:

1. determine **affected** projects from the git diff  
2. run only required tasks in graph order  
3. reuse **remote cache** artifacts from main/CI peers  

Example outcomes:

- change README only → almost no build work  
- change `packages/ui` → rebuild UI + dependent apps  
- change `apps/api` only → skip web/ui tests if graph says so  

If your monorepo CI still “builds the world,” you have a monorepo shape without monorepo economics.

---

## Ownership, Boundaries, and Politics

Tooling does not remove org design.

### CODEOWNERS

Map directories to teams so `packages/billing-ui` cannot merge without billing review.

### Dependency constraints

Prevent architectural spaghetti:

- apps may depend on packages  
- packages may not depend on apps  
- `feature-a` cannot import `feature-b` internals  

Nx has first-class support for this style of rule; in other setups you enforce with ESLint `import/no-restricted-paths` (or similar) and review culture.

### Versioning strategies

Common options inside monorepos:

1. **No publish** — apps consume workspace packages directly  
2. **Fixed/locked versions** — all packages share one version  
3. **Independent versioning** — each package versions separately (Changesets, Nx release, Lerna independent)  
4. **Private packages + selective publish** — only public SDKs leave the repo  

Pick based on whether packages are internal implementation details or external products.

---

## Migration Playbook (Practical)

If you are moving from polyrepos:

1. **Start with the shared library + 1–2 apps** that change together  
2. Establish **pnpm workspaces** and green local `build/test`  
3. Add **Turborepo or Nx** when CI time hurts  
4. Turn on **affected + remote cache** before adding more apps  
5. Add **boundary lint rules** once people start shortcuts  
6. Only then absorb more repos  

Big-bang “merge 40 repos this quarter” is how monorepo migrations fail.

---

## Common Failure Modes

| Failure | Symptom | Fix |
|---------|---------|-----|
| Fake monorepo | One repo, no workspaces | Add workspace links |
| World-building CI | 40-minute PR checks | Affected + cache |
| No boundaries | Circular deps everywhere | Lint rules + ownership |
| Over-tooling | Bazel for a 3-package repo | Start with pnpm (+ Turbo/Nx) |
| Under-tooling | Manual bash build scripts | Task graph runner |
| Giant root lockfile fights | Constant install churn | Stricter package manager + fewer root deps |

---

## What I Recommend by Stage

**Stage A — 2–5 packages, small team**  
`pnpm workspaces` + simple CI.

**Stage B — multiple apps, CI slowing down**  
Add **Turborepo** or **Nx** for pipelines/caching/affected.

**Stage C — many teams, need architecture enforcement**  
Lean **Nx** (or Rush in enterprise-heavy contexts).

**Stage D — polyglot + hermetic builds at huge scale**  
Evaluate **Bazel** (with dedicated build-platform ownership).

There is no single winner — only fit for scale and constraints.

---

## Conclusion

A monorepo is best understood as a **stack**:

> Git colocation → workspaces → task graph/caching → remote CI → ownership rules

Tools like **Nx** and **Turborepo** sit in the middle of that stack. They do not replace package managers, and they are not mandatory on day one. They become valuable when coordinated code changes are frequent and “build everything” stops being acceptable.

If you remember one decision framework:

1. Do packages change together?  
2. Can workspaces remove publish churn?  
3. Is CI time now the bottleneck?  
4. Do you need generators/boundaries, or only speed?

Answer those in order, and the tool choice (Nx vs Turbo vs Bazel vs workspaces-only) usually becomes obvious.

**Next steps:** sketch your package graph on paper, enable pnpm workspaces for one shared UI package, then add a task runner only after you measure CI pain.
