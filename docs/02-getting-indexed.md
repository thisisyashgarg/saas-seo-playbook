# 2. Getting indexed

Tell Google, Bing and the IndexNow engines about every page, automatically, and confirm they picked it up.

## What you'll set up

- A Google Search Console domain property, verified by DNS, with your sitemap submitted.
- A plan for the small daily "Request indexing" quota.
- Bing Webmaster Tools, imported from Search Console.
- IndexNow: a key file, a script that submits your sitemap, a GitHub Actions workflow that runs it after deploys and daily, and a server-side ping for pages created at runtime.
- Video SEO: a watch page per video, VideoObject JSON-LD, and a byte-range route so every browser can play the file.
- A way to verify each step.

This chapter assumes chapter 1 is done: one canonical host, a working `robots.txt`, and a sitemap at `https://example.com/sitemap.xml`.

---

## 2.1 Google Search Console

### Add a domain property

A domain property covers every protocol and subdomain (`https://`, `http://`, `www`, `blog.` and so on) in one place, so redirects and stray hosts show up in the same reports.

1. Go to `search.google.com/search-console` and sign in with the Google account that should own the property. Use a long-lived company or founder account, not a contractor's.
2. Open the property menu (top left) and choose **Add property**.
3. Pick **Domain** (the left box), enter `example.com` with no protocol and no `www`, and click **Continue**.
4. Google shows a TXT record like `google-site-verification=abc123...`. Click **Copy**.
5. In your DNS provider, add the record:

   | Field | Value |
   |---|---|
   | Type | `TXT` |
   | Name | `@` (the apex; some providers want `example.com` or an empty field) |
   | Content | the full `google-site-verification=...` string |
   | TTL | Auto, or 3600 |

   On Cloudflare: zone `example.com`, **DNS > Records > Add record**, Type `TXT`, Name `@`, Content as above, **Save**. TXT records are never proxied.
6. Back in Search Console, click **Verify**. It usually works within a few minutes. If it fails, wait an hour and try again; DNS can take time to propagate.
7. Leave the TXT record in place forever. Removing it un-verifies the property.

Check the record from your terminal before clicking Verify:

```bash
dig +short TXT example.com
# "google-site-verification=abc123..."
```

### Add other owners

**Settings > Users and permissions > Add user.** Give co-founders **Owner** and contractors **Full** or **Restricted**. If the only owner leaves, you lose access, so always have two owners.

### Submit the sitemap

1. In the property, open **Indexing > Sitemaps**.
2. Under **Add a new sitemap**, type `sitemap.xml` (the field already shows `https://example.com/`) and click **Submit**.
3. Within minutes to a day, the row shows **Status: Success** and a **Discovered pages** count. That count should match the number of `<loc>` entries in your sitemap.

You only submit once. Google re-reads the sitemap on its own, and the `Sitemap:` line in `robots.txt` points every other crawler to it.

If the status shows **Couldn't fetch**, open `https://example.com/sitemap.xml` in a private window and check that it returns 200, is XML, and is not blocked by `robots.txt` or a firewall rule.

### URL Inspection and Request indexing

URL Inspection tells you what Google knows about one URL and lets you ask for a crawl.

1. Paste the full URL into the search bar at the top of Search Console and press Enter.
2. The first screen shows the indexed state: **URL is on Google** or **URL is not on Google**, plus the reason.
3. Click **Test live URL**. This fetches the page now. Check:
   - **Page availability:** "Page can be indexed".
   - **View tested page > HTML:** your `<title>`, description and canonical are in `<head>` (see chapter 1, section 1.6).
   - **View tested page > Screenshot:** the page renders, not a blank shell.
   - **Detected items:** your structured data types (Breadcrumbs, FAQ, Videos).
4. Click **Request indexing**. It takes a minute and then confirms the URL was added to a priority crawl queue.

### The quota and how to spend it

Request indexing has a small daily limit per property. Google doesn't publish the number; in practice it is around 10 URLs a day before you see "Quota exceeded". Spend it on the pages that matter most and let the sitemap handle the rest.

| Priority | Request indexing for | Skip |
|---|---|---|
| 1 | The homepage and pricing, right after launch or a big change | |
| 2 | New hub pages (`/use-cases`, `/tools`), because they link to everything else | |
| 3 | New bottom-of-funnel pages (use-case, comparison, alternatives) | |
| 4 | A page you rewrote substantially (new title, new content) | Typo fixes and small edits |
| 5 | New blog posts, if quota is left | Posts the sitemap already lists; they will be found |
| | | The same URL twice in a week; repeating doesn't speed it up |
| | | App pages, redirects, noindexed pages |

A simple routine that works:

1. After a deploy that adds pages, write down the new URLs in order of value.
2. Request the top 10 that day, the next 10 the next day.
3. Check them a week later with URL Inspection. Most will be indexed. Pages still "Discovered, currently not indexed" usually need more internal links or more distinct content, not another request.

The Google Indexing API is not a workaround. Google allows it only for pages with JobPosting or BroadcastEvent markup.

### Read the Page indexing report

**Indexing > Pages** lists every URL Google knows about and why each one is or isn't indexed. Several "not indexed" reasons are expected and good:

| Reason | Expected? | What it means |
|---|---|---|
| Excluded by 'noindex' tag | Yes, for app pages | Your `X-Robots-Tag` works |
| Page with redirect | Yes, for `www`, `http` and old URLs | Your 301s work |
| Alternate page with proper canonical tag | Usually | URLs with query strings pointing to the clean URL |
| Not found (404) | Check | Old or mistyped links; fine if the URL never existed |
| Blocked by robots.txt | Only for `/api/` | If an app page appears here, see chapter 1, section 1.2 |
| Indexed, though blocked by robots.txt | No | The exact problem chapter 1 warns about; unblock and noindex |
| Discovered, currently not indexed | Watch | Google knows the URL but hasn't crawled it; add internal links |
| Crawled, currently not indexed | Watch | Google crawled it and chose not to index; improve or merge the page |
| Duplicate without user-selected canonical | No | Missing or wrong canonical; fix the metadata |

New sites take weeks to be crawled fully. Look at the trend, not a single day.

---

## 2.2 Bing Webmaster Tools

Bing's index also feeds Yahoo, DuckDuckGo and Microsoft Copilot, so it is worth the ten minutes.

### Import from Search Console

1. Go to `bing.com/webmasters` and sign in with a Microsoft, Google or Facebook account.
2. On the welcome screen, choose **Import your sites from GSC**.
3. Sign in with the Google account that owns your Search Console property and allow Bing read access.
4. Select your site and click **Import**. Bing verifies it through Google and copies the sitemaps you submitted there.

If your domain property doesn't appear in the import list, use **Add your site manually** instead: enter `https://example.com`, pick a verification method (an XML file in `public/`, a `<meta>` tag in your root layout, or a DNS CNAME), and verify.

### Submit the sitemap

**Sitemaps > Submit sitemap**, enter `https://example.com/sitemap.xml`, **Submit**. If it came in with the import, it is already listed.

### Other useful screens

| Screen | Use it for |
|---|---|
| URL Inspection | Same idea as Google's: indexed state and live fetch for one URL |
| URL Submission | Manual submission, with a daily quota shown on the page |
| IndexNow | URLs received through IndexNow (section 2.3), with dates |
| Search Performance | Clicks and impressions from Bing, by query and page |
| Site Explorer | How Bing sees your URL tree, with errors |

With IndexNow running, you rarely need URL Submission.

---

## 2.3 IndexNow

IndexNow is a protocol where you tell search engines which URLs changed, instead of waiting for them to crawl. One submission reaches every participating engine: Bing (and through it Yahoo and DuckDuckGo), Yandex, Seznam, Naver and others. **Google doesn't use IndexNow.** Google relies on your sitemap and Search Console.

### How it works

1. You create a key: a random string.
2. You publish it as a text file on your site, at `https://example.com/<key>.txt`, containing only the key. This proves you own the host.
3. You POST a list of URLs and the key to `https://api.indexnow.org/indexnow`. The engines fetch the key file to check it, then schedule the URLs for crawling.

### Step 1: create the key file

```bash
KEY=$(openssl rand -hex 32)
echo -n "$KEY" > "public/$KEY.txt"
echo "$KEY"
```

- The key may be 8 to 128 characters of letters, digits and dashes. A 64-character hex string is a good default and makes the file easy to find with a regex.
- Put the file in the folder your framework serves at the site root (`public/` in Next.js; `apps/web/public/` in a monorepo).
- Commit it. The key is public by design; anyone can read it from your site. Its only power is to submit URLs for your own host.
- Never rename or delete the file once you've submitted with it. The engines keep checking it.

After deploy, check it:

```bash
curl -s https://example.com/$KEY.txt
# prints the key, nothing else
```

### Step 2: the submit script

This script reads your live sitemap and submits every URL on your host. It waits for the key file to be live first, because it runs right after a deploy.

```js
// scripts/indexnow.mjs
// Submits sitemap URLs to IndexNow (Bing, Yahoo, DuckDuckGo, Yandex, Seznam, Naver...). Google doesn't use IndexNow.
//   node scripts/indexnow.mjs                      every <loc> in https://<host>/sitemap.xml
//   node scripts/indexnow.mjs --since=2026-01-15   only sitemap URLs with <lastmod> on or after that date
//   node scripts/indexnow.mjs <url>...             just those URLs
// The key is the hex-named .txt file in the public folder. IndexNow is a hint, not a gate: failures only warn.
import { readdirSync, readFileSync } from "node:fs";

const HOST = process.env.INDEXNOW_HOST ?? "example.com";
const ORIGIN = `https://${HOST}`;
const PUBLIC = new URL(`../${process.env.INDEXNOW_PUBLIC_DIR ?? "public"}/`, import.meta.url);
const BATCH = 10_000;

const args = process.argv.slice(2);
const since = args.find((a) => a.startsWith("--since="))?.slice("--since=".length);
const explicit = args.filter((a) => !a.startsWith("--"));

const keyFile = readdirSync(PUBLIC).find((f) => /^[a-f0-9]{32,128}\.txt$/.test(f));
if (!keyFile) {
  console.warn(`indexnow: no key file in ${PUBLIC.pathname}, skipping`);
  process.exit(0);
}
const key = readFileSync(new URL(keyFile, PUBLIC), "utf8").trim();

async function keyIsLive() {
  for (let i = 0; i < 20; i++) {
    const res = await fetch(`${ORIGIN}/${keyFile}`, { cache: "no-store" }).catch(() => null);
    if (res?.ok && (await res.text()).trim() === key) return true;
    await new Promise((r) => setTimeout(r, 15_000));
  }
  return false;
}

async function sitemapUrls() {
  const res = await fetch(`${ORIGIN}/sitemap.xml`, { cache: "no-store" });
  if (!res.ok) throw new Error(`sitemap.xml returned ${res.status}`);
  const xml = await res.text();
  return [...xml.matchAll(/<url>([\s\S]*?)<\/url>/g)].flatMap(([, block]) => {
    const loc = /<loc>([^<]+)<\/loc>/.exec(block)?.[1].trim();
    const lastmod = /<lastmod>([^<]+)<\/lastmod>/.exec(block)?.[1].trim();
    return loc && (!since || !lastmod || lastmod >= since) ? [loc] : [];
  });
}

try {
  if (!(await keyIsLive())) throw new Error(`${ORIGIN}/${keyFile} isn't serving the key yet`);
  const urls = explicit.length ? explicit : await sitemapUrls();
  const own = [...new Set(urls)].filter((u) => u === ORIGIN || u.startsWith(`${ORIGIN}/`));
  if (!own.length) console.log("indexnow: nothing to submit");
  for (let i = 0; i < own.length; i += BATCH) {
    const urlList = own.slice(i, i + BATCH);
    const res = await fetch("https://api.indexnow.org/indexnow", {
      method: "POST",
      headers: { "Content-Type": "application/json; charset=utf-8" },
      body: JSON.stringify({ host: HOST, key, keyLocation: `${ORIGIN}/${keyFile}`, urlList }),
    });
    // 200 accepted, 202 accepted pending key validation; anything else is worth a look but never fails the run.
    console.log(`indexnow: ${urlList.length} URLs -> ${res.status} ${res.statusText}`);
    if (res.status >= 400) console.warn(await res.text());
  }
} catch (err) {
  console.warn(`indexnow: ${err instanceof Error ? err.message : err}`);
}
```

What each part does:

| Part | Why |
|---|---|
| `INDEXNOW_HOST`, `INDEXNOW_PUBLIC_DIR` | Set these instead of editing the file. In a monorepo, `INDEXNOW_PUBLIC_DIR=apps/web/public`. The path is relative to the repo root, since the script sits in `scripts/`. |
| `keyIsLive` | Polls the key file for up to 5 minutes. Right after a deploy the new file may not be served yet, and submitting before it is live gets a 403. |
| `sitemapUrls` | Reads the live sitemap, so the list is always what is really deployed. |
| `--since` | Keeps daily runs to pages that changed. Pages without `<lastmod>` are always included. ISO dates compare correctly as strings. |
| `own` filter | Drops duplicates and any URL not on your host. IndexNow rejects a batch with foreign URLs (422). |
| `BATCH = 10_000` | The protocol's limit per request. |
| Never exits non-zero | IndexNow is a hint. A failed ping must not fail CI or block a deploy. |

Add it to `package.json` so it is easy to run by hand:

```json
{
  "scripts": {
    "indexnow": "node scripts/indexnow.mjs"
  }
}
```

```bash
pnpm indexnow                                  # whole sitemap
pnpm indexnow https://example.com/pricing      # one URL
```

Responses you may see:

| Status | Meaning | Action |
|---|---|---|
| 200 | Accepted | None |
| 202 | Accepted; key validation pending | Normal on first use; none |
| 400 | Bad request | Check the JSON body |
| 403 | Key not valid | The key file is missing, wrong, or not yet live |
| 422 | URLs don't belong to the host, or the key doesn't match the format | Check `HOST` and the URL list |
| 429 | Too many requests | You are submitting too often; submit changed URLs only |

> Example from MakeDecks: the script submits the whole sitemap (a few hundred URLs) after each site deploy and once a day, and that has been accepted without problems. IndexNow asks for changed URLs only, so on a larger site use `--since` for the daily run, as the workflow below does.

### Step 3: run it from GitHub Actions

Run it after every push that changes the site, once a day, and on demand:

```yaml
# .github/workflows/indexnow.yml
name: IndexNow
on:
  push:
    branches: [main]
    paths: ["app/**", "components/**", "content/**", "lib/**", "public/**"]
  schedule:
    - cron: "17 4 * * *"
  workflow_dispatch:
jobs:
  submit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
      # The host deploys on its own after a push; give it time before reading the live sitemap.
      - if: github.event_name == 'push'
        run: sleep 180
      - name: Submit
        env:
          INDEXNOW_HOST: example.com
          INDEXNOW_PUBLIC_DIR: public
        run: |
          if [ "${{ github.event_name }}" = "schedule" ]; then
            node scripts/indexnow.mjs --since=$(date -u -d '2 days ago' +%F)
          else
            node scripts/indexnow.mjs
          fi
```

Notes:

- **`paths`:** list the folders that change public pages. In a monorepo, use the web app and any shared package it reads (for example `apps/web/**` and `packages/shared/**`). Pushes that only touch the API or docs then don't trigger a submission.
- **`sleep 180`:** the script reads the live site, so it has to run after your host has deployed the push. Three minutes covers a typical Cloudflare or Vercel build. The key file check waits a further 5 minutes if needed.
- **The cron minute:** `17 4 * * *` is 04:17 UTC. Pick an odd minute; GitHub delays jobs scheduled on the hour when its runners are busy.
- **No secrets needed.** The key is public and sits in the repo.
- **`workflow_dispatch`** adds a **Run workflow** button under the Actions tab for a manual run.

If your host reports deployments back to GitHub (Vercel and Netlify do), you can trigger on the real deploy instead of sleeping:

```yaml
on:
  deployment_status:
jobs:
  submit:
    if: github.event.deployment_status.state == 'success' && github.event.deployment.environment == 'Production'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
      - run: node scripts/indexnow.mjs
        env:
          INDEXNOW_HOST: example.com
```

Check the environment name your host uses in a past run's event payload; it may differ from `Production`.

### Step 4: ping from server code for runtime pages

Pages created without a deploy (blog posts written by a cron job, user-published public pages you want indexed) won't trigger the workflow. Ping IndexNow from the code that publishes them:

```ts
// lib/indexnow.ts
import { SITE_URL } from "@/lib/site";

// Public by design: the same value as the key file in public/.
const INDEXNOW_KEY = process.env.INDEXNOW_KEY ?? "YOUR_KEY";

export async function indexNow(urls: string[]) {
  try {
    const res = await fetch("https://api.indexnow.org/indexnow", {
      method: "POST",
      headers: { "content-type": "application/json; charset=utf-8" },
      body: JSON.stringify({
        host: new URL(SITE_URL).host,
        key: INDEXNOW_KEY,
        keyLocation: `${SITE_URL}/${INDEXNOW_KEY}.txt`,
        urlList: urls,
      }),
      signal: AbortSignal.timeout(15_000),
    });
    if (res.status >= 400) console.warn("indexnow refused", res.status, (await res.text()).slice(0, 200));
  } catch (e) {
    console.warn("indexnow failed", e);
  }
}
```

Call it after the page is saved and live, without making the user or the job wait on it.

In a Next.js server action or route handler, use `after` (Next.js 15.1 and later), which runs once the response is sent:

```ts
import { after } from "next/server";
import { indexNow } from "@/lib/indexnow";
import { SITE_URL } from "@/lib/site";

export async function publishPost(slug: string) {
  await savePost(slug);
  after(() => indexNow([`${SITE_URL}/blog/${slug}`]));
}
```

In a Cloudflare Worker (a cron or queue handler), use `ctx.waitUntil(indexNow([url]))`, or simply `await` it at the end of a background job, since the 15-second timeout bounds it.

Rules:

- Ping only when the page really is public and returns 200. A ping for a page that 404s wastes the engine's trust in your submissions.
- Ping on create and on meaningful updates. Ping again on delete too; the engine then re-crawls and drops the URL.
- Also add the page to your sitemap (chapter 1), so Google finds it.
- If your logs go to an error tracker, log failures there. A silent IndexNow failure is harmless, but you want to notice if the key file goes missing.

> Example from MakeDecks: the blog writer runs on a cron in the API Worker, publishes to the database, then pings IndexNow with the new post's URL.

### Verify IndexNow

1. `curl -s https://example.com/<key>.txt` prints the key.
2. Run `pnpm indexnow` locally and look for `-> 200` or `-> 202`.
3. In the GitHub Actions tab, open the latest **IndexNow** run and check the same line in the log.
4. A day or two later, Bing Webmaster Tools **> IndexNow** lists the URLs it received, with dates.
5. Bing's **URL Inspection** on a new URL should show it crawled within days, not weeks.

---

## 2.4 Video SEO

Videos can appear in Google's video results and the video tab, which is often less competitive than web results. Google only shows a video for a page where the video is the main content.

### Watch page requirements

Each video gets its own page, for example `/watch/<slug>`. If the video sits lower on a long landing page, Search Console reports **"Video isn't on a watch page"** and won't show it. Keep the video on the landing page too if you like; the watch page is what Google indexes for video.

| Requirement | How |
|---|---|
| The video is the main content | Title, the player near the top, a one-paragraph description, then supporting content |
| One video per page | A page per video, not a gallery |
| The player is in the HTML | A real `<video>` element (or a standard embed) in the server render, not mounted on click |
| A crawlable thumbnail | `poster` image at a stable URL, at least 60 by 30 px; 1280 by 720 JPEG works |
| A crawlable video file | `contentUrl` returns the file with a 200 or 206; not blocked by robots.txt |
| A transcript | The full transcript as text on the page; also helps people who can't play sound |
| Captions | A WebVTT `<track>` |
| Structured data | VideoObject plus BreadcrumbList |
| Stable URLs | Version the files with `?v=N` instead of renaming them |
| Indexable page | In the sitemap, no noindex, a canonical to itself |

### One registry for videos

```ts
// lib/videos.ts
export const VIDEO_VERSION = 1;
export const VIDEO_UPLOADED = "2026-01-15";

export const VIDEOS = [
  {
    slug: "pdf-to-presentation",
    title: "Turn a PDF into a presentation",
    description: "Upload a PDF report and get an editable deck in under a minute.",
    seconds: 48,
    transcript: "<Full transcript of the voice-over, as plain text.>",
  },
] as const;

export type VideoSlug = (typeof VIDEOS)[number]["slug"];
export const video = (slug: VideoSlug) => VIDEOS.find((v) => v.slug === slug)!;

export const videoFile = (slug: VideoSlug) => `/api/videos/${slug}.mp4?v=${VIDEO_VERSION}`;
export const videoPoster = (slug: VideoSlug) => `/videos/${slug}.jpg?v=${VIDEO_VERSION}`;
export const videoCaptions = (slug: VideoSlug) => `/videos/${slug}.vtt?v=${VIDEO_VERSION}`;

export const isoDuration = (s: number) => {
  const t = Math.round(s);
  return `PT${Math.floor(t / 60) ? `${Math.floor(t / 60)}M` : ""}${t % 60}S`;
};
```

Bump `VIDEO_VERSION` whenever you replace a file. The files are cached for a week, and the new query string makes browsers and crawlers fetch the new one.

Add the watch pages to the sitemap from the same registry, in the `staticPages` list from chapter 1:

```ts
...VIDEOS.map((v) => `/watch/${v.slug}`),
```

### The watch page

```tsx
// components/watch-page.tsx
import type { Metadata } from "next";
import { JsonLd } from "@/components/json-ld";
import { SITE_NAME, SITE_URL } from "@/lib/site";
import { isoDuration, video, VIDEO_UPLOADED, videoCaptions, videoFile, videoPoster, type VideoSlug } from "@/lib/videos";

export function watchMetadata(slug: VideoSlug): Metadata {
  const v = video(slug);
  const url = `${SITE_URL}/watch/${slug}`;
  return {
    title: `${v.title} (video)`,
    description: v.description,
    alternates: { canonical: url },
    openGraph: {
      type: "video.other",
      siteName: SITE_NAME,
      url,
      title: v.title,
      description: v.description,
      images: [{ url: `${SITE_URL}${videoPoster(slug)}`, width: 1280, height: 720, alt: v.title }],
      videos: [{ url: `${SITE_URL}${videoFile(slug)}`, type: "video/mp4", width: 1280, height: 720 }],
    },
  };
}

export function WatchPage({ slug }: { slug: VideoSlug }) {
  const v = video(slug);
  const url = `${SITE_URL}/watch/${slug}`;
  const ld = {
    "@context": "https://schema.org",
    "@graph": [
      {
        "@type": "VideoObject",
        "@id": `${url}#video`,
        name: v.title,
        description: v.description,
        thumbnailUrl: [`${SITE_URL}${videoPoster(slug)}`],
        uploadDate: `${VIDEO_UPLOADED}T00:00:00+00:00`,
        duration: isoDuration(v.seconds),
        contentUrl: `${SITE_URL}${videoFile(slug)}`,
        transcript: v.transcript,
        inLanguage: "en",
        publisher: { "@id": `${SITE_URL}/#organization` },
      },
      {
        "@type": "BreadcrumbList",
        itemListElement: [
          { "@type": "ListItem", position: 1, name: SITE_NAME, item: SITE_URL },
          { "@type": "ListItem", position: 2, name: "Videos", item: `${SITE_URL}/watch` },
          { "@type": "ListItem", position: 3, name: v.title, item: url },
        ],
      },
    ],
  };

  return (
    <main>
      <JsonLd data={ld} />
      <h1>{v.title}</h1>
      <video poster={videoPoster(slug)} preload="none" controls playsInline aria-label={v.title} style={{ width: "100%", aspectRatio: "16 / 9" }}>
        <source src={videoFile(slug)} type="video/mp4" />
        <track kind="captions" src={videoCaptions(slug)} srcLang="en" label="English" />
      </video>
      <p>{v.description}</p>
      <h2>Transcript</h2>
      <p>{v.transcript}</p>
      {/* links to the related landing page, other videos, and one sign-up CTA */}
    </main>
  );
}
```

```tsx
// app/watch/pdf-to-presentation/page.tsx
import { WatchPage, watchMetadata } from "@/components/watch-page";

export const metadata = watchMetadata("pdf-to-presentation");

export default function Page() {
  return <WatchPage slug="pdf-to-presentation" />;
}
```

- The `JsonLd` helper is the one from chapter 1, section 1.7.
- If you have no `/watch` index page, drop the middle breadcrumb and link the related landing page instead.
- `preload="none"` with a `poster` keeps the page fast; the file loads only when someone presses play.

VideoObject properties:

| Property | Required by Google? | Notes |
|---|---|---|
| `name` | Yes | The video title |
| `thumbnailUrl` | Yes | Absolute URL; an array is allowed |
| `uploadDate` | Yes | ISO 8601 with a time zone |
| `description` | Recommended | Match the visible description |
| `duration` | Recommended | ISO 8601, for example `PT48S` or `PT1M30S` |
| `contentUrl` | Recommended | The file itself; lets Google fetch the video |
| `embedUrl` | Recommended if you use a third-party player | The player URL, for example a YouTube embed |
| `transcript` | Optional | The same text shown on the page |

### Encode for the web

```bash
ffmpeg -i input.mov -vf scale=-2:720 -c:v libx264 -crf 23 -preset slow \
  -c:a aac -b:a 128k -movflags +faststart public-video.mp4
```

`-movflags +faststart` moves the index to the start of the file so playback can begin before the whole file downloads. H.264 720p plays everywhere and keeps a one-minute demo around 2 to 3 MB.

---

## 2.5 Serving video with byte ranges (206)

Safari, and iOS in particular, won't play a video unless the server answers `Range` requests with `206 Partial Content`. Chrome falls back to a full download, so the bug often goes unnoticed until someone opens the page on an iPhone.

First test whether your current host already does it:

```bash
curl -sI -H "Range: bytes=0-1" https://example.com/videos/demo.mp4 | grep -iE "^(HTTP|content-range|accept-ranges)"
# Good:
# HTTP/2 206
# accept-ranges: bytes
# content-range: bytes 0-1/2400000
```

Most static hosts answer with 206 for files in `public/`. If yours returns `200` with the whole file, serve videos from a small route like the ones below.

> Example from MakeDecks: neither the web Worker nor Cloudflare static assets answered ranges, so videos are stored in R2 and served by a route on the API Worker.

### Cloudflare Workers and R2 (Hono)

R2 handles the range itself: pass it the request headers and it returns only the requested bytes plus the range it used.

```ts
// api/src/videos.ts
import { Hono } from "hono";

type Env = { MEDIA: R2Bucket };

export function byteRange(r: { offset?: number; length?: number; suffix?: number }, size: number) {
  const start = r.suffix !== undefined ? Math.max(0, size - r.suffix) : (r.offset ?? 0);
  const length = r.suffix !== undefined ? size - start : (r.length ?? size - start);
  return { start, end: start + length - 1, length };
}

export const videos = new Hono<{ Bindings: Env }>();

videos.get("/:file", async (c) => {
  const file = c.req.param("file");
  if (!/^[a-z0-9-]{1,60}\.mp4$/.test(file)) return c.text("Video not found", 404);
  const ranged = c.req.header("Range")?.startsWith("bytes=");
  const obj = await c.env.MEDIA.get(`site/videos/${file}`, ranged ? { range: c.req.raw.headers } : undefined);
  if (!obj) return c.text("Video not found", 404);
  const headers = new Headers({
    "Content-Type": "video/mp4",
    "Accept-Ranges": "bytes",
    "Cache-Control": "public, max-age=604800",
    ETag: obj.httpEtag,
  });
  if (ranged && obj.range) {
    const { start, end, length } = byteRange(obj.range, obj.size);
    headers.set("Content-Range", `bytes ${start}-${end}/${obj.size}`);
    headers.set("Content-Length", String(length));
    return new Response(obj.body, { status: 206, headers });
  }
  headers.set("Content-Length", String(obj.size));
  return new Response(obj.body, { headers });
});
```

```ts
// api/src/index.ts
app.route("/api/videos", videos);
```

Notes:

- The filename regex is the only input check needed: it blocks path traversal and keeps the route to `.mp4` files under one prefix.
- `byteRange` turns R2's `{ offset, length }` or `{ suffix }` into the numbers `Content-Range` needs. A request for `bytes=0-` has no `length`, so it runs to the end of the file.
- Add a per-IP rate limit (for example the Workers Rate Limiting binding) if the route is public and the files are large.
- Upload a file with Wrangler:
  ```bash
  npx wrangler r2 object put <bucket>/site/videos/pdf-to-presentation.mp4 --file=./pdf-to-presentation.mp4 --remote
  ```
- `robots.txt` must allow the path (`Allow: /api/videos/` in chapter 1), or Google can't fetch `contentUrl`.
- Keep posters (`.jpg`) and captions (`.vtt`) in `public/videos/`. They don't need ranges.

### Next.js route handler on Node

For a Node server that keeps files on disk:

```ts
// app/api/videos/[file]/route.ts
import { createReadStream, statSync } from "node:fs";
import path from "node:path";
import { Readable } from "node:stream";

export const runtime = "nodejs";
const DIR = path.join(process.cwd(), "media", "videos");

export async function GET(req: Request, { params }: { params: Promise<{ file: string }> }) {
  const { file } = await params;
  if (!/^[a-z0-9-]{1,60}\.mp4$/.test(file)) return new Response("Video not found", { status: 404 });
  const full = path.join(DIR, file);
  let size: number;
  try {
    size = statSync(full).size;
  } catch {
    return new Response("Video not found", { status: 404 });
  }
  const headers = new Headers({ "Content-Type": "video/mp4", "Accept-Ranges": "bytes", "Cache-Control": "public, max-age=604800" });
  const m = /^bytes=(\d*)-(\d*)$/.exec(req.headers.get("range") ?? "");
  if (!m || (!m[1] && !m[2])) {
    headers.set("Content-Length", String(size));
    return new Response(Readable.toWeb(createReadStream(full)) as ReadableStream, { headers });
  }
  const start = m[1] ? Number(m[1]) : Math.max(0, size - Number(m[2]));
  const end = m[1] && m[2] ? Math.min(Number(m[2]), size - 1) : size - 1;
  if (start >= size || start > end) {
    return new Response(null, { status: 416, headers: { "Content-Range": `bytes */${size}` } });
  }
  headers.set("Content-Range", `bytes ${start}-${end}/${size}`);
  headers.set("Content-Length", String(end - start + 1));
  return new Response(Readable.toWeb(createReadStream(full, { start, end })) as ReadableStream, { status: 206, headers });
}
```

- `bytes=100-` reads from 100 to the end; `bytes=-500` reads the last 500 bytes; `bytes=0-1` reads two bytes. These are the forms browsers send.
- Multi-range requests (`bytes=0-1,5-9`) don't match the regex and get the whole file with a 200, which is allowed.
- `createReadStream` treats `end` as inclusive, matching the `Range` header.

### Check it

```bash
URL="https://example.com/api/videos/pdf-to-presentation.mp4?v=1"
curl -sI "$URL" | grep -iE "^(HTTP|content-length|accept-ranges)"
# HTTP/2 200, accept-ranges: bytes

curl -sI -H "Range: bytes=0-1" "$URL" | grep -iE "^(HTTP|content-range|content-length)"
# HTTP/2 206, content-range: bytes 0-1/<size>, content-length: 2

curl -sI -H "Range: bytes=-100" "$URL" | grep -iE "^(HTTP|content-range)"
# HTTP/2 206, content-range: bytes <size-100>-<size-1>/<size>
```

Then open the watch page in Safari on a Mac or iPhone and press play.

### Verify the video is indexed

1. Rich Results Test on the watch page: **Videos** is detected with no errors.
2. URL Inspection: **Test live URL**, then **Request indexing** for each watch page.
3. A few weeks later, Search Console **Indexing > Videos** (Video pages) lists the pages where a video was indexed, and reasons for any that weren't. "Video isn't on a watch page" means the video is not the main content; "Cannot determine video position and size" usually means the player is hidden or mounted late.

---

## Checklist

- [ ] Search Console domain property added and verified by a DNS TXT record; the record stays.
- [ ] At least two owners on the property.
- [ ] Sitemap submitted; status is Success and the discovered count matches your `<loc>` count.
- [ ] URL Inspection's live test shows title, description and canonical in `<head>`, and a rendered screenshot.
- [ ] Request indexing used for about 10 high-value URLs a day after launches, highest value first.
- [ ] Page indexing report reviewed: noindex and redirect exclusions are expected; no "Indexed, though blocked by robots.txt".
- [ ] Bing Webmaster Tools imported from Search Console (or verified manually) with the sitemap listed.
- [ ] IndexNow key file in `public/`, live at `https://example.com/<key>.txt`.
- [ ] `scripts/indexnow.mjs` runs locally and returns 200 or 202.
- [ ] GitHub Actions workflow runs after site deploys, daily with `--since`, and on demand.
- [ ] Server code pings IndexNow for pages published at runtime.
- [ ] Bing Webmaster Tools IndexNow screen shows received URLs.
- [ ] Every video has a watch page with the player in the HTML, a transcript, captions and VideoObject plus BreadcrumbList.
- [ ] Watch pages are in the sitemap.
- [ ] Video files answer `Range` requests with 206; Safari plays them.
- [ ] `robots.txt` allows the video file path.
- [ ] Files are versioned with `?v=N` and the version is bumped on every replacement.

## Common mistakes

| Mistake | What happens | Fix |
|---|---|---|
| URL-prefix property for `https://www.example.com` only | Apex and `http` data missing from reports | Use a domain property |
| Deleting the verification TXT record after verifying | Property becomes unverified | Leave it forever |
| One owner on Search Console | Access lost when they leave | Add a second owner |
| Requesting indexing for every page every day | Quota runs out on low-value pages | Prioritise; let the sitemap do the rest |
| Using the Indexing API for normal pages | Against Google's rules; may be ignored | Use the sitemap and URL Inspection |
| Expecting IndexNow to help with Google | Google doesn't use it | Sitemap and Search Console for Google |
| IndexNow submitted before the key file is live | 403 | Wait for the key file (the script polls) |
| Renaming or deleting the key file | Later submissions fail | Keep the file forever |
| Submitting URLs from another host or `www` | 422 for the whole batch | Submit only canonical URLs on your host |
| Submitting the full sitemap many times a day | 429, and engines trust your pings less | After deploys plus a daily `--since` run |
| The workflow runs before the deploy finishes | Old sitemap submitted | `sleep` or trigger on `deployment_status` |
| Pinging IndexNow before the page is live | The engine sees a 404 | Ping after the save, once it returns 200 |
| Video only lower on a long landing page | "Video isn't on a watch page" | One watch page per video |
| Player mounted on click or by a deferred component | Google can't find the video | Render `<video>` in the server HTML |
| No transcript | Less text to rank; worse accessibility | Put the full transcript on the page |
| Video file served with 200 only | Won't play in Safari or on iPhones | Serve 206 for `Range` requests |
| `Disallow: /api/` with videos under `/api/` | Google can't fetch `contentUrl` | `Allow: /api/videos/` |
| Replacing a video file without a new version | Old file served from caches for a week | Bump `VIDEO_VERSION` |
