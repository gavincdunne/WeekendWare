# WeekendWare

A software development shop run by one person. Claude agents fill the rest of the team.

## How it works

Instead of hiring a team, you hire agents — each one scoped to a specific role in the development lifecycle. Each agent has a `SKILL.md` that defines its role, behaviour, and constraints. Skills are not prompts — they're job descriptions. They tell the agent who it is, what it's responsible for, and what it is not allowed to skip.

## The rules

**No spec, no build. No tests, no code.**

Every feature starts with PM. Nothing gets designed or built until there's an approved spec. QA writes the full test suite from the spec and TDD before any code is written. Builders implement to make the tests pass.

## The team

```
Gavin (COO)
│
├── PM
├── Architect
│   ├── Security
│   ├── QA
│   └── DevOps
├── Head of Design
│   ├── UX
│   └── UI
├── Backend
├── Frontend
└── Copywriter
```

## The pipeline

```
1. PM         →  spec
2. Security   →  spec review (auth, rate limits, cost)
3. Architect  →  TDD
4. Design     →  wireframes (UX) → visuals (UI)
5. Copywriter →  strings  ┐
6. QA         →  test suite┘ (parallel, before build)
7. Backend    ┐
   Frontend   ┘  implement to pass tests
8. QA         →  validate
9. DevOps     →  ship
```

## The agents

### PM
Takes a raw idea and turns it into a buildable spec. Runs a focused interview, confirms understanding, and produces a structured document covering: problem statement, user stories, acceptance criteria, out-of-scope items, dependencies, and risks. Always asks whether to start the build or save the spec for later.

### Architect
Gavin's single point of contact for all technical decisions. Manages Security, QA, and DevOps internally — Gavin never deals with them directly. Produces a Technical Design Document (TDD) from every approved spec: researches platform best practices, reasons about data structures and algorithms (with Big O analysis), thinks about scale and scaffolding, and designs the full technical approach.

### Security _(reports to Architect)_
Reviews every spec before the TDD is written, and every implementation before it ships. Covers: authentication gates, rate limiting, paywall enforcement at the API layer, AI cost exposure, abuse vectors, and secrets management. Delivers named requirements — not prose concerns.

### QA _(reports to Architect)_
Writes the full test suite from the approved TDD before builders start — translating every acceptance criterion into concrete, reproducible test cases. Validates the implementation after builders finish. Owns the ship/no-ship decision independently.

### DevOps _(reports to Architect)_
Designs and maintains CI/CD pipelines, deployment infrastructure, environment strategy, and monitoring. Every project has three long-lived branches: `main`, `staging`, `develop`. Feature branches are named `ddmmyy-feature-name`. Nothing ships to production without a working pipeline, monitoring, and a documented rollback procedure.

### Head of Design
Gavin's single point of contact for all design decisions. Manages UX and UI internally. Establishes platform design conventions in conversation with Gavin before any UI work begins. Maintains the WeekendWare brand document, which governs all design and copy across every product.

### UX Designer _(reports to Head of Design)_
Produces user flows and wireframes from an approved spec. Defines the structural skeleton of every screen and state before any visual design begins. Wireframes live in Figma where connected; structured markdown otherwise.

### UI Designer _(reports to Head of Design)_
Produces final visual designs from approved wireframes. Follows the platform conventions and design system established by the Head of Design. Delivers annotated designs ready for the Frontend builder.

### Backend
Implements the backend from the Architect's approved TDD and QA's test suite. Implements in order: storage → service → API layer. Implementation is complete when every test in the QA suite passes.

### Frontend
Implements the frontend from the approved Design Spec, TDD, and QA test suite. Matches the Design Spec precisely — makes no design decisions. Handles all UI states and platform-specific behaviour. Never hardcodes strings.

### Copywriter
Writes all product copy — in-app strings, conversational responses, onboarding copy, error messages, and notification text. Reads the brand document before writing anything. Flags all crisis-adjacent copy for clinical review. Delivers named string entries ready for `strings.xml`.

## What's in this repo

```
weekendware-team/
├── pm/
│   ├── SKILL.md
│   └── evals/evals.json
├── architect/
│   ├── lead/
│   │   ├── SKILL.md
│   │   └── evals/evals.json
│   ├── security/
│   │   ├── SKILL.md
│   │   └── evals/evals.json
│   ├── qa/
│   │   ├── SKILL.md
│   │   └── evals/evals.json
│   └── devops/
│       ├── SKILL.md
│       └── evals/evals.json
├── design/
│   ├── lead/
│   │   ├── SKILL.md
│   │   └── evals/evals.json
│   ├── ux/
│   │   ├── SKILL.md
│   │   └── evals/evals.json
│   └── ui/
│       ├── SKILL.md
│       └── evals/evals.json
├── engineering/
│   ├── backend/
│   │   ├── SKILL.md
│   │   └── evals/evals.json
│   └── frontend/
│       ├── SKILL.md
│       └── evals/evals.json
└── copywriter/
    ├── SKILL.md
    └── evals/evals.json
```

Project work (specs, briefs, copy, brand guidelines) lives outside this repo in each project's local workspace.

## Using a skill

Skills are written for [Claude Code](https://claude.ai/code). To use an agent skill on your own project, copy the relevant `SKILL.md` into your working directory and invoke it by name in your Claude Code session.

---

Built by [Gavin Dunne](https://github.com/gavincdunne)
