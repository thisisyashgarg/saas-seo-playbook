# 6. Off-page

Get your product listed, reviewed, recommended and linked on other people's sites, without paying for placements or getting banned.

## What you'll set up

- A launch kit with every asset and description you will paste into forms.
- One tracker file for every listing, launch, post and email.
- Listings on free launch sites and directories, with the badges some of them require.
- Profiles on AlternativeTo, SaaSHub, G2 and Crunchbase, with your rivals linked as alternatives.
- A ranked list of "best <category> tools" roundups and a short, personal pitch for each author.
- A routine for Reddit, Quora and niche communities that helps first and discloses always.
- A Medium presence built on founder stories, not cross-posted marketing.
- An automated cross-post to Dev.to with a canonical URL.
- An owned referral path: a "Made with <Brand>" link on public share pages.

## Why off-page matters early

A new domain has no links and no history. Two kinds of pages fill that gap:

1. **Pages Google already trusts** (directories, review sites, launch sites, big publications) that link to you or mention you.
2. **Pages AI assistants cite** when buyers ask for a tool. These are mostly listicles, review sites and community threads, not vendor sites.

> **Example from MakeDecks:** when 50 buyer prompts were run through Gemini with search, the cited sources were mostly vendor pages and "best AI presentation maker" listicles, and MakeDecks was named in none of them. Getting into those lists mattered more than adding more pages to its own site.

Set expectations: most directory links are `nofollow` or `ugc`. Their value is referral traffic, entity signals (more places that confirm you exist and say the same thing), and being visible to assistants. A few give followed links, often only after a review or a vote threshold.

## Step 1: Build a launch kit

Every form asks for the same things. Write them once, in `docs/launch/kit.md`, and copy from there. Take every fact from your facts file ([chapter 5](05-ai-search.md)).

| Asset | Spec | Notes |
|---|---|---|
| Logo | Square PNG, 512×512 and 240×240, transparent and on a solid background | Some sites crop to a circle. Keep the mark centred with padding. |
| Gallery images | 1270×760 PNG, 4 to 6 images | Real screenshots of the product doing its job, with one line of text on each. This size fits Product Hunt and most clones. |
| Cover or thumbnail | 1200×630 PNG | Doubles as your Open Graph image. |
| Tagline, 35 characters | | Some leaderboards cut anything longer. |
| Tagline, 60 characters | | Product Hunt style. |
| Description, 160 characters | | Directory cards and meta descriptions. |
| Description, 500 characters | | Most "About" fields. |
| Long description | 3 short paragraphs | What it does, who it's for, plans and prices. |
| Categories and tags | 3 to 5 | Pick the ones your rivals use, so you appear next to them. |
| Pricing line | One sentence | "Free for <N> <units> a month, no card. Pro is $<price>/month." |
| Founder bio | 1 line and 3 lines | Real role, real background. |
| Links | Site, X, LinkedIn, GitHub | Profiles you actually keep up. |
| Demo video | 30 to 60 seconds | Optional, but several launch sites rank listings with video higher. |

Fill in your own profile on each site too (bio, links). Several launch sites show the maker profile next to the listing.

## Step 2: One tracker file

Keep every off-page action in `docs/launch/tracker.md`. Without it you will forget which listing has the old price, which badge a site checks, and which thread you already answered.

```md
# Launch tracker

Status: **Live** (public now) · **Scheduled** (date booked) · **Submitted** (in a review queue) · **Draft** (saved, not public) · **Blocked** (needs something first) · **Skipped** (paid only or not a fit)

Last updated: <YYYY-MM-DD>

| Platform | Status | Date | Link | Notes / to do |
|---|---|---|---|---|
| <Site> | Submitted | <YYYY-MM-DD> | <listing URL> | Free queue, 1 to 4 weeks. Needs badge? Upvote threshold? Description last updated <date>. |
```

Rules:

1. Add a row the moment you submit, not when it goes live.
2. Note what each site requires to stay listed (a badge, a vote count, a backlink check).
3. Note the date you last updated the description, so a price change tells you what to edit.
4. Mark paid-only sites as Skipped with the price, so you don't check them again next month.

## Step 3: Launch sites and directories

Do the free ones first. Paid listings are rarely worth it before you have proof the channel converts.

Prices and rules below were checked in October 2026. They change often, so re-read each site's submit page on the day.

### Launch platforms

| Platform | Cost | What it requires | Tips |
|---|---|---|---|
| Product Hunt | Free | A maker account, gallery images, a scheduled launch day | Topics that match your rivals. Line up people who already use the product to look at it on the day, reply to every comment within the hour. Never ask for upvotes or use vote groups. |
| Hacker News (Show HN) | Free | An account in good standing | New accounts are often restricted. Build history with normal comments first. Write plainly, no marketing language, and be in the thread to answer. |
| Indie Hackers | Free | Product page is open; community posts need posting rights | Posting rights come from commenting, or a paid membership. Write a story with numbers, not an announcement. |
| Peerlist | Free | One upcoming launch per account | If another of your projects holds the weekly slot, you wait for the next free week. |
| Fazier | Free launch | Their badge on your site | Schedule a date; the badge must stay up. |
| MakerHunt | Free launch | A "Featured on" badge on your home page, then a badge check in their dashboard | Cover plus 4 gallery images, several categories. |
| TinyLaunch | Free launch | Review before scheduling | The backlink badge is only for top 3 finishers. |
| Uneed | Free waiting line | A minimum upvote score on launch day to stay published, and a higher one for a followed backlink | The free line can be weeks long. Plan the day like a small Product Hunt launch. |
| Tiny Startups | Free (paid deal slot optional) | A form | Popular launches go into their weekly email. Put a real number in the description. |
| Launch Llama | Free (paid instant option) | A form, about 4 weeks in the free queue | Ask to be considered for their newsletter. |
| StartupBase | Free with their badge, otherwise paid | Their badge on your site | Decide whether you want another badge in your footer. |
| Postmake | Free with their badge, otherwise paid | Their badge on your site | Same decision. |
| Microlaunch | Paid only | | Skip until a paid launch makes sense. |

### Directories and startup databases

| Platform | Cost | What it requires | Tips |
|---|---|---|---|
| Startup Stash | Free | A form | Pick the narrowest category that fits. |
| Launching Next | Free (paid upgrade optional) | A form with a simple check | About one business day of review. |
| Startups Gallery | Free | A form with a captcha | |
| SideProjectors | Free | A form | Mark "not for sale" unless it is. A few days of review. |
| F6S | Free | A company page | Fill tagline, description, logo, screenshot, markets and founding date. |
| 10words | Free queue | A 10-word description | The free queue is long. |
| Awesome Indie | Free queue | A form | Free slots can be months out. |
| PitchWall | Free | A form | Free review can take 30+ days. |
| LaunchIgniter | Free | A form with a math check | |
| Starter Story | Free | A founder interview form | Asks for revenue, team size and costs. Answer honestly or leave it blank. |
| Land-book | Free | Full-page desktop and mobile screenshots | A design gallery. Worth it only if your landing page is genuinely good. |
| Wellfound | Free | A company profile | Mostly hiring, but the company page is a trusted entity source. |
| Ventmark | Free | DNS TXT verification | |
| viberank | Free | A short tagline (about 35 characters) | Upvote leaderboard. |
| Future Tools | Free | A form; one submission, no resubmits within a week | AI tools only. |
| AI Valley | Free | A short form | Lead with the outcome. |
| The Rundown Supertools | Free directory (newsletter slots are ads) | An email with name, URL, category and one sentence | |
| AI Collection | Paid (small fee) | | Decide deliberately. |
| There's An AI For That | Paid review | | The only free route is their occasional maker thread. |
| Toolify | Paid | | Skip. |
| Futurepedia | Paid | | Skip. |

Also search "places to post your startup" lists. Many sites on them are dead, broken or paid only, so check each one and record the result as Skipped with a reason.

> **Example from MakeDecks:** one afternoon's sweep through a "places to post your startup" list found 22 sites. 11 were submitted, 5 needed the owner to sign in or finish a captcha, 4 were paid only or wanted a badge, and 2 were down or broken.

### How to submit well

1. Write a fresh description for each site. Some directories say they discard copied or AI-written text.
2. Tag your link with `?ref=<site>` (for example `https://example.com/?ref=futuretools`), so analytics shows which listing sends sign-ups. Some sites refuse query strings; then use the plain URL.
3. Choose the categories your rivals are in.
4. Pick "Freemium" or "Free" honestly. A listing marked free that leads to a paywall gets reported.
5. Record the listing URL in the tracker as soon as you have it.

## Step 4: Launch badges and your CSP

Some sites list you for free only if their badge is on your site, and they check it.

1. **Copy their exact link and image URL.** Their checker looks for that URL. Don't rewrite it, don't proxy the image, and don't add `nofollow` unless their instructions say you may.
2. **Put badges in one component**, so you can add and remove them in one place. Adapted from MakeDecks:

   ```tsx
   "use client";
   import { track } from "@/lib/analytics";

   const BADGES = [
     { site: "fazier", href: "https://fazier.com", img: "<exact badge image URL from their dashboard>", alt: "Fazier badge", width: 103, height: 44 },
     { site: "makerhunt", href: "<exact listing URL they give you>", img: "<exact badge image URL>", alt: "<Brand> on MakerHunt", width: 128, height: 44 },
   ] as const;

   export function LaunchBadges() {
     return (
       <div className="flex flex-wrap items-center gap-2">
         {BADGES.map((b) => (
           <a key={b.site} href={b.href} target="_blank" rel="noopener" onClick={() => track("launch_badge_clicked", { site: b.site })}>
             <img src={b.img} alt={b.alt} width={b.width} height={b.height} loading="lazy" />
           </a>
         ))}
       </div>
     );
   }
   ```

   Give each image its real width and height so the footer doesn't shift. If a site offers light and dark versions, render both and hide one with your theme class.
3. **Allow the badge host in your Content Security Policy.** If your CSP has `img-src`, a badge from a host that isn't listed is blocked silently, and the site's checker may still pass while visitors see a broken image. In `next.config.ts`:

   ```ts
   const csp = [
     "default-src 'self'",
     "img-src 'self' data: blob: https://fazier.com https://<other badge host>",
     // ...the rest of your policy
   ].join("; ");
   ```

4. **Check after deploy:** open the home page, confirm the badge renders in light and dark, then run the site's badge check in their dashboard.
5. **Keep the badge up** for as long as the listing needs it. Note that in the tracker, so nobody removes it during a footer redesign.

## Step 5: Review sites, and linking rivals as alternatives

Review and alternatives sites rank for "<rival> alternatives" searches and are cited by assistants. Listing is free on all four below.

| Site | What to do | Notes |
|---|---|---|
| AlternativeTo | Add your product, then add every rival you replace as an alternative | Free review queue, which can be slow. A paid fast review is offered; it isn't needed. |
| SaaSHub | Add the product, verify ownership, link alternatives | Free queue of 1 to 12 weeks. Manage the listing from its dashboard afterwards. |
| G2 | Create the product listing | Free. One G2 listing also covers Capterra, GetApp and Software Advice, since G2 now owns all three. Sign in with LinkedIn or a business email. G2 lists business software only, so describe the product as a business tool. |
| Crunchbase | Create the organization, then a person profile for the founder and link them | Free. Check for existing person profiles with the same name before you create one, and link the right one. Assistants and journalists look companies up here. |

### Verified revenue profiles (TrustMRR)

TrustMRR lists startups with revenue pulled straight from the payment provider, and its pages are read by founders, buyers and AI assistants (it publishes an `llms.txt` and a Markdown version of every startup page). A free profile is another trusted page about your product, even at $0 MRR.

1. Click **Add startup**, pick your payment provider (Stripe, Paddle, Lemon Squeezy, Polar, Dodo Payments, Creem, RevenueCat and others) and paste a **new read-only API key** made only for TrustMRR. Never reuse your app's own key, which can write.
2. Leave **List for sale** off unless you want buyers to contact you. The paid extras (a dofollow link, an AI boost, a sponsor spot) are optional.
3. In the dashboard, fill in everything: name, logo, description, website, up to 5 markets, tech stack, marketing channels, country, funding, team size, founded date, value proposition, problem solved, audience, pricing. Take the wording from `facts.md` so it matches your site.
4. Change the startup URL from the default (your name) to your product slug before you share it. The old link stops working.
5. Connect Google Analytics and Search Console (read-only OAuth) so traffic and search numbers show as verified. GitHub is optional: if you connect it, give the app access to selected repositories only.
6. Check the public Markdown page (`/startup/<slug>.md`) to confirm what assistants will read.

### Linking rivals as alternatives

On AlternativeTo and SaaSHub, an alternative link works both ways: when you mark <Rival A> as an alternative to your product, your product can appear on <Rival A>'s alternatives page. That page is where people searching "<Rival A> alternative" land.

1. List 6 to 16 rivals: the market leaders first, then the closest direct competitors.
2. Include the general tools people use instead of you (for a slide maker, that is PowerPoint and Google Slides).
3. Use the same description as everywhere else. Take it from `facts.md`.
4. Add the Crunchbase URL and any public company profile to your Organization `sameAs` ([chapter 5](05-ai-search.md)).

### Getting reviews

A listing with no reviews ranks poorly inside the site. After a user has had a good result (finished a project, came back a third time), ask them for a review with a direct link. Never pay for reviews or offer anything in exchange for a positive one; most review sites forbid incentives tied to the rating.

## Step 6: Roundup outreach

Find the "best <category> tools" articles that rank in Google and get cited by assistants, and ask their authors to consider you.

### Method

1. **Search** a dozen phrasings: "best <category> tools", "best free <category>", "<rival> alternatives", "<input> to <output> tool", "I tested <category> tools". Also take the sources your monthly AI check found ([chapter 5](05-ai-search.md)).
2. **Collect** each article in a table: publication, article URL, last updated date, which queries it ranks for, which tools it lists, the contact route, free or paid, status.
3. **Skip competitor blogs.** Many results are written by vendors in your category. They won't list a rival.
4. **Find a published contact route only:** an author page, a contact form, an email shown on the site, a LinkedIn linked from the article. Never guess an email address.
5. **Score** each target from 1 to 5 on authority, search position, freshness and whether they take suggestions. Pitch the top 15 personally.
6. **Read the live article** before writing. Bylines and dates change.
7. **Send 3 to 5 a day,** best first, from your own address. Do directory forms in between.
8. **Follow up once,** after 5 to 7 days, in the same thread. No third email.
9. **Log each send** in the table: Sent, Followed up, Replied, Listed, Declined. When one goes live, add it to the tracker.
10. **Don't pay for placement** without deciding it deliberately. Note which lists are affiliate-led.

> **Example from MakeDecks:** 37 roundup pages were checked and none mentioned MakeDecks. The market leader was on almost every independent list. But an independent tool topped one "25+ free tools" roundup, which showed that a small product can get in.

### Email template

Keep it under 120 words. One angle, one proof point, one offer.

```txt
Subject: A free <category> tool for your roundup

Hi <first name>,

I read your <article title> (updated <month year>). I built <Brand>, a <category>, and I think it fits next to <Tool they list> for readers who <specific need from their article>.

The angle: <the one real difference, stated as a fact: an export format, an input type, a free limit>.

The free plan is <limits>, <no watermark / no card>. Happy to set you up with a free Pro account for testing.

<Your name>, founder of <Brand>
https://example.com
```

Short version for forms with a character limit:

```txt
<Brand> turns <input> into <output>. <One differentiator>. Free, <key limit>.
```

Follow-up, once:

```txt
Just floating this back up in case it got buried. Happy to set up the Pro account whenever suits.
```

Have the free Pro account ready to switch on the same day someone says yes.

### Newsletters

AI and startup newsletters often have a free tool directory and a paid ad slot. Use the free submission, and treat ads as a separate, deliberate spend.

## Step 7: Communities

Reddit, Quora, Slack groups and forums can send your first users, and threads from them are cited by assistants. They also ban promotion fast.

### Rules

1. **Build history first.** For the first one to two weeks, post only helpful comments with no link, in subjects you know. Many communities filter new or low-karma accounts, and some expect self-promotion to be at most 1 in 10 of your activity.
2. **Help first.** If the comment is useless with the link removed, rewrite it.
3. **Always disclose.** "I built this" or "I'm the developer", in the same comment, right next to the link. Some communities also want you to say whether it's free.
4. **One mention a day across all communities** when you're starting, never two in the same community in a day. Spread them out.
5. **One link per comment.** Link the specific page that answers the question (a use-case or comparison page), otherwise the home page.
6. **Tag every link** with `?ref=<source>`, for example `?ref=reddit` or `?ref=quora`.
7. **Never manipulate votes.** No asking friends to upvote, no second accounts, no replying to yourself, no posting the same text twice. Reddit bans for this site-wide.
8. **Reply to follow-ups within a day,** including criticism. A calm answer to a hostile comment does more than the original post.
9. **If a comment is removed, don't repost it.** Note it in the tracker and move on.
10. **Use the designated places** for anything that looks like an announcement: weekly self-promotion threads, monthly "share your startup" threads, feedback threads. Read each community's rules page on the day you post.
11. **Be honest about limits** when asked: what the product doesn't do, and what happens to uploaded data.
12. **On Quora,** set your credential to "Founder of <Brand>", keep the disclosure line in every answer that links, and answer only a few questions a day.
13. **Track it.** Record each post in the tracker with its date, and note any thread that sends sign-ups.

### Finding threads

1. Search the community for the questions on your buyer prompt list.
2. Prefer recent threads with real discussion. Old threads may be archived (Reddit archives at about six months).
3. Expect removals in "best tool" threads. Many are vendor spam and get deleted by moderators.

> **Example from MakeDecks:** of 72 recent candidate threads checked, 37 had already been removed by moderators. Most "best AI presentation tool" posts were vendor spam. The useful threads were specific questions ("how do I turn this PDF into slides") with real discussion.

### One channel a day

Plan new channels as one a day for two or three weeks, not a burst. Each community sees one post from you, you can stay in the comments, and you can tell from `?ref=` which channel worked.

| Day | Channel | How | Status |
|---|---|---|---|
| 1 | <directory> | Email | |
| 2 | <directory> | Form | |
| 3 | <Q&A site> | One answer | |
| 4 | <community> | Weekly self-promotion thread | |

### How `?ref=` shows up

- Product analytics tools that record the full page URL (PostHog, for example) show `ref` in the current URL of the first pageview. Filter or break down by it.
- GA4 ignores `ref` for attribution. If you rely on GA4 for sources, use `utm_source=<source>&utm_medium=referral` instead, or both.
- Keep a fixed list of `ref` values in the tracker so you don't end up with `reddit`, `Reddit` and `rdt`.

## Step 8: Medium

Medium is a place to reach readers. It is not a source of backlinks and not a home for automated posts.

### What to know first

- **No API.** Medium issues no new API tokens and allows no new integrations (closed on 2025-01-01). Posting is manual, in the editor or through Import a story.
- **Links are `ugc nofollow`.** They pass little or no ranking credit. The value is Medium's readers, referral clicks, links from people who read the piece, and AI assistants that cite Medium.
- **AI-written text gets network-only distribution:** shown to your followers only, never recommended. Medium's guidelines also list content marketing and stories mainly meant to gather sign-ups as low-value. Never cross-post an automated blog as is.
- **What gets distributed** is first-person stories from real experience, with real numbers and takeaways. Medium's curators pick less than 1% of stories for Boost, and partner publications nominate about half of those.

### 1. Set up the profile

1. Sign up with the founder's real name and photo. Readers follow people, not companies.
2. Write a short bio that says who you are and what you build, for example: "Founder of <Brand>. Writing about building a small SaaS."
3. Follow and read a few publications in your area before you submit to them. You'll learn what they publish.
4. Optional: a publication of your own under the brand name. Start with your personal profile; a brand publication has no readers at first.

### 2. Import a story (with a canonical)

Use this for posts that a person wrote or rewrote and that already live on your site.

1. **Wait until Google has indexed the original.** Open URL Inspection in Search Console for the post's URL and confirm "URL is on Google". Importing first risks Medium's copy being treated as the original.
2. In Medium, open your profile menu and choose **Import a story**, then paste the URL of the original post.
3. Medium pulls the text and images into a draft. Fix formatting: headings, code blocks, image captions, tables (which often break).
4. Remove anything that reads as a teaser. The whole post goes on Medium.
5. Keep at most one link to your site, where it actually helps the reader.
6. Publish. Imported stories are backdated to the original date and set `rel=canonical` to your original URL.
7. **Verify the canonical:** open the published Medium page, view the page source, and search for `rel="canonical"`. It must point at your URL, not at Medium.
8. Record it in the tracker with the Medium URL.

### 3. Write founder stories

These are the posts that can be distributed beyond your followers.

1. Pick one thing that happened while building: a launch, a mistake, a pricing change, a channel that worked, a number that surprised you.
2. Write in first person, from your own experience. Use real numbers ("83 of about 300 sign-ups"), real dates and what you'd do differently.
3. Structure: the situation in two or three sentences, what you tried, what happened with numbers, what you learned as 3 to 5 takeaways a reader can use.
4. Mention the product only where the story needs it. Link to your site once, where it helps.
5. No "read the full post on our site" teasers. No paywall on product stories.
6. Write it in Medium as a new, unpublished draft if you plan to submit it to a publication (next step).

### 4. Submit to publications

Publications have their own readers, and some of them nominate stories for Boost.

1. Find publications that publish founder stories and whose editors nominate for Boost. Examples: Entrepreneur's Handbook (founder stories with takeaways), Better Marketing (case studies, no personal CTAs) and Start it up (solo builders).
2. Read each publication's submission guidelines page. Most want an unpublished draft and have rules on links and calls to action.
3. Request to become a writer through the method on their guidelines page.
4. Once added, open your unpublished draft, choose to submit it to the publication, and wait. Don't publish elsewhere in the meantime.
5. If it's declined or there's no answer within the time they state, submit it to the next one, or publish it yourself.

### 5. Disclose AI use

If any part of the text was written by AI, say so in the first two paragraphs, for example: "I used an AI assistant to draft parts of this post and edited it myself." Expect such stories to reach your followers only.

> **Example from MakeDecks:** after this research, the plan became: founder stories submitted to Boost-nominating publications, imports only for human-edited posts with a verified canonical, and automation pointed at Dev.to and Hashnode instead of Medium.

Sources (Medium help center): https://help.medium.com/hc/en-us/articles/213480228, https://help.medium.com/hc/en-us/articles/214550207, https://help.medium.com/hc/en-us/articles/360033930293, https://help.medium.com/hc/en-us/articles/22576852947223, https://help.medium.com/hc/en-us/articles/360006362473

## Step 9: Automated cross-posting to Dev.to

Dev.to (DEV, built on Forem) has a write API, accepts a `canonical_url`, and its links carry `noopener noreferrer` without `nofollow`. If your audience includes developers, cross-post technical posts there.

Rules before you automate:

- Cross-post only posts that are good for DEV readers (technical, practical). Not every marketing post.
- Set `canonical_url` to your original, so search engines credit your site.
- Wait until the original is indexed, as with Medium.
- DEV has guidelines for AI-assisted articles. Read them and disclose AI help in the post.
- Start with drafts (`published: false`) and publish by hand after a read-through.

### Setup

1. Sign in to DEV and create an API key under Settings, Extensions.
2. Store it as a secret (for example `DEVTO_API_KEY` in CI or your worker's secrets). Never commit it.

### Script

`scripts/devto-crosspost.mjs`. It creates a draft for one post, and skips posts that already exist on DEV with the same canonical URL.

```js
// node scripts/devto-crosspost.mjs <slug>
// Needs DEVTO_API_KEY. Creates an unpublished DEV draft with canonical_url pointing at the original.
const SITE = "https://example.com";
const API = "https://dev.to/api";
const key = process.env.DEVTO_API_KEY;
if (!key) throw new Error("DEVTO_API_KEY is not set");

const headers = { "api-key": key, "content-type": "application/json", accept: "application/vnd.forem.api-v1+json" };

// Replace with however you load a post: a database call, your public API, or a Markdown file.
async function loadPost(slug) {
  const res = await fetch(`${SITE}/api/blog/posts/${slug}`);
  if (!res.ok) throw new Error(`post ${slug} returned ${res.status}`);
  return res.json(); // { title, description, body, tags?, image? } with body in Markdown
}

// DEV renders links relative to dev.to, so make every site-relative link absolute.
const absolutize = (md) => md.replace(/\]\((\/[^)\s]*)\)/g, `](${SITE}$1)`);

async function existingCanonicals() {
  const urls = new Set();
  for (let page = 1; ; page++) {
    const res = await fetch(`${API}/articles/me/all?page=${page}&per_page=100`, { headers });
    if (!res.ok) throw new Error(`listing my articles returned ${res.status}`);
    const items = await res.json();
    for (const a of items) if (a.canonical_url) urls.add(a.canonical_url);
    if (items.length < 100) return urls;
  }
}

const slug = process.argv[2];
if (!slug) throw new Error("usage: node scripts/devto-crosspost.mjs <slug>");
const canonical = `${SITE}/blog/${slug}`;

if ((await existingCanonicals()).has(canonical)) {
  console.log(`already on DEV: ${canonical}`);
  process.exit(0);
}

const post = await loadPost(slug);
const res = await fetch(`${API}/articles`, {
  method: "POST",
  headers,
  body: JSON.stringify({
    article: {
      title: post.title,
      body_markdown: `${absolutize(post.body.trim())}\n\n*Originally published at [${SITE.replace("https://", "")}](${canonical}).*\n`,
      published: false,
      canonical_url: canonical,
      description: post.description,
      tags: (post.tags ?? []).slice(0, 4).map((t) => t.toLowerCase().replace(/[^a-z0-9]/g, "")).join(","),
      ...(post.image ? { main_image: post.image } : {}),
    },
  }),
});
if (!res.ok) throw new Error(`DEV returned ${res.status}: ${(await res.text()).slice(0, 300)}`);
const article = await res.json();
console.log(`draft created: ${article.url ?? article.id}`);
```

Notes:

- The request shape (`POST https://dev.to/api/articles`, an `api-key` header, a body wrapped in `article`, `tags` as a comma-separated string of up to 4, `canonical_url`, `published`) follows the Forem API v1 reference. Tags are lowercase letters and digits only, hence the cleanup.
- `GET /api/articles/me/all` lists your published and unpublished articles. `GET /api/articles/me` lists only published ones, so it would miss drafts and create duplicates.
- **This script has not been run against a live DEV account.** Run it once by hand, check the draft (canonical URL in the post settings, links, images, code blocks), then wire it into CI.
- To publish later from code, `PUT https://dev.to/api/articles/{id}` with `{ "article": { "published": true } }`. Publishing by hand after a read is safer.
- In CI, run it after a post is published and indexed, not on every deploy. Handle a 429 (too many requests) by waiting and retrying once.

### Hashnode

Hashnode lets you run a blog on your own custom domain. On a custom domain its links use `ugc` without `nofollow`, and posts have an "originally published at" canonical URL setting. It has a GraphQL API if you want to automate it the same way. If you already run your blog on your own site, use Hashnode the way you use DEV: cross-post selected posts with the canonical set to your original.

## Step 10: Referral paths you own

Your users' shared work is seen by people who don't know you yet. A small credit link turns some of them into visitors.

### "Made with <Brand>" on public share pages

1. Add a small, quiet link in the header or footer of every public share page (a shared document, a public dashboard, a hosted form).
2. Point it at your home page with `?ref=madewith`.
3. Track the click as an event.
4. **Never put it inside the user's exported files** (PDF, PPTX, images, downloads). Users own those, and a brand stamp there feels like a watermark.
5. Don't make it a toggle that defaults off. If share pages are on your own domain, the link is a referral path, not a backlink, so it costs the user nothing.

Adapted from MakeDecks's share viewer:

```tsx
<header className="flex h-12 shrink-0 items-center gap-3 border-b px-3 sm:px-4">
  <h1 className="min-w-0 truncate text-sm font-semibold">{doc.title}</h1>
  <a
    href="https://example.com/?ref=madewith"
    onClick={() => track("made_with_clicked", {})}
    className="ml-auto shrink-0 text-xs text-muted-foreground hover:text-foreground"
  >
    Made with <Brand>
  </a>
</header>
```

Share pages themselves should send `X-Robots-Tag: noindex` ([chapter 1](01-technical-foundations.md)), so users' private content never shows in search.

### Embeds

If you have a free tool that other sites would embed (a calculator, a timer, a widget), offer an iframe snippet that repeats a "<Tool> by <Brand>" credit link outside the iframe. A link inside an iframe belongs to your page, not theirs; the one outside is a real link from their page. See [chapter 3](03-page-programs.md) for the embed route and its frame headers.

## Checklist

- [ ] `docs/launch/kit.md` has the logo, gallery images, taglines at 35, 60 and 160 characters, descriptions, categories, pricing line and founder bio.
- [ ] `docs/launch/tracker.md` has a row for every submission, with status, date, link, requirements and the date the description was last updated.
- [ ] Free launch sites and directories submitted, with `?ref=<site>` links where allowed.
- [ ] Required badges live in one component, with exact URLs, real sizes and their hosts in the CSP `img-src`; badge checks passed.
- [ ] AlternativeTo, SaaSHub, G2 and Crunchbase listings created, rivals linked as alternatives, Crunchbase in your Organization `sameAs`.
- [ ] A ranked roundup table, top 15 pitched personally, one follow-up each, every send logged.
- [ ] Community posts follow the rules: history first, disclose, one mention a day, one link, `?ref=` tags, logged.
- [ ] Medium: profile set up, imports only after indexing with the canonical verified, founder stories submitted to publications, AI use disclosed.
- [ ] Dev.to script run once by hand and checked before it runs in CI; `DEVTO_API_KEY` stored as a secret.
- [ ] "Made with <Brand>" link with `?ref=madewith` on share pages, tracked, and never in exported files.

## Common mistakes

- **Paying for listings before free ones are done.** Most paid directories send little traffic to a new product.
- **Copying the same description everywhere word for word.** Some directories discard it; write a fresh version from the same facts.
- **Forgetting old listings when prices change.** Assistants then quote the old price. The tracker's "description updated" column prevents this.
- **Rewriting or proxying a badge URL.** The site's checker fails and the listing is pulled.
- **Adding a badge without updating the CSP.** It renders broken for visitors.
- **Pitching competitor blogs.** They won't list you. Check who owns the site first.
- **Guessing emails.** Use only routes the site publishes.
- **Following up more than once.** One follow-up is polite; two is spam.
- **Posting links in a new account.** It gets filtered or banned. Build history first.
- **Undisclosed mentions, second accounts or vote requests.** These get your account and sometimes your domain banned.
- **Cross-posting automated posts to Medium.** They get follower-only distribution and read as content marketing.
- **Importing to Medium before Google indexes the original.** The copy can be treated as the source.
- **Cross-posting without `canonical_url`.** You create duplicates that compete with your own post.
- **Branding the user's exported files.** It feels like a watermark and hurts trust. Keep the credit on your own share pages.
