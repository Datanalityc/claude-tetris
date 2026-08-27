# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page Tetris implementation in vanilla JavaScript (no dependencies, no build step, no package.json). Three files: `index.html` (DOM/canvas structure), `style.css` (dark/retro styling), `game.js` (all game logic, ~300 lines).

## Running the game

No install or build step. Either open `index.html` directly in a browser, or serve it locally (needed if you hit canvas/module CORS restrictions):

```bash
python3 -m http.server 8000
# or
npx serve .
```

There are no tests or linter in this repo.

## GitHub Actions

- **`.github/workflows/claude.yml`** — responds when someone mentions `@claude` in an issue or PR comment/review.
- **`.github/workflows/claude-code-review.yml`** — reviews every PR automatically.
- **`.github/workflows/claude-issue-triage.yml`** — on issue `opened`/`edited` (and manual `workflow_dispatch` with an `issue_number` input), Claude classifies the issue with the label taxonomy (`bug`/`enhancement`/... + `area:` / `priority:` / `complexity:` / `needs-info` / `ready-for-implementation`, created idempotently by the workflow) and posts a Spanish technical diagnosis (root cause in concrete files/functions, acceptance criteria, step-by-step fix plan) as an idempotent comment marked `<!-- claude-triage-v1 -->`. That diagnosis is the input for actually implementing the fix later (e.g. by mentioning `@claude` in a comment). Re-run manually from the Actions tab via **Run workflow**.

## Architecture

Everything lives in `game.js` as top-level functions operating on module-level `let` state (`board`, `current`, `next`, `score`, `lines`, `level`, `paused`, `gameOver`, `dropInterval`, etc.) — there are no classes or modules.

- **Board model**: `board` is a `ROWS × COLS` matrix; each cell is `0` (empty) or a color index `1–8` identifying which piece locked there.
- **Pieces**: defined in `PIECES` as square matrices of color indices. Rotation (`rotateCW`) is a transpose + row-reverse, not stored per-piece rotation states. Index `8` (`NUT_TYPE`) is the "tuerca": a 3×3 ring with a hollow center cell whose `0` is intentionally passed through by `collide`, so the nut can land over a filled block and seal an unfillable gap — that's the challenge. Its hole is drawn as a stroked circle by `drawNutHole` for the active/ghost/next piece only (locked nuts just show the real gap in the wall).
- **Collision** (`collide`): checks board bounds and overlap with locked cells; used for movement, rotation, and ghost-piece projection.
- **Wall kicks** (`tryRotate`): after rotating, tries horizontal offsets `[0, -1, 1, -2, 2]` before giving up on the rotation.
- **Game loop** (`loop`): driven by `requestAnimationFrame`, accumulates elapsed time in `dropAccum` and drops the piece one row once `dropInterval` is exceeded.
- **Locking/clearing**: `lockPiece` → `merge` (bake piece into `board`) → `clearLines` (bottom-up full-row removal, unshifts empty rows at top) → `spawn` (promote `next` to `current`, generate new `next`; if the new piece immediately collides, `endGame` fires).
- **Scoring**: `LINE_SCORES = [0, 100, 300, 500, 800]` multiplied by `level`; hard drop adds 2 pts/row traveled, soft drop adds 1 pt/row. Level increments every 10 lines; `dropInterval = max(100, 1000 - (level-1)*90)`.
- **Rendering** (`draw`/`drawNext`/`drawBlock`): plain Canvas 2D, redrawn in full every frame — grid, locked board, ghost piece (`globalAlpha = 0.2`), current piece, and the next-piece preview canvas.
- **Input**: a single `keydown` listener switches on `e.code` (arrows, `Space` for hard drop, `X`/`ArrowUp` for rotate, `P` for pause).

When changing board dimensions or block size (`COLS`, `ROWS`, `BLOCK` constants), also update the `<canvas id="board">` `width`/`height` attributes in `index.html` to match (`COLS×BLOCK` and `ROWS×BLOCK`).

The README (in Spanish) documents controls, tunable constants, and the full game flow in more detail if needed.
