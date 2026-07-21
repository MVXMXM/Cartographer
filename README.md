# 🗺️ Cartographer

Teach your agent to create beautiful user flows: explain what you'd like to map and it will turn observation into artifact by searching your repo or deploying computer-use agents to survey the terrain then deliver a diagram in your preference of FigJam or HTML.

Built to the open [Agent Skills](https://www.skills.sh/) format, so it runs in Claude Code, Cursor, Codex, and the other agents the ecosystem supports.

## What it does

Cartographer works in three phases:

1. **Survey** — does the legwork by walking the terrain and recording its findings through reading your code, docs, and sending out a computer-use agent as a last resort to drive your product like a human would.
2. **Synthesize** — turns the surveyed paths into a user flow indicating which entity (user, AI, deterministic system) owns each step.
3. **Render** — draws the flow as a diagram in FigJam, if you've provided a file, or a standalone HTML doc.

## Install

Install with the [skills.sh](https://www.skills.sh/) CLI:

```bash
# Into the current project, for your agent of choice
npx skills add MVXMXM/Cartographer

# Target a specific agent (claude-code, cursor, codex, cline, …)
npx skills add MVXMXM/Cartographer --agent cursor
```

Then invoke it from your agent:

```
/cartographer map the onboarding flow for apps/web-dashboard
```

Prefer to install by hand? Copy `SKILL.md` and the `references/` directory into your agent's skills folder — e.g. `.claude/skills/cartographer/` for Claude Code.

## Requirements

- An agent that supports the Agent Skills format ([list](https://www.skills.sh/))
- For FigJam output: a Figma integration exposing a `use_figma`-style tool — in Claude Code, the [`figma@claude-plugins-official`](https://github.com/anthropics/claude-plugins) plugin (`/plugin install figma@claude-plugins-official`)
- For HTML output: nothing extra — a self-contained `.html` file is the fallback when no Figma integration is connected
- For computer-use input: a browser-automation tool — the Yutori Browse / Research skills by default, or Playwright MCP / similar
