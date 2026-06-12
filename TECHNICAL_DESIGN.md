# World Cup 2026 Schedule — Technical Design

An interactive FIFA World Cup 2026 schedule & prediction web app, live at
**https://funprojects.ai/wc2026**.

Users enter group scores and knockout results; the app computes group standings
with the real FIFA tiebreakers, resolves qualification (including the "best 8
third-placed teams" rule), advances winners through a full knockout bracket
(90 min → extra time → penalties), and traces any team's path to the final.
Everything runs in the browser; predictions are saved locally and can be shared.

---

## 1. High-level architecture

```
                         ┌──────────────────────────────────────────┐
   Build time            │  world_cup_2026_v1.3.xlsx  (schedule+rules)│
   (one-off, offline)    │  WC Schedule PDF.pdf        (venues)       │
                         └───────────────┬──────────────────────────┘
                                         │  scripts/export_wc_data.py (Python)
                                         ▼
                            src/data/tournament.json   ← committed static data
                                         │
                                         │  bundled by Vite
                                         ▼
   Runtime (browser)     ┌──────────────────────────────────────────┐
                         │  React SPA  (pure functions + components)  │
                         │  state ──► standings ──► qualification     │
                         │        └─► bracket ──► path-to-final       │
                         │  persistence: localStorage + URL hash      │
                         └───────────────┬──────────────────────────┘
                                         │  static hosting
                                         ▼
                         Firebase Hosting  (funprojects.ai/wc2026)
```

**Key idea:** the app is a **pure, deterministic function of one state object**.
All tournament data is precomputed into a static JSON file at build time, so the
runtime has **no backend, no database, and no API calls** for its core logic.
(The only network call from this app is the privacy-respecting usage beacon — see
§9 — which is independent of the predictor.)

---

## 2. Tech stack

| Layer | Choice |
|---|---|
| Framework | React 19 + TypeScript |
| Bundler | Vite 6 (`base: '/wc2026/'`, builds to `../public/wc2026`) |
| Styling | Hand-written CSS (`src/index.css`), "Stadium Night" theme; no UI library |
| Data prep | Python 3 + `openpyxl` (Excel) + `PyMuPDF` (PDF venue extraction) |
| Hosting | Firebase Hosting (static), served under `/wc2026/` |
| Flags | `flagcdn.com` `<img>` (no bundled image assets) |

There is **no runtime server** for the predictor. State lives entirely in the
browser.

---

## 3. Data pipeline (build time)

`scripts/export_wc_data.py` is run once (re-run only if the source files change)
and produces `src/data/tournament.json`, which is committed to the repo.

Sources:
1. **`world_cup_2026_v1.3.xlsx`** — teams + FIFA ranks, the 72 group fixtures
   (date/time/teams), the knockout bracket wiring, and the 495-row best-thirds
   lookup table.
2. **`WC Schedule PDF.pdf`** (official FIFA wall-chart) — the venue (host city)
   for each match. The city is encoded by each match's *row* in the grid, so the
   script reads the PDF text layer's word coordinates, maps each match to its
   city row, and **cross-validates** every assignment against the team names
   already known from the Excel.

The script validates its output (48 teams, 12×4 groups, 72 + 32 matches, 495
lookup rows, every match has a venue) before writing the JSON.

### `tournament.json` shape

```jsonc
{
  "teams":   [{ "name": "France", "fifaRank": 1877.32, "group": "I" }, ...],
  "groups":  { "A": ["Mexico", "Korea Republic", ...], ... },          // 12 groups
  "groupMatches": [
    { "no": 1, "kickoffUTC": "2026-06-11T19:00:00Z", "group": "A",
      "home": "Mexico", "away": "South Africa",
      "venue": { "stadium": "Estadio Azteca", "city": "Mexico City" } }, ...
  ],
  "knockout": [
    { "no": 73, "round": "R32", "kickoffUTC": "...",
      "homeSlot": "2A", "awaySlot": "2B", "thirdConstraint": null,
      "venue": { "stadium": "SoFi Stadium", "city": "Los Angeles" } }, ...
  ],
  "thirdPlaceTable": { "ADEFGIJL": { "1A": "3E", "1B": "3G", ... }, ... }, // 495 rows
  "tiebreakers": [ "...documentation..." ]
}
```

**Slot grammar** (knockout feeders, resolved at runtime):
`1A`/`2B` = group winner/runner-up · `T:1E` = the best-third assigned to the 1E
slot · `W74` = winner of match 74 · `L101` = loser of match 101.

---

## 4. State model

One object is the single source of truth (`src/lib/types.ts`):

```ts
interface PredictionState {
  scores: Record<number, Score>;                 // group match no -> { home, away }
  knockoutScores: Record<number, KnockoutScore>; // ko match no -> { reg, et?, pen? }
}
interface KnockoutScore { reg: Score; et?: Score; pen?: Score; }
```

Everything else (standings, qualifiers, bracket, champion, a team's path) is
**derived** from this object by pure functions and recomputed on every edit via
`useMemo`. Nothing is stored redundantly.

---

## 5. Core logic (`src/lib/`, framework-agnostic & unit-tested)

| Module | Responsibility |
|---|---|
| `standings.ts` | Group tables with the **FIFA tiebreaker chain**: ① points → ② head-to-head mini-league among the teams tied on points (pts → GD → GF *within that subset*) → ③ goal difference → ④ goals scored → ⑤ FIFA rank. |
| `qualification.ts` | 12 winners + 12 runners-up; rank all 12 third-placed teams, take the **best 8**, build the combination key (e.g. `ADEFGIJL`), and use `thirdPlaceTable` to slot each qualifying third into its Round-of-32 fixture. |
| `knockout.ts` | Derive a tie's winner from its score: 90′ decisive → win; else add extra-time goals (aggregate); else penalty shoot-out. Also `needsExtraTime` / `needsPens` to gate input. |
| `bracket.ts` | `resolveBracket(state)` walks rounds in order, resolves each fixture's two teams from group results + prior winners/losers, derives winners from scores, and cascades them to the Final. |
| `pathToFinal.ts` | `tracePath(team)` follows a team through the resolved bracket round-by-round. |
| `persistence.ts` | localStorage auto-save, compact share-URL encode/decode, and named saved-session storage. |
| `time.ts` | Convert UTC kickoff times to the visitor's (or a chosen) timezone via `Intl`. |
| `data.ts` / `flags.ts` | Typed access to `tournament.json`; team → flag-CDN code. |

### Tiebreaker detail (the subtle part)
Step ② is a **mini-league recomputed only among the teams currently tied on
points** — counting only the matches played *between* those teams — matching the
spreadsheet's `Concerned teams (Pnt, GF-GA, GF)` rule. This is why head-to-head
correctly beats overall goal difference.

### Knockout scoring detail
Each knockout tie has three phases entered in the UI: **90 Min**, **Extra Time**
(goals scored *during* ET, added to the 90′ score), and **Penalties**. Inputs are
gated: ET unlocks only when 90′ is level; penalties unlock only when still level
after ET. The winner is computed, not clicked, and propagates to the next round.

---

## 6. UI (`src/components/`)

| Component | Role |
|---|---|
| `App.tsx` | Holds state, hydrates from URL→localStorage, tab nav (Group Stage · Knockout Bracket · My Team's Path · By Date · By Venue; last tab remembered per device), header (logos, timezone, Save/Load, Share, Reset). |
| `GroupStage.tsx` + `StandingsTable.tsx` | 12 group cards: editable scorelines + live standings with qualification highlighting + venues. |
| `KnockoutBracket.tsx` | R32→Final columns of `Tie` widgets. |
| `ScheduleViews.tsx` | By Date / By Venue tabs: all 104 matches in collapsible sections — day sections bucketed in the *selected* timezone (with a Today badge + auto-scroll during the tournament), venue sections alphabetical by city. Fully editable: group rows and knockout ties write to the same shared state. |
| `GroupFixture.tsx` | One editable group fixture row (shared by `GroupStage` and `ScheduleViews`; optional group badge, time-only kickoff, hideable venue). |
| `Tie.tsx` | One knockout tie: team×phase (90′/ET/Pens) score grid, winner auto-highlighted (shared by `KnockoutBracket`, `PathFinder` and `ScheduleViews`; optional round tag). |
| `PathFinder.tsx` | Pick a team → see its qualification status and route to the final, highlighted on the bracket. |
| `SessionManager.tsx` | Save/name/load/rename/delete named sessions; export/import a session file; copy a resume link. |
| `Flag.tsx` | Flag image with graceful fallback. |

---

## 7. Persistence & sharing

- **Auto-save:** the whole `PredictionState` is written to `localStorage` on every
  change, so progress survives refreshes and return visits.
- **Named sessions:** multiple snapshots stored separately; users can save, load,
  rename, delete, and export/import a session as a `.json` file (move between
  devices).
- **Share / resume link:** the state is compactly encoded into the URL hash
  (`#share=…`), so a link reproduces an entire bracket on any device.
- **Timezone:** kickoff times are stored in UTC and rendered in the visitor's
  local timezone, with a manual override dropdown. The By Date view buckets
  matches by calendar day in the selected timezone.
- **Last tab:** the active tab is remembered per device (`wc2026-tab-v1`) —
  a UI preference deliberately kept out of prediction state, sessions, and
  share URLs, so switching views can never alter a saved bracket.

---

## 8. Project structure

```
wc2026-app/
├── scripts/
│   ├── export_wc_data.py     # Excel + PDF -> src/data/tournament.json (run once)
│   └── verify.ts             # headless logic checks
├── src/
│   ├── data/tournament.json  # generated, committed
│   ├── lib/                  # pure logic (standings, qualification, knockout,
│   │                         #   bracket, pathToFinal, persistence, time, ...)
│   ├── components/           # React UI
│   ├── assets/               # logo + ball images
│   ├── App.tsx, main.tsx, index.css
│   └── vite-env.d.ts
├── index.html
├── vite.config.ts            # base '/wc2026/', outDir '../public/wc2026'
├── package.json, tsconfig.json
└── TECHNICAL_DESIGN.md        # this file
```

The build output goes to the sibling `public/wc2026/` folder, which Firebase
Hosting serves at `funprojects.ai/wc2026`.

---

## 9. Usage analytics (separate, optional subsystem)

The broader site (incl. this app) has a privacy-respecting, **no-PII** usage
tracker — *independent of the predictor logic*:

- A tiny beacon (`public/track.js`, no cookies) POSTs `{ site, unique }` to a
  Firebase Cloud Function (`/api/track`) on page load.
- The function geolocates the request IP **to a country in memory and discards
  it** (`geoip-lite`), then atomically increments a Firestore document keyed by
  `{ date, site, country }` — storing only aggregate `views` and `uniques`.
- `functions/export-stats.js` exports a date range to CSV/JSON for analysis.

No IP, cookie, or user identifier is ever stored — only counts per
country/site/day.

---

## 10. Build, run & deploy

### Local development
```bash
cd wc2026-app
npm install
npm run dev        # http://localhost:3000/wc2026/   (note the /wc2026/ base path)
npm run lint       # tsc --noEmit (type-check)
npx tsx scripts/verify.ts   # headless logic sanity checks
```

### Regenerate tournament data (only if the Excel/PDF change)
```bash
pip install openpyxl pymupdf
python3 scripts/export_wc_data.py     # rewrites src/data/tournament.json
```

### Production build & deploy
```bash
cd wc2026-app && npm run build        # outputs to ../public/wc2026
cd .. && firebase deploy --only hosting
```
`vite.config.ts` sets `base: '/wc2026/'` and `build.outDir: '../public/wc2026'`,
so no Firebase config change is needed — Hosting serves it at the sub-path.

### Quality gates
- **Type safety:** `npm run lint` (strict TypeScript).
- **Logic tests:** `scripts/verify.ts` checks tiebreakers, full-tournament
  qualification (best-8 thirds, 32-team R32), bracket resolution to a champion,
  and knockout winner derivation (reg / a.e.t. / pens).

---

## 11. Design principles & trade-offs

- **Static data, pure logic.** Precomputing the tournament into JSON keeps the
  runtime backendless, instant, and trivially cacheable; the cost is a one-off
  Python data step when the source schedule changes.
- **Single source of truth.** Deriving everything from one state object eliminates
  sync bugs and makes save/share/reset a one-liner each.
- **Framework-agnostic core.** All rules live in `src/lib` as plain functions, so
  they're independently unit-testable and portable.
- **Privacy by default.** No accounts, no PII; predictions stay on-device unless
  the user shares a link, and analytics store only country-level aggregates.
