# 3. Page programs

How to build the repeatable public pages (use cases, comparisons, alternatives, templates, free tools, embeds, trust pages and hubs) that bring search traffic to a SaaS, each from one registry, one template and one data file.

## What you'll build

- A **registry** per program: one typed list that feeds the routes, the sitemap, the hub pages, the footer and `llms.txt`, plus a test that keeps them in sync.
- **Use-case pages** `/for/<slug>` that show your product's real output for a sample input.
- **Comparison pages** `/vs/<rival>` with dated, sourced facts and a fair "good fit" section.
- **Alternatives pages** `/alternatives/<rival>` and neutral **head-to-head pages** `/compare/<a>-vs-<b>`.
- A **template gallery** `/templates/<slug>` where "Use this template" copies a full example into the visitor's account, even across sign-up.
- **Free tools** `/tools/<slug>`: browser-only tools and AI tools with abuse limits and an email gate.
- An **embeddable widget** under `/embed/<tool>` that earns credit links from other sites.
- **Trust pages** (`/about`, `/contact`, `/about/<founder>`), **hubs** and a **footer** of link lists.
- A **CTA tracking pattern** (`data-cta`, `data-cta-location`, `data-section`) shared by every page.

The code is Next.js App Router with TypeScript. The API examples use a SQLite style database (Cloudflare D1's `prepare().bind()` API). The SQL works the same in plain SQLite, and needs small changes for Postgres.

**Running example.** Code samples use a made-up product: `<Brand>` turns files and notes into finished reports. Swap in your own nouns. Rivals are `<Rival A>`, `<Rival B>` and so on.

---

## Contents

1. [Shared building blocks](#1-shared-building-blocks)
2. [The registry pattern](#2-the-registry-pattern)
3. [Use-case pages `/for/<slug>`](#3-use-case-pages-forslug)
4. [Comparison pages `/vs/<rival>`](#4-comparison-pages-vsrival)
5. [Alternatives and head-to-head pages](#5-alternatives-alternativesrival-and-head-to-head-compare-a-vs-b)
6. [Template gallery `/templates/<slug>`](#6-template-gallery-templatesslug)
7. [Free tools `/tools/<slug>`](#7-free-tools-toolsslug)
8. [Embeddable widget `/embed/<tool>`](#8-embeddable-widget-embedtool)
9. [Trust pages, hubs and the footer](#9-trust-pages-hubs-and-the-footer)
10. [Launch checklist per program](#10-launch-checklist-per-program)
11. [Common mistakes](#11-common-mistakes)

---

## 1. Shared building blocks

Every program uses the same four helpers. Write them once.

### 1.1 Site URL and metadata

```ts
// lib/site.ts
export const SITE_URL = "https://example.com";
export const BRAND = "<Brand>";
```

```ts
// lib/page-metadata.ts
import type { Metadata } from "next";
import { BRAND, SITE_URL } from "./site";

export function pageMetadata(path: string, title: string, description: string): Metadata {
  const url = `${SITE_URL}${path}`;
  return {
    title,
    description,
    alternates: { canonical: url },
    openGraph: {
      type: "website",
      siteName: BRAND,
      url,
      title: `${title} · ${BRAND}`,
      description,
      images: [{ url: "/og.png", width: 1200, height: 630, alt: `${BRAND}` }],
    },
  };
}
```

Rules for every page:

- Title 70 characters or fewer. Description 70 to 160 characters.
- The canonical is the absolute URL on your one host.
- Each programmatic page is its **own static route file** with a static `metadata` export. Some setups stream `generateMetadata` output into `<body>` instead of `<head>` (vinext does this; recent Next.js versions stream it to browsers and block only for known bots). A static export always lands in the head.

### 1.2 JSON-LD

```ts
// lib/json-ld.ts
export const ldJson = (data: unknown) => JSON.stringify(data).replace(/</g, "\\u003c");

export const faqLd = (faq: [string, string][]) =>
  ldJson({
    "@context": "https://schema.org",
    "@type": "FAQPage",
    mainEntity: faq.map(([q, a]) => ({ "@type": "Question", name: q, acceptedAnswer: { "@type": "Answer", text: a } })),
  });
```

The `<` escape stops a `</script>` inside your copy from breaking out of the tag. Render it like this:

```tsx
<script type="application/ld+json" dangerouslySetInnerHTML={{ __html: faqLd(faq) }} />
```

FAQ rich results in Google are now limited to a few kinds of sites, but FAQPage markup still tells Bing and AI assistants what the page answers. Keep it, and make sure every question in the JSON-LD is also visible on the page.

### 1.3 The CTA tracking pattern

Every link and button that matters carries two data attributes. Every section carries one.

| Attribute | Value | Example |
|---|---|---|
| `data-cta` | What the control does | `sign_up`, `usecase_link`, `vs_link`, `use_template`, `tool_link` |
| `data-cta-location` | Where it sits | `usecase_csv-to-report`, `usecase_csv-to-report_closing`, `footer_compare` |
| `data-section` | Which block of the page | `usecase_hero`, `vs_table`, `tool_faq` |

One client component, mounted once per marketing page, turns them into two events:

```tsx
// components/landing-tracking.tsx
"use client";
import { useEffect } from "react";
import { track } from "@/lib/analytics"; // your wrapper around posthog.capture or similar

export function LandingTracking() {
  useEffect(() => {
    const onClick = (e: MouseEvent) => {
      const el = (e.target as Element | null)?.closest<HTMLElement>("[data-cta]");
      if (!el) return;
      track("landing_cta_clicked", { cta: el.dataset.cta!, location: el.dataset.ctaLocation ?? "unknown" });
    };
    const seen = new Set<string>();
    const observer = new IntersectionObserver(
      (entries) => {
        for (const entry of entries) {
          const section = (entry.target as HTMLElement).dataset.section!;
          if (entry.isIntersecting && !seen.has(section)) {
            seen.add(section);
            track("landing_section_viewed", { section });
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

Naming rules:

1. Location is `<program>_<slug>`, so you can group by program or by page in one query.
2. The hero CTA and the closing CTA differ only by a `_closing` suffix. You learn whether people convert at the top or after reading.
3. Footer links use `footer_<column>`.
4. Never put emails, file names or user text in event properties.
5. Send marketing pages to a separate analytics project (or at least a separate event namespace) so app events don't drown them.

The primary CTA is one shared component:

```tsx
// components/cta.tsx
import Link from "next/link";

export function Cta({ location, href = "/signup", label = "Get started free", className }: { location: string; href?: string; label?: string; className?: string }) {
  return (
    <Link href={href} data-cta="sign_up" data-cta-location={location} className={className}>
      {label}
    </Link>
  );
}
```

### 1.4 Link cards

Related links, hub entries and "keep exploring" blocks all use one card list:

```tsx
// components/link-cards.tsx
import Link from "next/link";

export type LinkCard = { href: string; label: string; blurb: string; cta: string };

export function LinkCards({ links, location }: { links: LinkCard[]; location: string }) {
  return (
    <ul className="grid gap-3 sm:grid-cols-2 lg:grid-cols-3">
      {links.map((l) => (
        <li key={l.href}>
          <Link href={l.href} data-cta={l.cta} data-cta-location={location} className="block h-full rounded-2xl p-5">
            <span className="block font-semibold">{l.label}</span>
            <span className="block text-sm opacity-70">{l.blurb}</span>
          </Link>
        </li>
      ))}
    </ul>
  );
}
```

---

## 2. The registry pattern

### Why it matters

A page program has five places that list its pages: the route files, the sitemap, the hub page, the footer and `llms.txt`. If each keeps its own list, they drift. A page ships without a sitemap entry, a footer link points to a deleted page, or a hub misses half the program. A registry is one typed array that all five read.

### 2.1 Shape of a registry

Each entry holds only what lists need: a slug, a short label, a blurb of 8 words or fewer, and a group. The full page copy lives in a separate data file keyed by the same slug.

```ts
// lib/registry/use-cases.ts
export const USE_CASE_GROUPS = [
  { id: "files", label: "Files and links" },
  { id: "business", label: "Business" },
] as const;

type GroupId = (typeof USE_CASE_GROUPS)[number]["id"];

export const USE_CASES = [
  { slug: "csv-to-report", group: "files", label: "CSV to report", blurb: "A CSV export, as a finished report." },
  { slug: "pdf-to-summary", group: "files", label: "PDF to summary", blurb: "A long PDF, made short." },
  { slug: "board-update", group: "business", label: "Board update", blurb: "Board notes to a clean update." },
  { slug: "weekly-status-report", group: "business", label: "Weekly status report", blurb: "Your tracker, as a status report." },
] as const satisfies readonly { slug: string; group: GroupId; label: string; blurb: string }[];

export type UseCaseSlug = (typeof USE_CASES)[number]["slug"];
export const useCase = (slug: UseCaseSlug) => USE_CASES.find((u) => u.slug === slug)!;

export const FOOTER_USE_CASES: UseCaseSlug[] = ["csv-to-report", "pdf-to-summary", "board-update"];
```

The `as const` turns every slug into a literal type. `UseCaseSlug` is then a union, so `CASES: Record<UseCaseSlug, UseCase>` fails to compile when a slug has no copy, and `related: ["csv-to-reprot"]` fails on the typo.

Rivals and pairs work the same way:

```ts
// lib/registry/compare.ts
export const RIVAL_LIST = [
  { slug: "rival-a", name: "<Rival A>", blurb: "<Rival A> alternative with free exports." },
  { slug: "rival-b", name: "<Rival B>", blurb: "A <Rival B> alternative for small teams." },
  { slug: "rival-c", name: "<Rival C>", blurb: "<Rival C> alternative, no seat minimum." },
] as const;

export type RivalSlug = (typeof RIVAL_LIST)[number]["slug"];

// Pairs that autocomplete suggests in both orders (see section 5.2). Record the engine and the month.
export const HEAD_TO_HEAD = [
  { slug: "rival-a-vs-rival-b", a: "rival-a", b: "rival-b", label: "<Rival A> vs <Rival B>", blurb: "Speed or a design suite." },
  { slug: "rival-b-vs-rival-c", a: "rival-b", b: "rival-c", label: "<Rival B> vs <Rival C>", blurb: "Templates or data visuals." },
] as const satisfies readonly { slug: string; a: RivalSlug; b: RivalSlug; label: string; blurb: string }[];

export type HeadToHeadSlug = (typeof HEAD_TO_HEAD)[number]["slug"];

export const vsPath = (slug: RivalSlug) => `/vs/${slug}`;
export const alternativePath = (slug: RivalSlug) => `/alternatives/${slug}`;
export const headToHeadPath = (slug: HeadToHeadSlug) => `/compare/${slug}`;

export const VS_PATHS = RIVAL_LIST.map((r) => vsPath(r.slug));
export const ALTERNATIVE_PATHS = ["/alternatives", ...RIVAL_LIST.map((r) => alternativePath(r.slug))];
export const HEAD_TO_HEAD_PATHS = ["/compare", ...HEAD_TO_HEAD.map((c) => headToHeadPath(c.slug))];
```

Keep registry files free of `@/` path aliases and framework imports. The test below runs them with plain Node, which doesn't know your aliases.

> Example from MakeDecks: one registry file for use cases and `/vs` pages (23 use cases in 4 groups, 7 rivals), one for alternatives and head-to-head pairs, and the template and tool registries in a shared package so the API can read them too.

### 2.2 Everything reads the registry

**Sitemap**

```ts
// app/sitemap.ts
import type { MetadataRoute } from "next";
import { ALTERNATIVE_PATHS, HEAD_TO_HEAD_PATHS, VS_PATHS } from "@/lib/registry/compare";
import { TEMPLATES } from "@/lib/registry/templates";
import { TOOLS } from "@/lib/registry/tools";
import { USE_CASES } from "@/lib/registry/use-cases";
import { SITE_URL } from "@/lib/site";

const lastModified = "2026-10-24";

export const staticPages = (): MetadataRoute.Sitemap =>
  [
    "", "/pricing", "/about", "/about/<founder-slug>", "/contact", "/privacy",
    "/use-cases", "/tools", "/templates",
    ...USE_CASES.map((u) => `/for/${u.slug}`),
    ...VS_PATHS,
    ...ALTERNATIVE_PATHS,
    ...HEAD_TO_HEAD_PATHS,
    ...TEMPLATES.map((t) => `/templates/${t.slug}`),
    ...TOOLS.map((t) => `/tools/${t.slug}`),
  ].map((path) => ({ url: `${SITE_URL}${path}`, lastModified }));

export default function sitemap(): MetadataRoute.Sitemap {
  return staticPages();
}
```

Bump `lastModified` when you change page copy in bulk. Don't set it to "now" on every build: search engines learn to ignore a date that always changes.

**Footer** (section 9.4), **hubs** (section 9.3) and **`llms.txt`**:

```ts
// app/llms.txt/route.ts
import { LLMS_INTRO } from "@/content/llms"; // your hand-written product summary, as a string
import { RIVAL_LIST } from "@/lib/registry/compare";
import { TOOLS } from "@/lib/registry/tools";
import { USE_CASES } from "@/lib/registry/use-cases";
import { SITE_URL } from "@/lib/site";

const section = (title: string, rows: { path: string; label: string; blurb: string }[]) =>
  `## ${title}\n\n${rows.map((r) => `- [${r.label}](${SITE_URL}${r.path}): ${r.blurb}`).join("\n")}`;

export function GET() {
  const body = [
    LLMS_INTRO.trimEnd(),
    section("Use cases", USE_CASES.map((u) => ({ path: `/for/${u.slug}`, label: u.label, blurb: u.blurb }))),
    section("Comparisons", RIVAL_LIST.map((r) => ({ path: `/vs/${r.slug}`, label: `<Brand> vs ${r.name}`, blurb: r.blurb }))),
    section("Free tools", TOOLS.map((t) => ({ path: `/tools/${t.slug}`, label: t.name, blurb: t.blurb }))),
  ].join("\n\n");
  return new Response(`${body}\n`, { headers: { "content-type": "text/plain; charset=utf-8", "cache-control": "public, max-age=300" } });
}
```

### 2.3 The sync test

Static route files are good for metadata, but now the route list and the registry can drift. A small test closes the gap. It uses Node's built-in test runner, so there's nothing to install.

```ts
// lib/registry.test.ts
import assert from "node:assert/strict";
import { existsSync, readdirSync, readFileSync } from "node:fs";
import { test } from "node:test";
import { HEAD_TO_HEAD, HEAD_TO_HEAD_PATHS, RIVAL_LIST, ALTERNATIVE_PATHS, VS_PATHS } from "./registry/compare.ts";
import { USE_CASES } from "./registry/use-cases.ts";

const app = new URL("../app/", import.meta.url);
const page = (path: string) => new URL(`.${path}/page.tsx`, app);
const dirs = (dir: string) =>
  readdirSync(new URL(dir, app), { withFileTypes: true }).filter((d) => d.isDirectory()).map((d) => d.name).sort();

test("every registry entry has its own static route, and no route is unlisted", () => {
  const paths = [...USE_CASES.map((u) => `/for/${u.slug}`), ...VS_PATHS, ...ALTERNATIVE_PATHS, ...HEAD_TO_HEAD_PATHS];
  for (const path of paths) assert.ok(existsSync(page(path)), `${path} has no page.tsx`);
  for (const { slug } of RIVAL_LIST) assert.match(readFileSync(page(`/vs/${slug}`), "utf8"), new RegExp(`"${slug}"`));
  assert.deepEqual(dirs("for"), USE_CASES.map((u) => u.slug).sort());
  assert.deepEqual(dirs("vs"), RIVAL_LIST.map((r) => r.slug).sort());
  assert.deepEqual(dirs("alternatives"), RIVAL_LIST.map((r) => r.slug).sort());
  assert.deepEqual(dirs("compare"), HEAD_TO_HEAD.map((c) => c.slug).sort());
});

test("head-to-head pairs are distinct and slugs match their pair", () => {
  const pairs = HEAD_TO_HEAD.map((c) => [c.a, c.b].sort().join("+"));
  assert.equal(new Set(pairs).size, pairs.length);
  for (const c of HEAD_TO_HEAD) assert.equal(c.slug, `${c.a}-vs-${c.b}`);
});

test("page copy has no em or en dashes", () => {
  for (const file of ["for/samples.ts", "vs/rivals.ts", "alternatives/data.ts", "compare/data.ts"]) {
    assert.doesNotMatch(readFileSync(new URL(file, app), "utf8"), /[\u2013\u2014]/, file);
  }
});
```

Run it with `node --test --experimental-strip-types lib/*.test.ts` (Node 22.6 or later; newer Node versions strip types by default). Add it to CI.

The second check, that each `page.tsx` contains its own slug in quotes, catches the most common copy-paste bug: a new folder whose page still renders the slug of the folder it was copied from.

### 2.4 Adding a page, step by step

1. Add the entry to the registry.
2. Add its copy to the data file. TypeScript now fails until you do.
3. Copy any sibling route folder, rename it and change the slug in both places in `page.tsx`.
4. Run the test. Sitemap, hub, footer and `llms.txt` already include it.

---

## 3. Use-case pages `/for/<slug>`

### Why they rank

These pages target bottom-of-funnel searches where the visitor already has the input and wants the output: "<input> to <output>", "convert <file type> to <output>", "AI <thing> maker", "<job> template". Intent is high, competition is often weaker than for head terms, and a page that shows a real result for that exact input is a better answer than a generic home page.

### URL pattern

`/for/<slug>`, where the slug repeats the query: `/for/csv-to-report`, `/for/board-update`. Hub at `/use-cases`.

### 3.1 How to pick targets

1. **List what your product really accepts and produces.** File types, link types, integrations, formats you export. Each pair is a candidate: "<input> to <output>".
2. **List the jobs people do with it.** Board updates, weekly reports, lesson plans. Each is a candidate: "<job> maker", "<job> template", "AI <job> generator".
3. **Expand with autocomplete.** Type each seed into Google, Bing and DuckDuckGo followed by a space, then by each letter a to z ("csv to a", "csv to b"...). Note every suggestion that names your output. Autocomplete only shows queries with real volume, which is all you need at this stage.
4. **Check Search Console.** Queries where you already get impressions but have no dedicated page are the best targets of all.
5. **Look at the results page for each candidate.** If the top results are tools that do the conversion, a use-case page fits. If they are tutorials, a blog post fits better. If they are template galleries, build a template (section 6).
6. **Verify the claim in code.** Before you write the page, run the exact input through the product. If it fails or works only sometimes, drop the page. A page that promises an input you can't handle creates sign-ups that churn on day one.

> Example from MakeDecks: a "Notion to slides" page was skipped because public Notion pages need a JavaScript render that often timed out. Audio, transcript, Google Docs, Markdown, CSV and image pages were added only after each input was tested end to end.

Keep a short table while you research:

| Query | Engine suggestions seen | SERP type | Product handles it? | Decision |
|---|---|---|---|---|
| csv to report | Google, Bing | Tools | Yes, tested | Build `/for/csv-to-report` |
| notion to report | Google | Tools | Flaky | Skip |
| how to write a board update | Google | Guides | n/a | Blog post |

### 3.2 Data shape

```ts
// app/for/samples.ts
import type { UseCaseSlug } from "@/lib/registry/use-cases";
import type { ReportBlock } from "@/lib/product-types"; // the same type your real renderer takes

export type Source =
  | { kind: "notes"; name: string; lines: string[] }
  | { kind: "sheet"; name: string; rows: string[][] }
  | { kind: "pdf"; name: string; pages: number; page: number; heading: string; paragraphs: string[]; table: string[][] }
  | { kind: "web"; name: string; title: string; byline: string; paragraphs: string[] };

export type UseCase = {
  title: string;              // <title>, 70 chars or fewer
  description: string;        // meta description, 70 to 160 chars
  h1: string;
  sub: string;                // one short sentence, 12 words or fewer
  source: Source;             // the sample input, shown as a card
  output: [ReportBlock, ReportBlock, ReportBlock, ReportBlock]; // real product output for that input
  proof: string;              // caption under the example
  drop: string;               // step 1 text: what the user drops in
  faq: [string, string][];
  closing: string;            // closing section heading
  related: [UseCaseSlug, UseCaseSlug, UseCaseSlug];
};

export const CASES: Record<UseCaseSlug, UseCase> = {
  "csv-to-report": {
    title: "CSV to report: turn an export into a finished report",
    description: "Turn a CSV export into a designed report. Numbers stay exact, charts are built for you, and you download an editable file. Free to start.",
    h1: "CSV export to a finished report.",
    sub: "Drop in a CSV. Charts and tables keep every number exact.",
    source: {
      kind: "sheet",
      name: "q3-revenue-by-region.csv",
      rows: [
        ["Region", "Q2 ($M)", "Q3 ($M)"],
        ["Europe", "1.42", "1.81"],
        ["North America", "1.12", "1.37"],
        ["Total", "2.54", "3.18"],
      ],
    },
    output: [
      { layout: "chart", title: "Both regions grew, Europe added the most", chart: { kind: "bar", labels: ["Europe", "North America"], series: [{ name: "Q2 ($M)", values: [1.42, 1.12] }, { name: "Q3 ($M)", values: [1.81, 1.37] }] } },
      { layout: "title", title: "Q3 revenue review", subtitle: "Revenue by region, quarter over quarter" },
      { layout: "big-number", title: "Revenue this quarter", value: "$3.18M", label: "Up from $2.54M in Q2" },
      { layout: "table", title: "Q2 to Q3, region by region", columns: ["Region", "Q2 ($M)", "Q3 ($M)"], rows: [["Europe", "1.42", "1.81"], ["North America", "1.12", "1.37"]] },
    ],
    proof: "Same figures as the CSV, nothing rounded.",
    drop: "Upload the CSV export.",
    faq: [
      ["Does it keep my numbers exact?", "Yes. Values from the file are used as written. Check and edit any part before you share it."],
      ["Is my file stored?", "<Say exactly what happens to the file, matching your privacy page.>"],
      ["Can I edit the result?", "<Say exactly how, and in which formats it downloads.>"],
    ],
    closing: "Your next report is one upload away.",
    related: ["pdf-to-summary", "weekly-status-report", "board-update"],
  },
  // one entry per slug; TypeScript fails until every slug has one
};
```

Pull limits and prices from your shared constants instead of typing them (`` `${LIMITS.uploadMb} MB` ``). When a limit changes, every page changes with it.

### 3.3 Page template, section by section

| # | Section | `data-section` | Content |
|---|---|---|---|
| 1 | Hero | `usecase_hero` | H1, one-line sub, primary CTA (`location: usecase_<slug>`) |
| 2 | Example | `usecase_example` | Source card, an arrow, the lead output, then three smaller outputs; caption with the proof line and "Example, names are fictional." |
| 3 | Video (optional) | `usecase_video` | A real recording, with a link to its watch page |
| 4 | How it works | `usecase_steps` | Three steps: drop it in, optional question, edit and download |
| 5 | FAQ | `usecase_faq` | Three or four real questions, also in FAQPage JSON-LD |
| 6 | More ways to start | `usecase_more` | Three related use cases plus an "All use cases" card |
| 7 | Closing | `usecase_closing` | Closing line, CTA (`usecase_<slug>_closing`), one line of plan facts |

**The visuals must be real product output.** Render the `output` blocks with the same component your app uses. No mock screenshots, no designed-in-Figma "examples". It is more honest, it stays current when the product changes, and visitors can tell.

**The numbers must match.** Every figure in the output appears in the source card, or is derived from it in a way a reader can check (a total, a percent change). A visitor who spots a number that isn't in the input stops trusting the page.

### 3.4 Minimal working code

```tsx
// app/for/use-case.tsx
import Link from "next/link";
import { Cta } from "@/components/cta";
import { LandingTracking } from "@/components/landing-tracking";
import { LinkCards } from "@/components/link-cards";
import { ReportBlockView } from "@/components/report-block-view"; // your real renderer
import { faqLd } from "@/lib/json-ld";
import { pageMetadata } from "@/lib/page-metadata";
import { useCase, type UseCaseSlug } from "@/lib/registry/use-cases";
import { CASES, type Source } from "./samples";

export const useCaseMetadata = (slug: UseCaseSlug) => pageMetadata(`/for/${slug}`, CASES[slug].title, CASES[slug].description);

function SourceCard({ source }: { source: Source }) {
  return (
    <div className="rounded-2xl p-4 ring-1 ring-black/10">
      <p className="text-xs font-medium">{source.name}</p>
      {source.kind === "notes" && source.lines.map((l) => <p key={l} className="font-mono text-sm">{l}</p>)}
      {source.kind === "sheet" && (
        <table className="w-full font-mono text-xs">
          <tbody>
            {source.rows.map((r, i) => (
              <tr key={i}>{r.map((v, j) => <td key={j} className="border px-2 py-1">{v}</td>)}</tr>
            ))}
          </tbody>
        </table>
      )}
      {/* one branch per Source kind */}
    </div>
  );
}

export function UseCasePage({ slug }: { slug: UseCaseSlug }) {
  const c = CASES[slug];
  const [lead, ...rest] = c.output;
  const loc = `usecase_${slug}`;
  const related = [
    ...c.related.map(useCase).map((u) => ({ href: `/for/${u.slug}`, label: u.label, blurb: u.blurb, cta: "usecase_link" })),
    { href: "/use-cases", label: "All use cases", blurb: "Every way to start.", cta: "use_cases_link" },
  ];

  return (
    <div>
      <LandingTracking />
      <script type="application/ld+json" dangerouslySetInnerHTML={{ __html: faqLd(c.faq) }} />

      <section data-section="usecase_hero">
        <h1>{c.h1}</h1>
        <p>{c.sub}</p>
        <Cta location={loc} />
      </section>

      <section data-section="usecase_example" aria-label="Example">
        <figure>
          <div className="grid items-center gap-6 md:grid-cols-[5fr_9fr]">
            <SourceCard source={c.source} />
            <ReportBlockView block={lead} />
          </div>
          <div className="mt-6 grid gap-4 sm:grid-cols-3">
            {rest.map((b, i) => (
              <div key={i} className="aspect-video">
                <ReportBlockView block={b} />
              </div>
            ))}
          </div>
          <figcaption>{c.proof} Example, names are fictional.</figcaption>
        </figure>
      </section>

      <section data-section="usecase_steps">
        <h2>How it works</h2>
        <ol>
          <li><h3>Drop it in</h3><p>{c.drop}</p></li>
          <li><h3>Check the draft</h3><p>Fix anything before you share it.</p></li>
          <li><h3>Download</h3><p>An editable file, ready to send.</p></li>
        </ol>
      </section>

      <section data-section="usecase_faq" aria-labelledby="faq">
        <h2 id="faq">Questions</h2>
        <dl>
          {c.faq.map(([q, a]) => (
            <div key={q}><dt>{q}</dt><dd>{a}</dd></div>
          ))}
        </dl>
      </section>

      <nav data-section="usecase_more" aria-labelledby="more">
        <h2 id="more">More ways to start</h2>
        <LinkCards links={related} location={loc} />
      </nav>

      <section data-section="usecase_closing">
        <h2>{c.closing}</h2>
        <Cta location={`${loc}_closing`} />
        <p><Link href="/pricing">See plans</Link></p>
      </section>
    </div>
  );
}
```

```tsx
// app/for/csv-to-report/page.tsx
import { UseCasePage, useCaseMetadata } from "../use-case";

export const metadata = useCaseMetadata("csv-to-report");

export default function Page() {
  return <UseCasePage slug="csv-to-report" />;
}
```

Performance notes:

- Render the hero output eagerly. Mount the three smaller outputs only when they come within a screen of the viewport, inside a box that already has the final size (`aspect-video`), so nothing shifts.
- If the output includes images, serve them with `srcset` at about 720 and 1080 pixels wide.

### 3.5 Internal linking

- Three related use cases, chosen by meaning (same input type, or same job), plus "All use cases".
- The hub `/use-cases` lists every page by group.
- The footer lists your top five or six.
- Blog posts link to the use-case page that matches their topic.
- If a matching template exists, link it, and link back from the template.

### 3.6 JSON-LD

FAQPage on every use-case page. Add BreadcrumbList if you show breadcrumbs. The Organization and WebSite graph stays on the home page only.

### 3.7 Quality rules

1. **Only claim what the product supports.** Every input type, file size, page count and format on the page matches a real limit in code, read from the same constant.
2. Every number in the output appears in the sample input, or is a checkable sum or change.
3. Sample names are fictional and the caption says so.
4. The FAQ answers what a buyer would ask about this input: accuracy, privacy, editing, format. Not generic marketing.
5. One H1 that matches the query. One sub line. No paragraph of marketing copy above the fold.
6. Each page is genuinely different: different input, different output, different FAQ. If two pages would show the same example, merge them.

---

## 4. Comparison pages `/vs/<rival>`

### Why they rank

"<Your brand> vs <Rival>" and "<Rival> alternative" searches come from people choosing a tool. Many go to the rival's own comparison pages or to review sites. A clear, fair page with a feature table and checked facts can rank and converts well, because the visitor is already in buying mode.

### URL pattern

`/vs/<rival-slug>`, for example `/vs/rival-a`. Ordered in the registry by relevance (search volume and how often prospects mention them).

### 4.1 How to pick targets

1. List the tools your users mention: in sign-up surveys ("What did you use before?"), support emails, sales calls and churn reasons.
2. Type "<Rival> vs" and "<Rival> alternative" into autocomplete. Rivals with many suggestions have demand.
3. Check Search Console for queries that already contain a rival's name.
4. Start with five to seven rivals. Each page needs real research, so more pages means more facts to keep current.

### 4.2 Fact rules

These are not optional. A wrong fact about a rival is a legal risk and an instant loss of trust.

1. **Facts come only from the rival's own pages:** their pricing page, help center and official docs. Never from review sites, blog posts, AI answers or memory.
2. **Every page records the month you checked** and links each source. The footnote reads: "<Rival> details checked <Month Year> from their pricing page and help center. Plans change, so check their site."
3. **If you can't verify a cell, write "Not checked"** or leave the row out. Never guess.
4. **Quote their wording** for plan names and limits when you can, and attribute it ("per its pricing page").
5. **Re-check before editing any row,** and re-check every rival page on a schedule (quarterly is a good default). Update the month when you do.
6. **Your own column reads from your constants,** so it can't drift from your pricing page.

### 4.3 Legal caution

This is not legal advice. Check with a lawyer in your market before you publish.

- **Use names nominatively.** You may name a rival to say what you compare against. Don't use their logo, colours or screenshots of their product, and don't imply a partnership or endorsement.
- **Comparative advertising rules** in many places (the EU and UK, for example) allow comparisons that are objective, verifiable, not misleading and about like-for-like features. Dated, linked sources are how you show "verifiable".
- **Don't disparage.** State facts. "Exports carry a badge on the free plan, per their help center" is a fact. "Their exports look terrible" is not.
- **Respond fast to complaints.** If a rival says a fact is out of date, check it against their page that day and fix it or reply with your source.

### 4.4 Data shape

```ts
// app/vs/rivals.ts
import { PLAN } from "@/lib/plans"; // your own prices and limits, one source
import type { RivalSlug } from "@/lib/registry/compare";

export type Cell = boolean | string;

export type Rival = {
  name: string;
  title: string;
  description: string;
  h1: string;
  sub: string;
  why: [string, string][];          // three "why people switch" cards: title, body
  rows: [string, Cell, Cell][];     // feature, yours, theirs
  sources: [string, string][];      // label, URL on the rival's own site
  checked: string;                  // "October 2026"
  fit: string;                      // where the rival is the better choice
  faq: [string, string][];
  closing: string;
};

const FREE = `${PLAN.free.reportsPerMonth} reports a month`;

export const RIVALS: Record<RivalSlug, Rival> = {
  "rival-a": {
    name: "<Rival A>",
    title: "<Rival A> alternative with editable exports",
    description: `Compare <Brand> and <Rival A>: ${FREE} free, editable exports and native charts. Facts checked on <Rival A>'s own pages.`,
    h1: "The <Rival A> alternative that stays free.",
    sub: `${FREE}, editable files, no badge.`,
    why: [
      ["Free every month", `${FREE}, not one-time credits.`],
      ["No badge on downloads", "Your files stay unbranded."],
      ["Spreadsheets in, charts out", "Numbers become native, editable charts."],
    ],
    rows: [
      ["Free plan", FREE, "<their free plan, worded as on their pricing page>"],
      ["Paid plans", PLAN.pro.label, "<their entry paid plan, as on their pricing page>"],
      ["Editable export", true, true],
      ["Excel as a source", "Excel or CSV", "Not checked"],
      ["Publish as a website", false, true],
    ],
    sources: [
      ["pricing page", "https://rival-a.example/pricing"],
      ["export help", "https://help.rival-a.example/export"],
    ],
    checked: "October 2026",
    fit: "Reports that live on the web. <Rival A> publishes any report as a page with viewer analytics.",
    faq: [
      ["Is <Brand> free like <Rival A>?", `Yes, and it renews. Free gives you ${FREE}. <Rival A>'s free plan is <as worded on their pricing page>.`],
      ["Can I move my work from <Rival A>?", "Export it from <Rival A>, then drop the file into <Brand>."],
    ],
    closing: "Try it on your next report.",
  },
  // one entry per key; TypeScript fails until every key has one
};
```

Use `true` and `false` for plain yes or no cells. They render as a check or a dash with screen-reader text. Use a string whenever the answer needs a qualifier.

### 4.5 Page template, section by section

| # | Section | `data-section` | Content |
|---|---|---|---|
| 1 | Hero | `vs_hero` | Small line "<Brand> vs <Rival>", H1, sub, CTA |
| 2 | Proof | `vs_proof` | One real piece of your product's output, captioned |
| 3 | Why people switch | `vs_why` | Three short cards |
| 4 | Side by side | `vs_table` | The feature table and the dated source footnote |
| 5 | Where <Rival> is a good fit | `vs_fit` | One honest paragraph |
| 6 | FAQ | `vs_faq` | Questions people search: "Is X free?", "Does X export to Y?", "Can I move from X?" |
| 7 | Keep exploring | `vs_more` | Two other `/vs` pages, three use cases, "All use cases" |
| 8 | Closing | `vs_closing` | CTA |

**The "good fit" section is required.** Every tool is better at something. Saying so makes the rest of the page believable, and visitors who would churn anyway self-select out.

### 4.6 Minimal working code

```tsx
// app/vs/compare.tsx
import { Cta } from "@/components/cta";
import { LandingTracking } from "@/components/landing-tracking";
import { LinkCards } from "@/components/link-cards";
import { faqLd } from "@/lib/json-ld";
import { pageMetadata } from "@/lib/page-metadata";
import { RIVAL_LIST, vsPath, type RivalSlug } from "@/lib/registry/compare";
import { FOOTER_USE_CASES, useCase } from "@/lib/registry/use-cases";
import { RIVALS, type Cell } from "./rivals";

export const vsMetadata = (slug: RivalSlug) => pageMetadata(vsPath(slug), RIVALS[slug].title, RIVALS[slug].description);

export function Value({ cell }: { cell: Cell }) {
  if (typeof cell === "string") return cell;
  return cell ? <><span aria-hidden>✓</span><span className="sr-only">Yes</span></> : <><span aria-hidden>·</span><span className="sr-only">No</span></>;
}

export function VsPage({ slug }: { slug: RivalSlug }) {
  const r = RIVALS[slug];
  const loc = `vs_${slug}`;
  const i = RIVAL_LIST.findIndex((x) => x.slug === slug);
  const links = [
    ...[1, 2].map((k) => RIVAL_LIST[(i + k) % RIVAL_LIST.length]).map((x) => ({ href: vsPath(x.slug), label: `<Brand> vs ${x.name}`, blurb: x.blurb, cta: "vs_link" })),
    ...FOOTER_USE_CASES.slice(0, 3).map(useCase).map((u) => ({ href: `/for/${u.slug}`, label: u.label, blurb: u.blurb, cta: "usecase_link" })),
    { href: "/use-cases", label: "All use cases", blurb: "Every way to start.", cta: "use_cases_link" },
  ];

  return (
    <div>
      <LandingTracking />
      <script type="application/ld+json" dangerouslySetInnerHTML={{ __html: faqLd(r.faq) }} />

      <section data-section="vs_hero">
        <p>&lt;Brand&gt; vs {r.name}</p>
        <h1>{r.h1}</h1>
        <p>{r.sub}</p>
        <Cta location={loc} />
      </section>

      <section data-section="vs_why">
        <h2>Why people switch</h2>
        <ul>{r.why.map(([t, b]) => <li key={t}><h3>{t}</h3><p>{b}</p></li>)}</ul>
      </section>

      <section data-section="vs_table" aria-labelledby="table">
        <h2 id="table">Side by side</h2>
        <table className="w-full table-fixed">
          <caption className="sr-only">&lt;Brand&gt; compared with {r.name}</caption>
          <thead>
            <tr><th scope="col"><span className="sr-only">Feature</span></th><th scope="col">&lt;Brand&gt;</th><th scope="col">{r.name}</th></tr>
          </thead>
          <tbody>
            {r.rows.map(([feature, ours, theirs]) => (
              <tr key={feature}>
                <th scope="row" className="break-words">{feature}</th>
                <td className="break-words"><Value cell={ours} /></td>
                <td className="break-words"><Value cell={theirs} /></td>
              </tr>
            ))}
          </tbody>
        </table>
        <p className="text-xs">
          {r.name} details checked {r.checked} from their{" "}
          {r.sources.map(([label, href], k) => (
            <span key={href}>
              {k > 0 && (k === r.sources.length - 1 ? " and " : ", ")}
              <a href={href} rel="nofollow noopener" target="_blank">{label}</a>
            </span>
          ))}
          . Plans change, so check their site.
        </p>
      </section>

      <section data-section="vs_fit">
        <h2>Where {r.name} is a good fit</h2>
        <p>{r.fit}</p>
      </section>

      <section data-section="vs_faq">
        <h2>Questions</h2>
        <dl>{r.faq.map(([q, a]) => <div key={q}><dt>{q}</dt><dd>{a}</dd></div>)}</dl>
      </section>

      <nav data-section="vs_more">
        <h2>Keep exploring</h2>
        <LinkCards links={links} location={loc} />
      </nav>

      <section data-section="vs_closing">
        <h2>{r.closing}</h2>
        <Cta location={`${loc}_closing`} />
      </section>
    </div>
  );
}
```

```tsx
// app/vs/rival-a/page.tsx
import { VsPage, vsMetadata } from "../compare";

export const metadata = vsMetadata("rival-a");

export default function Page() {
  return <VsPage slug="rival-a" />;
}
```

Table notes: use `table-fixed` with `break-words` so long cells wrap at 360 px instead of scrolling the page sideways. Source links use `rel="nofollow noopener"`: you cite them, you don't vouch for them.

### 4.7 Internal linking and JSON-LD

- Link two other `/vs` pages (the next two in the registry, wrapping around), three top use cases and the use-case hub.
- The `/alternatives/<rival>` page links to `/vs/<rival>` and back.
- The footer "Compare" column lists the first four rivals plus "All comparisons".
- JSON-LD: FAQPage. Add BreadcrumbList if you show breadcrumbs.

### 4.8 Quality rules

1. Every rival cell is sourced from their own site, dated, or marked "Not checked".
2. No rival logos or screenshots.
3. The "good fit" section names a real strength.
4. Your own cells come from your constants.
5. A footnote with the checked month and links on every page.
6. A re-check date in your calendar.

---

## 5. Alternatives `/alternatives/<rival>` and head-to-head `/compare/<a>-vs-<b>`

### Why they rank

"<Rival> alternatives" is a bigger and earlier search than "<Brand> vs <Rival>": the visitor knows they want to leave but hasn't picked where to go. Results are usually listicles. A well-sourced list that puts you first and is honest about everyone else can compete.

Head-to-head pages ("<Rival A> vs <Rival B>") don't mention you in the query at all. They capture people comparing two tools you compete with, and introduce you as a third option. They must be genuinely neutral, or they won't be trusted or ranked.

### URL patterns

- `/alternatives` hub, `/alternatives/<rival>` per rival.
- `/compare` hub, `/compare/<a>-vs-<b>` per pair. The slug uses the order you chose; the reverse order is not a separate page.

### 5.1 Shared product data

Both programs read one product table, so a fact about a rival is written once, sourced once and dated once.

```ts
// app/alternatives/products.ts
import { PLAN } from "@/lib/plans";
import type { RivalSlug } from "@/lib/registry/compare";

export type ProductKey = RivalSlug | "brand" | "generic-tool"; // include widely used free tools people consider, too

export type Product = {
  name: string;
  url: string;
  free: string;
  paid: string;
  export: string;
  ai: string;
  collab: string;
  standout: string;
  bestFor: string;          // completes "Best for ..."
  sources: [string, string][];
};

export const CHECKED = "October 2026";

export const PRODUCTS: Record<ProductKey, Product> = {
  brand: {
    name: "<Brand>",
    url: "https://example.com",
    free: `${PLAN.free.reportsPerMonth} reports a month`,
    paid: PLAN.pro.label,
    export: "Every plan, editable",
    ai: "From a prompt, PDF, Word, Excel or a link",
    collab: "View links on Free, edit links on Pro",
    standout: "Numbers from your files become editable charts",
    bestFor: "turning your own files into an editable report.",
    sources: [],
  },
  "rival-a": {
    name: "<Rival A>",
    url: "https://rival-a.example",
    free: "<as worded on their pricing page>",
    paid: "<entry paid plan, as on their pricing page>",
    export: "<from their export help page>",
    ai: "<from their docs>",
    collab: "Not checked",
    standout: "<one real strength>",
    bestFor: "<the kind of user it really suits>.",
    sources: [["pricing", "https://rival-a.example/pricing"], ["export help", "https://help.rival-a.example/export"]],
  },
  // one entry per key; TypeScript fails until every key has one
};
```

```ts
// app/alternatives/data.ts
import type { RivalSlug } from "@/lib/registry/compare";
import type { UseCaseSlug } from "@/lib/registry/use-cases";
import type { ProductKey } from "./products";

export type Alt = {
  title: string;                        // "Best <Rival> alternatives in <year>"
  description: string;
  sub: string;
  reason: string;                       // why people look elsewhere, from facts on their pages
  ours: string;                         // why <Brand> is first
  picks: { key: ProductKey; why: string }[];   // 3 to 5 other real options
  choose: [string, string][];           // "If you need X" -> how to decide
  faq: [string, string][];
  forPages: UseCaseSlug[];
  closing: string;
};
```

```ts
// app/compare/data.ts
import type { HeadToHeadSlug } from "@/lib/registry/compare";
import type { UseCaseSlug } from "@/lib/registry/use-cases";
import type { Cell } from "../vs/rivals";

export type H2h = {
  title: string;
  description: string;
  sub: string;
  pickA: string;                                  // "Pick A if ..."
  pickB: string;                                  // "Pick B if ..."
  rows: [string, Cell, Cell][];                   // extra rows beyond the shared ones
  verdicts: [string, "a" | "b" | "tie", string][]; // use, winner, why
  tryUs: string;                                  // "Or try <Brand> if ..."
  faq: [string, string][];
  forPage: UseCaseSlug;
};
```

### 5.2 Choosing head-to-head pairs from autocomplete

Only build a pair when people search it **in both orders**: "a vs b" and "b vs a" both appear in autocomplete. One-way suggestions often reflect one brand's marketing; two-way ones reflect real comparison shopping.

1. Write down every rival in your registry.
2. For each rival, type "<rival> vs" into autocomplete and record which other rivals appear.
3. Keep a pair only if A's suggestions include B **and** B's suggestions include A.
4. Order the slug by the order that appears more often, or alphabetically if equal.
5. Record the engine and the month in a comment next to the registry.

A small script does steps 2 and 3. It uses DuckDuckGo's public autocomplete endpoint, which is unofficial: keep requests few and slow.

```js
// scripts/pairs.mjs
const RIVALS = { "rival-a": "rival a", "rival-b": "rival b", "rival-c": "rival c" }; // slug -> search name

const suggest = async (q) => {
  const res = await fetch(`https://duckduckgo.com/ac/?q=${encodeURIComponent(q)}`);
  const body = await res.json();
  return (Array.isArray(body[1]) ? body[1] : body.map((s) => s.phrase)).map((s) => s.toLowerCase());
};

const seen = {};
for (const [slug, name] of Object.entries(RIVALS)) {
  seen[slug] = await suggest(`${name} vs`);
  await new Promise((r) => setTimeout(r, 1500));
}

const slugs = Object.keys(RIVALS);
for (const a of slugs) {
  for (const b of slugs) {
    if (a >= b) continue;
    const ab = seen[a].some((s) => s.includes(RIVALS[b]));
    const ba = seen[b].some((s) => s.includes(RIVALS[a]));
    if (ab && ba) console.log(`${a}-vs-${b}`);
  }
}
```

Repeat the check in Google and Bing by hand for the pairs it finds. Use the same method with "<rival> alternative" to decide which alternatives pages to build first.

> Example from MakeDecks: 7 rivals gave 21 possible pairs; 6 showed up in both orders and became head-to-head pages.

### 5.3 Alternatives page, section by section

| # | Section | `data-section` | Content |
|---|---|---|---|
| 1 | Hero | `alt_hero` | "Best <Rival> alternatives in <year>", sub, CTA |
| 2 | Why people look elsewhere | `alt_reason` | One paragraph built from facts on their own pages |
| 3 | At a glance | `alt_table` | Your product, the picks, then the rival "(for reference)"; free plan, paid from, export; sources footnote |
| 4 | The alternatives | `alt_list` | Numbered cards: name, "Best for", why, four facts. Your card includes "Where <Rival> is better" |
| 5 | How to choose | `alt_choose` | Need, then how to decide, in four cards |
| 6 | FAQ | `alt_faq` | "What is the best free <Rival> alternative?" and similar |
| 7 | Keep exploring | `alt_more` | `/vs/<rival>`, head-to-heads with this rival, use cases, two other alternatives pages, the hub |
| 8 | Closing | `alt_closing` | CTA |

Rules specific to this page:

- You go first, and the page says why in one paragraph.
- Include at least one free or widely used option you don't compete with directly (a general office suite, for example). Leaving out the obvious choice makes the list look rigged.
- Your card carries "Where <Rival> is better". Reuse the `fit` text from the `/vs` page so the two pages never disagree.
- Put the year in one constant and update it each January together with a fact re-check.

### 5.4 Head-to-head page, section by section

| # | Section | `data-section` | Content |
|---|---|---|---|
| 1 | Hero | `compare_hero` | "Head to head, updated <Month Year>", H1 "<A> vs <B>", sub. No sign-up CTA here |
| 2 | Short answer | `compare_pick` | "Pick <A> if" and "Pick <B> if" cards |
| 3 | Side by side | `compare_table` | Shared rows from the product table plus pair-specific rows; sources footnote |
| 4 | Verdict by use | `compare_verdict` | Six use cases, each with a winner (or "Either works") and why |
| 5 | Or try <Brand> if | `compare_try` | One paragraph and a link to `/vs/<b>` |
| 6 | FAQ | `compare_faq` | "Is A better than B?", "Which is cheaper?", and so on |
| 7 | Keep exploring | `compare_more` | Both alternatives pages, related pairs, a `/vs` page, a use case |
| 8 | Closing | `compare_closing` | "Or make your next report free." with CTA |

The page is neutral until section 5. Your product appears only after the reader has their answer. At least one verdict should be "Either works" when that is the truth.

### 5.5 Minimal working code

The page template reuses `Value`, `faqLd`, `LinkCards` and a `Sources` footnote.

```tsx
// app/alternatives/sources.tsx
import { CHECKED, PRODUCTS, type ProductKey } from "./products";

export function Sources({ keys }: { keys: ProductKey[] }) {
  const list = keys.filter((k) => k !== "brand").map((k) => PRODUCTS[k]);
  return (
    <p className="text-xs">
      Details checked {CHECKED} from each company&apos;s own pages:{" "}
      {list.map((p, i) => (
        <span key={p.name}>
          {i > 0 && "; "}
          {p.name}{" "}
          {p.sources.map(([label, href], j) => (
            <span key={href}>{j > 0 && ", "}<a href={href} rel="nofollow noopener" target="_blank">{label}</a></span>
          ))}
        </span>
      ))}
      . Plans change, so check their sites.
    </p>
  );
}
```

```tsx
// app/compare/head-to-head.tsx
import { Cta } from "@/components/cta";
import { LandingTracking } from "@/components/landing-tracking";
import { LinkCards, type LinkCard } from "@/components/link-cards";
import { faqLd } from "@/lib/json-ld";
import { pageMetadata } from "@/lib/page-metadata";
import { alternativePath, HEAD_TO_HEAD, headToHeadPath, type HeadToHeadSlug } from "@/lib/registry/compare";
import { useCase } from "@/lib/registry/use-cases";
import { Sources } from "../alternatives/sources";
import { PRODUCTS } from "../alternatives/products";
import { Value } from "../vs/compare";
import { H2H } from "./data";

const pair = (slug: HeadToHeadSlug) => HEAD_TO_HEAD.find((c) => c.slug === slug)!;

export const h2hMetadata = (slug: HeadToHeadSlug) => pageMetadata(headToHeadPath(slug), H2H[slug].title, H2H[slug].description);

export function HeadToHeadPage({ slug }: { slug: HeadToHeadSlug }) {
  const { a, b, label } = pair(slug);
  const d = H2H[slug];
  const A = PRODUCTS[a];
  const B = PRODUCTS[b];
  const loc = `compare_${slug}`;
  const rows: [string, string, string][] = [
    ["Free plan", A.free, B.free],
    ["Paid from", A.paid, B.paid],
    ["Export", A.export, B.export],
    ["Working together", A.collab, B.collab],
    ["Stands out for", A.standout, B.standout],
  ];
  const links: LinkCard[] = [
    { href: alternativePath(a), label: `${A.name} alternatives`, blurb: `Other tools like ${A.name}.`, cta: "alternatives_link" },
    { href: alternativePath(b), label: `${B.name} alternatives`, blurb: `Other tools like ${B.name}.`, cta: "alternatives_link" },
    ...HEAD_TO_HEAD.filter((c) => c.slug !== slug && [c.a, c.b].some((x) => x === a || x === b)).slice(0, 2)
      .map((c) => ({ href: headToHeadPath(c.slug), label: c.label, blurb: c.blurb, cta: "compare_link" })),
    { href: `/for/${d.forPage}`, label: useCase(d.forPage).label, blurb: useCase(d.forPage).blurb, cta: "usecase_link" },
  ];

  return (
    <div>
      <LandingTracking />
      <script type="application/ld+json" dangerouslySetInnerHTML={{ __html: faqLd(d.faq) }} />

      <section data-section="compare_hero">
        <p>Head to head, updated October 2026</p>
        <h1>{label}</h1>
        <p>{d.sub}</p>
      </section>

      <section data-section="compare_pick">
        <ul>
          {([[A.name, d.pickA], [B.name, d.pickB]] as const).map(([name, body]) => (
            <li key={name}><h3>Pick {name} if</h3><p>{body}</p></li>
          ))}
        </ul>
      </section>

      <section data-section="compare_table">
        <h2>Side by side</h2>
        <table className="w-full table-fixed">
          <thead><tr><th scope="col"><span className="sr-only">Feature</span></th><th scope="col">{A.name}</th><th scope="col">{B.name}</th></tr></thead>
          <tbody>
            {[...rows, ...d.rows].map(([feature, x, y]) => (
              <tr key={feature}>
                <th scope="row" className="break-words">{feature}</th>
                <td className="break-words"><Value cell={x} /></td>
                <td className="break-words"><Value cell={y} /></td>
              </tr>
            ))}
          </tbody>
        </table>
        <Sources keys={[a, b]} />
      </section>

      <section data-section="compare_verdict">
        <h2>Verdict by use</h2>
        <ul>
          {d.verdicts.map(([use, winner, why]) => (
            <li key={use}>
              <p>{use}</p>
              <h3>{winner === "tie" ? "Either works" : PRODUCTS[winner === "a" ? a : b].name}</h3>
              <p>{why}</p>
            </li>
          ))}
        </ul>
      </section>

      <section data-section="compare_try">
        <h2>Or try &lt;Brand&gt; if</h2>
        <p>{d.tryUs}</p>
      </section>

      <section data-section="compare_faq">
        <h2>Questions</h2>
        <dl>{d.faq.map(([q, ans]) => <div key={q}><dt>{q}</dt><dd>{ans}</dd></div>)}</dl>
      </section>

      <nav data-section="compare_more"><LinkCards links={links} location={loc} /></nav>

      <section data-section="compare_closing">
        <h2>Or make your next report free.</h2>
        <Cta location={`${loc}_closing`} />
      </section>
    </div>
  );
}
```

The alternatives page follows the same structure with the sections in 5.3. Its numbered list maps `["brand", ...alt.picks.map((p) => p.key)]` to cards.

### 5.6 JSON-LD

Alternatives pages: an ItemList of SoftwareApplication entries in the order shown, plus FAQPage.

```ts
const keys: ProductKey[] = ["brand", ...alt.picks.map((p) => p.key)];
const itemList = ldJson({
  "@context": "https://schema.org",
  "@type": "ItemList",
  name: alt.title,
  itemListOrder: "https://schema.org/ItemListOrderAscending",
  numberOfItems: keys.length,
  itemListElement: keys.map((k, i) => ({
    "@type": "ListItem",
    position: i + 1,
    item: { "@type": "SoftwareApplication", name: PRODUCTS[k].name, url: PRODUCTS[k].url, applicationCategory: "BusinessApplication" },
  })),
});
```

Don't add ratings or review markup for rivals. You have no reviews to back them.

Head-to-head pages: FAQPage. Hubs: ItemList (section 9.3).

### 5.7 Quality rules

1. Same fact rules as `/vs`: their own pages, a checked month, "Not checked" when unverified.
2. Alternatives pages are honest about where the rival wins.
3. Head-to-head pages are neutral until "Or try <Brand> if".
4. Pairs exist in autocomplete in both orders, recorded with engine and month.
5. One product table feeds all three programs, so a fact changes in one place.
6. The registry test checks routes, distinct pairs and slug format.

---

## 6. Template gallery `/templates/<slug>`

### Why it ranks

"<thing> template" searches ("board update template", "weekly status report template") are huge, high-intent and evergreen. The visitor wants a finished example to start from, which your product can give in one click. A complete example rendered on the page also answers the query even for people who never sign up, which is why it ranks.

### URL pattern

`/templates` hub grouped by category, `/templates/<slug>` per template. The slug is the thing, not the word "template": `/templates/board-update`.

### 6.1 How to pick targets

1. Autocomplete "<job> template" and "<output> template" for every job your product does.
2. Check the results page: if it is full of galleries and downloadable files, a template page fits.
3. Prefer templates your product makes well and that real users already make (check your own usage data for common titles).
4. Start with 15 to 25 and make every one complete. A gallery of thin stubs is worse than ten real ones.

> Example from MakeDecks: 24 complete example decks across 5 categories, each with every slide rendered on its page.

### 6.2 Data shape

Templates live in a shared package or folder that both the web app and the API import, because the API copies the template into the user's account.

```ts
// lib/registry/templates/types.ts
import type { Report } from "@/lib/product-types"; // the same document type your app saves

export const TEMPLATE_CATEGORIES = [
  { id: "business", label: "Business reviews" },
  { id: "teams", label: "Teams and projects" },
] as const;
export type TemplateCategory = (typeof TEMPLATE_CATEGORIES)[number]["id"];

export type Template = {
  slug: string;
  name: string;            // "Board update"
  title: string;           // "Board update template (free, editable)"
  description: string;
  blurb: string;           // 8 words or fewer
  category: TemplateCategory;
  bestFor: string;         // hero sub line
  tips: string[];          // 4 to 6 practical tips for this kind of document
  faq: [string, string][];
  related: string[];       // other template slugs
  links: string[];         // "/for/..." or "/vs/..." paths
  content: Report;         // the full, valid document
};
```

```ts
// lib/registry/templates/index.ts
import boardUpdate from "./board-update";
import weeklyStatus from "./weekly-status-report";
import type { Template } from "./types";

export * from "./types";
export const TEMPLATE_LIMITS = { perDay: 30 } as const;
export const TEMPLATES: Template[] = [boardUpdate, weeklyStatus];
export const template = (slug: string) => TEMPLATES.find((t) => t.slug === slug);
```

Validate every template's `content` against your real document schema in a test, so a schema change can't break a template silently.

### 6.3 Page template, section by section

| # | Section | `data-section` | Content |
|---|---|---|---|
| 1 | Hero | `template_hero` | Breadcrumb link to Templates, H1 "<Name> template", `bestFor`, two buttons: "Use this template" and "Make one from your notes" |
| 2 | The full template | `template_deck` | Every page or block rendered with your real renderer; caption with the count and "Example, names are fictional." |
| 3 | Tips | `template_tips` | Numbered, practical advice for this document type |
| 4 | FAQ | `template_faq` | Including "Can I edit it in <format>?" |
| 5 | Related | `template_more` | Related templates, matching use-case and comparison pages, "All templates" |
| 6 | Closing | `template_closing` | "Make it yours." with "Use this template" again |

### 6.4 "Use this template" across sign-up

The flow:

1. Visitor clicks "Use this template". The client posts `{ slug }` to the copy endpoint.
2. **Signed in:** the server copies the template into the account and returns the new id. The client opens it.
3. **Signed out:** the server answers 401. The client sends the visitor to sign-up with `next=/templates/<slug>?use=1`.
4. After sign-up, the app sends them to `next`. The template page sees `?use=1`, removes it from the URL and runs the copy again, now signed in.

The client:

```tsx
// components/use-template.tsx
"use client";
import { useEffect, useState } from "react";
import { track } from "@/lib/analytics";

let busy = false;

async function copyTemplate(slug: string) {
  if (busy) return false;
  busy = true;
  try {
    const res = await fetch("/api/templates/use", { method: "POST", headers: { "content-type": "application/json" }, body: JSON.stringify({ slug }) });
    if (res.status === 401) {
      location.assign(`/signup?next=${encodeURIComponent(`/templates/${slug}?use=1`)}`);
      return true;
    }
    const body = (await res.json().catch(() => ({}))) as { id?: string; error?: string; code?: string };
    if (!res.ok || !body.id) throw Object.assign(new Error(body.error ?? "Couldn't copy the template. Try again in a moment."), { code: body.code ?? `HTTP_${res.status}` });
    track("template_used", { slug });
    location.assign(`/app/${body.id}`);
    return true;
  } catch (e) {
    const err = e as Error & { code?: string };
    console.error("template copy failed", err);
    track("template_use_failed", { code: err.code ?? "NETWORK" });
    alert(err.code ? err.message : "Network error, check your connection."); // use your toast component
    busy = false;
    return false;
  }
}

export function UseTemplateButton({ slug, location: where }: { slug: string; location: string }) {
  const [pending, setPending] = useState(false);
  return (
    <button
      type="button"
      disabled={pending}
      data-cta="use_template"
      data-cta-location={where}
      onClick={async () => {
        setPending(true);
        if (!(await copyTemplate(slug))) setPending(false);
      }}
    >
      {pending ? "Copying…" : "Use this template"}
    </button>
  );
}

// After sign-up the page comes back with ?use=1 and finishes the copy.
export function TemplateTracking({ slug }: { slug: string }) {
  useEffect(() => {
    track("template_viewed", { slug });
    if (new URLSearchParams(location.search).get("use") !== "1") return;
    history.replaceState(null, "", location.pathname);
    void copyTemplate(slug);
  }, [slug]);
  return null;
}
```

Reading `location.search` inside `useEffect` keeps the page static. `useSearchParams()` would opt the page out of static rendering unless you wrap it in Suspense.

The server, as a Next.js route handler. One SQL statement both checks the daily cap and inserts, so two quick clicks can't pass the cap together:

```ts
// app/api/templates/use/route.ts
import { z } from "zod";
import { getDb } from "@/lib/db";              // returns your D1 or SQLite handle
import { getUserId } from "@/lib/auth";        // your session lookup; null when signed out
import { template, TEMPLATE_LIMITS } from "@/lib/registry/templates";
import { reportSchema } from "@/lib/product-types";

const DAY_MS = 86_400_000;
const bodySchema = z.object({ slug: z.string().max(80) });

export async function POST(req: Request) {
  const userId = await getUserId(req);
  if (!userId) return Response.json({ error: "Sign in first.", code: "UNAUTHORIZED" }, { status: 401 });
  const parsed = bodySchema.safeParse(await req.json().catch(() => null));
  if (!parsed.success) return Response.json({ error: "Bad request.", code: "BAD_INPUT" }, { status: 400 });
  const t = template(parsed.data.slug);
  if (!t) return Response.json({ error: "Template not found.", code: "NOT_FOUND" }, { status: 404 });

  const content = reportSchema.parse(t.content);
  const id = crypto.randomUUID();
  const now = Date.now();
  const { meta } = await getDb()
    .prepare(
      `insert into report (id, userId, title, data, createdAt, updatedAt, fromTemplate)
       select ?, ?, ?, ?, ?, ?, ?
       where (select count(*) from report where userId = ? and fromTemplate is not null and createdAt > ?) < ?`,
    )
    .bind(id, userId, content.title, JSON.stringify(content), now, now, t.slug, userId, now - DAY_MS, TEMPLATE_LIMITS.perDay)
    .run();
  if (!meta.changes) {
    return Response.json({ error: `You've copied ${TEMPLATE_LIMITS.perDay} templates today. Try again tomorrow.`, code: "TEMPLATE_DAILY_LIMIT" }, { status: 429 });
  }
  return Response.json({ id });
}
```

Decisions worth copying:

- A template copy runs no AI, so it doesn't count toward the plan's usage cap. It has its own daily cap to stop scripted abuse.
- The copy is marked `fromTemplate` so you can measure template-driven sign-ups and exclude copies from "made with AI" stats.
- **Your sign-up page must only follow a `next` that is a same-site path.** Accept it only if it starts with `/` and not `//`. Otherwise `next` becomes an open redirect.

### 6.5 JSON-LD

A graph with the document type that fits your product (`DigitalDocument`, `PresentationDigitalDocument`, `SpreadsheetDigitalDocument`, or `CreativeWork`), FAQPage and BreadcrumbList:

```ts
const url = `${SITE_URL}/templates/${t.slug}`;
const ld = ldJson({
  "@context": "https://schema.org",
  "@graph": [
    { "@type": "DigitalDocument", name: `${t.name} template`, headline: t.title, description: t.description, url, isAccessibleForFree: true, inLanguage: "en", publisher: { "@id": `${SITE_URL}/#organization` } },
    { "@type": "FAQPage", mainEntity: t.faq.map(([q, a]) => ({ "@type": "Question", name: q, acceptedAnswer: { "@type": "Answer", text: a } })) },
    {
      "@type": "BreadcrumbList",
      itemListElement: [
        { "@type": "ListItem", position: 1, name: "<Brand>", item: SITE_URL },
        { "@type": "ListItem", position: 2, name: "Templates", item: `${SITE_URL}/templates` },
        { "@type": "ListItem", position: 3, name: t.name, item: url },
      ],
    },
  ],
});
```

The hub `/templates` uses a CollectionPage whose `mainEntity` is an ItemList of every template.

### 6.6 Internal linking

- Each template links to two or three related templates, its matching use-case page and, where it fits, a comparison page.
- The matching use-case page links back.
- The hub renders the first page of each template as its thumbnail (load the first row eagerly, defer the rest).

### 6.7 Quality rules

1. Every template is complete and realistic, with fictional names, and validates against your real schema.
2. Every template is rendered in full on its page, not as a blurred preview.
3. Tips are specific to that document type.
4. "Use this template" works for signed-out visitors and lands them inside the copied template after sign-up.
5. Copies are rate limited in one SQL statement.

---

## 7. Free tools `/tools/<slug>`

### Why they rank

Small, single-purpose tools match "calculator", "generator", "converter", "checker" searches. They earn links on their own (people link to a useful calculator, rarely to a product page), they pull top-of-funnel visitors who will need your main product later, and with an email gate they build a lead list.

### URL pattern

`/tools` hub grouped by the job (plan, write, fix, design), `/tools/<slug>` per tool. The slug repeats the query: `/tools/reading-time-calculator`.

### 7.1 How to pick targets

1. List the small jobs around your main product: planning before, fixing files after, measuring, formatting.
2. Autocomplete each with "calculator", "generator", "checker", "converter", "counter", "free".
3. Keep tools you can build in a day or two and that run with no server cost (browser tools), or with a small, bounded AI call (AI tools).
4. Prefer tools whose output leads naturally into your product ("now make the whole report").

> Example from MakeDecks: 10 tools, 6 browser-only (speech time, slide count, file compressor, word counter, font pairing, colour palette) and 4 AI (outline, pitch deck outline, speaker notes, titles).

### 7.2 Two kinds of tools

| | Browser-only tools | AI tools |
|---|---|---|
| Runs | Entirely in the visitor's browser | Calls your public API, which calls a model |
| Cost | None | One model call per use |
| Privacy | Input never leaves the device. Say so on the page | Input goes to your API and model provider. Say so |
| Abuse controls | None needed | Per-IP rate, per-IP daily cap, global daily cap, input size limits |
| Email gate | Client-side only (a soft gate) | Enforced on the server before any model call |

Build browser-only tools first. They are cheaper, faster and easier to trust.

### 7.3 Registry and schemas

```ts
// lib/registry/tools.ts
import { z } from "zod";

export const TOOLS = [
  { slug: "reading-time-calculator", kind: "browser", group: "plan", name: "Reading time calculator", blurb: "How long your text takes to read." },
  { slug: "report-outline-generator", kind: "ai", group: "write", name: "Report outline generator", blurb: "A section-by-section outline from a topic." },
  { slug: "report-title-generator", kind: "ai", group: "write", name: "Report title generator", blurb: "Titles that make people read." },
  { slug: "pdf-compressor", kind: "browser", group: "files", name: "PDF compressor", blurb: "Shrink a file without losing pages." },
] as const;

export type ToolSlug = (typeof TOOLS)[number]["slug"];
export type AiToolSlug = Extract<(typeof TOOLS)[number], { kind: "ai" }>["slug"];
export const TOOL_GROUPS = [
  { id: "plan", label: "Plan it" },
  { id: "write", label: "Write it" },
  { id: "files", label: "Fix your file" },
] as const;
export const tool = (slug: ToolSlug) => TOOLS.find((t) => t.slug === slug)!;
export const AI_TOOLS = TOOLS.filter((t) => t.kind === "ai").map((t) => t.slug) as AiToolSlug[];

export const TOOL_LIMITS = { perIpDay: 20, globalDay: 1500, inputChars: 6000 } as const;

export const toolLeadSchema = z.object({
  email: z.string().trim().toLowerCase().email().max(200),
  tool: z.enum(TOOLS.map((t) => t.slug) as [ToolSlug, ...ToolSlug[]]),
  website: z.string().max(0).optional(), // honeypot: real users leave it empty
});

const text = (max: number) => z.string().trim().min(3).max(max);
export const toolRequestSchemas = {
  "report-outline-generator": z.object({ topic: text(TOOL_LIMITS.inputChars), audience: z.string().trim().max(200).optional(), sections: z.number().int().min(3).max(20).optional() }),
  "report-title-generator": z.object({ topic: text(1000), tone: z.enum(["clear", "bold", "formal"]).optional() }),
} satisfies Record<AiToolSlug, z.ZodType>;
```

The `satisfies Record<AiToolSlug, ...>` line makes the build fail when you add an AI tool without an input schema.

### 7.4 Database tables

```sql
create table tool_lead (
  id integer primary key autoincrement,
  email text not null,
  tool text not null,
  ipHash text,
  createdAt integer not null
);
create index tool_lead_email_idx on tool_lead (email);
create index tool_lead_createdAt_idx on tool_lead (createdAt);

create table tool_use (
  id integer primary key autoincrement,
  tool text not null,
  ipHash text not null,
  leadEmail text,
  createdAt integer not null,
  failedAt integer
);
create index tool_use_ipHash_createdAt_idx on tool_use (ipHash, createdAt);
create index tool_use_createdAt_idx on tool_use (createdAt);
```

IP addresses are never stored, only an HMAC of them. You can still count uses per visitor without keeping personal data you don't need.

### 7.5 Abuse limits: one insert that enforces both caps

The order of checks in the AI tool endpoint:

1. Validate the slug and the input. Reject bad input before anything else.
2. **Email gate:** a valid signed lead cookie or a session, else 403 `TOOL_EMAIL_REQUIRED`. Use 403, not 401, so your app's "signed out, go to login" handling doesn't fire.
3. **Per-minute rate limit** per IP hash, if your platform has one (Cloudflare's Rate Limiting binding, Upstash, or similar).
4. **Claim a use** with one insert that counts today's successful uses per IP and site-wide. If either is at its cap, the insert writes nothing and returns no row.
5. Call the model. **If the call fails, mark the claim `failedAt`,** so a failure doesn't use up the visitor's quota.

The claim, in SQLite:

```sql
insert into tool_use (tool, ipHash, leadEmail, createdAt)
select ?1, ?2, ?3, ?4
where (select count(*) from tool_use where ipHash = ?2 and createdAt >= ?5 and failedAt is null) < ?6
  and (select count(*) from tool_use where createdAt >= ?5 and failedAt is null) < ?7
returning id
```

| Param | Value |
|---|---|
| `?1` | tool slug |
| `?2` | IP hash |
| `?3` | lead email or the signed-in user's email |
| `?4` | now (ms) |
| `?5` | start of the current UTC day: `now - (now % 86_400_000)` |
| `?6` | `TOOL_LIMITS.perIpDay` |
| `?7` | `TOOL_LIMITS.globalDay` |

Why one statement: a check-then-insert in two queries lets ten parallel requests all pass the check before any of them inserts. In SQLite and D1 a single statement runs atomically. In Postgres, take an advisory lock on the IP hash or use a counter row with `update ... where count < cap returning`.

Why a global cap: a botnet with many IPs passes every per-IP cap. The global daily cap is what bounds your bill. Set it to a number whose cost you can absorb, and alert when you hit it.

### 7.6 The AI tool endpoint

Adapted to a Next.js route handler. `runModel` is your provider call (with a timeout, no retries, structured output) and `recordUsage` is your spend log; both are yours to write.

```ts
// app/api/tools/[slug]/route.ts
import { captureException } from "@sentry/nextjs";
import { cookies } from "next/headers";
import { getDb } from "@/lib/db";
import { getSession } from "@/lib/auth";
import { AI_TOOLS, TOOL_LIMITS, toolRequestSchemas, type AiToolSlug } from "@/lib/registry/tools";
import { ipHash, leadEmail, LEAD_COOKIE, sameOrigin } from "@/lib/tool-auth";
import { runTool } from "@/lib/tool-run";

const DAY_MS = 86_400_000;
const fail = (status: number, error: string, code: string) => Response.json({ error, code }, { status });

export async function POST(req: Request, { params }: { params: Promise<{ slug: string }> }) {
  if (!sameOrigin(req)) return fail(403, "Forbidden.", "CROSS_SITE");
  const slug = (await params).slug as AiToolSlug;
  if (!AI_TOOLS.includes(slug)) return fail(404, "Tool not found.", "NOT_FOUND");

  const input = toolRequestSchemas[slug].safeParse(await req.json().catch(() => null));
  if (!input.success) return fail(400, "Check what you entered and try again.", "BAD_INPUT");

  const [email, session] = await Promise.all([leadEmail((await cookies()).get(LEAD_COOKIE)?.value), getSession(req)]);
  if (!email && !session) return fail(403, "Enter your email to see your result.", "TOOL_EMAIL_REQUIRED");

  const ip = req.headers.get("cf-connecting-ip") ?? req.headers.get("x-forwarded-for")?.split(",")[0]?.trim() ?? "local";
  const hash = await ipHash(ip);
  const now = Date.now();
  const db = getDb();
  const claim = await db
    .prepare(
      `insert into tool_use (tool, ipHash, leadEmail, createdAt) select ?1, ?2, ?3, ?4
       where (select count(*) from tool_use where ipHash = ?2 and createdAt >= ?5 and failedAt is null) < ?6
         and (select count(*) from tool_use where createdAt >= ?5 and failedAt is null) < ?7
       returning id`,
    )
    .bind(slug, hash, email ?? session?.email.toLowerCase() ?? null, now, now - (now % DAY_MS), TOOL_LIMITS.perIpDay, TOOL_LIMITS.globalDay)
    .first<{ id: number }>();
  if (!claim) return fail(429, "You've used today's free tool limit. Come back tomorrow, or make a free account to keep going.", "TOOL_LIMIT");

  try {
    return Response.json(await runTool(slug, input.data, session?.userId ?? null));
  } catch (e) {
    await db.prepare("update tool_use set failedAt = ? where id = ?").bind(Date.now(), claim.id).run();
    console.error("tool failed", slug);
    captureException(e, { tags: { tool: slug } });
    return fail(502, "That didn't work. Try again in a moment.", "TOOL_FAILED");
  }
}
```

Which header holds the real client IP depends on your host: `cf-connecting-ip` on Cloudflare, the first `x-forwarded-for` entry behind most proxies. Make sure a visitor can't set it themselves.

Prompt rules that keep AI tool output safe to publish:

- Never invent statistics, names, customers or quotes. Where a number would help, write a bracketed placeholder like `[your figure]`.
- Clean the output in code: strip characters you don't want (em dashes, stray quotes), drop empty items, dedupe, enforce length limits. Treat an empty result as a failure so the claim is released.
- Pick the cheapest model that does the job, with a short timeout and no retries.
- Record the spend of every call, with a null user for anonymous use.

### 7.7 The email gate

The server half is a signed cookie, set when the visitor submits an email. HMAC with a server secret means nobody can forge "I already gave an email".

```ts
// lib/tool-auth.ts
const COOKIE_SECRET = process.env.TOOL_COOKIE_SECRET!; // a long random value, server-only
export const LEAD_COOKIE = "tool_lead";

async function hmac(data: string) {
  const key = await crypto.subtle.importKey("raw", new TextEncoder().encode(COOKIE_SECRET), { name: "HMAC", hash: "SHA-256" }, false, ["sign"]);
  const sig = await crypto.subtle.sign("HMAC", key, new TextEncoder().encode(data));
  return [...new Uint8Array(sig)].map((b) => b.toString(16).padStart(2, "0")).join("");
}

const b64url = (s: string) => btoa(String.fromCharCode(...new TextEncoder().encode(s))).replace(/\+/g, "-").replace(/\//g, "_").replace(/=+$/, "");
const unb64url = (s: string) => new TextDecoder().decode(Uint8Array.from(atob(s.replace(/-/g, "+").replace(/_/g, "/")), (ch) => ch.charCodeAt(0)));

export const ipHash = async (ip: string) => (await hmac(`ip:${ip}`)).slice(0, 32);

export async function leadCookie(email: string, secure: boolean) {
  return `${LEAD_COOKIE}=${b64url(email)}.${await hmac(`toollead:${email}`)}; Path=/; Max-Age=31536000; HttpOnly; SameSite=Lax${secure ? "; Secure" : ""}`;
}

export async function leadEmail(raw: string | undefined) {
  const [data, sig] = raw?.split(".") ?? [];
  if (!data || !sig) return null;
  try {
    const email = unb64url(data);
    return sig === (await hmac(`toollead:${email}`)) ? email : null;
  } catch {
    return null;
  }
}

// Public POST endpoints refuse requests from other sites.
export function sameOrigin(req: Request) {
  const origin = req.headers.get("origin");
  return !origin || origin === new URL(process.env.SITE_URL!).origin;
}
```

For extra hardening, compare signatures with a constant-time check (`crypto.subtle.verify` does this for you).

The lead endpoint:

```ts
// app/api/tools/lead/route.ts
import { getDb } from "@/lib/db";
import { toolLeadSchema } from "@/lib/registry/tools";
import { ipHash, leadCookie, sameOrigin } from "@/lib/tool-auth";

export async function POST(req: Request) {
  if (!sameOrigin(req)) return Response.json({ error: "Forbidden.", code: "CROSS_SITE" }, { status: 403 });
  // also apply a per-minute rate limit per IP hash here
  const raw = (await req.json().catch(() => null)) as { website?: unknown } | null;
  if (raw?.website) return Response.json({ ok: true }); // honeypot filled: pretend success, store nothing
  const parsed = toolLeadSchema.safeParse(raw);
  if (!parsed.success) {
    const bad = parsed.error.issues.some((i) => i.path[0] === "email");
    return Response.json({ error: bad ? "Enter a valid email." : "Pick a tool.", code: bad ? "BAD_EMAIL" : "BAD_INPUT" }, { status: 400 });
  }
  const { email, tool } = parsed.data;
  const ip = req.headers.get("cf-connecting-ip") ?? "local";
  await getDb().prepare("insert into tool_lead (email, tool, ipHash, createdAt) values (?, ?, ?, ?)").bind(email, tool, await ipHash(ip), Date.now()).run();
  return Response.json({ ok: true }, { headers: { "Set-Cookie": await leadCookie(email, req.url.startsWith("https")), "Cache-Control": "no-store" } });
}
```

The client half is a hook. It shows the gate once per browser (a `localStorage` flag), skips it for signed-in users, and if the server still says `TOOL_EMAIL_REQUIRED` (the cookie expired or was cleared), it asks again and retries once.

```tsx
// components/tool-gate.tsx
"use client";
import { useCallback, useRef, useState } from "react";
import { track } from "@/lib/analytics";
import type { ToolSlug } from "@/lib/registry/tools";

const KEY = "tool-email";
const hasToolEmail = () => {
  try {
    return !!localStorage.getItem(KEY);
  } catch {
    return false;
  }
};

export function useToolGate(slug: ToolSlug, signedIn: boolean) {
  const [open, setOpen] = useState(false);
  const [email, setEmail] = useState("");
  const [website, setWebsite] = useState("");
  const [error, setError] = useState<string>();
  const waiting = useRef<((ok: boolean) => void) | null>(null);

  const requireEmail = useCallback((force = false) => {
    if (!force && (signedIn || hasToolEmail())) return Promise.resolve(true);
    track("tool_gate_shown", { tool: slug });
    setOpen(true);
    return new Promise<boolean>((resolve) => { waiting.current = resolve; });
  }, [signedIn, slug]);

  const close = (ok: boolean) => {
    setOpen(false);
    waiting.current?.(ok);
    waiting.current = null;
  };

  const submit = async (e: React.FormEvent) => {
    e.preventDefault();
    const res = await fetch("/api/tools/lead", { method: "POST", headers: { "content-type": "application/json" }, body: JSON.stringify({ email, tool: slug, website }) });
    if (!res.ok) {
      setError(((await res.json().catch(() => ({}))) as { error?: string }).error ?? "Try again.");
      track("tool_failed", { tool: slug, reason: `email_${res.status}` });
      return;
    }
    try { localStorage.setItem(KEY, "1"); } catch {}
    track("tool_email_captured", { tool: slug });
    close(true);
  };

  const gate = open && (
    <dialog open aria-labelledby="gate-title">
      <h2 id="gate-title">See your result</h2>
      <p>Enter your email once to use every free tool.</p>
      <form onSubmit={submit}>
        <input type="email" required autoComplete="email" aria-label="Email" value={email} onChange={(e) => setEmail(e.target.value)} />
        <input type="text" tabIndex={-1} autoComplete="off" aria-hidden value={website} onChange={(e) => setWebsite(e.target.value)} style={{ position: "absolute", left: -9999, height: 0, width: 0, opacity: 0 }} />
        {error && <p role="alert">{error}</p>}
        <button type="submit">Show my result</button>
        <p>We may email you about &lt;Brand&gt;. Unsubscribe anytime.</p>
        <button type="button" onClick={() => close(false)}>Cancel</button>
      </form>
    </dialog>
  );

  return { requireEmail, gate };
}
```

Using it in an AI tool:

```tsx
const run = async (input: unknown) => {
  if (!(await requireEmail())) return;
  let res = await fetch(`/api/tools/${slug}`, { method: "POST", headers: { "content-type": "application/json" }, body: JSON.stringify(input) });
  if (res.status === 403 && ((await res.clone().json()) as { code?: string }).code === "TOOL_EMAIL_REQUIRED") {
    if (!(await requireEmail(true))) return;
    res = await fetch(`/api/tools/${slug}`, { method: "POST", headers: { "content-type": "application/json" }, body: JSON.stringify(input) });
  }
  // handle 429 TOOL_LIMIT with a "make a free account" message, other errors with a retry button
};
```

Gate rules:

- The tool UI, the copy and the FAQ stay visible without an email. Only the result is gated. Crawlers see a full page.
- Ask once per browser, for every tool.
- Say in the gate what you'll do with the email, and keep a "Free tools" section on your privacy page: what is stored (email, which tool, an IP hash, timestamps), what is sent to the model provider, and that browser-only tools send nothing.
- Measure `tool_gate_shown` against `tool_email_captured`. If most people leave at the gate, consider gating only AI tools.

### 7.8 Page template, section by section

| # | Section | `data-section` | Content |
|---|---|---|---|
| 1 | Hero | `tool_hero` | Breadcrumb "Free tools", H1 (the tool name, matching the query), one-line sub |
| 2 | The tool | `tool_app` | The working tool, immediately usable |
| 3 | FAQ | `tool_faq` | Real questions about the topic ("How many words a minute do people read?"), plus "Is my text stored?" |
| 4 | More free tools | `tool_more` | Three related tools and "All free tools" |
| 5 | Closing | `tool_closing` | "Now make the whole report." with the main CTA (`tool_<slug>_closing`) |

The FAQ is where a tool page earns its ranking: answer the questions people search around the tool, with real numbers and ranges.

```tsx
// components/tool-page.tsx
import Link from "next/link";
import { Cta } from "@/components/cta";
import { LandingTracking } from "@/components/landing-tracking";
import { ldJson } from "@/lib/json-ld";
import { pageMetadata } from "@/lib/page-metadata";
import { tool, type ToolSlug } from "@/lib/registry/tools";
import { SITE_URL } from "@/lib/site";

export type ToolCopy = { title: string; description: string; h1: string; sub: string; faq: [string, string][]; related: ToolSlug[]; closing?: string };

export const toolMetadata = (slug: ToolSlug, copy: Pick<ToolCopy, "title" | "description">) => pageMetadata(`/tools/${slug}`, copy.title, copy.description);

export function ToolPage({ slug, copy, children }: { slug: ToolSlug; copy: ToolCopy; children: React.ReactNode }) {
  const t = tool(slug);
  const loc = `tool_${slug}`;
  const ld = ldJson({
    "@context": "https://schema.org",
    "@graph": [
      { "@type": "WebApplication", name: t.name, url: `${SITE_URL}/tools/${slug}`, applicationCategory: "BusinessApplication", operatingSystem: "Web", isAccessibleForFree: true, offers: { "@type": "Offer", price: "0", priceCurrency: "USD" }, publisher: { "@id": `${SITE_URL}/#organization` } },
      { "@type": "FAQPage", mainEntity: copy.faq.map(([q, a]) => ({ "@type": "Question", name: q, acceptedAnswer: { "@type": "Answer", text: a } })) },
    ],
  });
  return (
    <div>
      <LandingTracking />
      <script type="application/ld+json" dangerouslySetInnerHTML={{ __html: ld }} />
      <section data-section="tool_hero">
        <p><Link href="/tools" data-cta="tools_link" data-cta-location={loc}>Free tools</Link></p>
        <h1>{copy.h1}</h1>
        <p>{copy.sub}</p>
      </section>
      <section data-section="tool_app" aria-label={t.name}>{children}</section>
      <section data-section="tool_faq">
        <h2>Questions</h2>
        <dl>{copy.faq.map(([q, a]) => <div key={q}><dt>{q}</dt><dd>{a}</dd></div>)}</dl>
      </section>
      <nav data-section="tool_more">
        <h2>More free tools</h2>
        <ul>
          {[...copy.related.map(tool).map((r) => ({ href: `/tools/${r.slug}`, label: r.name })), { href: "/tools", label: "All free tools" }].map((l) => (
            <li key={l.href}><Link href={l.href} data-cta="tool_link" data-cta-location={loc}>{l.label}</Link></li>
          ))}
        </ul>
      </nav>
      <section data-section="tool_closing">
        <h2>{copy.closing ?? "Now make the whole report."}</h2>
        <Cta location={`${loc}_closing`} />
      </section>
    </div>
  );
}
```

```tsx
// app/tools/reading-time-calculator/page.tsx
import { ToolPage, toolMetadata, type ToolCopy } from "@/components/tool-page";
import { ReadingTime } from "@/components/tools/reading-time";

const copy: ToolCopy = {
  title: "Reading time calculator: words to minutes",
  description: "Paste your text or enter a word count to see how long it takes to read at a slow, average or fast pace. Free, and your text never leaves your browser.",
  h1: "Reading time calculator",
  sub: "See how long your text takes to read.",
  faq: [
    ["How many words a minute do people read?", "<A sourced range, with the source named.>"],
    ["Is my text stored anywhere?", "No. The count runs in your browser and your text isn't sent to our servers."],
  ],
  related: ["report-outline-generator", "report-title-generator", "pdf-compressor"],
  closing: "Turn your text into a report.",
};

export const metadata = toolMetadata("reading-time-calculator", copy);

export default function Page() {
  return <ToolPage slug="reading-time-calculator" copy={copy}><ReadingTime /></ToolPage>;
}
```

### 7.9 Events

`tool_gate_shown`, `tool_email_captured`, `tool_used`, `tool_failed { reason }`, `tool_limit_reached`, `tool_result_copied`, `tool_result_downloaded`, and a `tool_deck_started` style event when the visitor carries the result into your main product. Add `tool_failed` and `tool_limit_reached` to your user-error alert.

### 7.10 Quality rules

1. The tool works on first load, without an account, at 360 px wide.
2. Browser tools say "your file never leaves your browser", and that is true (check the network tab).
3. AI tools are bounded by per-IP rate, per-IP daily cap, global daily cap and input size, in that order, before any model call.
4. Failed calls don't count against the visitor.
5. The FAQ answers real searches around the tool, with sources for any numbers.
6. A lead list view for you (emails, tools used, first and last seen, whether they have an account), with a CSV export that neutralises spreadsheet formulas (prefix cells starting with `=`, `+`, `-` or `@` with a quote).

---

## 8. Embeddable widget `/embed/<tool>`

### Why it works

A widget other sites can paste into their pages carries a visible credit link back to you. Every embed is a backlink from a page about the same topic, which is the kind of link that helps rankings. A simple, useful browser tool (a timer, a calculator, a counter) is the best candidate.

### URL pattern

`/embed/<tool>`: a bare version of one tool with no header, footer or cookie banner. It is the **only** path on your site that other sites may frame.

### 8.1 Headers: framing allowed only under `/embed`

Your normal security headers deny framing everywhere (`frame-ancestors 'none'` and `X-Frame-Options: DENY`). Override them for `/embed/*` only:

```ts
// next.config.ts
import type { NextConfig } from "next";

const csp = [
  "default-src 'self'",
  "script-src 'self' 'unsafe-inline'",
  "style-src 'self' 'unsafe-inline'",
  "img-src 'self' data: blob:",
  "frame-ancestors 'none'",
  "object-src 'none'",
  "base-uri 'self'",
  "form-action 'self'",
].join("; ");

const embedCsp = csp.replace("frame-ancestors 'none'", "frame-ancestors *");

const securityHeaders = [
  { key: "Content-Security-Policy", value: csp },
  { key: "X-Frame-Options", value: "DENY" },
  { key: "X-Content-Type-Options", value: "nosniff" },
  { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
];

const nextConfig: NextConfig = {
  async headers() {
    return [
      ...["/", "/:path*"].map((source) => ({ source, headers: securityHeaders })),
      // Last match wins per header, so this overrides the rule above for /embed/*.
      // Browsers ignore X-Frame-Options when CSP sets frame-ancestors; the invalid value just replaces DENY.
      {
        source: "/embed/:path*",
        headers: [
          { key: "Content-Security-Policy", value: embedCsp },
          { key: "X-Frame-Options", value: "ALLOWALL" },
          { key: "X-Robots-Tag", value: "noindex" },
        ],
      },
    ];
  },
};

export default nextConfig;
```

Check it after deploy: `curl -sI https://example.com/embed/<tool>` shows `frame-ancestors *`, and `curl -sI https://example.com/pricing` still shows `frame-ancestors 'none'`.

Keep the embed free of anything that needs a session: no sign-in, no cookies that matter, no forms that post account data. Framing is only safe because there is nothing to clickjack.

### 8.2 The embed page

```tsx
// app/embed/reading-timer/page.tsx
import type { Metadata } from "next";
import { ReadingTimerEmbed } from "@/components/tools/reading-timer-embed";

export const metadata: Metadata = {
  title: "Reading timer",
  description: "See how long your text takes to read.",
  robots: { index: false, follow: true },
  alternates: { canonical: null },
};

export default function Page() {
  return <ReadingTimerEmbed />;
}
```

Noindex because the full tool page is the one that should rank; the embed is a stripped copy. `follow: true` lets crawlers follow the credit link. Hide your site header and footer on `/embed/*`: either put embeds in a route group with their own root layout, or have the shared layout skip its chrome when the path starts with `/embed`.

### 8.3 Credit link inside and outside the iframe

A link inside an iframe belongs to your page, not to the embedding page, so it is not a backlink. The snippet you offer therefore repeats the credit as a normal link **after** the iframe:

```tsx
// components/tools/reading-timer-embed.tsx (excerpt)
"use client";
import { useEffect, useState } from "react";
import { track } from "@/lib/analytics";

export const EMBED_CREDIT = "https://example.com/tools/reading-time-calculator?ref=embed";

export function ReadingTimerEmbed() {
  useEffect(() => {
    let host = "direct";
    try {
      if (document.referrer) host = new URL(document.referrer).hostname;
    } catch (e) {
      console.warn("embed referrer unreadable", e);
    }
    track("embed_viewed", { tool: "reading-time-calculator", host });
  }, []);

  return (
    <main>
      {/* the tool itself */}
      <p className="text-right text-xs">
        <a href={EMBED_CREDIT} target="_blank" rel="noopener" onClick={() => track("landing_cta_clicked", { cta: "embed_credit", location: "embed" })}>
          Reading timer by &lt;Brand&gt;
        </a>
      </p>
    </main>
  );
}

const SNIPPET = `<iframe src="https://example.com/embed/reading-timer" title="Reading timer" width="100%" height="420" style="border:0;max-width:640px" loading="lazy"></iframe>
<p><a href="${EMBED_CREDIT}">Reading timer by <Brand></a></p>`;

export function EmbedSnippet() {
  const [copied, setCopied] = useState(false);
  return (
    <section data-section="tool_embed" aria-labelledby="embed">
      <h2 id="embed">Embed this timer</h2>
      <p>Free to add to your site.</p>
      <textarea readOnly value={SNIPPET} aria-label="Embed code" onFocus={(e) => e.currentTarget.select()} />
      <button
        type="button"
        onClick={() =>
          navigator.clipboard.writeText(SNIPPET).then(
            () => { setCopied(true); track("embed_snippet_copied", { tool: "reading-time-calculator" }); },
            (e) => { console.error("copy failed", e); track("tool_failed", { tool: "reading-time-calculator", reason: "copy" }); },
          )
        }
      >
        {copied ? "Copied" : "Copy code"}
      </button>
    </section>
  );
}
```

Put `<EmbedSnippet />` under the tool on its normal tool page. The `embed_viewed` event with the referrer host tells you which sites embed you; check them now and then.

### 8.4 Link rules

Google's link spam policies name widgets that carry keyword-rich or hidden links. Stay on the safe side:

- The credit uses your brand and the tool name ("Reading timer by <Brand>"), not a keyword phrase.
- It is visible, in plain text, and the embedder can see and edit it in the snippet.
- One credit link, pointing at the tool page, not the home page or a money page.
- Never make the widget stop working when the credit is removed.

### 8.5 Quality rules

1. Only `/embed/*` may be framed. Every other page still denies it.
2. The embed is noindex, follow.
3. The embed works at any width from 300 px up and has no site chrome.
4. The snippet includes the credit link outside the iframe.
5. `embed_viewed` records the embedding host.

---

## 9. Trust pages, hubs and the footer

### Why they matter

Search engines and AI assistants weigh who is behind a site. A real About page, a Contact page that reaches a person, and a founder page with links to real profiles show experience and accountability (Google calls this E-E-A-T). Hubs and the footer give every programmatic page internal links from every other page, which is how crawlers find them and how link value spreads.

### 9.1 `/about` and `/contact`

Keep the content in one data file. The page renders it, and the same data becomes Markdown for AI agents that ask for `text/markdown`.

```ts
// lib/trust.ts
export const PUBLIC_EMAIL = "hello@example.com";

export type TrustPage = { path: string; title: string; description: string; sections: [string, string][] };

export const ABOUT: TrustPage = {
  path: "/about",
  title: "About <Brand>",
  description: "Who builds <Brand>, what it does and how it works.",
  sections: [
    ["What <Brand> is", "<What it does, which inputs it takes, what users get, in plain words.>"],
    ["Why it exists", "<The problem, and what the free plan includes. Use your plan constants.>"],
    ["Who builds it", "<Founder name, background, when it started. Link the founder page.>"],
    ["How your data is handled", "<What is stored, what is processed and deleted, and a link to the privacy page.>"],
  ],
};

export const CONTACT: TrustPage = {
  path: "/contact",
  title: "Contact <Brand>",
  description: "How to reach the <Brand> team for support, billing, privacy or press.",
  sections: [
    ["Email", `Write to ${PUBLIC_EMAIL} for anything. It goes straight to <who reads it>.`],
    ["Feedback from inside the app", "<Where the in-app feedback form is.>"],
    ["Billing", "<Who processes payments and how to manage a subscription.>"],
    ["Privacy and your data", "<How to request deletion or export.>"],
  ],
};

export const toMarkdown = (page: TrustPage, site: string) =>
  [`# ${page.title}`, "", page.description, "", ...page.sections.flatMap(([h, body]) => [`## ${h}`, "", body, ""]), `[<Brand>](${site}/) · [Privacy](${site}/privacy) · [llms.txt](${site}/llms.txt)`, ""].join("\n");
```

```tsx
// app/about/page.tsx
import type { Metadata } from "next";
import Link from "next/link";
import { LandingTracking } from "@/components/landing-tracking";
import { ABOUT, CONTACT } from "@/lib/trust";

export const metadata: Metadata = { title: ABOUT.title, description: ABOUT.description, alternates: { canonical: ABOUT.path } };

export default function About() {
  return (
    <div className="mx-auto max-w-2xl space-y-8">
      <LandingTracking />
      <h1>{ABOUT.title}</h1>
      {ABOUT.sections.map(([title, body]) => (
        <section key={title}>
          <h2>{title}</h2>
          <p>{body}</p>
        </section>
      ))}
      <p>
        <Link href="/about/<founder-slug>" data-cta="author_link" data-cta-location="about">Founder</Link>
        {" · "}
        <Link href={CONTACT.path} data-cta="contact_link" data-cta-location="about">Contact</Link>
        {" · "}
        <Link href="/privacy" data-cta="privacy_link" data-cta-location="about">Privacy</Link>
      </p>
    </div>
  );
}
```

Add a test that each trust page has at least 500 characters of section text. Thin trust pages don't count for much.

Also give the home page an Organization in JSON-LD with `email` and a `contactPoint`:

```ts
{
  "@type": "Organization",
  "@id": `${SITE_URL}/#organization`,
  name: "<Brand>",
  url: SITE_URL,
  logo: `${SITE_URL}/icon-512.png`,
  email: PUBLIC_EMAIL,
  contactPoint: { "@type": "ContactPoint", contactType: "customer support", email: PUBLIC_EMAIL, url: `${SITE_URL}/contact`, availableLanguage: ["English"] },
  sameAs: ["<your company's real profiles>"],
}
```

Leave out `address` until you have a real one you're willing to publish.

### 9.2 `/about/<founder>`

A ProfilePage whose main entity is a Person with `sameAs` links to real profiles. Every blog post's Article `author` points to this page's URL.

```tsx
// app/about/<founder-slug>/page.tsx
import type { Metadata } from "next";
import { ldJson } from "@/lib/json-ld";
import { SITE_URL } from "@/lib/site";

const PAGE_URL = `${SITE_URL}/about/<founder-slug>`;
const NAME = "<Founder Name>";
const LINKS: [string, string][] = [
  ["Website", "https://<founder-site>.example"],
  ["LinkedIn", "https://www.linkedin.com/in/<handle>"],
  ["GitHub", "https://github.com/<handle>"],
];

export const metadata: Metadata = {
  title: NAME,
  description: `${NAME}, founder of <Brand>.`,
  alternates: { canonical: PAGE_URL },
  openGraph: { type: "profile", url: PAGE_URL, title: `${NAME}, founder of <Brand>` },
};

const ld = ldJson({
  "@context": "https://schema.org",
  "@type": "ProfilePage",
  url: PAGE_URL,
  mainEntity: {
    "@type": "Person",
    "@id": `${PAGE_URL}#person`,
    name: NAME,
    jobTitle: "Founder, <Brand>",
    url: PAGE_URL,
    worksFor: { "@id": `${SITE_URL}/#organization` },
    sameAs: LINKS.map(([, href]) => href),
  },
});

export default function Founder() {
  return (
    <div className="mx-auto max-w-2xl space-y-8">
      <script type="application/ld+json" dangerouslySetInnerHTML={{ __html: ld }} />
      <h1>{NAME}</h1>
      <p>{/* two or three factual sentences: what they build, background, where they write */}</p>
      <section aria-labelledby="elsewhere">
        <h2 id="elsewhere">Elsewhere</h2>
        <ul>{LINKS.map(([label, href]) => <li key={href}><a href={href} target="_blank" rel="me noopener">{label}</a></li>)}</ul>
      </section>
    </div>
  );
}
```

Rules: only list profiles that are really the founder's and that link back where possible (`rel="me"` on both sides). Use the same name spelling everywhere. In each blog post's Article JSON-LD, set `author: { "@type": "Person", name: NAME, url: PAGE_URL }`.

### 9.3 Hubs

Every program has a hub: `/use-cases`, `/tools`, `/templates`, `/alternatives`, `/compare`. A hub is a list of groups of link cards, built from the registry, with ItemList JSON-LD.

```tsx
// components/hub.tsx
import { Cta } from "@/components/cta";
import { LandingTracking } from "@/components/landing-tracking";
import { LinkCards, type LinkCard } from "@/components/link-cards";
import { ldJson } from "@/lib/json-ld";
import { SITE_URL } from "@/lib/site";

export type HubGroup = { id: string; label: string; links: LinkCard[] };

export function Hub({ id, h1, sub, groups }: { id: string; h1: string; sub: string; groups: HubGroup[] }) {
  const all = groups.flatMap((g) => g.links);
  const ld = ldJson({
    "@context": "https://schema.org",
    "@type": "ItemList",
    name: h1,
    itemListElement: all.map((l, i) => ({ "@type": "ListItem", position: i + 1, name: l.label, url: `${SITE_URL}${l.href}` })),
  });
  return (
    <div>
      <LandingTracking />
      <script type="application/ld+json" dangerouslySetInnerHTML={{ __html: ld }} />
      <section data-section={`${id}_hero`}>
        <h1>{h1}</h1>
        <p>{sub}</p>
        <Cta location={id} />
      </section>
      {groups.map((g) => (
        <section key={g.id} id={g.id} data-section={`${id}_${g.id}`} aria-labelledby={`${g.id}-h`}>
          <h2 id={`${g.id}-h`}>{g.label}</h2>
          <LinkCards links={g.links} location={id} />
        </section>
      ))}
    </div>
  );
}
```

```tsx
// app/alternatives/page.tsx
import { Hub } from "@/components/hub";
import { pageMetadata } from "@/lib/page-metadata";
import { alternativePath, HEAD_TO_HEAD, headToHeadPath, RIVAL_LIST, vsPath } from "@/lib/registry/compare";

export const metadata = pageMetadata("/alternatives", "<Category> software alternatives, compared fairly", "Alternatives to the main <category> tools, with free plans, prices and exports checked on each tool's own site.");

export default function Page() {
  return (
    <Hub
      id="alternatives"
      h1="Find the right <category> tool."
      sub="Free plans, prices and exports, checked at the source."
      groups={[
        { id: "alternatives", label: "Alternatives", links: RIVAL_LIST.map((r) => ({ href: alternativePath(r.slug), label: `${r.name} alternatives`, blurb: r.blurb, cta: "alternatives_link" })) },
        { id: "head-to-head", label: "Head to head", links: HEAD_TO_HEAD.map((c) => ({ href: headToHeadPath(c.slug), label: c.label, blurb: c.blurb, cta: "compare_link" })) },
        { id: "vs", label: "<Brand> vs", links: RIVAL_LIST.map((r) => ({ href: vsPath(r.slug), label: `vs ${r.name}`, blurb: r.blurb, cta: "vs_link" })) },
      ]}
    />
  );
}
```

The use-case hub can also carry a "Compare" group with the `/vs` pages, and the footer's "All comparisons" link can point to `/use-cases#compare`, so one hub covers both programs.

### 9.4 The footer

Vertical link lists under small headings, built from the registries. Never inline rows of links: they are hard to scan and look like link stuffing.

```tsx
// components/footer.tsx
import Link from "next/link";
import { RIVAL_LIST, vsPath } from "@/lib/registry/compare";
import { FOOTER_USE_CASES, useCase } from "@/lib/registry/use-cases";

export function Footer({ signedIn }: { signedIn: boolean }) {
  const columns = [
    { id: "product", label: "Product", links: [
      { href: signedIn ? "/app" : "/signup", label: "Get started", cta: "sign_up" },
      { href: "/pricing", label: "Pricing", cta: "pricing_link" },
      { href: "/templates", label: "Templates", cta: "templates_link" },
      { href: "/tools", label: "Free tools", cta: "tools_link" },
    ] },
    { id: "use_cases", label: "Use cases", links: [
      ...FOOTER_USE_CASES.map(useCase).map((u) => ({ href: `/for/${u.slug}`, label: u.label, cta: "usecase_link" })),
      { href: "/use-cases", label: "All use cases", cta: "use_cases_link" },
    ] },
    { id: "compare", label: "Compare", links: [
      ...RIVAL_LIST.slice(0, 4).map((r) => ({ href: vsPath(r.slug), label: `vs ${r.name}`, cta: "vs_link" })),
      { href: "/alternatives", label: "Alternatives", cta: "alternatives_link" },
    ] },
    { id: "company", label: "Company", links: [
      { href: "/about", label: "About", cta: "about_link" },
      { href: "/contact", label: "Contact", cta: "contact_link" },
      { href: "/privacy", label: "Privacy", cta: "privacy_link" },
    ] },
  ];
  return (
    <footer>
      <div className="grid grid-cols-2 gap-10 sm:grid-cols-4">
        {columns.map((col) => (
          <nav key={col.id} aria-labelledby={`footer-${col.id}`}>
            <p id={`footer-${col.id}`} className="text-xs font-medium">{col.label}</p>
            <ul className="mt-3 space-y-1 text-sm">
              {col.links.map((l) => (
                <li key={l.href}><Link href={l.href} data-cta={l.cta} data-cta-location={`footer_${col.id}`}>{l.label}</Link></li>
              ))}
            </ul>
          </nav>
        ))}
      </div>
    </footer>
  );
}
```

Keep each column to six or seven links. The hub is where the full list lives; the footer points to it.

### 9.5 Quality rules

1. About and Contact have at least 500 characters each, name a real person and a reachable address.
2. The founder page lists only real, matching profiles.
3. Every program has a hub with ItemList JSON-LD, and every hub is in the sitemap and the footer.
4. The footer is built from registries, so a new page can't be missed and a removed one can't linger.

---

## 10. Launch checklist per program

### Every page

- [ ] Answers a real query (autocomplete, Search Console or your buyer prompt list), recorded with the date.
- [ ] Registry entry added; the sync test passes.
- [ ] Own static route with a static `metadata` export: title 70 or fewer characters, description 70 to 160, canonical, Open Graph.
- [ ] JSON-LD per the table below, validated with the Rich Results Test or the Schema.org validator.
- [ ] Links to three related pages and its hub; the hub and footer link to it.
- [ ] One primary CTA at the top and one at the bottom, with `data-cta` and `data-cta-location`; sections carry `data-section`.
- [ ] Every claim matches a real limit, read from your constants.
- [ ] No horizontal scroll at 360, 390 and 414 px; no layout shift from late-loading visuals.
- [ ] No em or en dashes in copy (the test checks the data files).
- [ ] After deploy: in the live sitemap, inspected in Search Console, IndexNow pinged.

| Program | JSON-LD |
|---|---|
| Use case | FAQPage (plus BreadcrumbList if shown) |
| `/vs` | FAQPage |
| Alternatives | ItemList of SoftwareApplication, FAQPage |
| Head-to-head | FAQPage |
| Template | DigitalDocument (or a subtype), FAQPage, BreadcrumbList |
| Template hub | CollectionPage with ItemList |
| Tool | WebApplication, FAQPage |
| Other hubs | ItemList |
| Founder | ProfilePage with Person and `sameAs` |
| Home | Organization with `email` and `contactPoint`, WebSite |

### Use-case pages

- [ ] The input was run through the product and works every time.
- [ ] Visuals are real renders from the product's own renderer.
- [ ] Every number in the output is in the sample input or is a checkable derivative.
- [ ] Caption says the example uses fictional names.
- [ ] FAQ answers accuracy, privacy and editing for this input.

### Comparison pages

- [ ] Every rival cell comes from their own pricing, help or docs pages, or reads "Not checked".
- [ ] Footnote with the checked month and links (`rel="nofollow noopener"`).
- [ ] "Where <Rival> is a good fit" names a real strength.
- [ ] No rival logos or screenshots; nothing implies a partnership.
- [ ] Re-check date in the calendar.

### Alternatives and head-to-head

- [ ] Pair appears in autocomplete in both orders; engine and month recorded.
- [ ] Facts come from the shared product table.
- [ ] The alternatives list includes at least one widely used option you don't compete with directly, and shows where the rival wins.
- [ ] Head-to-head stays neutral until "Or try <Brand> if", with at least one honest tie where true.
- [ ] Year in the title comes from one constant.

### Templates

- [ ] Content validates against your real schema in a test.
- [ ] Every part of the template is rendered on the page.
- [ ] "Use this template" works signed in and signed out; signed-out visitors land in their copy after sign-up.
- [ ] Sign-up only follows same-site `next` paths.
- [ ] Copies are capped per day in one SQL statement.

### Free tools

- [ ] Browser tools send nothing over the network (checked in dev tools).
- [ ] AI tools: input schema, email gate on the server, per-IP rate, per-IP daily cap, global daily cap, failed calls released.
- [ ] IPs stored only as an HMAC.
- [ ] Spend recorded per call; an alert when the global cap is hit.
- [ ] Privacy page has a "Free tools" section.
- [ ] Events wired, and failure events in your error alert.

### Embeddable widget

- [ ] `curl -I` shows `frame-ancestors *` on `/embed/<tool>` and `'none'` on every other page.
- [ ] Embed is noindex, follow, with no site chrome.
- [ ] Snippet repeats the brand credit link outside the iframe.
- [ ] `embed_viewed` records the embedding host.

### Trust pages, hubs, footer

- [ ] About and Contact have 500 or more characters, a real person and a reachable address.
- [ ] Founder page: ProfilePage with Person and real `sameAs` links; blog authors point to it.
- [ ] Organization JSON-LD with `email` and `contactPoint` on the home page.
- [ ] Every hub in the sitemap and footer; footer columns built from registries.

---

## 11. Common mistakes

1. **Hand-maintained lists.** The sitemap, footer and hub each keep their own list and drift apart. Use one registry and the sync test.
2. **One dynamic route with `generateMetadata` on a framework that streams it.** Titles and canonicals end up in `<body>`. Use a static route file per slug with a static `metadata` export, or confirm your framework puts metadata in `<head>` for crawlers.
3. **Copy-pasted route folders that still render the old slug.** Two URLs with identical content. The test that checks each `page.tsx` contains its own slug catches it.
4. **Mock screenshots.** They go stale and visitors can tell. Render real output with the real component.
5. **Sample numbers that don't match the input.** The first thing a careful visitor checks. One mismatch costs the whole page's credibility.
6. **Claiming inputs the product handles only sometimes.** Sign-ups from that page fail on their first try and never come back.
7. **Rival facts from memory, review sites or AI answers.** They are often wrong or out of date. Use only the rival's own pages, date them, and write "Not checked" when you can't verify.
8. **No "good fit" section.** A comparison that says the rival is bad at everything reads as an ad, and is the kind of claim that draws legal letters.
9. **Using rival logos and product screenshots.** Name them in text; that is all you need.
10. **Head-to-head pages nobody searches.** Build pairs only when autocomplete shows both orders.
11. **Thin templates.** A cover page and three empty sections won't rank or convert. Ship fewer, complete ones.
12. **An open redirect in the sign-up `next` parameter.** Accept only paths that start with `/` and not `//`.
13. **Check-then-insert rate limits.** Parallel requests all pass the check. Count and insert in one statement.
14. **Counting failed AI calls against the visitor.** They hit the cap through your bugs. Mark failures and exclude them.
15. **No global cap on anonymous AI tools.** Per-IP caps don't stop many IPs. A global daily cap bounds the bill.
16. **Storing raw IP addresses.** You don't need them. Store an HMAC.
17. **Gating the whole tool page behind an email.** Crawlers and visitors see nothing. Gate only the result.
18. **Allowing framing site-wide to make an embed work.** Allow it on `/embed/*` only, and keep that path free of anything that needs a session.
19. **Keyword-stuffed or hidden widget credit links.** That is link spam. Use a visible brand credit.
20. **Blocking program pages or embeds in `robots.txt`.** Google can't see a `noindex` it isn't allowed to fetch, and may index the bare URL from links. Use `noindex` headers or meta tags instead.
21. **Trust pages with two sentences.** Write real content: who, why, how data is handled, how to reach a person.
22. **Prices and limits typed into page copy.** They go stale the day pricing changes. Read them from the same constants your billing code uses.
23. **Forgetting the re-check.** Rival pages, the year in alternatives titles and the "updated" month on head-to-head pages all need a calendar reminder.
24. **Tracking without locations.** A `sign_up` click with no `data-cta-location` tells you nothing about which page or which position converts.
