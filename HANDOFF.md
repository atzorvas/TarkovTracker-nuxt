# Handoff — pinned-task map work + map panel toggle

Written 2026-08-15. This file is **untracked on purpose** — committing it would pollute the
diff against upstream. Read it, don't `git add` it.

---

## 1. Where you are

|                |                                                                                                                                      |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| Repo           | `C:\Users\atzorvas\Documents\GitHub\TarkovTracker-nuxt`                                                                              |
| Branch         | `local-build` — **this is the real state**, 8 commits ahead of `main`                                                                |
| `origin`       | `github.com/atzorvas/TarkovTracker-nuxt` (the user's fork)                                                                           |
| `upstream`     | `github.com/tarkovtracker-org/TarkovTracker` (never fetched; compare against local `main`, which tracks `origin/main` at `b20f31fb`) |
| Default branch | `main`, **not** `master`                                                                                                             |
| Scope          | Local only. The user chose **no PRs**. Nothing has been pushed.                                                                      |

The user's _other_ checkout, `C:\Users\atzorvas\Documents\GitHub\TarkovTracker`, is the **dead
Firebase project**. Don't work there.

### Running it

Two prerequisites, both non-obvious:

1. **Node 24 is required.** System Node is 22; `pnpm install` succeeds misleadingly and then
   `nuxt prepare` dies with `Named export 'attachScopes' not found … @rollup/pluginutils is a
CommonJS module`. A portable Node 24.19.0 lives at `C:\Users\atzorvas\node24` (system Node
   untouched). Prefix every command:

   ```bash
   export PATH="/c/Users/atzorvas/node24:$PATH"
   ```

   Husky hooks fail with `pnpm: command not found` without this too.

2. **`NITRO_PRESET=node-server`** must be set (already in the gitignored `.env`). Without it the
   dev server binds the port and then never serves: `nuxt.config.ts:451-463` runs
   `assertCloudflarePagesOutput` whenever the preset contains `cloudflare`, and
   `DEFAULT_NITRO_PRESET` is `cloudflare-pages`. In dev there is no `.nuxt/dev/index.html`, so it
   throws an unhandledRejection. This is an upstream dev-mode bug, only worked around here.
   Reproduced on a clean install to rule out an `--ignore-scripts` artifact.

`.env` also has blank Supabase vars (offline mode — progress in localStorage, no auth/sync/teams;
both pinned-task features are pure client-side and work fully offline), and commented-out perf
flags. **Both** `VITE_PERF_DEBUG=true` _and_ `NUXT_PUBLIC_LOG_LEVEL=debug` are needed —
`app/utils/perf.ts` logs via `logger.debug`, which the default `warn` client level suppresses.
Enabling only the first produces zero output and looks like proof of nothing happening.

---

## 2. What is committed

```
7e0a1800  feat(tasks): label the map panel toggle and open it by default
ef76b718  refactor(maps): scope pinned-only filter to map markers
1afd5d97  fix(maps): include pinned state in marker hash so pins repaint
aa93d449  fix(tasks): refresh visible tasks when only-pinned filter toggles
51a046dc  fix(maps): restore original selected label to avoid stale translations
079bcfd3  fix(maps): disambiguate pinned task and selected marker colour labels
7d6da603  feat(maps): color map markers for pinned tasks
b81e72b8  feat(tasks): add only-show-pinned-tasks filter
```

**History ≠ final state.** Several of these cancel out. `b81e72b8` built the pinned filter as a
_task_ filter; `ef76b718` moved it to map-only. `aa93d449` fixed a watch dependency that
`ef76b718` then made unnecessary. `079bcfd3` renamed an i18n label and `51a046dc` reverted it.
Read the diff, not the log.

Net diff vs `main` — 13 files, +567 / −92, roughly half of it tests:

```
app/composables/__tests__/useMapObjectiveMarks.test.ts    | 273 +++
app/composables/useMapObjectiveMarks.ts                   |   5 +
app/features/maps/LeafletMap.vue                          | 109 +--
app/features/maps/__tests__/useLeafletMapControls.test.ts |   1 +
app/features/maps/utils/__tests__/marksHash.test.ts       |  21 +
app/features/maps/utils/marksHash.ts                      |  82 +
app/locales/en.json                                       |   6 +-
app/pages/__tests__/tasks.page.test.ts                    |  29 +-
app/pages/tasks.vue                                       |  27 +-
app/stores/__tests__/usePreferences.test.ts               |  19 +
app/stores/usePreferences.ts                              |   9 +
app/utils/__tests__/theme-colors.test.ts                  |  49 +
app/utils/theme-colors.ts                                 |  29 +-
```

---

## 3. The three features

### 3a. Pinned tasks get their own colour on the map — GitHub issue #740

`PINNED_OBJECTIVE: '#7c3bed'` added to `MAP_MARKER_COLORS` (matches `--color-selection-500`).
Colour precedence in `LeafletMap.vue`, for both point markers and zone polygons:

```
mark.pinned ? PINNED_OBJECTIVE : isSelf ? SELF_OBJECTIVE : TEAM_OBJECTIVE
```

**The framing in the issue is slightly wrong and it matters.** Purple was never a "dead colour
that was never wired up" — the existing `SELECTED` key is live and means _this marker's popup is
pinned open_. It just happens to be labelled "Pinned Objective" in `en.json`, which is where the
confusion came from. So this is a naming collision plus a genuine feature request, not a
regression. That is why a **new** key was added rather than reusing `SELECTED`, and why
`attachHoverPinPopup()` / `setLayerSelected()` were left alone.

**Landmine, already defused:** adding any key to `MAP_MARKER_COLORS` silently breaks
`migrateLegacyMapMarkerColors()` for existing users, because `hasExactLegacyDefaults()` returns
false the moment a stored blob is missing _any_ key it iterates. Fixed by freezing a separate
`LEGACY_MAP_MARKER_COLOR_KEYS` list (the 11 original keys) and iterating that instead. There is a
regression test for it in `app/utils/__tests__/theme-colors.test.ts`. **If you add another marker
colour, do not add it to that legacy list.**

### 3b. "Pinned Only" map filter

Map-scoped, by explicit user decision — the task list is **never** filtered. Control lives in the
map controls row in `LeafletMap.vue` next to PMC Spawns / Map Colors (icon `i-mdi-pin`). State is
`mapOnlyShowPinnedTasks` in `app/stores/usePreferences.ts`, persisted to localStorage via the
`pick` array, **deliberately not synced to Supabase** — that would need a DB column that can't be
added to the live project.

The filter is applied in `useMapObjectiveMarks`, not in a watcher. That's the load-bearing design
choice: filtering there changes `marks.length`, a field the memo hash already covers, so correct
repainting follows _by construction_ rather than by remembering to list a dependency — which is
exactly the failure mode of bug #1 below.

### 3c. Labeled map panel toggle (design "2A")

Imported from the user's Claude Design project
`claude.ai/design/p/0b69f543-6a0d-4baa-98ac-01c70a7427dc`, file `Map Toggle Alternates.dc.html`,
section **2A** only. The bare chevron became a labeled pill: neutral + border + `▾` **SHOW MAP**
when closed, accent-tinted + `▴` **HIDE MAP** when open. Panel now defaults to **expanded**
(`useStorage(…, true)`); an explicit collapse still persists and wins.

Decisions worth not re-litigating:

- **Palette.** The mockup's green `#8fd6b4` has no counterpart in this theme (ramps are
  tan / cyan / forest / gray) and is also the design _document's_ own chrome colour — it's on the
  "5A" and "TURN 5" badges. Every other mockup colour does map to a real app token. So the pill
  uses `primary`/`surface`; the open state resolves to `bg-primary-500/15 … text-primary-300`,
  byte-identical to the map-pin badge beside it.
- **Chevron swaps, doesn't rotate.** `rotate-180` was on the whole `UButton`; with a label that
  would rotate the text too.
- **`aria-label` removed.** With visible text it would be a WCAG label-in-name mismatch. The
  `page.tasks.map.toggle_panel` key is now orphaned — **leave it in `en.json`**, deleting it would
  mark it "extra" in the 11 other locales that still carry it.
- **2A's LOADING state not implemented.** Unreachable: `isLoading` gates the whole page
  (`tasks.vue:5`), so the pill doesn't exist yet, and `LeafletMap.vue:51-55` already owns
  map-level loading as an in-map spinner.
- **Turn-4 build notes ignored on purpose** — the `M` shortcut and the hardcoded `#5aa9e0` 3px
  focus ring belong to a different section than the one the user scoped to.

---

## 4. Two runtime bugs found and fixed (both user-reported)

**Bug 1 — filter needed a map switch to apply.** Root cause: the watch source array in
`tasks.vue` was missing the getter. The task _list_ updated because it renders from the reactive
`filteredTasks` computed, while the map reads the watch-driven `visibleTasks` ref — which is why
it looked map-specific. Fixed in `aa93d449`, then made moot by the map-only refactor.

**Bug 2 — pin/unpin didn't repaint markers.** First hypothesis (in-place array mutation in
`togglePinnedTask`) was **wrong** — that function is fully immutable, both branches reassign.
Real cause: `getMarksHash()` omitted `mark.pinned`, so `createObjectiveMarkers()` early-returned.
Reproduced empirically first: 13 Customs tasks pinned, all 5 markers stayed orange.

The fix folds `pinned` into the hash, keeping the memoization intact rather than bypassing it or
reaching for a deep watcher. `getMarksHash` was extracted from `LeafletMap.vue` into
`app/features/maps/utils/marksHash.ts` because **no harness in the repo can mount that SFC**, so
inside it the function was untestable. It carries a comment stating the rule that caused the bug:
_any field that changes how a mark is drawn — not just its identity — belongs in the hash._

---

## 5. ⚠️ Collision with in-flight upstream work — read before touching `tasks.vue`

`origin/feat/tasks-map-panel-expanded-fullscreen` (DysektAI, commits `e3d7d652` + `ca338257`,
dated the same day as this handoff) contains:

> `feat(tasks): expand task map panel by default with obvious toggle and …`

It is **the same feature as 3c**, plus a fullscreen mode. It adds `show_map` / `hide_map` to
`en.json` with the _identical_ English strings, and also defaults the panel to expanded. Its
design differs: the entire header becomes one clickable button with an inline chevron+label chip
beside the title, rather than a separate pill on the right. It touches `tasks.vue` (+286),
`useLeafletMap.ts`, `LeafletMap.vue`, `en.json` and `tasks.page.test.ts`.

This was discovered _after_ `7e0a1800` was committed. It is not merged into `main`. Implications:

- Commit `7e0a1800` will conflict with that branch on `tasks.vue` and `en.json`.
- The user picked design 2A deliberately from their own design doc, so this is not automatically
  a "drop ours" situation — but it is the user's call, and they should be told before any further
  work goes into the toggle.
- If they want the fullscreen feature, rebasing onto that branch is likely cheaper than porting.

---

## 6. Branch map

| Branch                            | State                                                                                                                                                                       |
| --------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `local-build`                     | **The real state.** All 8 commits. Use this.                                                                                                                                |
| `feat/map-pinned-objective-color` | `7d6da603` only. Still cleanly extractable.                                                                                                                                 |
| `feat/only-show-pinned-filter`    | `b81e72b8` only — the **rejected** task-filter design. Do not PR from it. Has a stale worktree at `../tt-wt-filter`; `git worktree remove ../tt-wt-filter` when convenient. |

`app/pages/tasks.vue` used to have a **zero** net diff vs `main`; commit `7e0a1800` changed that.
Any future extraction of just the pinned-task work needs that commit separated out.

---

## 7. Verification

```bash
export PATH="/c/Users/atzorvas/node24:$PATH"
pnpm vitest run                                    # writes test-report.junit.xml
npx eslint app --max-warnings=0
npx nuxt typecheck
node scripts/lint-i18n.mjs
```

Last full run: **2443 passed / 256 files**. eslint exit 0, typecheck exit 0.

**23 failures in 2 files are pre-existing and unrelated** — `scripts/prod-db.test.mjs` and
`tests/llms-txt.test.ts`, Windows exec-bit and CRLF issues. Independently verified identical on
untouched `main`. Do not try to fix them as part of this work.

`lint-i18n` reports missing/extra keys as **non-fatal**. Adding keys to `en.json` alone is
correct and expected — Crowdin reconciles on the next sync.

### Repo conventions that will bite you

- **Only edit `app/locales/en.json`.** The other 11 locales are Crowdin-managed. Adding new keys
  is safe; **changing an existing key's meaning is not** — Crowdin only backfills _missing_ keys,
  so a rename leaves every other locale holding a translation of the old wording. This was learned
  the hard way: `selected` was renamed "Pinned Objective" → "Selected Marker", which improved
  English and made de/cs/ru/etc. actively wrong. Reverted in `51a046dc`.
- **Commitlint types:** `feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert|wip`.
  `i18n:` is rejected.
- Use `git commit -F <file>`. The shell here is Git Bash — PowerShell here-strings (`@'…'@`)
  produce an empty subject and commitlint rejects it.

### Browser testing

No Chrome installed and the Chrome extension isn't connected. Use `playwright-core` against the
cached Chromium at
`C:\Users\atzorvas\AppData\Local\ms-playwright\chromium-1224\chrome-win64\chrome.exe`.
Scripts from this session are in the session scratchpad under `pw/`.

**Gotcha:** on a fresh profile the map panel used to start collapsed, so controls existed in the
DOM but weren't visible until `[aria-label="Toggle map panel"]` was clicked. As of `7e0a1800` it
starts expanded and that aria-label is gone — select `[data-testid="map-panel-toggle"]` instead.

---

## 8. Open items

- **Nothing is pushed and no PRs exist** — the user chose local-only. Don't push without asking.
- Idle-prefetch of the settings-drawer chunk was offered and **not approved**. Measured and
  downgraded: production opens the drawer in ~80 ms with 1 bundled chunk; local dev takes ~153 ms
  with 10 modules / 214 KB. The gap is Vite's on-demand compile in dev, not our changes
  (`git diff main..local-build` for the drawer files is empty). `updateVisibleTasks` costs 1.00 ms
  and doesn't even run on drawer open. Would save ~60 ms once per page load.
- `git worktree remove ../tt-wt-filter`.
- The collision in §5 needs a decision from the user.
