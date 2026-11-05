# 1. Technical foundations

One host, clear crawl rules, correct metadata and fast pages, so every page you publish later can rank.

## What you'll set up

- One canonical host, with `www`, `http` and any extra domains 301-redirected to it.
- A single `SITE_URL` constant that every canonical, sitemap entry, Open Graph tag and JSON-LD object reads.
- A `robots.txt` that blocks only machine paths, never app pages.
- `X-Robots-Tag: noindex, nofollow` headers on every app page.
- A sitemap built from your page registries.
- Static metadata on every public route, with length rules.
- A head-injection fix for frameworks that stream `generateMetadata` into `<body>`.
- JSON-LD for each page type, rendered by one small helper.
- Speed work: deferred mounting, `srcset` images and a late-loading analytics script.
- Mobile width checks and a text-to-markup check.

The code is Next.js App Router with TypeScript. Notes for other stacks sit next to the parts where they differ.

---

## 1.1 Domain and canonical host

### Pick the apex

Choose one host and send everything else to it with a permanent redirect. Use the apex (`https://example.com`) unless you have a reason not to:

| Choice | Why |
|---|---|
| `https://example.com` (apex) | Shorter, matches what people type, and what most SaaS products use. |
| `https://www.example.com` | Fine too. Pick it only if your DNS or host makes apex records hard. |

What matters is that only one host serves pages. If both `www` and the apex answer with 200, Google may split signals across two copies of every page.

Every other host 301s to the same path on the canonical host:

| Request | Response |
|---|---|
| `http://example.com/pricing` | 301 to `https://example.com/pricing` |
| `https://www.example.com/pricing?a=1` | 301 to `https://example.com/pricing?a=1` |
| `https://example.io/pricing` (extra TLD) | 301 to `https://example.com/pricing` |
| `https://www.example.io/` | 301 to `https://example.com/` |

Keep the path and query string. Sending every old URL to the homepage throws away the link value of deep links.

### One constant for the canonical URL

```ts
// lib/site.ts
export const SITE_URL = "https://example.com";
export const SITE_NAME = "<Brand>";
export const SITE_TITLE = "<Brand>: <what it does in a few words>";
export const SITE_DESCRIPTION = "<One or two sentences, 70 to 160 characters, saying what the product does and for whom.>";
```

Canonicals, the sitemap, Open Graph URLs, JSON-LD `@id`s and any server code that builds absolute links all import `SITE_URL`. Never write the domain anywhere else.

Pick one trailing-slash style and stay with it. Next.js defaults to no trailing slash (`/pricing`, not `/pricing/`), and redirects the other form for you.

### Cloudflare: redirect `www` to the apex

Single Redirect rules run at the edge before your Worker or origin, so redirected requests never reach your app.

1. In the Cloudflare dashboard, open the zone `example.com`.
2. **DNS > Records:** make sure a `www` record exists and is **Proxied** (orange cloud). If you have none, add `CNAME`, name `www`, target `example.com`, Proxied. Without a proxied record, the redirect rule never sees the request.
3. **Rules > Redirect Rules** (Single Redirects) **> Create rule**.
4. Rule name: `www to apex`.
5. Under **If incoming requests match**, choose **Custom filter expression**:
   - Field: `Hostname`
   - Operator: `equals`
   - Value: `www.example.com`

   The expression preview reads:
   ```
   (http.host eq "www.example.com")
   ```
6. Under **Then**:
   - Type: **Dynamic**
   - Expression:
     ```
     concat("https://example.com", http.request.uri.path)
     ```
   - Status code: **301**
   - **Preserve query string:** on
7. **Deploy**.

Also turn on **SSL/TLS > Edge Certificates > Always Use HTTPS**, which handles `http` to `https`.

### Cloudflare: redirect an extra TLD

For a second domain (`example.io`, `example.in` and so on) that should only redirect:

1. Add `example.io` as its own zone in Cloudflare and point the registrar's nameservers at Cloudflare.
2. **DNS > Records** in the `example.io` zone: the domain needs a proxied record even though nothing serves it. Add:
   - `AAAA`, name `@`, content `100::`, **Proxied**
   - `AAAA`, name `www`, content `100::`, **Proxied**

   `100::` is a discard address. The request never reaches it, because the redirect answers first. If you prefer, attach the domain to your Worker as a custom domain instead, which also creates a proxied record.
3. **Rules > Redirect Rules > Create rule** in the `example.io` zone:
   - Rule name: `all to example.com`
   - If incoming requests match: **All incoming requests**
   - Then: Type **Dynamic**, expression `concat("https://example.com", http.request.uri.path)`, status **301**, **Preserve query string** on.
4. **Deploy**.

> Real-world example: the `.in` domain stays attached to the web Worker only so its proxied DNS record exists. The redirect rule on that zone sends every request to the same path on the `.com` before the Worker runs.

### nginx equivalent

```nginx
# Every non-canonical host and all plain http: one permanent redirect, path and query kept.
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com example.io www.example.io;
    return 301 https://example.com$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name www.example.com example.io www.example.io;
    ssl_certificate     /etc/ssl/example/fullchain.pem;
    ssl_certificate_key /etc/ssl/example/privkey.pem;
    return 301 https://example.com$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name example.com;
    # your real site config
}
```

`$request_uri` includes the path and the query string. The certificate on the redirect server must cover every host it answers for.

### Vercel equivalent

The simplest way is in the dashboard: **Project > Settings > Domains**, add `www.example.com` and `example.io`, and set each to **Redirect to** `example.com`. Choose the permanent status.

Or in `vercel.json`:

```json
{
  "redirects": [
    {
      "source": "/:path*",
      "has": [{ "type": "host", "value": "www.example.com" }],
      "destination": "https://example.com/:path*",
      "permanent": true
    },
    {
      "source": "/:path*",
      "has": [{ "type": "host", "value": "example.io" }],
      "destination": "https://example.com/:path*",
      "permanent": true
    }
  ]
}
```

`permanent: true` sends a 308. Google treats 308 the same as 301.

### Check it

```bash
curl -sI "https://www.example.com/pricing?x=1" | grep -iE "^(HTTP|location)"
# HTTP/2 301
# location: https://example.com/pricing?x=1

curl -sI "http://example.com/pricing" | grep -iE "^(HTTP|location)"
curl -sI "https://example.io/pricing" | grep -iE "^(HTTP|location)"
curl -sI "https://example.com/pricing" | grep -iE "^HTTP"
# HTTP/2 200
```

Each non-canonical URL must answer with exactly one redirect straight to the final URL. A chain such as `http://www` to `https://www` to `https://` wastes crawl budget; fix it by making the first hop go straight to the apex.

---

## 1.2 robots.txt

### The rule

Block only paths that are never pages: your API and your analytics proxy. Everything else stays crawlable, including the app.

```ts
// app/robots.ts
import type { MetadataRoute } from "next";
import { SITE_URL } from "@/lib/site";

export default function robots(): MetadataRoute.Robots {
  return {
    rules: {
      userAgent: "*",
      allow: ["/", "/api/videos/"],
      disallow: ["/api/", "/ingest/"],
    },
    sitemap: `${SITE_URL}/sitemap.xml`,
  };
}
```

This renders:

```
User-Agent: *
Allow: /
Allow: /api/videos/
Disallow: /api/
Disallow: /ingest/

Sitemap: https://example.com/sitemap.xml
```

- `/ingest/` is a placeholder for a first-party analytics proxy path. Use whatever path yours has, or drop it.
- `Allow: /api/videos/` lets Google fetch public media served by your API (see chapter 2). Google uses the longest matching rule, so the more specific `Allow` wins over `Disallow: /api/`.

### Why app pages are not blocked here

It feels natural to add `Disallow: /dashboard`. Don't.

- `robots.txt` controls crawling, not indexing. A blocked URL can still be indexed from links, shown as a bare URL with no title or snippet.
- Google can't crawl a blocked page, so it never sees the page's `noindex`. The noindex has no effect.
- The fix is the reverse: let Google crawl app pages and tell it, on each response, not to index them (section 1.3).

> Real-world example: app pages were blocked in robots.txt at first and started showing up in the index as bare URLs. Removing them from robots.txt and sending `X-Robots-Tag` instead fixed it.

### Other stacks

Serve a static `public/robots.txt` with the same content. The rules are the same everywhere.

---

## 1.3 noindex for app pages with X-Robots-Tag

Send the header on every response under an app path. A header covers HTML, JSON and files alike, and doesn't depend on a `<meta>` tag being rendered in `<head>`.

```ts
// next.config.ts
import type { NextConfig } from "next";

// Top-level path segments that are app, not marketing. Each one covers "/x" and "/x/anything".
const noindex = ["login", "signup", "dashboard", "app", "settings", "admin", "print", "s", "invite", "unsubscribe", "api", "ingest"];

const nextConfig: NextConfig = {
  async headers() {
    return [
      ...noindex
        .flatMap((p) => [`/${p}`, `/${p}/:path*`])
        .map((source) => ({
          source,
          headers: [{ key: "X-Robots-Tag", value: "noindex, nofollow" }],
        })),
    ];
  },
};

export default nextConfig;
```

Both forms are needed: `/dashboard/:path*` does not match `/dashboard` itself in every setup, so list both.

Which paths to include:

| Path type | Index? | Why |
|---|---|---|
| Marketing pages, docs, blog, tools, pricing | Yes | These are what you want found. |
| Login, sign-up, password reset | No | Thin, duplicate across sites, no search value. |
| Dashboard, editor, settings, admin | No | Private or empty for a crawler. |
| User share links (`/s/<token>`) | No | User content you don't control; may be private by obscurity. |
| Print and export views | No | Duplicates of other pages. |
| Unsubscribe and invite links | No | One-time, token-bearing URLs. |
| Embeddable widgets (`/embed/...`) | No | Their content lives on other sites; the credit link is what you want. |

### One deployment for site and app

When the marketing site and the app ship as one deployment, a new protected path has to be added in three places at once:

1. The auth gate (middleware or proxy matcher) that sends signed-out users to login.
2. The `noindex` list in `next.config.ts`.
3. Any route list used by other tools (an "is this a known path" check, a Markdown-for-agents handler, and so on).

Put a short line in your contributor guide so nobody adds a route to only one of them.

### Alternative inside the page

For a whole route group you can also set it in a layout:

```ts
// app/(app)/layout.tsx
export const metadata = { robots: { index: false, follow: false } };
```

Prefer the header. It works even when metadata is streamed into the body (section 1.6) and covers non-HTML responses.

### Check it

```bash
curl -sI https://example.com/dashboard | grep -i x-robots-tag
# x-robots-tag: noindex, nofollow

curl -sI https://example.com/pricing | grep -i x-robots-tag
# (nothing)
```

---

## 1.4 Sitemap built from registries

Keep each family of pages (use cases, tools, templates, comparisons, videos) in one registry: an array of objects in a shared file. The routes, hub pages, footer, `llms.txt` and the sitemap all read the same array, so adding an entry adds the page everywhere.

```ts
// lib/registries.ts
export const USE_CASES = [
  { slug: "pdf-to-presentation", label: "PDF to presentation" },
  { slug: "investor-update", label: "Investor update" },
] as const;

export const TOOLS = [
  { slug: "word-counter", name: "Word counter" },
] as const;

export const COMPARISONS = [
  { slug: "rival-a", label: "<Brand> vs Rival A" },
] as const;
```

```ts
// app/sitemap.ts
import type { MetadataRoute } from "next";
import { SITE_URL } from "@/lib/site";
import { COMPARISONS, TOOLS, USE_CASES } from "@/lib/registries";
import { listPosts } from "@/lib/blog";

// Bump this when the static pages really change. Google ignores lastmod values it learns are not accurate.
const lastModified = "2026-01-15";
export const revalidate = 300;

export const staticPages = (): MetadataRoute.Sitemap =>
  [
    "",
    "/pricing",
    "/about",
    "/contact",
    "/privacy",
    "/use-cases",
    "/tools",
    "/blog",
    ...USE_CASES.map((u) => `/for/${u.slug}`),
    ...TOOLS.map((t) => `/tools/${t.slug}`),
    ...COMPARISONS.map((c) => `/vs/${c.slug}`),
  ].map((path) => ({ url: `${SITE_URL}${path}`, lastModified }));

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const posts = await listPosts(); // [{ slug, updated }], newest first
  const pages = staticPages().map((p) =>
    p.url === `${SITE_URL}/blog` && posts.length
      ? { ...p, lastModified: posts.map((x) => x.updated).sort().at(-1) }
      : p,
  );
  return [...pages, ...posts.map((p) => ({ url: `${SITE_URL}/blog/${p.slug}`, lastModified: p.updated }))];
}
```

Notes:

- `staticPages` is exported so other code (a "known path" check, a Markdown handler, a test) can reuse the list.
- `revalidate = 300` regenerates the sitemap at most every 5 minutes, so new database-backed posts appear without a deploy.
- `listPosts` is whatever reads your posts. If it can fail, catch the error and return `[]`, so a database hiccup never breaks the whole sitemap.
- Only list canonical, indexable URLs with a 200 status. No redirects, no noindexed pages, no URLs with query strings.
- `changefreq` and `priority` are ignored by Google. Leave them out.
- One sitemap holds up to 50,000 URLs. Past that, use `generateSitemaps` to split it.

### A test that keeps registries honest

A small test catches a registry entry with no route, which would put a 404 in the sitemap:

```ts
// lib/sitemap.test.ts (vitest)
import { existsSync } from "node:fs";
import { expect, test } from "vitest";
import { staticPages } from "@/app/sitemap";
import { SITE_URL } from "@/lib/site";

test("every static sitemap URL has a route file", () => {
  for (const { url } of staticPages()) {
    const path = url.slice(SITE_URL.length) || "/";
    const file = `app${path === "/" ? "" : path}/page.tsx`;
    expect(existsSync(file), file).toBe(true);
  }
});
```

Adjust the file check if your routes use route groups like `app/(marketing)/...`.

### Check it

```bash
curl -s https://example.com/sitemap.xml | grep -c "<loc>"
curl -s https://example.com/sitemap.xml | grep -o "<loc>[^<]*</loc>" | head
```

Open a few of the listed URLs and confirm each answers 200 with no `X-Robots-Tag: noindex`.

---

## 1.5 Metadata per static route

### Root layout defaults

```ts
// app/layout.tsx
import type { Metadata } from "next";
import { SITE_DESCRIPTION, SITE_NAME, SITE_TITLE, SITE_URL } from "@/lib/site";

export const metadata: Metadata = {
  metadataBase: new URL(SITE_URL),
  title: { default: SITE_TITLE, template: `%s · ${SITE_NAME}` },
  description: SITE_DESCRIPTION,
  applicationName: SITE_NAME,
  alternates: { canonical: "./" },
  icons: { icon: "/favicon.svg", apple: "/apple-touch-icon.png" },
  openGraph: {
    type: "website",
    siteName: SITE_NAME,
    locale: "en_US",
    url: "./",
    images: [{ url: "/og.png", width: 1200, height: 630, alt: `${SITE_NAME}, <short description>` }],
  },
  twitter: { card: "summary_large_image" },
};
```

- `metadataBase` turns relative URLs (`/og.png`, `./`) into absolute ones.
- `alternates.canonical: "./"` gives every page a self-referencing canonical by default. Pages with a query string still canonicalise to the clean path.
- The `template` adds ` · <Brand>` to each page title. Count it in the length budget.

### One helper per page family

Programmatic pages share a template, so they share a metadata helper. Each page is its own static route with a static `metadata` export:

```ts
// app/for/use-case.tsx
import type { Metadata } from "next";
import { SITE_NAME, SITE_URL } from "@/lib/site";
import { CASES, type UseCaseSlug } from "./data";

export function useCaseMetadata(slug: UseCaseSlug): Metadata {
  const c = CASES[slug];
  const url = `${SITE_URL}/for/${slug}`;
  return {
    title: c.title,
    description: c.description,
    alternates: { canonical: url },
    openGraph: {
      type: "website",
      siteName: SITE_NAME,
      url,
      title: `${c.title} · ${SITE_NAME}`,
      description: c.description,
      images: [{ url: "/og.png", width: 1200, height: 630, alt: `${SITE_NAME}, <short description>` }],
    },
  };
}
```

```tsx
// app/for/investor-update/page.tsx
import { UseCasePage, useCaseMetadata } from "../use-case";

export const metadata = useCaseMetadata("investor-update");

export default function Page() {
  return <UseCasePage slug="investor-update" />;
}
```

A folder per slug looks repetitive next to a `[slug]` dynamic route. It is on purpose: a static `metadata` export always lands in `<head>`, while `generateMetadata` on a dynamic route may be streamed (section 1.6). A generator script or a few lines of copy-paste per page is a small price.

### Length rules

| Field | Rule | Notes |
|---|---|---|
| `<title>` | 70 characters or fewer, including the ` · <Brand>` suffix | Google cuts by pixel width, roughly 55 to 60 characters on desktop. Put the query words first. |
| Meta description | 70 to 160 characters | Shorter looks thin; longer is cut. Say what the page gives, not "Welcome to". |
| H1 | One per page, close to the title, not identical | Write it for people. |
| Canonical | Absolute URL on `SITE_URL` | Self-referencing unless the page is a true duplicate. |
| `og:image` | 1200 by 630 | One default image is fine; per-page images help social shares. |

Add a test so nobody ships a 95-character title:

```ts
// lib/metadata.test.ts (vitest)
import { expect, test } from "vitest";
import { SITE_NAME } from "@/lib/site";
import { CASES } from "@/app/for/data";

test.each(Object.entries(CASES))("%s has usable title and description", (_, c) => {
  expect(`${c.title} · ${SITE_NAME}`.length).toBeLessThanOrEqual(70);
  expect(c.description.length).toBeGreaterThanOrEqual(70);
  expect(c.description.length).toBeLessThanOrEqual(160);
});
```

### Check it

```bash
curl -s https://example.com/for/investor-update | grep -oE "<title>[^<]*</title>|<link rel=\"canonical\"[^>]*>|<meta name=\"description\"[^>]*>"
```

All three must appear, and they must appear before `</head>` (see the next section).

---

## 1.6 The streaming `generateMetadata` gotcha

### The problem

Some React frameworks stream the output of `generateMetadata` instead of waiting for it. The `<title>`, description and canonical then arrive inside `<body>`, after the shell has been sent. Browsers fix this up, but a crawler reading the raw HTML may not treat those tags as head metadata.

- vinext (Next.js on Vite, used on Cloudflare Workers) streams `generateMetadata` into the body for every request.
- Next.js 15.2 and later also stream metadata for normal browsers, but block for known bots by default (the list is configurable with `htmlLimitedBots` in `next.config.ts`).

Static `metadata` exports are not affected. Only dynamic routes that need data to build their metadata, such as blog posts stored in a database, hit this.

### Check whether you have it

```bash
curl -s -A "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" \
  https://example.com/blog/some-post > page.html

# Where does the head end, and where is the title?
grep -bo "</head>" page.html | head -1
grep -bo "<title>" page.html | head -1
```

If the first `<title>` offset is larger than the `</head>` offset, your metadata is in the body. Also try URL Inspection in Search Console (chapter 2), which shows the rendered HTML Google saw.

### The fix on Cloudflare Workers: HTMLRewriter

Wrap the framework's fetch handler in your own Worker entry. For blog post HTML responses, fetch the post and append the head tags with `HTMLRewriter`, which edits the stream without buffering the page.

```ts
// lib/blog-head.ts
import { SITE_NAME, SITE_URL } from "@/lib/site";

export type PostMeta = { slug: string; title: string; description: string; date: string; updated: string; author: string };

const esc = (s: string) =>
  s.replace(/&/g, "&amp;").replace(/"/g, "&quot;").replace(/</g, "&lt;").replace(/>/g, "&gt;");

export function blogHead(p: PostMeta) {
  const url = `${SITE_URL}/blog/${p.slug}`;
  const title = `${p.title} · ${SITE_NAME}`;
  const meta = (attr: "name" | "property", key: string, content: string) =>
    `<meta ${attr}="${key}" content="${esc(content)}">`;
  return [
    `<title>${esc(title)}</title>`,
    meta("name", "description", p.description),
    `<link rel="canonical" href="${esc(url)}">`,
    meta("property", "og:type", "article"),
    meta("property", "og:site_name", SITE_NAME),
    meta("property", "og:url", url),
    meta("property", "og:title", title),
    meta("property", "og:description", p.description),
    meta("property", "og:image", `${SITE_URL}/og.png`),
    meta("property", "article:published_time", p.date),
    meta("property", "article:modified_time", p.updated),
    meta("property", "article:author", p.author),
    meta("name", "twitter:card", "summary_large_image"),
    meta("name", "twitter:title", title),
    meta("name", "twitter:description", p.description),
  ].join("");
}
```

```ts
// worker.ts
// Replace this import with your framework's Worker fetch handler (for vinext: "vinext/server/fetch-handler").
import entry from "vinext/server/fetch-handler";
import { blogHead, type PostMeta } from "@/lib/blog-head";

export * from "vinext/server/fetch-handler";

type Env = { API: Fetcher };
const POST = /^\/blog\/([a-z0-9-]{1,100})\/?$/;

async function withBlogHead(req: Request, res: Response, env: Env) {
  const slug = POST.exec(new URL(req.url).pathname)?.[1];
  if (!slug || req.method !== "GET" || res.status !== 200 || !res.headers.get("content-type")?.includes("text/html")) return res;
  try {
    const r = await env.API.fetch(`https://api.internal/api/blog/posts/${slug}`);
    if (!r.ok) throw new Error(`blog post ${r.status}`);
    const tags = blogHead((await r.json()) as PostMeta);
    return new HTMLRewriter().on("head", { element: (head) => void head.append(tags, { html: true }) }).transform(res);
  } catch (e) {
    console.error("blog head failed", e);
    return res;
  }
}

export default {
  ...entry,
  async fetch(req: Request, env: Env, ctx: ExecutionContext) {
    return withBlogHead(req, await entry.fetch(req, env, ctx), env);
  },
};
```

Then point your Worker config at this file instead of the framework's default entry (for example `"main": "./worker.ts"` in `wrangler.jsonc`, or the entry option your framework's Cloudflare config exposes).

How it works and why each line matters:

- Only `GET` requests for `/blog/<slug>` with a 200 HTML response are touched. Everything else passes through.
- `env.API` is a service binding to the API Worker. Without one, call your public post API with plain `fetch` on `SITE_URL`, or read the database directly if the web Worker has the binding.
- On any failure it returns the original response. A broken head is better than a broken page.
- `esc` escapes every value, because titles and descriptions may contain quotes or `<`.
- Cache the post API response for a few minutes (`Cache-Control: public, max-age=300`) so this extra fetch is cheap.

**Keep the streamed copies.** Do not remove `generateMetadata` from the page. React hydrates against the tags it streamed; taking them out breaks hydration. The page will have the injected tags in `<head>` and the streamed copies in `<body>`. Crawlers use the first `<title>` and the canonical in `<head>`.

### Other stacks

- **Next.js on Node or Vercel:** check with the curl test above using a Googlebot user agent. If metadata is already in `<head>` for bots, do nothing. If you need it for every user agent, list a broad pattern in `htmlLimitedBots`, or render the route statically (`generateStaticParams` plus `export const dynamic = "force-static"` with `revalidate`).
- **Static site generators:** not affected; metadata is written at build time.

---

## 1.7 JSON-LD for each page type

### The helper

One server component renders any JSON-LD object safely:

```tsx
// components/json-ld.tsx
export function JsonLd({ data }: { data: object }) {
  return (
    <script
      type="application/ld+json"
      dangerouslySetInnerHTML={{ __html: JSON.stringify(data).replace(/</g, "\\u003c") }}
    />
  );
}
```

The `<` replacement matters: a FAQ answer containing `</script>` would otherwise close the tag early and inject HTML. `<` is still `<` to a JSON parser.

Render it anywhere in the page component. Google reads JSON-LD in the body as well as the head:

```tsx
<JsonLd data={{ "@context": "https://schema.org", "@type": "FAQPage", mainEntity: [/* ... */] }} />
```

When a page needs several types, use one `@graph` instead of several script tags. Link objects with `@id`, for example `publisher: { "@id": "https://example.com/#organization" }`, so the Organization is described once on the homepage and referenced everywhere else.

### Which types go where

| Page | Types |
|---|---|
| Home | Organization, WebSite, WebApplication (or SoftwareApplication), FAQPage |
| Use-case and comparison pages | FAQPage, BreadcrumbList |
| Hubs (`/use-cases`, `/tools`, `/templates`) | ItemList |
| Free tools | WebApplication, FAQPage |
| Blog posts | Article (author points to a Person page), BreadcrumbList, FAQPage when the post has a FAQ |
| Video watch pages | VideoObject, BreadcrumbList (chapter 2) |
| Author page | ProfilePage with a Person and `sameAs` links |

A note on FAQPage: since 2023 Google shows FAQ rich results only for a small set of authoritative government and health sites. Keep the markup anyway. It costs nothing, it makes the question and answer pairs explicit to other search engines and AI assistants, and the visible FAQ itself answers long-tail queries.

Only mark up what is visible on the page. Every question in FAQPage must be on the page, every item in an ItemList must be a link on the page.

### Home: Organization, WebSite, WebApplication

Google shows a site name above each result instead of the bare domain only when it trusts the name. It reads the homepage's `WebSite` `name`, then `og:site_name` and the title. Keep all three identical, give `WebSite` the homepage URL with a trailing slash and an `alternateName` for other spellings, and request a recrawl of the homepage. A new site, or a brand made of everyday words, can still show the domain for weeks until Google is confident.

```ts
// lib/site.ts (continued)
const org = `${SITE_URL}/#organization`;

export const siteJsonLd = {
  "@context": "https://schema.org",
  "@graph": [
    {
      "@type": "Organization",
      "@id": org,
      name: SITE_NAME,
      url: SITE_URL,
      logo: `${SITE_URL}/icon-512.png`,
      email: "hello@example.com",
      foundingDate: "2026-01",
      founder: { "@type": "Person", name: "<Founder Name>", url: `${SITE_URL}/about/<founder-slug>` },
      contactPoint: {
        "@type": "ContactPoint",
        contactType: "customer support",
        email: "hello@example.com",
        url: `${SITE_URL}/contact`,
        availableLanguage: ["English"],
      },
      sameAs: ["https://www.linkedin.com/company/<brand>", "https://github.com/<brand>"],
    },
    {
      "@type": "WebSite",
      "@id": `${SITE_URL}/#website`,
      url: `${SITE_URL}/`, // the homepage with its trailing slash, as Google's site-name guide shows
      name: SITE_NAME,
      alternateName: ["<Brand With Spaces>"], // other ways people write the name
      description: SITE_DESCRIPTION,
      inLanguage: "en",
      publisher: { "@id": org },
    },
    {
      "@type": "WebApplication",
      "@id": `${SITE_URL}/#app`,
      name: SITE_NAME,
      url: SITE_URL,
      image: `${SITE_URL}/og.png`,
      applicationCategory: "BusinessApplication",
      operatingSystem: "Web",
      browserRequirements: "Requires JavaScript",
      description: SITE_DESCRIPTION,
      isAccessibleForFree: true,
      publisher: { "@id": org },
      offers: { "@type": "Offer", price: "0", priceCurrency: "USD", availability: "https://schema.org/InStock" },
      featureList: ["<Feature one>", "<Feature two>", "<Feature three>"],
    },
  ],
};
```

```tsx
// app/page.tsx
import { JsonLd } from "@/components/json-ld";
import { siteJsonLd } from "@/lib/site";

export default function Home() {
  return (
    <>
      <JsonLd data={siteJsonLd} />
      {/* page */}
    </>
  );
}
```

Use a real, public support address and only `sameAs` profiles you control. Add `address` only when you have a real one to publish.

### FAQPage

Build it from the same array that renders the visible FAQ:

```ts
const FAQ: [string, string][] = [
  ["Is <Brand> free?", "Yes. The free plan includes <real limits>."],
  ["Can I export to PowerPoint?", "Yes. Every plan downloads an editable .pptx."],
];

const faqLd = {
  "@context": "https://schema.org",
  "@type": "FAQPage",
  mainEntity: FAQ.map(([q, a]) => ({ "@type": "Question", name: q, acceptedAnswer: { "@type": "Answer", text: a } })),
};
```

### BreadcrumbList

```ts
const breadcrumbLd = {
  "@context": "https://schema.org",
  "@type": "BreadcrumbList",
  itemListElement: [
    { "@type": "ListItem", position: 1, name: "Home", item: `${SITE_URL}/` },
    { "@type": "ListItem", position: 2, name: "Use cases", item: `${SITE_URL}/use-cases` },
    { "@type": "ListItem", position: 3, name: "Investor update", item: `${SITE_URL}/for/investor-update` },
  ],
};
```

### ItemList (hub pages)

```ts
const hubLd = {
  "@context": "https://schema.org",
  "@type": "ItemList",
  name: `${SITE_NAME} use cases`,
  itemListElement: USE_CASES.map((u, i) => ({
    "@type": "ListItem",
    position: i + 1,
    name: u.label,
    url: `${SITE_URL}/for/${u.slug}`,
  })),
};
```

### Free tool: WebApplication plus FAQPage

```ts
const toolLd = {
  "@context": "https://schema.org",
  "@graph": [
    {
      "@type": "WebApplication",
      name: "Word counter",
      url: `${SITE_URL}/tools/word-counter`,
      applicationCategory: "BusinessApplication",
      operatingSystem: "Web",
      isAccessibleForFree: true,
      offers: { "@type": "Offer", price: "0", priceCurrency: "USD" },
      publisher: { "@id": `${SITE_URL}/#organization` },
    },
    {
      "@type": "FAQPage",
      mainEntity: faq.map(([q, a]) => ({ "@type": "Question", name: q, acceptedAnswer: { "@type": "Answer", text: a } })),
    },
  ],
};
```

### Blog post: Article, BreadcrumbList, FAQPage

```ts
const url = `${SITE_URL}/blog/${post.slug}`;
const postLd = {
  "@context": "https://schema.org",
  "@graph": [
    {
      "@type": "Article",
      headline: post.title,
      description: post.description,
      datePublished: post.date,
      dateModified: post.updated,
      author: { "@type": "Person", name: "<Founder Name>", url: `${SITE_URL}/about/<founder-slug>` },
      publisher: { "@id": `${SITE_URL}/#organization` },
      mainEntityOfPage: url,
      image: `${SITE_URL}/og.png`,
      inLanguage: "en-US",
      citation: post.sources.map((s) => s.url),
    },
    {
      "@type": "BreadcrumbList",
      itemListElement: [
        { "@type": "ListItem", position: 1, name: "Home", item: `${SITE_URL}/` },
        { "@type": "ListItem", position: 2, name: "Blog", item: `${SITE_URL}/blog` },
        { "@type": "ListItem", position: 3, name: post.title, item: url },
      ],
    },
    ...(faq.length
      ? [{ "@type": "FAQPage", mainEntity: faq.map(([q, a]) => ({ "@type": "Question", name: q, acceptedAnswer: { "@type": "Answer", text: a } })) }]
      : []),
  ],
};
```

`citation` is optional. It lists the external sources the post cites, which is a useful trust signal for a post written with research.

### Author page: ProfilePage with Person

```ts
const PAGE_URL = `${SITE_URL}/about/<founder-slug>`;
const authorLd = {
  "@context": "https://schema.org",
  "@type": "ProfilePage",
  url: PAGE_URL,
  mainEntity: {
    "@type": "Person",
    "@id": `${PAGE_URL}#person`,
    name: "<Founder Name>",
    jobTitle: "Founder, <Brand>",
    url: PAGE_URL,
    worksFor: { "@id": `${SITE_URL}/#organization` },
    sameAs: [
      "https://www.linkedin.com/in/<handle>",
      "https://github.com/<handle>",
      "https://x.com/<handle>",
    ],
  },
};
```

Every blog post's `author.url` points to this page, and the page links back to the real profiles in `sameAs`. That chain is how a post shows a real, findable author.

### Check it

1. Paste a URL into Google's Rich Results Test (`search.google.com/test/rich-results`). It shows which types Google parsed and any errors.
2. Paste the same URL into the Schema Markup Validator (`validator.schema.org`), which checks types Google doesn't show as rich results.
3. After a few weeks, Search Console's **Enhancements** section lists Breadcrumbs, Videos and other parsed types with any errors.

---

## 1.8 Performance

Core Web Vitals are a small ranking factor, but slow pages also lose visitors. Three changes did most of the work.

### Defer heavy below-the-fold components

Mount expensive visual components (live product renders, charts, canvases) only when they come within a screen of the viewport, in a box that already has its final size:

```tsx
// components/defer.tsx
"use client";
import { useEffect, useRef, useState, type HTMLAttributes } from "react";

export function Defer({ children, ...props }: HTMLAttributes<HTMLDivElement>) {
  const ref = useRef<HTMLDivElement>(null);
  const [on, setOn] = useState(false);
  useEffect(() => {
    const el = ref.current;
    if (!el) return;
    const io = new IntersectionObserver(
      ([e]) => {
        if (!e.isIntersecting) return;
        setOn(true);
        io.disconnect();
      },
      { rootMargin: "100% 0px" },
    );
    io.observe(el);
    return () => io.disconnect();
  }, []);
  return <div ref={ref} {...props}>{on && children}</div>;
}
```

```tsx
<Defer className="relative aspect-video">
  <ProductRender data={sample} />
</Defer>
```

- `rootMargin: "100% 0px"` starts mounting one viewport height early, so the user rarely sees it pop in.
- The `aspect-video` class (or an explicit width and height) reserves the space. Without it, the page jumps when the child mounts, which hurts Cumulative Layout Shift.
- Deferred children are not in the server HTML. Never defer text you want indexed. Defer visuals; keep headings, copy and FAQs in the normal render.

### Responsive images with srcset

Serve each large image at a few widths and let the browser pick:

```bash
# One source image, three widths, WebP. cwebp ships with libwebp (brew install webp, apt install webp).
cwebp -q 80 -resize 720 0  hero.png -o public/landing/hero-720.webp
cwebp -q 80 -resize 1080 0 hero.png -o public/landing/hero-1080.webp
cwebp -q 80                hero.png -o public/landing/hero.webp
```

```tsx
<img
  src="/landing/hero.webp"
  srcSet="/landing/hero-720.webp 720w, /landing/hero-1080.webp 1080w, /landing/hero.webp 1376w"
  sizes="(min-width: 768px) 700px, calc(100vw - 2rem)"
  width={1376}
  height={774}
  alt="<What the image shows>"
  loading="lazy"
  decoding="async"
/>
```

- `sizes` must describe the real rendered width, or the browser downloads too large a file.
- The image that is your Largest Contentful Paint (usually the hero) gets `loading="eager"` and `fetchPriority="high"`. Everything else is `loading="lazy"`.
- Always set `width` and `height` (or an aspect-ratio box) so the layout doesn't shift.
- Make a naming rule (`<key>.webp`, `<key>-720.webp`, `<key>-1080.webp`) so a template can build the `srcSet` string from a key.

`next/image` does the same automatically on hosts that support its optimizer. On Workers or static exports, plain `<img>` with your own widths is simpler.

### Load GA4 late, and only in production

`gtag.js` is large and costs Lighthouse points if it loads with the page. Load it after the `load` event and an idle callback, and only on your production host:

```tsx
// components/google-analytics.tsx
"use client";
import { useEffect } from "react";

declare global {
  interface Window { dataLayer?: unknown[]; gtag?: (...args: unknown[]) => void }
}

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
    const idle = () =>
      "requestIdleCallback" in window ? requestIdleCallback(start, { timeout: 5000 }) : setTimeout(start, 3000);
    if (document.readyState === "complete") idle();
    else addEventListener("load", idle, { once: true });
  }, [id, host]);
  return null;
}
```

```tsx
// in your root layout or providers
<GoogleAnalytics id={process.env.NEXT_PUBLIC_GA_ID!} host="example.com" />
```

- The host check keeps localhost, preview deployments and other domains out of your data.
- Pushing `arguments` (not an array) is required; gtag.js expects the arguments object.
- The trade-off: visitors who leave within a second or two are not counted. For traffic-source reporting that is acceptable.
- If you have a Content Security Policy, allow `https://www.googletagmanager.com` in `script-src`, `connect-src` and `img-src`, and `https://*.google-analytics.com https://*.analytics.google.com` in `connect-src` and `img-src`.

### Measure

- PageSpeed Insights (`pagespeed.web.dev`) for lab and field data per URL.
- Search Console **Core Web Vitals** report for field data across the site, once you have enough traffic.
- Chrome DevTools **Lighthouse** in an incognito window with the mobile preset.

---

## 1.9 Mobile width checks

Google indexes the mobile version of every page. Any element wider than the screen causes horizontal scrolling, which looks broken and hurts usability.

Check every public page at **360, 390 and 414 px** wide. Those cover most Android phones and iPhones.

### By hand

1. Chrome DevTools, **Toggle device toolbar**, choose **Responsive**, set the width to 360.
2. Scroll the whole page and try to swipe sideways.
3. If it scrolls, paste this in the console to find the culprit:

```js
[...document.querySelectorAll("body *")].filter((e) => e.getBoundingClientRect().right > innerWidth + 1)
```

Common causes: long unbroken strings (URLs, code), tables without an `overflow-x: auto` wrapper, fixed widths in px, negative margins, and wide decorative elements without `overflow: hidden` on their parent.

### Scripted

```js
// scripts/check-width.mjs
// npm i -D playwright && npx playwright install chromium
// node scripts/check-width.mjs https://example.com https://example.com/pricing
import { chromium } from "playwright";

const urls = process.argv.slice(2);
const browser = await chromium.launch();
let bad = 0;
for (const width of [360, 390, 414]) {
  const page = await browser.newPage({ viewport: { width, height: 800 } });
  for (const url of urls) {
    await page.goto(url, { waitUntil: "networkidle" });
    const over = await page.evaluate(() => document.documentElement.scrollWidth - document.documentElement.clientWidth);
    if (over > 0) {
      bad++;
      console.log(`${width}px ${url}: ${over}px too wide`);
    }
  }
  await page.close();
}
await browser.close();
process.exit(bad ? 1 : 0);
```

Feed it the URLs from your sitemap to check every page at once:

```bash
node scripts/check-width.mjs $(curl -s https://example.com/sitemap.xml | grep -o "<loc>[^<]*" | sed "s/<loc>//")
```

---

## 1.10 Text-to-markup ratio

Modern frameworks ship a lot of markup: inline SVG, class-heavy HTML and serialized component payloads. When the visible text is a tiny share of the HTML, some audit tools flag the page as thin, and agents reading the raw HTML have to dig for the content. Keep the homepage at **5% or more** text to markup.

### Measure

```js
// scripts/text-ratio.mjs
// node scripts/text-ratio.mjs https://example.com
const url = process.argv[2];
const html = await (await fetch(url)).text();
const text = html
  .replace(/<script[\s\S]*?<\/script>/gi, "")
  .replace(/<style[\s\S]*?<\/style>/gi, "")
  .replace(/<svg[\s\S]*?<\/svg>/gi, "")
  .replace(/<[^>]+>/g, " ")
  .replace(/\s+/g, " ")
  .trim();
console.log(`${url}: ${text.length} text / ${html.length} html = ${((text.length / html.length) * 100).toFixed(1)}%`);
```

### Raise it

- Mount decorative layers (animated backgrounds, floating shapes, large inline SVG scenes) after hydration, in a client component's `useEffect`, so they are not in the server HTML.
- Move large inline SVGs to files referenced by `<img>`.
- End the homepage with a real FAQ section: five to eight questions people ask, with plain answers. It adds useful text and doubles as FAQPage JSON-LD.
- Don't pad with hidden text. It has to be visible and useful.

> Real-world example: an animated dot-grid backdrop was rendered on the server. Mounting its pieces after hydration and adding a Questions section lifted the homepage above 5%.

---

## Checklist

- [ ] One canonical host chosen; `SITE_URL` constant used everywhere.
- [ ] `www`, `http` and every extra domain 301 to the same path on the canonical host, in one hop, query string kept.
- [ ] `robots.txt` blocks only API and analytics proxy paths, allows public media under the API, and lists the sitemap.
- [ ] No app page is blocked in `robots.txt`.
- [ ] Every app path sends `X-Robots-Tag: noindex, nofollow`, for both `/x` and `/x/...`.
- [ ] New protected paths go into the auth gate, the noindex list and the route list together.
- [ ] Sitemap built from registries, plus dynamic posts with `revalidate`; only 200, indexable, canonical URLs.
- [ ] A test checks every sitemap URL has a route.
- [ ] Root layout sets `metadataBase`, a title template, a default description, canonical, Open Graph and Twitter card.
- [ ] Every programmatic page is a static route with a static `metadata` export.
- [ ] Titles 70 characters or fewer with the brand suffix; descriptions 70 to 160; a test enforces it.
- [ ] Dynamic routes checked with curl for metadata in `<head>`; HTMLRewriter (or static rendering) used where it isn't.
- [ ] JSON-LD rendered through one escaping helper, with the types from the table, matching visible content.
- [ ] Rich Results Test passes on one page of each type.
- [ ] Heavy below-the-fold visuals deferred inside fixed-size boxes; indexable text never deferred.
- [ ] Large images have 720 and 1080 width variants, `sizes`, dimensions and lazy loading; the LCP image is eager.
- [ ] GA4 (or any heavy tag) loads after `load` plus idle, only on the production host.
- [ ] Every public page passes at 360, 390 and 414 px with no horizontal scroll.
- [ ] Homepage text-to-markup ratio is 5% or more.

## Common mistakes

| Mistake | What happens | Fix |
|---|---|---|
| Both `www` and the apex return 200 | Signals split across two copies of every page | 301 one to the other |
| Redirecting every old URL to the homepage | Deep-link value lost; Google may treat it as a soft 404 | Keep the path in the redirect |
| Extra domain has no proxied DNS record | The Cloudflare redirect rule never runs; the domain doesn't resolve | Add a proxied `AAAA 100::` record or attach it to the Worker |
| `Disallow: /dashboard` in robots.txt | Bare URLs indexed from links; noindex never seen | Allow crawling, send `X-Robots-Tag` |
| Header rule for `/x/:path*` only | `/x` itself stays indexable | List both `/x` and `/x/:path*` |
| Sitemap lists redirects, noindexed or 404 URLs | Search Console warnings, wasted crawl | Build it from registries, test it |
| `lastmod` set to "now" on every request | Google learns to ignore your lastmod | Use real change dates |
| Hard-coded domain in a component | Wrong canonicals after a domain change | Import `SITE_URL` |
| Title over 70 characters once the brand suffix is added | Cut off in results | Count the suffix; test it |
| `generateMetadata` streamed into `<body>` unnoticed | Crawlers may miss title and canonical | Curl with a bot user agent; inject with HTMLRewriter |
| Removing the streamed metadata after adding injection | Hydration errors | Keep both copies |
| JSON-LD built with plain `JSON.stringify` | A `</script>` in content breaks the page | Escape `<` as `<` |
| FAQPage questions not shown on the page | Structured data against guidelines | Build both from one array |
| Deferring text content | It is not in the server HTML, so it isn't indexed | Defer visuals only |
| Deferred box without a size | Layout shift when it mounts | Give it `aspect-*` or width and height |
| GA4 loaded in `<head>` on every host | Lower Lighthouse scores; dev and preview traffic in reports | Load late, check the hostname |
