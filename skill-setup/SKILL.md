---
name: competitor-radar-setup
description: Set someone up, from scratch, with their own weekly Competitor Radar dashboard. Use when a user wants to start tracking competitors, build their own competitor dashboard, monitor rivals' socials/SEO/subscribers, or "set me up with something like the competitor radar." Runs an interactive Q&A (niche, competitors, which platforms), can auto-discover and recommend competitors, wires up Apify + Firecrawl connectors, builds a branded HTML dashboard, deploys it to a stable Vercel URL, and schedules a weekly/monthly cloud routine to refresh and Slack it. This is the giveaway companion to the `competitor-radar` skill.
---

# Competitor Radar Setup

Turns "I want to track my competitors" into a live, self-refreshing, branded dashboard in one guided session. This skill sets up the machine; the `competitor-radar` skill is the machine.

## Step 1 — Discovery Q&A (interactive)

Ask, one topic at a time, adapting to answers:
1. **Niche / what they do** (so competitor discovery and framing are accurate).
2. **Competitors** — three modes:
   - They name them, or
   - They ask you to **find and recommend** competitors (do it: search YouTube + the web + their niche communities, propose 5-8 with a one-line why each, let them confirm/trim), or
   - A mix (they name a few, you fill the rest).
3. **Platforms to track** — which of YouTube, Instagram, LinkedIn, TikTok, community (Skool/Circle), SEO. Only track what matters to their niche (a local business cares about Google reviews + local SEO; a creator cares about YouTube + shorts platforms).
4. **Brand** — colors, fonts, logo. If they have a design system or a site, extract from it; else use sensible defaults and confirm.
5. **Cadence** — weekly (default, Monday) or monthly (1st). And where to post it (Slack channel, email).

Write their answers into a `config.json` shaped like the `competitor-radar` skill's config (roster + platforms + apify_actors + brand + slack_channel + live_url + deploy_repo).

## Step 2 — Connect the data sources

The dashboard needs scrapers. Get them connected before building.

- **Apify** (social scraping). If not connected, walk them through adding the Apify connector / API token. Then map platforms to these battle-tested actors:
  - Instagram: `apify/instagram-scraper`
  - TikTok: `clockworks/tiktok-scraper`
  - LinkedIn: `harvestapi/linkedin-profile-scraper` (works on the Apify FREE plan; `dev_fusion` does not). Input `{"queries":["<profile-url>"],"profileScraperMode":"Profile details no email ($4 per 1k)"}`, follower count is `followerCount`
  - X/Twitter: `apidojo/tweet-scraper`
  - YouTube (fallback if no YouTube API): `automation-lab/youtube-scraper`
- **Firecrawl** (everything non-social: community member counts from public Skool/Circle pages, SEO from SimilarWeb public pages, any website audit). If not connected, help them get a Firecrawl API key and install the `firecrawl` skill. Firecrawl cannot scrape social; that is why Apify is required too.
- **YouTube** (primary, real): the YouTube Data API via the youtube/vidiq connector, or a Data API key.

## Step 3 — Build the branded dashboard

Reuse the `competitor-radar` skill's `assets/template.html` + `scripts/build_dashboard.py`, restyled to their brand (swap the CSS color/font tokens, keep the structure: Demo/Actual tabs, per-platform columns, expand cards, focus/blur toggle, week-over-week deltas). Gather the first week of real data via the connectors, write `radar_data.js`, run the build, and open it for their approval before deploying.

> [!important] Avatars must be inlined
> Hotlinked `yt3.ggpht.com` (and most social CDN) avatars are ORB-blocked in the browser. Download each at a small size and base64-embed it in the data file. Never `<img src>` a remote social CDN URL.

## Step 4 — Deploy to a stable URL

Deploy so the URL never changes across weekly refreshes.
- **First deploy (local):** `vercel deploy --prod` is fine from a laptop. Create the project, then **disable Deployment Protection** (`PATCH /v9/projects/<id>` `{"ssoProtection":null}`) so the weekly Slack link is publicly openable.
- **Set up git-connected deploy:** create a small public repo (just `index.html` + the skill), and link it to the Vercel project (`POST /v9/projects/<id>/link` `{"type":"github","repo":"owner/name"}`). From now on, a push to `main` auto-deploys the same URL. This is what makes the routine work.

## Step 5 — Schedule the refresh routine

Create a cloud routine (RemoteTrigger) on the chosen cadence: weekly `30 3 * * 1` (Mon 09:00 IST) or monthly `30 3 1 * *` (1st, 09:00 IST). Cron is UTC; 09:00 IST = `30 3`. It clones the repo, runs the `competitor-radar` skill to refresh the data, deploys, and Slacks the link.

> [!important] The connector vs. embedded-API decision (determined empirically, 2026-07-08)
> This is the crux of making a cloud routine actually work. Two publishing/data rules that cost real debugging to learn:
>
> **1. Deploy: git push, never the Vercel CLI/token/API from a cloud routine.** A cloud routine's Vercel egress resolves to the wrong account by design, so `vercel deploy` and `api.vercel.com` silently hit the wrong place. The reliable path is: repo git-connected to Vercel, routine commits to `main`, Vercel auto-builds. The git push IS the deploy. (Locally, the CLI is fine.)
>
> **2. Data + messaging: prefer http MCP connectors; fall back to embedded API calls only where no connector exists.**
> - **Attach connectors** by leaving `mcp_connections: []` in the routine's `job_config` — an empty list grants ALL of the user's claude.ai connectors (Slack, Firecrawl, Gmail, Vercel-mcp, etc.). Use these: they authenticate via the user's account, so no secrets sit in the routine prompt.
> - **Plugin-based tools are NOT http connectors.** Apify and the youtube/vidiq connector ship via a plugin/marketplace, so they are absent from a fresh cloud routine even with `mcp_connections: []`. For those, either (a) attach the plugin marketplace to the routine, or (b) embed a direct API call. The Apify run API works cleanly from the cloud in one Bash call: `curl -s -X POST "https://api.apify.com/v2/acts/<actor>/run-sync-get-dataset-items?token=$APIFY_TOKEN" -H "Content-Type: application/json" -d '<input>'` (actor id uses `~` not `/`, e.g. `apify~instagram-scraper`). Route YouTube through an Apify YouTube actor when the youtube connector is absent. If you embed keys, do the whole call in a SINGLE Bash invocation, because env vars do NOT persist between Bash calls in a cloud routine. Put the token in the routine PROMPT (private), never in a public repo `.env`.
> - **Never split a credential-dependent call across two Bash calls.** The second call will have lost the env and silently fall back to the wrong identity.
>
> Rule of thumb: connector when the service has a claude.ai http MCP (clean, no secrets); embedded single-Bash-call API when it does not (Vercel is the exception: git push, not API).

## Step 6 — Hand off

Give them: the live URL, the repo, the routine id and its next run time, and a one-paragraph "how to add/remove a competitor" note (edit `config.json` roster, the next run picks it up). Confirm the first refresh by triggering one manual run and checking the Slack post lands.

## Reference implementation

The working, deployed example is `insinexzy/benai-competitor-radar` (live at `https://benai-competitor-radar.vercel.app`), refreshed by routine `Competitor Radar — Weekly Refresh (Mon 9:00 IST)`. Clone its `skill/` folder as the starting point rather than rebuilding from scratch.
