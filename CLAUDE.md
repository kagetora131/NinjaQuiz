# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

This directory holds a growing set of self-contained ninja quiz apps built around kuji-in (九字印), the hand seals associated with ninja lore. Each quiz is a single HTML file (inline CSS/JS, no build tooling, no dependencies, no package.json):

- `index.html` — "忍者クイズ 九字の教え" (入門編). Basics: the chant order, the term 九字印, and what each of the nine seals is used for.
- `okugi.html` — "忍者クイズ 九字印 奥義編" (応用編). Goes deeper into *why* each seal works the way it does, sourced from `忍者クイズ参考用.txt`.

`クイズ一覧.md` is the index of every quiz in this directory (title, file, question count, topic, what the final 難問 covers). **Whenever you add a new quiz file, add an entry there too** — it's the canonical list, not something to reconstruct by scanning the directory.

`忍者クイズ参考用.txt` holds source notes (mudra-by-mudra explanations, links to background reading on ninja history/schools/ranks/mindset) that quiz content gets drawn from. Treat it as reference material, not something to render directly.

This app is also a content source for the parent portfolio site (see `../../CLAUDE.md` at the repo root). Per that project's plan (§5/§9), these files get embedded directly via `<iframe src="/apps/<slug>/">` on the `/works` page specifically *because* they have zero external dependencies. **Keep it that way** — no CDN links, no external fonts/images/scripts. Japanese type relies on system font stacks (`"Shippori Mincho"` / `"Yu Gothic Medium"` with OS fallbacks) rather than embedded webfonts, since CJK webfont files are too large to inline.

## Running it

No dev server or build step. Open any quiz's `.html` file directly in a browser.

There are no automated tests, lint config, or build commands in this directory.

## Git remote

This directory is pushed to its own GitHub repo, **https://github.com/kagetora131/NinjaQuiz**, independent of the `Homepage` monorepo this folder physically lives inside of. Push here after every update to files in this directory.

## Adding a new quiz

Copy an existing quiz file (`index.html` or `okugi.html`) as a starting point rather than writing one from scratch — reuse the CSS custom properties, the shuffle logic, and the three-view (`progress` / `quizView` / `resultView`) structure described below. Update: the `<title>`/`<h1>`, the `quiz` array, the `hardResult` heading text (references the specific 難問 topic), and the `rankFor`/`showResult` copy if the theme calls for it. Then add an entry to `クイズ一覧.md`.

## Architecture

Everything lives in one IIFE at the bottom of each quiz's `<script>`:

- **`quiz`** — array of 10 question objects (`q`, `choices`, `correct` index into `choices`, `explain`, optional `hard: true` on the last question). This array is the single source of truth for content; editing questions means editing this array only.
- **Answer shuffling** — `renderQuestion()` calls `shuffledIndices()` (Fisher–Yates) on every render to build `currentOrder`, a mapping from *displayed* button position back to the *original* index in `choices`. `selectChoice(displayIdx)` always resolves correctness through `currentOrder[displayIdx] === item.correct`, never through the displayed position directly. If you touch this logic, preserve that indirection — it's what stops the "always click the top option" exploit.
- **State** — two module-level vars: `current` (question index) and `answered` (array of `true`/`false`/`null`, one per question). No framework, no reactivity; every state change is followed by an explicit `render*()` call.
- **Three view regions**, toggled via the `hidden` attribute: the progress dots row (`#progress`, shuriken icons reused both mid-quiz and as the final score history), the question view (`#quizView`), and the results view (`#resultView`).
- **Scoring/rank** — `rankFor(score, hardOk)` in `showResult()` maps total correct + whether the final "難問" question was correct to a rank (見習い → 下忍 → 中忍 → 上忍 → 皆伝). The last question's result is *always* surfaced separately in `#hardResult`, regardless of overall score — that's a deliberate product requirement, not just a detail of the rank calc.

## Design system

CSS custom properties on `:root` define the palette (paper/ink/seal-red/jade/gold tones meant to evoke washi paper + hanko ink stamps) and are consumed throughout rather than hardcoded — reuse the existing tokens (`var(--seal-red)`, `var(--jade)`, etc.) for any new UI rather than introducing new colors. This palette intentionally matches the "裏面" (back-of-business-card / ninja brand) palette defined in the root CLAUDE.md, since this quiz is `/ninja`-side content.

All animation (shuriken spin, falling petals, stamp pop-in, progress-dot pulse) is gated behind `@media (prefers-reduced-motion: reduce)` — keep new animated elements consistent with that pattern.
