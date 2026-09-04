# Agents-Skill Repository

This repository contains Hermes Agent skills for AI coding agents.

## Repository Structure

```
Agents-Skill/
├── README.md           # This file
├── skills/             # All Hermes Agent skill collections
│   ├── autonomous-ai-agents/     # Claude Code, Codex, Hermes-agent, etc.
│   ├── creative/                   # ASCII art, design, video generation
│   ├── devops/                     # Deployment and SDLC skills
│   ├── email                       # Email management skills
│   ├── github                      # GitHub workflow skills
│   ├── hermes-repo-integration     # Hermes repo integration
│   ├── media                       # GIF, YouTube, audio skills
│   ├── mlops                       # ML training, serving, evaluation
│   ├── note-taking                 # Obsidian and note skills
│   ├── productivity                # Documents, spreadsheets, presentations
│   ├── research                    # Academic and market research
│   ├── smart-home                  # Home automation skills
│   ├── social-media                # X/Twitter and social platforms
│   ├── software-development        # TDD, debugging, code reviews
│   └── web                         # Blocked page recovery, web tools
```

## Installation

### Prerequisites

- Hermes Agent installed and running
- Git configured with your GitHub account

### Option 1: Clone the repository

```bash
git clone https://github.com/TanvirItWorldLLC/Agents-Skill.git
cd Agents-Skill
```

### Option 2: Install via Hermes

```bash
# From your Hermes session
hermes skills install agents-skill
# Or install individual skill categories
hermes skills install autonomous-ai-agents
hermes skills install creative
# ... etc
```

### Option 3: Manual skill loading

```bash
# Copy skills to your Hermes profile
cp -r skills/.hermes/skills/  # or use hermes skills install
# Then load skills in session
hermes skills load <skill-id>
```

## Usage

### List available skills

```bash
hermes skills list
```

### View a specific skill

```bash
hermes skills view <skill-name>
# Example: hermes skills view hermes-agent
```

### Load/install a skill

```bash
# Install from this repository
hermes skills install <skill-id>

# Or load directly from the skills directory
hermes skills load <skill-id>

# Example: Load the Hermes-agent skill
hermes skills load hermes-agent
```

### Use skill commands

Each skill provides specific commands. For example:

```bash
# Hermes-agent skill commands
hermes agent configure
hermes agent status
hermes agent tasks

# GitHub skills
gh pr create
gh issue list
```

## Skill Categories

| Category | Description | Key Skills |
|----------|-------------|------------|
| **autonomous-ai-agents** | AI agent orchestration and deployment | claude-code, codex, computer-use, hermes-agent, merge-reconciler, opencode |
| **creative** | Creative content generation | architecture-diagram, ascii-art, ascii-video, baoyu-infographic, claude-design, comfyui, design-md, excalidraw, humanizer, manim-video, p5js, popular-web-designs, pretext, sketch, songwriting-and-ai-music, touchdesigner-mcp |
| **devops** | Deployment and SDLC | fullstack-deployment |
| **email** | Email management | himalaya, inbox-triage |
| **github** | GitHub workflows | github-auth, github-code-review, github-issue-to-pr, github-issues, github-pr-workflow, github-repo-management |
| **hermes-repo-integration** | Hermes repo integration | hermes-repo-integration |
| **media** | Media content | gif-search, songsee, youtube-content |
| **mlops** | ML operations | huggingface-hub, evaluation, inference, weights-and-biases |
| **note-taking** | Note taking | obsidian |
| **productivity** | Document and presentation tools | airtable, box, docx, google-workspace, maps, meeting-action-items, nano-pdf, notion, pdf, powerpoint, product-price-monitor, session-librarian, teams-meeting-pipeline, weekly-review-planning, xlsx |
| **research** | Academic research | arxiv, blogwatcher, competitor-news-monitor, grounded-citations, llm-wiki, topic-research |
| **smart-home** | Home automation | openhue |
| **social-media** | Social platform integration | xurl |
| **software-development** | Engineering best practices | agent-skills, android-mediapipe-cv, codebase-inspection, dogfood, github, hermes-agent-skill-authoring, inspecting-hermes-desktop-dom, node-inspect-debugger, plan, python-debugpy, requesting-code-review, simplify-code, spike, systematic-debugging, taste-learning, test-driven-development |
| **web** | Web tools | blocked-page-recovery |

## Updating Skills

To update skills from the repository:

```bash
cd Agents-Skill
git pull origin main
# Then re-install or re-load skills as needed
```

## Contributing

1. Fork the repository
2. Add or update skills in the `skills/` directory
3. Ensure each skill has proper SKILL.md frontmatter
4. Submit a pull request

## License

MIT License - see LICENSE file for details.