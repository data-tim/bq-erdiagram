# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev      # dev server at http://localhost:5173
npm run build    # outputs static files to dist/
npm run preview  # preview the production build locally
```

No test suite exists — verify changes by running the dev server and exercising the feature manually.

## Architecture

Single-page browser app. No backend. Vite builds it; `@dagrejs/dagre` is the only runtime dependency.

### Data flow

```
SQL text
  → parser.js (parseSchema)        — AST with source spans
  → diagram.js (Diagram.setModel)  — measures tables, preserves positions
  → layout.js (layout)             — dagre-based auto-layout (optional)
  → renderer.js (rasterizeTable)   — draws cached bitmaps per table
  → Diagram._draw                  — viewport-culled canvas render loop
```

`main.js` is the orchestrator: it wires the SQL textarea, toolbar buttons, localStorage persistence, URL-hash share links, and all callbacks.

### Key design constraints

**Source span editing** — `parser.js` blanks comments to equal-length whitespace so every character offset in the returned AST maps 1:1 to the original SQL string. `edit.js` uses these spans to do surgical splices, preserving comments, formatting, and unsupported clauses. A table rename cascades through every `REFERENCES` span in the same pass.

**Bitmap cache** — `renderer.js` renders each table into an offscreen canvas (`rasterizeTable`). `Diagram._draw` blits these cached bitmaps onto the main canvas, doing viewport culling before blitting. The cache is invalidated per-table on edits, and fully on theme change.

**Position persistence** — layout positions and camera state are serialized to `localStorage` under key `dbdiga-layout`, debounced 400ms after any drag/pan/zoom. On reload, `applyLayoutData` restores saved positions onto the freshly-parsed model before `setModel` is called. Only brand-new tables (not in the saved layout) get auto-placed via `placeNewTables`.

**Share links** — the entire project (SQL + positions + camera + dialect + annotations) is gzip-compressed, base64-encoded, and stored in the URL `#s=…` fragment via `share.js`. Nothing is sent to a server.

### Module responsibilities

| File | Responsibility |
|---|---|
| `parser.js` | SQL DDL → `{ tables, relations }` AST with source spans |
| `edit.js` | Applies canvas edits back to SQL text via span splices |
| `diagram.js` | Canvas controller: camera, pan/zoom/drag, inline editor, annotation UI |
| `renderer.js` | Table bitmap rasterization, theme definitions, `measureTable` |
| `layout.js` | Dagre-based auto-layout with hub-aware edge weighting |
| `annotations.js` | Sticky notes and group boxes: data model + helpers |
| `svg-export.js` | Vector SVG export (separate from the canvas renderer) |
| `highlight.js` | SQL syntax highlighting (runs in a `<div>` painted behind the textarea) |
| `share.js` | gzip + base64 encode/decode for URL share links |
| `dialects.js` | Per-dialect default column types and type suggestion lists |
| `examples.js` | Example SQL shown on first visit |
| `main.js` | App entry point — wires everything together |

### localStorage keys

| Key | Content |
|---|---|
| `dbdiga-sql` | Last SQL text |
| `dbdiga-layout` | `{ positions, camera, annotations }` |
| `dbdiga-theme` | `'dark'` or `'light'` |
| `dbdiga-dialect` | Active dialect key |
| `dbdiga-dir` | Layout direction `'LR'` or `'TB'` |
| `dbdiga-spacing` | Layout spacing `'comfortable'` / `'compact'` / `'spacious'` |
| `dbdiga-sql-hidden` | `'1'` if the SQL panel is collapsed |
