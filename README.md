# Keyword.com Claude Code skills

A **Claude Code plugin marketplace** of SEO workflows built on the [Keyword.com](https://keyword.com) MCP (and, for prospecting, [Apollo.io](https://apollo.io)). Install a plugin and drive real rank-tracking, keyword research, and reporting from Claude Code — in the app or the CLI.

## Quick start

```bash
# 1. Add this marketplace
claude plugin marketplace add benjamin-thorn/skills
# 2. Install a plugin (see the table below)
/plugin install project-setup@keyword-com-skills
# 3. Authenticate the Keyword.com MCP (pulled in automatically)
/mcp        # keyword-com → Authenticate
```

In the **desktop / web app**: **+ → Plugins → Add marketplace →** `benjamin-thorn/skills`, then install the plugin you want.

No Keyword.com account yet? Start a free trial (100 keywords, 14 days) at [app.keyword.com/users/signup](https://app.keyword.com/users/signup) — the OAuth step needs one.

## Plugins

| Plugin | What it does | Install |
|---|---|---|
| **[project-setup](./project-setup)** | Turn a URL into a fully populated, tag-organized tracking project — crawl the site's pages + blog, distill product & content topics, expand into real keywords, and create the tagged project. | `/plugin install project-setup@keyword-com-skills` |
| **[agency-prospecting](./agency-prospecting)** | End-to-end prospect outreach: Apollo contact search → enrich → seed keyword → white-labeled ViewKey report → personalized cold-email draft. | `/plugin install agency-prospecting@keyword-com-skills` |
| **[ppc-savings](./ppc-savings)** | Quantify what organic rankings are worth: CPC × estimated organic traffic for top-ranking keywords → a client-ready "equivalent ad spend" report. | `/plugin install ppc-savings@keyword-com-skills` |

Each links to its own README for the full workflow, prerequisites, and caveats.

## How the marketplace is organized

The repo root is the marketplace; each top-level folder is a plugin, listed in [`.claude-plugin/marketplace.json`](./.claude-plugin/marketplace.json). Each skill plugin **bundles the Keyword.com MCP** in its own `.mcp.json`, so installing it registers the server automatically. If you install more than one, Claude Code **dedupes the identical server to a single connection** — so there's no double-registration to worry about.

## Prerequisites

- **Claude Code** (app or CLI)
- **A Keyword.com account** with the `write:data` scope for plugins that create/modify data (`project-setup`, `agency-prospecting`); read scope is enough for `ppc-savings`. Each plugin bundles the Keyword.com MCP, so it's registered on install; you complete the OAuth login via `/mcp`.
- **Apollo** — only for `agency-prospecting`, added from the Connectors directory (**+ → Connectors → Apollo**), not bundled.

## Contributing

Issues and PRs welcome. To add a skill: package it as a plugin (`<name>/.claude-plugin/plugin.json` + `<name>/skills/<name>/SKILL.md`), add an entry to `.claude-plugin/marketplace.json`, and include a `.mcp.json` bundling the Keyword.com MCP if it needs it.

## License

MIT — see [LICENSE](./LICENSE).
