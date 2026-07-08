---
name: competitor-radar
description: Build and weekly-refresh a branded HTML "Competitor Radar" dashboard that tracks a roster of competitors across YouTube, Instagram, LinkedIn, TikTok, community, and SEO. Use when the user wants to track competitors, monitor competitor socials/engagement/subscribers, build a competitor dashboard, or refresh the weekly competitor report. Gathers real data via connectors (YouTube, Apify, Firecrawl), renders a two-tab (Demo/Actual) dashboard in Ben AI branding, deploys it to a stable Vercel URL via git push, and posts the link to Slack.
---

# Competitor Radar

Tracks a fixed roster of competitors weekly and renders one branded HTML dashboard: follower counts, posting cadence, median engagement, week-over-week subscriber growth, standout post of the week, and SEO, per platform. Two tabs: **Demo** (a sample niche) and **Actual** (real competitors, with a focus/blur toggle so only "you" shows on camera).

## Architecture (why it is split this way)

- `config.json` — the stable roster: who to track, their handles per platform, and which platforms to include. Edit this to add/remove competitors.
- `radar_data.js` — the weekly artifact: `WEEK_ENDING`, `AV` (base64 avatars, stable), and `DATA` ({demo, actual}) with the numbers. The refresh step rewrites this.
- `assets/template.html` — the fixed dashboard shell (CSS, render logic, tabs, focus mode). Never regenerate; it carries a single `/*__RADAR_DATA__*/` marker.
- `scripts/build_dashboard.py` — deterministic: template + radar_data.js → `index.html`. No network.

## Weekly refresh workflow

1. **Read** `config.json` (roster) and the current `radar_data.js` (last week's numbers, needed for week-over-week deltas).
2. **Gather fresh data** for each creator (do NOT fabricate; mark missing as `null`). Locally, use the Apify / Firecrawl / youtube MCP connectors. **In a cloud routine those are local plugins and are NOT available** — call the Apify HTTP API directly (the routine injects an `APIFY_TOKEN`), doing each run + parse in ONE Bash call (env does not persist between Bash calls in cloud): `curl -s -X POST "https://api.apify.com/v2/acts/<actor>/run-sync-get-dataset-items?token=$APIFY_TOKEN" -H "Content-Type: application/json" -d '<input>'` (actor id uses `~` not `/`). Route YouTube through an Apify YouTube actor when the youtube connector is absent. Sources:
   - **YouTube** (youtube / vidiq connector locally; Apify `streamers~youtube-scraper` or WebFetch the `/about` page in cloud): subscribers, uploads in last 7 days (title + views), median views + median engagement on last ~10 videos, standout video of the week.
   - **Instagram** (Apify `apify/instagram-scraper`): followers, posts last 7d, avg engagement.
   - **TikTok** (Apify `clockworks/tiktok-scraper`): followers, posts last 7d, avg engagement.
   - **LinkedIn** (Apify `dev_fusion/Linkedin-Profile-Scraper` + `harvestapi/linkedin-profile-posts`): followers, posts last 7d, avg engagement.
   - **Community** (Firecrawl on the public Skool/Circle page): member count.
   - **SEO** (Firecrawl on the SimilarWeb public page for the domain): est. monthly visits + global rank. If SimilarWeb blocks, set `null`.
3. **Compute deltas**: set each creator's `youtube.prev` to last week's `youtube.subs` before overwriting `subs`, so the demo tab shows week-over-week growth. (Actual tab omits deltas; it is blurred on camera.)
4. **Rewrite** `radar_data.js` with the new `WEEK_ENDING` and refreshed `DATA`. Keep `AV` (avatars) unchanged unless a creator is new (then fetch their YouTube avatar, downscale to `=s176`, and base64-embed it — hotlinked `yt3.ggpht.com` URLs are ORB-blocked, so they MUST be inlined).
5. **Build**: `python3 scripts/build_dashboard.py <skill_dir> ../index.html`.
6. **Deploy** (see below) and **Slack** the live URL.

## Deploy — git push, NOT the Vercel CLI

The Vercel project is git-connected to `insinexzy/benai-competitor-radar` (branch `main`); a push to `main` auto-builds production at the stable URL `https://benai-competitor-radar.vercel.app`. **The URL never changes.**

> [!important] In a cloud routine, never use the Vercel CLI, a Vercel token, or api.vercel.com. Cloud egress resolves to the wrong Vercel account by design. The git push IS the deploy. (Locally, `vercel deploy --prod` works and is fine for first setup.)

```bash
cd <repo checkout>
git config user.name "insinexzy"
git config user.email "200930792+insinexzy@users.noreply.github.com"   # unknown authors are blocked by Vercel
# overwrite index.html with the freshly built file, then:
git add index.html radar_data.js
git commit -m "Competitor radar refresh <YYYY-MM-DD>"
git pull --rebase origin main && git push origin HEAD:main
# verify: poll curl -s "https://benai-competitor-radar.vercel.app/?v=<n>" until HTTP 200 shows the new week
```

## Slack

Post to `#team-chat` via the Slack connector: a short line plus the live URL, e.g. `Competitor Radar refreshed for week ending <date>: https://benai-competitor-radar.vercel.app`.

## Guardrails

- YouTube numbers are the reliable core; always real. Never invent socials/SEO — `null` and move on.
- Never edit the template shell, CSS, or render JS. Only `radar_data.js` (and `config.json` when the roster changes).
- Ben AI voice, no em dashes.
