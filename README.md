# Treasures Finance Agent Skills

**Agent Skills** for building AI agents on the Treasures finance APIs.

A skill is a folder of plain-Markdown instructions (`SKILL.md`) that a coding agent loads on demand. The skills here teach an agent to call the Treasures finance APIs correctly — discover tokenized stocks, quote and execute trades, bridge USDC across chains, operate a delegated wallet, and read portfolios — including the signing details and footguns that are easy to get wrong.

## Skill catalog

| Skill | What it does |
| ----- | ------------ |
| [`treasures-b2b-api`](skills/treasures-b2b-api/SKILL.md) | Build an agent on the Treasures public B2B API: discover tokenized stocks, quote/execute trades, bridge USDC across Solana and Ethereum, and read portfolio + trade history for a single end-user wallet pair. Covers endpoint selection, ownership-proof signing (incl. embedded wallets), trade/bridge execution, and error handling. |
| [`treasures-wallet`](skills/treasures-wallet/SKILL.md) | Operate a Treasures delegated wallet over HTTP: onboard (provision a wallet + mint a scoped API key), quote, execute async buys/sells (server-custodied — the agent never signs), read balances/portfolio/trade history, and manage API keys. Trades tokenized equities (xStocks / Ondo) vs USDC on Solana or Ethereum with only HTTPS + an API key — no web3 libraries, keys, or RPC. |

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

## License

[MIT](LICENSE)
