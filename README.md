# SaaS SEO playbook

A complete, step-by-step guide to getting a small SaaS found on Google, Bing and AI assistants, with working code you can copy into your own repo.

It is written from what one AI SaaS product actually built and tested in its first weeks. Every chapter is generic: swap in your domain, your product and your pages. The examples use Next.js (App Router) and TypeScript, with notes for other stacks where it matters.

## Who it's for

- Founders and small teams with a web app and no SEO budget.
- Developers who want the SEO work done in code, once, instead of by hand every week.
- Anyone who wants to show up when people ask ChatGPT, Perplexity or Gemini for a tool like theirs.
- Teams that want their product usable inside ChatGPT and Claude as an app, and found in those directories.

## What the results looked like

In its first 10 days, the product behind this playbook got about 300 sign-ups. Asked how they found it, 83 said "ChatGPT or an AI assistant" and 10 said Google. The site was days old and barely ranked, so the early traffic came from AI assistants, directories and communities while search pages were still being indexed. Chapter 5 covers the AI-search side.

## Chapters

| # | Chapter | What you'll have at the end |
|---|---|---|
| 1 | [Technical foundations](docs/01-technical-foundations.md) | One canonical host, robots.txt and noindex rules that don't backfire, a sitemap built from code, metadata and JSON-LD on every page type, fast pages |
| 2 | [Getting indexed](docs/02-getting-indexed.md) | Search Console and Bing set up, IndexNow submitting every URL from CI, video pages Google accepts |
| 3 | [Page programs](docs/03-page-programs.md) | Use-case, comparison, alternatives, template, free-tool, embed and trust pages, all fed from one registry |
| 4 | [Automated blog writer](docs/04-automated-blog-writer.md) | A scheduled job that picks real queries from Search Console, researches them, writes cited posts and rejects anything that fails strict quality gates |
| 5 | [AI search](docs/05-ai-search.md) | `llms.txt`, Markdown for agents, trust pages, a buyer prompt list and a monthly check of whether assistants cite you |
| 6 | [Off-page](docs/06-off-page.md) | Launch sites, directories, review sites, roundup outreach, communities, Medium and Dev.to, and referral links you own |
| 7 | [Measurement](docs/07-measurement.md) | Search Console reports that matter, CTA tracking on landing pages, a weekly and monthly review routine |
| 8 | [Launching in ChatGPT and Claude](docs/08-ai-app-directories.md) | One MCP server and OAuth setup that both accept, tool metadata that gets called, a golden prompt set, listings in both directories and the MCP Registry |

Then the [strategy log](STRATEGY-LOG.md): every change made to the product behind it, with the date and the reason.

## How to use it

1. Read chapter 1 and fix your foundations first. Pages built on a split host or blocked robots rules won't rank, however good they are.
2. Set up chapter 2 the same day, so everything you publish from then on gets indexed automatically.
3. Pick the page programs from chapter 3 that match how people search for your product. Start with one.
4. Add the rest in the order of the plan below. Each chapter ends with a checklist and a list of common mistakes.

## A four-week rollout

| Week | Do this | Chapter |
|---|---|---|
| 1 | Canonical host and redirects, robots.txt, noindex headers, sitemap, metadata, JSON-LD | 1 |
| 1 | Search Console, Bing Webmaster Tools, IndexNow in CI | 2 |
| 1 | `llms.txt`, About and Contact pages, Organization JSON-LD, a "How did you hear about us?" question at sign-up | 5 |
| 2 | Your first page program: 5 to 10 use-case pages for the jobs people search for | 3 |
| 2 | Launch kit, tracker file, free directories and review sites | 6 |
| 3 | Comparison and alternatives pages, with dated facts | 3 |
| 3 | One free tool or a template gallery | 3 |
| 4 | The automated blog writer, with a human reviewing its first posts | 4 |
| 4 | CTA tracking, then the weekly review routine from here on | 7 |
| 5+ | Once the product has a stable API: an MCP server, listings in ChatGPT and Claude, and the MCP Registry | 8 |

## Principles

- **Real pages for real searches.** Every page answers a query that people type. Google penalises scaled, low-value content, so quality beats volume.
- **Show the product, don't describe it.** Use real output from your product, never mock screenshots.
- **Never state an unchecked fact about a rival.** Competitor claims come only from their own pricing or help pages, with the month you checked. Anything unverified is left out or marked "Not checked".
- **One source of truth.** Lists of pages live in one registry. The routes, sitemap, hubs, footer and `llms.txt` all read from it.
- **Every page links into the funnel.** Each page links to related pages and ends with one sign-up call to action, tracked by location.
- **Say the same facts everywhere.** Prices, limits and features read the same on your site, in `llms.txt` and in every listing. Assistants repeat whatever they find.

## Keeping it current

This repo is updated whenever the product behind it changes something on the SEO side: the matching chapter is edited and a row is added to the [strategy log](STRATEGY-LOG.md). Claims about third-party services (Medium, directories, Search Console limits) were checked on the dates given; re-check them before you rely on them.

Found a mistake or have a better method? Open an issue or a pull request.
