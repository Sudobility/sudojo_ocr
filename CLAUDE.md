# CLAUDE.md

> **Git policy — never auto-commit or auto-push.** Leave your work in the working tree. Run
> `git commit`, `git push`, `gh pr create`, or `scripts/push_all.sh` **only when the user explicitly
> asks in that turn** — approval for an earlier change does not carry forward. Stay on the current
> branch; do not create feature branches for code changes.

This file is also shipped in the npm tarball (`"files": ["dist/**/*", "CLAUDE.md"]`).

## Project Overview

`@sudobility/sudojo_ocr` (BUSL-1.1) reads a Sudoku board, and optionally its pencilmarks, from an
image. It does **board detection in-process** (hand-written image processing on raw RGBA pixels) and
**digit recognition over HTTP** via the `paddle_ocr` service. Platform canvas work sits behind a
`CanvasAdapter`; only the Node adapter (`/node`, `@napi-rs/canvas`) exists.

There is no web adapter and no React Native adapter. Every frontend uploads the image to
`sudojo_api`, which runs this library on Node — so this package is backend-only in practice.

**Why the split.** PaddleOCR emits one text block per digit on a board that fills the frame, but it
has no board detector. Cropping first is what makes the block→cell mapping exact arithmetic. On an
uncropped photo the digits land in the wrong rows and surrounding page text arrives as blocks
(measured: 81, 62 and 35 of 81 correct uncropped; 81/81 on all three once cropped).

ESM-only (`"type": "module"`, no CJS build). Node 18+. No model and no language data: recognition is
somebody else's service.

## Commands

**Bun only.** Don't use npm, yarn, or pnpm. Verified with Bun 1.4.2.

| Command | What it does | Verified |
|---|---|---|
| `bun install` | install deps (bun auto-installs peers such as `@sudobility/sudojo_types`) | n/a |
| `bun run typecheck` | `tsc --noEmit` (src only, **excludes `*.test.ts`**) | ✅ passes |
| `bun run lint` / `lint:fix` | `eslint src --ext .ts` (prettier violations are lint errors) | ✅ passes |
| `bun run format` / `format:check` | prettier on `src/**/*.{ts,js,json,md}` | ✅ check passes |
| `bun run test` (alias `test:unit`) | `vitest run`, no config file, default globs (6 files) | ✅ 106 passed, 3 skipped (109), ~0.7 s |
| `bunx vitest run src/algorithms` | pure-pixel unit tests only (4 files, 91 tests) | ✅ 91/91 |
| `bunx vitest run src/test` | e2e on fixtures with a faked paddle response; the 3 live tests are skipped offline | ✅ 2 passed, 3 skipped |
| `PADDLE_OCR_URL=http://ocr.sudobility.com bun run test` | also runs the 3 live board checks, asserting **81/81** per board | ✅ 109/109 |
| `bun run build` | `tsc -p tsconfig.esm.json` → `dist/` (`.js` + `.d.ts` + maps) | ✅ (built to scratch) |
| `bun run clean` | `rimraf dist` | no |
| `bun run dev` | `tsc --watch` on `tsconfig.json`, which is `noEmit`: **watch typecheck only, emits nothing** | no |
| `bun run test:watch` | `vitest` watch | no |
| `bun run test:coverage` | `vitest --coverage`: **`@vitest/coverage-v8` isn't installed**, so this won't work as-is | no |
| `bun run verify` | typecheck, then lint, test, build | all parts verified separately |
| `bun run tests/evaluate-truth.ts` / `validate-pencilmarks.ts` | accuracy corpus scripts; **stale** (pass a Tesseract arg, predate the paddle swap). `validate-pencilmarks.ts` overwrites tracked `tests/*_results.json` | ⚠️ not updated |

**Release (document only, don't run).** `prepublishOnly` runs `clean` then `verify`. CI
(`.github/workflows/ci-cd.yml` reuses `johnqh/workflows` `unified-cicd.yml@main`) publishes to npm
(public) on pushes to `main`/`develop` when `NPM_TOKEN` is set. Family releases go through
`sudojo_app/scripts/push_all.sh`, where this package is **2nd**, right after `sudojo_types`; that
script bumps the `sudojo_types` peer and the patch version (the `chore: bump sudojo_types …` commits).

## Architecture

```
extractSudokuFromImage(adapter, source, paddle, config?, onProgress?)         src/ocr.ts
 ├─ adapter.loadImage → sourceCanvas
 ├─ detectBoardRectangle → squarifyRectangle → crop    (skipped if skipBoardDetection)
 │    Sobel edges → long H/V line runs → best square-ish rectangle
 │    → edge-density fallback → dark-pixel fallback (never returns null in practice)
 ├─ recognizeBoard(adapter, croppedCanvas, paddle)                          src/paddle.ts
 │    toDataURL → strip `data:` prefix → POST ${url}/v1/ocr { image }
 │    mapBlocksToBoard: bucket each block by the cell its bbox centre lands in; >1 block → pencil-
 │      marks; a single [1-9] block ≥ 0.45× cell height → given; a small/merged candidate column
 │      ('147') → pencilmarks (full-width digits '３６'→'36' normalized first)
 ├─ removeConflictingDigits: duplicate in a row/col/box → drop the lower-confidence one
 ├─ computeLegalPencilmarks (pencilmark mode, only if any cell looked pencilmarked)
 └─ OCRResult { board: SolverBoard, confidence, digitCount }
```

`docs/OCR.md` has the pencilmark-mode thresholds and the reasoning behind them.

## Public API

For the full export list see `src/index.ts`. The pipeline (`extractSudokuFromImage`,
`detectAndCropBoard`→data URL, `extractCellImages`→81 data URLs), `DEFAULT_OCR_CONFIG`, the paddle
client (`recognizeBoard`, `mapBlocksToBoard`), the pure-pixel algorithms, and the types are all
exported from the root. The `/node` subpath adds `createNodeAdapter()` (**async**, also the default
export) and `NodeCanvasAdapter`, which accept a Buffer or a file path.

Not exported from the root: `medianFilter`, `dilate`, `removeEdgeSpanningLines`, and the `OCR_*`
constants. Import those from `src/algorithms` or `src/types` inside the repo.

**Result contract.** `OCRResult.board` is a `SolverBoard` from `@sudobility/sudojo_types`:
`board.original` and `board.user` are the same 81-char row-major string (`'0'` = empty);
`board.pencilmark.numbers` is 81 comma-separated entries (`''` or `1-9`); `board.pencilmark.autopencil`
is `true` iff any entry is non-empty. `confidence` is the mean paddle block confidence (0–100,
typically 100) and `digitCount` the number of recognized givens.

**`OCRConfig`** (`Partial`, merged over `DEFAULT_OCR_CONFIG`):

| Option | Default | Notes |
|---|---|---|
| `cellMargin` | `0.154` | **no-op.** Paddle reads the whole board; kept for `extractCellImages` and API compatibility |
| `minConfidence` | `1` | **no-op.** Paddle returns per-block confidence and the caller does not threshold it |
| `preprocess` | `true` | **no-op.** The 243/243 fixture run went through the raw crop; PP-OCRv6 does its own normalization |
| `skipBoardDetection` | `false` | use when the image is already a tight board crop. The one option that still matters |
| `recognizePencilmarks` | `false` | when `true` and any cell looks pencilmarked, emit legal candidates and set `autopencil` |

## Assets & relation to sudojo_ocr_ml

- **No assets.** There is no model file and no language data. Recognition is an HTTP call.
- **`paddle_ocr`** (`~/projects/paddle_ocr`, `ocr.sudobility.com`, loopback :8080) is a generic
  PaddleOCR text service: `POST /v1/ocr { image }` → `{ blocks: [{ text, confidence, bbox }] }`.
  It is **not** Sudoku-aware — this library supplies the board geometry. It is shared with
  `svgr_api`, so changing its contract affects another consumer.
- **`sudojo_ocr_ml`** (`sudoocr.sudobility.com`, loopback :8081) is a separate Python ONNX service
  (whole-board recognizer + CornerNet detector, `POST /v1/ocr`) returning the same
  `SolverBoard`-shaped `{board, confidence, digitCount}`. `sudojo_api` dispatches between the two by
  image source: camera captures come here, library images prefer sudojo_ocr_ml and fall back here.
  Note both services expose `/healthz` and `/v1/ocr` with **incompatible payloads** — same paths,
  different bodies.
- **Known gap:** `detectBoardRectangle` does an axis-aligned crop with **no perspective warp**, so a
  tilted phone photo stays tilted. All measured fixtures are flat scans. Camera accuracy on skewed
  photos is unmeasured; see `docs/superpowers/plans/2026-09-13-paddle-ocr-recognition.md` Task 5.

## Repo Map

```
src/
  index.ts                 root exports (curated subset of algorithms)
  types.ts                 OCRConfig/Result/Progress, CanvasAdapter, PaddleConfig, CellRecognition, OCR_* constants
  paddle.ts                paddle_ocr client + block→cell geometry + given/pencilmark rule (~146 lines)
  ocr.ts                   entire pipeline (~375 lines, including the private pencilmark passes)
  adapters/node.ts         the only CanvasAdapter implementation; adapters/types.ts + adapters/index.ts
                           are type re-exports. There is no web.ts
  algorithms/              pure-pixel fns: imageProcessing (grayscale/blur/Sobel/binarize/Otsu/
                           removeGridLines/isCellEmpty), boardDetection (detectBoardRectangle+fallbacks,
                           squarifyRectangle), cellClassification (unused by ocr.ts), digitParsing
    *.test.ts              unit tests on synthetic pixel data
  test/ocr.test.ts         e2e: 2 offline tests (exact match on a faked paddle response, empty board)
                           + 3 live tests (opt-in via PADDLE_OCR_URL, assert 81/81 per board)
  test/fixtures/           Sudoku-Board-1.jpg, -2.png, -3.jpg, Sudoku-Board-Pencilmarks.png
tests/                     NOT vitest (no .test suffix), bun scripts plus the accuracy corpus
  <img> + <img>_truth.txt + <img>_results.json   for lvc39h9c6ra81, pencilmarks-full, pencil_marks_sample
  evaluate-truth.ts, validate-pencilmarks.ts, diagnose-cells*.ts
docs/                      see docs/README.md
plans/IMPROVEMENTS.md      improvement backlog
```

## Conventions

- Algorithms are pure functions on `ImageDataLike` (RGBA `Uint8ClampedArray`) with no canvas
  access. Canvas I/O belongs in adapters.
- Binarized images use black ink on white. "Dark" means channel 0 `< 128`. Index `= row * 9 + col`.
- Defensive index guards (`arr[i] ?? 0`, `safeGet`) are used throughout. Keep them.
- Imports use `.js` suffixes (ESM). Prettier: single quotes, semicolons, width 80, `es5` trailing commas.
- A new public function goes in `algorithms/index.ts`, and also in `src/index.ts` if public.
- A new adapter needs a `src/adapters/<name>.ts` factory plus a `package.json` `exports` entry.
- Pipeline tuning is empirical. Re-run `tests/evaluate-truth.ts` and `src/test` before and after,
  and record findings in `docs/OCR.md`.

## Gotchas

- **Output pencilmarks aren't the OCR'd marks.** In pencilmark mode, OCR only decides *whether*
  the board has pencilmarks. If any cell does, every empty cell gets the **legal candidates**
  computed from the recognized givens (`computeLegalPencilmarks`). OCR'd sub-cell digits are
  discarded.
- **Recognition needs the network.** `recognizeBoard` throws on a non-2xx (with `.status` set) and
  on its own timeout (`timeoutMs`, default 30000). There is no local fallback any more — a paddle
  outage is a hard failure, which `sudojo_api` surfaces as 503/500 rather than a slow degradation.
- **Cropping is not optional in practice.** With `skipBoardDetection: true` on an image where the
  board does not fill the frame, blocks map to the wrong cells and stray text becomes digits.
- `createNodeAdapter()` is async. `new NodeCanvasAdapter().createCanvas()` throws until `init()`.
- Most of `OCRConfig` is now inert — see the table above. Only `skipBoardDetection` and
  `recognizePencilmarks` change behaviour.
- `parseDigitFromText` maps `'0'`/`O`/`o` → 9 on purpose: OCR "0" in a non-empty cell is usually a 9.
- `OCR_BINARIZE_THRESHOLD` (160) and `OCR_PENCILMARK_MIN_INK_RATIO` are **dead constants**. Real
  binarization is range-relative (`binarize(img, 0.3)`) or Otsu.
- Test files are excluded from `tsc` (`tsconfig.json` `exclude`). `tests/*.ts` is covered by
  neither typecheck, lint, nor prettier.
- Both `eslint.config.js` and `eslint.config.mjs` exist. ESLint loads **`.js`**, so `.mjs` is dead.
- `@sudobility/sudojo_types` is a peer only, not a devDependency. The peer range is `^1.2.80`, but
  `bun.lock` resolves it to `1.2.79` (below the range) — `bun install` pulls 1.2.79 from the
  GitHub Packages registry (`.npmrc`: `@sudobility:registry=https://npm.pkg.github.com`).
- `dist/` is gitignored but a stale local copy may exist. Rebuild before linking into a sibling.

Known issues (not fixed): skewed photos are unmeasured (see the Assets "Known gap" above);
`boardDetection.ts` is documented as returning `null` but the dark-pixel fallback always returns a
rectangle; `OCRProgress.status: 'error'` is never emitted and pencilmark progress stalls ~57%→95%;
`tests/*.ts` and `docs/OCR.md` still describe the retired Tesseract pipeline (the `tests/*.ts`
scripts pass a Tesseract arg to `extractSudokuFromImage`, whose signature is now
`(adapter, source, paddle, config?)`).

## Cross-Repo Contracts

| Repo | Relation |
|---|---|
| `sudojo_types` | **Peer dep** (`^1.2.80`). `OCRResult.board` is its `SolverBoard` |
| `sudojo_api` | **The only consumer.** `/node`. `src/services/ocr-paddle.ts` runs `{skipBoardDetection: false, recognizePencilmarks: true}` and supplies `PADDLE_OCR_URL` |
| `paddle_ocr` | The recognition service this library POSTs to. Shared with `svgr_api` |
| `sudojo_app`, `sudojo_app_rn`, `sudojo_extension`, `sudojo_bot` | no dependency; server-side OCR through sudojo_api |
| `sudojo_ocr_ml` | Sibling service, the preferred backend for library images. `bench/` may still pin an old version of this package |

Changing `OCRResult`, `OCRConfig` defaults, or subpath exports affects `sudojo_api`. The `./web`
subpath export and the `tesseract.js` peer dependency were both removed — a consumer still importing
either will break on upgrade.
