# ppc-savings

A Claude Code skill that answers one question every client and CFO asks: **"What is SEO actually worth in dollars?"**

For a Keyword.com project, it takes every keyword ranking in positions 1–3, multiplies each keyword's CPC by its estimated monthly organic clicks, and totals it into a client-ready report:

> 💰 **$24,670/month equivalent ad spend** — $296,000/year — what it would cost in Google Ads to buy the clicks your top-3 organic rankings already deliver.

This is the same "traffic value" methodology used by Ahrefs/Semrush, computed from your own tracked rankings instead of a third-party index.

## What it does

1. **Resolves scope** — one project, a tag slice, or the whole account
2. **Pulls tier-1 keywords** (positions 1–3) with CPC + estimated traffic from the Keyword.com MCP
3. **Cleans the data** — device dedupe, outlier sanity checks, branded/non-branded split, missing-CPC handling
4. **Computes savings** — `CPC × estimated monthly clicks` per keyword, totalled with a script (no LLM arithmetic in a money report)
5. **Adds the upside** — keywords ranking 4–10, valued conservatively at position-3 CTR: *"another $31,400/mo within reach"*
6. **Renders a markdown report** — headline number, top-20 drivers table, branded section, tier-2 opportunities, honest methodology footnote

## Built-in safeguards

These came out of real-world testing, not theory:

- **Outlier guard** — demo/competitor-watch keywords with null or foreign `ranking_url` are excluded and listed. In our test account, two stray demo keywords (`dentist`, `dentist near me`) would have added **$3.5M/month of fake savings**.
- **Generic-brand detection** — if your brand token is a dictionary word (e.g. "keyword" for keyword.com), the skill matches domain variants only and confirms the branded classification with you instead of silently misclassifying the whole project.
- **Cross-tier dedupe** — the same keyword ranking 1–3 on desktop and 4–10 on mobile counts once.
- **Non-branded headline** — branded terms are reported separately; you rank #1 for your own brand regardless of SEO effort.
- **Honest framing** — the number is always labeled *equivalent ad spend*, never literal savings, with the assumptions spelled out in the footnote.

## Prerequisites

- **Claude Code** (app or CLI)
- **A Keyword.com account** — read scope is enough; this skill writes nothing. No account yet? Free trial at [app.keyword.com/users/signup](https://app.keyword.com/users/signup). The plugin bundles the Keyword.com MCP, so it's registered automatically; you just complete the OAuth login.

## Installation

Ships as a **plugin** that bundles the Keyword.com MCP (registered on install).

### CLI
```bash
claude plugin marketplace add benjamin-thorn/keyword-com-skills
/plugin install ppc-savings@keyword-com-skills
/mcp        # keyword-com → Authenticate
```

### Desktop / web app
**+ → Plugins → Add marketplace →** `benjamin-thorn/keyword-com-skills` → install **ppc-savings** → `/mcp` → **keyword-com** → **Authenticate**.

## Usage

In Claude Code, run the namespaced command:
```
/ppc-savings:ppc-savings
```

Or in plain English:

- *"How much is SEO saving this client in PPC?"*
- *"Run a PPC savings report on the Acme project"*
- *"What would our organic traffic cost in Google Ads?"*
- *"Traffic value for the whole account"*

## Methodology & caveats

- **Savings = CPC × estimated monthly organic clicks** for every keyword ranking 1–3. Assumes you'd pay full listed CPC for each organic click — directionally right, slightly generous. The report says so.
- **Traffic estimates are CTR-modelled** from rank and search volume; where Keyword.com's estimate is missing, a standard CTR curve (27% / 15% / 9% for positions 1–3) is applied and flagged.
- **Keywords with no CPC data contribute $0**, so the total is *understated* — the safest direction to be wrong in.
- **Tier-2 potential** values positions 4–10 at position-3 CTR (9%), the conservative end of a top-3 outcome.
- **MSV is national** — local-tracking projects may overstate volume; the report flags this.

## License

MIT — see the repo's `LICENSE`.
