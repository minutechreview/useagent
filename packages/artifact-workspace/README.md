# @useagent/artifact-workspace

The provider-neutral rules for artifacts the user can open, edit and export in
the workspace: what kinds of workpiece exist, what each one can do, how its saved
state is shaped and checked, and how a spreadsheet formula is evaluated.
Everything here is plain TypeScript with no I/O, so the browser, the backend and
other packages all run the same code.

`private: true`. No build step - raw TypeScript consumed via `file:` links.

## What it owns

| File | Contents |
|---|---|
| `src/contracts.ts` | The shared types and limits: content types (`DOCX_CONTENT_TYPE`, `XLSX_CONTENT_TYPE`, `PPTX_CONTENT_TYPE`, `PDF_CONTENT_TYPE`), the action lists (`ARTIFACT_WORKPIECE_ACTIONS`, `ARTIFACT_AUTHORING_ACTIONS`), the saved-state shapes, and size limits such as `MAX_WORKPIECE_STATE_BYTES`. |
| `src/index.ts` | The capability matrix: `artifactCapabilitiesFor`, `artifactActionContractFor`, `artifactWorkpieceExports`, `artifactFidelityFor`, `inferWorkpieceKind`, `contentTypeForName`, `canPreviewInline`, and the authoring profiles. It also re-exports every other file in `src/`. |
| `src/document.ts`, `src/spreadsheet.ts`, `src/presentation.ts` | For one workpiece kind each: `normalizeDocument`, `normalizeWorkbook` and `normalizeDeck` check stored data, the `coerce*` functions upgrade old saved states, and the `empty*` functions build a blank one. |
| `src/formula.ts` | The spreadsheet formula engine (see below). |
| `src/csv.ts`, `src/rich-html.ts` | `parseArtifactCsv` and `serializeArtifactCsv`, and `normalizeArtifactRichHtml`, which keeps only the HTML tags and attributes a document may contain. |

## Workpiece kinds

A workpiece is an artifact the workspace can edit as structured state, not just
show. There are four kinds (`ArtifactWorkpieceKind`):

| Kind | Saved state |
|---|---|
| `document` | `{ document }` (a themed document), or `{ text }` for a plain-text source |
| `spreadsheet` | `{ workbook }`, up to 20 sheets of 10,000 rows by 256 columns |
| `presentation` | `{ deck }`, up to 200 slides |
| `pdf` | `{ pdfText }` |

`inferWorkpieceKind` picks the kind from a file's name, content type and size, and
returns `null` for anything else. Other files (media, HTML, SVG, other binaries) get
a `kind` from `artifactCapabilitiesFor` too, but are only previewed and downloaded.

The four actions are `preview`, `download`, `edit` and `export`. Edit and export are
not granted by kind alone: `artifactActionContractFor` adds them for one artifact
that has a matching workpiece. `ARTIFACT_FIDELITY` says, per kind, what an edit
keeps in the native export and what it drops.

## Spreadsheet formulas

`evaluateWorkbook` computes the display value of every cell. Formulas stay the
source of truth in the workbook. It supports:

- arithmetic: `+ - * /`
- cell references (`A1`), ranges (`A1:B4`) and cross-sheet references (`Sheet2!A1`)
- `SUM`, `AVG` / `AVERAGE`, `MIN`, `MAX`, `COUNT` and `IF`
- comparisons

A bad formula gives an error value instead of throwing: `#CYCLE!` for a cycle,
`#REF!` for a bad reference, `#DIV/0!` for division by zero, plus `#NAME?`,
`#VALUE!` and `#ERR!`. A single range may not expand past 200,000 cells.
`VLOOKUP` is deliberately not supported, so the small function set can be tested
completely.

## Who imports it

`frontend` and `backend` use it directly. Two packages depend on it:
`agent-client` and `artifact-formats` (which renders the DOCX, XLSX, PPTX and PDF files).

## Boundary

The package has no dependencies and no runtime imports, so it works in any
runtime. `test/boundary.test.ts` enforces it: `package.json` has no
`dependencies`, `tsconfig.json` sets `lib` to `["ESNext"]`, and `src/contracts.ts`
and `src/index.ts` import nothing from `node:`, React, Next, `@/` or the
backend. Never widen that test to land a change.

## Run and test

```sh
cd packages/artifact-workspace && bun install --frozen-lockfile && bun run typecheck && bun run test
```
