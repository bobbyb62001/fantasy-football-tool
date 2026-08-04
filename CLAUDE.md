# CHAINS — Fantasy Football Draft Desk

## What this is

A single-file, client-side-only fantasy football draft assistant. No backend, no build step, no
framework — one `index.html` with inline `<style>` and `<script>`, hosted for free on GitHub
Pages. All data is fetched live from public APIs directly in the browser on page load and on a
5-minute auto-refresh timer.

**Repo file:** `index.html` (must stay named this, at repo root, for the clean GitHub Pages URL
`https://<user>.github.io/<repo>/`)

**Deploy process:** edit → commit/push to the repo's default branch → GitHub Pages rebuilds
automatically in 1-2 minutes. No CI, no npm install, no bundler. Anyone with the repo can
literally upload the file via GitHub's web UI — that's how the owner has been deploying it (no
local git usage on their end).

## Architecture

- Vanilla JS, ES2017+ (arrow functions, async/await, template-free string concatenation —
  deliberately avoided template literals in a few older sections, inconsistent but functional
  either way).
- Global `STATE` object holds all runtime data (player pool, roster, drafted set, sync status,
  etc.) — see the top of the `<script>` block for the full shape.
- No modules, no imports. Everything is one file, functions declared at top level (hoisted,
  so declaration order mostly doesn't matter).
- Persistence: `localStorage`, namespaced per "league" (see below). No cookies, no
  server-side storage, no accounts.

## Two tabs

**Draft Board** — My Leagues selector, Rankings Source (FantasyPros optional), Draft Sync
(Sleeper/ESPN/MFL/Yahoo/NFL.com/manual), Strategy selector, on-the-clock banner,
positional run alerts, league history (prior-season draft tendencies), multi-team draft grid,
Best Available strip, tiered/ranked player list with NEED/VALUE/BYE/injury badges and
Research links.

**My Roster** — Roster Settings (configurable slot counts, including superflex), My Roster list
with per-slot recommendation sentences for empty slots, roster tools (Undo/Copy/Reset),
Draft Summary (grades picks vs. expected round value), AI Assistant (optional, see below).

Start/Sit was built and then **deliberately removed** at the user's request — the tool is draft-
only now. Don't re-add it without being asked.

## Data sources (all free, all fetched client-side)

| Source | Used for | CORS status |
|---|---|---|
| `api.sleeper.app` | Player pool, injuries, rank (`search_rank`), 24h waiver trends, live draft picks, league/draft/roster/user metadata | ✅ Open, no key needed |
| `site.api.espn.com` | Live scores, team schedules → bye weeks | ✅ Open |
| `fantasy.espn.com/apis/v3/...` | ESPN draft sync (experimental) | ⚠️ Undocumented, CORS not guaranteed — code has a try/catch fallback to manual mode |
| `raw.githubusercontent.com/dynastyprocess/data` | ESPN player ID ↔ Sleeper player ID crosswalk (CSV, parsed client-side) | ✅ Open (GitHub raw serves `Access-Control-Allow-Origin: *`) |
| `myfantasyleague.com` | — | ❌ Confirmed blocked, both by their CORS policy and robots.txt. Do not try again without a proxy. Manual mode only. |
| `api.fantasypros.com` | Optional consensus rank override | ❌ Confirmed blocked (tested live with a real key — "likely blocked by browser CORS" in production). Needs a proxy (see Open Items) to ever work from the browser. |
| `api.anthropic.com` | Optional AI pick assistant | ✅ Works with `anthropic-dangerous-direct-browser-access: true` header + user's own API key |

**Pattern used throughout:** every "maybe this site doesn't allow browser access"
integration (ESPN sync, FantasyPros) is wrapped in try/catch with a clear status message
on failure and an automatic fallback — never a silent failure, never a crash. Keep this pattern
for any new integration.

## State persistence model

- `localStorage` key `chains_ff_v1` (base) — never used directly; always suffixed.
- `chains_ff_v1:__labels` — JSON array of saved league names (for the My Leagues dropdown).
- `chains_ff_v1:__lastLabel` — which league was last active, restored on page load.
- `chains_ff_v1:<label>` — one JSON blob per saved league: drafted set, roster, roster
  meta (round/keeper-cost per pick), roster slot config, undo history, site/draft IDs,
  league size, scoring format, strategy.
- `chains_ff_v1:__secrets` — FantasyPros + Anthropic API keys, global, not per-league
  (shared across all saved leagues on this browser/device).

Switching leagues in the dropdown (`setLeague()`) fully resets in-memory state and reloads
from that league's storage key — each league is fully isolated.

**Auto-reconnect:** on page load and on league switch, if a league has a saved site (`sleeper`
or `espn`) plus a saved draft/league ID, it reconnects and resumes polling automatically — no
manual "Connect" tap needed after the first time. (`autoReconnectIfPossible()`)

## Roster configuration

Roster slots are not hardcoded — `STATE.rosterSlots` is a mutable array, editable via the
Roster Settings panel (QB/RB/WR/TE/FLEX/SFLEX/K/DEF/BN counts → `buildRosterSlots()`).
Changing it mid-draft re-slots existing picks (`applyRosterConfig()`) and drops any that no
longer fit, with a count reported to the user. `DEFAULT_ROSTER_SLOTS` is the fallback (standard
12-team PPR: 1QB/2RB/2WR/1TE/1FLEX/1K/1DEF/6BN).

## Rank source

`rankOf(p)` is the single function everything (sort order, tiers, Best Available, value alerts,
keeper cost, AI prompt) reads rank through. It checks `STATE.fp.rankMap` (FantasyPros,
name-matched via `normalizeName()`) first, falls back to Sleeper's `search_rank`. If you add
another rank source, wire it into this one function rather than threading a new field through
every caller.

## Testing approach used during development

No test framework installed. Regressions were caught by writing throwaway Node scripts
using `jsdom` with `beforeParse(window)` to mock `window.fetch` before the page's own
`<script>` runs (critical — mocking fetch after JSDOM construction misses the initial
`init()` call entirely, a mistake made once and worth avoiding). Pattern:

```js
const { JSDOM } = require('jsdom');
const dom = new JSDOM(html, {
  runScripts: 'dangerously', resources: 'usable', url: 'https://example.com/',
  beforeParse(window){
    window.fetch = async (url) => { /* mock based on url.includes(...) */ };
    window.alert = ()=>{}; window.confirm = ()=>true; window.prompt = ()=>'x';
  }
});
setTimeout(() => { /* assert on window.document / window.eval('STATE...') */ }, 1500);
```

`npm install jsdom` in the working directory before running. No package.json was ever
committed — it's disposable test tooling, not part of the shipped artifact. Worth formalizing
into a real test file if this project keeps growing.

## Known open items

1. **FantasyPros needs a proxy to ever work.** Confirmed CORS-blocked in production
   with a real key. The user has a reminder set to look into a Cloudflare Worker proxy (free
   tier) when they have time. If you build this: the Worker holds the FP key as a secret, the
   page calls the Worker's URL instead of `api.fantasypros.com` directly,
   `connectFantasyPros()` just needs its fetch URL swapped.
2. **ESPN sync is unverified in a real draft** — logic is tested with mocked data only; the
   owner hadn't tried it live as of the last update.
3. **No point projections or true multi-expert consensus** — by design/budget choice,
   not an oversight. Sleeper's `search_rank` and (optionally) FantasyPros ECR are the rank
   signals; nothing here predicts weekly points.
4. **AI Assistant cost/security tradeoff was explicitly accepted by the user** after being
   warned twice — the Anthropic key sits in `localStorage` in plaintext, visible to anyone
   with access to that browser/device. Don't "fix" this by removing the feature; it was a
   deliberate, informed choice.

## Things NOT to reintroduce without being asked

- Start/Sit tab (removed on purpose)
- Any scraping of FantasyPros/RotoBaller/Walter Football content — those sites'
  rankings/articles are copyrighted; the tool only deep-links to them via `researchUrl()`,
  never reproduces their content
- `localStorage`/`sessionStorage` usage assumptions inside Claude.ai's own artifact
  preview — this file works there in a limited way but is designed to be self-hosted; some
  things (persistence, auto-reconnect) only fully work once deployed to GitHub Pages,
  not in a temporary Claude-hosted preview
