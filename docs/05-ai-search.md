# 5. AI search

Make your product easy for ChatGPT, Perplexity, Gemini, Claude and Copilot to read, describe correctly and recommend.

## What you'll set up

- A "How did you hear about us?" answer for AI assistants, so you can see this channel at all.
- `robots.txt` and CDN settings that let AI search crawlers read your public pages.
- `/llms.txt`: a hand-written product summary plus a generated list of posts.
- Markdown for agents: the same URLs answer with Markdown when a client asks for it, with a Markdown 404 for pages that don't exist.
- `/about` and `/contact` trust pages, served as HTML and Markdown from one source.
- Organization JSON-LD with `email`, `contactPoint`, `founder` and `sameAs`.
- A buyer prompt list of about 50 questions, each mapped to a target page.
- One facts file that every description of your product is copied from.
- A monthly routine to check whether assistants name or cite you.

## Why AI assistants are a channel

People now ask an assistant before they search. "What's the best free tool to do X without a watermark?" gets a short list of named products, often with links. If you are not on that list, you don't exist for that buyer.

A new domain also ranks slowly in Google. Assistants with web search read listicles, directories, review sites, Reddit threads and your own pages, and some of those sources can mention you within days.

> **Real-world example:** in its first 10 days, 83 of about 300 sign-ups answered "ChatGPT or an AI assistant" to "How did you hear about us?", against 10 who answered "Google search". The site was days old, so Google barely ranked it, while assistants were already recommending it.

Two things follow:

1. **Your own pages must be easy to read and quote.** That is what this chapter covers.
2. **Third-party pages matter more than your own.** When the product ran 50 buyer prompts through Gemini with search, the sources it cited were mostly vendor pages and "best X tools" listicles. Getting into those lists is the job of [chapter 6](06-off-page.md) (Off-page).

### Step 1: Measure the channel before you work on it

Referrers from assistants are unreliable. Many clicks arrive with no referrer, and many people read the answer and type your name into the address bar later. Ask instead.

1. Add one question to sign-up or onboarding: "How did you hear about us?"
2. Offer fixed options, including a separate one for assistants. These options worked well in production:

   ```ts
   export const HEARD_FROM = {
     google: "Google search",
     ai: "ChatGPT or an AI assistant",
     reddit: "Reddit",
     linkedin: "LinkedIn",
     x: "X (Twitter)",
     friend: "Friend or colleague",
     launch: "Product Hunt or a launch site",
     other: "Other",
   } as const;
   ```

3. Store the answer on the user row, and send only the category (never free text or an email) to analytics as a person property.
4. Show the breakdown in your admin page or a saved analytics insight. You will read it every week ([chapter 7](07-measurement.md)).

Keep the options short and stable. If you rename one, keep the old key's label so past answers still show.

## Step 2: Let AI crawlers in

Assistants can only cite what their crawlers can fetch.

1. Open `https://example.com/robots.txt`. It should block only private paths, for example:

   ```txt
   User-agent: *
   Allow: /api/media/
   Disallow: /api/
   Disallow: /ingest/

   Sitemap: https://example.com/sitemap.xml
   ```

2. Don't add `Disallow` rules for AI user agents unless you mean to opt out. The search and fetch agents to keep allowed include `OAI-SearchBot` and `ChatGPT-User` (OpenAI), `PerplexityBot`, `Claude-SearchBot` and `Claude-User` (Anthropic) and `Bingbot` (which also feeds Copilot). Check each vendor's crawler page for the current names before you write rules.
3. Check your CDN. Some CDNs and hosting plans offer a switch that blocks AI crawlers or rewrites `robots.txt` for you. On Cloudflare, look under the bot and AI crawler settings for the zone. Make sure what it does matches what you want.
4. Make sure your public pages render their main text in the HTML response. A page that is only a JavaScript shell is empty to most crawlers.

## Step 3: llms.txt

`/llms.txt` is a Markdown file at the root of your site that summarises the product and links its key pages. It follows the proposal at llmstxt.org: an H1 with the name, a blockquote summary, optional short paragraphs or bullets, then H2 sections of links.

Set expectations correctly. Google has said it doesn't use `llms.txt` for Search. Some agents, coding tools and AI crawlers do read it, and it costs an hour to write. Treat it as a cheap, accurate summary that agents can find, not a ranking factor.

### What to put in it

- **The first 3 lines carry the product.** Name, one-sentence summary, the free plan and paid plan in numbers.
- **Facts copied from your facts file** (step 7). Same prices, same limits, same wording.
- **Every public page that answers a question**, one line each: `- [Title](absolute URL): what the page answers`.
- **Absolute URLs** with your canonical host.
- **No marketing adjectives.** Assistants quote you. "Free for 30 projects a month" is quotable; "the best tool ever" is not.
- **Honest limits.** If a feature is paid only, say so. Assistants that repeat a wrong claim cost you trust.

### Template

Save this as `content/llms.txt` (or as a string constant, see below) and replace every placeholder.

```md
# <Brand>

> <Brand> is a <category> for <audience>. You <input or starting point>, and it <outcome>. You can <key action> and <export or result>.

- Free plan: <limits in numbers>, <what is included>, <what is not, such as watermark or card>. Pro is $<price>/month or $<price>/year for <the main paid features, in numbers>.
- <Two or three product facts people ask about: formats, integrations, platforms, languages.>
- Sign up with <methods>.

## Pages

- [Home](https://example.com/): what <Brand> does
- [Sign up](https://example.com/signup): create an account and start
- [Pricing](https://example.com/pricing): Free (<limits>) and Pro ($<price>/month, <main features>)
- [About](https://example.com/about): what <Brand> is, why it exists and who builds it
- [<Founder name>](https://example.com/about/<founder-slug>): the founder and author of the blog
- [Contact](https://example.com/contact): email <public support address> for support, billing, privacy or press
- [Privacy](https://example.com/privacy): what <Brand> stores and where uploads go
- [All use cases](https://example.com/use-cases): every use case and comparison page, grouped
- [Templates](https://example.com/templates): free examples to copy and edit
- [Blog](https://example.com/blog): researched guides, with sources

## Use cases

- [<Use case 1>](https://example.com/for/<slug-1>): <input> to <output>, <the one detail that matters>
- [<Use case 2>](https://example.com/for/<slug-2>): <input> to <output>, <the one detail that matters>

## Comparisons

- [<Brand> vs <Rival A>](https://example.com/vs/<rival-a>): a <Rival A> alternative with <your real difference>, compared on price, <feature> and <feature>
- [<Rival A> alternatives](https://example.com/alternatives/<rival-a>): <8 words or fewer>

## Free tools

No account needed. <Say which tools ask for an email and which run in the browser.>

- [All free tools](https://example.com/tools): every free tool on one page
- [<Tool 1>](https://example.com/tools/<tool-1>): <what it calculates or generates>

## Videos

- [<How to do X>](https://example.com/watch/<slug>): <length> demo with transcript
```

Leave out sections you don't have yet. Add the Blog section in code, not by hand (next step).

### Serve it with a generated section

The hand-written part changes when your product changes. The blog list changes every day. Keep the first in a file and append the second at request time.

`lib/llms.ts`:

```ts
import { BASE } from "@/content/llms";
import { listPosts } from "@/lib/blog";
import { SITE_URL } from "@/lib/site";

// The hand-written overview plus the blog, which changes daily.
export async function llmsTxt() {
  const posts = await listPosts();
  if (!posts.length) return BASE;
  return `${BASE.trimEnd()}\n\n## Blog\n\n${posts.map((p) => `- [${p.title}](${SITE_URL}/blog/${p.slug}): ${p.description}`).join("\n")}\n`;
}
```

`content/llms.ts` holds the template above as a string, which works in every runtime (Node, edge, Workers):

```ts
export const BASE = `# <Brand>

> <Brand> is a ...
`;
```

If you build with Vite, you can keep a real `content/llms.txt` file and import it raw instead. This is the setup used in production:

```ts
/// <reference types="vite/client" />
const BASE = import.meta.glob<string>("../content/llms.txt", { eager: true, query: "?raw", import: "default" })["../content/llms.txt"];
```

`listPosts()` is your own function. It returns `{ slug, title, description }[]`, newest first, from your CMS or database. If posts live in Markdown files in the repo, read them at build time instead.

`app/llms.txt/route.ts` (Next.js App Router):

```ts
import { llmsTxt } from "@/lib/llms";

export async function GET() {
  return new Response(await llmsTxt(), { headers: { "content-type": "text/plain; charset=utf-8", "cache-control": "public, max-age=300" } });
}
```

A 5-minute cache keeps it cheap and still shows a new post within minutes.

Check it:

```sh
curl -sI https://example.com/llms.txt | grep -i content-type
curl -s https://example.com/llms.txt | head -20
```

Link it from the Markdown 404 (next step) and from the trust pages' Markdown footer, so agents that land anywhere can find it.

## Step 4: Markdown for agents

Agents that fetch a page get thousands of lines of HTML, scripts and styles. When a client says it prefers Markdown, give it Markdown for the same URL. Browsers never ask for Markdown, so they keep getting HTML.

The rules:

- Answer with Markdown only when the `Accept` header names `text/markdown` and rates it at least as high as `text/html`, using the q-values from RFC 9110.
- Serve Markdown for pages that have a real Markdown source: `/` (your `llms.txt`), the trust pages, and blog posts.
- Every Markdown response carries `Vary: Accept` (so caches keep the two versions apart) and `X-Robots-Tag: noindex` (so the Markdown copy never competes with the HTML page in search).
- HTML responses for those same paths also carry `Vary: Accept`.
- A path that isn't a real page gets a 404 with a short Markdown body that links home, `llms.txt` and the sitemap. Agents guess URLs; this gives them a way back.

### lib/agent-markdown.ts

```ts
import { SITE_URL } from "@/lib/site";

// q of the most specific range that matches the type (exact, then type/*, then */*), per RFC 9110.
const quality = (accept: string, type: string) => {
  const group = `${type.split("/")[0]}/*`;
  let best = { rank: 0, q: 0 };
  for (const part of accept.split(",")) {
    const [range, ...params] = part.split(";").map((x) => x.trim().toLowerCase());
    const rank = range === type ? 3 : range === group ? 2 : range === "*/*" ? 1 : 0;
    if (rank <= best.rank) continue;
    const q = Number(params.find((x) => x.startsWith("q="))?.slice(2) ?? 1);
    best = { rank, q: Number.isFinite(q) ? Math.min(Math.max(q, 0), 1) : 1 };
  }
  return best.q;
};

// Markdown only when the client names text/markdown and rates it at least as high as HTML, so browsers keep getting HTML.
export function wantsMarkdown(accept: string | null) {
  if (!accept || !/text\/markdown/i.test(accept)) return false;
  const md = quality(accept, "text/markdown");
  return md > 0 && md >= quality(accept, "text/html");
}

// Pages the app serves outside the sitemap: auth, the signed-in app, share links. Edit to match your routes.
const APP_ROUTES = /^\/(login|signup|dashboard|settings)$|^\/(app|s)\/.+/;

export function isKnownPath(path: string, publicPaths: ReadonlySet<string>) {
  const p = path.length > 1 ? path.replace(/\/+$/, "") : path;
  return publicPaths.has(p) || APP_ROUTES.test(p);
}

export function notFoundMarkdown(path: string) {
  const shown = path.length > 200 ? `${path.slice(0, 200)}...` : path;
  return [
    "# Page not found",
    "",
    `There is no page at \`${shown.replace(/`/g, "")}\` on <Brand>, a <one-line description>.`,
    "",
    `- [Home](${SITE_URL}/)`,
    `- [Site overview for agents (llms.txt)](${SITE_URL}/llms.txt)`,
    `- [Sitemap](${SITE_URL}/sitemap.xml)`,
    "",
  ].join("\n");
}

export const markdownHeaders = { "content-type": "text/markdown; charset=utf-8", vary: "Accept", "x-robots-tag": "noindex" };
```

Notes on `quality`:

- It takes the most specific matching range: `text/markdown` beats `text/*`, which beats `*/*`. This is the RFC 9110 rule.
- A missing `q` means 1. A broken `q` also counts as 1, and values are clamped to 0..1.
- A typical browser sends `text/html,application/xhtml+xml,...,*/*;q=0.8`. It never names `text/markdown`, so the first check returns false and the browser gets HTML.

### The list of public paths

`isKnownPath` needs every public path. Take it from the same function that builds your sitemap, so the two never disagree. In Next.js, split the static part out of `app/sitemap.ts`:

```ts
import type { MetadataRoute } from "next";
import { listPosts } from "@/lib/blog";
import { SITE_URL } from "@/lib/site";
import { USE_CASES } from "@/lib/use-cases";
import { TOOLS } from "@/lib/tools";

const lastModified = new Date();

export const staticPages = (): MetadataRoute.Sitemap =>
  ["", "/pricing", "/about", "/contact", "/privacy", "/use-cases", "/tools", "/blog", ...TOOLS.map((t) => `/tools/${t.slug}`), ...USE_CASES.map((u) => `/for/${u.slug}`)].map((path) => ({ url: `${SITE_URL}${path}`, lastModified }));

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const posts = await listPosts();
  return [...staticPages(), ...posts.map((p) => ({ url: `${SITE_URL}/blog/${p.slug}`, lastModified: p.updated }))];
}
```

Blog posts are not in `staticPages()`. The middleware checks them separately by looking the post up.

### The middleware

In Next.js 16 the file is `proxy.ts` and exports `proxy`. In Next.js 15 it is `middleware.ts` and exports `middleware`. The body is the same.

```ts
import { NextResponse, type NextRequest } from "next/server";
import { staticPages } from "@/app/sitemap";
import { isKnownPath, markdownHeaders, notFoundMarkdown, wantsMarkdown } from "@/lib/agent-markdown";
import { getPost } from "@/lib/blog";
import { llmsTxt } from "@/lib/llms";
import { SITE_URL } from "@/lib/site";
import { ABOUT, CONTACT, toMarkdown } from "@/lib/trust";

const PUBLIC_PATHS = new Set(staticPages().map((p) => new URL(p.url).pathname));
const MARKDOWN = new Map<string, string>([
  [ABOUT.path, toMarkdown(ABOUT, SITE_URL)],
  [CONTACT.path, toMarkdown(CONTACT, SITE_URL)],
]);
const HAS_MARKDOWN = (path: string) => path === "/" || MARKDOWN.has(path) || /^\/blog\/[a-z0-9-]+$/.test(path);

// Agents that ask for text/markdown get Markdown for pages that have it, and a Markdown 404 for paths that don't exist.
async function markdown(pathname: string) {
  const path = pathname.length > 1 ? pathname.replace(/\/+$/, "") : pathname;
  const slug = /^\/blog\/([a-z0-9-]+)$/.exec(path)?.[1];
  const post = slug ? await getPost(slug).catch((e) => void console.error("blog markdown failed", e)) : undefined;
  if (path === "/") return new NextResponse(await llmsTxt(), { status: 200, headers: markdownHeaders });
  if (slug && post) return new NextResponse(`# ${post.title}\n\n${post.description}\n\n${post.body.trim()}\n`, { status: 200, headers: markdownHeaders });
  const body = MARKDOWN.get(path);
  if (body) return new NextResponse(body, { status: 200, headers: markdownHeaders });
  if (!isKnownPath(path, PUBLIC_PATHS)) return new NextResponse(notFoundMarkdown(path), { status: 404, headers: markdownHeaders });
  return null;
}

export async function proxy(req: NextRequest) {
  const { pathname } = req.nextUrl;
  if (wantsMarkdown(req.headers.get("accept"))) {
    const md = await markdown(pathname);
    if (md) return md;
  }
  const res = NextResponse.next();
  if (HAS_MARKDOWN(pathname)) res.headers.set("vary", "Accept");
  return res;
}

export const config = { matcher: ["/((?!_next/|api/|.*\\.).*)"] };
```

What each part does:

- **The matcher** runs on page paths only. It skips `/_next/`, `/api/` and any path with a dot, so `llms.txt`, `sitemap.xml`, images and scripts are untouched. Add any other prefix you proxy (for example an analytics proxy).
- **`getPost(slug)`** is your own function. It returns `{ title, description, body }` with `body` as Markdown, or null. A failed lookup is logged and falls through to the 404 check rather than crashing the page.
- **A known page without a Markdown source** returns `null`, so it falls through to the normal HTML response. Agents still get the page, just as HTML.
- **If you already have auth logic in middleware**, put the Markdown check first, as the production setup does, and keep your auth checks after it. Never serve Markdown for signed-in pages; they aren't in `PUBLIC_PATHS` or `MARKDOWN`, so the code above can't.

### Check it with curl

```sh
# Markdown for the home page, with the right headers
curl -s -D - -o /dev/null -H "Accept: text/markdown" https://example.com/
# expect: 200, content-type: text/markdown; charset=utf-8, vary: Accept, x-robots-tag: noindex

# A client that prefers HTML still gets HTML
curl -s -D - -o /dev/null -H "Accept: text/html, text/markdown;q=0.5" https://example.com/

# A browser-style header gets HTML, with Vary: Accept
curl -s -D - -o /dev/null -H "Accept: text/html,application/xhtml+xml,*/*;q=0.8" https://example.com/about

# Unknown path: Markdown 404
curl -s -i -H "Accept: text/markdown" https://example.com/this-page-does-not-exist
```

### One small test

Leave one check behind for the q-value logic, since it is easy to break when someone edits it. `lib/agent-markdown.test.ts`:

```ts
import { test } from "node:test";
import assert from "node:assert/strict";
import { wantsMarkdown } from "./agent-markdown";

test("wantsMarkdown follows RFC 9110 q-values", () => {
  assert.equal(wantsMarkdown(null), false);
  assert.equal(wantsMarkdown("text/html,application/xhtml+xml,*/*;q=0.8"), false);
  assert.equal(wantsMarkdown("text/markdown"), true);
  assert.equal(wantsMarkdown("text/markdown, text/html"), true);
  assert.equal(wantsMarkdown("text/html, text/markdown;q=0.5"), false);
  assert.equal(wantsMarkdown("text/markdown;q=0.9, text/*;q=0.5"), true);
  assert.equal(wantsMarkdown("text/markdown;q=0"), false);
});
```

Run it with `npx tsx --test lib/agent-markdown.test.ts`, or move the asserts into your existing test runner.

### Caching

`Vary: Accept` tells caches to store the HTML and Markdown answers separately. Some CDNs ignore `Vary` for HTML. If your CDN caches HTML pages at the edge, either make sure the middleware runs before the cache, or bypass the cache when the request carries `text/markdown`. Otherwise a browser could be served the Markdown copy.

## Step 5: Trust pages and Organization JSON-LD

Assistants and search engines look for signs that a real, reachable business stands behind a product. Two short pages and one block of JSON-LD cover most of it.

### One source for the HTML page and the Markdown

`lib/trust.ts`:

```ts
export const PUBLIC_EMAIL = "hello@example.com";

export type TrustPage = { path: string; title: string; description: string; sections: [string, string][] };

export const ABOUT: TrustPage = {
  path: "/about",
  title: "About <Brand>",
  description: "Who builds <Brand>, a <category>, and how it works.",
  sections: [
    ["What <Brand> is", "<Two or three sentences: what goes in, what comes out, what people do with it.>"],
    ["Why it exists", "<The problem with existing tools, in plain words, and what you keep free or simple. Use real prices and limits from the facts file.>"],
    ["Who builds it", "<Brand> was founded in <month year> by <founder name>, <one line of relevant background>. Support email goes straight to <founder name>."],
    ["How your data is handled", "<What you store, what you don't, where uploads go, and a pointer to the privacy page.>"],
  ],
};

export const CONTACT: TrustPage = {
  path: "/contact",
  title: "Contact <Brand>",
  description: "How to reach the <Brand> team for support, billing, privacy or press.",
  sections: [
    ["Email", `Write to ${PUBLIC_EMAIL} for anything: help, a bug, a billing question, a privacy request or a partnership.`],
    ["Feedback from inside the app", "<Where the feedback button is.>"],
    ["Billing", "<Who processes payments and how to change or cancel a plan.>"],
    ["Privacy and your data", "<How to ask for data to be restored, exported or erased.>"],
  ],
};

export const toMarkdown = (page: TrustPage, site: string) =>
  [`# ${page.title}`, "", page.description, "", ...page.sections.flatMap(([h, body]) => [`## ${h}`, "", body, ""]), `[<Brand>](${site}/) · [Privacy](${site}/privacy) · [llms.txt](${site}/llms.txt)`, ""].join("\n");
```

The HTML page renders the same object:

```tsx
import type { Metadata } from "next";
import { ABOUT } from "@/lib/trust";
import { SITE_URL } from "@/lib/site";

export const metadata: Metadata = { title: ABOUT.title, description: ABOUT.description, alternates: { canonical: `${SITE_URL}${ABOUT.path}` } };

export default function AboutPage() {
  return (
    <main>
      <h1>{ABOUT.title}</h1>
      <p>{ABOUT.description}</p>
      {ABOUT.sections.map(([heading, body]) => (
        <section key={heading}>
          <h2>{heading}</h2>
          <p>{body}</p>
        </section>
      ))}
    </main>
  );
}
```

Rules for these pages:

1. At least 500 characters of real text each. A two-line "About us" reads as a thin page.
2. Name a real person and a real, monitored email address. Use a role address (for example `hello@`) rather than a personal one.
3. Put both pages in the sitemap, the footer and `llms.txt`.
4. Add a founder page (`/about/<founder-slug>`) with Person JSON-LD and `sameAs` links to real profiles. Blog posts point their author to it.

### Organization JSON-LD

Render this once, in the root layout or the home page. `lib/site.ts`:

```ts
import { PUBLIC_EMAIL } from "./trust";

export const SITE_URL = "https://example.com";
export const SITE_DESCRIPTION = "<One sentence, 70 to 160 characters, matching your home page description.>";

const org = `${SITE_URL}/#organization`;

export const siteJsonLd = {
  "@context": "https://schema.org",
  "@graph": [
    {
      "@type": "Organization",
      "@id": org,
      name: "<Brand>",
      url: SITE_URL,
      logo: `${SITE_URL}/icon-512.png`,
      email: PUBLIC_EMAIL,
      foundingDate: "<YYYY-MM>",
      founder: { "@type": "Person", name: "<Founder name>", url: "<founder's own site>", sameAs: ["<founder LinkedIn URL>", "<founder GitHub URL>"] },
      contactPoint: { "@type": "ContactPoint", contactType: "customer support", email: PUBLIC_EMAIL, url: `${SITE_URL}/contact`, availableLanguage: ["English"] },
      sameAs: ["<Crunchbase organization URL>", "<company LinkedIn URL>", "<public GitHub URL>"],
    },
    { "@type": "WebSite", "@id": `${SITE_URL}/#website`, url: SITE_URL, name: "<Brand>", description: SITE_DESCRIPTION, inLanguage: "en", publisher: { "@id": org } },
  ],
};
```

Render it with `<` escaped, so text in your data can't close the script tag:

```tsx
const ld = (o: object) => JSON.stringify(o).replace(/</g, "\\u003c");

<script type="application/ld+json" dangerouslySetInnerHTML={{ __html: ld(siteJsonLd) }} />
```

Notes:

- Add `address` only when you have a real one to publish. Don't invent one.
- `sameAs` should list profiles you control and keep current: Crunchbase, LinkedIn company page, GitHub. Each one is a place an assistant can confirm you exist.
- Test the page with Google's Rich Results Test and the Schema Markup Validator after deploy.

> **Real-world example:** an agent-readiness audit flagged missing trust signals. The fix was `/about` and `/contact` built from one `TrustPage` object (HTML and Markdown), and an Organization block with `email` and a `contactPoint` pointing at `/contact`. No `address` was added because there was no real one to publish.

## Step 6: The buyer prompt list

This list tells you which pages to build and gives you something concrete to check every month.

### Method

1. **List your personas.** Use your onboarding answers ("What do you do?") if you have them, otherwise your first 20 customer conversations. Aim for 5 to 8 personas.
2. **For each persona, write 5 to 8 prompts** the way a buyer types into ChatGPT: a full sentence, with their situation and constraints ("free", "without a watermark", "that exports to Excel", "for a team of 5").
3. **Write the matching Google query** for each prompt: 2 to 6 words. Check it in Google autocomplete and keep the phrasing people actually use.
4. **Pick the target page** that should win the answer: a use-case page, a comparison, a template, a free tool or a post. If none fits, write the home page and mark it as a gap.
5. **Add a "Compare" group** for prompts like "best free X", "A vs B" and "alternative to A". Almost every buyer asks one.
6. **Stop at about 50.** More than that and you won't check them.
7. **Turn every gap into work.** A prompt that falls back to the home page is a candidate for a new page or a blog post.

> **Real-world example:** 50 prompts across 8 personas (founders, students, teachers, consultants, marketing, sales, product, managers). 6 of the 50 had no dedicated page and fell back to the home page; each became a candidate page or post.

### Template

Keep it as a Markdown table or a sheet. The last three columns are filled in by the monthly check (step 8).

| # | Persona | AI prompt | Google query | Target page | Status | Rivals named | Sources cited |
|---|---|---|---|---|---|---|---|
| 1 | | | | | Named / Linked / Absent | | |

### Five example rows

For an imaginary invoicing app for freelancers, `<Brand>`:

| # | Persona | AI prompt | Google query | Target page |
|---|---|---|---|---|
| 1 | Freelance designer | I just finished my first client project. What's a free way to send a professional invoice that clients can pay by card? | free invoice maker | /for/freelance-invoice |
| 2 | Consultant | I bill hourly across three clients. Is there a tool that turns my time log spreadsheet into invoices automatically? | timesheet to invoice | /for/timesheet-to-invoice |
| 3 | Small agency owner | Best alternative to <Rival A> for a 5 person agency that doesn't charge per user | <rival a> alternative | /alternatives/<rival-a> |
| 4 | Freelance developer | How do I write a late payment reminder that stays polite but gets paid? | late payment email template | /tools/payment-reminder-generator |
| 5 | Anyone comparing | <Rival A> vs <Rival B> for freelancers: which is cheaper, and is there something free? | <rival a> vs <rival b> | /compare/<rival-a>-vs-<rival-b> |

## Step 7: Keep facts consistent everywhere

Assistants repeat whatever they find, including an old price on a directory you forgot about. If your site says "$15 a month" and a listing says "$12", the assistant may say either.

### One facts file

Create `facts.md` in the repo. It is the only place product facts are written by hand. Everything else copies from it.

```md
# <Brand> facts

Last checked: <YYYY-MM-DD>

## One line
<Brand> is a <category> that turns <input> into <output>.

## Plans
- Free: <limits in numbers>. <What is not included.> No card.
- Pro: $<price>/month or $<price>/year (<N>% off yearly). <Main features in numbers.>
- <Any discount: students, nonprofits, launch offer, with its rule.>

## What it does
- Inputs: <formats, integrations>
- Outputs: <exports, formats>
- <Three to five more features people ask about.>

## What it doesn't do
- <Honest limits. Assistants are asked about these too.>

## Company
- Founded <month year> by <founder name>.
- Contact: <public support address>, https://example.com/contact
- Payments: <processor>

## Short descriptions
- 35 characters: <tagline>
- 60 characters: <tagline>
- 160 characters: <description>
```

### Where the facts appear

When a price, limit or feature changes, update all of these in the same day:

1. The constants in code (prices and limits should live in one shared module that the pricing page reads).
2. `facts.md`.
3. `content/llms.txt` (or `content/llms.ts`).
4. The pricing page metadata and FAQ, and the `/about` page.
5. JSON-LD that mentions offers or prices.
6. Anything your blog writer or email templates read.
7. Every directory and review listing: AlternativeTo, SaaSHub, G2, Crunchbase, launch sites. Keep their links in your tracker ([chapter 6](06-off-page.md)) so you can find them.
8. Your launch kit (bios, taglines, descriptions you paste into forms).

> **Real-world example:** the free plan changed on 2026-10-26. The AlternativeTo, SaaSHub and Crunchbase descriptions were updated the same day, and the tracker notes which listings still needed the new text after their review.

### Guard it with a test

A test can't check directories, but it can stop the repo drifting. Compare the hand-written files against your constants:

```ts
import { test } from "node:test";
import assert from "node:assert/strict";
import { readFileSync } from "node:fs";
import { PRO_PRICES, FREE_LIMITS } from "../packages/shared/src/limits";

const files = ["content/llms.txt", "docs/facts.md"].map((f) => [f, readFileSync(f, "utf8")] as const);

test("hand-written facts match the constants", () => {
  for (const [name, text] of files) {
    assert.ok(text.includes(`$${PRO_PRICES.monthly}/month`), `${name} is missing the monthly price`);
    assert.ok(text.includes(`${FREE_LIMITS.projects}`), `${name} is missing the free limit`);
  }
});
```

Adjust the names to your own constants and the exact phrasing you use. The point is that a price change fails CI until the text is updated.

## Step 8: Track whether assistants cite you

There is no Search Console for assistants. Check by hand once a month, and keep the results so you can see change.

### Monthly routine

1. **Use clean sessions.** Signed out, or a temporary chat with memory and custom instructions off. Your own history biases the answer.
2. **Run every AI prompt from your list** in each assistant your buyers use: ChatGPT (with search), Perplexity, Gemini, Claude, Copilot. Run each Google query in Google and note the AI Overview if one shows.
3. **Record three things per prompt and assistant:**
   - **Status:** Named (your brand appears), Linked (it links to your site), or Absent.
   - **Rivals named,** in the order given.
   - **Sources cited:** the domains and pages the answer links to.
4. **Save it with the date** as a new section or a new sheet tab. Don't overwrite last month.
5. **Count:** prompts where you are Named, prompts where you are Linked, per assistant. Compare with last month.
6. **Turn results into work:**
   - A source cited for many prompts (a listicle, a directory, a Reddit thread) goes to the top of your outreach list ([chapter 6](06-off-page.md)).
   - A prompt where you are Absent and have no target page becomes a page or post.
   - A wrong fact in an answer (old price, missing feature) means a listing somewhere is out of date. Find it and fix it.
7. **Read your analytics alongside it:** sessions referred by `chatgpt.com`, `perplexity.ai`, `gemini.google.com`, `copilot.microsoft.com` and `claude.ai`, and links tagged `utm_source=chatgpt.com`, which ChatGPT often adds. And the "ChatGPT or an AI assistant" count from your sign-up question.

Some teams script part of this through model APIs with search turned on. That gives a repeatable sample, but API answers can differ from the consumer apps, so keep the manual check as the source of truth.

> **Real-world example:** a run of all 50 prompts through the Gemini API, with and without Google Search grounding, named the product in 0 of 99 answers. The tools it named most were Google Slides, Gamma, Pitch, Beautiful.ai and Canva, and the sites it cited were mostly vendor pages and "best AI presentation maker" listicles. That moved effort from on-site pages to getting into those lists. Meanwhile, sign-up answers showed assistants already sending users, so both measures are needed: one shows where you stand on a fixed sample, the other shows real users.

## Checklist

- [ ] Sign-up asks "How did you hear about us?" with a separate AI assistant option, and the breakdown is visible.
- [ ] `robots.txt` blocks only private paths; CDN AI-crawler settings checked.
- [ ] `/llms.txt` returns `text/plain`, starts with the name and a one-line summary, lists every public page with absolute URLs, and appends the blog automatically.
- [ ] `Accept: text/markdown` on `/`, `/about`, `/contact` and a blog post returns Markdown with `Vary: Accept` and `X-Robots-Tag: noindex`.
- [ ] A browser `Accept` header gets HTML, with `Vary: Accept` on those paths.
- [ ] An unknown path with `Accept: text/markdown` returns a Markdown 404 linking home, `llms.txt` and the sitemap.
- [ ] The `wantsMarkdown` test runs in CI.
- [ ] `/about` and `/contact` have 500+ characters each, a real person and a real email, and are in the sitemap, footer and `llms.txt`.
- [ ] Organization JSON-LD has `email`, `contactPoint`, `founder` and `sameAs`, and passes the validators.
- [ ] A buyer prompt list of about 50 prompts, each with a Google query and a target page, with gaps marked.
- [ ] `facts.md` exists, and a test fails when its prices drift from the constants.
- [ ] A dated monthly citation check, with Named, Linked and Absent counts per assistant.

## Common mistakes

- **Treating `llms.txt` as a ranking trick.** It is a summary for agents. Google doesn't use it. Keep it accurate and short, and spend the real effort on pages and third-party mentions.
- **Writing `llms.txt` once and forgetting it.** An old price there is worse than no file. Update it with every plan change.
- **Serving Markdown to browsers.** Check for `text/markdown` explicitly and compare q-values. A rule like "anything that isn't a browser user agent gets Markdown" breaks link previews and some crawlers.
- **Forgetting `Vary: Accept`.** A cache can then hand the Markdown copy to a browser, or the HTML copy to an agent.
- **Letting Markdown copies get indexed.** Without `X-Robots-Tag: noindex` you create a duplicate of every page.
- **Serving Markdown for signed-in pages.** Only public pages with a real Markdown source belong in the map.
- **A soft 404.** Unknown paths must return status 404, not a 200 with "not found" text.
- **An invented address or fake team in JSON-LD.** Publish only what is true.
- **Prompts written like keywords.** "invoice tool free" is a Google query. Buyers ask assistants full questions with context; write them that way.
- **Checking citations while signed in.** Memory and past chats make you look more visible than you are.
- **Fixing only your own site when an assistant gets a fact wrong.** The wrong fact usually comes from a directory, a review site or an old article. Fix it at the source.
