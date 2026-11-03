# SPEC: Live Agent Traffic Monitor

## 1. Goal and scope

A browser page that shows, live, every message going between a coding agent (Claude Code) and
the model, while the agent works. It is built on the course's `proxy.py`, which already sits
between the agent and the API and forwards traffic unchanged.

The point is **readability**: messages pile up fast, and at a glance I must be able to tell apart
**my prompt**, **the model's text**, **a tool call**, and **its result**, and follow the
conversation as it grows.

In scope: capturing each request/reply in the proxy, a local web page that displays them live,
drill-down into every part of a request and reply, basic statistics.

Out of scope: changing or replaying traffic, authentication, multi-user access, persistence
across restarts (unless a cheap log file falls out naturally), hosting anywhere but localhost.

## 2. What the page shows (observable behaviour)

### 2.1 The feed
- One entry per exchange (a request plus its reply), **newest first**, appearing as soon as the
  request is sent, before the reply arrives. The entry shows "waiting…", then updates as the
  reply streams in, then shows the final state.
- Each collapsed entry is one line: time, model, endpoint, a short summary of the **new**
  message (e.g. first line of my prompt, or "tool_result: Read ✓"), what the reply asked for
  (text / tool call names / stop reason), tokens in/out, duration, HTTP status.
- Errors (HTTP status ≥ 400, upstream unreachable, an `error` event in the stream) are visibly
  different (red) and are never collapsed out of view.
- **Side requests** are labelled as such and visually quieter, and one toggle hides them. The
  harness sends them besides the main conversation (see §7): the `HEAD /api/hello` probe, the
  `"quota"` probe (`max_tokens: 1`), session-title requests (no tools, structured JSON output),
  and "suggestion mode" requests sent after a turn ends. Each gets a recognisable label
  (probe / quota / title / suggestion / other). A side request that failed is an error first:
  it stays visible even when side requests are hidden. A side request's summary says what it
  produced (e.g. the title text), not its wrapper tags (`<session>`).
- The dashboard's own browser requests (e.g. `/favicon.ico`) are answered locally, never
  forwarded upstream and never shown as entries.
- Exchanges are grouped by agent session (the `X-Claude-Code-Session-Id` header, also in
  `metadata.user_id`). With one session running this is invisible; with two, I can tell them
  apart.

### 2.2 Opening an entry: the request, in clearly separated sections
1. **System instructions**: shown in full, collapsed by default, with their length. The
   `system` field is a list of blocks; show each block separately with its length and whether it
   is cached (`cache_control`).
2. **Tool definitions**: a list of tool names; each expands to its description and input schema.
   Tools can also be added mid-conversation, inside a message with role `system` and blocks of
   type `tool_addition`; show those as "tools added here" in the conversation and in this list.
   Tools marked `defer_loading` are labelled as deferred.
3. **Conversation so far**: the earlier messages, collapsed into one section.
4. **New message**: what this request adds compared with the previous request in the same
   conversation. Expanded by default. This is usually my prompt or the tool results.
   A single user message can mix harness-injected text blocks (`<system-reminder>…`) with what I
   actually typed; my typed text is shown prominently and the injected blocks are labelled and
   collapsed. Messages with role `system` inside `messages` are never the "new message": skip
   them when looking for it. The summary of a session's first request shows my prompt.
   When comparing with the previous request, ignore `cache_control` (the harness moves it to the
   newest message every turn).
5. **Request settings**: model, max_tokens, other top-level fields, headers. `authorization`
   and `x-api-key` values are **masked**. The page must never show a full token.

### 2.3 Opening an entry: the reply
- The model's text, rendered readably (Markdown allowed; never raw SSE lines by default).
- Each tool call it asks for: tool name and its input as formatted JSON.
- Thinking blocks, if present, collapsed and labelled as such. Their text may be empty
  (redacted); then show "thinking (redacted, N thinking tokens)" from
  `usage.output_tokens_details.thinking_tokens`.
- Stop reason and usage (input, output, cache read/write tokens).
- A "raw" toggle that shows the exact request JSON and the raw reply stream.

### 2.4 Tool calls beside their results
A tool call (in a reply) and its result (in a later request) are shown **together**: wherever
the call is displayed, its result appears right next to it, matched by id. If the result has not
arrived yet, show "pending". If the result is an error, mark it red. Clicking either jumps to the
exchange where the other one lives.

### 2.5 Telling message kinds apart
Each kind has its own consistent colour **and** label/icon (not colour alone):
my prompt · model text · tool call · tool result · tool error · system instructions ·
harness-injected text (e.g. `<system-reminder>` blocks inside a user message). The same style is
used everywhere these appear.

### 2.6 Formatting
- Long text (over ~20 lines) is collapsed with a "show all (N lines)" control. Text is never
  cut off without that control in view (tool results included).
- JSON is pretty-printed, indented, monospaced, and collapsible per level.
- Sections of a request are visually separate (headings, spacing, borders).
- Copy button on every block.

### 2.7 Statistics (a header bar, never instead of the messages)
Number of requests, total input/output tokens, time per request, count of calls per tool.
Input tokens are shown as three numbers: uncached `input_tokens`, `cache_read_input_tokens`,
`cache_creation_input_tokens`. With caching, `input_tokens` alone is tiny (often 2) and would
make the totals look wrong. Optional: estimated cost.
The totals cover the entries currently shown (after the side-request toggle and filters) and
say so: "showing N of M".

### 2.8 Extras (if time allows, in this order)
1. Filter by tool name and by message kind.
2. "What changed since the previous request" (diff of system/tools/messages).
3. Search across all captured text.
4. Pause the live feed (new entries queue up with a counter) so it doesn't jump while I read.

## 3. Integration and constraints

- **One command to start**: `python3 proxy.py --port 8041` still starts the proxy, and it now
  also serves the dashboard at `http://localhost:8041/__monitor/` (same port; every other path
  is forwarded as before). The agent is pointed at the proxy exactly as before
  (`ANTHROPIC_BASE_URL=http://localhost:8041 claude`).
- **Code layout**: forwarding stays in `proxy.py`; capture, parsing and the web server go in a
  new `monitor.py`; the page in `dashboard/` (HTML, CSS, JS).
- **While building and testing, never use port 8041**: the user's own proxy and other builds may
  be running on it. Pick a free port in 8050–8099 and pass it with `--port`.
- **Forwarding must stay unchanged**: same bytes, same headers, reply streamed through chunk by
  chunk with no added buffering. The dashboard must never slow down or break the agent. If the
  capture or the dashboard fails, forwarding continues.
- **No dependencies**: Python 3.9+ standard library only; the page is plain HTML/CSS/JS with no
  build step and no CDN, so it works offline.
- **Localhost only**: everything binds to `127.0.0.1`.
- **Live updates** are pushed to the page with Server-Sent Events (no manual reload, no
  polling). A page opened mid-session first loads everything captured so far, then follows the
  live stream, and resyncs after a reconnect.
- **Side requests are recognised by content**, in one small commented function: any path other
  than `/v1/messages` → probe; `max_tokens: 1` → quota; no tools and a JSON schema in
  `output_config` → title; text containing `[SUGGESTION MODE` → suggestion; else main.
- **Claude Code traffic only.** Other formats (e.g. Codex) are shown as raw JSON.
- **Streaming replies are parsed**: the reply arrives as Server-Sent Events; the dashboard
  reassembles them into text blocks, tool calls (with their JSON input), thinking, stop reason
  and usage. Unknown event or block types are shown as raw JSON, not dropped and not a crash.
  Non-streaming replies (plain JSON) and error bodies are shown too.
- **Requests run concurrently**: the harness sends two requests at the same moment (e.g. the
  session title and the main turn). Each exchange is captured on its own; the proxy's threads
  must not mix one reply into another.
- **Bodies are big**: each main request is ~160 KB of JSON, resent (and growing) on every step. The
  page must stay responsive after 100 such exchanges (render details only when opened).
- Captured data is kept in memory for the life of the proxy process.
- The terminal printing of `proxy.py` may stay or be reduced to one line per exchange.

## 4. Why (the important choices)

- Same process and port as the proxy: nothing extra to run in the lab, and the dashboard sees
  each exchange the moment it happens. API paths start with `/v1` or `/api`, so `/__monitor/`
  cannot clash.
- Errors win over hiding: a hidden failure is the thing I most need to see.
- "New message" is separated from "conversation so far" because every request resends the
  whole history; without this, every entry looks almost the same.
- Tool calls and results are paired because they arrive in different requests, which is
  exactly what makes the raw log hard to follow.
- Tokens are masked because the page shows headers and may be screen-shared in the discussion.

## 5. Acceptance checks

Run the proxy, open the dashboard, then in another terminal start Claude Code through the proxy.

1. With the page already open, I send "hi". A new entry appears at the top within 1 s of the
   request, before the reply finishes, and its reply text fills in live.
2. Ask the agent to read a small file. I can find: the full system instructions, the `Read`
   tool's definition, the tool call (with the file path), and its result shown beside it, in at
   most 2 clicks from the feed.
3. The entry whose new message is the tool result says so in its one-line summary.
4. My prompt, model text, tool call and tool result each look different and carry a label.
5. Reload the page mid-session: all earlier entries are still there.
6. No full `authorization`/`x-api-key` value appears anywhere on the page or in its data feed.
7. Stop the dashboard page (close the tab): the agent keeps working normally.
8. Point the proxy at an unreachable upstream (`--upstream http://127.0.0.1:9`): the failed
   exchange appears in red with the error text.
9. A system prompt of ~1,000 words and a 160 KB request body render without freezing the page;
   long blocks start collapsed.
10. The statistics bar's token totals equal the sum of the shown entries' usage numbers, and it
    says "showing N of M".
11. Starting an **interactive** Claude Code session (`claude -p` sends no title or suggestion
    requests) shows the quota probe and the title request labelled as side requests; hiding
    side requests leaves only the main turn's exchanges, plus any side request that failed.
12. A failing Bash command (`is_error: true` in the tool result) is red both in its result and
    in the one-line summary of the exchange that carries it.
13. Two requests sent at the same moment produce two separate, complete entries.
14. The first entry of a session shows my prompt in its summary, not "(no new message)" and not
    "system message (N chars)".
15. Opening the dashboard adds no entry to the feed (no `/favicon.ico` 404 rows).

## 6. Open decisions

| Decision | Depends on it | Answer by |
| --- | --- | --- |
| Layout: one list with expanding entries, or list + detail pane? (Builds so far: one list; one build put request left / reply right inside an entry.) | whole UI | after comparing builds |
| How "new message" is computed when the history was compacted or reordered. | §2.2 item 4 | after a long session |
| Keep an append-only JSONL log on disk, so traffic survives a restart and can be re-opened? | persistence | optional |

Settled by the first builds (now in §3): same port under `/__monitor/`, SSE, the `monitor.py`
split, side requests by content, Claude Code only.

## 7. Observed traffic (Claude Code 2.1.311, one "read a file" turn)

So the builder knows what to expect; the dashboard must not depend on exact counts.

- Endpoint `POST /v1/messages?beta=true`, streamed (`stream: true`). Startup also sends
  `HEAD /api/hello` and a `"quota"` request with `max_tokens: 1` (which got a 429 here).
- One question produced 6 POSTs: a session-title request and the first main step at the same
  second, then 3 tool steps (Bash failed, Bash, Read), the final answer, then a suggestion-mode
  request. Messages grew 2 → 4 → 6 → 8 → 10.
- `system` is 3 blocks: a billing header line, "You are Claude Code…", then ~12 K characters;
  the last two are cached with a 1 h TTL. A further ~60 K characters of instructions arrive as a
  `system`-role message inside `messages`, plus `tool_addition` blocks.
- 16 tools in `tools`, including a `DeferredToolPlaceholder` with `defer_loading: true`.
- Other request fields: `thinking: {type: adaptive}`, `context_management` (clear thinking),
  `output_config` (effort; the title request uses a JSON schema), `metadata.user_id` (JSON string
  with device, account, session ids).
- Reply events: `message_start`, `content_block_start` (types `text`, `thinking`, `tool_use`),
  `content_block_delta`, `content_block_stop`, `message_delta` (stop reason, usage), `ping`,
  `message_stop`.
- Tool ids pair across requests: `tool_use.id` in a reply = `tool_result.tool_use_id` in the next
  request.
