---
name: agency-prospecting
description: End-to-end prospect-outreach workflow for SEO agencies that combines Apollo.io and Keyword.com MCPs. Given search criteria for a target contact (free-form description or structured fields), finds the contact, enriches their company, picks a relevant non-branded seed keyword from their homepage, builds a tracking project with 5–100 keywords, gets a white-labeled ViewKey share link, confirms branding, and drafts a personalized cold email referencing the report. Use when the user asks to "prospect", "agency prospecting", "build a ViewKey report for a lead", "run agency-prospecting", or wants to combine Apollo + Keyword.com for SEO outreach.
---

# Agency Prospecting

This skill chains Apollo.io and Keyword.com to produce a single output: a personalized cold email containing a white-labeled Keyword.com ViewKey share link, ready to copy into the user's email client.

The user does **not** have an Apollo custom field for the report URL, and we do not send via Apollo or Gmail. The deliverable is a draft email the user sends manually.

## Prerequisites (check on first run only, cache result)

Read `~/.agents/skills/agency-prospecting/.state.json` if it exists. If `prereqs_ok: true`, skip this section.

Otherwise verify in parallel:

1. **Apollo MCP authenticated** — call `mcp__claude_ai_Apollo_io__apollo_users_api_profile` with `include_credit_usage: false`. If it errors, tell the user to connect Apollo via `/mcp` and stop.
2. **Keyword.com MCP authenticated with write scope** — call `mcp__keyword-com__list_projects` with `limit: 1`. If it errors with `missing_scope`, tell the user to reissue their Keyword.com token with `write:data` (needed for `add_project`/`add_keywords`) and stop.
3. **Keyword.com white-label** — the skill confirms the share link's branding at **Step 10b**, after the project is created and we have the actual URL. No upfront check needed: an early check would require a hypothetical URL or asking the user to paste an existing share-link, and the real verification only matters at the moment we hand the URL to the prospect.

On success, write `{"prereqs_ok": true, "checked_at": "<ISO date>"}` to `.state.json`.

## Workflow

Run these steps in order. Each step's output feeds the next — do not parallelize. Surface checkpoints to the user as plain questions; do not use AskUserQuestion unless the choice is genuinely multi-select with discrete options.

### Step 1 — Find the prospect

Offer the user **two ways** to describe the prospect — they pick whichever is easier:

**Option A — Free-form description (default, preferred):**
*"Describe the kind of prospect you're looking for in plain English. Example: 'Head of Marketing at a B2B SaaS company in NYC, 50–200 employees, ideally in fintech.' Or just paste a target company URL if you have a specific company in mind."*

Parse the description into Apollo search parameters. Map natural-language phrases to API filters:

| Phrase pattern in description | Apollo filter |
|---|---|
| Job titles ("VP of Marketing", "Head of SEO", "CMO") | `person_titles` |
| Seniority words ("director-level", "C-suite", "manager", "senior") | `person_seniorities` (allowed: `senior`, `manager`, `director`, `vp`, `c_suite`, `head`, `owner`, `partner`) |
| Industry/sector ("SaaS", "ecommerce", "fintech", "legal") | `q_organization_keyword_tags` |
| Location ("NYC", "in California", "United States") | `person_locations` |
| Employee range ("50-200", "small companies", "enterprise") | `organization_num_employees_ranges` — e.g. `["51,200"]`, `["1,10"]`, `["10001,20000"]` |
| Specific domain/URL ("at stripe.com", company URL) | `q_organization_domains_list` |
| Technology stack ("uses Hubspot", "on Salesforce") | `currently_using_any_of_technology_uids` (underscores replace spaces/periods) |

If the description is missing a critical filter (no title at all, no industry at all), ask **only** for the gap — do not re-ask anything the user already specified. Echo back the parsed filters before running so the user can correct any misreads: *"I'll search for: titles=[...], seniority=[...], industry=[...], location=..., size=... — go?"*

**Option B — Structured prompts (if the user prefers, or after a failed parse):**
Ask one at a time: job titles, seniority, industry/keywords, location, company headcount range, and (optional) a specific company domain.

After filters are confirmed, call `mcp__claude_ai_Apollo_io__apollo_mixed_people_api_search` with `per_page: 10`.

Show the results as a numbered list: `#. Name — Title @ Company (location)`. Ask the user to pick one by number, or refine the search (loosen titles, expand seniority, etc. — the agent should suggest specific loosenings if the result set is small or low-quality).

Capture from the picked contact:
- `id`, `first_name`, `last_name`, `title`, `email`, `email_status`, `linkedin_url`
- `organization.id`, `organization.name`, `organization.website_url`, `organization.primary_domain`

### Step 1b — Reveal email if needed

The skill's deliverable is a draft email the user sends manually, so the contact must have a usable email address. Check the picked contact's email:

- If `email` is missing or `email_status` is `"unverified"` / `"unavailable"` / null: call `mcp__claude_ai_Apollo_io__apollo_people_match` with the contact's `id` (or `first_name` + `last_name` + `organization.primary_domain`) to enrich. **This costs 1 lead credit** — tell the user the cost first and ask permission, unless they've already opted in for this run.
- If `email_status` is `"verified"`: continue.
- If `email_status` is `"likely to engage"` or similar partial-confidence: surface the status to the user but continue.
- If `apollo_people_match` still returns no email: stop. Tell the user this contact has no findable email and offer to go back to Step 1 and pick another.

### Step 2 — Capture organization data

The `apollo_people_match` response from Step 1b already includes a full `organization` block — do **not** call `apollo_organizations_enrich` separately, it's redundant and wastes the user's API budget.

Capture from `response.person.organization`: `industry`, `keywords`, `estimated_num_employees`, `annual_revenue_printed`, `city`, `state`, `country`, `short_description`, and `technology_names`.

Note: `apollo_people_match` does **not** return competitor or similar-company lists. The skill handles competitor identification by asking the user directly in Step 5 — do not try to scrape competitors from the keywords array (they're audience/positioning tags, not real competitors).

### Step 3 — Scrape the homepage

Use `WebFetch` on `https://{primary_domain}` with a prompt asking the model to extract: page language (`<html lang>`), the H1, the main value proposition (1 sentence), the 3–5 most prominent product or service terms, and any "for [industry]" / "serving [city]" copy that signals local intent.

If the homepage fetch fails (timeout, redirect loop, blocked), fall back to `apollo_organizations_enrich`'s `short_description` and `keywords` — note this gracefully and continue.

### Step 4 — Propose seed keywords

Based on the homepage extract + org enrichment, propose **3–5 candidate seed keywords**. Each must be:
- A broad commercial term someone would search to find this kind of business
- NOT branded (does not contain the company name)
- Aligned with the prospect's primary product or service

Show them as a numbered list with a one-line rationale per seed. Ask the user to pick a number, or type a custom seed.

### Step 4b — Pick keyword count

Ask the user: *"How many keywords should the project track? (default 25, range 5–100)"*

- Default to **25** if the user replies with "default", presses enter, or doesn't answer.
- Clamp to the range 5–100. If they ask for >100, warn that Keyword.com plans usually price per tracked keyword and confirm before continuing.
- Save the chosen number as `N` — used in Steps 5, 6, 8, and 9.

### Step 5 — Generate the keyword list

Call `mcp__keyword-com__suggest_related_keywords` with the chosen seed. Default to `limit: 100` if the tool supports it.

Build a **branded-term blocklist**:
- The company name (and obvious variations: with/without "Inc", "Ltd", "Co", trailing TLD, kebab/space variants).
- Product/brand names extracted from the homepage scrape (Step 3).
- Competitor names: **ask the user** for 2–3 known competitor brand names. Apollo's people-match response does not include competitor lists, so this is always a direct question to the user — never silently skip it.

Filter the returned suggestions:
- Drop any keyword whose tokens (case-insensitive, normalized) contain a blocklist term.
- Drop branded queries the suggester missed (heuristic: contains a trademark symbol, contains "vs", or matches a known-brand pattern in the user's own confirmed list).

Sort remaining by a balance of MSV and commercial intent (prefer competition between 0.2 and 0.7 over very low / very high). Take the top **N** (from Step 4b).

Present them as a compact table with columns: `#`, `keyword`, `MSV`, `CPC`, `competition`. Total below the table.

### Step 6 — Checkpoint 1: approve keyword list

Ask: "Approve these N keywords, swap specific rows, or regenerate?" Handle:
- Approve → continue
- Swap → user gives row numbers to drop and either explicit replacements or "find me more" (re-call suggester with a tweaked seed or pull from the unused-but-filtered list)
- Regenerate → loop back to Step 4 (different seed) or Step 5 (same seed, different filter)

Do not move on until the user explicitly approves.

### Step 7 — Decide tracking geography

Apply this heuristic to the org enrichment:

**City-level tracking** if BOTH:
- `estimated_num_employees < 50`, AND
- `industry` matches a local-service category: legal, dental, healthcare practice, plumbing, HVAC, electrical, restaurant, retail (physical), real estate, salon/spa, fitness/gym, automotive service, home services, trades, local agency.

**National tracking** otherwise.

Resolve the region by calling `mcp__keyword-com__list_tracked_regions` and matching on the org's `country` (for national) or `city` + `country` (for local). If no exact city match exists in tracked regions, fall back to national and note it.

Print the decision with reasoning: e.g. *"Going national (US): SaaS, 200 employees — not a local-service business."* Ask the user to confirm or override.

### Step 8 — Checkpoint 2: approve project creation

Show the user:
- Project name: `{Company name} — Rankings` (truncate to 60 chars)
- Tracked region (from Step 7)
- Keyword count: N (from Step 4b)
- Estimated keyword-credit cost if the tool reports it

Ask: "Create this project in Keyword.com?" Wait for explicit yes.

### Step 9 — Create project and add keywords

In sequence (not parallel — the project_id from step 1 feeds step 2):

1. `mcp__keyword-com__add_project` with the name, region, and any defaults.
2. `mcp__keyword-com__add_keywords` with the N keywords and the new `project_id`.

If either call fails, surface the error verbatim and stop — do not retry silently.

### Step 10 — Get the share link

Call `mcp__keyword-com__get_project_sharing_settings` with the new `project_id`. Extract the public share URL.

If sharing is disabled by default on the user's account, the call may return no URL — in that case ask the user to enable public sharing on the project in Keyword.com UI, then re-run this step (don't go all the way back).

### Step 10b — Confirm share-link branding

This is THE branding gate. Do not move to the email draft until the user has explicitly approved what the prospect will see.

1. Fetch both levels of branding:
   - Call `mcp__keyword-com__get_account_sharing_settings` for account-wide config.
   - Use the `get_project_sharing_settings` response from Step 10 for the per-project overrides (no need to re-call).

2. Compute the **effective branding** the prospect will actually see:
   - If `project.branding.override_account_branding` is `true` → use the per-project values (`company_name`, `company_link`, `company_logo`, `company_description`).
   - Otherwise → use the account-level values (`branding.company_name`, `branding.company_url`, `branding.company_logo_url`, `branding.company_description`).
   - If account-level `domain.use_whitelabel` is `false` → branding fields don't apply at all; the link renders with Keyword.com defaults. Surface this clearly.

3. Display the effective branding in a clear block:

   ```
   Share URL:       {share_url}
   Brand name:      {effective.company_name}
   Brand URL:       {effective.company_url|company_link}
   Brand logo URL:  {effective.company_logo|company_logo_url}
   Description:     {effective.company_description}
   White-label on:  {domain.use_whitelabel}
   Source:          {"per-project override" | "account-level" | "Keyword.com defaults (whitelabel off)"}
   ```

4. Flag obvious issues automatically — don't be silent:
   - Typos in description (mid-word case changes like `iS`, double spaces, trailing punctuation issues)
   - `company_name` doesn't match the hostname in `company_url` (e.g. name says "Acme" but URL is `example.com`)
   - Missing logo / description / URL
   - `use_whitelabel: false` (no branding applied)
   - Share URL host is `app.keyword.com` rather than a custom domain (note this is fine for most users; only call it out if the brand_name suggests they'd want a custom subdomain)

5. Ask: *"This is what {first_name} will see when they click the link. Approve to draft the email, or tell me what to change."*

6. Handle responses:
   - **Approve** → continue to Step 11.
   - **Change account-level branding** (collect new company_name / URL / logo / description) → call `mcp__keyword-com__update_account_sharing_settings`, then re-fetch and re-display. Loop.
   - **Change just this project** (override account branding for this one prospect) → call `mcp__keyword-com__update_project_sharing_settings` with the new values + `project_id`. Re-fetch and re-display. Loop.
   - **Enable white-label** (if currently off) → call `update_account_sharing_settings` with `use_whitelabel: true`. Re-fetch.
   - **Visual check** → tell the user to open the share URL in a browser, verify the page renders with the expected logo / colors / name. Wait for their yes/no.

7. Never proceed to Step 11 with branding the user hasn't explicitly approved. The whole point of the skill is the branded report — sending the wrong-brand link to a prospect defeats the purpose.

### Step 11 — Draft the email

Compose a cold email using these inputs:
- Contact: first name, title
- Company: name, industry, employee size band
- Seed keyword from Step 4
- One specific observation from the homepage scrape (Step 3) — quote or paraphrase a phrase that proves you actually looked
- The ViewKey share URL from Step 10
- A two-option CTA: either "happy to walk through the rankings I'm seeing" or "if useful I can run a deeper analysis on more terms"

Constraints on the draft:
- Subject line under 60 chars, no clickbait, references the company or their category
- Body under 130 words
- No "I hope this email finds you well" or other LLM-cold-email tells
- No em dashes that look auto-generated; use commas or short sentences
- Sign off with a generic placeholder `[Your name]` — the user will fill it in
- Don't claim specific ranking positions you haven't verified — phrase as "I pulled a quick report on N terms in your space" (substitute the actual N from Step 4b)

Output the subject and body in two adjacent code blocks so they're easy to copy independently.

### Step 11b — (Optional) Tag the contact in Apollo

After drafting the email, offer to tag the contact in Apollo with a label so the user can find/filter them later.

Ask: *"Want to tag this contact in Apollo with a label so you can find them later? (yes / no / type a custom label name — default is `agency-prospecting`)"*

If declined → skip to Step 12.

If accepted:

1. **Find the contact in the user's Apollo team account.** Call `mcp__claude_ai_Apollo_io__apollo_contacts_search` with `q_keywords` set to the contact's name + organization (e.g. `"Jeff Culkin Culkin Plumbing"`).

2. **Handle the search result:**
   - **Exactly one match** → continue to step 3.
   - **Zero matches** → the prospect came from `apollo_mixed_people_api_search` (the People API), which returns *persons* from Apollo's global database, not *contacts* in the user's team account. The Apollo MCP can't tag a person who isn't a saved contact. Tell the user: *"This person isn't saved as a Contact in your Apollo team account, so I can't tag them via the MCP. To track them, add them manually in Apollo (Contacts → Add Contact). Skipping the tag step."* Continue to Step 12.
   - **Multiple matches** → list them with company + email and ask the user which one is the right person.

3. **Read existing labels.** The contact record returns a `label_ids` array (e.g. `["689aec1fbf48ad00150de831"]`). Important: `apollo_contacts_update` takes `label_names` (string names, not IDs) and **overwrites the entire list** of labels. There is no append-only mode in the MCP, and there is no MCP tool to resolve label IDs → label names.

4. **Warn before overwriting.** If `label_ids` is non-empty, tell the user: *"This contact has N existing label(s) on the Apollo record. The MCP can only overwrite, not append. If I save just the new label, the existing labels will be replaced. Options: (a) proceed and lose existing labels, (b) paste the existing label names so I can preserve them in the update, (c) cancel."*

5. **Call the update.** `mcp__claude_ai_Apollo_io__apollo_contacts_update` with the contact's `id` and `label_names: [<chosen label>, ...any preserved labels>]`.

6. **Confirm.** Echo the response back. Surface any error verbatim — do not retry silently.

### Note: Reading custom fields (informational)

The Apollo MCP does **not** support writing to custom fields (`typed_custom_fields`) — `apollo_contacts_update` has no slot for them. However, `apollo_contacts_search` *does* return `typed_custom_fields` keyed by field ID. If the user has a custom field they want to read in future skill iterations (e.g. "has this prospect already received a Keyword.com live report?"), the skill can search and inspect the `typed_custom_fields` map. Field IDs are account-specific — there is no MCP tool to look up field-ID-by-label, so the user would need to provide the ID once.

### Step 12 — Summary

Print one final block with:
- **Contact:** Name, title, company, email
- **Apollo contact ID:** `<id>` (so the user can find them later)
- **Keyword.com project:** name + project_id + region
- **ViewKey URL:** the link
- **Draft email:** "above ↑"

Stop. Do not offer to send the email or do anything else. The user takes it from here.

## Failure modes — handle these explicitly

- **Apollo search returns 0 results:** suggest broadening criteria (drop seniority filter, expand titles, remove location). Do not invent results.
- **Org block in people-match returns null fields:** continue with whatever fields are present. Mark missing ones as "unknown" and ask the user to fill in for the geo step if industry is missing.
- **Homepage fetch blocked (403/Cloudflare):** fall back to org enrichment text. Note it.
- **`suggest_related_keywords` returns <N unbranded results:** show the user what you have, ask whether to (a) loosen filters, (b) try a different seed, (c) combine multiple seeds, or (d) proceed with fewer keywords.
- **Project creation fails with `missing_scope`:** user's Keyword.com token doesn't have `write:data`. Stop and explain.
- **Sharing settings return no public URL:** see Step 10.

## State file

Path: `~/.agents/skills/agency-prospecting/.state.json`

Stores only:
```json
{
  "prereqs_ok": true,
  "checked_at": "2026-05-19T12:00:00Z"
}
```

Never store contact, project, or keyword data here — those are one-shot per run.
