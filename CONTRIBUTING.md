# Contributing (guide for agents)

This repo holds one or more trip itineraries, each self-contained under
`trips/<city-slug>-<YYYY-MM>/`. Every trip folder has exactly three files:

- `trip.json` — the single source of truth (route, POIs, meals, transport, skipped items)
- `<city-slug>.html` — a self-contained static page rendering `trip.json` (map + itinerary)
- `<city-slug>.kml` — the same route for Google My Maps import

`index.html` at the repo root just links to each trip page. The whole repo is
published as a static site via GitHub Pages (branch `main`, root `/`) — any
push to `main` goes live in 1–2 minutes at the Pages URL.

**Golden rule: `trip.json` is authoritative. Never hand-edit content directly
inside `<city-slug>.html` — always edit `trip.json`, then re-inject it (see
below). The only things that belong directly in the HTML are structural/UI
changes (CSS, the rendering `<script>`) that apply to every trip, not
trip-specific content.**

## trip.json shape

```
{
  "meta": { city, country, dates, language, party, pace, generatedOn },
  "i18n": { ...UI strings in meta.language, see required keys below },
  "transport": { summary, ticketAdvice, apps: [...], facts: [{text, verified}] },
  "mymaps": { kmlFile, shareUrl },
  "days": [ { label, theme, items: [...], skipped: [...] } ]
}
```

Required `i18n` keys (rendering breaks if any is missing — the renderer falls
back to printing the raw key name):
`tripPlan, askClaude, copied, copyManually, alternative, min, fullDayRoute,
transport, tickets, apps, verified, unverified, onYourPhone, mymapsStep1,
mymapsStep2, mymapsStep3, breakfast, lunch, dinner, readMore, notIncluded`

### `days[].items[]` — ordered mix of:

**`type: "poi"`**
```
{ "type": "poi", "name", "coords": [lat, lng],
  "brief":   "one short punchy sentence — always visible",
  "summary": "longer history / what-to-look-for text — collapsed under a 'Подробнее' spoiler",
  "duration", "price", "hours",           // optional, one-line facts shown under the title
  "links": { "official"?, "wikipedia"?, "maps"? },  // maps auto-derives from coords if omitted
  "askPrompt": "prompt copied to clipboard / sent to the AI-bot menu",
  "alternatives": [ { name, coords, summary, when } ]  // optional, 0-2, own spoiler
}
```
`brief` is mandatory for the two-tier description format (see "Card format"
below) — a POI without `brief` falls back to showing `summary` in both places.

**`type: "meal"`**: `{ kind: "breakfast"|"lunch"|"dinner", options: [{ name, coords?, cuisine?, price?, note? }] }` (1–2 options).

**`type: "transit"`**: `{ mode, from, to, minutes, note? }`.

### `days[].skipped[]` — "not included" items

Anything considered but deliberately left out of the route (from a source
document, a prior draft, etc.) goes here instead of being silently dropped.
Same POI-like shape, plus a mandatory `reason`:
```
{ "name", "coords", "brief", "summary", "price"?, "links"?, "askPrompt", "reason": "why it didn't make the cut" }
```
These render in their own header tab ("🗒 Не вошло в маршрут"), grouped by
day — never inline in the day's own item list. `reason` should be honest and
specific (time conflict, duplicate niche, off-route, doesn't match stated
preferences, etc.), not a vague "didn't fit."

## Authoring rules (carried over from the original trip-planner skill, still apply)

1. **Coordinates from model knowledge**, 4+ decimals. Before adding any
   `coords` (POIs, alternatives, meal options, skipped items), sanity-check
   they fall inside the city's rough bounding box. For Berlin: lat
   52.33–52.68, lng 13.05–13.76. Still unsure → drop the coords rather than
   guess.
2. **Volatile facts** (prices, hours, closing days, transport fares) must be
   traceable to a real check, not invented. If unverified, phrase
   approximately ("~€10") and don't claim a `verified` date in
   `transport.facts`. Don't silently present a guess as settled fact — flag it
   in the text.
3. Everything user-visible stays in `meta.language`.
4. Valid strict JSON: no comments, no trailing commas, UTF-8.
5. **Escape every literal `<` in JSON string values as `<`.** `trip.json`
   gets injected into a `<script>` block; a literal `</script>` inside a
   string would break out of it. `<` is valid JSON and parses back to `<`.

## Card format (brief + spoiler)

Every POI and skipped-item card shows `brief` inline, with `summary` (plus
links, the Ask-AI button, and — for skipped items — the reason) collapsed
under a "▶ Подробнее" `<details>` spoiler. Don't revert to dumping the full
`summary` inline — that was an explicit user preference change from the
original skill template.

## The "Ask AI" menu

Each card's `askPrompt` renders as a "🤖 Спросить ИИ" button that expands a
menu of 5 bots (Claude, ChatGPT, Gemini, Grok, DeepSeek). Selecting one copies
`askPrompt` to the clipboard and opens that bot's site in a new tab with a
best-effort `?q=` prefill. **Only ChatGPT is confirmed to actually prefill the
message box** (verified live). Claude removed URL prefill in Oct 2025 for
security reasons; Gemini/Grok/DeepSeek have no documented support. The
clipboard copy is the fallback for all of them — don't remove it, and don't
claim in UI copy that prefill is guaranteed to work everywhere.

## Editing an existing trip

1. Edit `trip.json` directly (Edit tool, not regex).
2. Validate: `python -c "import json; json.load(open('trips/.../trip.json', encoding='utf-8'))"`.
3. Re-inject into the HTML. **Do not re-run the full template placeholder
   replacement** — the HTML's CSS/JS has been hand-customized beyond the
   stock skill template (brief/spoiler format, ask-menu, skipped tab), so
   re-templating from scratch would destroy those customizations. Instead,
   swap only the embedded data blob via a literal (non-regex) string
   replace:
   - Extract the currently-embedded `TRIP_DATA` JSON from the HTML (text
     between `const TRIP_DATA = ` and `\n;\n(function(){`).
   - `.Replace(oldEmbeddedJson, newTripJsonText)` on the full HTML file
     (PowerShell `.Replace()` or equivalent literal replace — never a regex
     substitution, since `$` sequences in JSON and Cyrillic content will
     corrupt a regex-based replace).
4. Sanity-check the result is still valid JS:
   `node -e "new Function(require('fs').readFileSync('trips/.../<city>.html','utf8').match(/<script>([\s\S]*?)<\/script>\s*<\/body>/)[1])"`.
5. If you only changed `trip.json` content (no template/JS/CSS changes), the
   two files' embedded data should be byte-identical after the swap — spot
   check with a `grep -c` on some unique substring, or diff the extracted
   blob against `trip.json`.
6. Preview locally before pushing: `python -m http.server` from the repo
   root, open `http://localhost:<port>/trips/.../<city>.html`, click through
   both day tabs and the "не вошло" tab, expand a spoiler, open the Ask-AI
   menu.
7. Only regenerate the `.kml` if POI coordinates, names, or route order
   actually changed — a pure text/wording edit doesn't need it.

## Structural/template changes (CSS, rendering `<script>`)

These affect every trip page, not just one trip's data. Make the change
directly in the relevant `<city>.html` file(s) (there's currently no shared
template file in this repo — each trip's HTML is a standalone artifact).
If there end up being multiple trips, keep their `<style>`/`<script>` blocks
in sync by hand, or consider factoring out a shared template at that point
rather than before it's needed.

## Git / publishing

- Commit `trip.json` and `<city>.html` together (they must stay in sync) plus
  `<city>.kml` if it changed.
- Push to `main` — GitHub Pages redeploys automatically, usually live within
  1–2 minutes.
- Don't add build tooling, a bundler, or a JS framework. The whole point of
  this repo is that each trip page is a single dependency-free HTML file
  (only external dependency: the Leaflet CDN for the map).
