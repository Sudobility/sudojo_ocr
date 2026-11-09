# @sudobility/sudojo_ocr

Node.js library for extracting Sudoku puzzles, and optionally pencilmarks, from images.

Board detection, squaring and cropping happen in-process; digit recognition is a single HTTP call to
the `paddle_ocr` service. Every frontend goes through `sudojo_api` rather than importing this
directly. ESM-only.

The split matters: PaddleOCR emits one text block per digit on a board that fills the frame, but it
has no board detector. Cropping first is what makes the geometry mapping exact -- on an uncropped
photo the digits land in the wrong rows and any surrounding page text arrives as blocks.

## Installation

```bash
bun add @sudobility/sudojo_ocr @sudobility/sudojo_types
bun add @napi-rs/canvas   # required by the Node adapter
```

`@sudobility/sudojo_types` is a peer dependency. There is no OCR engine to bundle and no language
data to download.

## Usage

```typescript
import { extractSudokuFromImage } from '@sudobility/sudojo_ocr';
import { createNodeAdapter } from '@sudobility/sudojo_ocr/node';

const adapter = await createNodeAdapter(); // async
const result = await extractSudokuFromImage(
  adapter,
  imageBuffer, // Buffer or file path
  { url: 'http://ocr.sudobility.com', timeoutMs: 30000 },
  { recognizePencilmarks: true } // optional, default false
);
console.log(result.board.original);            // "004002008100400000..." ('0' = empty)
console.log(result.board.pencilmark.numbers);  // 81 comma-separated entries
console.log(result.confidence);                // 100
console.log(result.digitCount);                // 40
```

### Accuracy

Measured against the three committed digit-board fixtures and their ground truth, cropped then
recognized: **243/243 cells correct** (the live `src/test` suite asserts 81/81 on each),
0.5-1.6s per board. The `recognizePencilmarks` flag only decides whether
`autopencil` is set and the legal candidates are computed -- the marks themselves are derived from
the givens by constraint propagation, never read off the image.

`result.board` is a `SolverBoard` from `@sudobility/sudojo_types`.

## API

- `extractSudokuFromImage()` -- Full pipeline: image -> board detection -> crop -> paddle_ocr -> block-to-cell mapping -> board assembly
- `detectAndCropBoard()` -- Detect and crop board, returns data URL
- `extractCellImages()` -- Extract 81 cell images as data URLs
- `recognizeBoard()` / `mapBlocksToBoard()` -- The paddle client and its geometry mapping, usable on their own
- Platform adapter: `createNodeAdapter()`
- Low-level image-processing and board-detection algorithms are also exported (see `src/index.ts`)

## Development

```bash
bun run build        # Build ESM to dist/ (no CJS)
bun run test         # Run Vitest (offline; fakes the paddle response)
PADDLE_OCR_URL=http://ocr.sudobility.com bun run test   # also run the live 81/81 checks
bun run typecheck    # TypeScript check
bun run lint         # ESLint (includes Prettier rules)
bun run verify       # Typecheck + lint + test + build
```

See [CLAUDE.md](CLAUDE.md) for the architecture and gotchas, and [docs/](docs/README.md) for OCR
tuning notes.

## Related Packages

- `sudojo_api` -- Backend `/ocr` route (Node adapter). Dispatches by image source: camera captures come here, library images prefer `sudojo_ocr_ml` and fall back here
- `paddle_ocr` -- The PaddleOCR service this library calls for recognition
- `sudojo_app`, `sudojo_app_rn`, `sudojo_extension`, `sudojo_bot` -- Frontends; they call `sudojo_api`, not this library
- `sudojo_ocr_ml` -- Python ONNX whole-board OCR service; the preferred backend for library images
- `@sudobility/sudojo_types` -- Peer dependency; defines `SolverBoard`, the result's `board` field

## License

BUSL-1.1
