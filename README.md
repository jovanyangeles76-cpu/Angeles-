# Angeles-
Angeles Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# Manage costs effectively

> Track token usage, set team spend limits, and reduce Claude Code costs with context management, model selection, extended thinking settings, and preprocessing hooks.

Claude Code charges by API token consumption. For subscription plan pricing (Pro, Max, Ultra, Team, Enterprise), see [claude.com/pricing](https://claude.com/pricing). Per-developer costs vary widely based on model selection, codebase size, and usage patterns such as running multiple instances or automation.

Across enterprise deployments, the average cost is around \$13 per developer per active day and \$150-250 per developer per month, with costs remaining below \$30 per active day for 90% of users. To estimate spend for your own team, start with a small pilot group and use the tracking tools below to establish a baseline before wider rollout.

This page covers Claude Code usage only: how to [track your costs](#track-your-costs), [manage costs for your organization](#manage-costs-for-your-organization), and [reduce token usage](#reduce-token-usage). For usage limits in other Claude products, see the [Claude Help Center](https://support.claude.com).

## Track your costs

### Using the `/usage` command

> [!NOTE]
>
> The Session block in `/usage` shows API token usage and is intended for API users. Claude Pro, Max, and Ultra subscribers have usage included in their subscription, so the session cost figure isn't relevant for billing purposes. Subscribers see plan usage bars, activity stats, and a usage breakdown on the same screen.

The Session block at the top of `/usage` shows detailed token usage statistics for your current session.

Claude Code computes the dollar figure locally from token counts at list price, unless a [`modelPricing`](https://code.claude.com/docs/en/settings-reference#modelpricing) table is in effect. An administrator sets one in your organization's managed settings so the figure uses your contracted rates, and while a table is in effect the `Total cost` line carries the note `at your organization's configured rates`. The figure is an estimate, so for authoritative billing see [Manage costs for your organization](#manage-costs-for-your-organization).

```text theme={null}
Total cost:            $0.55
Total duration (API):  6m 20s
Total duration (wall): 6h 33m 10s
Total code changes:    0 lines added, 0 lines removed
Usage by model:
   claude-sonnet-4-6:  1.2k input, 5.3k output, 940.0k cache read, 50.0k cache write ($0.55)
```

These totals reset when `/clear` starts a new session, so the next session's total cost starts at \$0. Before v2.1.211, they kept accumulating across `/clear` for the lifetime of the Claude Code process.

For a response from the Claude API billed at the 1.1× [data residency rate](https://platform.claude.com/docs/en/about-claude/pricing#data-residency-pricing), Claude Code multiplies the list price of that response's tokens by 1.1 in the session cost figure. The same total appears in the [status line's cost field](https://code.claude.com/docs/en/statusline.md#cost-and-duration-tracking), and the multiplied figure also counts toward [`--max-budget-usd`](https://code.claude.com/docs/en/cli-reference#cli-flags).

#### Prompt cache statistics

After the main conversation's first API response, Claude Code also adds a `Prompt cache (main)` line to the Session block, summarizing the session's [prompt cache](https://code.claude.com/docs/en/prompt-caching.md) use: the request count, the share of input tokens served from cache, cache misses, and whether the cache is warm right now. Requires Claude Code v2.1.251 or later.

```text theme={null}
Prompt cache (main):   14 requests · 91% of input tokens from cache · 2 misses (last 6m 10s ago, 310.2k tokens re-cached) · 1 expected rebuild (compaction or tool-result clearing) · warm (1h TTL, last activity 40s ago)
```

The misses, expected rebuilds, and warm or cold parts of the line mean the following:

* **Misses**: requests that re-processed content the cache already held, with the time of the last miss and how many tokens those requests wrote back to the cache. [Actions that invalidate the cache](https://code.claude.com/docs/en/prompt-caching.md#actions-that-invalidate-the-cache) lists the usual causes. When Claude Code can identify a likely cause for the last miss, the line names it too, for example `likely cause: tool definitions changed`. The likely-cause text requires Claude Code v2.1.260 or later.
* **Expected rebuilds**: when Claude Code has itself just rewritten the conversation, by [compaction](https://code.claude.com/docs/en/prompt-caching.md#compacting-the-conversation) or by clearing old tool results from context, it counts the same kind of miss as an expected rebuild instead. This part appears only after at least one expected rebuild has happened.
* **Warm or cold**: whether the cached prefix is still within its [cache lifetime](https://code.claude.com/docs/en/prompt-caching.md#cache-lifetime), with the TTL in effect. When the cache is cold, the line shows how long the session has been idle. When no response has reported cache tokens, the line ends with `no prompt caching reported by the API` instead.

The counts come from the cache token fields in the API's responses, so the line works on every provider and gateway. It covers the main conversation only, not subagents. `/clear` resets it with the rest of the Session block.

Status line scripts can read the same numbers from the [`prompt_cache` object](https://code.claude.com/docs/en/statusline#prompt-cache-fields).

#### Plan usage breakdown

On a Pro, Max, Ultra, Team, or Enterprise plan, `/usage` also shows a breakdown of what counts against your plan limits, with recommendations:

* **Attribution**: recent usage attributed to skills, subagents, plugins, and individual MCP servers, each shown as a percentage of the total. An MCP server's share counts only the requests that consumed one of its tool results. Before v2.1.222, after one call to an MCP server, Claude Code attributed every subsequent request to that server, overstating its share.
* **Recommendations**: suggested changes, one per row. Each row starts with a share of your recent usage, at least 10%, as in `25%  Trim ~/code/api/CLAUDE.md · 49k tokens per request`.
  * **How to read the share**: it tells you how much of your usage the row applies to, not how much the change saves. One request can count toward several rows, so the shares can add up to more than 100%.
  * **How to apply recommendations**: to have Claude Code apply the ones it can, see [Apply recommendations with the fix shortcut](#apply-recommendations-with-the-fix-shortcut)
* **Loops**: a row for each of the heaviest [`/loop` or other session-scoped scheduled tasks](https://code.claude.com/docs/en/scheduled-tasks) that ran recently, ordered by total tokens, with a count of the rest. Requires Claude Code v2.1.242 or later.
  * **What each row reports**: how often the task fires, how many times it ran, its total and per-run tokens, and when it last ran
  * **Stopped and re-created loops**: Claude Code keys a row by the task's prompt, so a loop you stop and re-create stays one row

Press `d` or `w` to switch between the last 24 hours and the last 7 days. The figures are approximate and computed from local session history on this machine, so usage from other devices or claude.ai is not included.

In the [VS Code extension](https://code.claude.com/docs/en/vs-code.md#check-account-and-usage), the attribution shares and recommendations appear in the Account & usage dialog with a Day and Week toggle, without the Loops rows.

#### Apply recommendations with the fix shortcut

When the `/usage` footer shows a hint such as `f to fix 2 of 3`, press `f` to have Claude Code act on the recommendations it can. The `f` shortcut requires Claude Code v2.1.305 or later.

1. After any turn in progress ends, Claude Code runs each [`/effort`](https://code.claude.com/docs/en/model-config#adjust-effort-level) command shown in a row without `run`.
2. Claude then gets one prompt with a step for each of the other rows it can act on. The prompt asks Claude to show you each change before making it.

To see what pressing `f` does with each of your rows, find the row's shape in the first column:

| Row shape | Example row | When you press `f` |
| :- | :- | :- |
| Ends in an `/effort` command with no `run` before it | `40%  Switch from max to high effort · /effort high` | Claude Code runs the command |
| Ends in `run` and a command | `30%  Compact before you step away · run /compact` | Nothing happens for this row. Run the command yourself at the Claude Code prompt |
| Ends in anything else | `25%  Trim ~/code/api/CLAUDE.md · 49k tokens per request` | Claude gets a step in the prompt for some of these rows, and nothing happens for the rest. If Claude shows you no change for a row, follow that recommendation yourself |

In `f to fix 2 of 3`, the first number counts the rows Claude Code acts on when you press `f`, and the second counts the rows shown. A hint with no `of` in it, such as `f to fix both`, means Claude Code acts on every row shown.

While [Remote Control](https://code.claude.com/docs/en/remote-control.md) is connected, the footer shows no `f` hint. If pressing `f` closes `/usage` with `Nothing was sent. Turn off Remote Control to use fixes.`, Remote Control connected while `/usage` was open. To use `f` again, [disconnect Remote Control](https://code.claude.com/docs/en/remote-control.md#check-connection-status), then run `/usage` at the Claude Code prompt and press `f` again.

#### Check your usage-credits spend

`/usage` also shows a usage-credits row while [usage credits](#add-usage-credits-to-your-subscription) are on. What the row shows depends on your plan:

* **Pro, Max, and Ultra**: your spend for the current month, measured against your monthly spend limit when you have set one. When you haven't set a limit, the row shows `Unlimited` and no spend figure.
* **Team and Enterprise**: your own spend for the current month, measured against any [limit your organization set](#claude-for-teams-and-enterprise) that applies to you. A limit that covers the whole organization doesn't appear in the row. When you have no limit of your own, the row shows your spend with no limit beside it. While usage credits are off for you, `/usage` shows no usage-credits row.

When you have a spend limit, the row appears as soon as usage credits are on and shows 0% until you first spend usage credits. Before v2.1.236, `/usage` showed the row only on Pro, Max, and Ultra plans, and a row with a spend limit stayed hidden until you had spent something.

#### When the usage request fails

When the request for your plan limits fails, most often because the usage endpoint is rate limited, `/usage` shows the last usage bars it loaded on this machine within the past 60 minutes, along with a `Showing last-known usage` note stating how long ago that data was fetched. Press `r` to retry; a successful retry replaces the last-known bars with fresh data. Without a snapshot from the past 60 minutes, `/usage` reports that the usage endpoint is rate limited and offers the same retry shortcut. Before v2.1.208, a rate-limited request in a session that hadn't loaded usage yet always showed the error with no bars.

A message such as `rate limited until 3:14am` in `/usage` means Claude Code can't fetch your plan limits again until then, not that your work with Claude is limited. If you press `r` earlier, expect the same message. Press `r` after that time for fresh data.

### Read the token count beside the spinner

While Claude works in the main conversation, the line beside the spinner can end with an elapsed time and a token count, as in `Deciphering… (10m 27s · ↓ 5.5k tokens)`. It is a live, approximate count of the output the current turn has produced so far. The arrow beside the count points down while output arrives or tools run.

* **What it covers**: output the turn generates, such as the text and tool calls Claude streams back, its thinking, and the output of subagents that run in the [foreground](https://code.claude.com/docs/en/sub-agents.md#run-subagents-in-foreground-or-background)
* **What it leaves out**: what Claude Code sends to the model, so your prompt, the conversation history, and the rest of your context don't move it
* **When it resets**: at the start of each turn, and when Claude Code [compacts the conversation](https://code.claude.com/docs/en/prompt-caching.md#compacting-the-conversation)
* **When it stays hidden**: until output starts arriving, when the row is too narrow to fit it, and in [screen reader mode](https://code.claude.com/docs/en/accessibility.md)

It won't match the `Usage by model` output figures in the `/usage` Session block, which add up the whole session rather than one turn. It also differs from the token count [verbose mode adds](https://code.claude.com/docs/en/statusline.md#notifications-share-the-status-line-row), because that count measures the size of your context rather than one turn's output.

### Analyze your usage patterns

Run [`/insights`](https://code.claude.com/docs/en/commands#all-commands) for a report on how you work rather than how many tokens you've used. It analyzes your recent sessions on this machine and writes an HTML report covering what you work on, friction points such as misunderstood requests or buggy code, and suggestions for using Claude Code more effectively. A single run analyzes up to 200 sessions it hasn't seen before and skips very short ones. When sessions are left out, the report header shows the analyzed count with the total in parentheses, for example `200 sessions (412 total)`.

When [auto mode](https://code.claude.com/docs/en/permission-modes.md#eliminate-prompts-with-auto-mode) is available to the session and your recent sessions mostly ran without it, the report can also include an estimate of how many permission prompts auto mode could have handled across those sessions.

Claude Code writes the latest report to `~/.claude/usage-data/report.html` and saves a timestamped copy of each run in the same directory, so earlier reports aren't overwritten. Claude Code deletes reports on the same schedule as the rest of your session data: at startup, it removes files older than [`cleanupPeriodDays`](https://code.claude.com/docs/en/claude-directory#cleaned-up-automatically), 30 days by default.

You can run `/insights` on any plan and with any provider. The analysis runs through the same provider and account as your regular sessions, and the tokens count against your plan or API usage. Sessions from other devices and claude.ai aren't included.

### Add usage credits to your subscription

[Usage credits](https://support.claude.com/en/articles/12429409-extra-usage-for-paid-claude-plans) let you keep working past your plan's usage limit. To manage them, run `/usage-credits` after signing in with your claude.ai subscription through `/login`; the command isn't available with API key authentication. In self-serve Enterprise organizations, Enterprise trials, and Enterprise organizations billed through AWS Marketplace, the command requires Claude Code v2.1.248 or later; earlier versions reject it with [`Unknown command: /usage-credits`](https://code.claude.com/docs/en/errors#unknown-command). What it opens depends on your role:

| Your role | What `/usage-credits` does |
| :- | :- |
| Pro, Max, or Ultra subscriber | Opens [**Settings > Usage**](https://claude.ai/settings/usage) on claude.ai in the browser. In its **Usage credits** section you can turn usage credits on or off and check your credit balance, this month's spend, and your monthly spend limit |
| Team or Enterprise member with billing access | Opens your organization's usage settings, [**Organization settings > Usage**](https://claude.ai/admin-settings/usage), in the browser |
| Team or Enterprise member without billing access | Asks you to confirm, then sends a request to your organization's admins. Before v2.1.211, Claude Code sent the request without a confirmation step |

For Team and Enterprise members without billing access, the confirmation appears only in interactive sessions: in non-interactive mode with the `-p` flag and from [Remote Control](https://code.claude.com/docs/en/remote-control.md), the command sends no request and tells you to run it in an interactive session instead.

If you run `/usage-credits` again while your earlier request is waiting on an admin, Claude Code tells you a request has already been sent rather than sending a duplicate. After an admin dismisses your request, running the command again sends a new one. Before v2.1.222, a dismissed request also blocked new requests.

On Pro, Max, and Ultra plans, when you reach your spend limit with usage credits left, Claude Code prompts you to raise or remove the limit without leaving the CLI. If the server rejects the change, see [Could not update your spend limit](https://code.claude.com/docs/en/errors.md#could-not-update-your-spend-limit).

## Manage costs for your organization

Which controls you have depends on how your organization accesses Claude Code: a Claude for Teams or Enterprise plan, the Claude Console, or a cloud provider. On Teams and Enterprise plans, usage draws from each member's seat allowance. On the Console and on cloud providers, usage is billed per token to your organization. If your organization mixes sign-in methods, each developer is metered according to the one they authenticated with.

The table maps each setup to where you see spend, where you cap it, and how you pull per-user numbers. On an individual Pro, Max, or Ultra plan you have no organization to manage, so track your own usage-credit spend, including [fast mode](https://code.claude.com/docs/en/fast-mode.md#see-where-fast-mode-spend-appears), under [Add usage credits to your subscription](#add-usage-credits-to-your-subscription).

| Your setup | See spend | Cap spend | Per-user reporting |
| :- | :- | :- | :- |
| [Claude for Teams or Enterprise](#claude-for-teams-and-enterprise) | [Spend report in org analytics](https://support.claude.com/en/articles/12883420-view-usage-analytics-for-team-and-enterprise-plans) | Spend limits in admin settings | [Spend report CSV](https://support.claude.com/en/articles/12883420-view-usage-analytics-for-team-and-enterprise-plans); [Enterprise Analytics API](https://platform.claude.com/docs/en/api/admin/analytics) on Enterprise |
| [Claude Console (API)](#claude-console) | [Console Cost page](https://platform.claude.com/cost) | Workspace spend limits | [Console dashboard](https://platform.claude.com/claude-code), [Claude Code Analytics API](https://platform.claude.com/docs/en/build-with-claude/claude-code-analytics-api) |
| [Amazon Bedrock, Google Cloud's Agent Platform, or Microsoft Foundry](#cloud-providers) | Your cloud billing console | Your cloud's budget controls | [OpenTelemetry](https://code.claude.com/docs/en/monitoring-usage.md) or an [LLM gateway](https://code.claude.com/docs/en/llm-gateway) |

[OpenTelemetry export](https://code.claude.com/docs/en/monitoring-usage.md) works on every setup and is the only option that streams per-user token and cost metrics into your own observability stack in near real time.

### Report spend at your contracted rates

By default, Claude Code computes every cost figure it shows developers at list price, so if your organization pays contracted rates, the figures in `/usage`, the status line, and OpenTelemetry don't match your bill. To make them match, set the [`modelPricing`](https://code.claude.com/docs/en/settings-reference#modelpricing) managed setting to your rates. The setting changes what Claude Code reports, not what Anthropic charges. Requires Claude Code v2.1.242 or later.

**1. Take the rates from your

