# Spec history: Live Agent Traffic Monitor

Source: `git log -p SPEC.md` in this repo (3 commits). Reasons come from the commit messages
unless marked otherwise; build details marked *(Itamar)* come from his build sessions.

Builds: **1** (Adar, spec #1, ~28 min) · **a** 22.7 min, **b** 18.0, **c** 15.7 (Itamar, spec #1,
in parallel) · **2** (Adar, spec #2, ~22 min) · **d** (Itamar, spec #3, 15.0 min).

## e080920 (10:24): Start spec from Adar's prep draft
**What:** first version: goal (readability), feed, request sections, reply, tool call beside its
result, colours + labels, stats, constraints, 13 acceptance checks, 7 open decisions, and an
"observed traffic" section.
**Why:** drafted before the lab, then revised after capturing one "read a file" turn through
`proxy.py`: side requests (title, suggestion, quota), tools added mid-conversation, cache token
counts, concurrent requests, ~160 KB bodies. *(commit message)*

## 69dd6c2 (10:58): Pin what builds 1, b, c agreed on; fix two gaps they exposed
**What:** §3 now fixes the dashboard at `/__monitor/` on the same port, SSE, a `monitor.py`
split, side requests by content, Claude Code only, Python 3.9+, test ports 8050–8099. §2.2 skips
system-role messages and ignores `cache_control` when finding the "new message". Failed side
requests stay visible. Stats cover shown entries ("showing N of M"). Checks 11 and 14.
**Why:** all four spec #1 builds chose the same port, path, push and file split, so those open
decisions were closed. build-c showed "(no new user message)" for the first entry because it took
the system-role message as new. Builds 1, a and c hit a contradiction: a failed quota probe is an
error *and* a hideable side request (a hid it behind a red counter, c kept it visible). Build 1
asked whether stats total all or visible entries. Parallel builds collided on port 8041.
*(commit message; build-a detail from Itamar; Python version and `cache_control` from build 1's
report)*
Layouts differed and the spec still leaves this open: a used a list + detail pane; b and c an
expanding list, b with request left / reply right. *(Itamar; §6's "one list" note is inaccurate)*

## 0da6764 (11:01): Build 1 on live traffic: favicon leak, clipped results, side summaries
**What:** the page's own requests (`/favicon.ico`) answered locally; text never cut off without
"show all"; side-request summaries say what they produced; check 14 sharpened, check 15 added.
**Why:** running build 1 on live traffic and checking it in the browser: opening the page added
two `/favicon.ico` rows forwarded upstream (404); a Read result looked clipped; the title request
was summarised as `<session>` (build-b too, which also badged title and suggestion rows as MY
PROMPT); the first entry showed "system message (61,750 chars)", the case spec #2 already
targets. *(commit message; build-b detail from Itamar)*
**Result:** build 2 (spec #2) showed the prompt in its first entry and no favicon rows on live
traffic. build-d (spec #3), on live interactive traffic, passed checks 14 and 15, titled the
title request ("title: Last line of long.txt"), kept the 429 visible with side requests hidden,
and gave long results "show all". New bug in d: a re-sent suggestion request was labelled main.

## Rebuild vs edit
Every build was from scratch; we never switched to editing code. After spec #3 we would have:
the changes left were details (an icon, a control, summary text), not structure.

## Findings in the traffic
- One question produced 6–7 requests: a session title, the tool steps, the answer, then a
  "suggestion mode" request guessing what the user types next. `claude -p` sends no title or
  suggestion requests; only interactive sessions do.
- Startup sends `HEAD /api/hello` and a `"quota"` request (`max_tokens: 1`) that got 429 every
  time, with no rate-limit headers, while the real requests went through.
- Most instructions are not in `system` (~12 K chars): ~60 K chars arrive as a `system`-role
  message inside `messages`, with `tool_addition` blocks. The harness also appends a short
  (~125-char) system-role message after each step, so the last message is often not the user's.
- Suggestion mode can call tools: the model called ScheduleWakeup, the harness rejected it ("No
  tools needed for suggestion") and re-sent the request.
- Caching: each step bills ~2 uncached input tokens and reads ~56–80 K from cache. Each request is
  ~160 KB because the full history is resent every step. Thinking blocks come back empty.
