# Spec history: Live Agent Traffic Monitor

Source: `git log -p SPEC.md` in this repo (3 commits). Reasons are taken from the commit
messages unless marked otherwise.

## e080920 (10:24): Start spec from Adar's prep draft
**What:** first version: goal (readability), feed, request sections, reply, tool call beside
result, colours + labels, stats, constraints, 13 acceptance checks, 7 open decisions, and an
"observed traffic" section.
**Why:** drafted before the lab, then revised after capturing one "read a file" turn through
`proxy.py`: side requests (title, suggestion, quota), tools added mid-conversation, cache token
counts, concurrent requests, ~160 KB bodies. *(commit message)*

## 69dd6c2 (10:58): Pin what builds 1, b, c agreed on; fix two gaps they exposed
**What:** §3 now fixes the dashboard at `/__monitor/` on the same port, SSE, a `monitor.py`
split, side requests by content, Claude Code only, Python 3.9+, test ports 8050–8099. §2.2: skip
system-role messages and ignore `cache_control` when finding the "new message". Failed side
requests stay visible. Stats cover shown entries ("showing N of M"). Checks 11 and 14.
**Why:** all four builds (1, a, b, c) chose the same port/path/push/split, so the open decisions
were closed. build-c showed "(no new message)" for the first entry because it took the
system-role message as new. Build 1 and build-c both hit a contradiction: a failed quota probe
is an error *and* a hideable side request. Build 1 asked whether stats total all or visible
entries. Parallel builds collided on port 8041. *(commit message; Python version and
`cache_control` from build 1's report to us)*

## 0da6764 (11:01): Build 1 on live traffic: favicon leak, clipped results, side summaries
**What:** the page's own requests (`/favicon.ico`) answered locally; text never cut off without
"show all"; side-request summaries say what they produced; check 14 sharpened, check 15 added.
**Why:** running build 1 on live traffic and checking it in the browser: opening the page added
two `/favicon.ico` rows forwarded upstream (404); a Read result looked clipped; the title request
was summarised as `<session>`; the first entry showed "system message (61,750 chars)", the case
spec #2 already targets. *(commit message)*

## Rebuild vs edit
All builds (1, 2, a–d) were built from scratch from the spec. We did not switch to editing code
during the lab. With more time we would have switched after spec #3: the changes left were
details (favicon, a control, summary text), not structure.

## Findings in the traffic
- One "read a file" question produced 6–7 requests: a session-title request and the first step
  at the same second, the tool steps, the answer, then a "suggestion mode" request guessing what
  the user types next.
- Startup sends `HEAD /api/hello` and a `"quota"` request with `max_tokens: 1`, which got a 429,
  yet the real requests went through.
- Most instructions are not in `system` (~12 K chars): ~60 K chars arrive as a `system`-role
  message inside `messages`, with `tool_addition` blocks adding tools mid-conversation.
- Caching: each step bills ~2 uncached input tokens and reads ~56–80 K from cache.
- Each request is ~160 KB because the full history is resent every step.
- Thinking blocks come back empty; only a thinking-token count remains.
