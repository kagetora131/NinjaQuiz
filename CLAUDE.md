# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`index.html` is a single self-contained ninja quiz app ("忍者クイズ 九字の教え") about kuji-in (九字印), the hand seals associated with ninja lore. No build tooling, no dependencies, no package.json — inline HTML/CSS/JS in one file. The quiz is strictly scoped to kuji-in itself; broader ninja topics (history, schools, ranks, mindset) are out of scope here.

The app opens on a difficulty picker offering two tracks, each a self-contained 10-question set (see `quizSets` in the script):

- **下忍(GENIN)・基礎編** — the chant order, the term 九字印, and what each of the nine seals is used for.
- **中忍(CHUNIN)・応用編** — goes deeper into *why* each seal works the way it does, sourced from `忍者クイズ参考用.txt`.

`クイズ一覧.md` is the index of both difficulty tracks (question count, topic, what the final 難問 covers). **Whenever you add or change a tier, update it too** — it's the canonical summary, not something to reconstruct by reading the script.

`忍者クイズ参考用.txt` holds source notes (mudra-by-mudra explanations, plus background on 摩利支天 since it's directly tied to 隠形印) that quiz content gets drawn from. Treat it as reference material, not something to render directly. It has already been trimmed to kuji-in-relevant content — don't reintroduce broader ninja-lore links if re-deriving questions from it.

This app is also a content source for the parent portfolio site (see `../../CLAUDE.md` at the repo root). Per that project's plan (§5/§9), it gets embedded directly via `<iframe src="/apps/<slug>/">` on the `/works` page specifically *because* it has zero external dependencies. **Keep it that way** — no CDN links, no external fonts/images/scripts. Japanese type relies on system font stacks (`"Shippori Mincho"` / `"Yu Gothic Medium"` with OS fallbacks) rather than embedded webfonts, since CJK webfont files are too large to inline.

## Localization (ja/en)

The app ships bilingual (Japanese default, English via the toggle button top-left). All text lives in `CONTENT.ja` / `CONTENT.en` at the top of the script — `ui` (buttons, labels, templated strings like `scoreLine`/`hardHeading`), `ranks` (the 5 achievement-rank labels), and `sets.genin`/`sets.chunin` (the two question sets, positionally aligned between languages — same array order and `correct` index in both, only the JA/EN text differs). **When editing quiz content, update both languages' `sets` entries in the same positions** rather than just one, or the two languages will silently drift out of sync (same `correct` index but different questions).

Named kuji-in terms in the English text follow a fixed romanization + gloss convention (e.g. `Tokko-in ("Vajra Mudra")`, `Ongyo-in ("Concealment Mudra")`) — reuse the existing romanizations rather than inventing new spellings if a term recurs. No macrons (ASCII romaji only, matching common English usage for these terms). The genre term for 印 is **"mudra"**, not "seal" — this was a deliberate correction (see git history), so don't reintroduce "seal" if re-deriving or adding English content.

Toggling language is designed to be non-destructive at any point in the flow: `refreshAll()` re-renders whatever view is currently active from the new language's data without resetting quiz progress. This works because `diffKey`/`current`/`answered`/`currentOrder`/`selectedDisplayIdx` are all language-agnostic state — `renderQuestion(true)` reuses the existing shuffle order instead of re-shuffling, and `applyAnsweredState()` re-derives the already-answered question's feedback/coloring from `answered[current]` + `selectedDisplayIdx` rather than needing the user to re-answer. If you add new persistent state, keep it language-agnostic too so language switching doesn't need special-casing.

## Running it

No dev server or build step. Open `index.html` directly in a browser.

There are no automated tests, lint config, or build commands in this directory.

## Git remote / deployment

This directory is pushed to its own GitHub repo, **https://github.com/kagetora131/NinjaQuiz**, independent of the `Homepage` monorepo this folder physically lives inside of. Push here after every update to files in this directory.

The repo is connected to Vercel and auto-deploys on every push to `main`, publicly at **https://ninja-quiz-kappa.vercel.app/**. No build step is configured (none needed — static HTML). GitHub Pages was evaluated but intentionally turned off in favor of Vercel, to keep this app on the same hosting platform as the other apps in this project.

## Adding a new difficulty tier

Add a matching key to **both** `CONTENT.ja.sets` and `CONTENT.en.sets` (`rankName`, `pickerTag`, `pickerDesc`, `subtitle`, `hardTopic`, `hardComment.ok`/`.ng`, `questions`), reusing the shape of `genin`/`chunin` — keep the two languages' `questions` arrays in the same order with the same `correct` index. Add a matching `.diff-card` button in `#difficultyView` (with `rank`/`tag`/`desc` child elements) wired to `startDifficulty('<key>')`, and wire it into `renderDifficultyPicker()`. Then add an entry to `クイズ一覧.md`.

## Architecture

Everything lives in one IIFE at the bottom of `index.html`:

- **`CONTENT[lang].sets`** — object keyed by difficulty (`genin`, `chunin`), each holding a `questions` array of 10 question objects (`q`, `choices`, `correct` index into `choices`, `explain`, optional `hard: true` on the last question) plus display metadata (`rankName`, `pickerTag`, `pickerDesc`, `subtitle`, `hardTopic`, `hardComment`). `currentSet()` resolves `CONTENT[lang].sets[diffKey]` fresh on every call — nothing caches a language-specific object, so a language switch just needs `refreshAll()`, not a state rebuild.
- **Difficulty selection** — `startDifficulty(key)` sets the module-level `diffKey`, resets `current`/`answered`/`selectedDisplayIdx`/`currentOrder`, and switches from `#difficultyView` to `#quizView`. `changeDiffBtn` reverses this back to the picker.
- **Answer shuffling** — `renderQuestion(reuseOrder)` calls `shuffledIndices()` (Fisher–Yates) to build `currentOrder`, a mapping from *displayed* button position back to the *original* index in `choices` — unless `reuseOrder` is true and a shuffle already exists (used when re-rendering the same question after a language switch, so the button order doesn't jump around). `selectChoice(displayIdx)` always resolves correctness through `currentOrder[displayIdx] === item.correct`, never through the displayed position directly. If you touch this logic, preserve that indirection — it's what stops the "always click the top option" exploit.
- **State** — module-level `lang` (`'ja'`/`'en'`), `diffKey` (the active difficulty key, or `null` on the picker), `current` (question index), `answered` (array of `true`/`false`/`null`, one per question), `selectedDisplayIdx` (which button the user clicked for the *current* question, needed to redraw its feedback after a language switch), and `currentOrder`. No framework, no reactivity; every state change is followed by an explicit `render*()` call.
- **Four view regions**, toggled via the `hidden` attribute: the difficulty picker (`#difficultyView`), the progress dots row (`#progress`, shuriken icons reused both mid-quiz and as the final score history), the question view (`#quizView`), and the results view (`#resultView`).
- **Scoring/rank** — `rankFor(score, hardOk, total)` in `showResult()` maps total correct + whether the final "難問" question was correct to a rank (見習い → 下忍 → 中忍 → 上忍 → 皆伝, from `CONTENT[lang].ranks`), independent of which difficulty tier was chosen. The last question's result is *always* surfaced separately in `#hardResult` (topic and copy pulled from `currentSet().hardTopic`/`hardComment`), regardless of overall score — that's a deliberate product requirement, not just a detail of the rank calc.

## Design system

CSS custom properties on `:root` define the palette (paper/ink/seal-red/jade/gold tones meant to evoke washi paper + hanko ink stamps) and are consumed throughout rather than hardcoded — reuse the existing tokens (`var(--seal-red)`, `var(--jade)`, etc.) for any new UI rather than introducing new colors. This palette intentionally matches the "裏面" (back-of-business-card / ninja brand) palette defined in the root CLAUDE.md, since this quiz is `/ninja`-side content.

All animation (shuriken spin, falling petals, stamp pop-in, progress-dot pulse) is gated behind `@media (prefers-reduced-motion: reduce)` — keep new animated elements consistent with that pattern.
