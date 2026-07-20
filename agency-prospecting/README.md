# agency-prospecting

A Claude Code skill that turns a single sentence ("Head of Marketing at B2B SaaS in NYC, 50–200, fintech") into a personalized cold email containing a white-labeled Keyword.com ViewKey ranking report for the prospect's own domain.

Chains the **Apollo.io** and **Keyword.com** MCPs end-to-end.

## What it does

1. **Find a prospect** — Apollo search (you describe the persona in plain English, or fill in structured fields)
2. **Enrich** — pull the verified email + company details with a single Apollo `people_match` call
3. **Scrape homepage** — extract the H1, value proposition, services, geo footprint, credentials
4. **Propose seed keywords** — 3–5 non-branded commercial terms aligned with the prospect's business
5. **Build a keyword project** — call Keyword.com `suggest_related_keywords`, strip branded/competitor noise, pick the top N (5–100, default 25)
6. **Decide tracking geography** — heuristic based on company size + industry (local-service businesses get city-level, B2B/SaaS gets national)
7. **Confirm branding** — fetch effective white-label config, flag typos / name mismatches / missing fields, let you fix in-skill via `update_account_sharing_settings`
8. **Draft the email** — concise (<130 words), references one concrete observation from the prospect's site, ends with the ViewKey share URL
9. **(Optional) Tag the contact** — add them to an Apollo List (default name: `agency-prospecting`) so you can filter for processed prospects later. ⚠️ Apollo's `label_names` API overwrites, doesn't append — the skill warns you if the contact has existing labels and offers to preserve them.

### A note on Apollo terminology

What the API calls `label_names` shows up in Apollo's UI as **Lists** (left nav: Search → People → "Lists" filter). There's no separate "Tag" concept in Apollo via this MCP — Lists are what you'd find tagged contacts under.

## Prerequisites

- **Claude Code** (app or CLI)
- **A Keyword.com account** with the `write:data` scope — no account yet? Free trial (100 keywords, 14 days) at [app.keyword.com/users/signup](https://app.keyword.com/users/signup). The plugin bundles the Keyword.com MCP, so it's registered automatically; you just complete the OAuth login.
- **Apollo.io** — this skill uses Apollo's connector tools, so add Apollo from the **Connectors directory** (app: **+ → Connectors → Apollo**; or run `/mcp`). It is *not* bundled by the plugin (a bundled copy would load under the wrong tool namespace).
- *(Recommended)* **Keyword.com white-label** configured — agency logo, brand color, optional custom subdomain. The skill verifies branding at the end of every run, but pre-configuring it saves a remediation step.

## Costs per run

- **1–2 Apollo lead credits** — one per contact enriched. If the first contact has no findable email, you'll spend another to pick a different one.
- **N tracked-keyword slots** on your Keyword.com plan (where N is what you chose at the keyword-count step).
- No additional Claude API costs beyond standard message billing.

## Installation

Ships as a **plugin** that bundles the Keyword.com MCP (registered on install); you then add Apollo from Connectors.

### CLI
```bash
claude plugin marketplace add benjamin-thorn/skills
/plugin install agency-prospecting@keyword-com-skills
/mcp        # keyword-com → Authenticate (grant write:data)
```
Then add **Apollo** via **+ → Connectors → Apollo** (or `/mcp`).

### Desktop / web app
1. **+ → Plugins → Add marketplace →** `benjamin-thorn/skills`
2. Install **agency-prospecting**
3. `/mcp` → **keyword-com** → **Authenticate**
4. **+ → Connectors → Apollo** → connect

## Usage

In Claude Code, run the namespaced command:
```
/agency-prospecting:agency-prospecting
```

Then describe your prospect in plain English. Examples:

- *"Head of Marketing at B2B SaaS in NYC, 50–200 employees, ideally fintech"*
- *"Director of SEO at ecommerce brands, 200–500, on Shopify"*
- *"Business owners of plumbing companies in London"*
- *"Anyone at stripe.com"* (specific company)

The skill walks you through each step and pauses for approval at three checkpoints (keyword list, project creation, branding gate). The final output is a draft email you copy into Gmail / your client and send manually — the skill does not send for you.

## Caveats

- **Apollo's industry / keyword tags are leaky.** Marketing agencies that *target* SaaS clients often get tagged "SaaS" themselves. Skim the candidate list and the enriched org's `industry` field before committing.
- **Hyperlocal seeds may return very few keywords.** When the suggester returns fewer than N unbranded terms, the skill will offer to combine multiple seeds.
- **Local SEO geo precision lives in the keywords, not the tracking region.** Keyword.com's organic-SERP API tracks at the country level (e.g. `google.co.uk`). True city/postcode-level tracking only exists for Maps-pack (`type: maps`) which requires a Google Business Profile ID.
- **The share-link host stays on `app.keyword.com`** unless you've configured a custom subdomain in Keyword.com white-label settings.
- **Drafted emails are placeholders for your voice.** The skill ends with `[Your name]` and a generic CTA. Personalize before sending — your prospect's bullshit detector is calibrated for templates.

## License

MIT — see `LICENSE`.
