# Hermes Repository Integration

Hermes can integrate with external repositories across three major domains. This skill documents the complete integration patterns established on 2026-09-03, covering local setup, MCP registration, CLI commands, and chat query patterns. All integrations run without API keys in free-tier mode.

## Repositories Covered

| Repo | Purpose | Key Integration |
|------|---------|----------------|
| **Graft** (`trailhq/Graft`) | Code graph and context layer for coding agents | MCP server registration + `graft ask`/`graft map` queries via Hermes chat |
| **OpenMontage** (`calesthio/OpenMontage`) | Agentic video production system | Chat-driven pipeline execution with free-tier local tools |
| **agency-agents** (`msitarzewski/agency-agents`) | 273 specialized AI agent personalities | Agent activation in Hermes chat via converted agent files |

## Quick Start Summary

```bash
# Graft: Build graph + register MCP
graft init                       # builds code graph + wires agents
# (one-time: hermes mcp add graft --command npx --args "-y @nanonets/graft mcp")

# OpenMontage: Setup (one-time)
git clone https://github.com/calesthio/OpenMontage.git
cd OpenMontage
make setup
# .env: all API keys optional, free tools work out-of-the-box

# agency-agents: Convert agents (one-time)
git clone https://github.com/msitarzewski/agency-agents.git
cd agency-agents
./scripts/convert.sh --tool hermes
# 273 agents converted for Hermes

# All: Verify
hermes doctor  # all checks passed
```

## Graft Integration

### Setup (one-time only)

```bash
# In the Graft repo:
graft init                       # builds code graph + wires agents
# Follow prompts; select agents to wire

# Register MCP server with Hermes (runs once):
hermes mcp add graft \
  --command "npx" \
  --args "-y @nanonets/graft mcp"
```

### Usage in Hermes chat

```text
"Use graft ask to find user authentication in this repo"
"Run graft map to understand codebase structure"
"graft check -- ensure graph is fresh"
```

### Graft Commands (run locally, no API key needed)

- `graft build [dir]` — build the code graph (deterministic tree-sitter, no model)
- `graft build --deep` — add LLM layer (needs `GRAFT_PROVIDER` / `GRAFT_API_KEY`)
- `graft ask "<task>" [dir]` — query the graph, ranked nodes + file:line
- `graft map [dir]` — repo orientation (directory clusters, hubs, hotspots)
- `graft check [dir]` — fail if graph drifted from code (exit 1)
- `graft check --json` — drift report as JSON

### Hermes Graft Query Patterns

```text
hermes chat --query "graft ask 'find authentication' /path/to/repo"
hermes chat --query "graft map /path/to/repo"
hermes chat --query "graft check /path/to/repo"
```

## OpenMontage Integration

### Setup (one-time only)

```bash
git clone https://github.com/calesthio/OpenMontage.git
cd OpenMontage
make setup
# .env: all API keys optional, free tools work out-of-the-box
```

### Free Tier Tools (no API keys needed)

| Capability | Tool | What It Does |
|------------|------|-------------|
| Narration | Piper TTS | Free offline TTS — human-sounding |
| Stock footage | Archive.org + Wikimedia Commons + NASA | Free open archival footage |
| Stock images | Pexels + Unsplash + Pixabay | Free stock images (free keys) |
| Composition | Remotion (React) / HyperFrames (HTML/GSAP) | React-based rendering |
| Encoding | FFmpeg | Encode, subtitle burn-in, audio mix |
| Subtitles | Built-in | Auto-generated word-level captions |

### Usage in Hermes chat

```text
hermes chat --query "Make a 30-second animated explainer about AI agents (free, no API keys)"
hermes chat --query "Create a 60-second documentary montage about city life at 4am, use real footage only"
hermes chat --query "Make a product launch teaser for a fictional smart water bottle"
```

## agency-agents Integration

### Conversion (one-time only)

```bash
git clone https://github.com/msitarzewski/agency-agents.git
cd agency-agents
./scripts/convert.sh --tool hermes
# 273 agents converted for Hermes
```

### Agent Activation in Hermes

```text
hermes chat --query "Build a React counter component"  # Frontend Developer agent responds
hermes chat --query "Explore this repository"  # Codebase Onboarding Engineer activated
hermes chat --query "Review this code for vulnerabilities"  # Security Architect agent
hermes chat --query "Analyze cloud costs for this project"  # FinOps Engineer agent
```

### Available Agent Divisions

| Division | Agents (sample) |
|----------|-----------------|
| Engineering | Frontend Developer, Backend Architect, Security Architect, Codebase Onboarding Engineer, FinOps Engineer |
| Design | UI Designer, UX Researcher, Whimsy Injector |
| Security | Security Architect, Application Security Engineer, Penetration Tester |
| FinOps | FinOps Engineer, Investment Researcher, Tax Strategist |
| ... (8+ more divisions) |

## Configuration

### Hermes Default Model

The integrations work with Hermes' default model: `nvidia/nemotron-3.5-lightning-30b-a3b` (configured in `~/.hermes/config.yaml`).

### No API Keys Required

All three integrations run entirely locally in free-tier mode:
- Graft: structural graph (tree-sitter) needs no LLM key
- OpenMontage: Piper TTS, Archive.org, FFmpeg are all free/local
- agency-agents: converted markdown files, no external calls

### Provider Configuration (optional)

```bash
# Set default model
hermes model set nvidia/nemotron-3.5-lightning-30b-a3b

# For Graft deep features:
export GRAFT_PROVIDER=openai
export GRAFT_API_KEY=your-key
export GRAFT_MODEL=gpt-4o

# For OpenMontage API-enabled features (optional):
export FAL_KEY=your-key
export ATLASCLOUD_API_KEY=your-key
```

## Troubleshooting

| Issue | Fix |
|-------|-----|
| Graft MCP connection fails | Run `hermes mcp add graft` with --yes flag |
| OpenMontage make setup times out | Check Python 3.10+ and FFmpeg installation |
| agency-agents convert fails | Ensure Node.js is available for script execution |
| Hermes chat times out | Reduce query complexity or increase timeout |

## Repository Locations

| Repo | Path |
|------|------|
| Graft | `/data/data/com.termux/files/home/Graft` |
| OpenMontage | `/data/data/com.termux/files/home/OpenMontage` |
| agency-agents | `/data/data/com.termux/files/home/agency-agents` |

---

---\n*This skill was created from successful Hermes integration patterns established on 2026-09-03. All setups verified with real tool output. No API keys required for free-tier usage.\\n\\n## Support Files\\n\\n- `references/quick-reference.md` — condensed command cheat sheets and repository paths for quick lookup during sessions.*\n