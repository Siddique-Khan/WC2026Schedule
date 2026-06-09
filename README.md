# ⚽ World Cup 2026 Predictor

An interactive FIFA World Cup 2026 predictor served at **funprojects.ai/wc2026**.

Enter scorelines for all 72 group matches and watch:
- **Group standings** re-sort live using the real FIFA tiebreaker chain
  (points → head-to-head mini-league → goal difference → goals scored → FIFA rank).
- **Qualification** resolve automatically: the 12 group winners, 12 runners-up,
  and the **8 best third-placed teams** (via the official 495-row lookup table).
- An **interactive knockout bracket** (Round of 32 → Final) you advance by
  clicking winners.
- A **path-to-final** tracer for any team, branching on where it finishes its group.

Predictions persist in `localStorage`, can be shared via URL, and kickoff times
display in the visitor's timezone (overridable).

## Tech
React 19 + TypeScript + Vite 6. No backend, no API keys — all logic runs in the
browser from a static data file generated from the source spreadsheet.

## Data
`src/data/tournament.json` is **generated** from `../world_cup_2026_v1.3.xlsx`
by `scripts/export_wc_data.py`. Regenerate only if the spreadsheet changes:

```bash
pip3 install openpyxl
python3 scripts/export_wc_data.py
```

It emits teams + FIFA ranks, group membership, the 72 group fixtures, the full
knockout bracket wiring (with dates), and the best-thirds lookup table — with
built-in validation (48 teams, 12×4 groups, 72 + 32 matches, 495 lookup rows).

## Develop

```bash
npm install
npm run dev        # http://localhost:3000/wc2026/
npm run lint       # tsc --noEmit
npx tsx scripts/verify.ts   # headless logic checks (tiebreakers, qualification, bracket)
```

Note: in dev the app runs under `/wc2026/` (the production base path).

## Build & deploy

`vite.config.ts` sets `base: '/wc2026/'` and writes the build to
`../public/wc2026`, so Firebase Hosting serves it at `funprojects.ai/wc2026`
with no config change.

```bash
npm run build                 # -> ../public/wc2026
cd .. && firebase deploy --only hosting
```

## Layout
```
wc2026-app/
├── scripts/
│   ├── export_wc_data.py   # Excel -> src/data/tournament.json (run once)
│   └── verify.ts           # headless logic sanity checks
├── src/
│   ├── data/tournament.json   # generated tournament data
│   ├── lib/                   # pure, framework-agnostic logic
│   │   ├── standings.ts       # group tables + FIFA tiebreakers
│   │   ├── qualification.ts   # winners/runners-up + best-8 thirds lookup
│   │   ├── bracket.ts         # knockout resolution + winner picks
│   │   ├── pathToFinal.ts     # team route tracer
│   │   ├── persistence.ts     # localStorage + shareable URL
│   │   ├── time.ts            # timezone formatting
│   │   └── flags.ts           # team -> flagcdn code
│   ├── components/            # GroupStage, StandingsTable, KnockoutBracket, PathFinder, Flag
│   ├── App.tsx                # state, tabs, share/reset/timezone
│   └── index.css              # "Stadium Night" football theme
```

By Khan Siddique
