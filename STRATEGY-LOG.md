# Strategy log

Every SEO change made to the product behind this playbook, newest first: the date, what changed and why. The chapters in `docs/` are updated in the same commit.

| Date | Change | Why |
|---|---|---|
| 2026-11-18 | Free tools program grown from 10 to 16: six more AI generators around the core job (topic ideas, a timed talk script, opening lines, closing lines, text to slide bullets, audience Q&A prep), each its own indexable page with WebApplication + FAQPage, one shared form component, the same email gate and per-IP and site-wide daily caps; hub, sitemap and llms.txt read the shared tool list | Tools were earning leads and visits, so we widened coverage to adjacent "how do I start/end/script/prepare" searches that students and first-time presenters make; each result links into making the deck, and one component kept each new page to copy and a prompt |
| 2026-11-18 | AI search step 9: WebMCP declarative API (`toolname`, `tooldescription`, `toolparamdescription` on the core create form; no `toolautosubmit` on paid or quota actions; `agentInvoked` / `respondWith` to answer the agent; invocations tracked) | Makes the product's main action callable by in-browser agents with plain HTML, alongside the imperative tool; Chrome's early preview supports it, and unsupported browsers ignore the attributes |
| 2026-11-18 | AI search step 10: Google's Preferred Sources button in the site footer (Google's `publisher.js` loaded lazily, `data-theme` matched to the site theme, CSP allows it, deep-link fallback, click event) | Readers can ask Google to show the site more in Top Stories, AI Overviews, AI Mode and Discover; open to any domain-level site, not just news |
| 2026-11-15 | Off-page step 11: a `/recommends` page of genuinely-used products with dofollow links from one shared list, plus rotating blog shout-outs (2 per product per rolling week, rendered outside the linted body) | Give editorial backlinks to sister products you use or friends build; they notice and often link back, and it reads human, not a link scheme |
| 2026-11-15 | WebMCP: the landing registers a `create_deck` tool via `document.modelContext` (fallback `navigator.modelContext`), feature-detected and a no-op where missing; it writes the brief into the entry form's storage and opens the funnel page | Forward-looking agent-readiness; the client-side counterpart to the hosted MCP server, kept cheap because no shipping browser consumes WebMCP yet |
| 2026-11-11 | Chapter 8, step 9: how to list a remote MCP server on Smithery (by URL or CLI, CIMD, server card if the scan is blocked), Glama (connectors, imported from the official registry) and mcp.so (Remote Server form needs a repository URL) | MCP directories give backlinks and developer discovery. Each lists remote servers differently, and Glama reads the official registry, so the order matters |
| 2026-11-11 | New chapter 8: launching in ChatGPT and Claude (one MCP server for both, OAuth with CIMD and DCR, tool metadata and annotations, golden prompt set, listing copy, review material, MCP Registry). Ranking section built from vendor docs, an OpenAI employee statement, GitHub issue #954, developer forum reports and tool-selection papers | Both assistants now have app directories with in-chat suggestions. Key findings: Claude rank = distinct accounts that connect in 30 days; Claude suggestions show only Verified listings in practice; ChatGPT ranks on conversion, usage and retention; approval is not placement |
| 2026-11-10 | Off-page chapter: added Alternative.me, a regional/niche-directory note, and an "own channels" (X, LinkedIn, portfolio, repo) note to the launch-sites list | Capture every type of place a startup gets posted, kept product-neutral; the private per-listing tracker stays in the app repo |
| 2026-11-07 | Free-tools audit: retargeted the speech tool to "words to minutes" (its real query), strengthened the slide-count FAQ, kept all 10 tool pages indexed (no deletions) | Tools had 1 lead ever; cause was ~0 traffic, not the gate. All pages are indexed with impressions, so deleting would forfeit coverage; retarget to the query users type instead |
| 2026-11-07 | Content Signals line in `robots.txt` (`search=yes, ai-input=yes, ai-train=yes`, all AI use allowed including training), served from a text route instead of the framework robots helper | Cloudflare Agent Readiness flagged "No Content Signals found in robots.txt"; owner wants every AI allowed |
| 2026-11-06 | `WebSite` structured data: homepage URL with trailing slash and an `alternateName` | Google showed the bare domain instead of the brand as the site name above results |
| 2026-11-05 | Video files under `/api/` exempted from the blanket `/api/*` noindex header | The file carried `X-Robots-Tag: noindex`, so Search Console reported "Video not processed" |
| 2026-11-05 | Indexing audit by Performance > Pages diffed against the sitemap instead of the Page indexing report | The report was 13 days stale: 98 of 116 sitemap URLs already had impressions, and the rest were mostly posts from the last few days |
| 2026-11-01 | Verified revenue profile on TrustMRR (Dodo read-only key, GA4, Search Console and GitHub connected, not for sale) | Another trusted page about the product that founders, buyers and AI assistants read |
| 2026-11-01 | Playbook moved to its own public repo and expanded into 7 detailed chapters with working code | So any team can implement it on their own app |
| 2026-10-31 | Medium strategy researched: founder stories through Boost-nominating publications; import only human-edited posts with a canonical; Dev.to and Hashnode for automation | Medium closed its API, nofollows links and limits AI text to followers |
| 2026-10-31 | This playbook created | Reuse the pipeline on other apps |
| 2026-10-30 | `www` now 301s to the apex | The `www` host didn't redirect, so pages were split across two hosts |
| 2026-10-30 | FAQ accordion on the homepage | Shorter page, same FAQPage content |
| 2026-10-29 | Blog writer raised to 2 new posts a day, never refreshing old ones | More pages for long-tail queries while quality gates hold |
| 2026-10-28 | Blog moved from a GitHub workflow into the API (database storage, cron, Search Console topics, funnel-stage rotation) | Posts target real queries we already get impressions for |
| 2026-10-28 | `/alternatives` and `/compare` hubs: an alternatives page per rival, 6 head-to-heads | SEO research: "<rival> alternative" and "a vs b" searches |
| 2026-10-28 | `/templates` gallery, 24 real decks | "<thing> template" searches, and it converts |
| 2026-10-28 | Author page `/about/yash-garg`, linked from every post | E-E-A-T: posts have a real, linked author |
| 2026-10-28 | Embeddable speech timer with a credit link | Earn backlinks from pages that embed it |
| 2026-10-28 | Use-case pages expanded to 23 (audio, transcript, Google Docs, Markdown, CSV, image) | Each input checked in code first; Notion skipped |
| 2026-10-27 | `/tools` with 10 free tools and an email gate | Top-of-funnel traffic, links and leads |
| 2026-10-27 | Video watch pages `/watch/<slug>` | Search Console: "Video isn't on a watch page" |
| 2026-10-27 | Markdown for agents, `/about` and `/contact` trust pages, Organization contactPoint | Agent-readiness audit |
| 2026-10-27 | robots blocks only `/api/` and `/ingest/`; app pages noindexed by header | Blocked pages were getting indexed as bare URLs |
| 2026-10-24 | `/vs` comparison pages for 7 rivals, the `/use-cases` index, and the footer as link lists | Middle-of-funnel searches; internal linking |
| 2026-10-24 | IndexNow script and CI workflow; Search Console and Bing set up | Faster indexing outside Google |
| 2026-10-24 | GA4 loaded after load and idle, production only | Traffic sources without a Lighthouse cost |
| 2026-10-24 | Launch directories and review sites listed in a tracker | Backlinks and referral traffic |
| 2026-10-23 | First `/for/<slug>` use-case pages with real slide renders | Bottom-of-funnel "X to presentation" searches |
| 2026-10-23 | Landing speed: deferred slide mounts, srcset images | Core Web Vitals |
