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

## Usage

Skills are written for [Claude Code](https://claude.ai/code). To use one, open a session in your project directory and invoke the skill by name — Claude Code will load it as context for the conversation.

---

Built by [Gavin Dunne](https://github.com/gavincdunne)
