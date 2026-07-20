# project-setup

A Claude Code skill that turns a single **site URL** into a fully populated Keyword.com tracking project — keywords grouped by topic (each topic is a tag) plus a dedicated `branded` tag for brand-defense tracking.

A sibling of [`agency-prospecting`](../agency-prospecting): that skill finds and pitches a cold prospect; this one sets up a live tracking project for any site — your own or one you manage.

## Who is this for

- **Getting started with keyword tracking** — you want a fast overview of how your website ranks for high-value keywords, without hand-building a project keyword by keyword.
- **Agencies onboarding a new client** — stand up an organized, topic-tagged tracking project for a client's site in a couple of minutes.
- **Marketers, SEOs, and site owners** setting up (or re-organizing) rank tracking for a site they own or manage.

All you need is a Keyword.com account (a free trial works — 100 keywords, 14 days) and the Keyword.com MCP connected. See **[Prerequisites](#prerequisites)** below.

## What it does

1. **Build the topic list — two ways:**
   - **Manual — you bring the topics.** Tell it the areas you want to rank for; it validates and refines them into well-formed, non-branded seeds.
   - **Automated — it reads the site.** Sitemap-first crawl (`sitemap.xml` / `robots.txt`, locale-pinned), then `WebFetch` the high-signal pages (home, features, solutions, industries, pricing) to infer what they sell and who they serve. It also **mines the blog** (from the sitemap, or the `/blog` listing if the blog isn't in the sitemap) and clusters recurring post themes into extra topic candidates the nav alone would miss — surfaced for your approval, never auto-added. No browser dependency.
2. **Confirm topics — two tiers.** **Product topics** (commercial terms your product/landing pages should rank for, from the nav) and **content topics** (informational themes your blog ranks for, from blog mining — tagged `content:`). Both become tags so product vs content ranking can be reported separately. You approve the list, and it asks how many keywords to track in total.
3. **Expand each topic** — run Keyword.com `suggest_related_keywords` one seed at a time, strip branded noise, dedupe across topics, rank by MSV × intent, and take each topic's allocation.
4. **Build a branded bucket** — brand + product terms and high-intent modifiers (`{brand} pricing`, `{brand} reviews`, `{brand} alternatives`, …), tagged `branded` so the site's own name is tracked defensively.
5. **Offer competitor expansion (count-aware)** — after tallying topic + branded keywords against your target, it offers to pull keywords from competitors: to *fill the gap* if you came up short, or to *find stronger, more-winnable terms* to swap in if you're at target. It suggests right-sized peer competitors from its own research (the rival brands that recurred in your topic pools), and you can add or swap your own. Competitors are only a source — every keyword they surface is deduped and filed under one of your existing topics (a new topic is proposed only if a cluster fits none).
6. **Decide tracking geography + devices** — local-service businesses get city-level, B2B/SaaS gets national; and it asks whether to track **desktop, mobile, or both** (both doubles the tracked-keyword count, since each keyword is tracked once per device).
7. **Create the project with tagged keywords** — `add_project` then `add_keywords`, tagging **inline** (each keyword carries its topic tag; tags are created on the fly — no separate tagging pass).
8. **Verify + summarize** — read back per-tag counts and print a project summary.

You approve at three checkpoints: topics, the full keyword list, and project creation.

### Key contrast with `agency-prospecting`

`agency-prospecting` **drops** branded terms — you don't pitch a prospect on their own name. `project-setup` **keeps and tags** them: when tracking a site you own or manage, you want brand-defense tracking from day one.

## Prerequisites

- **Claude Code** (or any Claude Agent SDK client that supports skills)
- **Keyword.com MCP** connected with the `write:data` scope (required for `add_project` and `add_keywords`):
  ```bash
  claude mcp add --transport http keyword-com https://app.keyword.com/mcp
  ```
  Then run `/mcp` and complete the Keyword.com OAuth login (grant `write:data`). The skill checks this on first run and walks you through it if it's missing.
- **A Keyword.com account.** No account yet? Start a free trial — 100 keywords included, 14 days — at [app.keyword.com/users/signup](https://app.keyword.com/users/signup). That's enough for a full first project.

No Apollo, no browser install. The crawl uses only `WebFetch` against the site's public pages.

## Costs per run

- **N tracked-keyword slots** on your Keyword.com plan (N is the keyword budget you set — the skill always asks, range 10–200, and suggests a number scaled to the topic count / site breadth).
- No additional Claude API costs beyond standard message billing.

## Installation

`project-setup` ships as a **plugin** (so it works in the Claude Code app, not just the CLI) and **bundles the Keyword.com MCP** — installing it registers `keyword-com` automatically, and you just complete the OAuth login. (If you install several Keyword.com plugins, Claude Code dedupes the server to a single connection.)

### CLI

```bash
# 1. Add this marketplace
claude plugin marketplace add benjamin-thorn/keyword-com-skills
# 2. Install the plugin
/plugin install project-setup@keyword-com-skills
# 3. Authenticate the bundled Keyword.com MCP (grant write:data)
/mcp        # select keyword-com → Authenticate
```

### Desktop / web app

1. **+** button → **Plugins** → **Add marketplace** → `benjamin-thorn/keyword-com-skills`
2. Install **project-setup**
3. Run `/mcp` → **keyword-com** → **Authenticate** (Keyword.com OAuth, grant `write:data`)

Don't have a Keyword.com account? Start a free trial (100 keywords, 14 days) at [app.keyword.com/users/signup](https://app.keyword.com/users/signup) first — the OAuth step needs one.

## Usage

In Claude Code, run the namespaced command:
```
/project-setup:project-setup
```
…or just say *"set up rank tracking for example.com."*

Give it the site URL; the skill asks for country/language, device(s), and a keyword budget, reads the site, and walks you through topic approval → keyword approval → project creation.

## Caveats

- **JS-rendered SPAs with no sitemap** return thin content to a plain fetch. The skill falls back to homepage-only and asks you to name topics manually. It does not drive a headless browser by default.
- **Topics become tag names** — keep them human-readable (the skill enforces ≤100 chars). Fixing tags after creation uses `attach_tag`.
- **`add_project` is idempotent by name.** If a project with the same name already exists, the skill confirms with you before adding keywords into it, so you don't pollute an existing project.
- **Geo precision lives in the keywords, not the tracking region.** Organic-SERP tracking is country-level (e.g. `google.co.uk`); true city-level tracking needs Maps-pack (`type: maps`) with a Google Business Profile ID.
- **Branded MSV is often near-zero** in the keyword suggester, so branded terms are composed deterministically from the brand/product names rather than pulled purely from `suggest_related_keywords`.

## License

MIT — see [`LICENSE`](../LICENSE).
