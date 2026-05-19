# skills

A collection of Claude Code skills for SEO agencies and Keyword.com users. Each skill is a self-contained workflow built on top of the [Apollo.io](https://apollo.io), [Keyword.com](https://keyword.com), and other MCP integrations available in Claude Code.

## Available skills

| Skill | What it does |
|---|---|
| [agency-prospecting](./agency-prospecting) | End-to-end prospect outreach: Apollo search → enrich → keyword research → ranking report → personalized cold email draft. |

More skills will be added here over time — keep an eye on this repo or watch it on GitHub.

## Installation

### Manual (always works)

Clone the repo and symlink the skill you want into your Claude skills directory:

```bash
git clone https://github.com/keyword-rank-tracker/skills ~/keyword-skills
ln -s ~/keyword-skills/agency-prospecting ~/.claude/skills/agency-prospecting
```

Repeat the `ln -s` line for each skill you want to install.

### Via the Skills CLI

If you use the [Skills CLI](https://skills.sh), subdirectory installs from a monorepo may be supported via:

```bash
npx skills add github:keyword-rank-tracker/skills/agency-prospecting
```

(If the CLI doesn't support the subdirectory form on your version, fall back to the manual install above.)

## Prerequisites

Each skill has its own prerequisites — see the individual skill's README for details. Common requirements across the collection:

- **Claude Code** (or any Claude Agent SDK client that supports skills)
- One or more MCP integrations connected via `/mcp` (Apollo, Keyword.com, etc. — varies by skill)
- A Keyword.com account with the relevant scopes if the skill writes data

## Contributing

Issues and PRs welcome. If you've built a skill that uses the Keyword.com MCP and want it added here, open an issue with a link to your repo or branch.

## License

MIT — see [LICENSE](./LICENSE). Each skill in this repo is covered by the same license unless its own directory contains a `LICENSE` file overriding it.
