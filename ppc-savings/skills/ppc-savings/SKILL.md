---
name: ppc-savings
description: Quantify the equivalent Google Ads spend a site's organic rankings replace ("PPC savings"). For one Keyword.com project (or a tag slice), takes every keyword ranking 1–3, multiplies its CPC by its estimated monthly organic traffic, and totals it into a client-ready savings report — with branded terms split out and a tier-2 section showing the additional savings available if keywords ranking 4–10 were pushed into the top 3. Use when the user asks "how much is SEO saving us/this client in PPC", "PPC savings report", "traffic value", "equivalent ad spend", "what would this organic traffic cost in Google Ads", or runs /ppc-savings.
---

# PPC Savings

This skill produces one deliverable: a markdown report that says **"your organic rankings are doing the work of $X/month in Google Ads spend"** — grounded entirely in Keyword.com tracking data (no invented numbers).

Methodology in one line: for every keyword ranking in positions 1–3, `savings = CPC × estimated monthly organic clicks`. This is the same "traffic value" approach used by Ahrefs/Semrush, and the report must label it as **equivalent ad spend**, not literal savings (paying full CPC for every organic click slightly overstates — say so in the footnote, it keeps the number credible).

## Prerequisites

Keyword.com MCP connected (read scope is enough — this skill writes nothing). Verify with `mcp__keyword-com__list_projects` `limit: 1`; if it errors, tell the user to connect via `/mcp` and stop.

## Workflow

### Step 1 — Resolve scope

- If the user named a project, resolve it with `mcp__keyword-com__search_projects`. If ambiguous, list the matches and ask.
- If no project named, call `mcp__keyword-com__list_projects` and ask the user to pick.
- If the user asked for a tag slice ("just the product keywords"), resolve the tag with `mcp__keyword-com__search_tags` and pass `tag_id` to every `list_keywords` call below.
- **Account mode:** if the user says "all projects" / "the whole account", run Steps 2–5 per project and add a final per-project summary table. Warn first if the account has more than ~10 projects (it's one paginated keyword pull per project).

### Step 2 — Pull tier-1 keywords (positions 1–3)

Call `mcp__keyword-com__list_keywords` with:

```
project_id, rank_min: 1, rank_max: 3, limit: 100,
sort_by: "estimated_traffic", sort_direction: "desc",
output_format: "json"
```

Paginate via `page` until `total_pages` is exhausted. Capture per keyword: text, `current_rank`, `msv`, `cpc`, `estimated_traffic`, `keyword_type`, location, `ranking_url`.

If zero keywords rank 1–3, skip to Step 4 and frame the whole report around tier-2 potential ("here's what top-3 rankings would be worth").

### Step 3 — Clean the data

Apply these rules **before** any math:

1. **Device dedupe.** The same keyword tracked on desktop + mobile (same text + location) would double-count. Keep the row with the higher `estimated_traffic`; count how many duplicates you dropped and note it in the report footer.
2. **Outlier guard.** Exclude — and list in the report footer with the dollar amount they *would* have added — any keyword that fails a sanity check:
   - `ranking_url` is null, or its domain doesn't match the project's domain (the keyword may be a demo/competitor-watch entry, not a real ranking);
   - the keyword is obviously off-topic for the project's domain (e.g. `dentist` in a rank-tracker project — these demo keywords are common in real accounts and can add **millions/month of fake savings**);
   - any single keyword contributing >30% of the running total → pause and ask the user to confirm it's genuinely theirs before including it.
3. **Branded split.** Detection order: (a) if the project has a tag named `branded`/`brand`, use it (resolve via `search_tags`, pull its keyword set); (b) otherwise match **domain-variant patterns only** — the full domain with/without TLD, spacing, and plural (`keyword.com`, `keyword com`, `keywords.com`…) plus navigational modifiers (`login`, `pricing`, `api` appended to the domain name). Do **not** treat a bare brand token as branded when it's a dictionary word the project's whole niche is built on (e.g. "keyword" for keyword.com — that would classify the entire project as branded). When the brand token is generic, show the user the branded classification and confirm before finalizing. Compute branded and non-branded totals separately — **the headline number is non-branded**. Rationale (state it in the report): you rank #1 for your brand regardless of SEO effort, and few advertisers pay full listed CPC on their own brand.
4. **Missing CPC.** `cpc` null or 0 → the keyword contributes zero dollars and is excluded from the total. Report the count separately ("N ranking keywords had no CPC data — savings understated").
5. **Missing traffic estimate.** `estimated_traffic` null/0 but `msv > 0` → fall back to a CTR-curve estimate: `msv × CTR(rank)` with CTR = 27% (pos 1), 15% (pos 2), 9% (pos 3). Mark these rows with `~` in the table.

### Step 4 — Tier 2: potential savings (positions 4–10)

Call `list_keywords` again with `rank_min: 4, rank_max: 10` (same pagination, dedupe, outlier guard, branded split). **Cross-tier dedupe:** skip any keyword text already counted in tier 1 — the same keyword often ranks 1–3 on one device and 4–10 on another, and counting both inflates the opportunity. For each remaining non-branded keyword:

```
potential_traffic   = msv × 0.09          # valued conservatively at position 3
additional_savings  = cpc × max(0, potential_traffic − current_estimated_traffic)
```

This is the "money still on the table" section — keep the position-3 valuation conservative so the report never looks inflated.

### Step 5 — Compute totals

For any project with more than ~50 qualifying rows, do **not** sum in your head: write the rows to a temp file (e.g. `/tmp/ppc-savings-<project_id>.json`) and total with a short python3 one-liner/script. Arithmetic slips in a money report are fatal to credibility.

Compute:
- `monthly_savings_nonbranded` = Σ cpc × traffic (tier 1, non-branded)
- `monthly_savings_branded` = same for branded rows
- `annual = monthly × 12`
- `tier2_potential_monthly` = Σ additional_savings (Step 4)

### Step 6 — Render the report

```markdown
# PPC Savings Report — {Project name}
*{date} · {N} keywords ranking 1–3 ({M} non-branded) · data: Keyword.com*

## 💰 ${monthly_nonbranded}/month equivalent ad spend
**${annual}/year** — what it would cost in Google Ads to buy the clicks
your top-3 organic rankings already deliver.
(+ ${monthly_branded}/mo from branded terms, reported separately below.)

## Top savings drivers
| # | Keyword | Pos | MSV | Est. clicks/mo | CPC | Savings/mo |
|---|---------|-----|-----|----------------|-----|------------|
(top 20 non-branded by savings; `~` marks CTR-curve traffic fallbacks)
… and {K} more keywords contributing ${rest}/mo.

## Branded terms (excluded from headline)
{1-line total + top 3 rows, or "none detected"}

## 📈 Another ${tier2_potential}/mo within reach
{top 10 keywords ranking 4–10 by additional_savings: Keyword | Pos | MSV | CPC | Potential savings/mo}
Pushing these into the top 3 would add ${tier2_potential}/mo (${tier2_annual}/yr)
of equivalent ad value.

---
*Figures are **equivalent ad spend** — what these clicks would cost in Google Ads — not revenue. Directional. Ask for the full methodology (dedupe, CTR model, exclusions) any time.*
```

**Keep the report lean — do not print the full methodology by default.** The one-line caveat above is the *only* mandatory disclosure, because it's the guardrail against the number being misread as literal revenue. Everything else (dedupe count, CTR model, no-CPC count, outliers excluded, branded rationale, local-MSV note) stays computed internally but **out of the deliverable**.

**Only if the user asks** "how did you calculate this?", "show your methodology", "is that number real?", or similar — expand the full breakdown then:
- Savings = CPC × estimated monthly organic clicks, per keyword ranking 1–3 (the Ahrefs/Semrush "traffic value" method).
- Traffic estimates are CTR-modelled from rank + search volume.
- {N_nocpc} keywords had no CPC data → total is understated.
- {dupes} device duplicates removed; {fallbacks} rows used CTR-curve fallback.
- {N_outliers} keywords excluded by sanity check (null/foreign ranking_url or off-topic) — they would have added the amount you'd list per keyword.
- {if local project} MSV is national; local tracking may overstate volume.

Round dollar figures: above 1,000 round to the nearest 10; above 10,000 round to the nearest 100 — false precision reads as fake. (Note: never write a dollar sign followed by a digit in this file — Claude Code substitutes `$<digit>` patterns with skill arguments.)

Offer at the end (don't do it unsolicited): save the report to a file, or re-run scoped to a tag / different project.

## Failure modes — handle explicitly

- **No keywords rank 1–3:** lead with the tier-2 potential section; never fabricate a savings number.
- **All CPCs are 0/null** (common for fresh projects whose metrics haven't populated): say so plainly, suggest re-running in a few days. Do not substitute guessed CPCs.
- **`estimated_traffic` missing project-wide:** use the CTR-curve fallback for every row and flag the whole report as modelled.
- **Maps-type keywords (`keyword_type: "maps"`):** organic CTR curves don't apply to map-pack results — exclude them from the math, note the count.
- **Huge projects (>1,000 tier-1 rows):** paginate fully anyway; totals must cover everything even if the table shows top 20.

## What this skill never does

- Never invents CPC, MSV, or traffic numbers not returned by the API.
- Never writes to Keyword.com (read-only).
- Never presents the headline as literal dollars saved — always "equivalent ad spend".
