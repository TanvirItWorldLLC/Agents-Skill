---
name: taste-learning
description: "Capture coding preferences as skills and memory."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [taste, preferences, learning, patterns, style]
    related_skills: [hermes-agent-skill-authoring, plan, test-driven-development]
---

# Taste Learning for Hermes Agent

Command Code has a "taste-1" model that continuously learns your coding style. Hermes achieves similar results through **skills**, **memory**, and **project context files**.

## How Taste Learning Works in Hermes

### 1. Skills = Learned Procedures
When you successfully complete a workflow, save it as a skill:
```bash
hermes skills install <skill-id>  # From hub
skill_manage(action='create', name='my-pattern', content='...')  # Custom
```

### 2. This Session's Taste Patterns (embedded for future sessions)

After configuring Graft + OpenMontage + agency-agents with Hermes, the following patterns were captured:

**Graft Code Graph Integration** — When working with code repos, first build the graft graph (`graft build`) then query via `graft ask` from Hermes chat. Pattern: `hermes chat --query "graft ask 'find auth logic'"`.

**OpenMontage Video Production** — For video pipeline execution, use Hermes chat with oneshot or background mode. Free-tier pipelines work without API keys; configure `.env` for premium features. Pattern: `hermes chat --query "Make a 30-second animated explainer about AI agents (free, no API keys)"`.

**agency-agents Persona Activation** — For specialized agent expertise, convert and install agency agents via `./scripts/install.sh --tool hermes`, then activate in chat. Pattern: `hermes chat --query "Build a React counter component"` (invokes Frontend Developer agent).

### 3. Memory = Learned Preferences
Save persistent facts about your preferences:
```bash
memory(action='add', target='user', content='User prefers pytest over unittest')
memory(action='add', target='user', content='User uses pnpm, not npm')
memory(action='add', target='memory', content='Project uses FastAPI with SQLModel')
```

### 3. Project Context (AGENTS.md/.hermes.md) = Team Standards
Commit project-level conventions that all agents follow.

### 4. Session History = Searchable Experience
Use `session_search` to find past solutions to similar problems.

## Capturing Taste from Interactions

After completing a task, ask yourself:
- What pattern worked well? → Save as skill
- What preference did I notice? → Save in memory
- What project convention emerged? → Update AGENTS.md
- What debugging approach worked? → Save as skill

## Taste Commands

### Save a successful pattern as a skill
```python
skill_manage(action='create', 
    name='fastapi-auth-pattern',
    content='''---
name: fastapi-auth-pattern
description: "FastAPI JWT auth with SQLModel pattern"
---
# Pattern: FastAPI JWT Auth with SQLModel
## When to use: New FastAPI projects needing auth
## Files: auth.py, models.py, main.py
## Dependencies: fastapi, python-jose, passlib, sqlmodel
...''')
```

### Save a preference
```python
memory(action='add', target='user', 
    content='User prefers explicit type hints over inference')
```

### Browse learned skills
```bash
hermes skills browse
hermes skills list
```

### Search past sessions for similar problems
```python
session_search(query="authentication jwt fastapi")
```

## Taste Persistence Across Sessions

| Storage | Scope | Persistence |
|---------|-------|-------------|
| Skills (~/.hermes/skills/) | User + Project | Forever |
| Memory (state.db) | User + Session | Forever |
| AGENTS.md | Project (git) | Forever |
| .hermes.md | Project (git) | Forever |
| Session DB | Session | Forever |

## Mimicking Command Code's Taste Features

| Command Code Feature | Hermes Equivalent |
|---------------------|-------------------|
| taste-1 model | Skills + Memory + Context files |
| `.commandcode/taste/taste.md` | `~/.hermes/skills/` + `memory` tool |
| `/taste` panel | `hermes skills browse` + `memory` tool |
| `/learn-taste` | `skill_manage(create)` + `memory(add)` |
| `/import claude` | `hermes skills install` from hub |
| Taste sharing (push/pull) | Git commit AGENTS.md + share skills |

## Best Practices

1. **After every significant task**: Save the pattern as a skill
2. **When you notice a preference**: Save it in memory immediately
3. **When project conventions emerge**: Update AGENTS.md
4. **Before starting similar work**: Search sessions and skills first
5. **Share with team**: Commit AGENTS.md, publish useful skills

## Auto-Capture Hook (Advanced)

Add to your workflow: after each completed task, run a quick capture:
```python
# Pseudo-code for post-task hook
def capture_taste(task_description, outcome, files_changed):
    if outcome == "success":
        # Extract reusable pattern
        pattern = extract_pattern(task_description, files_changed)
        if pattern:
            skill_manage(action='create', name=slugify(pattern), content=pattern)
        
        # Note preferences
        preferences = extract_preferences(task_description)
        for pref in preferences:
            memory(action='add', target='user', content=pref)
```

This mimics Command Code's continuous RL feedback loop.