---
name: project-setup
description: Set up a new Keyword.com tracking project end to end. Build the topic list one of two ways — the user supplies the topics they want to rank for (the skill refines them), or the skill reads the site (sitemap + WebFetch) to infer what they sell, their features, and who they serve. Either way it expands each topic into real MSV-backed keywords, asks how many keywords to track in total, then creates a tracking project with every keyword tagged by its topic, plus a dedicated `branded` tag for brand-defense terms. Use when the user asks to "set up a project" / "set up rank tracking for a site", "track a new site" / "start tracking {domain}", "create a new project", "build a keyword list for {site}", "what should I track for {site}", "onboard a client", or runs `/project-setup`.
---

# Project Setup

This skill turns a URL (or a user-supplied topic list) into a fully populated Keyword.com tracking project: keywords grouped by topic (each topic is a tag) plus a `branded` tag that tracks the site's own brand terms defensively. It works for any site — your own or one you manage for someone else. It's a sibling of `agency-prospecting` — same MCP, same house style, but the deliverable is a live, tagged project rather than a cold email.

Key contrast with `agency-prospecting`: that skill **drops** branded terms because you don't pitch a prospect on their own name. Here you **keep and tag** them — when tracking a site you own or manage, you want brand-defense tracking from day one.

## Prerequisites — connect Keyword.com first (check on first run, then cache)

This skill runs entirely on the Keyword.com MCP, so the connection must be live with write access before anything else. Read `~/.agents/skills/project-setup/.state.json` if it exists; if `prereqs_ok: true`, skip this section.

Otherwise verify by calling `mcp__keyword-com__list_projects` with `limit: 1`, and handle the outcome:

- **Succeeds** → connected with read + write. Cache `{"prereqs_ok": true, "checked_at": "<ISO date>"}` to `.state.json` and continue.
- **The `keyword-com` tools aren't available at all** (MCP not added) → tell the user to add it in Claude Code, then stop until they confirm:
  ```
  claude mcp add --transport http keyword-com https://app.keyword.com/mcp
  ```
  Then run `/mcp` and complete the OAuth login for Keyword.com. (Other clients: add the same HTTP endpoint `https://app.keyword.com/mcp` in their MCP settings.)
  - **No Keyword.com account yet?** The OAuth login needs one. Point the user to start a **free trial — 100 keywords included, 14 days** — at **https://app.keyword.com/users/signup**, then come back and add the MCP. (A free-trial account is plenty for a first project: this skill defaults well within 100 keywords.)
- **Errors with `missing_scope`** → connected, but the token lacks write access. Tell the user to reconnect with the **`write:data`** scope (required for `add_project` / `add_keywords`): re-run the OAuth via `/mcp`, or reissue the API token in **Keyword.com → Settings → API** with `write:data`, then reconnect. Stop until fixed.
- **Any other auth error** → tell the user to reconnect via `/mcp` and stop.

Do not proceed past this section until `list_projects` succeeds.

## Workflow

Run in order. Each step feeds the next — do not parallelize the pipeline. Surface checkpoints as plain questions; only use AskUserQuestion when the choice is genuinely multi-select with discrete options.

**Keep the UX fluid.** Present the result of a step and the next decision, then stop. Do not narrate your own method, caveats, or meta-commentary ("honest note", "this is the trap", "our method", explanations of why a tool behaves as it does). Do not expose internal workflow scaffolding — no "Checkpoint 1 of 3", "Step 5", or phase labels; just ask the decision plainly. The user wants a smooth setup flow, not a play-by-play. If something genuinely needs flagging (a thin topic, a filtered-out set), say it in one short line and move on.

### Step 1 — Site, locale, and input mode

Open with a short, energizing framing of what they're about to get — not a bare checklist. Aim for a sentence or two like: *"Give me a URL and I'll turn it into a ready-to-track Keyword.com project — I'll work out what you should rank for, pull the real high-volume keywords, group them by theme, add brand-defense terms, and set the whole thing up. Bare domain to live rank tracking in a couple of minutes."* Then ask:

- **Site URL** (required in both modes — it's the `tracking_url` for every keyword and the source of the brand name for the `branded` tag). Normalize to a bare domain (`https://{domain}`) and keep both the full URL and the registrable domain.
- **Target country + language** (default `US` / `en`). These feed `suggest_related_keywords` and region selection.
- **Device(s)** — track on **desktop**, **mobile**, or **both**? Rankings often differ by device, so **both** gives the fullest picture — but flag the cost: **tracking both doubles the tracked-keyword count** (each keyword is tracked once per device), so a 45-keyword list on both devices consumes 90 slots against the plan. Suggest a sensible default and let the user pick: **desktop** for most B2B / SaaS, **mobile or both** for local-service, consumer, or ecommerce sites where mobile search dominates.
- **How should I build the topics?** — explain both paths so the choice is obvious, don't just name them:
  - **Manual** — *you* already have areas in mind. List them in plain words (e.g. "email marketing, automation, CRM") and I'll sharpen each into a proper tracking topic. Fastest when you know your priorities.
  - **Automated** — *I* read the site (pages **and** blog) and propose a full topic set for you to approve: product topics from the nav + content topics from the blog. Best when you want thorough coverage or aren't sure where to start.
- **Keyword budget** — always ask, never silently default: *"How many keywords should we track in total?"* Offer a sensible anchor rather than a fixed number: in Manual mode suggest `~5–8 per topic × their topic count`; in Automated mode suggest a range scaled to how broad the site is (a single-service local business ~15–25; a broad multi-product SaaS 60–120). Clamp to 10–200 and warn above ~100 that Keyword.com bills per tracked keyword. Save as `N`. `N` counts **unique keyword terms**; if the user picked **both devices**, the slots actually consumed are **2 × N** — call that out when confirming.

Echo the scope back before continuing: *"Setting up {domain}, {country}/{language}, {desktop|mobile|both}, {Manual|Automated}, ~{N} keywords. Go?"*

### Step 2A — Collect and refine user topics (Manual)

Take the topics the user listed and **guide each toward a good tracking seed** — do not just accept them verbatim. For each proposed topic, check and coach:
- **Too vague** ("marketing", "software") → propose 1–2 sharper commercial variants ("email marketing software", "marketing automation platform") and let the user pick.
- **Branded** (contains the site or a competitor name) → move own-brand topics into the `branded` bucket; flag competitor-brand topics and ask whether they really want to track the site ranking for a rival's name (usually no).
- **Actually a long-tail keyword, not a topic** ("free email marketing software for nonprofits under 500 contacts") → keep it as a seed but note it will expand narrowly; suggest a broader parent topic alongside it.
- **Duplicate / overlapping topics** → merge and say so.

Produce the refined topic list, then go to Step 3 for approval. Skip the site crawl entirely in this mode (but still derive the brand name from the domain for the `branded` tag; optionally do a single homepage `WebFetch` only to confirm brand/product names).

### Step 2B — Read the site (Automated — sitemap-first, WebFetch for content)

Goal: recover the site's information architecture and understand what they sell, their features, and who they serve. Do NOT use a headless browser — this is a lightweight, dependency-free crawl.

**Pin the locale first.** Many sites geo-redirect by visitor IP, so `WebFetch` may resolve to the wrong country. Force the target locale: prefer the country/language path if the site uses one (e.g. `/en/`, `/us/`), and tell the `WebFetch` prompt to report the `<html lang>` it actually got. If the returned `lang` doesn't match the target language, retry with an explicit locale path — do not distill topics from the wrong-language page.

1. **Find the sitemap.** Try in order, stopping at the first that returns XML:
   - `https://{domain}/sitemap.xml`
   - `https://{domain}/sitemap_index.xml`
   - Parse `https://{domain}/robots.txt` for a `Sitemap:` line and follow it.
   If a sitemap index is returned, fetch the first 1–2 child sitemaps. Collect the URL list — this is the raw IA signal.

2. **Select high-signal pages** to read (cap at ~8 fetches to stay fast). Prefer, in this order, whatever exists in the sitemap (match on path substrings):
   - the homepage (`/`)
   - `features`, `product`, `platform`, `capabilities`
   - `solutions`, `use-cases`, `use_cases`
   - `industries`, `who-we-serve`, `for-*` (audience pages)
   - `pricing`
   If there's no sitemap, derive candidates from the homepage nav (sub-step 3 below) instead.

3. **WebFetch each selected page.** Prompt the fetch to extract: the nav/menu labels, the H1 and one-sentence value proposition, the 3–7 most prominent product/feature terms, and any "for {industry/role}" / "serving {audience}" copy that names who they serve. Also capture `<html lang>` and any product/brand names.

4. **Mine the blog for extra topics.** Product pages show what the site *sells*; the blog shows themes they actively *invest in* that the nav misses.
   - **Prefer the blog's own taxonomy.** If the blog exposes category or tag pages (e.g. `/blog/category/...`), use them — that's the company's own clustering, it covers the whole archive, and it's cheaper than reading posts. Only if there's no taxonomy, fall back to pulling post slugs/titles (from the sitemap or the `/blog` listing) and clustering them yourself.
   - **Filter for relevance either way.** Blog taxonomies skew to broad, top-of-funnel content themes that may not map to anything the site can rank-track. Keep themes that map to something the site could realistically track; drop only genuinely useless buckets — overly broad ones (`marketing`, `seo`), `uncategorized` / brand / changelog buckets (`updates`), one-off posts, and competitor-comparison (`x vs y`) posts. Optionally drill into a promising category's posts for a sharper sub-topic.
   - **Overlap with a product topic is NOT a reason to drop a category.** A blog category that shares a theme with a product topic (e.g. product `email-automation` and blog `automation`) is still a valid **content topic** — the two tiers capture different intent (commercial vs informational) and rank different pages, so they're tracked separately. Keep it; just note the overlap to the user rather than silently dropping it. Apply this consistently — never keep one overlapping category while dropping another.
   - Blog-derived themes are **content topics** (a distinct tier — see Step 3), not product topics. Present them at Step 3 in their own group, flagged as blog-sourced — never auto-added.

5. **Fallback.** If there's no sitemap and the homepage fetch is thin (JS-rendered SPA returning near-empty content), say so plainly and continue with homepage-only. Only suggest a Playwright-based crawl if the user asks why coverage is thin — it's an optional heavier path, not part of this skill's default.

Keep a running list of every product name and brand token you see — you'll need them for the branded bucket and the blocklist.

### Step 3 — Distill topics (checkpoint)

Assemble the topic list — **from the crawl (Automated)** or **from the user's refined list (Manual, Step 2A)**. Each topic is a broad, **non-branded** seed keyword representing a product area, use case, or audience the site serves — the kind of term a buyer would search. Examples of good topics: "rank tracking software", "local seo tools", "seo agencies" (an audience), not "AcmeRank Pro" (branded) and not "best" (too vague). In Automated aim for 4–8 topics; in Manual use as many as the user gave (after refinement).

**Topics come in two tiers — keep them distinct:**
- **Product topics** — commercial terms the site's product / landing pages should rank for (`rank tracker`, `local seo tool`). Sourced from the nav (Automated) or the user (Manual).
- **Content topics** — informational themes the site's *blog* ranks for (`keyword clustering`, `google algorithm updates`), sourced from blog mining (Step 2B, item 4). These track content visibility, not product demand.

Both are legitimate and both become tags. Present them in two labelled groups at the checkpoint below, and tag each topic so product vs content can be reported separately (e.g. prefix content-topic tags with `content:` — `content:keyword-clustering`). If a site has no blog or you're in Manual mode, the content tier is simply empty.

Present them as a numbered list. In Automated, add a one-line rationale naming the page(s) that informed each; in Manual, echo the refinement you made (or "as provided"):

```
1. rank tracking software   — from /features (nav: "Rank Tracker", "SERP monitoring")
2. seo reporting tools       — from /solutions ("white-label reports")
3. local seo                 — from /industries ("for local businesses")
...
Brand terms detected: Acme, AcmeRank, Acme SEO  → will be tracked under a `branded` tag.
```

**Checkpoint:** "Approve these topics, edit the list, or add/remove any? The `branded` tag is added automatically." Do not proceed until the user approves. Topics become tag names, so keep them tag-friendly (≤100 chars, human-readable).

### Step 4 — Allocate the budget across topics

Split `N` across approved topics plus the branded bucket. Default split: reserve ~15–20% of `N` for `branded`, distribute the rest roughly evenly across topics (in Automated, weight slightly toward topics that appeared on more pages; in Manual, even split unless the user prioritized some). Tell the user the plan in one line and let them adjust:

*"Plan: 6 topics × ~6 keywords + ~8 branded = 44. Adjust any allocations?"*

### Step 5 — Expand each topic (one seed at a time)

For **each** approved topic, call `mcp__keyword-com__suggest_related_keywords` with `keyword: <topic>`, `language`, `country`, `limit: 100`. Do them one at a time — do not invent keywords.

**Seed content topics with informational modifiers, not the bare theme.** A content topic's bare head term (`link building`) returns commercial/service results the blog won't rank for. Expand it through the **modifier words that signal blog intent**, grouped by how informational they are:
- **Purest — prefer these:** question (`how to`, `what is`, `why`, `when to`), instructional (`guide`, `tutorial`, `step by step`, `for beginners`, `examples`, `basics`, `explained`), actionable (`tips`, `strategies`, `techniques`, `best practices`, `checklist`, `template`, `framework`, `ideas`, `mistakes to avoid`).
- **Mid-funnel — use, but they pull in roundups/reviews:** `best`, `top`, `types of`, `vs`, `alternatives`, `comparison`.
- **Freshness — sparingly, ages fast:** `trends`, `latest`, the current year.

Then two passes, combined:
1. **Modifier-seeded queries.** Match modifiers to how people actually phrase *this* theme — don't bolt on all of them. `link building` → `how to build backlinks`, `link building strategies`, `link building examples`; `content marketing` → `what is content marketing`, `content marketing examples`, `content marketing tips`. Run ~6–8 fitting variants through `suggest_related_keywords`.
2. **Mine the plain-theme pool.** Also run the bare theme once and **keep only returned keywords that already carry an informational modifier**; discard the bare commercial head terms.

Merge, dedupe, and **let real MSV decide** — keep the modifier variants that come back with actual volume, drop the rest. Take the topic's allocation. Product topics keep the plain head-term seed; everything downstream (brand/competitor filtering, cross-topic dedupe, allocation, auto-re-seed) is identical for both tiers.

Build a **brand blocklist** covering two kinds of names:
- **The site's own brand** — company + product names plus obvious variants (with/without `Inc`/`Ltd`/`Co`, trailing TLD, kebab/space forms). These go to the `branded` bucket, not a topic.
- **Competitor brands** — suggestion pools are full of rival brand names, and the site won't rank organically for a competitor's name, so these are noise. Either ask the user for 2–3 known competitors up front, or brand-detect on the fly (a token that looks like a product/company name the site doesn't own) and drop those suggestions.

Also drop **off-intent noise** the suggester mixes in — definitional or unrelated high-MSV terms (dictionary definitions, stock tickers, unrelated homonyms). Do not blindly take top-N by MSV; sanity-check relevance to what the site actually sells.

For each topic's returned ideas:
- **Remove own-brand and competitor-brand matches** from the topic pool (any idea whose normalized tokens contain a blocklist term) — own-brand terms belong only in the branded bucket, never double-counted under a topic; competitor-brand terms are dropped entirely.
- **Dedupe within a tier, not across tiers.** So the same keyword isn't tracked twice, assign it to a single best-fit topic *within* its tier. But do **not** dedupe the content tier against the product tier — they're independent by design (same theme, different intent), so a product topic and a content topic on the same theme both stand, each with its own keywords and tag.
- **Rank** the remainder by MSV and relevance to what the site sells. Do **not** sort by the `competition` field — it's Google *Ads* advertiser competition, not organic ranking difficulty, and optimizing for it is what floats unwinnable category head terms to the top. It's fine to track terms the site doesn't rank for yet; that's how tracking shows progress.
- Take that topic's allocation from Step 4.

Carry provenance for every kept keyword: `{keyword, msv, cpc, competition, tag: <topic>}`.

**Auto-re-seed thin topics (silent).** Measure each topic's *usable* count — what's left after brand/competitor/off-intent filtering and dedupe. If it's below the topic's Step 4 allocation (or below ~3 in absolute terms), re-seed once before moving on — don't accept the scraps and don't stop to ask. Derive 2–4 sharper child seeds and call `suggest_related_keywords` on each:
- **Automated:** seed from the concrete sub-pages / nav items under that theme (the crawl already surfaced them). An `ai search visibility` topic backed by `/chatgpt-tracker` and `/ai-overview-tracker` pages re-seeds as `chatgpt rank tracker`, `ai overview tracker`, `perplexity tracker` — the abstract umbrella term has little volume; the concrete product terms under it do.
- **Otherwise:** decompose the abstract topic into the concrete instances a buyer would actually type (specific product types, tools, sub-categories).

Merge, filter, and dedupe the re-seed results into the topic. **Cap at one re-seed pass per topic.** If it's still short afterwards, it's a genuinely low-volume / emerging niche — silently redistribute the shortfall to richer topics, and flag the thin tag in a single line at Step 7 (do not loop further or interrogate the user mid-run).

### Step 6 — Build the branded bucket

Brand terms usually have low or no MSV in `suggest_related_keywords`, so generate them deterministically from the brand + product names captured in Step 2. Compose:
- the bare brand (`acme`), and product names (`acmerank`)
- high-intent brand modifiers: `{brand} pricing`, `{brand} reviews`, `{brand} login`, `{brand} alternatives`, `{brand} vs`, `{brand} demo`

Optionally also run `suggest_related_keywords` on the bare brand to catch real brand+modifier variants with actual MSV, and merge (dedupe). Take the branded allocation from Step 4. Tag every one `branded`.

### Step 6b — Offer competitor keyword expansion (count-aware)

Competitors are a first-class keyword source, not just a gap-filler. **Always surface this offer — the agent must not skip it, even when already at or over `N`** (only the user may decline). Tally the candidates so far (topics + branded) against `N`, then offer in one short line, framed by the count:

- **Under target:** *"That gives {X} of your {N} target — want me to pull keywords from competitors to close the gap?"*
- **At or over target:** *"We're at your target — want me to check competitors too, in case there are stronger terms to swap in?"*

If the user declines → go to Step 7 with what we have (if short, lower `N` with their OK or note the shortfall in one line). If they accept:

1. **Suggest competitors — right-sized, not category giants.** The peer signal is already in hand: the competitor brand names filtered out of the topic pools in Step 5. A brand that keeps recurring across the site's topic suggestions is competing for the same terms. Rank candidates by how often they recurred, keep them in the site's weight class (a niche brand's peers are other niche brands, not category giants), and present 3–5 with a one-line "why this is a peer." Let the user add, remove, or replace any of them. If no crawl ran (Manual), just ask the user to name 2–3 competitors. Warn if a user-named competitor is a category giant that will drag in broad head terms.

2. **Pull their keywords.** For each confirmed competitor domain, call `mcp__keyword-com__suggest_competitor_keywords`.

3. **Filter and categorize.** `suggest_competitor_keywords` returns the domain's *entire* footprint — often mostly broad/off-niche terms — so expect to discard most of it. Drop the site's brand (→ `branded`), drop other competitors' brand terms, dedupe against everything selected, and keep only terms that map to an existing topic. Competitors are a *source*, not a category — every surviving keyword gets a **topic tag**, same as topic-sourced ones. If a cluster of competitor terms fits none of the current topics, offer to add a **new topic**; never make a separate "competitor" tag. Re-tally toward `N`.

4. If still short after competitors, offer the levers explicitly: accept fewer, add a topic, or loosen filters — do not pad with off-topic terms to hit the number.

### Step 7 — Checkpoint: approve the full keyword list

Present the keywords **grouped by tag**, in tier order — product topics first, then content topics (`content:` tags), then `branded` — one compact table per group with columns `#, keyword, MSV, CPC, competition`. Show a per-group subtotal and a grand total against `N`.

Ask: "Approve this list, swap specific rows, or regenerate a group?" Handle:
- **Approve** → continue.
- **Swap** → user names rows to drop + replacements or "find me more" (pull from that topic's unused-but-filtered pool, or re-seed).
- **Regenerate** → re-run Step 5 for the named topic with a tweaked seed.

Do not move on until the user explicitly approves.

### Step 8 — Decide tracking geography

Reuse the `agency-prospecting` heuristic:

**City-level** if the site is a small local-service business (few locations, category like legal / dental / healthcare / trades / restaurant / real estate / salon / gym / home services). **National** otherwise (SaaS, ecommerce, national brands).

Resolve the region by calling `mcp__keyword-com__list_tracked_regions` and matching the site's country (national) or city+country (local). If no city match exists, fall back to national and note it. Print the decision with reasoning and let the user confirm or override. This region maps to the `region` field on each keyword (default `google.com`).

### Step 9 — Checkpoint: approve project creation

Show:
- **Project name:** `{Company} — Rankings` (truncate to 100 chars; `name` cannot start with `.`)
- **Domain:** the site domain (sets `domain` on the project for self-domain detection)
- **Region** (Step 8)
- **Device(s)** (Step 1) — and if **both**, state the doubled slot count explicitly: "N terms × 2 devices = 2N tracked keywords"
- **Tag breakdown:** each topic tag + count, and `branded` + count; grand total = `N` terms

Ask: "Create this project in Keyword.com?" Wait for explicit yes.

### Step 10 — Create the project and add tagged keywords

In sequence (project must exist before keywords):

1. `mcp__keyword-com__add_project` with `name` and `domain` (and `currency_code` if the user cares about CPC display). It's idempotent by name — `created: false` means a project with that name already existed and was returned as-is; if so, confirm with the user before adding keywords into it.

2. `mcp__keyword-com__add_keywords` with the new `project_id`. **Tag inline** — each keyword entry carries its own `tags: [<its topic or "branded">]`; new tags are created on the fly, so no separate `create_tag` pass is needed. Per entry set:
   - `keyword`
   - `tracking_url: https://{domain}` (the site's own site — this is what we're tracking them ranking for)
   - `tags: [<tag>]`
   - `region` (from Step 8), `language`, and `url_tracking_method: "broad"` (domain-level match — correct for a fresh project; a site ranks with different URLs over time)
   - `type` — set from the Step 1 device choice: `desktop` or `mobile`. For **both**, add each keyword **twice** — one entry with `type: "desktop"` and one with `type: "mobile"` (same keyword, tag, and tracking_url). This is what doubles the slot count, so a 45-term list becomes 90 entries — mind the 100-per-call batch cap and split accordingly.

   **Batching:** `add_keywords` accepts max 100 keywords per call. If `N > 100`, split into batches of ≤100 — each entry keeps its own tag, so batches can mix tags freely.

If either call fails, surface the error verbatim and stop — do not retry silently. If a batch reports duplicates, that's fine (already-tracked keywords are coalesced, not an error).

### Step 11 — Verify and summarize

Confirm the result with `mcp__keyword-com__list_projects` (or `list_keywords` for the project) to read back the per-tag counts. Then print one summary block:

- **Site:** company + domain
- **Keyword.com project:** name + `project_id` + region
- **Tags created:** each topic tag with its keyword count, plus `branded`
- **Total tracked:** N
- **Next:** "Open the project in Keyword.com to review rankings as they populate (first pull takes a few minutes)."

Stop there. If the user wants a shareable white-labeled report, point them at `agency-prospecting`'s Step 10/10b branding flow or offer to fetch `get_project_sharing_settings` — but only on request.

## Correction path — retagging after the fact

If tags need fixing after creation, use `mcp__keyword-com__attach_tag` (`project_id` + `keyword_ids` + `tag_names`/`tag_ids`) to add tags, or `create_tag` to pre-create an empty tag. Foreign keyword/tag ids are silently skipped, so double-check `attached_count`. Inline tagging in Step 10 is the primary path; this is only for repairs.

## Failure modes — handle these explicitly

- **No sitemap and homepage is JS-rendered / thin:** continue homepage-only, say coverage is reduced, and ask the user to name 2–3 topics manually to supplement. Mention Playwright only if they ask why.
- **`suggest_related_keywords` returns `pending` with a `job_id`:** re-call with the same arguments to retry.
- **A topic is thin after filtering:** handled in Step 5 by the auto-re-seed pass (one retry with sharper child seeds). If it's *still* thin after that, silently redistribute the shortfall to richer topics and flag the thin tag in one line at Step 7 — don't interrogate the user mid-run.
- **`add_project` returns `created: false`:** a same-named project already exists — confirm with the user before adding keywords into it (avoid polluting an existing project).
- **`add_keywords` / `add_project` errors with `missing_scope`:** token lacks `write:data`. Stop and explain.
- **No region match in `list_tracked_regions`:** fall back to national `google.com` and note it.

## State file

Path: `~/.agents/skills/project-setup/.state.json`

Stores only:
```json
{ "prereqs_ok": true, "checked_at": "2026-07-20T12:00:00Z" }
```

Never store site, project, or keyword data here — those are one-shot per run.
