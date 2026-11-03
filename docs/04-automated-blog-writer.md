# 4. Automated blog writer

A scheduled job that picks a real search query, researches it with live sources, writes a cited post in a fixed voice, rejects anything that fails strict quality gates, and publishes the rest.

## What you'll build

- A cron trigger that starts at most N runs a day and never two at once.
- A background runner that gets 15 minutes of wall time, outside the cron invocation.
- Three tables: posts, a topic queue, and a log of every run with its cost and failure reason.
- Topic selection from your own Google Search Console data, rotating through funnel stages, skipping queries a post already ranks for.
- A research step with web search grounding, where every source must answer HTTP 200 and pass a trust triage.
- A writing pipeline: outline, draft, self-check, and a citation check that reads each cited page.
- A persona file, a knowledge file and a single facts file, bundled with the code and editable without a deploy of new logic.
- A lint function with hard thresholds, plus up to two "fix only these issues" repair passes.
- Time limits, a cost cap per run, recorded spend, and one retry on the next topic.
- Rendering with raw HTML disabled, Article, BreadcrumbList and FAQPage JSON-LD, and a sitemap and `llms.txt` that read posts at request time.
- An admin view of runs and a weekly review routine.

The worked example uses Gemini with Google Search grounding, Cloudflare Workers with a Durable Object as the runner, and D1 (SQLite) as the database. Each part says how to swap it.

---

## 1. Why automate, and the quality risk

### Why

A small SaaS has long-tail queries it already gets impressions for and no page that answers them. Writing two sourced posts a day by hand is not realistic for a founder. A pipeline can do the research legwork, follow rules every time, and refuse to publish when it can't meet them.

### The risk

Google's spam policies include **scaled content abuse**: producing many pages mainly to manipulate rankings rather than help people, no matter whether a human, an AI or a mix made them. Google's guidance on AI-generated content says the method doesn't matter; helpfulness does. A pipeline that publishes thin, generic or invented content at volume is exactly what that policy targets, and the penalty can hit the whole site, not only the blog.

So the design goal is not "more posts". It is "only posts a careful human editor would also publish". Every gate in this chapter exists for that reason:

- Topics come from real queries people type, not from a keyword spinner.
- Every external claim is cited to a page that was fetched and returned 200, and a second pass checks the claim against that page's text.
- Product claims come only from one facts file you maintain.
- Volume is low (two a day in the example) and capped in the database.
- A post that fails lint after two repairs is not published. The run records why.
- A human reads the output every week.

### When not to do this

- **You can't review it.** If nobody will read the posts weekly, don't publish them automatically. Write them as drafts and publish by hand instead.
- **YMYL topics.** Health, finance, legal or safety advice needs real expertise and accountability. Don't automate it.
- **No product knowledge to teach.** If your posts can only be generic advice that a hundred other sites already cover, they won't rank and may hurt you.
- **Too few pages to link into.** The pipeline links each post into your use-case, comparison or tool pages. With none, posts become orphans with no path to sign-up. Build those pages first.
- **A brand-new domain with no Search Console data.** It still works on the fallback queue, but you lose the best signal. Consider one or two hand-written posts a week until queries appear.
- **You'd publish more than you can fix.** Start at one post a day or less. Raise it only after a few weeks of clean reviews.

> **Real-world example:** the blog first ran as a GitHub workflow that committed Markdown files. It moved into the API with database storage and Search Console topics on 2026-10-28, then went to two new posts a day (never refreshing old ones) on 2026-10-29.

---

## 2. Architecture

```
  hourly cron (scheduled handler)
        |
        |  hour is in BLOG_HOURS?
        v
  start guard (one SQL statement)
   - close "running" rows older than 30 min as failed
   - insert a "running" row only if none is running
     and today's published count < daily quota
        |
        |  run id (or nothing: stop)
        v
  background runner (Durable Object alarm, 15 min wall time,
                     pinned to a region the model provider serves)
        |
        v
  pipeline (12 min hard limit, $1 cost cap)
   1. load state: posts, topic queue, sitemap routes, Search Console rows
   2. pick topic: stage of the day -> open query -> model shapes a topic
                  (fallback: queued topic -> model proposes 10)
   3. research: grounded search -> resolve -> HTTP 200 check -> triage
   4. write: outline -> draft -> self-check -> citation check -> clean
   5. lint -> repair (max 2) -> lint
        |                      \
        | pass                   \ fail: record reason, skip topic,
        v                         \ retry once on the next topic if < 6 min
  database (post row + topic "used" in one batch; run "published", cost)
        |
        v
  site (reads posts at request time: /blog, /blog/<slug>, sitemap.xml, llms.txt)
        |
        v
  IndexNow ping for the new URL
```

### Why the work can't run inside the cron invocation

On serverless platforms a scheduled invocation is short-lived. On Cloudflare Workers, work handed to `waitUntil` after the handler returns gets roughly 30 seconds. One run here makes around ten model calls, several with web search, and a single call can take minutes. A run takes several minutes in total.

There is a second, less obvious reason. The isolate that runs a cron can land in any data centre, and some model providers refuse requests from some locations ("User location is not supported"). A background runner can be pinned near a region the provider serves.

So the cron only does two cheap things: decide whether to start, and hand off a run id. The runner does the rest.

### The cron handler

```ts
export const BLOG_HOURS: readonly number[] = [3, 15];
export const BLOG_JOB = "blog:";

export default {
  async scheduled(event: ScheduledController, env: Env, ctx: ExecutionContext) {
    if (BLOG_HOURS.includes(new Date(event.scheduledTime).getUTCHours()))
      ctx.waitUntil(startBlog(env, "cron", event.scheduledTime).catch((e) => logError("blog run start failed", e)));
  },
};
```

The cron itself fires every hour (`0 * * * *`) because the same trigger runs other hourly jobs. If the blog is your only job, schedule it at the exact hours instead.

### The start guard

One statement claims the run. It succeeds only when nothing else is running and today's quota is not met. A crashed run would block everything forever, so stale `running` rows are closed first.

```ts
const DAY_MS = 86_400_000;
const STALE_MS = 30 * 60_000;
const dayStart = (ms: number) => ms - (ms % DAY_MS);

export async function claimRun(env: Env, trigger: "cron" | "manual", capped = false, now = Date.now()) {
  await env.DB.prepare(
    "update blog_run set status = 'failed', error = 'Stopped without finishing.', finishedAt = ? where status = 'running' and startedAt <= ?",
  ).bind(now, now - STALE_MS).run();
  const row = await env.DB.prepare(
    `insert into blog_run (startedAt, kind, status, "trigger") select ?1, 'new', 'running', ?2
     where not exists (select 1 from blog_run where status = 'running')
       and (?3 = 0 or (select count(*) from blog_run where kind = 'new' and status = 'published' and startedAt >= ?4) < ?5)
     returning id`,
  ).bind(now, trigger, capped ? 1 : 0, dayStart(now), BLOG_HOURS.length).first<{ id: number }>();
  return row?.id ?? null;
}
```

Manual runs from the admin pass `capped = false`, so you can force one even when today's quota is met. They are still blocked while another run is going.

### Handing off to the runner

```ts
export async function startBlog(env: Env, trigger: "cron" | "manual", now = Date.now()) {
  const id = await claimRun(env, trigger, trigger === "cron", now);
  if (!id) return null;
  try {
    const stub = env.JOB_RUNNER.get(env.JOB_RUNNER.idFromName(`${BLOG_JOB}${id}`), { locationHint: "wnam" });
    const res = await stub.fetch(`https://job-runner/${BLOG_JOB}${id}`, { method: "POST" });
    if (!res.ok) throw new Error(`job runner refused: ${res.status}`);
  } catch (e) {
    logError("blog run dispatch failed", e, { runId: id });
    await finishRun(env, id, { status: "failed", error: "Couldn't start the run." });
    throw new HttpError(502, "Couldn't start the blog run. Try again in a minute.");
  }
  return id;
}
```

### The runner

A Durable Object stores the job id, sets an alarm for now, and does the work in `alarm()`, which gets 15 minutes of wall time. A throw retries the alarm once.

```ts
export class JobRunner {
  constructor(private state: DurableObjectState, private env: Env) {}

  async fetch(req: Request) {
    const jobId = new URL(req.url).pathname.slice(1);
    await this.state.storage.put("message", { jobId });
    await this.state.storage.setAlarm(Date.now());
    return new Response(null, { status: 202 });
  }

  async alarm(info?: { retryCount: number }) {
    const message = await this.state.storage.get<{ jobId: string }>("message");
    if (!message) return;
    try {
      if (message.jobId.startsWith(BLOG_JOB)) await runBlog(this.env, Number(message.jobId.slice(BLOG_JOB.length)));
    } catch (e) {
      if ((info?.retryCount ?? 0) < 1) throw e;
      logError("job runner gave up", e, { jobId: message.jobId });
    }
    await this.state.storage.deleteAll();
  }
}
```

`runBlog` starts by checking that its run row is still `running`, so a retried alarm never writes a second post for the same run.

Wrangler config for this part:

```jsonc
{
  "triggers": { "crons": ["0 * * * *"] },
  "rules": [{ "type": "Text", "globs": ["**/*.md"], "fallthrough": true }],
  "durable_objects": { "bindings": [{ "name": "JOB_RUNNER", "class_name": "JobRunner" }] },
  "migrations": [{ "tag": "v1", "new_sqlite_classes": ["JobRunner"] }]
}
```

The `Text` rule lets the worker `import persona from "./persona.md"` as a string, so the voice and facts stay in plain Markdown files.

### Swapping the parts

| Part | Worked example | Alternatives |
|---|---|---|
| Scheduler | Workers cron trigger | Vercel cron, GitHub Actions `schedule`, Kubernetes CronJob, `node-cron` in a long-lived process |
| Runner | Durable Object alarm (15 min) | Cloudflare Workflows or Queues, AWS Lambda (15 min max) via SQS, Google Cloud Run jobs, a BullMQ worker, a plain VM process |
| Database | D1 (SQLite) | Postgres, MySQL, any SQL store with `insert ... where not exists ... returning` or an equivalent advisory lock |
| Model | Gemini with `google_search` grounding | Any LLM with a web search tool (OpenAI's or Anthropic's web search tools), or any LLM plus a separate search API whose results you fetch yourself |
| Index ping | IndexNow | IndexNow works for any host; Google reads your sitemap |

What matters is the shape: a cheap guard in the scheduler, a single-flight claim in the database, and a runner with enough wall time.

---

## 3. Database schema

Three tables. JSON lists are stored as text. Times are epoch milliseconds; post dates are `YYYY-MM-DD` strings because that is what the page shows.

```sql
create table blog_post (
  slug text primary key,
  title text not null,
  description text not null,
  blurb text not null,
  date text not null,          -- YYYY-MM-DD, first published
  updated text not null,       -- YYYY-MM-DD, last meaningful edit
  author text not null,
  related text not null,       -- JSON array of site paths, e.g. ["/for/x", "/vs/y"]
  sources text not null,       -- JSON array of {title, url, checked}
  body text not null,          -- Markdown, no H1
  createdAt integer not null,
  updatedAt integer not null,
  deletedAt integer            -- soft delete; every read filters "deletedAt is null"
);

create table blog_topic (
  id integer primary key autoincrement,
  slug text not null unique,
  title text not null,
  intent text not null,        -- one sentence: what the reader needs
  related text not null,       -- JSON array of site paths
  tool text,                   -- optional /tools/<slug> for bottom-of-funnel posts
  keyword text,                -- the Search Console query it targets, if any
  stage text,                  -- top | middle | bottom
  status text not null check (status in ('queued', 'used', 'skipped')),
  createdAt integer not null,
  usedAt integer
);

create table blog_run (
  id integer primary key autoincrement,
  startedAt integer not null,
  finishedAt integer,
  kind text not null check (kind in ('new', 'refresh')),
  status text not null check (status in ('running', 'published', 'skipped', 'failed')),
  slug text,
  query text,
  stage text,
  costUsd real,
  error text,
  "trigger" text not null check ("trigger" in ('cron', 'manual'))
);
create index blog_run_startedAt_idx on blog_run (startedAt);
```

Notes:

- `trigger` is quoted because it is a reserved word in SQLite and Postgres.
- `kind = 'refresh'` is kept in the check for a future "update an old post" mode. The example only writes new posts.
- In Postgres, use `jsonb` for `related` and `sources`, `bigint` for times, and `identity` for ids. The guard query works unchanged.
- Never hard-delete a post row. Set `deletedAt`. The writer still loads deleted posts so it won't reuse their slugs.

> **Real-world example:** the migration also seeded `blog_topic` from the old topics file and inserted the posts the GitHub workflow had already written, so the move lost nothing.

---

## 4. Topic selection

The order is:

1. Today's funnel stage, from the day of the year.
2. The best Search Console query for that stage that no post already ranks for.
3. A model call that shapes that query into a topic, validated in code.
4. If there's no usable query, or the model's topic is rejected: the next queued topic.
5. If the queue is empty: the model proposes 10 topics, they're de-duplicated and queued, and the first is used.

### 4.1 Search Console API with a service account

Setup, once:

1. In Google Cloud Console, create a project (or use an existing one).
2. Go to **APIs & Services > Library**, search for **Google Search Console API**, and enable it.
3. Go to **IAM & Admin > Service Accounts > Create service account**. Give it a name like `blog-reader`. It needs no project roles.
4. Open the account, go to **Keys > Add key > Create new key > JSON**. A JSON file downloads. Treat it like a password.
   - If key creation is blocked, your organisation has the `iam.disableServiceAccountKeyCreation` policy. An org admin has to allow it for this project.
5. Copy the service account's email address (it ends in `.iam.gserviceaccount.com`).
6. In Search Console, open your property, go to **Settings > Users and permissions > Add user**, paste that email, and choose **Restricted**. Read access to performance data is all it needs.
7. Store the whole JSON file contents as one secret, for example `wrangler secret put GSC_SERVICE_ACCOUNT` and paste it. Never commit it.
8. Note your property id:
   - a domain property is `sc-domain:example.com`;
   - a URL-prefix property is the full URL with a trailing slash, `https://example.com/`.

The API returns data that lags by two to three days, and new sites may have very few rows at first. The code treats "no data" as normal and falls back to the queue.

### 4.2 Signing the JWT with WebCrypto

Google's client libraries don't run on every edge runtime. The service account flow is small enough to do by hand: build an RS256 JWT, exchange it for an access token, call the Search Analytics endpoint.

```ts
const GSC_SITE = "sc-domain:example.com";

const b64url = (bytes: Uint8Array) => btoa(String.fromCharCode(...bytes)).replace(/\+/g, "-").replace(/\//g, "_").replace(/=+$/, "");
const b64json = (v: unknown) => b64url(new TextEncoder().encode(JSON.stringify(v)));
const isoDay = (ms: number) => new Date(ms).toISOString().slice(0, 10);

export async function serviceJwt({ client_email, private_key }: { client_email: string; private_key: string }, now = Date.now()) {
  const der = Uint8Array.from(atob(private_key.replace(/-----[^-]+-----|\s/g, "")), (c) => c.charCodeAt(0));
  const key = await crypto.subtle.importKey("pkcs8", der, { name: "RSASSA-PKCS1-v1_5", hash: "SHA-256" }, false, ["sign"]);
  const iat = Math.floor(now / 1000);
  const unsigned = `${b64json({ alg: "RS256", typ: "JWT" })}.${b64json({
    iss: client_email,
    scope: "https://www.googleapis.com/auth/webmasters.readonly",
    aud: "https://oauth2.googleapis.com/token",
    iat,
    exp: iat + 3600,
  })}`;
  const sig = new Uint8Array(await crypto.subtle.sign("RSASSA-PKCS1-v1_5", key, new TextEncoder().encode(unsigned)));
  return `${unsigned}.${b64url(sig)}`;
}

export type GscRow = { query: string; page: string; clicks?: number; impressions: number; ctr?: number; position: number };

export async function fetchQueries(env: { GSC_SERVICE_ACCOUNT?: string }, now = Date.now()): Promise<GscRow[]> {
  if (!env.GSC_SERVICE_ACCOUNT) return [];
  try {
    const assertion = await serviceJwt(JSON.parse(env.GSC_SERVICE_ACCOUNT), now);
    const tok = await fetch("https://oauth2.googleapis.com/token", {
      method: "POST",
      headers: { "content-type": "application/x-www-form-urlencoded" },
      body: new URLSearchParams({ grant_type: "urn:ietf:params:oauth:grant-type:jwt-bearer", assertion }),
      signal: AbortSignal.timeout(20_000),
    });
    if (!tok.ok) throw new Error(`token exchange returned ${tok.status} ${(await tok.text()).slice(0, 200)}`);
    const { access_token } = (await tok.json()) as { access_token: string };
    const res = await fetch(`https://www.googleapis.com/webmasters/v3/sites/${encodeURIComponent(GSC_SITE)}/searchAnalytics/query`, {
      method: "POST",
      headers: { authorization: `Bearer ${access_token}`, "content-type": "application/json" },
      body: JSON.stringify({ startDate: isoDay(now - 27 * 864e5), endDate: isoDay(now), dimensions: ["query", "page"], rowLimit: 1000 }),
      signal: AbortSignal.timeout(30_000),
    });
    if (!res.ok) throw new Error(`searchAnalytics returned ${res.status} ${(await res.text()).slice(0, 200)}`);
    const data = (await res.json()) as { rows?: { keys: string[]; clicks: number; impressions: number; ctr: number; position: number }[] };
    return (data.rows ?? []).map((r) => ({ query: r.keys[0], page: r.keys[1], clicks: r.clicks, impressions: r.impressions, ctr: r.ctr, position: r.position }));
  } catch (e) {
    logWarn("search console unavailable", { error: e instanceof SyntaxError ? "GSC_SERVICE_ACCOUNT is not valid JSON" : e instanceof Error ? e.message.slice(0, 200) : "unknown" });
    return [];
  }
}
```

Details that matter:

- The private key in the JSON has literal `\n` line breaks and PEM headers. The regex strips the headers and all whitespace before base64 decoding.
- The `query` and `page` dimensions together tell you which page ranks for which query. That is how the picker knows a post already owns a query.
- `rowLimit` can go up to 25,000. 1,000 rows of the last 28 days is plenty for a young site.
- A failure logs a warning and returns `[]`. Search Console being down must never stop the blog; the queue takes over.
- On Node, `crypto.subtle` is available as `globalThis.crypto.subtle` (Node 19+), so this code runs there too. Or use the `google-auth-library` package.

### 4.3 Funnel stages and the daily rotation

Each query is classified by keywords. Middle-of-funnel words are checked first, because "best pdf to ppt converter" is a comparison even though it contains "converter".

```ts
export const STAGES = ["top", "middle", "bottom"] as const;
export type Stage = (typeof STAGES)[number];
export const CYCLE: Stage[] = ["bottom", "middle", "bottom", "middle", "top"];
export const STRIKING = { min: 5, max: 30 };

// Replace with your category's words and your rivals' names.
export const KEYWORDS: Record<Stage, string[]> = {
  middle: ["vs", "versus", "alternative", "best", "review", "compare", "comparison", "pricing", "price", "cost", "rival one", "rival two"],
  top: ["how to", "how do", "how many", "how long", "what is", "what are", "why", "should", "when", "tip", "example", "structure", "guide", "idea", "rule"],
  bottom: ["maker", "generator", "converter", "convert", "template", "free", "make a", "make an", "create", "online", "app"],
};

const norm = (s: string) => String(s).toLowerCase().replace(/[^a-z0-9]+/g, " ").trim();
const has = (q: string, words: string[]) => words.some((w) => new RegExp(`\\b${w}s?\\b`).test(q));
const BRANDED = /\byour ?brand\b/; // your own brand name: those searches already find you

export function funnelStage(query: string): Stage {
  const q = norm(query);
  if (has(q, KEYWORDS.middle)) return "middle";
  if (has(q, KEYWORDS.top)) return "top";
  if (has(q, KEYWORDS.bottom)) return "bottom";
  return "top";
}

export const dayOfYear = (d = new Date()) =>
  Math.floor((Date.UTC(d.getUTCFullYear(), d.getUTCMonth(), d.getUTCDate()) - Date.UTC(d.getUTCFullYear(), 0, 0)) / 864e5);
```

The cycle is bottom, middle, bottom, middle, top. Bottom and middle posts are closer to a sign-up, so they get four days in five. Top-of-funnel "how to" posts still get one, because they earn links and build topical authority. Change the cycle to fit your goals.

Both runs on the same day share a stage. That is fine: the second run picks the next-best query for it.

### 4.4 Picking a query no post ranks for

```ts
const usable = (q: string) => q.length >= 3 && !BRANDED.test(q);
const striking = (pos: number) => pos >= STRIKING.min && pos <= STRIKING.max;

export function blogSlug(page: string) {
  try {
    return /^\/blog\/([a-z0-9-]+)$/.exec(new URL(page).pathname.replace(/\/+$/, ""))?.[1] ?? null;
  } catch {
    return null;
  }
}

export type TopicPick =
  | { kind: "new"; stage: Stage; query: string; why: string }
  | { kind: "queue"; stage: Stage; query: null; why: string };

export function pickTopic({ rows, posts, queue, day }: { rows: GscRow[]; posts: string[]; queue: { query?: string | null }[]; day: number }): TopicPick {
  const stage = CYCLE[day % CYCLE.length];
  const slugs = new Set(posts);
  const targeted = new Set(queue.map((t) => t.query && norm(t.query)).filter(Boolean));
  const byQuery = new Map<string, { query: string; impressions: number; position: number; owned: boolean }>();
  for (const r of rows) {
    const q = norm(r.query);
    if (!usable(q) || funnelStage(q) !== stage) continue;
    const g = byQuery.get(q) ?? { query: r.query, impressions: 0, position: Infinity, owned: false };
    g.impressions = Math.max(g.impressions, r.impressions);
    g.position = Math.min(g.position, r.position);
    const slug = blogSlug(r.page);
    if (slug && slugs.has(slug) && r.position <= STRIKING.max) g.owned = true;
    byQuery.set(q, g);
  }
  const best = [...byQuery]
    .filter(([q, g]) => !targeted.has(q) && !g.owned)
    .map(([, g]) => g)
    .sort((a, b) => Number(striking(b.position)) - Number(striking(a.position)) || b.impressions - a.impressions)[0];
  if (best) return { kind: "new", stage, query: best.query, why: `top ${stage}-of-funnel search with no post (${best.impressions} impressions, best position ${best.position.toFixed(1)})` };
  return { kind: "queue", stage, query: null, why: `no ${stage}-of-funnel Search Console query to target, so the next queued topic` };
}
```

How to read it:

- A query is **owned** when a live post already ranks for it in the top 30. Writing another post for it would split ranking signals between two of your own pages (cannibalisation).
- A query is **targeted** when a queued, used or skipped topic already names it. Skipped queries from this run are added too, so a retry doesn't pick the same one.
- **Striking distance** (position 5 to 30) comes first: Google already associates you with the query, but no page answers it well. Then impressions break ties.

Two helpers feed the writer later:

- `postQueries(rows)` groups queries by the post that ranks for them, so the writer is told "these posts already own these searches; link them instead of covering them in depth".
- `openQueries(rows, stage)` lists other unowned queries of the same stage, which go in as possible angles and FAQ questions.

```ts
export function openQueries(rows: GscRow[], stage: Stage, limit = 10) {
  const owned = new Set(rows.filter((r) => blogSlug(r.page)).map((r) => norm(r.query)));
  const best = new Map<string, GscRow>();
  for (const r of rows) {
    const q = norm(r.query);
    if (!usable(q) || owned.has(q) || funnelStage(q) !== stage) continue;
    if ((best.get(q)?.impressions ?? -1) < r.impressions) best.set(q, r);
  }
  return [...best.values()].sort((a, b) => b.impressions - a.impressions).slice(0, limit);
}
```

### 4.5 Turning a query into a topic

The model gets the query, the stage, the list of allowed internal pages and the existing topics, and returns JSON: `slug`, `title`, `intent`, `related`, and for bottom-of-funnel posts an optional `tool`. The prompt says:

- The title answers the search, contains its main words, is at most 65 characters, and has no dashes.
- The slug is lowercase words joined by hyphens, at most 6 words, built from the search.
- `related` depends on the stage:
  - bottom: 1 or 2 use-case pages that do what the searcher wants, and no comparison page;
  - middle: 1 or 2 comparison pages, optionally one use-case page;
  - top: exactly one comparison page and one use-case page.
- Don't target the searches existing posts already rank for (listed), and don't paraphrase an existing topic (listed).

Then code validates it, because a model will sometimes ignore instructions:

```ts
const SLUG = /^[a-z0-9]+(-[a-z0-9]+)*$/;

const vs = t.related.filter((r) => allowed.includes(r) && r.startsWith("/vs/"));
const fo = t.related.filter((r) => allowed.includes(r) && r.startsWith("/for/"));
const related = { bottom: fo.slice(0, 2), middle: [...vs.slice(0, 2), ...fo.slice(0, 1)], top: vs.length && fo.length ? [vs[0], fo[0]] : [] }[pick.stage];
const problems = [
  !related.length && `related ${JSON.stringify(t.related)} doesn't fit the ${pick.stage} stage`,
  !SLUG.test(t.slug) && `bad slug ${t.slug}`,
  (/[–—]/.test(t.title) || t.title.length > 65) && `bad title ${t.title}`,
  (topics.some((o) => o.slug === t.slug || similarity(o.title, t.title) >= 0.6) || posts.some((p) => p.slug === t.slug)) && `"${t.title}" repeats an existing topic`,
].filter(Boolean);
if (problems.length) return null; // fall back to the queue
```

`allowed` is not a hand-kept list. The writer fetches your live `sitemap.xml` at the start of each run and keeps the paths of your use-case, comparison, tool and blog pages. A post can never link a page that doesn't exist yet, and new pages become linkable without a code change.

```ts
async function knownRoutes(posts: { slug: string; deleted: boolean }[]) {
  const res = await fetch(`${SITE}/sitemap.xml`, { signal: AbortSignal.timeout(20_000) });
  if (!res.ok) throw new Error(`sitemap.xml returned ${res.status}`);
  const paths = [...(await res.text()).matchAll(/<loc>([^<]+)<\/loc>/g)].map((m) => new URL(m[1].trim()).pathname.replace(/(.)\/$/, "$1"));
  const known = new Set(paths.filter((r) => /^\/(for|vs|blog|tools)(\/|$)|^\/(pricing|use-cases|privacy|about|contact)?$/.test(r)));
  for (const p of posts) if (!p.deleted) known.add(`/blog/${p.slug}`);
  if (![...known].some((r) => r.startsWith("/for/"))) throw new Error("sitemap.xml lists no /for pages");
  return known;
}
```

The last line is a sanity check: if the sitemap fetch returned an error page that happens to be 200, the run fails loudly instead of writing a post with no internal links.

### 4.6 De-duplication by title similarity

A Jaccard index over title words, with stop words removed and a crude plural strip:

```ts
const STOP = new Set(["a", "an", "the", "to", "how", "in", "for", "of", "and", "or", "your", "with", "into", "on", "vs", "what", "is"]);
const words = (s: string) =>
  new Set(s.toLowerCase().split(/[^a-z0-9]+/).filter((w) => w && !STOP.has(w)).map((w) => w.replace(/(?<=..)s$/, "")));

export function similarity(a: string, b: string) {
  const x = words(a), y = words(b);
  const shared = [...x].filter((w) => y.has(w)).length;
  return shared / (new Set([...x, ...y]).size || 1);
}
```

Two thresholds are used:

- **0.6 for topics.** "How many slides for a 10 minute presentation" and "How many slides for a 10 minute talk" score high and the second is rejected before any research money is spent.
- **0.75 for the final lint**, on both title and slug, against every other post. This is the last backstop.

It is deliberately simple: no embeddings, no extra API. It catches near-copies, not two different angles on one subject. The Search Console "owned" check and the "don't compete with these posts" prompt handle the semantic overlap.

### 4.7 The fallback queue and model-proposed topics

When there is no usable query (new site, missing secret, everything owned), the writer takes the oldest `queued` topic. When the queue is empty, one model call proposes 10:

- Topics your audience actually searches (how-to and comparison questions in your field) where a practical, sourced article would help.
- Each relates to one or two allowed pages, listed in `related`.
- None repeats or closely paraphrases an existing topic (the full list is in the prompt).
- Slug at most 6 words, title at most 65 characters with no dashes, intent one sentence.

Code keeps only proposals with a valid slug, no dash in the title, at least one allowed related page, and similarity under 0.6 to every existing and newly added topic. The survivors are inserted as `queued`.

```ts
const saveTopic = (env: Env, t: Topic, status: "queued" | "used" | "skipped", now = Date.now()) =>
  env.DB.prepare(
    "insert into blog_topic (slug, title, intent, related, tool, keyword, stage, status, createdAt, usedAt) values (?1, ?2, ?3, ?4, ?5, ?6, ?7, ?8, ?9, ?10) on conflict (slug) do update set status = ?8, usedAt = ?10",
  ).bind(t.slug, t.title, t.intent, JSON.stringify(t.related), t.tool ?? null, t.query ?? null, t.stage ?? null, status, now, status === "queued" ? null : now);
```

You can also seed the queue by hand. A list of 20 to 50 good topics written by someone who knows the field is worth more than any model proposal.

---

## 5. Research

### 5.1 Grounded search

The first research call asks the model to search several times with different queries and write a brief with five parts:

1. Key facts and numbers, each with where it comes from.
2. The step-by-step practice experts recommend.
3. Common mistakes.
4. Questions people search about this, with short answers.
5. Anything disputed, recently changed, or that differs between sources.

For comparison topics a sixth part asks, per product, for current prices, free plan limits, export formats and what it's best for, from the vendor's own pages only. The prompt includes today's date, so "current" means something.

The prompt also names the source policy up front: primary and authoritative sources only (a company's own pages about its own product, universities, government and standards bodies, recognised experts, established publications), and never one vendor's page about a competitor, review or affiliate sites, AI tool directories or "best tools" lists.

### 5.2 Keeping only sources that answer 200

The search tool returns URLs, but a URL in a grounding result is not proof the page exists or says anything. Every one is fetched:

```ts
const UA = "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/141.0.0.0 Safari/537.36";

const canonical = (u: string) => {
  const url = new URL(u);
  url.hash = "";
  for (const k of [...url.searchParams.keys()]) if (/^(utm_|gclid|fbclid|srsltid)/.test(k)) url.searchParams.delete(k);
  return url.toString();
};

async function check(url: string) {
  try {
    const res = await fetch(url, { redirect: "follow", headers: { "user-agent": UA, accept: "text/html,application/xhtml+xml,*/*" }, signal: AbortSignal.timeout(20_000) });
    if (res.status !== 200) return null;
    const final = canonical(res.url || url);
    if (!final.startsWith("https://")) return null;
    const html = (await res.text()).slice(0, 300_000);
    const title = /<title[^>]*>([^<]{3,300})<\/title>/i.exec(html)?.[1] ?? /property="og:title"\s+content="([^"]+)"/i.exec(html)?.[1] ?? new URL(final).hostname;
    return { url: final, title: cleanTitle(title, new URL(final).hostname) };
  } catch {
    return null;
  }
}
```

- Redirects are followed and the **final** URL is stored, with tracking parameters stripped. That's what gets cited.
- Only `https` and exactly `200` count. A 403 from a bot wall, a 404 or a soft redirect to a login page is dropped.
- The title comes from `<title>` or `og:title`, decoded, cut at the first " | " or " - " separator if what's left is at least 12 characters, and capped at 140. It becomes the link label in the Sources list.

Gemini returns grounding URLs as redirect links. Resolve them without following the target:

```ts
async function resolveRedirect(uri: string) {
  if (!uri.includes("grounding-api-redirect")) return uri;
  try {
    const res = await fetch(uri, { redirect: "manual", signal: AbortSignal.timeout(15_000) });
    return res.headers.get("location");
  } catch {
    return null;
  }
}
```

Each surviving source gets an id (`S1`, `S2`, ...). The grounding metadata says which sentence of the brief each source supports, so the brief is annotated in place:

```ts
async function research(run: Run, label: string, prompt: string, sources: Source[]) {
  const { text, grounding } = await gemini(run, label, { prompt, search: true });
  const chunks = grounding?.groundingChunks ?? [];
  const ids = await Promise.all(chunks.map(async (c) => {
    const real = c.web?.uri && (await resolveRedirect(c.web.uri));
    if (!real) return null;
    const ok = await check(real);
    if (!ok) return null;
    let s = sources.find((x) => x.url === ok.url);
    if (!s) sources.push((s = { id: `S${sources.length + 1}`, ...ok }));
    return s.id;
  }));
  let brief = text;
  for (const sup of grounding?.groundingSupports ?? []) {
    const seg = sup.segment?.text;
    const tags = [...new Set((sup.groundingChunkIndices ?? []).map((i) => ids[i]).filter(Boolean))];
    if (seg && tags.length && brief.includes(seg)) brief = brief.replace(seg, `${seg} [${tags.join(", ")}]`);
  }
  return brief;
}
```

With another provider: OpenAI and Anthropic web search tools return citations with URLs and the cited text span. Map those the same way. With a plain search API, fetch the top results yourself, pass their text to the model, and ask it to tag each fact with the result id.

### 5.3 Trust triage

A 200 page can still be a content farm. One structured call classifies every new source as keep or drop. Paraphrased:

> These pages were found while researching "<title>". Decide which are trustworthy enough to cite.
> **Keep:** a company's own page about its own product (pricing, help centre, docs); established companies' guides on practice in their own field (a CRM vendor on sales, a project tool on project management, a platform owner on its own apps); official documentation; universities and schools; government and standards bodies; professional bodies; established publications and news outlets.
> **Drop:** any page where one software vendor describes, prices or compares a competitor's product; "alternatives" and "vs" pages; review, affiliate and coupon sites; AI tool directories; "best tools" listicles; anonymous content farms; and regional copies of a page outside your target market (for example `/en-in/` paths for a US audience).
> Then the list: `S1: <title> <url>` per line.

The response schema is `[{ id, keep }]`. Dropped sources are removed from the list, and their `[S<n>]` tags are removed from the brief, so the writer never sees them as citable.

If fewer than 6 sources survive, a second grounded call asks for more authoritative sources not already in the list (official docs, university guides, recognised experts, established publications) and summarises what each says. It goes through the same check and triage.

After both passes, fewer than 3 sources is a hard stop: the run fails with "only N authoritative sources returned 200, need 3" and the topic is skipped.

### 5.4 Two more source types

- **Verified competitor data.** If the topic touches rivals, a file of rival facts (prices, limits, formats), each read from the vendor's own pages with a "checked" month, goes into the context with "prefer these over anything else, never contradict them". Its URLs are checked and added as sources. This is the same data your comparison pages use, so the blog and those pages never disagree.
- **The persona's knowledge file.** A curated list of insights with sources (section 6.3). The 3 sections whose keywords best match the topic are picked, each insight URL is re-checked at run time, and the ones still live are added with ids `K1`, `K2`, ...

---

## 6. Writing

### 6.1 The rules block

Every writing call starts with the persona file, then a fixed rules block. The real one covers, in this order:

- **Audience and tone.** US English, plain and practical, for a busy professional who has to get this done this week. Concrete steps, examples, numbers and templates beat general advice.
- **No dashes.** Never em or en dashes; use commas, colons, parentheses or two sentences. Number ranges as "3 to 5" or "3-5".
- **No filler.** The banned phrase list (same as the lint list), no exclamation marks, no rhetorical openers, no summary that repeats the article.
- **No invention.** Never invent statistics, studies, quotes, prices, dates or product features. Every outside fact comes from the brief or the knowledge notes and is cited inline with the exact URL from the source list. Cite 4 to 10 different sources, each at most twice. Link text is words, never a URL. At most one citation per sentence.
- **Statistics carry their citation in the same sentence.** Illustrative numbers are labelled as examples.
- **Use 3 to 6 knowledge insights** where they help, cited like any other source.
- **Product facts only from the facts file.** Name the product at most 3 times, only where it genuinely helps, honest about limits. The article must be useful to someone who never uses the product.
- **Other products: facts only from the brief or the verified rival data.** Be fair and say where each is a good fit.
- **Markdown only.** `##` and `###` headings in sentence case, paragraphs, lists, tables, blockquotes. No H1, no HTML, no code blocks, no images, no footnotes, no horizontal rules.
- **Internal links.** Link each required related page once with descriptive text; at most 2 other pages from the allowed list; no other internal paths.
- **FAQ.** End with `## FAQ`: 3 or 4 real questions as `###` headings, each answered in 1 to 3 sentences.
- **Length.** 1,100 to 1,400 words including the FAQ.

The prompt asks for tighter limits than the lint allows (1,100 to 1,400 words against a gate of 900 to 1,600; 4 to 10 sources against 3 to 12; each source twice against three times). Models drift. The gap means normal drift passes and only real failures need a repair.

### 6.2 The context block

Assembled once and appended to every writing call:

```
Title: <topic title>
Reader intent: <one sentence>
Required related pages (link each once): /for/x, /vs/y[, /tools/z]
Other pages you may link (at most 2):
- /for/...
- /vs/...

Research brief ([S<n>] marks which verified source supports a claim):
<brief>

Verified sources (cite only these, with the exact URL):
S1: <title> <url>
...

Research notes (vetted insights; [K<n>] or [S<n>] marks the verified source to cite):
### <section>
- <insight> [K1]

Product facts:
<facts.md>

[Verified competitor data, if relevant]

[Search notes: the target query and its stage, related open queries to use as
 angles or FAQ questions ("never stuff keywords"), and searches existing posts
 already rank for, with "link that post instead of covering it in depth"]
```

### 6.3 The four calls

1. **Outline** (structured output). Answer the reader's question early, then go step by step. 5 to 7 sections, each with a word budget and the specific points it will make (facts with their source ids, insights, steps, examples). Budgets add up to about 1,050 words, leaving about 200 for the FAQ. The schema also returns the meta description (120 to 155 characters, no dashes, says what the reader gets), a blurb for the index (at most 7 words) and the FAQ questions.
2. **Draft.** Write the article following the outline, keeping each section within its budget, starting with the first paragraph (no title).
3. **Self-check.** "You are a strict editor." The model gets the draft, its word count, the rules and the context, and returns the corrected full article. It is told to fix: claims not supported by the brief, notes, facts or rival data; statistics without a citation in the same sentence; unlabelled illustrative numbers; first-person anecdotes or claimed experience; links to unlisted URLs; missing or extra internal links; dashes; filler; repetition; anything unfair to a competitor; and length out of range.
4. **Citation check.** Code, plus one model call. Every sentence with a citation is checked against the actual text of the page it cites.

```ts
async function pageText(url: string) {
  try {
    const res = await fetch(url, { headers: { "user-agent": UA }, signal: AbortSignal.timeout(20_000) });
    const html = (await res.text()).slice(0, 1_000_000);
    return decode(html.replace(/<(script|style|noscript|svg)[^>]*>[\s\S]*?<\/\1>/gi, " ").replace(/<[^>]+>/g, " ")).replace(/\s+/g, " ").slice(0, 14_000);
  } catch {
    return "";
  }
}

async function verifyCitations(run: Run, body: string) {
  const claims: { sentence: string; urls: string[] }[] = [];
  for (const line of body.split("\n")) {
    for (const sentence of line.startsWith("|") ? [line] : line.split(/(?<=[.!?])\s+(?=[A-Z*[])/)) {
      const urls = [...sentence.matchAll(/\]\((https:\/\/[^)\s]+)\)/g)].map((m) => m[1]);
      if (urls.length) claims.push({ sentence: sentence.replace(/^\s*([-*]|\d+\.)\s+/, ""), urls });
    }
  }
  const urls = [...new Set(claims.flatMap((c) => c.urls))];
  const pages = await Promise.all(urls.map(pageText));
  const schema = { type: "array", items: { type: "object", properties: { i: { type: "integer" }, supported: { type: "boolean" }, fix: { type: "string" } }, required: ["i", "supported", "fix"] } };
  const { text } = await gemini(run, "verify-citations", {
    schema,
    prompt: `Check each numbered claim against the text of the page(s) it links to. supported is true only if the page states it (a faithful paraphrase is fine; numbers must match). If not, "fix" is a rewrite of the claim in the same markdown style that keeps only what the page supports, with the same link, or "" to drop the claim. For supported claims, fix is "". Never use em or en dashes.

${urls.map((u, i) => `PAGE ${u}\n${pages[i] || "(could not be read)"}`).join("\n\n")}

CLAIMS
${claims.map((c, i) => `${i}: ${c.sentence}`).join("\n")}`,
  });
  let out = body;
  for (const r of JSON.parse(text) as { i: number; supported: boolean; fix: string }[]) {
    const c = claims[r.i];
    if (!c || r.supported) continue;
    out = out.replace(c.sentence, r.fix.trim());
  }
  return out.replace(/^\s*([-*]|\d+\.)\s*$/gm, "").replace(/ {2,}/g, " ").replace(/\n{3,}/g, "\n\n");
}
```

- Table rows are checked as one claim; other lines are split into sentences.
- Each page's text is capped at 14,000 characters, which keeps the call affordable and covers most articles and docs pages.
- An unsupported claim is rewritten to what the page does say, or dropped. Empty list items left behind are removed.

Finally a cleanup pass in code runs before every lint:

```ts
function clean(md: string, sources: Source[]) {
  let out = md.trim().replace(/^```(?:markdown|md)?\n([\s\S]*?)\n```$/, "$1").trim(); // unwrap a fenced reply
  out = out.replace(/^# .*\n+/, "");                                                  // drop a stray H1
  out = out.replace(/<\/?[a-z][^>]*>/gi, "");                                          // drop raw HTML
  out = out.replace(/(\d)\s*–\s*(\d)/g, "$1-$2");                                 // 3–5 -> 3-5
  out = out.replace(/\s*\[[SK]\d+(?:,\s*[SK]\d+)*\]/g, "");                           // leftover [S1] tags
  out = out.replace(/\]\((https:\/\/[^)\s]+)\)/g, (m, url: string) => {               // normalise trailing slashes
    const hit = sources.find((s) => s.url === url || s.url.replace(/\/$/, "") === url.replace(/\/$/, ""));
    return hit ? `](${hit.url})` : m;
  });
  return out;
}
```

The post's `sources` field is then built from the sources actually cited in the final body, each with today's date as `checked`.

### 6.4 The persona file

The persona is an internal editorial voice, not a person. Posts are published under a real author (the founder), so the persona's first job is to stop the model from claiming experience that author doesn't have. Template:

```markdown
# <Persona name>: the blog's writing voice

<Persona name> is an internal editorial persona, not a person. He or she is a
<field> specialist who has read the research in knowledge.md and knows how
<the job your product does> gets done. Posts are published under <real author>,
so nothing may claim an experience, credential or event <real author> didn't have.

## Honesty (never break these)
- The persona never appears in a post. No name, no byline, no bio.
- No first-person anecdotes or invented experience ("in my 15 years", "our clients").
  Second person ("you"), or the editorial "we" for advice, never for events.
- No invented statistics, studies, quotes, prices or features. A number is cited
  or clearly labelled as an example.
- Opinions are stated as reasoned advice with a source, not personal history.
- A single expert's view is attributed to that expert, not presented as settled fact.

## Who it writes for
<One paragraph: who searched, what they need this week, what they don't want.>

## How it writes
- Answer the search question in the first two or three sentences.
- Short paragraphs, second person, <locale> spelling.
- Every section gives the reader something to do: a step, an order, a checklist,
  a before/after example.
- Examples use realistic numbers and are labelled as examples.
- Point out trade-offs and when advice doesn't apply.
- Fair to every tool, including competitors. The product at most 3 times, with limits.
- Pull 3 to 6 insights from the knowledge notes and cite them at the claim.

## What it always includes
- A direct answer near the top.
- At least one copyable artefact: an outline, a checklist, a template table.
- Common mistakes, each with its fix.
- An FAQ answering what people actually search.

## What it never does
- Em or en dashes.
- The banned filler phrases, exclamation marks, rhetorical openers,
  a closing summary that repeats the article.
- Overstating the product or putting down a competitor without a sourced reason.
- Padding to reach a word count.
```

### 6.5 The knowledge file

A curated research file is what lifts posts above generic advice. It is parsed by code, so the format is strict:

```markdown
# <Persona>'s knowledge base

Free text: what this file is, the source policy, how to add an insight.

## <Section title>
keywords: comma, separated, lowercase, phrases, that, match, topics
- **Short claim.** The insight with any number exactly as the page states it. ([Page title](https://example.org/page), checked 2026-01-15)
- ...
```

Each insight line must match `- <text> ([<title>](https://...), checked YYYY-MM-DD)`, and every section needs both keywords and insights, or the parser throws. Run the parser in a unit test so a bad edit fails CI instead of a run.

```ts
const INSIGHT = /^- (.+) \(\[([^\]]+)\]\((https:\/\/[^)\s]+)\), checked (\d{4}-\d{2}-\d{2})\)$/;

export function pickKnowledge(sections: KnowledgeSection[], topic: string, max = 3) {
  const q = topic.toLowerCase().replace(/[^a-z0-9]+/g, " ");
  const score = (s: KnowledgeSection) => s.keywords.filter((k) => new RegExp(`\\b${k.replace(/[^a-z0-9 ]/g, "")}s?\\b`).test(q)).length;
  return sections.map((s) => ({ s, n: score(s) })).sort((a, b) => b.n - a.n).slice(0, max).map((x) => x.s);
}
```

Source policy for the knowledge file: HTML pages from universities, standards bodies, official product docs, peer-reviewed papers, recognised practitioners and established publications. No vendor marketing, no affiliate lists. Every number must appear on the cited page. Use the final URL after redirects.

> **Real-world example:** the knowledge file has 16 sections (story and structure, cognitive load, charts, accessibility, pitch decks, investor updates and more) and over 250 cited insights, each with a keywords line per section. It is why posts can cite a specific finding instead of "studies show".

### 6.6 The facts file

One file is the only source of claims about your product. The model is told that if a claim isn't in it, leave it out. Template:

```markdown
# <Product> facts

The only facts a post may state about <Product>. Checked against the code on
<YYYY-MM-DD>. If a claim isn't here, leave it out.

## What it is
- One or two sentences: what it does, for whom, on which platforms.

## Price and limits
- Free plan: <exact limits>.
- Paid plan: <exact prices and limits>.
- Any discounts and who qualifies.

## Inputs
- What a user can start from, with exact caps.

## Features
- One line per feature people search for. Exact numbers.

## Outputs
- Formats, and what each contains or doesn't.

## Limits and honest caveats
- What it can't do. Where a competitor is the better fit.

## Pages to link
- Use cases: /for/...
- Comparisons: /vs/...
- Free tools: /tools/...
- Pricing, about, contact.
```

Keep it short and exact. Update it in the same change that alters a price, limit or feature. A stale facts file is the fastest way to publish false claims at scale.

---

## 7. The lint gates

### 7.1 A standalone `lintPost`

This is the real function with the product-specific values moved into a config. It has no dependencies. It returns a list of plain-English issues, which doubles as the repair prompt.

```ts
export type BlogSource = { title: string; url: string; checked: string };
export type BlogMeta = {
  title: string;
  description: string;
  blurb: string;
  date: string;     // YYYY-MM-DD
  updated: string;  // YYYY-MM-DD
  author: string;
  related: string[];
  sources: BlogSource[];
};

export type LintConfig = {
  brand: string;               // e.g. "YourProduct"
  author: string;              // the real byline
  relatedPrefix: RegExp;       // e.g. /^\/(for|vs|tools)\//
  words: { min: number; max: number };
  sources: { min: number; max: number; maxPerSource: number };
  maxBrandMentions: number;
  banned: RegExp[];
};

export const DEFAULT_LINT: Omit<LintConfig, "brand" | "author"> = {
  relatedPrefix: /^\/(for|vs|tools)\//,
  words: { min: 900, max: 1600 },
  sources: { min: 3, max: 12, maxPerSource: 3 },
  maxBrandMentions: 3,
  banned: [/in today's/i, /fast-paced/i, /\bdelve/i, /game[- ]changer/i, /\bunlock\b/i, /\belevate\b/i, /\bseamless/i, /revolutioni[sz]e/i, /look no further/i, /in conclusion/i, /\btapestry\b/i, /\bembark\b/i],
};

export type LintContext = {
  routes: Set<string>;                         // every live site path, plus this post's own
  others: { slug: string; title: string }[];   // every other post
  sourceUrls?: Set<string>;                    // every checked source for this run
};

const stripCode = (md: string) => md.replace(/```[\s\S]*?```/g, "").replace(/`[^`\n]*`/g, "");

export const wordCount = (md: string) =>
  stripCode(md).replace(/\]\([^)]*\)/g, "]").replace(/[#>*_|`[\]-]/g, " ").split(/\s+/).filter((w) => /\w/.test(w)).length;

export const linksIn = (md: string) => [...stripCode(md).matchAll(/\]\(([^)\s]+)(?:\s+"[^"]*")?\)/g)].map((m) => m[1]);

const STOP = new Set(["a", "an", "the", "to", "how", "in", "for", "of", "and", "or", "your", "with", "into", "on", "vs", "what", "is"]);
const words = (s: string) => new Set(s.toLowerCase().split(/[^a-z0-9]+/).filter((w) => w && !STOP.has(w)).map((w) => w.replace(/(?<=..)s$/, "")));
export function similarity(a: string, b: string) {
  const x = words(a), y = words(b);
  const shared = [...x].filter((w) => y.has(w)).length;
  return shared / (new Set([...x, ...y]).size || 1);
}

const escapeRe = (s: string) => s.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");

export function lintPost(slug: string, meta: BlogMeta, body: string, ctx: LintContext, cfg: LintConfig): string[] {
  const issues: string[] = [];

  const n = wordCount(body);
  if (n < cfg.words.min || n > cfg.words.max) issues.push(`word count ${n} is outside ${cfg.words.min}-${cfg.words.max}`);

  const text = [meta.title, meta.description, meta.blurb, ...meta.sources.map((s) => s.title), body].join("\n");
  if (/[–—]/.test(text)) issues.push("contains an em or en dash");
  if (/<\/?[a-z][^>]*>/i.test(stripCode(body))) issues.push("contains raw HTML");
  if (/^# /m.test(stripCode(body))) issues.push("body has an H1; the title is the only H1");
  for (const re of cfg.banned) if (re.test(text)) issues.push(`uses filler phrase ${re}`);

  const brand = (body.match(new RegExp(escapeRe(cfg.brand), "g")) ?? []).length;
  if (brand > cfg.maxBrandMentions) issues.push(`mentions ${cfg.brand} ${brand} times (max ${cfg.maxBrandMentions})`);

  if (!/^[a-z0-9]+(-[a-z0-9]+)*$/.test(slug)) issues.push("slug must be lowercase words joined by hyphens");
  if (meta.title.length > 70) issues.push("title is over 70 characters");
  if (meta.description.length < 70 || meta.description.length > 160) issues.push(`description is ${meta.description.length} characters (70-160)`);
  if (meta.blurb.split(/\s+/).length > 8) issues.push("blurb is over 8 words");
  if (meta.updated < meta.date) issues.push("updated is before date");
  if (meta.author !== cfg.author) issues.push(`author must be ${cfg.author}`);

  if (!meta.related.length) issues.push("related is empty");
  for (const r of meta.related) if (!cfg.relatedPrefix.test(r) || !ctx.routes.has(r)) issues.push(`related ${r} is not an allowed, live page`);

  const external = new Set<string>();
  const listed = ctx.sourceUrls ?? new Set(meta.sources.map((s) => s.url));
  for (const href of linksIn(body)) {
    if (href.startsWith("/")) {
      const path = href.split(/[?#]/)[0].replace(/(.)\/$/, "$1");
      if (!ctx.routes.has(path)) issues.push(`internal link ${href} has no route`);
    } else if (/^https:\/\//.test(href)) {
      external.add(href);
      if (!listed.has(href)) issues.push(`external link ${href} is not a listed, checked source`);
    } else issues.push(`link ${href} must be an https URL or a site path`);
  }
  if (external.size < cfg.sources.min) issues.push(`cites ${external.size} external sources inline (min ${cfg.sources.min})`);
  if (external.size > cfg.sources.max) issues.push(`cites ${external.size} external sources (max ${cfg.sources.max})`);

  const counts = new Map<string, number>();
  for (const [, label, href] of stripCode(body).matchAll(/\[([^\]]*)\]\(([^)\s]+)\)/g)) {
    if (/^(\/|https?:)/.test(label.trim()) || !label.trim()) issues.push(`link text "${label}" must be words, not a URL or path`);
    counts.set(href, (counts.get(href) ?? 0) + 1);
  }
  for (const [href, c] of counts) if (/^https:/.test(href) && c > cfg.sources.maxPerSource) issues.push(`cites ${href} ${c} times (max ${cfg.sources.maxPerSource})`);
  if (meta.sources.length < cfg.sources.min) issues.push(`lists ${meta.sources.length} sources (min ${cfg.sources.min})`);

  for (const o of ctx.others) {
    if (o.slug === slug) continue;
    if (similarity(o.title, meta.title) >= 0.75 || similarity(o.slug, slug) >= 0.75) issues.push(`near-duplicate of /blog/${o.slug}`);
  }
  return issues;
}
```

What each rule protects against:

| Rule | Threshold | Why |
|---|---|---|
| Word count | 900 to 1,600 | Below: thin. Above: padded, and the model starts repeating itself. |
| Dashes | none, in title, description, blurb, source titles and body | The most recognisable machine-writing tell, and a house style rule. |
| Raw HTML | none | The renderer drops it anyway; its presence means the model ignored the format. |
| H1 in body | none | The page renders the title as the only H1. |
| Banned phrases | 12 patterns | Filler that signals generic AI copy. Extend it with your own pet hates. |
| Brand mentions | 3 at most | A post that sells is not a post that ranks. |
| Slug | lowercase words and hyphens | Stable, readable URLs. |
| Title | 70 characters at most | Fits a search result. |
| Description | 70 to 160 characters | Shown as the snippet, not rewritten. |
| Blurb | 8 words at most | The index stays scannable. |
| Author | must equal the real byline | No invented authors. |
| Related | at least one, each an allowed and live page | Every post feeds the funnel. |
| Internal links | every path is a live route | No 404s from day one. |
| External links | every one is a checked source for this run | No hallucinated URLs. |
| Inline sources | 3 to 12 distinct | Real research, without turning into a link list. |
| Per-source citations | 3 at most | No leaning on one page for the whole post. |
| Link text | words, never a URL or path | Accessibility and anchor text quality. |
| Listed sources | at least 3 | The Sources section is never empty. |
| Near-duplicates | title or slug similarity 0.75 or more | The last backstop against cannibalisation. |

A minimal check to keep next to it (Vitest shown; any runner works):

```ts
import { expect, test } from "vitest";
import { DEFAULT_LINT, lintPost } from "./lint";

test("flags dashes, brand overuse and unlisted links", () => {
  const cfg = { ...DEFAULT_LINT, brand: "Acme", author: "Jane Doe" };
  const meta = { title: "How to plan a talk", description: "x".repeat(100), blurb: "Plan a talk", date: "2026-01-01", updated: "2026-01-01", author: "Jane Doe", related: ["/for/talks"], sources: [] };
  const body = "Acme Acme Acme Acme — see [this](https://example.org/a).";
  const issues = lintPost("plan-a-talk", meta, body, { routes: new Set(["/for/talks"]), others: [] }, cfg);
  expect(issues.some((i) => i.includes("em or en dash"))).toBe(true);
  expect(issues.some((i) => i.includes("mentions Acme 4 times"))).toBe(true);
  expect(issues.some((i) => i.includes("not a listed, checked source"))).toBe(true);
});
```

### 7.2 The repair loop

The issues list is the prompt. The model is told to fix those and change as little as possible otherwise. Length gets an explicit target, because "too long" alone makes models cut by a few words.

```ts
let body = clean(await verifyCitations(run, clean(reviewed, sources)), sources);
let meta = metaFor(body);
let issues = lintPost(topic.slug, meta, body, lintCtx, cfg);

for (let round = 1; issues.length && round <= 2; round++) {
  const n = wordCount(body);
  const length =
    n > 1400 ? `The article is ${n} words. Cut about ${n - 1250} words to land near 1,250: tighten every section, drop repetition and the weakest examples, keep facts and links.\n`
    : n < 1100 ? `The article is ${n} words. Add about ${1250 - n} words of concrete, sourced detail to land near 1,250.\n`
    : "";
  const fixed = (await gemini(run, `repair-${round}`, {
    prompt: `${length}Revise this article to fix every issue listed. Change as little as possible otherwise. Return the full corrected article in markdown and nothing else.
Issues:
${issues.map((i) => `- ${i}`).join("\n")}
Current word count: ${wordCount(body)}.
Meta description (fix it too if listed, return it on the first line as "DESCRIPTION: ..."; blurb as "BLURB: ..." on the second line):
DESCRIPTION: ${meta.description}
BLURB: ${meta.blurb}

${RULES}

Article:
${body}

${context}`,
  })).text;
  const d = /^DESCRIPTION:\s*(.+)$/m.exec(fixed)?.[1];
  const b = /^BLURB:\s*(.+)$/m.exec(fixed)?.[1];
  if (d) outline.description = d;
  if (b) outline.blurb = b;
  body = clean(fixed.replace(/^(DESCRIPTION|BLURB):.*\n/gm, ""), sources);
  meta = metaFor(body);
  issues = lintPost(topic.slug, meta, body, lintCtx, cfg);
}
if (issues.length) throw new Gate(`quality gates failed: ${issues.join("; ")}`);
```

Notes:

- The repair pass doesn't re-run the citation check. It is told to keep facts and links, and the lint still rejects any unlisted URL. If you see repairs adding claims, run `verifyCitations` again after each round.
- The target is the middle of the prompt range (1,250), not the edge of the lint range, so one repair usually lands inside.
- Description and blurb come back on tagged first lines so one call fixes the body and the meta together.
- Two rounds is the limit. A post that still fails after two targeted repairs usually has a deeper problem (too few sources, a topic that doesn't fit). Skip it.

---

## 8. Failure handling, time and cost

### 8.1 Two kinds of failure

```ts
class Gate extends Error {}
```

- A **Gate** is a quality refusal: too few sources, lint failed after repairs, no topic, slug already exists. The topic is marked `skipped` in the database so it isn't tried again.
- Any other error is infrastructure: a timeout, a 500 from the model, a bad sitemap. The topic stays `queued`, so a later run can try it.

Either way the run row gets `status = 'failed'`, the error message (up to 2,000 characters) and the cost so far.

```ts
async function finishRun(env: Env, id: number, r: { status: "running" | "published" | "skipped" | "failed"; error?: string; costUsd?: number }) {
  await env.DB.prepare("update blog_run set status = ?, error = ?, costUsd = ?, finishedAt = ? where id = ?")
    .bind(r.status, r.error?.slice(0, 2000) ?? null, r.costUsd ?? null, Date.now(), id).run();
}

async function failed(run: Run, s: State, id: number, e: unknown) {
  logError("blog run failed", e, { runId: id, gate: e instanceof Gate });
  await finishRun(run.env, id, { status: "failed", error: e instanceof Error ? e.message : String(e), costUsd: cost(run) });
  const t = run.topic;
  if (!t) return;
  s.skip.add(t.slug);
  if (t.query) s.skipQueries.add(t.query);
  if (e instanceof Gate) {
    await saveTopic(run.env, t, "skipped").run();
    t.status = "skipped";
  }
}
```

### 8.2 Time limits and the retry

```ts
const MINUTE = 60_000;
const RUN_LIMIT_MS = 12 * MINUTE; // hard limit for the whole run, under the runner's 15 minutes
const START_BY_MS = 6 * MINUTE;   // a retry only starts while a whole pipeline still fits

export async function runBlog(env: Env, runId: number, now = Date.now()) {
  const first = await env.DB.prepare(`select "trigger" from blog_run where id = ? and status = 'running'`).bind(runId).first<{ trigger: "cron" | "manual" }>();
  if (!first) return;
  const started = Date.now();
  const newRun = (): Run => ({ env, model: env.FAST_MODEL, today: isoDay(now), deadline: started + RUN_LIMIT_MS, usage: { input: 0, cached: 0, output: 0, searches: 0, calls: 0 } });
  if (!env.MODEL_API_KEY) return finishRun(env, runId, { status: "skipped", error: "MODEL_API_KEY is not set" });

  let s: State;
  try {
    s = await loadState(env);
  } catch (e) {
    logError("blog run failed", e, { runId, step: "load" });
    return finishRun(env, runId, { status: "failed", error: e instanceof Error ? e.message : String(e) });
  }

  for (let attempt = 0, id: number | null = runId; id; attempt++) {
    const run = newRun();
    try {
      const slug = await writeNew(run, s, id, now);
      await finishRun(env, id, { status: "published", costUsd: cost(run) });
      await indexNow(`${SITE}/blog/${slug}`);
      break;
    } catch (e) {
      await failed(run, s, id, e);
      if (attempt >= 1 || Date.now() - started > START_BY_MS) return;
      id = await claimRun(env, first.trigger);
    }
  }
}
```

- The deadline is shared by both attempts, so the retry can never push past 12 minutes.
- The retry claims a **new** run row. Each attempt has its own log line, error and cost.
- The retry is uncapped by the daily quota (the first attempt already counted against the slot) but still single-flight.
- The retry skips the failed topic and its query, so it picks the next one.

Every model call goes through one wrapper. It checks the budget and the clock, sets its own timeout, retries transient errors (network failure, 429, 5xx) twice with a 5 then 10 second wait, and records usage. Anything else throws at once.

```ts
async function gemini(run: Run, label: string, { prompt, search = false, schema }: { prompt: string; search?: boolean; schema?: object }) {
  if (cost(run) > BUDGET) throw new Error(`budget of $${BUDGET} reached before ${label}`);
  const body = {
    contents: [{ role: "user", parts: [{ text: prompt }] }],
    ...(search && { tools: [{ google_search: {} }] }),
    ...(schema && { generationConfig: { responseMimeType: "application/json", responseJsonSchema: schema } }),
  };
  for (let attempt = 0; ; attempt++) {
    const left = run.deadline - Date.now();
    if (left < 10_000) throw new Error(`out of time before ${label}`);
    let netError: unknown;
    const res = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/${run.model}:generateContent`, {
      method: "POST",
      headers: { "content-type": "application/json", "x-goog-api-key": run.env.MODEL_API_KEY },
      body: JSON.stringify(body),
      signal: AbortSignal.timeout(Math.min(240_000, left)),
    }).catch((e) => ((netError = e), null));
    if (!res?.ok) {
      const status = res?.status ?? 0;
      const reason = res ? (await res.text()).slice(0, 300) : String(netError);
      if (attempt < 2 && (status === 0 || status === 429 || status >= 500)) {
        await sleep(5000 * (attempt + 1));
        continue;
      }
      throw new Error(`${label}: model returned ${status} ${reason}`);
    }
    const data = (await res.json()) as {
      usageMetadata?: { promptTokenCount?: number; cachedContentTokenCount?: number; candidatesTokenCount?: number; thoughtsTokenCount?: number };
      candidates?: { content?: { parts?: { text?: string; thought?: boolean }[] }; finishReason?: string; groundingMetadata?: Grounding }[];
    };
    const u = data.usageMetadata ?? {};
    const output = (u.candidatesTokenCount ?? 0) + (u.thoughtsTokenCount ?? 0);
    run.usage.calls++;
    run.usage.input += u.promptTokenCount ?? 0;
    run.usage.cached += u.cachedContentTokenCount ?? 0;
    run.usage.output += output;
    const cand = data.candidates?.[0];
    run.usage.searches += cand?.groundingMetadata?.webSearchQueries?.length ?? 0;
    await recordUsage(run.env, { userId: null, kind: "other", model: run.model, inputTokens: u.promptTokenCount ?? 0, cachedInputTokens: u.cachedContentTokenCount ?? 0, outputTokens: output });
    const text = (cand?.content?.parts ?? []).filter((p) => !p.thought).map((p) => p.text ?? "").join("");
    if (!text) throw new Error(`${label}: empty response (${cand?.finishReason ?? "no candidate"})`);
    return { text, grounding: cand?.groundingMetadata };
  }
}
```

To swap providers, keep the signature (`prompt`, optional `search`, optional JSON `schema`; returns `text` plus whatever citation metadata the provider gives) and rewrite only the body. Every other function in the pipeline calls this one.

### 8.3 Cost cap and recorded spend

Each run keeps a running total of tokens and search queries, and every call checks it first:

```ts
const BUDGET = 1;                 // USD per run
const SEARCH_PRICE = 14 / 1000;   // your provider's price per grounded search query

const cost = (run: Run) =>
  (aiCost(run.model, { inputTokens: run.usage.input, cachedInputTokens: run.usage.cached, outputTokens: run.usage.output, images: 0 }) ?? 0)
  + run.usage.searches * SEARCH_PRICE;

// at the top of every model call:
if (cost(run) > BUDGET) throw new Error(`budget of $${BUDGET} reached before ${label}`);
```

After each call:

- Add prompt, cached and output tokens to the run. Count reasoning ("thinking") tokens as output; they are billed as output.
- Add the number of search queries the model ran (`groundingMetadata.webSearchQueries.length` on Gemini).
- Write one row to your usage table (model, tokens, cost, kind `other` or `blog`, no user id). The same table that tracks your product's AI spend then shows the blog's spend, by day.

`aiCost` is a lookup in a price table (USD per million input, cached input and output tokens, per model). Keep prices in one place and update them when the provider changes them. A model with no price should log a warning, not silently cost $0.

The run row stores the final cost, so you can see cost per published post and cost wasted on failures.

### 8.4 Other failure paths

| Failure | What happens |
|---|---|
| Model API key missing | Run marked `skipped` with the reason. |
| Search Console fails or the secret is missing | Warning logged, empty rows, queue fallback. Never blocks. |
| Sitemap unreachable or has no use-case pages | Run fails at load, before any model spend. |
| Runner can't be reached | Run marked failed at once ("Couldn't start the run"), admin gets a 502. |
| Runner crashes mid-run | Alarm retries once; `runBlog` exits if the row is no longer `running`. A row stuck as `running` is closed after 30 minutes by the next claim. |
| IndexNow refuses | Warning logged. The post is already live. |

---

## 9. Rendering

### 9.1 API

Two public, cached read routes. Posts are read at request time, so a new post is live within the cache window and no deploy is needed.

```ts
const CACHE = "public, max-age=300";
const toPost = (r: Record<string, unknown>) => ({ ...r, related: JSON.parse(String(r.related)), sources: JSON.parse(String(r.sources)) }) as BlogPost;

blog.get("/posts", async (c) => {
  const { results } = await c.env.DB.prepare(
    "select slug, title, description, blurb, date, updated, author, related, sources from blog_post where deletedAt is null order by date desc, updated desc, slug",
  ).all<Record<string, unknown>>();
  c.header("Cache-Control", CACHE);
  return c.json({ posts: results.map(toPost) });
});

blog.get("/posts/:slug", async (c) => {
  const row = await c.env.DB.prepare(
    "select slug, title, description, blurb, date, updated, author, related, sources, body from blog_post where slug = ? and deletedAt is null",
  ).bind(c.req.param("slug")).first<Record<string, unknown>>();
  if (!row) throw new HttpError(404, "Post not found");
  c.header("Cache-Control", CACHE);
  return c.json(toPost(row));
});
```

On the web side, validate the slug before calling the API (`/^[a-z0-9-]{1,100}$/`), and have the list call return `[]` on failure so the index and sitemap degrade instead of erroring.

### 9.2 Markdown to HTML, with raw HTML disabled

The body came from a model. Treat it as untrusted. With `marked`, an `html` renderer that returns an empty string drops every raw HTML token:

```ts
import { Marked } from "marked";

const esc = (s: string) => s.replace(/&/g, "&amp;").replace(/"/g, "&quot;").replace(/</g, "&lt;").replace(/>/g, "&gt;");

function render(slug: string, body: string) {
  const md = new Marked({
    gfm: true,
    renderer: {
      html: () => "",
      link({ href, tokens }) {
        const text = this.parser.parseInline(tokens);
        return href.startsWith("/")
          ? `<a href="${esc(href)}" data-cta="inline_link" data-cta-location="blog_${slug}">${text}</a>`
          : `<a href="${esc(href)}" rel="noopener">${text}</a>`;
      },
    },
  });
  return (md.parse(body, { async: false }) as string)
    .replace(/<table>/g, '<div class="blog-table"><table>')
    .replace(/<\/table>/g, "</table></div>");
}
```

- Internal links carry an analytics location, so you can see which posts send people into the funnel.
- Inline citations are normal followed links. The Sources list at the end uses `rel="nofollow noopener"`.
- Tables are wrapped so they scroll sideways on phones instead of breaking the layout.
- With another library, the equivalent is `markdown-it` with `html: false`, or `remark` without `rehype-raw`, plus `rehype-sanitize`.

### 9.3 Page layout

Title as the only H1, then byline, date, "Updated" when it differs, and reading time (`wordCount / 230`, at least 1 minute). Then the body, a numbered Sources list (title, host, "checked <date>"), cards for the `related` pages, and one closing CTA.

### 9.4 JSON-LD

One `@graph` with three types. The FAQ is extracted from the `## FAQ` section of the body, so the visible FAQ and the structured data can't drift apart.

```ts
const plain = (s: string) => s.replace(/\[([^\]]+)\]\([^)]+\)/g, "$1").replace(/[*_`]/g, "").trim();

function faqOf(body: string): [string, string][] {
  const out: [string, string][] = [];
  let inFaq = false;
  for (const t of new Marked().lexer(body)) {
    if (t.type === "heading" && t.depth <= 2) inFaq = /^(faqs?|frequently asked questions)$/i.test(t.text.trim());
    else if (inFaq && t.type === "heading" && t.depth === 3) out.push([plain(t.text), ""]);
    else if (inFaq && out.length && t.type !== "space") out[out.length - 1][1] = `${out[out.length - 1][1]} ${plain(t.raw)}`.trim();
  }
  return out.filter(([, a]) => a);
}

const AUTHOR_URL = `${SITE_URL}/about/<author-slug>`;
const url = `${SITE_URL}/blog/${p.slug}`;
const faq = faqOf(p.body);
const graph = [
  {
    "@type": "Article",
    headline: p.title,
    description: p.description,
    datePublished: p.date,
    dateModified: p.updated,
    author: { "@type": "Person", name: p.author, url: AUTHOR_URL },
    publisher: { "@id": `${SITE_URL}/#organization` },
    mainEntityOfPage: url,
    image: `${SITE_URL}/og.png`,
    inLanguage: "en-US",
    citation: p.sources.map((s) => s.url),
  },
  {
    "@type": "BreadcrumbList",
    itemListElement: [
      { "@type": "ListItem", position: 1, name: "Home", item: `${SITE_URL}/` },
      { "@type": "ListItem", position: 2, name: "Blog", item: `${SITE_URL}/blog` },
      { "@type": "ListItem", position: 3, name: p.title, item: url },
    ],
  },
  ...(faq.length ? [{ "@type": "FAQPage", mainEntity: faq.map(([q, a]) => ({ "@type": "Question", name: q, acceptedAnswer: { "@type": "Answer", text: a } })) }] : []),
];

// in the page: escape "<" so a title can't close the script tag
const ld = (o: object) => JSON.stringify(o).replace(/</g, "\\u003c");
// <script type="application/ld+json" dangerouslySetInnerHTML={{ __html: ld({ "@context": "https://schema.org", "@graph": graph }) }} />
```

- `publisher` points at the Organization node your home page already defines, by `@id`.
- `citation` lists the sources. It costs nothing and states plainly that the post is researched.
- The author `url` points at a real author page with a Person JSON-LD and `sameAs` links to real profiles. That page is what turns a byline into an E-E-A-T signal.

### 9.5 Head tags for dynamic routes

Use your framework's metadata API (`generateMetadata` in Next.js) to emit title, description, canonical, Open Graph (`og:type` `article`, published and modified times, author) and Twitter tags.

If your framework streams that metadata into `<body>` (vinext does), crawlers may miss it. The fix in the example is to wrap the worker's fetch handler and append the tags to `<head>` with `HTMLRewriter`, only for successful HTML responses on post URLs:

```ts
const POST = /^\/blog\/([a-z0-9-]{1,100})\/?$/;

async function withBlogHead(req: Request, res: Response, env: Env) {
  const slug = POST.exec(new URL(req.url).pathname)?.[1];
  if (!slug || req.method !== "GET" || res.status !== 200 || !res.headers.get("content-type")?.includes("text/html")) return res;
  try {
    const r = await env.API.fetch(`https://internal/api/blog/posts/${slug}`);
    if (!r.ok) throw new Error(`blog post ${r.status}`);
    const tags = blogHead((await r.json()) as BlogPost); // <title>, description, canonical, og:*, article:*, twitter:*, all attribute-escaped
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

Keep the streamed copies in the body as well; removing them breaks hydration in that framework. A failure here returns the page unchanged rather than an error. On other hosts, do the same in middleware or render the head on the server.

### 9.6 Sitemap, llms.txt and Markdown for agents

All three read posts at request time with a short revalidate (300 seconds):

```ts
export const revalidate = 300;

export default async function sitemap() {
  const posts = await listPosts();
  const pages = staticPages().map((p) =>
    p.url === `${SITE_URL}/blog` && posts.length ? { ...p, lastModified: posts.map((x) => x.updated).sort().at(-1) } : p,
  );
  return [...pages, ...posts.map((p) => ({ url: `${SITE_URL}/blog/${p.slug}`, lastModified: p.updated }))];
}
```

```ts
export async function llmsTxt() {
  const posts = await listPosts();
  if (!posts.length) return BASE; // the hand-written llms.txt
  return `${BASE.trimEnd()}\n\n## Blog\n\n${posts.map((p) => `- [${p.title}](${SITE_URL}/blog/${p.slug}): ${p.description}`).join("\n")}\n`;
}
```

If you serve Markdown to agents (see "AI search (GEO)" in the README), a post's Markdown is its stored body with the title on top. No conversion needed.

### 9.7 IndexNow

After a post is saved, ping IndexNow for that one URL. Bing, Yandex and others pick it up within minutes. Google ignores IndexNow and reads the sitemap.

```ts
async function indexNow(url: string) {
  try {
    const res = await fetch("https://api.indexnow.org/indexnow", {
      method: "POST",
      headers: { "content-type": "application/json; charset=utf-8" },
      body: JSON.stringify({ host: new URL(SITE).host, key: INDEXNOW_KEY, keyLocation: `${SITE}/${INDEXNOW_KEY}.txt`, urlList: [url] }),
      signal: AbortSignal.timeout(15_000),
    });
    if (res.status >= 400) logWarn("indexnow refused", { status: res.status, url, reason: (await res.text()).slice(0, 200) });
  } catch (e) {
    logWarn("indexnow failed", { url, error: errorInfo(e) });
  }
}
```

The key is the name and content of a text file you host at the site root. It isn't a secret in the password sense, but keep it in config rather than spreading it around.

---

## 10. Operating it

### 10.1 Admin view

Two owner-only routes are enough:

- `GET /api/admin/blog`: the last 30 runs (started, finished, status, slug, query, stage, cost, error, trigger), the live post count, and the next scheduled run.
- `POST /api/admin/blog/run`: start a run now. Returns 202 with the run id, or 409 while a run is going.

```ts
export function nextRunAt(now = Date.now()) {
  const today = BLOG_HOURS.map((h) => dayStart(now) + h * 3_600_000);
  return today.find((at) => at > now) ?? today[0] + DAY_MS;
}

admin.get("/blog", async (c) => {
  const [runs, posts] = await c.env.DB.batch([
    c.env.DB.prepare(`select id, startedAt, finishedAt, kind, status, slug, query, stage, costUsd, error, "trigger" from blog_run order by startedAt desc, id desc limit 30`),
    c.env.DB.prepare("select count(*) n from blog_post where deletedAt is null"),
  ]);
  return c.json({ runs: runs.results, posts: Number(posts.results[0]?.n ?? 0), nextRunAt: nextRunAt() });
});

admin.post("/blog/run", async (c) => {
  const runId = await startBlog(c.env, "manual");
  if (!runId) return c.json({ error: "A blog run is already running.", code: "BLOG_RUNNING" }, 409);
  return c.json({ runId }, 202);
});
```

A table of runs with the error text in full is the most useful screen. Most problems are obvious from the reason ("quality gates failed: word count 1712...", "only 2 authoritative sources returned 200").

### 10.2 Fixing or removing a bad post

Never delete the row. Edit it or soft-delete it.

```sql
-- fix a sentence, and say the post changed
update blog_post
set body = replace(body, 'the wrong sentence.', 'the right sentence.'),
    updated = '2026-11-01',
    updatedAt = strftime('%s','now') * 1000
where slug = 'the-slug';

-- take a post down; it 404s within the cache window and leaves the sitemap
update blog_post set deletedAt = strftime('%s','now') * 1000 where slug = 'the-slug';
```

- Only bump `updated` for a meaningful change. It feeds `dateModified` and the sitemap's `lastModified`; using it for typo fixes trains Google to ignore it.
- A soft-deleted slug stays reserved, so the writer won't reuse it.
- If the topic itself was a bad idea, also set its `blog_topic.status` to `skipped`.
- If a deleted post had traffic, redirect its URL to the closest page instead of letting it 404.

### 10.3 Weekly review routine

About 30 minutes a week:

1. **Read the newest posts in full.** At least two, more if the week had failures. Ask: would I publish this under my name? Does it answer the query in the first paragraphs? Is there something the reader can copy?
2. **Click five citations.** Does the page say what the sentence claims? The citation check is good, not perfect.
3. **Check every product claim against the facts file,** and the facts file against the product. Prices and limits change.
4. **Read the failed runs.** The same reason three times means a rule or prompt needs changing, not more retries.
5. **Search Console > Pages:** are the posts indexed? "Crawled, currently not indexed" on many posts is a quality signal from Google. Slow down and improve.
6. **Search Console > Performance, filtered to `/blog/`:** which posts get impressions, for which queries, at which positions.
7. **Look for cannibalisation:** two posts ranking for the same query. Merge them (keep the better one, redirect the other) or rewrite one to a different angle.
8. **Top up the queue** with a few hand-picked topics if the model's proposals are getting generic.
9. **Spot check the HTML:** JSON-LD validates in the Rich Results Test, head tags are present, no horizontal scroll at 360 px.

### 10.4 Metrics to watch

| Metric | Where | Healthy sign |
|---|---|---|
| Publish rate (published runs / all runs) | `blog_run` | Most runs publish. A sudden drop means a gate or a source broke. |
| Failure reasons | `blog_run.error` | Spread out. One dominant reason needs a fix. |
| Cost per published post | `blog_run.costUsd` | Stable and well under the cap. |
| Cost wasted on failures | `blog_run.costUsd where status = 'failed'` | Small. Large means failures happen late; move cheap checks earlier. |
| Indexed share | Search Console Pages report | Most posts indexed within a few weeks. |
| Impressions per post at 28 days | Search Console | Rising over time. Zero after two months is a candidate for rewrite or removal. |
| Queries owned | `postQueries` count | Rising. |
| CTR and position per post | Search Console | Positions moving toward the first page. |
| Clicks into the funnel | Analytics events with `data-cta-location` `blog_<slug>` | Some posts send sign-ups. Learn from those. |

Useful queries:

```sql
-- runs in the last 14 days by status, with spend
select status, count(*) runs, round(sum(costUsd), 2) usd
from blog_run
where startedAt >= (strftime('%s','now') - 14 * 86400) * 1000
group by status;

-- cost per published post
select round(sum(costUsd) / nullif(sum(status = 'published'), 0), 3) usd_per_post
from blog_run
where startedAt >= (strftime('%s','now') - 30 * 86400) * 1000;

-- most common failure reasons (first 60 characters)
select substr(error, 1, 60) reason, count(*) n
from blog_run where status = 'failed'
group by reason order by n desc limit 10;
```

---

## Checklist

- [ ] You have use-case, comparison or tool pages for posts to link into, and they're in the sitemap.
- [ ] Persona, knowledge and facts files written, reviewed by a human, and parsed in a unit test.
- [ ] Facts file checked against the product today.
- [ ] Three tables created; posts are soft-deleted only.
- [ ] Search Console API enabled, service account added as a Restricted user, JSON stored as a secret.
- [ ] Funnel keywords and the branded-query filter set for your market.
- [ ] Cron decides only whether to start; a background runner with at least 15 minutes does the work.
- [ ] Start guard is one statement: single-flight, daily quota, stale rows closed.
- [ ] Every source is fetched and must return 200 over https; triage drops vendor-on-rival pages, listicles and directories.
- [ ] Fewer than 3 kept sources stops the run.
- [ ] Citation check reads each cited page and fixes or drops unsupported claims.
- [ ] `lintPost` runs with your brand, author and thresholds; repair loop limited to 2 rounds.
- [ ] Hard run limit, per-call timeouts, a cost cap checked before every call, spend recorded per call and per run.
- [ ] Renderer drops raw HTML; JSON-LD has Article (author links to a real author page), BreadcrumbList and FAQPage.
- [ ] Head tags really are in `<head>` (view source, not dev tools).
- [ ] Sitemap and `llms.txt` read posts at request time.
- [ ] IndexNow pinged after each publish.
- [ ] Admin view of runs and a "run now" button.
- [ ] A weekly review on the calendar.

## Common mistakes

- **Running the pipeline inside the cron handler.** It gets cut off after about 30 seconds and leaves half-done rows. Hand off to a runner.
- **No single-flight guard.** A manual run plus a cron run, or a retried cron, writes two posts on the same topic.
- **Trusting URLs from the search tool.** Many 404, redirect to a login or block bots. Fetch every one and cite the final URL.
- **Letting the model cite anything it found.** Without triage, posts cite competitors' "alternatives" pages and affiliate listicles. That is both unreliable and a gift to rivals.
- **Product facts from the model's memory.** It will state last year's price with confidence. One facts file, kept current, is the only source.
- **A persona that claims experience.** "In my years as a consultant" under a founder's name is a false claim. The persona never speaks in the first person about events.
- **Asking for the exact lint limits in the prompt.** Normal drift then fails the gate every time. Prompt for a tighter range than you enforce.
- **Unbounded repair loops.** Each round costs money and rarely fixes a deep problem. Two rounds, then skip.
- **Skipping on infrastructure errors.** A timeout is not a bad topic. Mark only quality failures as skipped.
- **No cost cap.** A model that loops on searches can spend tens of dollars on one post. Check the budget before every call.
- **Rendering model output as raw HTML.** Disable it in the Markdown renderer, every time.
- **Metadata that crawlers can't see.** Check the served HTML for `<title>` and canonical inside `<head>`.
- **Writing for queries you already rank for.** Two of your own pages then compete. Check what posts already own before picking.
- **Volume first.** Ten posts a day with no review is the scaled content abuse pattern. Start at one, review weekly, and raise it only when the reviews stay clean.
- **Hard-deleting bad posts.** You lose the slug history and the writer may reuse the slug. Soft-delete, and redirect URLs that had traffic.
- **Never updating the facts or knowledge files.** The pipeline is only as current as they are.
