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

## Running it

No dev server or build step. Open `index.html` directly in a browser.

There are no automated tests, lint config, or build commands in this directory.

## Git remote

This directory is pushed to its own GitHub repo, **https://github.com/kagetora131/NinjaQuiz**, independent of the `Homepage` monorepo this folder physically lives inside of. Push here after every update to files in this directory.

## Adding a new difficulty tier

Add a new key to the `quizSets` object (`rankName`, `subtitle`, `hardTopic`, `hardComment.ok`/`.ng`, `questions`), reusing the shape of `genin`/`chunin`. Add a matching `.diff-card` button in `#difficultyView` wired to `startDifficulty('<key>')`. Then add an entry to `クイズ一覧.md`.

## Architecture

Everything lives in one IIFE at the bottom of `index.html`:

- **`quizSets`** — object keyed by difficulty (`genin`, `chunin`), each holding a `questions` array of 10 question objects (`q`, `choices`, `correct` index into `choices`, `explain`, optional `hard: true` on the last question) plus display metadata (`rankName`, `subtitle`, `hardTopic`, `hardComment`). This is the single source of truth for content; editing questions means editing these objects only.
- **Difficulty selection** — `startDifficulty(key)` sets the module-level `difficulty`/`quiz` vars from `quizSets[key]`, resets `current`/`answered`, and switches from `#difficultyView` to `#quizView`. `changeDiffBtn` reverses this back to the picker.
- **Answer shuffling** — `renderQuestion()` calls `shuffledIndices()` (Fisher–Yates) on every render to build `currentOrder`, a mapping from *displayed* button position back to the *original* index in `choices`. `selectChoice(displayIdx)` always resolves correctness through `currentOrder[displayIdx] === item.correct`, never through the displayed position directly. If you touch this logic, preserve that indirection — it's what stops the "always click the top option" exploit.
- **State** — module-level `difficulty` (the active `quizSets` entry), `quiz` (its `questions` array), `current` (question index), and `answered` (array of `true`/`false`/`null`, one per question). No framework, no reactivity; every state change is followed by an explicit `render*()` call.
- **Four view regions**, toggled via the `hidden` attribute: the difficulty picker (`#difficultyView`), the progress dots row (`#progress`, shuriken icons reused both mid-quiz and as the final score history), the question view (`#quizView`), and the results view (`#resultView`).
- **Scoring/rank** — `rankFor(score, hardOk)` in `showResult()` maps total correct + whether the final "難問" question was correct to a rank (見習い → 下忍 → 中忍 → 上忍 → 皆伝), independent of which difficulty tier was chosen. The last question's result is *always* surfaced separately in `#hardResult` (topic and copy pulled from `difficulty.hardTopic`/`hardComment`), regardless of overall score — that's a deliberate product requirement, not just a detail of the rank calc.

## Design system

CSS custom properties on `:root` define the palette (paper/ink/seal-red/jade/gold tones meant to evoke washi paper + hanko ink stamps) and are consumed throughout rather than hardcoded — reuse the existing tokens (`var(--seal-red)`, `var(--jade)`, etc.) for any new UI rather than introducing new colors. This palette intentionally matches the "裏面" (back-of-business-card / ninja brand) palette defined in the root CLAUDE.md, since this quiz is `/ninja`-side content.

All animation (shuriken spin, falling petals, stamp pop-in, progress-dot pulse) is gated behind `@media (prefers-reduced-motion: reduce)` — keep new animated elements consistent with that pattern.
