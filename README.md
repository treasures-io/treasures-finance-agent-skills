# Treasures Finance Agent Skills

**Agent Skills** for building AI agents on the Treasures finance APIs.

A skill is a folder of plain-Markdown instructions (`SKILL.md`) that a coding agent loads on demand. The skills here teach an agent to call the Treasures public B2B API correctly — discover tokenized stocks, quote and execute trades, bridge USDC across chains, and read portfolios — including the signing details and footguns that are easy to get wrong.

## Skill catalog

| Skill | What it does |
| ----- | ------------ |
| [`treasures-b2b-api`](skills/treasures-b2b-api/SKILL.md) | Build an agent on the Treasures public B2B API: discover tokenized stocks, quote/execute trades, bridge USDC across Solana and Ethereum, and read portfolio + trade history for a single end-user wallet pair. Covers endpoint selection, ownership-proof signing (incl. embedded wallets), trade/bridge execution, and error handling. |

## Install

### Option A — `npx skills`

[`npx skills`](https://github.com/vercel-labs/skills) installs `SKILL.md` files into the right place for 70+ coding agents (Claude Code, Codex, Cursor, GitHub Copilot, Windsurf, Cline, OpenCode, …) and auto-detects which ones you have. From your project root:

```bash
npx skills add treasures-io/treasures-finance-agent-skills
```

Target specific agents with `-a`:

```bash
npx skills add treasures-io/treasures-finance-agent-skills -a claude-code -a codex -a cursor
```

It reads this repo's `skills/<name>/SKILL.md` layout directly, so no extra setup is required.

### Option B — Manual copy

Each agent loads skills from its own directory — for example Claude Code from `~/.claude/skills/` (or a project's `.claude/skills/`), Codex from `~/.codex/skills/`, and several agents from a shared `.agents/skills/`. Clone the repo and drop the skill folder into your agent's skills directory:

```bash
git clone https://github.com/treasures-io/treasures-finance-agent-skills.git
# example: Claude Code, available in all projects
mkdir -p ~/.claude/skills
cp -r treasures-finance-agent-skills/skills/treasures-b2b-api ~/.claude/skills/
```

Check your agent's docs for its exact skills path, or just use Option A, which handles this for you.

### Option C — Claude Code plugin marketplace

This repo also doubles as a [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces). Inside Claude Code:

```
/plugin marketplace add treasures-io/treasures-finance-agent-skills
/plugin install treasures-finance-agent-skills@treasures-finance-agent-skills
```

Manage anytime with `/plugin`; pull updates with `/plugin marketplace update`. Add `--scope project` to the install to share it with everyone on a project.

## Using a skill

Once installed, you don't invoke skills by hand — your agent reads each skill's `description` and pulls in the right one automatically when your request matches. Just describe the task, e.g.:

> "Quote a buy of 2 AAPL tokenized shares on Solana for this wallet and walk me through submitting it."

The agent will load `treasures-b2b-api`, follow the endpoint flow, and open the relevant reference (auth, trading, bridging, data, errors) for the step it's on.

## Repository layout

```
.
├── .claude-plugin/           # Claude Code plugin/marketplace manifests (Option C)
│   ├── marketplace.json
│   └── plugin.json
└── skills/
    └── treasures-b2b-api/
        ├── SKILL.md          # entry doc: map + footguns + endpoint index
        └── references/       # loaded on demand, per task
            ├── auth.md        # ownership-proof signing (incl. embedded wallets)
            ├── trading.md     # quoting/submitting buys & sells, approvals
            ├── bridging.md    # USDC bridge across Solana/Ethereum
            ├── data.md        # /stocks, /portfolio, /trades reads
            └── errors.md      # error matrix, rate limits, simulation
```

## License

[MIT](LICENSE)
