# 8. Launching in ChatGPT and Claude

Put your product inside ChatGPT and Claude as an app, list it in their directories, and get it picked when a user asks for the job it does.

Chapter 5 is about assistants *talking about* your product. This chapter is about assistants *using* it: the user stays in the chat, and the assistant calls your product's tools for them.

> Facts in this chapter were checked on 2026-11-11 against the vendors' own docs, linked inline. Both platforms change their rules often. OpenAI renamed its "app directory" to the "Plugin directory" in July 2026, and many older Apps SDK pages are partly stale. Re-check the linked pages before you submit.

## What you'll set up

- One remote MCP server that works in both ChatGPT and Claude.
- OAuth that both platforms accept: CIMD and DCR, PKCE, the right metadata and the right redirect URIs.
- Tools that pass review: plain verb names, narrow descriptions, all three annotations, specific errors.
- A golden prompt set to measure whether the assistant picks your tool when it should, and leaves it alone when it shouldn't.
- Listing copy and review material for both directories: name, descriptions, starter prompts, test account, test cases, demo video.
- An entry in the official MCP Registry, under your own domain.
- A weekly routine that turns tool-call data into metadata changes.

## Why this is a channel

A directory listing is a store page inside the product your buyers already use. The assistant can also recommend apps in the chat. ChatGPT shows a "ChatGPT needs to use a plugin" card. Claude shows "Suggested Connectors" when a connector fits the user's task.

Set your expectations from what developers report, not from the launch posts:

- **Approval is not placement.** On ChatGPT, most published apps are found only by exact-name search or a direct link. In one tracker's count, none of 2,758 apps published in two weeks of October 2026 reached the Popular, New & Noteworthy or Small Business shelves ([theopenapps.com](https://theopenapps.com/blog/how-to-submit-a-chatgpt-app), 2026-10-25). Developers on OpenAI's forum report "~50 people/day" and "Discovery is non-existent" ([forum thread](https://community.openai.com/t/when-do-our-apps-surface-to-users-clarification-requested/1379612), May 2026).
- **Connected apps aren't always called.** Developers report that ChatGPT often answers without calling an installed app unless the user @-mentions it ([forum](https://community.openai.com/t/chatgpt-apps-reject-with-same-feedback/1376488), March 2026; [3minapi](https://3minapi.com/blog/building-chatgpt-app-with-apps-sdk), April 2026).
- **Review is slow on both.** Expect 2 to 8 weeks per review cycle on ChatGPT, with reports of 60 days or more. Expect 3 to 9 weeks on Claude. Neither platform publishes an SLA.
- **Most categories already have big names.** In e-signature, for example, DocuSign, SignNow, Jotform Sign, SignWell, Lumin and PandaDoc were already in Claude's directory in November 2026.

So the listing is the start, not the launch. Plan to bring the first users yourself (Step 10). Usage is what both platforms rank on.

## How ranking works

Neither platform publishes a full formula. This section separates what the vendors state, what their staff have said, and what developers have measured. Re-check it: OpenAI says ranking is still changing.

### Claude: directory rank is a published number

Claude's publisher docs define the rank exactly ([manage your listing](https://claude.com/docs/connectors/building/managing-your-listing)):

> Directory rank: your position among published directory servers, highest first, ranked by the number of distinct Claude accounts that sent your server any MCP message in the last 30 days. Every MCP message counts toward the ranking, including `initialize` and `tools/list`, and it includes accounts whose connection attempt never finished authenticating.

- **Rank = reach, not quality.** It counts accounts that connected, even if they never finished signing in or called a tool. Every person you send to "Add to Claude" moves you up.
- **Trending tag:** the top 10 servers by recent growth in the same count.
- **The directory page** sorts by Recommended, Most popular, Trending, Newest and A–Z, with Top, Trending and New sections. It shows no ratings or install counts ([claude.com/connectors](https://claude.com/connectors)).
- **Directory search** seems to match the name, the one-liner and tool names, not the long description. A vendor's listing matched "deploy" only through a tool name, and did not match "website", a word in its description ([trytofu.ai](https://trytofu.ai), 2026-11-06, third-party). Put your category words in the name, the one-liner and the tool names.
- **No paid placement.** "There are no paid placements or sponsored answers" ([Anthropic](https://claude.com/resources/articles/connectors-for-everyday-life), 2026-04-23).

### Claude: Verified matters more than rank for suggestions

The docs say every directory entry is included in Suggested Connectors automatically ([directory](https://claude.com/docs/connectors/directory)). The evidence says otherwise, for now:

- **GitHub issue [anthropics/claude-ai-mcp#954](https://github.com/anthropics/claude-ai-mcp/issues/954)**, opened 2026-08-29, still open: a Community connector with 3,378 tool-call users in 30 days was the #1 directory result for "ads". In-chat suggestions returned nothing for it, even for its exact brand name.
- **An independent reproduction in the same issue** (2026-10-07, free account, fresh chat per prompt): 0 of 9 Community connectors found, 7 of 7 Verified connectors found.
- **A second report** ([#762](https://github.com/anthropics/claude-ai-mcp/issues/762), for a different connector) was closed by an Anthropic maintainer as a duplicate of #954. Anthropic has acknowledged the report but had not explained or fixed it as of 2026-11-11.
- **A third developer** saw the same thing: a Community connector ignored, three Verified ones suggested ([dev.to](https://dev.to/tantuc), 2026-08-04).

What Verified needs:

- **You can't apply.** "Anthropic may then escalate listings flagged as highly useful to Claude users to Verified review. That escalation is assessed automatically" ([verification](https://claude.com/docs/connectors/verification)).
- **It isn't a usage threshold.** One analysis found a Community connector with about 62,000 accounts and a Verified one with about 15 ([trytofu.ai](https://trytofu.ai), third-party).
- **Verified listings get an indexable page on claude.com.** Community listings don't. That page can rank in Google for "<brand> Claude" ([trytofu.ai](https://trytofu.ai), third-party, 916 pages counted).
- **Verified unlocks MCP App entry points and embedded websites.**

Until #954 is fixed, assume a Community listing gets directory browsing and search, but not in-chat suggestions to people who haven't heard of you.

### ChatGPT: three signals, from one OpenAI employee

OpenAI's official statements are general:

- "Rankings now prioritize plugins people keep using after installation" (ChatGPT release notes, 2026-08-21, web and mobile; quoted by secondary sources because the page blocks automated fetches).
- "Improved ranking and recommendations help people find relevant plugins in the directory and in conversations" ([DevDay 2026 recap](https://openai.com/index/devday-2026-recap/), 2026-09-29).
- In-chat suggestions use "conversational context, app usage patterns, and user preferences" ([OpenAI](https://openai.com/index/developers-can-now-submit-apps-to-chatgpt/), 2025-12-17).
- "Plugins that demonstrate strong real-world utility and high user satisfaction may be eligible for enhanced distribution opportunities, such as directory placement or proactive suggestions" ([guidelines](https://developers.openai.com/apps-sdk/app-submission-guidelines)).
- "We do not guarantee any particular placement, visibility, ranking, or level of promotion" ([developer terms](https://openai.com/policies/developer-apps-terms/), 7.4).

The most specific statement came from Max Stoiber, who works on plugins at OpenAI, in an [X post](https://x.com/mxstbr/status/2108600018789634370) on 2026-10-09: ranking "across the directory and in-chat suggestions is fully deterministic and based on three factors: 1. conversion 2. active usage 3. retention". That is one employee's post, not documentation. It matches the release note above.

What that means in practice:

- **Conversion:** people who see your listing or suggestion card install it. Your name, short description, icon and starter prompts decide this.
- **Active usage:** installed users keep calling your tools. Recall on indirect prompts (Step 6) decides this.
- **Retention:** they come back in later weeks. A first run that finishes the job decides this.

How the directory is laid out ([help center](https://help.openai.com/en/articles/20001256-plugins-in-chatgpt); [theopenapps.com](https://theopenapps.com/blog/how-to-submit-a-chatgpt-app), third-party):

- **Shelves:** Popular, New & Noteworthy, Small Business, plus about 13 category lists. Popular and the categories look algorithmic. New & Noteworthy and Small Business look editorial.
- **No ratings or install counts** are shown.
- **Main pages show only apps "selected for enhanced distribution".** "There is no process by which to request this" ([submission](https://developers.openai.com/apps-sdk/deploy/submission)).
- **In-chat suggestions** may come from a pool of a few hundred apps, not the whole directory (pluginsradar.com, third-party estimate, not confirmed).
- **App pages on chatgpt.com render in the browser with no app-specific title or description.** Developers report Google doesn't index them, so links to your ChatGPT listing pass little SEO value. Link to your own landing page instead, and link the listing from there.
- **No publisher analytics.** ChatGPT gives you no dashboard for installs, suggestions or retention. Measure on your own server.

### Both: metadata decides whether you're even considered

Before ranking, the assistant has to find your tool. Both vendors now search large tool sets before the model sees them:

- **Claude's tool search** matches tool names, descriptions, argument names and argument descriptions, and returns 5 tools by default. Anthropic's advice when Claude misses a tool: "Add common keywords to tool descriptions" ([tool search](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool)).
- **OpenAI's tool search** shows the model only the server's name and description first, and loads its functions only if that matches. OpenAI says to state the use case clearly there and keep each group under 10 functions ([tool search](https://developers.openai.com/api/docs/guides/tools-tool-search)). This is the API; ChatGPT's internals aren't documented, but they likely work the same way.

Research on tool selection points the same way:

- **A clear, distinct purpose matters most.** In a study with gpt-4o-mini, a description that made the tool's purpose distinct raised selection from 6.2% to 17.8%. Accuracy added 8.8 points and complete parameter docs 5.9 points. Conciseness barely mattered ([arXiv 2602.18914](https://arxiv.org/html/2602.18914v1)).
- **Matching the user's words helps a little.** Similarity between the query and the description correlates with selection, weakly to moderately ([BiasBusters, ICLR 2026](https://arxiv.org/html/2510.00307v2)).
- **Descriptions move selection more than names.** Swapping descriptions shifted selection more than changing names. Parameter-only changes had the smallest effect (same paper).
- **Most descriptions are poor.** 97% of 856 real tool descriptions had at least one flaw, most often missing limits, missing usage guidance or unclear parameters ([arXiv 2602.14878](https://arxiv.org/pdf/2602.14878)). Only 9% of 20,403 ChatGPT app tools open with "Use this when" ([theopenapps.com](https://theopenapps.com/blog/how-to-write-a-chatgpt-app-listing)). Doing the basics well already puts you ahead.

### Don't game it

Papers show that promotional and "authoritative" wording raises selection a lot in lab tests. "Trusted by over 100,000 users" lifted one model's selection to 89.5% ([ToolTweak](https://arxiv.org/html/2510.02554v1)). Authoritative descriptions reached 95% on Claude ([MPMA, AAAI 2026](https://arxiv.org/html/2505.11154v2)). Both platforms ban exactly this:

- OpenAI's "fair play" rule bans metadata that steers the model to prefer your app, comparisons, "best" or "official", and promotional claims ([guidelines](https://developers.openai.com/apps-sdk/app-submission-guidelines)).
- Anthropic rejects descriptions that promote products, hide instructions or interfere with other tools ([review criteria](https://claude.com/docs/connectors/building/review-criteria)).
- Every submission is scanned, and ChatGPT rescans your server daily after launch. A trick that works in a paper gets you rejected or removed.

Tuning a description to be *accurate and distinct* is the job. Tuning it to *win* is manipulation.

### Summary of signals

| Signal | ChatGPT | Claude |
|---|---|---|
| Distinct accounts that connect (30 days) | Not stated | **Directory rank, official** |
| Growth in that count | Not stated | **Trending tag, official** |
| Conversion (view → install) | Employee statement | Not stated |
| Active usage | Employee statement | Counts toward rank |
| Retention after install | **Official** (release notes) and employee statement | Not stated |
| Verified / editorial selection | Editorial shelves, "enhanced distribution", no application | Verified label, automatic, no application. Needed for in-chat suggestions in practice (#954) |
| Name, one-liner, tool names | Exact-name search finds you | Directory search reads them (third-party test) |
| Tool metadata | **Official:** decides tool calls | Tool search reads it (official) |
| Error rate, latency | Not stated | Shown to publishers; not stated as a signal |
| Ratings, installs shown | No | No |
| Paid placement | None found; ads banned | **None, official** |
| Publisher analytics | None | **Dashboard, official (beta)** |

## Step 1: Decide what the app does in a chat

Don't port your whole UI. Pick the 3 to 8 jobs a user would ask an assistant to do, where the chat already holds the input.

1. List your product's core jobs.
2. For each job, ask: can the user state it in one sentence, with everything the tool needs in that sentence or in a file?
3. Keep the jobs where the answer is yes. A job that needs dragging, drawing or careful visual placement becomes a link back to your web app.
4. Split each job into read tools and write tools.

> **Example:** an e-signature product. "Make an NDA for Acme and send it to jo@acme.com" is all text, so it works in chat. "Put a signature box at the bottom of page 3" needs a visual editor, so the tool either places fields by a fixed rule (for example a signature page at the end) or returns a link to finish in the web app.

Check each tool against the content rules before you build it (Step 5). For example, ChatGPT doesn't allow selling subscriptions or showing upgrade links inside the app, so "upgrade my plan" can't be a tool.

## Step 2: Build one server for both platforms

Both platforms connect to a **remote MCP server** over HTTPS with Streamable HTTP, usually at `/mcp`. Claude no longer accepts local (MCPB) servers in the directory ([Claude submission](https://claude.com/docs/connectors/building/submission)). ChatGPT accepts one MCP server per app ([MCP server](https://developers.openai.com/apps-sdk/build/mcp-server)).

Rules that save you a rewrite:

- **Reuse your API code.** Tool handlers should call the same functions your web API calls, with the signed-in user's account. Don't write a second set of business rules.
- **Call only your own APIs.** Claude rejects servers that proxy third-party APIs ([review criteria](https://claude.com/docs/connectors/building/review-criteria)).
- **Return structured results and short text.** The assistant reads the text. A UI, if you have one, reads the structured data.
- **Put it on your own domain**, for example `https://mcp.example.com/mcp` or `https://example.com/mcp`. Both platforms want a stable, public URL. ChatGPT also verifies the domain (Step 7).

### A tool definition that passes both reviews

```ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { z } from "zod";

const server = new McpServer({ name: "example", version: "1.0.0" });

server.registerTool(
  "get_document_status",
  {
    title: "Get document status",
    description:
      "Use this when the user asks whether a document they sent has been signed, or who still has to sign. " +
      "Returns the status and each recipient's progress. Do not use it to send or change a document.",
    inputSchema: {
      documentId: z.string().uuid().describe("The document ID returned by send_for_signature, for example 3f2a…"),
    },
    annotations: { readOnlyHint: true, destructiveHint: false, openWorldHint: false },
  },
  async ({ documentId }, extra) => {
    const status = await documentStatus(accountFrom(extra), documentId);
    return {
      content: [{ type: "text", text: summarize(status) }],
      structuredContent: status,
    };
  },
);
```

### Interactive UI (optional)

Both platforms render the **MCP Apps** standard: a `ui://` resource with the MIME type `text/html;profile=mcp-app`, linked from the tool with `_meta.ui.resourceUri` ([MCP Apps](https://modelcontextprotocol.io/extensions/apps/overview), [ChatGPT](https://developers.openai.com/apps-sdk/mcp-apps-in-chatgpt), [Claude](https://claude.com/docs/connectors/building/mcp-apps/getting-started)). Use `registerAppTool()` and `registerAppResource()` from the ext-apps SDK. They write each host's metadata for you.

Ship text-only tools first. Add a UI only for a job where text is clearly worse, such as a preview of a generated document. Notes if you do:

- Declare every outside origin in `_meta.ui.csp` (`connectDomains`, `resourceDomains`, `frameDomains`).
- Claude needs `_meta.ui.domain` set to the first 32 hex characters of `sha256(serverUrl)` plus `.claudemcpcontent.com`.
- `window.openai.*` APIs work only in ChatGPT. Feature-detect them, and prefer the standard `ui/*` messages.
- Claude asks for 3 to 5 PNG screenshots, at least 1000 px wide, for an app with a UI.
- Claude App entry points and embedded websites need a Verified listing.

## Step 3: OAuth that both platforms accept

Your tools act for a signed-in user, so you need OAuth 2.1 with the authorization code flow and PKCE, as the MCP authorization spec describes ([Claude auth](https://claude.com/docs/connectors/building/authentication), [ChatGPT auth](https://developers.openai.com/apps-sdk/build/auth)). If your stack has an OAuth provider library for MCP, use it. Don't write token handling by hand.

### What both need

- **Protected resource metadata** (RFC 9728) at `/.well-known/oauth-protected-resource`. Its `resource` must equal your MCP URL exactly. Claude uses only the first entry in `authorization_servers`.
- **A `401` with `WWW-Authenticate: Bearer resource_metadata="…"`** on an unauthenticated request. Claude ignores the header on a `200`.
- **PKCE S256**, listed in `code_challenge_methods_supported`.
- **CIMD and DCR both.** CIMD (client ID metadata documents) is preferred by both. DCR is the fallback. Claude warns that DCR registers a new client on every connection, which adds up with directory traffic.
- **A `/token` endpoint that accepts `application/x-www-form-urlencoded`.**
- **Refresh-token rotation** for public clients. Return `invalid_grant` for a dead refresh token.
- **No `client_credentials` grant.** Neither platform uses machine-to-machine auth.

### Authorization server metadata

```json
{
  "issuer": "https://example.com",
  "authorization_endpoint": "https://example.com/oauth/authorize",
  "token_endpoint": "https://example.com/oauth/token",
  "registration_endpoint": "https://example.com/oauth/register",
  "response_types_supported": ["code"],
  "grant_types_supported": ["authorization_code", "refresh_token"],
  "code_challenge_methods_supported": ["S256"],
  "token_endpoint_auth_methods_supported": ["none", "private_key_jwt"],
  "client_id_metadata_document_supported": true,
  "authorization_response_iss_parameter_supported": true,
  "scopes_supported": ["read", "write"]
}
```

- `"none"` plus `client_id_metadata_document_supported: true` turns on CIMD for Claude. Without both, Claude falls back to DCR.
- ChatGPT's CIMD client ID is `https://chatgpt.com/oauth/client.json`. It uses `none` or `private_key_jwt`.
- `authorization_response_iss_parameter_supported` is RFC 9207. ChatGPT wants it, and then you must return `iss` on every authorization response.

### Redirect URIs to allow

| Client | Redirect URI |
|---|---|
| Claude (web and apps) | `https://claude.ai/api/mcp/auth_callback` |
| Claude Code | `http://localhost/callback` and `http://127.0.0.1/callback`, any port |
| ChatGPT, with RFC 9207 | `https://chatgpt.com/connector_platform_oauth_redirect` |
| ChatGPT, without RFC 9207 | `https://chatgpt.com/connector/oauth/{callback_id}`. Copy the exact value from your app's settings page. |

### Other details that break connections

- **Timeouts.** Claude gives discovery, registration and token endpoints 10 seconds, and refresh 30 seconds.
- **Firewalls.** Claude connects from `160.79.104.0/21`. Allow it in any WAF or bot rule in front of your server and your sign-in pages.
- **Token checks.** ChatGPT sends `resource=<your MCP URL>`. Check `aud`, `iss`, `exp` and scopes on every request.
- **Public tools.** Claude supports lazy auth: tools that need no account work without sign-in, and a `401` from a protected tool starts sign-in ([lazy auth](https://claude.com/docs/connectors/building/lazy-authentication)). In ChatGPT, declare `securitySchemes` per tool and return `_meta["mcp/www_authenticate"]` to show the sign-in prompt.
- **The consent screen is your normal sign-in page.** The user signs in to your product, then approves the client. Reuse your existing sign-in, but see Step 7 about reviewer accounts.

## Step 4: Write metadata that gets you called

This is the SEO of app directories. The assistant reads your tool names, descriptions and parameter docs, then decides whether to call you.

### Tool names

- A plain verb and object: `send_for_signature`, `get_document_status`, `list_templates`. OpenAI suggests pairing the domain and action.
- At most 64 characters (Claude).
- No "best", "official" or brand claims (ChatGPT).
- Unique within your server.
- Separate read tools from write tools. Both platforms reject a catch-all tool such as `api_request` with a `method` parameter.

### Tool descriptions

Follow OpenAI's pattern. Start with when to use the tool, then say when not to:

```
Use this when the user wants a document signed by one or more people and has
given their names and email addresses. Sends a signing invitation by email to
each person. Do not use it to draft or edit a document's text.
```

- **Describe the user's job, not your API.** "Use this when the user wants a document signed" matches more requests than "Creates an envelope resource".
- **Use the words users say.** If users say "get this signed", put that phrase in.
- **Keep it narrow and true.** Claude rejects descriptions that tell the model how to behave, mention other tools, promote products, or hide instructions. ChatGPT rejects descriptions that try to manipulate tool selection.
- **No comparisons.** Don't name rivals or say "better than".

### Parameter docs

- Describe every parameter, with an example.
- Use enums for fixed values, and `describe()` each one.
- Say what format you expect (ISO dates, email addresses, IDs from another tool).

### Annotations

Set all three as explicit booleans on every tool, plus a `title`. ChatGPT requires all three. Claude requires `title` plus `readOnlyHint` or `destructiveHint`. Set all three and both are satisfied.

| Tool kind | `readOnlyHint` | `destructiveHint` | `openWorldHint` |
|---|---|---|---|
| List, get, search | `true` | `false` | `false` |
| Create a draft the user can delete | `false` | `false` | `false` |
| Send email, publish, invite, delete, cancel | `false` | `true` | `false` unless it posts publicly |

- Annotations drive permissions in Claude: read-only tools can run without a prompt, and destructive tools always ask the user first.
- OpenAI: "being able to undo an action does not, by itself, justify setting destructiveHint to false." Anything that emails a real person counts as destructive.
- `openWorldHint: false` fits tools that stay inside the user's private account. Set it to `true` for tools that post publicly or reach outside systems.

### Errors

Return errors the assistant can act on: what went wrong and what to do. "The PDF has 140 pages; the limit is 100. Ask the user for a shorter file." passes review. "Internal Server Error" fails it.

## Step 5: Stay inside the content rules

Read both policies before you design the tools. The ones most likely to catch a SaaS:

| Rule | ChatGPT | Claude |
|---|---|---|
| Selling digital goods, subscriptions or credits in the app | Not allowed. No plan lists, upgrade prompts, or links to checkout or upgrade flows. Commerce is limited to physical goods with checkout on your own domain. | Not listed as banned. |
| Pricing, trials, promotions in listing copy | Not allowed. No ChatGPT-only pricing either. | Not listed. Keep it out anyway, for one listing text. |
| Ads | Not allowed. | Not listed. |
| Money transfers, crypto | Not allowed. | Not allowed. |
| AI image, video or audio generation | Not listed. | Not allowed. |
| Reading the chat history or memory | Not allowed. | Not allowed. |
| Sensitive data | No card data, health data, government IDs, passwords, API keys or one-time codes collected through the app. No precise location. | Calls only your own first-party APIs. |
| Demo or trial-only apps | Rejected. | Not listed. |

Two practical results for most SaaS products:

- **Plan limits need a neutral message.** When a free user hits a limit, return "You've used all 7 documents for this month on your plan." Don't return a link to upgrade, and don't list plans.
- **Don't ask for credentials in a tool.** Sign-in happens only on your OAuth page.

Sources: [ChatGPT guidelines](https://developers.openai.com/apps-sdk/app-submission-guidelines), [Claude review criteria](https://claude.com/docs/connectors/building/review-criteria).

## Step 6: Test with a golden prompt set

You can't see the ranking, but you can measure whether the assistant calls you. Write the set before you write the listing.

### Method

1. Write about 30 prompts in three groups:
   - **Direct:** the user names your product. "Use Example to send this NDA to jo@acme.com."
   - **Indirect:** the user states only the outcome. "Get this contract signed by both partners." This group is where new users come from, so make it the biggest.
   - **Negative:** prompts that look close but belong to another tool or to the assistant itself. "Summarize this contract." "Draw my signature as an image."
2. For each prompt, write the expected result: which tool, with which arguments, or no tool.
3. Connect your server: in ChatGPT developer mode, and in Claude as a custom connector.
4. Run every prompt in a fresh chat on each platform. Record: called the right tool, called the wrong tool, called nothing.
5. Count **precision** (of the times your tool was called, how often it was the right call) and **recall** (of the prompts that should call it, how often it did).
6. Change **one** metadata field, rerun the whole set, and compare. Keep the change only if recall goes up without precision going down.

### Template

```markdown
# Golden prompts

Last run: <date>, ChatGPT <recall/precision>, Claude <recall/precision>

| # | Group | Prompt | Expected | ChatGPT | Claude |
|---|---|---|---|---|---|
| 1 | Direct | Use <Brand> to send nda.pdf to jo@acme.com | send_for_signature | ✓ | ✓ |
| 2 | Indirect | Get this lease signed by my tenant, ravi@example.com | send_for_signature | ✓ | none |
| 3 | Indirect | Has Acme signed the proposal yet? | get_document_status | ✓ | ✓ |
| 4 | Negative | Summarize this contract in 5 bullets | none | ✓ | ✓ |
| 5 | Negative | Make a picture of my signature | none | wrong tool | ✓ |
```

Pull the indirect prompts from your buyer prompt list in [chapter 5](05-ai-search.md). The same phrasing that gets you cited also gets you called.

## Step 7: Prepare the review material

Gather all of this before you open either submission form.

### For both

- **Public URLs:** website, documentation, privacy policy, terms of service, support contact. All HTTPS. A help article or blog post counts as documentation, but it must be live by your publish date.
- **The privacy policy must name the app.** Say what data the app reads from the chat, what it stores, who receives it, how long you keep it, and how users delete it. ChatGPT checks for data categories, purposes, recipients, retention and user controls.
- **A square icon.** ChatGPT accepts at least 48×48 px, up to 5 MiB. Use a large source file.
- **A reviewer test account,** fully populated with realistic sample data. It must work straight away. ChatGPT rejects accounts that need MFA, email or SMS codes, magic links, a new sign-up or a private network. If your product signs in only with email codes, add a password sign-in for this one account before you submit.
- **Proof you ran every tool.** Claude asks you to confirm you tested each tool in MCP Inspector or as a custom connector.

### Listing copy

| Field | ChatGPT | Claude |
|---|---|---|
| Name | At most 30 characters. Not generic, no "MCP" or "Plugin". | At most 100 characters. |
| Short text | `shortDescription`, at most 30 characters | One-liner, at most 200 characters |
| Long text | At most 4,000 characters | At most 2,000 characters. Anthropic can't edit it for you. |
| Starter prompts | 3, at most 128 characters each | None |
| Category | One, from the dashboard list | 1 to 5 |
| URL slug | None | Permanent once published |
| Screenshots | Not confirmed in the docs | 3 to 5 PNGs, only for apps with a UI |

Write the copy from your facts file ([chapter 5, step 7](05-ai-search.md#step-7-keep-facts-consistent-everywhere)):

- **Lead with the job, in the user's words.** "Send PDFs for signature and track who has signed" beats "AI-powered agreement platform".
- **Put your category words in the name, the one-liner and the tool names.** Claude's directory search seems to read those and skip the long description, and ChatGPT finds most apps by name search (see "How ranking works"). A name like "Example: e-signature for small firms" is found by "signature"; "Example" alone is not. Stay inside the length limits and the rule against generic names.
- **Name the inputs and outputs.** What the user gives, what they get back.
- **Name the audience** if it's narrow: "for small firms", "for landlords".
- **Starter prompts are indirect prompts** from your golden set: the three with the best recall.
- **Leave out** prices, trials, comparisons and claims you can't prove.
- **Copy your category from the strongest listed rival.** In e-signature, the verified listing in Claude used Legal and Productivity.

### ChatGPT only

- **A verified organization** (individual or business) in the OpenAI Platform dashboard.
- **5 positive and 3 negative test cases.** Take them from your golden set.
- **A video walkthrough URL** showing sign-in and each main tool.
- **Domain verification.** Serve the token as plain text at `https://<host>/.well-known/openai-apps-challenge`.
- **No secrets in the uploaded package.**

## Step 8: Submit

### Claude

1. Go to [claude.ai/directory/manage](https://claude.ai/directory/manage) and choose Submit new, then MCP connector. You need a paid plan. On Team and Enterprise plans, an Owner or a member with the Directory permission submits.
2. Work through the steps: connection, tools, listing, use cases, company, authentication, data handling, test and launch, compliance, review.
3. After the automated scan, the listing goes live as Community. There is no fixed review time. Write to `mcp-review@anthropic.com` if it's stuck.

([Publish](https://claude.com/docs/directory/publish), [submission](https://claude.com/docs/connectors/building/submission))

### ChatGPT

1. In the OpenAI Platform dashboard, upload the app package and choose the organization, project and verified identity.
2. Fix the automated findings, connect the server, complete domain verification, and wait for the tool scan.
3. Add the reviewer credentials, test cases, video and release notes.
4. Complete the policy attestations and submit. Only one review can be active at a time.
5. After approval, click Publish. You can limit countries with `publication.countries`.
6. Rejections arrive by email. Reply to appeal.

After publishing, OpenAI scans your server daily. Tool changes that pass the automated checks go live without a new review, so you can tune descriptions after launch. ([Submission](https://developers.openai.com/apps-sdk/deploy/submission))

## Step 9: List in the MCP Registry

The official [MCP Registry](https://modelcontextprotocol.io/registry/quickstart) is a public index that other directories read from. PulseMCP, for example, stopped taking submissions and picks servers up from it. Publish under your own domain so nobody else can claim the name.

```json
{
  "$schema": "<the schema URL that mcp-publisher init writes>",
  "name": "com.example/example",
  "description": "Send PDFs for signature and track who has signed.",
  "version": "1.0.0",
  "remotes": [{ "type": "streamable-http", "url": "https://mcp.example.com/mcp" }]
}
```

1. Install `mcp-publisher` and run `mcp-publisher init` to get a `server.json`. Keep the `$schema` URL it writes.
2. Prove you own the domain. Make a key with `openssl genpkey -algorithm Ed25519 -out key.pem`, then either:
   - **DNS:** add a TXT record `v=MCPv1; k=ed25519; p=<public key>`, then run `mcp-publisher login dns --domain example.com --private-key <hex>`.
   - **HTTP:** serve the same string at `https://example.com/.well-known/mcp-registry-auth`, then run `mcp-publisher login http --domain example.com --private-key <hex>`.
3. Run `mcp-publisher publish`.
4. Check it: `curl "https://registry.modelcontextprotocol.io/v0.1/servers?search=example"`.

### List on Smithery, Glama and mcp.so

These three sites are where developers and people who use AI agents look for MCP servers. They don't change your ranking inside ChatGPT or Claude. They give you backlinks and a second route for people to find you. Treat them like the directories in [chapter 6](06-off-page.md), and add each one to your launch tracker.

Do them after the registry entry, because Glama and others read from it.

| Site | How to list a remote server | Notes |
|---|---|---|
| [Smithery](https://smithery.ai) | Go to [smithery.ai/new](https://smithery.ai/new) and enter your server's HTTPS URL. With the CLI: `smithery mcp publish "https://mcp.example.com/mcp" -n @your-org/your-server` ([docs](https://smithery.ai/docs/build/publish)). | Needs Streamable HTTP, plus OAuth if your tools need sign-in. Smithery registers itself as a client with CIMD, so Step 3's setup already works. If its scan gets a `403` (a bot rule or WAF challenge), serve a static server card at `/.well-known/mcp/server-card.json` so it can skip the scan. Smithery is now part of Arcade. |
| [Glama](https://glama.ai/mcp/connectors) | Search its connectors list for your name first. Glama builds that list partly from the official MCP Registry, so your entry may already be there ([FAQ](https://glama.ai/mcp/faq)). If not, use "Add Connector" on the connectors page. | Remote servers are listed as "connectors", separate from GitHub-based "servers". Glama also has an MCP Inspector, a quick way to check your server from outside before reviewers see it. Glama sells hosting and ads. You don't need either to be listed. |
| [mcp.so](https://mcp.so/submit) | Submit the form, with the type set to "Remote Server". | The form requires a repository URL. Give the public repo or docs page for the server if you have one. If you don't, write to `support@mcp.so` with your server URL and docs page. |

For each listing:

1. **Use the same name and one-liner** as your ChatGPT and Claude listings and your facts file. Assistants repeat whatever they find, so one wording everywhere avoids mixed descriptions.
2. **Link your help page,** not the bare `/mcp` URL. The help page tells a person how to connect. The endpoint only returns `401` in a browser.
3. **Put your category words in the title,** as in Step 7. These sites search titles first.
4. **Check the listing after a day.** Make sure it shows your tools and the right auth type. A listing that shows "0 tools" or "no auth" means the site's scan could not connect. Check your WAF and bot rules first.

Facts in this table come from each site's own pages, checked on 2026-11-11. These sites change their submission flows often, so follow their current page if it differs.

## Step 10: Promote and measure after launch

### Send people to the listing

You bring the first users. Neither directory places a new listing in front of strangers, and Claude's in-chat suggestions skip Community listings in practice.

- **Add "Add to ChatGPT" and "Add to Claude" buttons** to your site, docs, app settings and onboarding emails. On Claude, every account that connects counts toward rank, even before it signs in.
- **Ask your existing users first.** They already have accounts, so they finish sign-in and call tools. That builds the usage and retention both platforms rank on.
- **Link your own landing page, not the chatgpt.com listing,** from other sites. The ChatGPT listing page isn't indexed by Google; your page is.
- Write one help article per platform: how to connect, three example prompts, how to disconnect. It doubles as the documentation URL the reviewers want, and it's a page that can rank for "<brand> ChatGPT" searches.
- Add the integration to `llms.txt` and your facts file, so assistants mention it when someone asks about you.
- Announce it on the channels from [chapter 6](06-off-page.md).

### Measure

| What | Where |
|---|---|
| Connections, sign-ins and tool calls per tool | Your server logs or analytics. Log the tool name and the client (ChatGPT, Claude, Claude Code) from the OAuth client ID. Never log arguments that hold user content. |
| Tool errors per tool | The same events. A tool that fails often loses usage, and usage is the ranking signal. |
| Claude rank, accounts that used a tool, calls, error rate, latency by product and tool | Claude's publisher dashboard at claude.ai/directory/manage (beta, updated daily). Keep the error rate at 2% or less for the "Healthy" badge. |
| Installs, suggestions, retention on ChatGPT | ChatGPT has no publisher dashboard. Count new OAuth grants from ChatGPT's client ID, and accounts that call a tool again in later weeks. |
| Sign-ups that came through an app | Mark accounts created from the OAuth flow, and add "ChatGPT or Claude app" to your "How did you hear about us?" question. |
| Wrong-tool calls | Your golden set rerun. Neither platform reports them. |

### Weekly routine

1. Read tool calls and errors per tool. Fix the top error first.
2. Rerun the golden set on both platforms.
3. Change one description if a prompt group fell. Rerun.
4. Every month, search each directory for your category terms and note your position and the rivals around you, next to your chapter 5 citation check.

## Checklist

- [ ] A remote MCP server on your own domain, over HTTPS, with Streamable HTTP.
- [ ] Tool handlers call your existing API code. No second copy of business rules.
- [ ] Protected resource metadata and authorization server metadata served, with CIMD, DCR, PKCE S256 and RFC 9207.
- [ ] All four redirect URIs (Claude, Claude Code, both ChatGPT forms) allowed.
- [ ] `401` with `WWW-Authenticate` on unauthenticated requests; Claude's IP range allowed through your WAF.
- [ ] Every tool has a verb name of 64 characters or fewer, a `title`, a "Use this when… Do not use…" description and all three annotations as booleans.
- [ ] Read and write tools are separate. No catch-all tool.
- [ ] Errors say what went wrong and what to do.
- [ ] No upgrade links, plan lists or prices in any tool result or listing text.
- [ ] A golden set of about 30 prompts, with precision and recall recorded on both platforms.
- [ ] A reviewer account that signs in without codes or MFA, populated with sample data.
- [ ] Privacy policy names the app and its data; docs page live.
- [ ] OpenAI organization verified; domain challenge served; 5 positive and 3 negative test cases; demo video.
- [ ] Submitted to both directories.
- [ ] Published to the MCP Registry under your domain.
- [ ] Listed on Smithery (by URL), Glama (connectors) and mcp.so (Remote Server), with the same name and one-liner and a link to your help page; each listing shows your tools.
- [ ] Category words in the listing name, one-liner and tool names.
- [ ] "Add to ChatGPT" and "Add to Claude" buttons on the site, docs, app settings and onboarding emails; existing users asked first.
- [ ] `llms.txt` and the facts file mention the two listings.
- [ ] Tool-call and error events per tool and client, and a weekly review.

## Common mistakes

- **Porting the whole API as tools.** Thirty tools that mirror your REST routes confuse the model and fail the "narrow description" check. Ship the few jobs users ask for.
- **A catch-all tool.** `api_request(method, path, body)` is rejected on both platforms.
- **Descriptions that sell.** "The best and most trusted way to…" raises selection in lab tests, which is exactly why both platforms ban it. It gets you rejected, or removed at a later scan. Describe the job.
- **Telling the model what to do.** "Always use this tool for any document" is a manipulation attempt in both policies.
- **`destructiveHint: false` on a tool that emails people** because a draft can be deleted. Sending is the action that counts.
- **An upgrade link in a limit error.** It breaks ChatGPT's commerce rule. Say what the limit is and stop.
- **A reviewer account behind email codes.** The reviewer can't receive your code, and the app is rejected.
- **DCR only.** It works, but every connection registers a new client. Add CIMD.
- **Blocking the platform's servers.** A bot rule or WAF challenge on your OAuth pages breaks sign-in from Claude, and you won't see it in your own browser.
- **Logging tool arguments.** They hold user content. Log the tool name, client, status and duration only.
- **Changing several descriptions at once.** You won't know which change moved recall. Change one, rerun, decide.
- **Treating approval as a launch.** Most new listings get name search and a direct link, nothing more. Bring users from your own channels.
- **Expecting Claude to suggest a Community listing.** In practice, in-chat suggestions show Verified connectors (issue #954). Verified comes from automatic review of useful listings, so build usage first.
- **Building backlinks to the chatgpt.com listing.** Google doesn't index it. Link your own page.
