# WeekendWare

A software development shop run by one person. Claude agents fill the rest of the team.

## How it works

Instead of hiring a team, you hire agents — each one scoped to a specific role in the development lifecycle. Each agent has a `SKILL.md` that defines its role, behaviour, and constraints. Skills are not prompts — they're job descriptions. They tell the agent who it is, what it's responsible for, and what it is not allowed to skip.

## The rule

**No spec, no build.**

Every feature starts with the PM agent. It interviews the founder, synthesises the answers, and produces a structured spec. Nothing gets designed or built until there's an approved spec. This single rule eliminates the most common failure mode in solo development: building the wrong thing.

## The team

```
                              Gavin (COO)
                                   │
         ┌──────────┬──────────────┼──────────────┬──────────┬────────────┐
         │          │              │              │          │            │
        PM      Architect    Head of Design   Backend   Frontend         QA      DevOps   Copywriter
                                   │           Builder    Builder
                          ┌────────┴────────┐
                      UX Designer     UI Designer
```

Gavin deals with PM, Architect, Head of Design, Backend Builder, Frontend Builder, QA, DevOps, and Copywriter directly. UX and UI report to the Head of Design.

## The pipeline

```
PM → Architect → Head of Design → UX → UI → QA (write tests) → Backend Builder + Frontend Builder → QA (validate) → DevOps
                                                                          ↑
                                                                    Copywriter (parallel — feeds Design Spec + strings)
```

## The agents

### PM
Takes a raw idea and turns it into a buildable spec. Runs a focused interview, confirms understanding, and produces a structured document covering: problem statement, user stories, acceptance criteria, out-of-scope items, dependencies, and risks.

### Architect
Takes an approved spec and produces a Technical Design Document (TDD) before any code is written. Researches platform best practices, reasons about data structures and algorithms (with Big O analysis), thinks about scale and scaffolding, and designs the full technical approach — data model, API contract, and component breakdown.

### Head of Design
Gavin's single point of contact for all design. Manages UX and UI designers internally — Gavin never deals with them directly. Responsible for the WeekendWare brand document, which grows with every engagement and governs design decisions across all products and marketing.

### UX Designer _(reports to Head of Design)_
Produces user flows and Figma wireframes from an approved spec. Defines the structural skeleton of every screen and state before any visual design begins. All wireframes are built in Figma — greyscale, structurally precise, no visual design decisions.

### UI Designer _(reports to Head of Design)_
Produces final visual designs in Figma from approved wireframes and a TDD. Follows Material Design 3 and HIG conventions, works within the established design system, and delivers annotated Figma frames ready for the Frontend Builder.

### Backend Builder
Implements the backend from the Architect's approved TDD. Reads the project `CLAUDE.md` before writing a line of code. Implements in order: storage → service → API layer. Tests every code path. Never ships without QA sign-off.

### Frontend Builder
Implements the frontend from the approved Design Spec and TDD. Matches the Design Spec precisely — makes no design decisions. Handles all UI states, all platforms, and all platform-specific behaviour. Never hardcodes strings.

### QA
Tests every feature against the spec's acceptance criteria before anything ships. Writes test plans, executes manual and automated tests, files bug reports, and gives a clear ship/no-ship verdict. Safety features (guardrails, PHI handling) are always P0.

### DevOps
Designs and maintains CI/CD pipelines, deployment infrastructure, environment strategy, and monitoring. Nothing ships to production without a working pipeline and monitoring in place. Owns HIPAA-adjacent infrastructure posture for Basil.

### Copywriter
Writes all product copy — in-app strings, Basil's conversational responses, onboarding copy, error messages, and notification text. Reads the brand document before writing anything. Flags all crisis-adjacent copy for clinical review. Delivers named string entries, not documents.

## What's in this repo

```
weekendware-team/
├── pm/
│   ├── SKILL.md
│   └── evals/evals.json
├── architect/
│   ├── SKILL.md
│   └── evals/evals.json
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
├── backend-builder/
│   ├── SKILL.md
│   └── evals/evals.json
├── frontend-builder/
│   ├── SKILL.md
│   └── evals/evals.json
├── qa/
│   ├── SKILL.md
│   └── evals/evals.json
├── devops/
│   ├── SKILL.md
│   └── evals/evals.json
└── copywriter/
    ├── SKILL.md
    └── evals/evals.json

projects/
├── brand/
│   └── brand-guidelines.md   ← WeekendWare brand document (grows with every project)
└── basil/
    ├── CLAUDE.md             ← internal project briefing
    └── specs/
        ├── spec-checkin-system-05252026.md
        ├── spec-onboarding-05252026.md
        └── spec-persistent-memory-05252026.md
```

## Using a skill

Skills are written for [Claude Code](https://claude.ai/code). To use an agent skill on your own project, copy the relevant `SKILL.md` into your working directory and invoke it by name in your Claude Code session.

---

Built by [Gavin Dunne](https://github.com/gavincdunne)
