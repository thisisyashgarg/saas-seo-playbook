# 7. Measurement

Know which pages get found, which queries they win, which visits turn into sign-ups, and whether the work is paying off.

## What you'll set up

- Google Search Console, read the right way: the five reports that matter and what each status means.
- A script that exports your "striking distance" queries from the Search Console API.
- Bing Webmaster Tools for Bing, its partners and IndexNow status.
- Product analytics on landing pages, kept separate from the app, with one click tracker for every CTA via `data-cta`.
- Section-view tracking, so you know how far people read.
- GA4 for traffic sources only, loaded late so it costs no speed.
- A weekly and a monthly review routine.
- A short list of numbers that tell you the work is working.

## Step 1: Google Search Console

Set-up is covered in [chapter 2](02-getting-indexed.md) (a domain property verified by DNS, sitemap submitted). This step is about reading it.

### The reports that matter

| Report | Where | What to look at | What it tells you |
|---|---|---|---|
| Performance | Search results | Queries, Pages, total impressions and clicks, average position, CTR | Which searches you appear for, and where |
| Page indexing | Indexing, Pages | Indexed count, and the "Why pages aren't indexed" table | Whether Google keeps your pages |
| Sitemaps | Indexing, Sitemaps | Status "Success", discovered URL count | Whether Google reads your sitemap |
| URL Inspection | Search bar at the top | "URL is on Google", the canonical Google chose, the last crawl date | The truth for one page |
| Video indexing | Indexing, Videos | "Video isn't on a watch page" and similar | Whether video pages qualify for video results |

Also check Core Web Vitals and the structured data reports (under Experience and Enhancements) once a month. Core Web Vitals needs real-user traffic, so a new site shows "Not enough data" for weeks; use Lighthouse or PageSpeed Insights until then.

### Reading the Performance report

1. **Set the date range to the last 28 days** and turn on "Compare" with the previous period.
2. **Look at impressions first, clicks second.** A new site gets impressions weeks before it gets clicks. Rising impressions mean Google is testing your pages for more queries.
3. **Open the Queries tab and add a filter for one page** (Pages, then click the page) to see what that page actually ranks for. It's often not what you wrote it for.
4. **Find striking-distance queries:** average position 5 to 30 with real impressions. These are the cheapest wins. Improve the page that ranks (answer the query in a heading, add a FAQ entry, link to it from related pages), or write a post that targets it.
5. **Find low-CTR queries:** high impressions, position under 10, CTR far below other queries at that position. Rewrite the title and description for that page.
6. **Read average position with care.** It is averaged over every impression, including ones on page 5 for a single searcher. Use it for direction, not as a rank.
7. **Exclude your brand** with a query filter ("Doesn't contain" your brand name) to see non-branded growth, which is what SEO work drives.

### Reading the Page indexing report

| Status | What it means | What to do |
|---|---|---|
| Indexed | In Google | Nothing |
| Discovered, currently not indexed | Google knows the URL but hasn't crawled it yet | Normal on new sites. Link to it from more pages; request indexing for key pages. |
| Crawled, currently not indexed | Google read it and chose not to keep it | Usually thin or near-duplicate content. Improve the page or merge it. |
| Duplicate without user-selected canonical | Google sees two URLs with the same content and no canonical | Add a canonical, or 301 one to the other (often `www` vs apex, or trailing slashes). |
| Alternate page with proper canonical | A variant pointing at its canonical | Fine. |
| Excluded by "noindex" tag | You asked for it to be left out | Fine for app pages. Check no public page is in this list. |
| Page with redirect | A URL that redirects | Fine for old URLs. Remove redirecting URLs from your sitemap. |
| Not found (404) | A linked URL returns 404 | Fix the link, or 301 to the right page if it ever existed. |
| Blocked by robots.txt | Disallowed in robots | Fine for `/api/`. If an app page is here, see [chapter 1](01-technical-foundations.md): don't block pages in robots, send `noindex` instead. |

> **Real-world example:** `www` didn't redirect to the apex, so pages were split across two hosts. The fix was a 301 from `www` to the apex ([chapter 1](01-technical-foundations.md)). Earlier, app pages blocked in `robots.txt` were showing up as indexed bare URLs; moving to `noindex` headers fixed it.

### Request indexing for key pages

URL Inspection has a "Request indexing" button with a small daily quota. Use it for about 10 key pages a day after a launch or a big change: the home page, pricing, your strongest use-case and comparison pages, new posts. Everything else is found through the sitemap.

### Export striking-distance queries with the API

The Performance report in the browser caps rows and is slow to filter. A script gets the full query and page list so you can sort it, or feed it to a content pipeline ([chapter 4](04-automated-blog-writer.md) does).

Set up access once:

1. In Google Cloud, create a project (or use an existing one) and enable the **Google Search Console API**.
2. Create a **service account** and a **JSON key** for it. Store the JSON as a secret, for example `GSC_SERVICE_ACCOUNT`. Never commit it.
3. In Search Console, open Settings, Users and permissions, and add the service account's email as a user with **Restricted** permission.
4. Your property name is `sc-domain:example.com` for a domain property, or `https://example.com/` for a URL-prefix property.

`scripts/gsc-queries.mjs`, adapted from a production blog writer. It signs a JWT with Web Crypto, swaps it for an access token, and pulls the last 28 days by query and page. It runs on Node 20 or later with no packages.

```js
// GSC_SERVICE_ACCOUNT='<json>' node scripts/gsc-queries.mjs > striking.csv
const SITE = "sc-domain:example.com";
const BRAND = /\bexample\b/i;
const STRIKING = { min: 5, max: 30 };

const b64url = (bytes) => btoa(String.fromCharCode(...bytes)).replace(/\+/g, "-").replace(/\//g, "_").replace(/=+$/, "");
const b64json = (v) => b64url(new TextEncoder().encode(JSON.stringify(v)));
const isoDay = (ms) => new Date(ms).toISOString().slice(0, 10);

async function serviceJwt({ client_email, private_key }, now = Date.now()) {
  const der = Uint8Array.from(atob(private_key.replace(/-----[^-]+-----|\s/g, "")), (c) => c.charCodeAt(0));
  const key = await crypto.subtle.importKey("pkcs8", der, { name: "RSASSA-PKCS1-v1_5", hash: "SHA-256" }, false, ["sign"]);
  const iat = Math.floor(now / 1000);
  const unsigned = `${b64json({ alg: "RS256", typ: "JWT" })}.${b64json({ iss: client_email, scope: "https://www.googleapis.com/auth/webmasters.readonly", aud: "https://oauth2.googleapis.com/token", iat, exp: iat + 3600 })}`;
  const sig = new Uint8Array(await crypto.subtle.sign("RSASSA-PKCS1-v1_5", key, new TextEncoder().encode(unsigned)));
  return `${unsigned}.${b64url(sig)}`;
}

const account = process.env.GSC_SERVICE_ACCOUNT;
if (!account) throw new Error("GSC_SERVICE_ACCOUNT is not set");
const now = Date.now();

const tok = await fetch("https://oauth2.googleapis.com/token", {
  method: "POST",
  headers: { "content-type": "application/x-www-form-urlencoded" },
  body: new URLSearchParams({ grant_type: "urn:ietf:params:oauth:grant-type:jwt-bearer", assertion: await serviceJwt(JSON.parse(account), now) }),
});
if (!tok.ok) throw new Error(`token exchange returned ${tok.status} ${(await tok.text()).slice(0, 200)}`);
const { access_token } = await tok.json();

const res = await fetch(`https://www.googleapis.com/webmasters/v3/sites/${encodeURIComponent(SITE)}/searchAnalytics/query`, {
  method: "POST",
  headers: { authorization: `Bearer ${access_token}`, "content-type": "application/json" },
  body: JSON.stringify({ startDate: isoDay(now - 27 * 864e5), endDate: isoDay(now), dimensions: ["query", "page"], rowLimit: 1000 }),
});
if (!res.ok) throw new Error(`searchAnalytics returned ${res.status} ${(await res.text()).slice(0, 200)}`);
const rows = ((await res.json()).rows ?? []).map((r) => ({ query: r.keys[0], page: r.keys[1], clicks: r.clicks, impressions: r.impressions, ctr: r.ctr, position: r.position }));

const striking = rows
  .filter((r) => !BRAND.test(r.query) && r.position >= STRIKING.min && r.position <= STRIKING.max)
  .sort((a, b) => b.impressions - a.impressions);

const cell = (v) => `"${String(v).replace(/"/g, '""')}"`;
console.log("query,page,impressions,clicks,ctr,position");
for (const r of striking) console.log([cell(r.query), cell(r.page), r.impressions, r.clicks, (r.ctr * 100).toFixed(1), r.position.toFixed(1)].join(","));
```

Notes:

- Search Console data lags about two days, so the last day or two are partial.
- `rowLimit` goes up to 25,000 per request. Add `startRow` to page through more.
- Queries with very few searches are hidden by Google for privacy, so the totals by query are lower than the totals in the report header.
- The 403 "User does not have sufficient permission" error means the service account isn't added as a user on that exact property.

## Step 2: Bing Webmaster Tools

Bing's index also feeds Yahoo and DuckDuckGo results and Microsoft Copilot. Bing Webmaster Tools is free and takes ten minutes.

1. Sign in at bing.com/webmasters and choose **Import from Google Search Console**. It copies your verified sites and sitemaps. Or add the site and verify by DNS.
2. Check **Sitemaps**: your sitemap is listed and was read recently.
3. Check **IndexNow** (or URL Submission): your IndexNow pings show up with their URLs. If you set up the IndexNow script in [chapter 2](02-getting-indexed.md), this is where you confirm it works.
4. Read **Search Performance** the same way as Google's: impressions first, clicks later, queries per page.
5. Use **URL Inspection** for single pages, as in Google.
6. Glance at **Backlinks** once a month. Bing shows referring domains for free, which is a quick way to see which directory and outreach links went live.
7. Run **Site Scan** after a big change. It flags missing titles, duplicate descriptions and broken links.

## Step 3: Product analytics on landing pages

Search Console stops at the click. Product analytics shows what visitors do next.

### Keep landing and app separate

Use two projects (or two properties) in your analytics tool: one for public pages, one for the signed-in app. Marketing pages get many anonymous visitors; mixing them into the app project inflates its numbers and makes funnels hard to read.

Decide the surface by path, in one function:

```ts
type Surface = "landing" | "app";

export const surfaceFor = (path: string): Surface =>
  path === "/" ||
  ["/pricing", "/about", "/contact", "/privacy", "/use-cases", "/tools", "/templates", "/blog", "/alternatives", "/compare"].includes(path) ||
  ["/for/", "/vs/", "/tools/", "/templates/", "/blog/", "/alternatives/", "/compare/", "/watch/", "/about/", "/s/"].some((p) => path.startsWith(p))
    ? "landing"
    : "app";
```

Every pageview and event goes to the client for `surfaceFor(location.pathname)`. When you add a new page program, add its prefix here in the same change.

Keep analytics off on localhost, and never send emails, file names or user content as event properties.

### Track every CTA with `data-cta`

Instead of an `onClick` on every button, mark each CTA in the markup and use one listener.

1. **Mark every CTA** with two attributes: what it is (`data-cta`) and where it is (`data-cta-location`).

   ```tsx
   <Link href="/signup" data-cta="sign_up" data-cta-location="header">Get started</Link>
   <Link href="/signup" data-cta="sign_up" data-cta-location="hero">Start free</Link>
   <Link href="/pricing" data-cta="pricing_link" data-cta-location="footer_product">Pricing</Link>
   <Link href={`/tools/${t.slug}`} data-cta="tool_link" data-cta-location={`tool_${slug}`}>{t.name}</Link>
   ```

2. **Name locations by a fixed convention,** so you can group them: `header`, `hero`, `pricing`, `faq`, `footer_<column>`, `usecase_<slug>`, `vs_<rival>`, `tool_<slug>`, `blog_<slug>`.
3. **Mark page sections** you want read-depth for with `data-section="<name>"` (`hero`, `demo`, `features`, `pricing`, `faq`).
4. **Mount one tracker component** on each landing page (or once in the landing layout). Adapted from a production app:

   ```tsx
   "use client";
   import { useEffect } from "react";
   import { trackLanding } from "@/lib/analytics";

   export function LandingTracking() {
     useEffect(() => {
       const onClick = (e: MouseEvent) => {
         const el = (e.target as Element | null)?.closest<HTMLElement>("[data-cta]");
         if (!el) return;
         trackLanding("landing_cta_clicked", { cta: el.dataset.cta!, location: el.dataset.ctaLocation ?? "unknown" });
       };
       const seen = new Set<string>();
       const observer = new IntersectionObserver(
         (entries) => {
           for (const entry of entries) {
             const section = (entry.target as HTMLElement).dataset.section!;
             if (entry.isIntersecting && !seen.has(section)) {
               seen.add(section);
               trackLanding("landing_section_viewed", { section });
             }
           }
         },
         { threshold: 0.4 },
       );
       document.querySelectorAll<HTMLElement>("[data-section]").forEach((s) => observer.observe(s));
       document.addEventListener("click", onClick);
       return () => {
         document.removeEventListener("click", onClick);
         observer.disconnect();
       };
     }, []);
     return null;
   }
   ```

   `closest()` means a click on the icon or text inside a link still counts. Each section is reported once per page view, when 40% of it is on screen.

5. **Write `trackLanding` for your tool.** Keep the event names typed in one place, so a typo is a compile error:

   ```ts
   type LandingEvents = {
     landing_cta_clicked: { cta: string; location: string };
     landing_section_viewed: { section: string };
   };
   ```

   Then send through whichever tool you use. All of these are the tools' own calls:

   ```ts
   // PostHog (a client initialised for the landing project)
   export const trackLanding = <E extends keyof LandingEvents>(event: E, props: LandingEvents[E]) => landingClient?.capture(event, props);

   // GA4
   export const trackLanding = <E extends keyof LandingEvents>(event: E, props: LandingEvents[E]) => window.gtag?.("event", event, props);

   // Plausible (custom properties)
   export const trackLanding = <E extends keyof LandingEvents>(event: E, props: LandingEvents[E]) => window.plausible?.(event, { props });

   // Umami
   export const trackLanding = <E extends keyof LandingEvents>(event: E, props: LandingEvents[E]) => window.umami?.track(event, props);
   ```

### What to build from it

1. **A funnel:** landing pageview, then `landing_cta_clicked`, then sign-up completed (from the app project or your database), then the first real action in the product.
2. **CTA clicks broken down by `location`.** This shows which page programs drive sign-ups, and which CTAs nobody clicks.
3. **Section views as a drop-off chart:** how many visitors reach the demo, the pricing block, the FAQ.
4. **First-touch `ref` and UTM values,** from the landing URL of each visitor's first pageview, broken down by sign-ups.
5. **The "How did you hear about us?" answers** ([chapter 5](05-ai-search.md)) next to all of the above. Analytics sees clicks; only the answer sees an assistant recommendation that arrived with no referrer.

## Step 4: GA4 for traffic sources only

Product analytics is enough for behaviour. GA4 is still the easiest free view of where sessions come from (channel groups, referrers, campaigns), and it reads `utm_` tags automatically. Use it for that, and nothing else.

The cost is speed: `gtag.js` is large. Load it after the page has finished loading and the browser is idle, and only on the production host.

`components/google-analytics.tsx`, adapted from a production app:

```tsx
"use client";
import { useEffect } from "react";

declare global {
  interface Window { dataLayer?: unknown[]; gtag?: (...args: unknown[]) => void }
}

// Loaded after the page settles: gtag.js is large and would otherwise cost the landing page's speed scores.
export function GoogleAnalytics({ id, host }: { id: string; host: string }) {
  useEffect(() => {
    if (window.gtag || location.hostname !== host) return;
    const start = () => {
      window.dataLayer = window.dataLayer || [];
      window.gtag = function () {
        // gtag.js reads the arguments object itself, not an array.
        window.dataLayer!.push(arguments);
      };
      window.gtag("js", new Date());
      window.gtag("config", id);
      const s = document.createElement("script");
      s.async = true;
      s.src = `https://www.googletagmanager.com/gtag/js?id=${id}`;
      document.head.appendChild(s);
    };
    const idle = () => ("requestIdleCallback" in window ? requestIdleCallback(start, { timeout: 5000 }) : setTimeout(start, 3000));
    if (document.readyState === "complete") idle();
    else addEventListener("load", idle, { once: true });
  }, [id, host]);
  return null;
}
```

Mount it once in the root layout:

```tsx
<GoogleAnalytics id="G-XXXXXXXXXX" host="example.com" />
```

Notes:

- **Push `arguments`, not an array.** `gtag.js` expects the arguments object. Pushing `[...args]` looks the same and silently records nothing.
- **The host check** keeps localhost, preview deployments and staging out of your data without any config.
- **Allow it in your CSP:** `script-src` and `connect-src` need `https://www.googletagmanager.com`, and `connect-src` and `img-src` need `https://*.google-analytics.com` and `https://*.analytics.google.com`.
- **Never send personal data:** no emails, names or user content in events or page URLs.
- **Consent:** if you have visitors in regions that require consent for analytics cookies, decide how you handle that before turning it on.
- A visitor who leaves within the first second or two isn't counted. That's acceptable for a traffic-source view; your product analytics still records them.

Where to read it: Reports, Acquisition, Traffic acquisition, with "Session source / medium" as the dimension. Referrals from `chatgpt.com`, `perplexity.ai`, `copilot.microsoft.com` and `gemini.google.com` show up here, alongside `utm_source=chatgpt.com` sessions.

## Step 5: The review routine

### Weekly (30 minutes)

- [ ] Search Console Performance, last 7 days vs the previous 7: impressions, clicks, number of queries with impressions. Note the three numbers in a log.
- [ ] Top 10 pages by impressions: any new page entering the list? Any page dropping out?
- [ ] Run the striking-distance export. Pick 2 or 3 queries and improve the page that ranks, or queue a post.
- [ ] Page indexing: any new "Not found (404)", "Duplicate without user-selected canonical" or "Crawled, currently not indexed" entries since last week?
- [ ] Request indexing for pages published this week.
- [ ] Bing Webmaster: IndexNow submissions accepted; no new crawl errors.
- [ ] Product analytics: landing pageviews, CTA clicks by location, sign-ups. Which page program drove the most sign-ups?
- [ ] "How did you hear about us?" counts for the week.
- [ ] Sign-ups by `ref` value: did any listing, thread or outreach send people?
- [ ] Off-page tracker: update statuses, follow up on outreach sent 5 to 7 days ago.

### Monthly (2 hours)

- [ ] Search Console, last 28 days vs the previous 28: the same three numbers, plus non-branded clicks (brand excluded).
- [ ] Queries where a page ranks for something it wasn't written for: retitle the page, or write the page that query deserves.
- [ ] Low-CTR queries in the top 10: rewrite titles and descriptions.
- [ ] Core Web Vitals and structured data reports: fix any new errors.
- [ ] Bing Backlinks: which new referring domains appeared? Mark those listings Live in the tracker.
- [ ] AI citation check across the buyer prompt list ([chapter 5](05-ai-search.md)). Named, Linked and Absent counts per assistant, compared with last month.
- [ ] Facts check: pricing and limits the same on the site, `llms.txt`, `facts.md` and every listing.
- [ ] GA4 traffic acquisition: share of sessions by channel, and referrals from assistants.
- [ ] Prune or merge pages that have had zero impressions for three months and no inbound links.
- [ ] Add a line to your strategy log for every SEO change made this month, with the reason.

## Step 6: What "working" looks like

SEO is slow and noisy. Watch direction over weeks, not single days. The patterns below are rules of thumb for a new SaaS site, not guarantees.

| Signal | Where | Working | Not working yet | What to do |
|---|---|---|---|---|
| Pages indexed | Search Console, Page indexing | Most sitemap URLs indexed within the first one to two months | Most stuck in "Discovered" or "Crawled, not indexed" | More internal links to those pages; make thin pages substantial or merge them |
| Queries with impressions | Search Console, Performance | The count of distinct queries grows week over week | Flat for a month after indexing | Pages may target queries nobody types; recheck against autocomplete and the prompt list |
| Impressions | Search Console | Rising trend over 28-day windows | Flat or falling | Check indexing first, then whether pages answer their query in the title and first heading |
| Average position for target queries | Search Console | Moving from 30s into the teens, then under 10 | Stuck past 30 | The page may be too thin or the query too competitive; target longer, more specific queries |
| Clicks | Search Console | Follow impressions after a few weeks | Impressions up, clicks flat | Titles and descriptions don't earn the click; rewrite them |
| Non-branded clicks | Search Console, brand excluded | A growing share of all clicks | Nearly all clicks are your brand name | Your content isn't winning yet; people only find you by name |
| CTA click rate | Product analytics | Use-case and comparison pages convert better than the blog | Pages get visits but no CTA clicks | The page doesn't lead to the product; make the CTA specific to the page |
| Sign-ups by source | Sign-up question, `ref`, UTM | Search, AI assistants and listings each show up | Everything is "Other" or direct | Ask the question; tag every link you place |
| AI mentions | Monthly prompt check | Named for more prompts each month | Absent everywhere | Off-page work ([chapter 6](06-off-page.md)): get into the lists assistants cite |
| Referring domains | Bing Backlinks, Search Console Links | New domains each month as listings go live | None after two months | Check the tracker: listings stuck in queues, badges missing, outreach unsent |

> **Real-world example:** in its first 10 days, 83 of about 300 sign-ups said they came from ChatGPT or an AI assistant and 10 from Google search. Search Console showed impressions but few clicks at that point. Without the sign-up question, the AI channel would have looked like direct traffic, and the effort would have gone to the wrong place.

Two rules for reading all of this:

1. **Compare like with like.** 28 days against the previous 28, weekdays against weekdays. Many products see lower traffic at weekends.
2. **Change one thing at a time on a page,** and note the date in the strategy log, so you can tell what moved the number.

## Checklist

- [ ] Search Console domain property verified, sitemap read, key pages requested for indexing.
- [ ] You know how to read Performance (impressions, striking distance, low CTR, brand excluded) and Page indexing (each status and its fix).
- [ ] `scripts/gsc-queries.mjs` runs with a service account added as a Restricted user, and the key is stored as a secret.
- [ ] Bing Webmaster Tools imported from Search Console, sitemap read, IndexNow submissions visible.
- [ ] Landing and app analytics are separate projects, chosen by `surfaceFor(path)`.
- [ ] Every CTA has `data-cta` and `data-cta-location`; key sections have `data-section`; one `LandingTracking` component reports both.
- [ ] Event names are typed in one place; no personal data in event properties.
- [ ] GA4 loads after load and idle, only on the production host, with CSP entries, and is used for sources only.
- [ ] A funnel from landing pageview to first real action exists, broken down by CTA location and first-touch `ref`.
- [ ] The weekly and monthly routines are on your calendar, and results go into a log.

## Common mistakes

- **Judging SEO in the first two weeks.** New pages need weeks to be indexed and tested. Watch impressions and query count, not clicks.
- **Treating average position as a rank.** It is an average over every impression. Use it for direction.
- **Ignoring "Crawled, currently not indexed".** It is Google telling you a page isn't worth keeping. Improve or merge it.
- **Blocking pages in `robots.txt` to keep them out of search.** Google can't see the `noindex`, so the bare URL can still be indexed. Send `noindex` instead.
- **Mixing landing and app analytics.** Funnels and retention become unreadable.
- **An `onClick` on every CTA.** Some get missed. One delegated listener with `data-cta` covers every link, including ones added later.
- **Free-text CTA locations.** "Hero", "hero" and "hero-button" become three rows. Fix the naming convention and stick to it.
- **Loading GA4 in the head.** It costs Lighthouse points on every landing page for data you only read monthly.
- **Pushing an array into `dataLayer`.** `gtag.js` ignores it; nothing is recorded and nothing errors.
- **Relying on referrers for AI traffic.** Many assistant clicks arrive without one. Ask "How did you hear about us?".
- **Not logging changes.** Without dated notes you can't tell which change moved a number.
