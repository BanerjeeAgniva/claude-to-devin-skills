---
name: event-map
description: Build an interactive event map — producers → exchanges/topics → consumers with curved arrows, hover-to-trace, a pinned detail panel and a generated per-module table — as one self-contained HTML file verified line-by-line against code. Use for RabbitMQ/Kafka/SQS/Spring-event topology, "who publishes / who consumes", dead or orphaned channels.
argument-hint: "[scope: repo, module or package] [output path]"
---

# Event Map

Scope for this run: **$ARGUMENTS** (if empty: the whole repository; output to `docs/architecture/event-map.html`).

## Outcome

One HTML file that opens offline (fonts degrade gracefully) and shows **every** message edge in scope:

- **Left column** — modules that publish (rounded rectangles).
- **Middle column** — channels: exchanges / topics (hexagons).
- **Right column** — modules with a queue / consumer group bound (rounded rectangles), plus a `∅ no consumer` sink.
- **Curved arrows** (horizontal cubic Bézier, arrowheads via SVG `<marker>`), labelled with the producer class on the left and the queue name on the right. Labels are hidden at rest and appear on the highlighted trace.
- **Hover** previews a trace (everything else dims to ~7%); **click** pins it; `Esc` or empty-space click clears; `Tab` walks nodes. A module lights up on **both** sides, so one hover answers "what does M10 produce and consume".
- **Detail panel** under the diagram, **per-module table** (generated from the same data), **findings** (authored), optional **glossary**, and a **provenance footer**.

The template below already does all of this. Your job is the research and the `DATA` block, not the rendering.

## Procedure

1. Read the repo's own guidance first (`CLAUDE.md`, `AGENTS.md`, `README`, `docs/`). Treat every topology claim in them as a claim to verify, not as fact.
2. Find every channel **declaration**. For Spring/Java start with:
   - `rg -n "TopicExchange|DirectExchange|FanoutExchange|ExchangeBuilder|QueueBuilder|new Queue\(|BindingBuilder|Declarables" --type java`
   - `rg -n "NewTopic|TopicBuilder" --type java` (Kafka)
   - every constant class that names exchanges/topics (`rg -n "EVENTS\s*=|_EXCHANGE\s*=|TOPIC\s*=" --type java`)
3. Find every **publish** call and trace it to the module that owns the calling class:
   - `rg -n "convertAndSend|rabbitTemplate\.send|kafkaTemplate\.send|StreamBridge|\.publish\(" --type java`
   - If publishes go through a wrapper (e.g. `RabbitEventPublisher`), find the wrapper's callers instead.
4. Find every **consumer**: `rg -n "@RabbitListener|@KafkaListener|@SqsListener|@StreamListener|@EventListener" --type java`. For each, record the queue/group and check `autoStartup`, `@ConditionalOnProperty`, `@Profile` and feature flags.
5. Write an evidence ledger to a scratch file (`/tmp/event-ledger.csv`), one row per edge, **each with `file:line`**:
   `kind(pub|sub), module, channel_constant, broker_name, class, queue, file:line, flags`
6. Classify every channel and edge (these drive the colours — get them right):
   | Class | Rule | Rendered as |
   |---|---|---|
   | live | ≥1 publisher and ≥1 bound consumer | grey edge |
   | `seam` (PUBS kind) | publisher is a module that does not own the channel (a bridge) | dashed edge |
   | `dormant` (SUBS kind) | consumer exists but `autoStartup=false` / flag-off / profile-off | dashed edge |
   | `dead` (CHANNELS flag + SUBS row to `'NONE'`) | published to, but no queue bound | red hexagon, red edge to `∅ no consumer` |
   | `orphan` (CHANNELS flag) | constant exists, no publisher, no declaration, no consumer | red dashed hexagon, "orphan — no edges" tag |
   | `warn` (CHANNELS flag) | naming / ownership / doc drift — the code works but something lies | amber hexagon |
   | `bad` (MODULES flag) | module whose output reaches nobody | red outlined module box |
7. Copy the template: `cp .agents/skills/event-map/templates/event-map.html <output>` if that file exists; otherwise write the template at the end of this skill **verbatim** to `<output>`.
8. Edit **only**: `<title>`, `<h1>`, `.sub`, the findings `<ol>`, the glossary rows, the footer, and the `DATA` block (`CONFIG`, `MODULES`, `CHANNELS`, `PRODUCERS`, `CHANNEL_ORDER`, `CONSUMERS`, `PUBS`, `SUBS`). Do not touch CSS or the renderer.
   - Order columns to minimise crossings: channels grouped by their main producer; consumers ordered by the first channel they read; problem channels (`dead`, `orphan`) last; `'NONE'` last in `CONSUMERS`. If still tangled, set `CONFIG.autoOrder = true`.
   - For Kafka set `CONFIG.cols = ['PRODUCES','TOPIC','CONSUMES']` and `CONFIG.noun = {channel:'topic', inbox:'group', module:'module'}`.
   - Put counts that come from code rather than the drawing (e.g. number of `@RabbitListener` methods) in `CONFIG.extraStats`.
9. Write 5–8 findings, most severe first. Each one: a **bold one-sentence claim**, then the evidence (constant, class, `file:line`), then the consequence or the fix. Patterns worth a finding: channels with >1 producer, dead/orphan channels, dormant listeners, kill-switch flags on publishes, doc/code drift, replay or DLQ re-injection paths, modules with no broker role.
10. Write the footer: date, branch, and exactly which classes/annotations were swept.
11. Verify in the browser (see Specifications). Serve the folder (`python3 -m http.server 8765`), open the page, read the console, hover every flagged node, toggle the theme button through light and dark, and resize to 390 px wide. Fix anything off, re-check.
12. Deliver: commit the file, open a PR titled `docs: event map (<scope>)`, and post one full-page screenshot plus one screenshot of a highlighted trace in the PR description and in chat. In chat, list the findings in one line each.

## Specifications

- The page is a single `.html` file with inline CSS and JS; the only external requests are Google Fonts (IBM Plex Sans + Mono).
- The browser console shows **no errors** (the renderer logs `PUBS row references unknown id` / `SUBS row references unknown id` for any typo).
- Every `PUBS` and `SUBS` row matches a ledger row with a `file:line`. Stats in the header equal the ledger totals.
- Hovering any node highlights exactly its edges; clicking pins; `Esc` clears; the table row for a module highlights with it.
- Both themes are legible: light (`--bg #f5f7fa`) and dark (`--bg #14181d`), via `prefers-color-scheme` and the theme button.
- At 390 px wide the page has no horizontal page scroll; the diagram scrolls sideways inside its card.
- Findings are ordered by severity and every one names its evidence.

## Advice

- **The data block is the single source of truth.** Diagram, panel, table and stats are all generated from it, so they can never disagree. Never hand-write table rows or stat numbers that the renderer can compute.
- **Verify, don't infer.** A constant named `orders.tile_queue_depth` may be produced by dispatch, not orders. Names, javadoc and `CLAUDE.md` drift; the `convertAndSend` call does not.
- **Bridges matter.** If several modules publish to one channel, mark the non-owners `seam` and say so in a finding — that is where debugging goes wrong ("the emitter could be any of four modules").
- **A dead channel is a finding, not a footnote.** An exchange with no binding silently drops every message.
- **Keep labels short.** Producer class names on the left arrows, queue names on the right. Put long explanations in `note` (shown in the panel) instead.
- **Node names fit their box**: module boxes are 170 px (≈ 20 characters at 12.5 px), channel hexagons 240 px (≈ 30 monospace characters). Shorten with a module prefix (`M4 orders`) rather than widening.
- Write findings like an engineer's code review: present tense, specific, no hedging. "The code is right; the labels are stale." "Build the service or delete the constant."

### Project notes — 1dd / Godspeed (`one_day_delivery`)

- Transport is **RabbitMQ** even where packages say `kafka` (historical). Publishes go through `common/kafka/RabbitEventPublisher` (routing key = event-type name, bindings use `#`).
- Declarations live in the `*MessagingTopology` classes plus `BarcodeMessagingConfig`. Kill-switch flags live in properties classes such as `DispatchProperties`.
- Module labels: M1 auth · M2 pricing · M3 grid · M4 orders · M5 dispatch · M6 routing · M7 hub · M8 barcode · M9 airline · M10 sla · M11 exceptions · M13 assets.
- `DlqReplayServiceImpl` can re-publish parked messages — mention replay paths in findings, don't draw them.

## Forbidden actions

- Do not restyle, rename CSS classes, change the colour tokens or swap the fonts.
- Do not add a framework, bundler, chart library or any external script.
- Do not draw an edge you have not seen in code. No "probably", no "should".
- Do not split the page into multiple files or move the data into a separate JSON file.
- Do not hand-edit the generated table or the stats.

## Template — `event-map.html`

Copy this verbatim, then edit only the parts listed in step 8. The sample data is the real 1DD map as of 2026-10-24.

````html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover">
<!-- EDIT: 2–4 word name, no "X: explainer" -->
<title>1DD Event Map</title>
<meta name="description" content="Every message-broker edge in the system, verified against code: who publishes, through which exchange, to which consumer.">
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=IBM+Plex+Sans:wght@400;500;600&family=IBM+Plex+Mono:wght@400;500&display=swap">
<style>
  /* ════════════════════════════════════════════════════════════════════════════════════
     EVENT MAP — single column. The flow diagram is the page; a detail panel under it answers
     "what does the thing I'm pointing at touch"; the table + findings carry the record at rest.
     Do not restyle. Change content only in the DATA block at the bottom.
     ════════════════════════════════════════════════════════════════════════════════════ */
  :root{
    --bg:#f5f7fa; --fg:#1b2129; --muted:#5e6975; --line:#d9dfe7; --card:#ffffff; --chip:#eef2f6;
    --accent:#2c5f93; --ok:#3b7a57; --warn:#9c6b1c; --bad:#a7463b; --edge:#8a95a3;
    --font-ui:"IBM Plex Sans",system-ui,-apple-system,"Segoe UI",sans-serif;
    --font-mono:"IBM Plex Mono",ui-monospace,SFMono-Regular,Menlo,Consolas,monospace;
    color-scheme:light;
  }
  @media (prefers-color-scheme:dark){:root:not([data-theme="light"]){
    --bg:#14181d; --fg:#e7ebf0; --muted:#98a2ae; --line:#2a313a; --card:#1a1f26; --chip:#20262e;
    --accent:#82b4e6; --ok:#7cc79b; --warn:#d9a95a; --bad:#e3907f; --edge:#6b7684; color-scheme:dark;
  }}
  :root[data-theme="dark"]{
    --bg:#14181d; --fg:#e7ebf0; --muted:#98a2ae; --line:#2a313a; --card:#1a1f26; --chip:#20262e;
    --accent:#82b4e6; --ok:#7cc79b; --warn:#d9a95a; --bad:#e3907f; --edge:#6b7684; color-scheme:dark;
  }
  *{box-sizing:border-box}
  html,body{margin:0}
  body{background:var(--bg); color:var(--fg); font:14px/1.5 var(--font-ui); -webkit-font-smoothing:antialiased}
  .wrap{max-width:1240px; margin:0 auto;
    padding-left:max(16px,env(safe-area-inset-left,0px)); padding-right:max(16px,env(safe-area-inset-right,0px));
    padding-block:30px 64px}
  header{display:flex; flex-wrap:wrap; align-items:flex-end; justify-content:space-between; gap:14px 28px; margin-bottom:18px}
  h1{font-size:24px; font-weight:600; letter-spacing:-.01em; margin:0; text-wrap:balance}
  .sub{color:var(--muted); font-size:13.5px; margin:4px 0 0; max-width:62ch}
  .sub a{color:var(--accent); text-decoration:none}
  .stats{display:flex; flex-wrap:wrap; gap:8px; align-items:center}
  .stat{background:var(--chip); border:1px solid var(--line); border-radius:8px; padding:6px 10px; font-size:12.5px; color:var(--muted); font-variant-numeric:tabular-nums}
  .stat b{color:var(--fg); font-weight:600}
  .stat.bad b{color:var(--bad)}
  .theme{font:500 12px var(--font-ui); color:var(--muted); background:var(--card); border:1px solid var(--line); border-radius:8px; padding:6px 10px; cursor:pointer}
  .theme:hover{color:var(--fg); border-color:var(--accent)}

  /* diagram */
  figure{margin:0}
  .dwrap{background:var(--card); border:1px solid var(--line); border-radius:12px; padding:10px 8px 6px; overflow-x:auto; -webkit-overflow-scrolling:touch}
  svg.map{display:block; width:100%; min-width:920px; height:auto; font-family:var(--font-ui)}
  figcaption{color:var(--muted); font-size:12.5px; margin:10px 2px 0; display:flex; flex-wrap:wrap; gap:6px 18px; align-items:center}
  .key{display:inline-flex; align-items:center; gap:6px; white-space:nowrap}
  .key i{display:inline-block; width:22px; height:0; border-top:2px solid var(--edge)}
  .key i.dash{border-top-style:dashed}
  .key i.bad{border-color:var(--bad)}
  .key i.warn{border-color:var(--warn)}
  .key i.hi{border-color:var(--accent)}

  .colhead{font:600 11px var(--font-ui); letter-spacing:.14em; fill:var(--muted)}
  .node{cursor:pointer; outline:none}
  .node rect,.node polygon{fill:var(--card); stroke:var(--line); stroke-width:1.3}
  .node text{fill:var(--fg); font-size:12.5px; font-weight:500}
  .node.x text{font-family:var(--font-mono); font-size:11.5px; font-weight:500}
  .node.warn polygon{stroke:var(--warn)}
  .node.dead polygon{stroke:var(--bad)} .node.dead text{fill:var(--bad)}
  .node.orphan polygon{stroke:var(--bad); stroke-dasharray:4 3} .node.orphan text{fill:var(--bad)}
  .node.sink rect{stroke:var(--bad); stroke-dasharray:4 3} .node.sink text{fill:var(--bad)}
  .node.flagbad rect{stroke:var(--bad)}
  .node:focus-visible rect,.node:focus-visible polygon{stroke:var(--accent); stroke-width:2.5}
  .edge{fill:none; stroke:var(--edge); stroke-width:1.3; opacity:.5; marker-end:url(#arr)}
  .edge.seam,.edge.dormant{stroke-dasharray:5 4}
  .edge.dead{stroke:var(--bad); marker-end:url(#arr-bad)}
  .elabel{font:10.5px var(--font-mono); fill:var(--muted); opacity:0; paint-order:stroke; stroke:var(--card); stroke-width:4px; stroke-linejoin:round; pointer-events:none}
  .tag{font:10px var(--font-mono); fill:var(--muted); opacity:0; paint-order:stroke; stroke:var(--card); stroke-width:3px; pointer-events:none}
  .tag.always{opacity:1}
  #arr path{fill:var(--edge)} #arr-hi path{fill:var(--accent)} #arr-bad path{fill:var(--bad)}
  /* focus state: dim everything, light the selection */
  svg.focus .edge{opacity:.07}
  svg.focus .edge.hi{opacity:1; stroke:var(--accent); stroke-width:2; marker-end:url(#arr-hi)}
  svg.focus .edge.hi.dead{stroke:var(--bad); marker-end:url(#arr-bad)}
  svg.focus .node{opacity:.28}
  svg.focus .node.hi{opacity:1}
  svg.focus .node.hi rect,svg.focus .node.hi polygon{stroke:var(--accent); stroke-width:2}
  svg.focus .node.hi.dead polygon,svg.focus .node.hi.sink rect,svg.focus .node.hi.orphan polygon{stroke:var(--bad)}
  svg.focus .elabel.hi{opacity:1}
  svg.pinned .node.pin rect,svg.pinned .node.pin polygon{stroke-width:2.6}
  @media (prefers-reduced-motion:no-preference){
    .edge,.node,.elabel{transition:opacity .15s ease, stroke .15s ease}
  }

  /* detail panel */
  .detail{margin-top:14px; background:var(--card); border:1px solid var(--line); border-radius:12px; padding:14px 16px; min-height:96px}
  .d-head{display:flex; align-items:baseline; gap:10px; flex-wrap:wrap}
  .d-kind{font:500 11px var(--font-mono); letter-spacing:.1em; text-transform:uppercase; color:var(--muted); background:var(--chip); border:1px solid var(--line); border-radius:20px; padding:2px 8px}
  .d-kind.pinned{color:var(--accent); border-color:var(--accent)}
  .detail h2{font-size:16px; font-weight:600; margin:0}
  .detail h2 code{font-family:var(--font-mono); font-size:14px; color:var(--accent)}
  .d-body{margin-top:8px; display:grid; grid-template-columns:repeat(auto-fit,minmax(260px,1fr)); gap:10px 22px; font-size:13.5px}
  .d-body > div{min-width:0}
  .d-lbl{font:600 11px var(--font-ui); letter-spacing:.12em; text-transform:uppercase; color:var(--muted); margin:0 0 4px}
  .d-body ul{margin:0; padding-left:16px}
  .d-body li{margin:2px 0}
  .d-body code{font-family:var(--font-mono); font-size:12.5px; color:var(--accent)}
  .d-note{grid-column:1/-1; color:var(--muted); font-size:13px; border-top:1px solid var(--line); padding-top:8px}
  .d-note.rest{border-top:0; padding-top:0}
  .d-note.bad{color:var(--bad)}
  .d-note.warn{color:var(--warn)}
  .hint{color:var(--muted); font-size:13px; margin:6px 0 0}
  .hint kbd{font:11.5px var(--font-mono); background:var(--chip); border:1px solid var(--line); border-radius:4px; padding:0 5px}

  /* sections */
  h2.sec{font-size:12px; font-weight:600; letter-spacing:.1em; text-transform:uppercase; color:var(--muted); margin:34px 0 10px; display:flex; align-items:center; gap:10px}
  h2.sec::after{content:""; flex:1; height:1px; background:var(--line)}
  .twrap{overflow-x:auto; -webkit-overflow-scrolling:touch; border:1px solid var(--line); border-radius:12px; background:var(--card)}
  table{border-collapse:collapse; width:100%; font-size:13.5px; min-width:720px}
  th,td{border-bottom:1px solid var(--line); padding:8px 11px; text-align:left; vertical-align:top}
  th{background:var(--chip); font-weight:600; font-size:12.5px}
  tr:last-child td{border-bottom:0}
  td code{font-family:var(--font-mono); font-size:12.5px; color:var(--accent)}
  #modtable tbody tr[data-mod]{cursor:pointer}
  tr.hi td{background:color-mix(in srgb, var(--accent) 9%, transparent)}
  td.mod{font-weight:600; white-space:nowrap}
  .dim{color:var(--muted)}
  .b{color:var(--bad); font-weight:600} .w{color:var(--warn); font-weight:600}
  ol.findings{margin:0; padding-left:22px; font-size:14px; display:grid; gap:9px; max-width:78ch}
  ol.findings li{padding-left:2px}
  ol.findings code{font-family:var(--font-mono); font-size:12.5px; color:var(--accent)}
  .gloss-lead{color:var(--muted); font-size:13.5px; margin:0 0 10px; max-width:80ch}
  .gloss-lead code{font-family:var(--font-mono); font-size:12.5px; color:var(--accent)}
  table.gloss{min-width:640px}
  table.gloss td.t{font-weight:600; white-space:nowrap}
  table.gloss td{font-size:13px}
  table.gloss em{font-style:normal; color:var(--fg); font-weight:500}
  .foot{color:var(--muted); font-size:12.5px; margin-top:28px; max-width:78ch}
  .foot code{font-family:var(--font-mono); font-size:12px}
  @media (max-width:640px){ h1{font-size:21px} }
</style>
</head>
<body>
<div class="wrap">
  <header>
    <div>
      <!-- EDIT: title + one-sentence scope. Say what was verified and against which branch. -->
      <h1 id="title">1DD Event Map</h1>
      <p class="sub">Every RabbitMQ edge in the monolith, verified against code: the exchange declaration, the actual publish call, and the actual listener. Hover or click a module or exchange to trace what it touches.</p>
    </div>
    <div class="stats" id="stats" aria-label="Totals"></div>
  </header>

  <figure>
    <div class="dwrap" id="dwrap"></div>
    <figcaption>
      <span id="capcols">Left: who publishes. Middle: the exchange. Right: who has a queue bound to it.</span>
      <span class="key"><i></i> live edge</span>
      <span class="key"><i class="dash"></i> seam / dormant</span>
      <span class="key"><i class="bad"></i> nothing consumes it</span>
      <span class="key"><i class="warn"></i> amber outline = see findings</span>
      <span class="key"><i class="hi"></i> highlighted trace</span>
    </figcaption>
  </figure>

  <section class="detail" id="detail" aria-live="polite">
    <div class="d-head"><span class="d-kind" id="dKind">At rest</span><h2 id="dName"></h2></div>
    <div class="d-body" id="dBody"></div>
    <p class="hint">Click to pin · <kbd>Esc</kbd> or click empty space to clear · <kbd>Tab</kbd> moves between nodes</p>
  </section>

  <h2 class="sec">Per module</h2>
  <div class="twrap"><table id="modtable">
    <thead><tr><th>Module</th><th id="thProd">Produces → exchange</th><th id="thCons">Consumes ← exchange (its queue)</th></tr></thead>
    <tbody></tbody>
  </table></div>

  <!-- EDIT: findings are AUTHORED, ordered by severity, each one a bold claim + the evidence (class / constant / file:line). -->
  <h2 class="sec">Findings from the sweep</h2>
  <ol class="findings">
    <li><b><code>SCAN_EVENTS</code> has four producers, not one.</b> M8's real scan-gun path plus three bridges each labelled <em>M8-SEAM</em> in their own javadoc: M7's dock scan, M5's hub-custody scans in no-van cities, and M9's four airport custody scans. All land on M4's single <code>ScanEventsConsumer</code>, which is what moves parcel state.</li>
    <li><b><code>TILE_QUEUE_DEPTH</code> is produced by M5, not M4.</b> The constant is still named <code>orders.tile_queue_depth</code>, but <code>DispatchProperties</code> states M5 is the sole producer. The code is right; the labels are stale.</li>
    <li><b>Three exchanges publish into the void.</b> <code>GRID_EVENTS</code>, <code>SLA_EVENTS</code> and <code>ASSET_EVENTS</code> have no queue bound to them, so the broker drops every message at the exchange.</li>
    <li><b><code>NOTIFICATIONS_EVENTS</code> is an orphan.</b> Nothing publishes to it, declares it or listens on it. Build the service or delete the constant.</li>
    <li><b>One dormant listener.</b> <code>sla.flight</code> is bound but its consumer is <code>autoStartup="false"</code>, so flight events pile up in that durable queue unread.</li>
  </ol>

  <!-- OPTIONAL: glossary. Keep the "In this map" column — it ties every term to something on the page. -->
  <h2 class="sec">Terminology</h2>
  <p class="gloss-lead">The flow is always <code>producer → exchange → (binding) → queue → consumer</code>: a module publishes once, the exchange copies the message into every queue bound to it, and each consumer reads its own copy.</p>
  <div class="twrap"><table class="gloss">
    <thead><tr><th>Term</th><th>What it is</th><th>In this map</th></tr></thead>
    <tbody>
      <tr><td class="t">Producer</td><td>A module that emits an event. It names the <em>exchange</em>, never the recipient.</td><td>Left column.</td></tr>
      <tr><td class="t">Exchange</td><td>The router. Copies each message to every queue bound to it.</td><td>Middle hexagons.</td></tr>
      <tr><td class="t">Queue</td><td>A consumer's private inbox. Each consumer gets its own, so one reader doesn't drain another's.</td><td>Labels on the right-side arrows.</td></tr>
      <tr><td class="t">Binding</td><td>The rule "copy messages from this exchange into this queue".</td><td>Every exchange → module arrow.</td></tr>
      <tr><td class="t">Fan-out</td><td>One publish → many queues.</td><td><code>HUB_EVENTS</code>: one producer, five inboxes.</td></tr>
    </tbody>
  </table></div>

  <!-- EDIT: provenance. Date, branch, and exactly what was grepped. -->
  <p class="foot">Verified 2026-10-24 against <code>main</code>: every <code>*MessagingTopology</code> class for declarations, every publish through <code>RabbitEventPublisher</code>, every <code>@RabbitListener</code> method.</p>
</div>

<script>
/* ═══════════════════════════════════ DATA — the only block you edit ═══════════════════════════════════
   One source feeds the diagram, the panel, the table and the stats. Every row must come from code you read.
   MODULES   id → {name, note?, flag?:'bad'}                         boxes on the left and right
   CHANNELS  id → {name, addr, carries, flag?:'warn'|'dead'|'orphan', note?}   hexagons in the middle
   PRODUCERS / CHANNEL_ORDER / CONSUMERS  top-to-bottom order of each column (order to minimise crossings,
             or set CONFIG.autoOrder = true). Put 'NONE' last in CONSUMERS if any channel has no consumer.
   PUBS      [moduleId, channelId, via (producer class), kind?:'seam']            left arrows
   SUBS      [channelId, moduleId|'NONE', inbox (queue / consumer group), kind?:'dormant'|'dead']  right arrows
   ════════════════════════════════════════════════════════════════════════════════════════════════════ */
var CONFIG = {
  cols:   ['PRODUCES', 'EXCHANGE', 'CONSUMES'],     // Kafka: ['PRODUCES','TOPIC','CONSUMES']
  noun:   {channel:'exchange', inbox:'queue', module:'module'},  // Kafka: {channel:'topic', inbox:'group', ...}
  sinkLabel: '∅ no consumer',
  noRole: 'M1 auth · M2 pricing · ops · shuttle · vision · app — no broker role; they talk through in-process ports.',
  extraStats: [{n:25, label:'listeners'}],          // counts that come from code, not from the drawing
  autoOrder: false,                                  // true = one barycentre pass to reduce crossings
  geom: {W:1180, TOP:66, ROW:44, NH:30, PX:40, PW:170, XX:470, XW:240, CX:970, CW:170}
};
var MODULES = {
  M4:{name:'M4 orders', note:'Sole consumer of SCAN_EVENTS; its ScanEventsConsumer is what advances parcel state.'},
  M5:{name:'M5 dispatch', note:'Both publishes are config kill-switches (default on).'},
  M8:{name:'M8 barcode', note:'Declares its exchange in BarcodeMessagingConfig; no queues of its own.'},
  M7:{name:'M7 hub', note:'hub.flight listener is autoStartup="true" in code; CLAUDE.md still says false.'},
  M6:{name:'M6 routing', note:'Publishes the cron schedule plus VAN_ARRIVED / VAN_RUNNING_LATE on the same exchange.'},
  M9:{name:'M9 airline', note:'The four airport custody scans are console-triggered per AWB.'},
  M11:{name:'M11 exceptions', note:'exceptions.da is the authoritative stream for attempt counting.'},
  M3:{name:'M3 grid', note:'GRID_EVENTS has no consumer today; alerts are dropped at the exchange.', flag:'bad'},
  M10:{name:'M10 sla', note:'SLA_EVENTS has no consumer; sla.flight listener is dormant.', flag:'bad'},
  M13:{name:'M13 assets', note:'ASSET_EVENTS has no consumer; downstream inboxes are deferred.', flag:'bad'}
};
var CHANNELS = {
  SHIP:{name:'SHIPMENTS_EVENTS', addr:'oneday.shipments.events', carries:'CREATED · STATE_CHANGED · CANCELLED · MEASUREMENT_RECORDED'},
  DCONF:{name:'DELIVERY_CONFIRMATIONS', addr:'oneday.delivery.confirmations', carries:'RECEIVER_REJECTED'},
  DA:{name:'DA_EVENTS', addr:'oneday.da.events', carries:'DaLifecycleEvent (pickup / drop / cron / attendance)'},
  TQD:{name:'TILE_QUEUE_DEPTH', addr:'orders.tile_queue_depth', carries:'per-tile queue depth for M3 sizing', flag:'warn', note:'Produced by M5, not M4, despite the "orders." name. Rename the constant.'},
  SCAN:{name:'SCAN_EVENTS', addr:'oneday.scan.events', carries:'ScanEvent (custody scans)', flag:'warn', note:'Four producers: M8 proper plus M8-SEAM bridges in M5, M7 and M9. Only M4 consumes.'},
  HUB:{name:'HUB_EVENTS', addr:'oneday.hub.events', carries:'PARCEL_SORTED_FOR_DELIVERY · BAG_SEALED · HUB_OVERLOAD_ALERT …'},
  CRON:{name:'CRON_EVENTS', addr:'oneday.cron.events', carries:'DA_CRON_SCHEDULED · VAN_ARRIVED · VAN_RUNNING_LATE …'},
  FLT:{name:'FLIGHT_EVENTS', addr:'oneday.flight.events', carries:'FLIGHT_REASSIGNED · FLIGHT_TIME_CHANGED …'},
  EXC:{name:'EXCEPTIONS_EVENTS', addr:'oneday.exceptions.events', carries:'RTO_INITIATED · reschedule / reassign'},
  GRID:{name:'GRID_EVENTS', addr:'oneday.grid.events', carries:'TILE_OVERLOAD_ALERT · NO_DA_ALERT', flag:'dead', note:'No queue is bound to this exchange. Every alert is dropped.'},
  SLA:{name:'SLA_EVENTS', addr:'oneday.sla.events', carries:'breaches · escalations', flag:'dead', note:'No queue is bound. Intended readers are not wired.'},
  ASSET:{name:'ASSET_EVENTS', addr:'oneday.asset.events', carries:'asset lifecycle', flag:'dead', note:'No queue is bound; downstream inboxes are deferred.'},
  NOTIF:{name:'NOTIFICATIONS_EVENTS', addr:'oneday.notifications.events', carries:'(nothing)', flag:'orphan', note:'Orphaned constant: no publisher, no declaration, no listener.'}
};
var PRODUCERS     = ['M4','M5','M8','M7','M6','M9','M11','M3','M10','M13'];
var CHANNEL_ORDER = ['SHIP','DCONF','DA','TQD','SCAN','HUB','CRON','FLT','EXC','GRID','SLA','ASSET','NOTIF'];
var CONSUMERS     = ['M5','M11','M10','M6','M3','M4','M7','M9','NONE'];
var PUBS = [
  ['M4','SHIP','ShipmentEventProducer · MeasurementEventProducer'],
  ['M4','DCONF','DeliveryConfirmationServiceImpl'],
  ['M5','DA','DaEventProducer (kill-switch, on)'],
  ['M5','TQD','TileQueueDepthPublisher (kill-switch, on)'],
  ['M5','SCAN','HubScanSeamProducer','seam'],
  ['M8','SCAN','ScanEventProducer'],
  ['M7','HUB','HubEventProducer'],
  ['M7','SCAN','HubArrivalScanProducer','seam'],
  ['M6','CRON','CronEventProducer · RouteDeviationProducer'],
  ['M9','FLT','FlightEventProducer'],
  ['M9','SCAN','CustodyScanProducer','seam'],
  ['M11','EXC','ExceptionEventProducer'],
  ['M3','GRID','NoDaAlertProducer · TileOverloadAlertProducer'],
  ['M10','SLA','SlaEventProducer'],
  ['M13','ASSET','AssetEventProducer']
];
var SUBS = [
  ['SHIP','M5','m5.shipments'],['SHIP','M10','sla.shipments'],['SHIP','M11','exceptions.shipments'],
  ['DCONF','M5','m5.delivery-confirmations'],['DCONF','M11','exceptions.delivery_confirmations'],
  ['DA','M4','orders.da'],['DA','M6','routing.da'],['DA','M10','sla.da'],['DA','M11','exceptions.da'],
  ['TQD','M3','grid.tile-queue-depth'],
  ['SCAN','M4','orders.scan'],
  ['HUB','M4','orders.hub'],['HUB','M5','m5.hub'],['HUB','M6','routing.hub'],['HUB','M9','airline.hub'],['HUB','M10','sla.hub'],
  ['CRON','M4','orders.cron'],['CRON','M5','m5.cron'],['CRON','M10','sla.cron'],
  ['FLT','M4','orders.flight'],['FLT','M7','hub.flight'],['FLT','M10','sla.flight','dormant'],
  ['EXC','M4','orders.exceptions'],['EXC','M10','sla.exceptions'],
  ['GRID','NONE','no queue bound','dead'],['SLA','NONE','no queue bound','dead'],['ASSET','NONE','no queue bound','dead']
];
/* ═══════════════════════════════ RENDERER — do not edit below this line ═══════════════════════════════ */
(function(){
  'use strict';
  var G = CONFIG.geom, NS = 'http://www.w3.org/2000/svg';
  var N = CONFIG.noun;
  function el(n,a,txt){ var e=document.createElementNS(NS,n); for(var k in a) e.setAttribute(k,a[k]); if(txt!=null) e.textContent=txt; return e; }
  function r1(v){ return Math.round(v*10)/10; }
  function esc(s){ return String(s).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;'); }
  function cap(s){ return s.charAt(0).toUpperCase()+s.slice(1); }

  // ── sanity: every edge must reference declared ids (fails loudly in the console) ──
  PUBS.forEach(function(p){ if(!MODULES[p[0]]||!CHANNELS[p[1]]) console.error('PUBS row references unknown id',p); });
  SUBS.forEach(function(s){ if(!CHANNELS[s[0]]||(s[1]!=='NONE'&&!MODULES[s[1]])) console.error('SUBS row references unknown id',s); });

  // ── optional barycentre ordering: channels by mean producer index, consumers by mean channel index ──
  if(CONFIG.autoOrder){
    var pi={}; PRODUCERS.forEach(function(id,i){pi[id]=i;});
    var bc=function(list,score){ return list.slice().sort(function(a,b){ return score(a)-score(b); }); };
    var mean=function(a){ return a.length? a.reduce(function(s,v){return s+v;},0)/a.length : 1e9; };
    CHANNEL_ORDER = bc(CHANNEL_ORDER,function(c){ return mean(PUBS.filter(function(p){return p[1]===c;}).map(function(p){return pi[p[0]];})); });
    var ci={}; CHANNEL_ORDER.forEach(function(id,i){ci[id]=i;});
    CONSUMERS = bc(CONSUMERS,function(m){ return m==='NONE'?1e12:mean(SUBS.filter(function(s){return s[1]===m;}).map(function(s){return ci[s[0]];})); });
  }

  // ── stats (computed, so they never drift from the drawing) ──
  var live = CHANNEL_ORDER.filter(function(c){ return PUBS.some(function(p){return p[1]===c;}); }).length;
  var noCons = CHANNEL_ORDER.filter(function(c){ return CHANNELS[c].flag==='dead'; }).length;
  var orph = CHANNEL_ORDER.filter(function(c){ return CHANNELS[c].flag==='orphan'; }).length;
  var stats = [[CHANNEL_ORDER.length, N.channel+'s'],[live,'live'],[noCons,'with no consumer','bad'],[orph,'orphaned','bad']]
    .concat((CONFIG.extraStats||[]).map(function(s){return [s.n,s.label];}))
    .concat([[PUBS.length+SUBS.length,'edges']]);
  document.getElementById('stats').innerHTML = stats.filter(function(s){ return !(s[2]&&s[0]===0); })
    .map(function(s){ return '<span class="stat'+(s[2]?' '+s[2]:'')+'"><b>'+s[0]+'</b> '+esc(s[1])+'</span>'; }).join('')
    + '<button class="theme" id="themeBtn" type="button" title="Cycle theme">Theme: auto</button>';

  // ── theme toggle (auto → light → dark) ──
  (function(){
    var order=['auto','light','dark'], btn=document.getElementById('themeBtn'), cur='auto';
    try{ cur=localStorage.getItem('em-theme')||'auto'; }catch(e){}
    function set(t){ cur=t; if(t==='auto') document.documentElement.removeAttribute('data-theme'); else document.documentElement.setAttribute('data-theme',t);
      btn.textContent='Theme: '+t; try{ localStorage.setItem('em-theme',t); }catch(e){} }
    btn.addEventListener('click',function(){ set(order[(order.indexOf(cur)+1)%3]); }); set(cur);
  })();

  // ── geometry ──
  var span = G.ROW*(CHANNEL_ORDER.length-1);
  var H = G.TOP + span + G.NH + 26;
  var pos = {};
  function spread(list,i){ return list.length>1 ? G.TOP+i*(span/(list.length-1)) : G.TOP+span/2; }
  CHANNEL_ORDER.forEach(function(id,i){ pos['x:'+id] = {x:G.XX, y:G.TOP+i*G.ROW, w:G.XW}; });
  PRODUCERS.forEach(function(id,i){ pos['p:'+id] = {x:G.PX, y:spread(PRODUCERS,i), w:G.PW}; });
  CONSUMERS.forEach(function(id,i){ pos['c:'+id] = {x:G.CX, y:spread(CONSUMERS,i), w:G.CW}; });

  var svg = el('svg',{viewBox:'0 0 '+G.W+' '+H, class:'map', role:'img',
    'aria-label':'Three-column flow: producing '+N.module+'s on the left, '+N.channel+'s in the middle, consuming '+N.module+'s on the right.'});
  var defs = el('defs');
  ['arr','arr-hi','arr-bad'].forEach(function(id){
    var mk = el('marker',{id:id, viewBox:'0 0 10 10', refX:'9', refY:'5', markerWidth:'7', markerHeight:'7', orient:'auto-start-reverse'});
    mk.appendChild(el('path',{d:'M0,0 L10,5 L0,10 z'})); defs.appendChild(mk);
  });
  svg.appendChild(defs);
  svg.appendChild(el('text',{x:G.PX, y:34, class:'colhead'}, CONFIG.cols[0]));
  svg.appendChild(el('text',{x:G.XX+G.XW/2, y:34, class:'colhead', 'text-anchor':'middle'}, CONFIG.cols[1]));
  svg.appendChild(el('text',{x:G.CX+G.CW, y:34, class:'colhead', 'text-anchor':'end'}, CONFIG.cols[2]));
  var gEdges = el('g'), gLabels = el('g'), gNodes = el('g');
  svg.appendChild(gEdges); svg.appendChild(gLabels); svg.appendChild(gNodes);

  // edges: horizontal cubic Bézier, control points at the x-midpoint → the signature "S" curve
  var edges = [];
  function curve(a,b){
    var x1=a.x+a.w, y1=a.y+G.NH/2, x2=b.x, y2=b.y+G.NH/2, mx=(x1+x2)/2;
    return {d:'M'+x1+','+r1(y1)+' C'+mx+','+r1(y1)+' '+mx+','+r1(y2)+' '+x2+','+r1(y2), lx:mx, ly:r1((y1+y2)/2)-5};
  }
  function addEdge(from,to,label,kind,mod,ch,side){
    var c = curve(pos[from],pos[to]);
    var p = el('path',{d:c.d, class:'edge'+(kind?' '+kind:'')});
    var t = el('text',{x:c.lx, y:c.ly, class:'elabel', 'text-anchor':'middle'}, label);
    gEdges.appendChild(p); gLabels.appendChild(t);
    edges.push({el:p, label:t, mod:mod, ch:ch, side:side});
  }
  PUBS.forEach(function(e){ addEdge('p:'+e[0],'x:'+e[1],e[2],e[3],e[0],e[1],'p'); });
  SUBS.forEach(function(e){ addEdge('x:'+e[0],'c:'+e[1],e[2],e[3],e[1],e[0],'c'); });

  var nodes = {};
  function addNode(id,label,cls,title){
    var p = pos[id], g = el('g',{class:'node '+cls, tabindex:'0', role:'button', 'data-id':id, 'aria-label':title});
    g.appendChild(el('title',{},title));
    if(cls.split(' ')[0]==='x'){
      var x=p.x, y=p.y, w=p.w, h=G.NH, c=9;   // hexagon = channel
      g.appendChild(el('polygon',{points:[x+c,y, x+w-c,y, x+w,y+h/2, x+w-c,y+h, x+c,y+h, x,y+h/2].join(' ')}));
    } else {
      g.appendChild(el('rect',{x:p.x, y:p.y, width:p.w, height:G.NH, rx:7}));   // rounded rect = module
    }
    g.appendChild(el('text',{x:p.x+p.w/2, y:p.y+G.NH/2+4.5, 'text-anchor':'middle'}, label));
    gNodes.appendChild(g); nodes[id]=g; return g;
  }
  PRODUCERS.forEach(function(id){ var m=MODULES[id]; addNode('p:'+id, m.name, 'm'+(m.flag?' flag'+m.flag:''), m.name+' — publishes'); });
  CONSUMERS.forEach(function(id){
    if(id==='NONE') addNode('c:NONE', CONFIG.sinkLabel, 'm sink', 'No '+N.inbox+' is bound: messages are dropped');
    else addNode('c:'+id, MODULES[id].name, 'm', MODULES[id].name+' — consumes');
  });
  CHANNEL_ORDER.forEach(function(id){
    var x=CHANNELS[id];
    addNode('x:'+id, x.name, 'x'+(x.flag?' '+x.flag:''), x.name+(x.addr?' ('+x.addr+')':''));
    if(x.flag==='orphan'){ var p=pos['x:'+id]; gLabels.appendChild(el('text',{x:p.x+p.w+10, y:p.y+G.NH/2+4, class:'tag always'},'orphan — no edges')); }
  });
  document.getElementById('dwrap').appendChild(svg);

  // ── highlight: a module lights up on BOTH sides; a channel lights its producers and consumers ──
  var pinned = null;
  function related(kind,id){
    var E=[], Nn={};
    if(kind==='m'){
      Nn['p:'+id]=1; Nn['c:'+id]=1;
      edges.forEach(function(e){ if(e.mod===id){ E.push(e); Nn['x:'+e.ch]=1; } });
    } else {
      Nn['x:'+id]=1;
      edges.forEach(function(e){ if(e.ch===id){ E.push(e); Nn[e.side+':'+e.mod]=1; } });
    }
    return {E:E, N:Nn};
  }
  function clearHi(){
    svg.classList.remove('focus');
    edges.forEach(function(e){ e.el.classList.remove('hi'); e.label.classList.remove('hi'); });
    for(var k in nodes) nodes[k].classList.remove('hi');
  }
  function apply(sel){
    clearHi();
    if(!sel){ renderRest(); markRows(null); return; }
    var kind = sel.charAt(0)==='x' ? 'x' : 'm', id = sel.slice(2);
    var r = related(kind,id);
    svg.classList.add('focus');
    r.E.forEach(function(e){ e.el.classList.add('hi'); e.label.classList.add('hi'); });
    for(var k in r.N) if(nodes[k]) nodes[k].classList.add('hi');
    renderPanel(kind,id); markRows(kind==='m'?id:null);
  }
  function setPin(sel){
    pinned = sel; svg.classList.toggle('pinned',!!sel);
    for(var k in nodes) nodes[k].classList.toggle('pin', k===sel || !!(sel && sel.charAt(0)!=='x' && k.charAt(0)!=='x' && k.slice(2)===sel.slice(2)));
    apply(sel); document.getElementById('dKind').classList.toggle('pinned',!!sel);
  }
  gNodes.addEventListener('mouseover',function(ev){ var g=ev.target.closest('.node'); if(g&&!pinned) apply(g.getAttribute('data-id')); });
  gNodes.addEventListener('mouseout',function(ev){ var g=ev.target.closest('.node'); if(g&&!pinned) apply(null); });
  gNodes.addEventListener('focusin',function(ev){ var g=ev.target.closest('.node'); if(g&&!pinned) apply(g.getAttribute('data-id')); });
  gNodes.addEventListener('focusout',function(){ if(!pinned) apply(null); });
  gNodes.addEventListener('click',function(ev){
    var g=ev.target.closest('.node'); if(!g) return; ev.stopPropagation();
    var id=g.getAttribute('data-id'); setPin(pinned===id?null:id);
  });
  gNodes.addEventListener('keydown',function(ev){
    var g=ev.target.closest('.node'); if(!g) return;
    if(ev.key==='Enter'||ev.key===' '){ ev.preventDefault(); var id=g.getAttribute('data-id'); setPin(pinned===id?null:id); }
  });
  svg.addEventListener('click',function(){ if(pinned) setPin(null); });
  document.addEventListener('keydown',function(ev){ if(ev.key==='Escape'&&pinned) setPin(null); });

  // ── detail panel ──
  var dKind=document.getElementById('dKind'), dName=document.getElementById('dName'), dBody=document.getElementById('dBody');
  var kindTag=function(k){ return k?' · <b>'+esc(k)+'</b>':''; };
  function renderRest(){
    dKind.textContent = 'At rest';
    dName.textContent = 'Point at a '+N.module+' or '+N.channel;
    dBody.innerHTML = '<div class="d-note rest">Hover previews a trace; click pins it so you can read this panel. A '+N.module+' lights up on <em>both</em> sides, so one hover answers "what does it produce and consume".</div>';
  }
  function renderPanel(kind,id){
    dKind.textContent = (pinned?'Pinned · ':'')+(kind==='m'?N.module:N.channel);
    var h='';
    if(kind==='m' && id==='NONE'){
      var dropped = SUBS.filter(function(s){return s[1]==='NONE';});
      dName.textContent = CONFIG.sinkLabel;
      h = '<div><p class="d-lbl">'+cap(N.channel)+'s that drop every message · '+dropped.length+'</p><ul>'
        + dropped.map(function(s){ return '<li><code>'+esc(CHANNELS[s[0]].name)+'</code></li>'; }).join('') + '</ul></div>'
        + '<div class="d-note bad">Nothing is bound to these '+N.channel+'s, so the broker discards what is published to them.</div>';
    } else if(kind==='m'){
      var m=MODULES[id], pr=PUBS.filter(function(p){return p[0]===id;}), su=SUBS.filter(function(s){return s[1]===id;});
      dName.textContent = m.name;
      h+='<div><p class="d-lbl">Produces → '+pr.length+' '+N.channel+(pr.length===1?'':'s')+'</p>'+(pr.length?'<ul>'+pr.map(function(p){return '<li><code>'+esc(CHANNELS[p[1]].name)+'</code> <span class="dim">via '+esc(p[2])+kindTag(p[3])+'</span></li>';}).join('')+'</ul>':'<span class="dim">nothing</span>')+'</div>';
      h+='<div><p class="d-lbl">Consumes ← '+su.length+' '+N.channel+(su.length===1?'':'s')+'</p>'+(su.length?'<ul>'+su.map(function(s){return '<li><code>'+esc(CHANNELS[s[0]].name)+'</code> <span class="dim">'+N.inbox+' '+esc(s[2])+kindTag(s[3])+'</span></li>';}).join('')+'</ul>':'<span class="dim">nothing</span>')+'</div>';
      if(m.note) h+='<div class="d-note'+(m.flag?' '+m.flag:'')+'">'+esc(m.note)+'</div>';
    } else {
      var x=CHANNELS[id], pr2=PUBS.filter(function(p){return p[1]===id;}), su2=SUBS.filter(function(s){return s[0]===id&&s[1]!=='NONE';});
      dName.innerHTML = '<code>'+esc(x.name)+'</code>'+(x.addr?' <span class="dim" style="font-weight:400;font-size:13px">'+esc(x.addr)+'</span>':'');
      h+='<div><p class="d-lbl">Producers · '+pr2.length+'</p>'+(pr2.length?'<ul>'+pr2.map(function(p){return '<li>'+esc(MODULES[p[0]].name)+' <span class="dim">'+esc(p[2])+kindTag(p[3])+'</span></li>';}).join('')+'</ul>':'<span class="b">none</span>')+'</div>';
      h+='<div><p class="d-lbl">Consumers · '+su2.length+'</p>'+(su2.length?'<ul>'+su2.map(function(s){return '<li>'+esc(MODULES[s[1]].name)+' <span class="dim">'+N.inbox+' '+esc(s[2])+kindTag(s[3])+'</span></li>';}).join('')+'</ul>':'<span class="b">none — dropped</span>')+'</div>';
      if(x.carries) h+='<div><p class="d-lbl">Carries</p><span class="dim">'+esc(x.carries)+'</span></div>';
      if(x.note) h+='<div class="d-note '+(x.flag==='warn'?'warn':'bad')+'">'+esc(x.note)+'</div>';
    }
    dBody.innerHTML=h;
  }

  // ── per-module table (generated from the same data) ──
  document.getElementById('thProd').textContent = 'Produces → '+N.channel;
  document.getElementById('thCons').textContent = 'Consumes ← '+N.channel+' (its '+N.inbox+')';
  var tb=document.querySelector('#modtable tbody'), rows={};
  var ORDER = PRODUCERS.concat(CONSUMERS.filter(function(c){ return c!=='NONE' && PRODUCERS.indexOf(c)<0; }));
  ORDER.forEach(function(id){
    var m=MODULES[id], pr=PUBS.filter(function(p){return p[0]===id;}), su=SUBS.filter(function(s){return s[1]===id;});
    var tr=document.createElement('tr'); tr.setAttribute('data-mod',id);
    tr.innerHTML='<td class="mod">'+esc(m.name)+'</td>'
      +'<td>'+(pr.length?pr.map(function(p){return '<code>'+esc(CHANNELS[p[1]].name)+'</code>'+(p[3]?' <span class="w">'+esc(p[3])+'</span>':'')+' <span class="dim">'+esc(p[2])+'</span>';}).join('<br>'):'<span class="dim">—</span>')+'</td>'
      +'<td>'+(su.length?su.map(function(s){return '<code>'+esc(CHANNELS[s[0]].name)+'</code> <span class="dim">'+esc(s[2])+'</span>'+(s[3]?' <span class="b">'+esc(s[3])+'</span>':'');}).join('<br>'):'<span class="dim">— none</span>')+'</td>';
    tr.addEventListener('mouseenter',function(){ if(!pinned) apply('p:'+id); });
    tr.addEventListener('mouseleave',function(){ if(!pinned) apply(null); });
    tr.addEventListener('click',function(){ setPin(pinned==='p:'+id?null:'p:'+id); });
    tb.appendChild(tr); rows[id]=tr;
  });
  if(CONFIG.noRole){
    var none=document.createElement('tr');
    none.innerHTML='<td class="mod dim">No broker role</td><td colspan="2" class="dim">'+esc(CONFIG.noRole)+'</td>';
    tb.appendChild(none);
  }
  function markRows(id){ for(var k in rows) rows[k].classList.toggle('hi', k===id); }
  renderRest();
})();
</script>
</body>
</html>
````
