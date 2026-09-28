# Idle Obelisk Miner — Stargazing Tracker: Project Summary

## What this is
A single self-contained HTML file (`stargazing-tracker.html`) — no build step, no external deps — tracking progression for the idle game **Idle Obelisk Miner** (wiki: shminer.miraheze.org). It has 4 tabs: **Stars**, **Telescope Upgrades**, **Ores**, **Statue**.

**IMPORTANT: attach `stargazing-tracker.html` itself to the new chat along with this summary.** This summary is not the app — it's context for continuing to edit the app.

## Tabs

### 1. Stars (21 Stargazing stars)
- Each star: input current level + stockpile, tick shared "cap source" checkboxes to raise its level cap.
- Shows exact currency needed to reach a target level, and to reach next level.
- Cost curves are exponential (`baseCost * mult^level`) but many stars have **override anchors** — exact costs at specific levels (and sometimes a scaling-multiplier change from that level onward) taken verbatim from the wiki's `CostTable` templates (`overrideLevel=`, `overrideScaling=`). Implemented in `STAR_OVERRIDES` + `buildStarCostArray()`.
- `startingLevelOffset=1`: level 1 is free/base — cost arrays start real costs at level 2 (an off-by-one bug here was found and fixed; verified against a real screenshot, Draco Lv14→7.57M).
- **Cap sources** (`SHARED_DEFS`): one unified top panel listing every cap-boosting source (Stargazing tree, Arcanist, Skill-Tree, Archaeology, Fishing, Construct statues, Pets), each with which star(s) it applies to (`appliesTo`). Ticking one updates every applicable star's cap. Includes leveled sources (e.g. "Mining Game Skill", 3 levels, +2 Gemini/+3 Scorpio per level — NOT "Gilded Mining Skill", that name doesn't exist).
- **Global hourly gain**: one input (not per-star) — user only farms one star at a time. Drives:
  - Time to next level (per star)
  - Time to max (per star)
  - **Time to max all stars** — summed (not maxed) across stars, EXCLUDING: stars at level 0 (not yet unlocked), and any star toggled off via a per-star "count toward total" checkbox (`excludeTotal`).
- **Extra stars** (separate panel, positioned under the Stars tab): per-star free-form math expression (e.g. `"25b+350m"`) for an *additional* star need unrelated to that star's own level (e.g. crafting costs elsewhere). Fully decoupled from the star's own `costToLevel()`/`nextLevelCost()`/`hoursToMax()` — it does NOT affect those. It DOES have its own independent time-to-farm (`hoursForExtra()`) and IS added into the aggregate "Time to max all stars" total.
- Toggle to temporarily hide completed (maxed) stars — list reorder happens on `change`, not `input`, to preserve typing focus.

### 2. Telescope Upgrades (14 items)
- Same override-anchor pattern (`TELESCOPE_UPGRADES`, per-level cost/currency + override anchors from wiki `CostTable`).
- Shows current-level cost breakdown AND amount required for next level.
- `VEIN_ORDER` — exact vein ordering the user specified ("Deep Sea Vein" spelling) used to sort all vein displays consistently.
- Has its own "hide completed" toggle.

### 3. Ores (86 ores across 4 worlds)
- Global inputs: Crit Damage, Bomb Damage — both accept free-form math expressions (e.g. `"1541x(383+390.5)"`) via a hand-written tokenizer/evaluator `parseDamageInput()` (supports `+ - * / x` and parentheses).
- Crit × Bomb damage compared against each ore's health (with damage-reduction modifiers like "Damage -X%" folded in) to show which ores can be one-shot and by how much more damage is needed for the rest.
- Known limitation: crit-gated ores ("Ultra crit+ only" etc.) compare against raw ore health since no crit-multiplier data exists; "Bomb Damage -X%" badges are informational only, not folded into the damage math; "Golden floor = X%" badges are purely informational.
- Per-world hide toggle (`hiddenWorlds`).
- No ore icons — user explicitly rejected embedded icon images ("looked horrible, not lined up") for both Ores and Statue tabs; reverted to plain text everywhere.

### 4. Statue (World 3 — currently "Gilded" tier)
- 9 levels, each needing 4 bars (Dynamite/Genevium/Manhattite/Rationium/Vaporium/Palmite/Ransomite/MVPD-1988/Cyrogem) + vein + gems.
- User ticks completed levels; remaining totals shown sorted by floor/vein.
- Data has been swapped at least twice (base tier → Gilded tier → possibly "Platinized statues" per a later screenshot — **verify current STATUE_ROWS content matches the most recent data the user provided**, this was flagged as uncertain). Each swap must increment `STATUE_DATA_VERSION` so stale checkmarks from an old tier don't silently carry over.

## Core utilities
- `parseAbbrev()` / `toAbbrev()` / `fmt()` — number abbreviation parsing/formatting (k/m/b/t/q/qi/sx/sp/oc/no/dc/udc/ddc suffixes).
- `parseDamageInput()` — expression evaluator for the Ores tab.
- `formatDuration(hours)` — human string like "3d 5h" or "2y 130d".
- `window.storage.get/set` for persistence — has intermittently thrown "Internal server error" (platform-side, not fixable in app code); mitigated with retry/backoff plus manual Download/Restore backup buttons.
- `DATA_VERSION` / `STATUE_DATA_VERSION` constants + `migrate()` to detect stale/incompatible saved state rather than silently misapplying it.

## Critical patterns / gotchas to preserve

1. **The "5 places" rule**: any new field added to the persisted `state` object must be added in ALL FIVE of:
   1. the default `state` object literal
   2. `migrate()`'s return object
   3. `load()`'s normalization block
   4. the restore-from-backup handler's normalization block
   5. the `resetBtn` click handler
   Missing one has caused bugs multiple times.

2. **Focus preservation**: never fully re-render a section the user is actively typing in. Use targeted per-card updates (`refreshCard()`), and defer list-reordering/filtering (e.g. "hide completed") to the `change` event, not `input`.

3. **Extra stars must stay fully separate** from a star's own level-cost math — only feeds into the aggregate total, via its own `hoursForExtra()`.

4. **Collapsible panels**: `.panel-body` wrapper div + `.collapse-toggle` button (`data-panel-key`) + `state.collapsedPanels{}` persisted map + `initCollapsiblePanels()`/`applyCollapsedPanels()`. **When wrapping ANY section in a new `.panel-body` div, double-check the closing `</div>` is present** — a missing one previously nested the entire rest of the page (Statue tab + footer) inside a hidden tab view, making it render as an invisible 0×0 box with no console errors. Diagnosed via Playwright DOM parent-chain walking; verify structural changes with `getBoundingClientRect()` checks and/or `el.parentElement` chain assertions, not just visual screenshots.

5. **Tab structure**: exactly 7 direct children expected under `.wrap`: `header`, `.tabs`, `starsView`, `telescopeView`, `oresView`, `statueView`, `footer`.

6. No formal test framework — testing has been done ad hoc via Playwright (Node) in the sandbox. **Test scripts do NOT persist across environment resets** — expect to rewrite quick regression scripts each session if needed (check focus retention, star ceilings, ore row counts, telescope card counts, statue bar row counts).

## Reference data (wiki source, already incorporated)
Full wiki source text for Star Costs, Star Level Caps, and Telescope Upgrade Costs was supplied and incorporated into `STAR_OVERRIDES` / `TELESCOPE_UPGRADES` / `SHARED_DEFS`. If any numbers need re-verification, the raw wiki `CostTable`/`Stat` template text is the ground truth (re-fetch from shminer.miraheze.org/wiki/Stargazing if needed — this summary doesn't reproduce it in full since it's already in the HTML).

## Outstanding / open items
- No specific pending feature request beyond general "keep fixing bugs as found."
- Worth double-checking on resume: confirm the Statue tab's current data (bar names/costs) matches the LAST screenshot the user sent (possible "Platinized statues" swap was mentioned but not confirmed applied).
- User has been sensitive about apparent stalling during long debugging sessions — when a fix takes many steps, it's worth a brief status note rather than long silence.
