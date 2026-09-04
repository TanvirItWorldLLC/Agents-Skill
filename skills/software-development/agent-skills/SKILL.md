---
name: agent-skills
description: "Senior engineering skills: spec, plan, build, verify, ship."
version: 1.0.0
author: Addy Osmani
license: MIT
platforms: [linux, macos, windows]
---

# Agent Skills

A collection of engineering workflow skills for senior software engineers, organized by development phase.

## Source

Repository: https://github.com/addyosmani/agent-skills
Local path: ~/agent-skills

## Skill Catalog

| Phase | Skill | Description |
|-------|-------|-------------|
| Define | interview-me | Surface what the user actually wants before any plan, spec, or code exists |
| Define | idea-refine | Refine ideas through structured divergent and convergent thinking |
| Define | spec-driven-development | Requirements and acceptance criteria before code |
| Plan | planning-and-task-breakdown | Decompose into small, verifiable tasks |
| Build | incremental-implementation | Thin vertical slices, test each before expanding |
| Build | source-driven-development | Verify against official docs before implementing |
| Build | doubt-driven-development | Adversarial fresh-context review of every non-trivial decision |
| Build | context-engineering | Right context at the right time |
| Build | frontend-ui-engineering | Production-quality UI with accessibility |
| Build | api-and-interface-design | Stable interfaces with clear contracts |
| Verify | test-driven-development | Failing test first, then make it pass |
| Verify | browser-testing-with-devtools | Chrome DevTools MCP for runtime verification |
| Verify | debugging-and-error-recovery | Reproduce → localize → fix → guard |
| Review | code-review-and-quality | Five-axis review with quality gates |
| Review | code-simplification | Preserve behavior while reducing unnecessary complexity |
| Review | security-and-hardening | OWASP prevention, input validation, least privilege |
| Review | performance-optimization | Measure first, optimize only what matters |
| Ship | git-workflow-and-versioning | Atomic commits, clean history |
| Ship | ci-cd-and-automation | Automated quality gates on every change |
| Ship | deprecation-and-migration | Remove old systems and migrate users safely |
| Ship | documentation-and-adrs | Document the why, not just the what |
| Ship | observability-and-instrumentation | Structured logs, RED metrics, traces, symptom-based alerts |
| Ship | shipping-and-launch | Pre-launch checklist, monitoring, rollback plan |

## Usage

Load individual skills as needed:

```bash
skill_view(name='agent-skills')
# Then load specific skill references from ~/agent-skills/skills/<skill-name>/SKILL.md
```

## Core Operating Behaviors

1. **Surface Assumptions** - Before implementing anything non-trivial, explicitly state your assumptions
2. **Manage Confusion Actively** - When you encounter inconsistencies, STOP, name the confusion, present tradeoffs, wait for resolution
3. **Push Back When Warranted** - Point out issues directly, explain concrete downsides, propose alternatives
4. **Enforce Simplicity** - Actively resist overcomplication; prefer boring, obvious solutions
5. **Maintain Scope Discipline** - Touch only what you're asked to touch
6. **Verify, Don't Assume** - Every skill includes a verification step; "seems right" is never sufficient

## Lifecycle Sequence

1. interview-me → Extract what the user actually wants
2. idea-refine → Refine vague ideas
3. spec-driven-development → Define what we're building
4. planning-and-task-breakdown → Break into verifiable chunks
5. context-engineering → Load the right context
6. source-driven-development → Verify against official docs
7. incremental-implementation → Build slice by slice
8. observability-and-instrumentation → Instrument as you build
9. doubt-driven-development → Cross-examine non-trivial decisions in-flight
10. test-driven-development → Prove each slice works
11. code-review-and-quality → Review before merge
12. code-simplification → Reduce unnecessary complexity while preserving behavior
13. git-workflow-and-versioning → Clean commit history
14. documentation-and-adrs → Document decisions
15. deprecation-and-migration → Retire old systems and move users safely when needed
16. shipping-and-launch → Deploy safely

## Quick Access

- Skills directory: ~/agent-skills/skills/
- Each skill has its own SKILL.md with detailed workflow
- Supporting files in references/, scripts/, templates/ as needed