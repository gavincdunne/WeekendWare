# WeekendWare

A set of agent skill files for running a software development pipeline with Claude Code.

Each file defines a role — what the agent owns, what it checks before starting, what it produces, and what it is not allowed to skip. The roles follow the standard development lifecycle: PM, Architect, Security, QA, DevOps, Design, and Engineering.

## Skills

```
weekendware-team/
├── pm/                    product management, spec writing
├── architect/
│   ├── lead/              technical design, TDD
│   ├── security/          auth, rate limiting, secrets, cost exposure
│   ├── qa/                test suite authorship, ship/no-ship
│   └── devops/            CI/CD, deployment, monitoring
├── design/
│   ├── lead/              design direction, platform conventions
│   ├── ux/                user flows, wireframes
│   └── ui/                visual design, Figma
├── engineering/
│   ├── backend/           API and data layer
│   └── frontend/          UI implementation
└── copywriter/            in-app strings, onboarding copy, error messages
```

## Rules

Standards are split into three layers, read in order:

1. **`rules/weekendware-rules.md`** — universal rules that apply to every WeekendWare project (git, architecture, build, QA, security, accessibility, documentation)
2. **`rules/<platform>-rules.md`** — platform-specific rules; currently `kmp-rules.md` for Kotlin Multiplatform. Future: `android-rules.md`, `ios-rules.md`, `web-rules.md`

A third layer of project-specific rules extends these for each active project, but lives outside this repo.

Agents determine the platform first, then read universal → platform → project rules in that order.

## Usage

Skills are written for [Claude Code](https://claude.ai/code). To use one, open a session in your project directory and invoke the skill by name — Claude Code will load it as context for the conversation.

---

Built by [Gavin Dunne](https://github.com/gavincdunne)
