# Typo Confusion Matrix — Implementation Guide

Goal: a view showing **which key was actually hit instead of which key** when you mistype —
per lesson, per session, and (optionally) overall.

## Current state of the codebase

`Step` (`packages/keybr-textinput/lib/textinput.ts:19`) holds only the **expected** codePoint plus
`typo: boolean`. The wrong key you actually pressed lives only in the `#garbage` buffer during the
lesson and is then thrown away. `Histogram` (`packages/keybr-textinput/lib/histogram.ts`) aggregates
`hitCount`/`missCount` per expected char. The binary format
(`packages/keybr-result-io/lib/binary.ts`) persists the same. So "which key instead of which key" is
missing everywhere — it must be captured first.

Two things already exist to build on:

- **`Ngram2`** (2D codepoint matrix). `packages/page-practice/lib/practice/state/last-lesson.ts`
  already builds `hits2`/`misses2`; note `misses2` is currently never filled.
- **Heatmap rendering**: `packages/keybr-chart/lib/KeyFrequencyHeatmap.tsx`,
  `packages/keybr-keyboard-ui/lib/HeatmapLayer.tsx`.

## Plan (three steps)

**1. Capture (small).** `TextInput.appendChar` mismatch branch (`textinput.ts`, ~line 178) knows both
`expected` and the pressed `codePoint`. Add `#typos: {expected, actual, timeStamp}[]`, expose
`get typos()`. Record *before* the `stopOnError`/`forgiveErrors` split so all modes are covered. Skip
the normalize-equal and space-skip cases — those are not real typos.

**2. Per-lesson/session view (small, no schema change).** `makeLastLesson`
(`packages/page-practice/lib/practice/state/last-lesson.ts`) already receives the `TextInput`. Fill a
`confusion: Ngram2` (rows = expected, cols = actual) from `textInput.typos`. Render a new
`ConfusionChart` in `keybr-chart`, copying the structure of `KeyFrequencyHeatmap`. Mount it in
`KeyExtendedDetails.tsx` or the practice `Presenter`. Session-wide = accumulate across lessons in the
in-memory practice state.

**3. Overall/historical (medium).** Needs persistence. Two ways:

- Bump `HEADER_VERSION` 2 → 3 (`packages/keybr-result-io/lib/header.ts`), append typo pairs per
  result, reader branches on version. Cost: data files no longer import/export against upstream
  keybr.
- **Recommended:** a separate store — an own confusion-count file next to results (follow the
  `packages/keybr-result-userdata/lib/userdata.ts` pattern) or IndexedDB client-side. `Result` format
  untouched, upstream compat intact, and confusion data is aggregate-only (small: map of
  `expected → actual → count`).

---

# Learn map

## Stack (what to know)

- **TypeScript strict, ESM only** (`"type": "module"`, `.ts`/`.tsx` extensions in imports — note
  `./keyusage.ts` is imported *with* extension. Copy that; ESLint enforces it).
- **React 19** function components, hooks. No Redux — a plain mutable state class plus
  `useRef`/re-render (see `LessonState`).
- **react-intl** (FormatJS) for every user string. New UI text → `useIntl()`/`<FormattedMessage>`,
  ids extracted by `scripts/translate.js`.
- **Less CSS modules** (`*.module.less` + `import * as styles from "./X.module.less"`). Stylelint
  rules (logical props: `inline-size`, not `width`).
- **SVG rendering** for keyboard/charts — no canvas, no chart library. You draw `<rect>`/`<text>`
  yourself. Read `packages/keybr-chart/lib/graph.ts`, `geometry.ts`, `Chart.tsx`.
- **node:test** runner + `@testing-library/react` + `rich-assert`. Not jest.
- **Monorepo**: npm workspaces + **lage** task graph + per-package `tsc` project refs.

## Monorepo mechanics

Each `packages/*` is its own `package.json`, `main: lib/index.ts`, with deps declared as
`"@keybr/x": "*"`. Two rules that will bite you:

1. A new cross-package import → **must add the dep to that package's `package.json`**, else compile
   fails.
2. A new exported symbol → **must add it to that package's `lib/index.ts`**.

Commands:

```
npm run compile      # tsc all packages (lage)
npm run test         # node:test all packages
npm run lint-fix
npm run watch        # webpack dev bundle
npm start            # dev server, root/index.js
```

Single package is faster: `cd packages/keybr-textinput && npm run test`.

## The data path to trace (do this first)

Read in order — this *is* the feature:

1. `packages/keybr-textinput-events/` — raw keyboard events → layout emulation → input events.
2. `packages/keybr-textinput/lib/textinput.ts` — `TextInput.appendChar`. **Where expected vs actual
   are compared.** Your capture point.
3. `packages/keybr-textinput/lib/stats.ts` + `histogram.ts` — steps → `Stats`/`Histogram`.
4. `packages/page-practice/lib/practice/state/lesson-state.ts:96` — `makeStats(textInput.steps)` →
   `Result`.
5. `packages/page-practice/lib/practice/Controller.tsx:76` —
   `makeLastLesson(result, textInput.steps)`. **Change the signature here to pass typos.**
6. `packages/page-practice/lib/practice/state/last-lesson.ts` — builds
   `hits`/`misses`/`hits2`/`misses2`.
7. `packages/page-practice/lib/practice/KeyboardPresenter.tsx:42-52` — renders those into keyboard
   layers. **Your render point.**

Then the persistence side (only if you want historical data):
`packages/keybr-result/lib/result.ts` → `packages/keybr-result-io/lib/binary.ts` + `header.ts` →
`packages/keybr-result-userdata/lib/userdata.ts` → `packages/server/`.

## Concepts specific to this codebase

- **`CodePoint`** is a plain `number` everywhere (`@keybr/unicode`). Chars are never strings.
- **`Ngram2`** (`@keybr/keyboard`) is a square matrix over an alphabet of codepoints,
  `.add(a, b, n)`. Your confusion matrix is exactly this shape (expected × actual). Already used for
  `hits2`/`misses2`.
- **`Histogram`** — two different classes, don't confuse them: the `@keybr/textinput` one
  (per-codepoint hit/miss/time, persisted) vs the `@keybr/math` one (generic keyed counter, used in
  `last-lesson.ts`).
- **`KeySet`/`Letter`** (`@keybr/phonetic-model`) — the lesson alphabet model.
- **Layers pattern**: `<VirtualKeyboard>` wraps `<KeyLayer>`, `<HeatmapLayer>`,
  `<TransitionsLayer>`. For key-to-key arrows, read `TransitionsLayer` in `keybr-keyboard-ui` — it
  already draws expected→next arcs; a confusion view is the same drawing with different data.
- **Fakes for tests**: `ResultFaker`, `FakePhoneticModel`, `FakeIntlProvider`,
  `FakeSettingsContext`. Every UI test wraps in the last two — copy
  `packages/keybr-chart/lib/KeyFrequencyHeatmap.test.tsx` verbatim as your test skeleton.

## Suggested build order

1. **Milestone 1 (day 1):** add the `#typos` array + `get typos()` to `TextInput`, plus tests in
   `textinput.test.ts`. Pure logic, no UI. Watch the edge cases: `stopOnError`, `forgiveErrors`,
   `spaceSkipsWords`, `filterText.normalize` equality — decide per case whether it counts as a typo.
2. **Milestone 2:** thread it through `Controller.tsx` → `makeLastLesson` → a new `confusion: Ngram2`
   field. `console.log` it, confirm real data.
3. **Milestone 3:** render. Cheapest is a new `TransitionsLayer`-style layer on the practice
   keyboard. Nicer is a new `ConfusionMatrix.tsx` in `keybr-chart` (grid heatmap, rows expected, cols
   actual), added to the lesson details area or a new tab in `Presenter.tsx`.
4. **Milestone 4 (optional):** persist. Use a separate store, not the `Result` binary format — that
   keeps upstream import/export working.

## Gotchas

- Strings must be i18n'd or lint fails (`eslint-plugin-formatjs`).
- `packages/keybr-chart/lib/dist/` is generated — ignore it.
- Session state resets on window blur/focus (`Controller.tsx:41-43`) — "session totals" need to
  survive that if you want cross-lesson aggregation.
- This fork already diverges from upstream (Dockerfile, ads). Keep feature commits separate from
  self-host commits so future rebases stay sane.

Start point: open `packages/keybr-textinput/lib/textinput.ts` and `textinput.test.ts` side by side.
