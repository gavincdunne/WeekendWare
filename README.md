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
└── architect-agent/
    └── SKILL.md          ← Architect agent instructions
```

This repo tracks the team — the skill files and evals for each agent role. Project work, specs, and briefings live separately, scoped to each project.

## The PM agent

The PM agent takes a raw idea and turns it into a buildable spec. It:

1. Reads the project context before asking a single question
2. Runs a focused interview — 2–3 questions at a time, not a form
3. Confirms its understanding before writing anything
4. Produces a structured spec: problem statement, user stories, acceptance criteria, out-of-scope items, dependencies, risks

The evals in `pm-agent/evals/evals.json` define what good output looks like for real scenarios. They're used to test whether the skill is behaving correctly before it touches live product work.

## The Architect agent

The Architect agent takes an approved spec and produces a Technical Design Document (TDD) before any code is written. It:

1. Reads all project engineering docs and the existing codebase
2. Interrogates the spec for gaps, conflicts, and open decisions
3. Confirms blocking decisions with Gavin before designing
4. Produces a TDD covering: data model, API design, client architecture, implementation sequence, and test strategy

No implementation begins until there is an approved TDD.

## Using a skill

Skills are written for [Claude Code](https://claude.ai/code). To use an agent skill on your own project, copy the relevant `SKILL.md` into your working directory and invoke it by name in your Claude Code session.

---

Built by [Gavin Dunne](https://github.com/gavincdunne)
