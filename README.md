# SEO pipeline

A playbook for bringing search and AI-assistant traffic to a small SaaS app. It is written from what MakeDecks (an AI presentation maker, Next.js on Cloudflare Workers) actually built. Copy it, swap in your product, and skip anything that doesn't fit.

The strategy log at the bottom is kept current: every SEO change made to MakeDecks is added there with its date and reason.

## Principles

- **Real pages for real searches.** Every page answers a query that people type. Google penalises scaled, low-value content, so quality beats volume.
- **Show the product, don't describe it.** Use real renders of the product's own output, never mock screenshots.
- **Never state an unchecked fact about a rival.** Competitor claims come only from their own pricing or help pages, with the month you checked. Anything unverified is left out or marked "Not checked".
- **One source of truth.** Lists of pages (use cases, tools, templates, rivals, videos) live in one registry. The routes, sitemap, footer and `llms.txt` all read from it.
- **Every page links into the funnel.** Each page links to related pages and ends with one sign-up CTA, tracked by location.

## 1. Technical foundations

### Domain and canonicals
- Pick one host, the apex (e.g. `https://makedecks.com`), and 301 everything else to it:
  - `www` → apex;
  - extra TLDs → the same path on the apex. On Cloudflare, use a Single Redirect rule so it runs before the worker.
- Keep `SITE_URL` as one constant. Canonicals, the sitemap, Open Graph and JSON-LD all read it.

### Crawling and indexing rules
- `robots.txt` blocks only the API and analytics proxy (`/api/`, `/ingest/`).
  - Allow any API path that serves public media, e.g. `/api/videos/`.
  - Never block app pages in robots. Google then can't see their noindex and may index the bare URL from links.
- App pages (login, dashboard, editor, share links, print, unsubscribe) send `X-Robots-Tag: noindex, nofollow` as a response header (set in `next.config.ts`).
- When the marketing site and the app share one deployment, every new protected path goes in three places: the auth matcher, the noindex list and the route list.

### Sitemap
- `app/sitemap.ts` builds every static page from the registries, then adds dynamic posts at request time (`revalidate` about 5 minutes).
- A new public page only needs to be in the sitemap. Search Console and IndexNow pick it up from there.

### Metadata
- Each programmatic page is its own static route with a static `metadata` export: title (70 characters or fewer), description (70 to 160 characters), canonical, Open Graph and Twitter tags.
- **Framework gotcha (vinext / streaming):** `generateMetadata` gets streamed into `<body>`, which crawlers may miss.
  - For truly dynamic pages such as blog posts, the worker wraps the handler and injects the head tags with `HTMLRewriter`.
  - Keep the streamed copies too, or React hydration breaks.

### Structured data (JSON-LD)

| Page | Types |
|---|---|
| Home | Organization (with `email` and `contactPoint`), WebSite, FAQPage |
| Use-case and comparison pages | FAQPage, BreadcrumbList |
| Hubs (`/use-cases`, `/tools`, `/templates`) | ItemList |
| Free tools | WebApplication, FAQPage |
| Blog posts | Article (author → Person page), BreadcrumbList, FAQPage |
| Video watch pages | VideoObject, BreadcrumbList |
| Author page | Person with `sameAs` links to real profiles |

### Speed and mobile
- Below-the-fold product renders mount within a screen of the viewport, inside a box with the final size (`aspect-video`) so nothing shifts.
- Images use `srcset` with 720 and 1080 widths.
- Analytics scripts that cost Lighthouse points (GA4) load after `window.load` and an idle callback, and only on the production host.
- Check every page at 360, 390 and 414 px wide, with no horizontal scroll.
- Keep at least 5% text to markup on the homepage: decorative layers mount after hydration, and the page ends with a real FAQ.

## 2. Getting indexed

1. **Google Search Console:** add a domain property, verified by a DNS TXT record, and submit the sitemap. Request indexing for about 10 key pages a day, since the daily quota is small.
2. **Bing Webmaster Tools:** import from Search Console or add the site, then submit the sitemap.
3. **IndexNow** reaches Bing, Yahoo, DuckDuckGo, Yandex, Seznam and Naver; Google doesn't use it.
   - Put a 64-hex key file in `public/`.
   - `scripts/indexnow.mjs` posts every URL from the live sitemap.
   - A CI workflow runs it after deploys that touch the site, once a day and on demand. The blog writer also pings it for each new post.
4. **Video:** each video needs its own watch page (the video is the main content, plus a transcript and VideoObject). Without one, Search Console reports "Video isn't on a watch page".
5. **Serving video:** serve files with byte ranges (206 responses), or Safari won't play them. Static asset hosts often can't do this, so a small API route does.

## 3. Page programs

Each program is a template, a data file and a registry entry. All of them carry CTAs with `data-cta="<location>"` and fire analytics on a separate "landing" project.

### Use-case pages `/for/<slug>` (bottom of funnel)
- Target "X to presentation" and "AI X maker" searches, for example PDF to PPT, investor update or lesson plan to slides.
- Each page has:
  - a sample input;
  - four real rendered slides whose numbers match that input exactly;
  - a FAQ;
  - three related use cases and an "All use cases" card.
- **Rule:** only claim inputs and limits the product really supports. Check each one in code before adding the page. We skipped Notion because public Notion pages need a JavaScript render that often times out.

### Comparison pages `/vs/<rival>` (middle of funnel)
- One template and one data file, ordered by relevance.
- Each page has a feature table, a fair "Where <rival> is a good fit" section and a footnote with the month the facts were checked.
- Names are used nominatively, with no rival logos. Re-check their pages before editing a row.

### Alternatives `/alternatives/<rival>` and head-to-head `/compare/<a>-vs-<b>`
- "Best <Rival> alternatives in 2026" lists your product first, and is honest about where the rival wins.
- Head-to-head pages are neutral comparisons of two rivals. Pick pairs where autocomplete suggests both orders ("a vs b" and "b vs a").
- A test checks that the registry, paths and data agree.

### Template gallery `/templates/<slug>`
- 24 complete example decks, each rendered in full on its own static page.
- "Use this template" copies it into the user's account. Signed-out visitors sign up and come back to the same page.
- Templates rank for "<thing> template" searches and convert well.

### Free tools `/tools/<slug>` (top and middle of funnel, and link bait)
- Browser-only tools never upload anything: speech timer, slide count calculator, file compressor, word counter, font pairing and colour palette.
- AI tools (outline, pitch deck, speaker notes, titles) call a public API:
  - per-IP and global daily caps, with IPs stored only as a hash;
  - spend recorded per call.
- **Email gate:** a tool asks for an email once per browser before showing its result. This builds a lead list; signed-in users skip it.
- Each tool page has the tool, a FAQ, related tools and a CTA to the main product.

### Embeddable widget (backlinks)
- One tool is offered as an iframe snippet: `/embed/speech-timer`. It is the only path other sites may frame (`frame-ancestors *` only there) and is noindexed.
- The snippet repeats a "<Tool> by <Brand>" credit link outside the iframe, so the embedding page carries a real link.

### Blog `/blog/<slug>`
- Written by an automated pipeline (section 4). Each post answers one real search query and links into the use-case, comparison and tool pages.

### Trust pages (E-E-A-T)
- `/about` and `/contact`: at least 500 characters each, in the sitemap and footer. The same text is served as Markdown to agents.
- `/about/<founder>`: a Person JSON-LD with `sameAs` links to real profiles. Every blog post's author points to it.

### Hubs and footer
- `/use-cases`, `/tools`, `/templates`, `/alternatives` and `/compare` are hub pages, each with ItemList JSON-LD.
- The footer is vertical link lists under small headings (Product, Use cases, Compare, Company), so every program is a click from every page.

## 4. Automated blog writer

It runs on a cron inside the API worker. The real work runs in a Durable Object, because cron isolates get only about 30 seconds and may land where the model provider refuses.

1. **Schedule:** two runs a day (03:00 and 15:00 UTC). A run starts only if none is running and today's quota isn't met; a run stuck for 30 minutes is marked failed.
2. **Topic:**
   - Pull Search Console queries with a service account (RS256 JWT, Search Analytics API).
   - Rotate funnel stages by day of the year: bottom, middle, bottom, middle, top.
   - Pick a query for today's stage that no existing post ranks for.
   - If none fits, take the next queued topic. If the queue is empty, the model proposes 10 topics.
   - Reject topics whose title is 60% or more similar to an existing one.
3. **Research:** the model searches with Google grounding. Keep only sources that return 200, then a triage pass drops untrustworthy pages.
4. **Write:** outline, then draft, self-check and citation check. Related open queries go in as angles and FAQ questions, never as keyword stuffing.
5. **Lint gates** (`lintPost`):
   - 900 to 1,600 words;
   - 3 to 12 cited external sources, each listed and checked, each cited at most 3 times;
   - every internal link is a real route; `related` holds only /for, /vs or /tools pages that fit the stage;
   - no H1 in the body, no raw HTML, no em or en dashes, no banned filler phrases;
   - link text is words, never a bare URL;
   - brand named at most 3 times;
   - title 70 characters or fewer, description 70 to 160, blurb 8 words or fewer.
6. **Repair:** up to two "fix only these issues" passes. If it still fails, the run records the reason, skips the topic, and retries once on the next topic while under 6 minutes have passed. The hard limit is 12 minutes.
7. **Voice and facts:**
   - The persona and its knowledge file live in `.md` files bundled with the worker, so they stay editable.
   - Product facts come only from one `facts.md`; update it whenever prices or limits change.
   - The byline is always the real founder.
8. **Publish and notify:** save to the database, ping IndexNow and record the spend. Spend is capped at $1 a run.
9. **Fix bad posts in place:** edit or soft-delete the database row; never delete it.

## 5. AI search (GEO)

- **`/llms.txt`:** a hand-written product summary (what it does, plans, limits, key pages) plus a generated list of blog posts.
- **Markdown for agents:** when a request's `Accept` header ranks `text/markdown` at least as high as HTML (RFC 9110 q-values), answer with Markdown:
  - `/` with `llms.txt`, the trust pages with their Markdown, and blog posts with their source;
  - responses carry `Vary: Accept` and `X-Robots-Tag: noindex`;
  - unknown paths get a Markdown 404 that links `llms.txt` and the sitemap.
- **Buyer prompt list:** keep a list of about 50 prompts. Each one has the question a buyer would type into an assistant, the shorter Google query, and the page that should win it. Use it to find missing pages and to track whether assistants cite you.
- **Consistent facts everywhere:** pricing, limits and features are said the same way on the site, in `llms.txt`, in `facts.md` and in directory listings. Assistants repeat whatever they find.

## 6. Off-page

- **Launch sites and directories:** track each one (status, date, link, badge requirements) in a single tracker file.
  - Free ones first; skip paid-only listings.
  - Some require a footer badge. Keep its exact link and image URL, and allow its host in the CSP `img-src`.
- **Review sites:** AlternativeTo, SaaSHub, G2 (one G2 listing also covers Capterra, GetApp and Software Advice) and Crunchbase. Link your rivals as alternatives so you appear on their pages.
- **Roundup outreach:** find "best AI <category> tools" articles and pitch the authors a short, specific note.
- **Communities:** help first. Disclose any mention, keep to one post a day, and follow each sub's self-promotion rules. Tag links with `?ref=<source>`.
- **Medium** is a place to reach readers, not a source of backlinks or a home for automated posts (researched 2026-10-31):
  - **No API:** Medium issues no new API tokens and allows no new integrations (closed 2025-01-01). Posting is manual in the editor or through Import a story.
  - **Import a story** backdates the post and sets `rel=canonical` to your original. Import only after Google has indexed the original (check it with URL Inspection), then view the Medium page source to confirm the canonical.
  - **Links are `ugc nofollow`,** so they pass little or no ranking credit. The value is Medium's readers, referral clicks, links from people who read the piece, and AI assistants that cite Medium.
  - **AI-written text gets network-only distribution:** followers only, never recommended. Medium's guidelines also list content marketing and stories mainly meant to gather sign-ups as low-value. So never cross-post the automated blog as is.
  - **What gets distributed** is first-person stories from real experience, with real numbers and takeaways. Medium's curators pick less than 1% of stories for Boost, and partner publications nominate about half of those. Submit unpublished drafts to a publication whose editors nominate for Boost, such as Entrepreneur's Handbook (founder stories plus takeaways), Better Marketing (case studies, no personal CTAs) or Start it up (solo builders).
  - **Format:** one link to your site where it actually helps the reader, no "read the full post on our site" teasers, no paywall for product stories. Disclose any AI-written text in the first two paragraphs.
  - **Automated cross-posting:** Dev.to has a write API, and its links carry no nofollow (`noopener noreferrer`). Hashnode on a custom domain uses `ugc` without nofollow. Both are better than Medium for automation.
- **Referral paths you own:** a small "Made with <Brand>" link on public share pages, with `?ref=madewith`. It never goes inside the user's exported files.

## 7. Measurement

- **Search Console:** queries by page and position. The blog writer reads the same data.
- **Product analytics:** landing pages report to their own project, and every CTA carries a `data-cta` location.
- **GA4:** traffic sources only. It is loaded late and only on the production host.
- **Bing Webmaster Tools:** indexing and IndexNow status.

## 8. Checklist for a new public page

- [ ] It answers a real query (from Search Console, autocomplete or the prompt list).
- [ ] Static route with static `metadata`: title, description, canonical and OG.
- [ ] The right JSON-LD types.
- [ ] Added to the registry, so the sitemap, hub, footer and `llms.txt` all show it.
- [ ] Links to 3 related pages, plus one CTA with a `data-cta` location.
- [ ] Every claim matches a real limit or feature; rival facts are dated.
- [ ] No horizontal scroll at 360 px, and no layout shift.
- [ ] After deploy: request indexing in Search Console; IndexNow runs by itself.

## Strategy log

Newest first. Add a line for every SEO change: the date, what changed and why.

| Date | Change | Why |
|---|---|---|
| 2026-10-31 | Medium strategy researched: founder stories through Boost-nominating publications; import only human-edited posts with a canonical; Dev.to and Hashnode for automation | Medium closed its API, nofollows links and limits AI text to followers |
| 2026-10-31 | This playbook created | Reuse the pipeline on other apps |
| 2026-10-30 | `www` now 301s to the apex | `www.makedecks.com` didn't redirect, so pages were split across two hosts |
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
