# frontend-nextjs-dev

Claude Code skills for building production-grade Next.js applications. These skills activate automatically when working on Next.js projects.

## Skills

| Skill | Activates When |
|-------|---------------|
| [frontend-nextjs-dev](skills/frontend-nextjs-dev/SKILL.md) | Working on any Next.js application — routing, data fetching, performance, styling |

## Usage

Once installed, Claude automatically applies the skill when you're working on a Next.js project:

```
"Add a new page to my Next.js app"
"How do I fetch data in a Server Component?"
"Optimize my Next.js app's performance"
"Set up Tailwind CSS in my Next.js project"
```

## Installation

```bash
# Install via Claude Code plugin manager
claude plugin install github:dhruvdoshi/claude-skills
```

Or clone and install locally:

```bash
git clone https://github.com/dhruvdoshi/claude-skills
claude plugin install ./claude-skills
```

## Structure

```
frontend-nextjs-dev/
├── .claude-plugin/
│   └── plugin.json          # Plugin metadata
├── agents/
│   └── nextjs-reviewer.md   # Code review agent
├── commands/
│   └── nextjs-dev.md        # /nextjs-dev slash command
├── skills/
│   └── frontend-nextjs-dev/
│       └── SKILL.md         # Auto-applied Next.js knowledge
├── .mcp.json                # MCP server integrations (placeholder)
├── LICENSE
└── README.md
```

## Author

Dhruv Doshi
