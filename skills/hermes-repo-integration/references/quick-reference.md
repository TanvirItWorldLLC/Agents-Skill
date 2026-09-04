# Hermes Repo Integration — Quick Reference

## Graft MCP Registration

```bash
hermes mcp add graft \
  --command "npx" \
  --args "-y @nanonets/graft mcp"
```

Or add to `~/.hermes/agent-mcp.json`:
```json
{ "mcpServers": { "graft": { "command": "npx", "args": ["-y", "@nanonets/graft", "mcp"] } }
```

## Graft Command Cheat Sheet

| Command | Purpose | Example |
|---------|---------|---------|
| `graft build [dir]` | Build code graph | `graft build /path/to/repo` |
| `graft build --deep` | Add LLM layer | `graft build --deep` |
| `graft ask "<task>" [dir]` | Query graph | `graft ask "find auth" /path/to/repo` |
| `graft map [dir]` | Repo orientation | `graft map /path/to/repo` |
| `graft check [dir]` | Drift detection | `graft check --exit-code` |
| `graft init` | Build + wire agents | `graft init` (prompts on TTY) |
| `graft init --dry-run` | List files to touch | `graft init --dry-run` |
| `graft init --agents claude` | Wire Claude only | `graft init --agents claude` |

## OpenMontage Free Tier Cheat Sheet

| Capability | Tool | Command |
|------------|------|---------|
| Narration | Piper TTS | `piper -f script.txt -o audio.wav` |
| Stock footage | Archive.org | Browse at archive.org |
| Stock images | Pexels/Unsplash/Pixabay | Free API keys at respective sites |
| Composition | Remotion | `npx @remotion/cli kV9vE9VIj7eI7VIj render` |
| Encoding | FFmpeg | `ffmpeg -i input.mp4 -vf "subtitles=subs.srt" output.mp4` |

## agency-agents Conversion

```bash
./scripts/convert.sh --tool hermes
# Output: 273 agents converted for hermes
# Files written to: integrations/hermes/agency-agents-router/
```

## Agent Activation Patterns

```text
# Frontend Developer
hermes chat --query "Build a React counter component"

# Codebase Onboarding  
hermes chat --query "Explore this repository"

# Security Architect
hermes chat --query "Review this code for vulnerabilities"

# FinOps Engineer
hermes chat --query "Analyze cloud costs for this project"
```

## Repository Paths

| Repo | Absolute Path |
|------|--------------|
| Graft | `/data/data/com.termux/files/home/Graft` |
| OpenMontage | `/data/data/com.termux/files/home/OpenMontage` |
| agency-agents | `/data/data/com.termux/files/home/agency-agents` |

## Default Model

`nvidia/nemotron-3.5-lightning-30b-a3b` (configured in `~/.hermes/config.yaml`)

All tool calls are `$0` structural—no LLM key needed for graph/build/check.