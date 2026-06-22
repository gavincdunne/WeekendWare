# WeekendWare

A software development shop run by one person. Claude agents fill the rest of the team.

## How it works

WeekendWare runs on a simple premise: instead of hiring a team, you hire agents — each one scoped to a specific role in the development lifecycle. Agents are brought in one at a time, in order. No agent starts work until the one before it finishes.

The current hiring order:

```
PM → Architect → Design → Backend Builder → Frontend Builder → QA → DevOps → Copywriter
```

Each agent has a `SKILL.md` that defines its role, behaviour, and constraints. Skills are not prompts — they're job descriptions. They tell the agent who it is, what it's responsible for, and what it is not allowed to skip.

## The rule

**No spec, no build.**

Every feature starts with the PM agent. It interviews the founder, synthesises the answers, and produces a structured spec. Nothing gets designed or built until there's an approved spec. This single rule eliminates the most common failure mode in solo development: building the wrong thing.

## What's in this repo

```
weekendware-team/
├── pm-agent/
│   ├── SKILL.md          ← PM agent instructions
│   └── evals/
│       └── evals.json    ← test cases for the PM skill
├── architect-agent/
│   ├── SKILL.md          ← Architect agent instructions
│   └── evals/
│       └── evals.json    ← test cases for the Architect skill
└── design-agent/
    └── SKILL.md          ← Design agent instructions
```

This repo tracks the team — the skill files and evals for each agent role. Project work, specs, and briefings live separately, scoped to each project.

## The pipeline

Each agent is a gate. Nothing moves to the next stage without the previous one finishing.

| Agent | Input | Output |
|---|---|---|
| PM | A raw idea from Gavin | Approved spec |
| Architect | Approved spec | Technical Design Document (TDD) |
| Design | Approved TDD | Screen-by-screen Design Spec |
| Backend Builder | Approved TDD | Working API |
| Frontend Builder | Approved Design Spec + TDD | Working UI |
| QA | Acceptance criteria from spec | Test sign-off |
| DevOps | Working build | Deployed product |
| Copywriter | Product screens + brand context | Final copy and content |

## The agents

### PM
Takes a raw idea and turns it into a buildable spec. Runs a focused interview, confirms understanding, and produces a structured document covering: problem statement, user stories, acceptance criteria, out-of-scope items, dependencies, and risks.

### Architect
Takes an approved spec and produces a Technical Design Document before any code is written. Researches platform best practices, reasons about data structures and algorithms (with Big O analysis), thinks about scale and scaffolding, and designs the full technical approach — data model, API contract, and component breakdown.

### Design
Takes an approved TDD and produces a screen-by-screen Design Spec before any frontend work begins. Inventories every screen and state (including empty, loading, and error states), follows platform design conventions (Material Design, HIG), and produces specs detailed enough for the Frontend Builder to implement without making visual or interaction decisions themselves.

## Using a skill

Skills are written for [Claude Code](https://claude.ai/code). To use an agent skill on your own project, copy the relevant `SKILL.md` into your working directory and invoke it by name in your Claude Code session.

---

Built by [Gavin Dunne](https://github.com/gavincdunne)
