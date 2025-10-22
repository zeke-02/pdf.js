### PDFFindController: Using the built-in search engine in your own app

This guide shows how to integrate the viewer’s search engine (`PDFFindController`) into a custom application, without using the full default viewer UI. You’ll learn how to wire the controller, provide inputs via an event bus, and consume the output to highlight matches and navigate.

---

## What is PDFFindController?

`PDFFindController` is the component that:

- Parses user search queries, including options like case sensitivity, entire-word matching, and diacritic handling.
- Extracts page text from a `PDFDocumentProxy` and computes matches per page.
- Navigates to and highlights matches.
- Emits UI update events (state and counts) and page-level update events to re-render highlights.

Source: `web/pdf_find_controller.js`

Key helpers it expects you to provide:

- `EventBus` (from `web/event_utils.js`) to send it search commands and listen for updates
- `IPDFLinkService` implementation to report current page, total pages, and allow navigation by setting `page`

---

## Prerequisites

- A loaded PDF document (`PDFDocumentProxy`), typically obtained from `pdfjsLib.getDocument(...).promise`.
- A minimal link service implementing the properties the find controller uses:
  - `pagesCount: number`
  - `page: number` (getter/setter). Setting `page` should navigate your viewer to that page.
- An `EventBus` instance to dispatch/find events and handle emitted updates.
- (Recommended) A text layer/highlight mechanism that can highlight match ranges on each page.

---

## Install/imports

When consuming inside this repository or a build that ships the viewer code:

```js
import { PDFFindController } from "../web/pdf_find_controller.js";
import { EventBus } from "../web/event_utils.js";
```

If you are using `pdfjs-dist` only, note that `PDFFindController` is part of the viewer layer, not the core rendering API. You need to include the viewer files (or copy the controller and its direct deps) in your build.

---

## Minimal wiring

```js
// 1) Create an EventBus
const eventBus = new EventBus();

// 2) Provide a minimal link service
class LinkService {
  constructor(getPagesCount, setCurrentPage, getCurrentPage) {
    this._getPagesCount = getPagesCount;
    this._setCurrentPage = setCurrentPage;
    this._getCurrentPage = getCurrentPage;
  }
  get pagesCount() { return this._getPagesCount(); }
  get page() { return this._getCurrentPage(); }
  set page(val) { this._setCurrentPage(val); }
}

// Example viewer state
let currentPage = 1;
const linkService = new LinkService(
  () => viewer.pagesCount,              // implement in your viewer
  (p) => { currentPage = p; viewer.scrollToPage(p); },
  () => currentPage,
);

// 3) Create PDFFindController
const findController = new PDFFindController({
  linkService,
  eventBus,
  // Optional tuning: only emit counts when the whole doc has been scanned
  // updateMatchesCountOnProgress: false,
});

// 4) Provide the document (once loaded)
// pdfDocument is the PDFDocumentProxy from pdfjsLib.getDocument(...).promise
findController.setDocument(pdfDocument);

// 5) Hook UI inputs to dispatch find commands
function dispatchFind(state) {
  eventBus.dispatch("find", { source: this, ...state });
}

// Example: initial search, next/prev, toggle highlight-all
dispatchFind({
  type: undefined,            // undefined = initial/debounced search
  query: "invoice number",   // string or string[]
  caseSensitive: false,
  entireWord: false,
  highlightAll: true,
  findPrevious: false,        // true for Shift+Enter / "Previous"
  matchDiacritics: false,
});

// Next result
dispatchFind({ type: "again", query: "invoice number", findPrevious: false });

// Previous result
dispatchFind({ type: "again", query: "invoice number", findPrevious: true });

// Toggle highlight-all without changing query
dispatchFind({ type: "highlightallchange", query: findController.state?.query, highlightAll: true });
```

Notes:

- When `type` is omitted (or falsy), the controller debounces the search by ~250ms to avoid searching while the user is still typing.
- Use `type: "again"` for next/previous navigation.
- Use `type: "highlightallchange"` when toggling only `highlightAll`.

---

## Events you should listen to

`PDFFindController` emits these events on the same `EventBus`:

- `updatetextlayermatches`:
  - `{ source, pageIndex }` where `pageIndex` is either a page index (0-based) or `-1` meaning “update all visible/active pages”.
  - Your text-layer/highlight code should respond by re-rendering highlights for the indicated page(s). Use the controller’s `pageMatches[pageIndex]` and `pageMatchesLength[pageIndex]` arrays to know where to place highlights within the page text items.

- `updatefindcontrolstate`:
  - `{ source, state, previous, entireWord, matchesCount, rawQuery }`
  - `state` is one of: `FOUND`, `NOT_FOUND`, `WRAPPED`, `PENDING`.
  - `matchesCount` has `{ current, total }` for “result X of Y” UI.
  - Use this to update your search bar indicators and button enablement.

- `updatefindmatchescount`:
  - `{ source, matchesCount }` with `{ current, total }` as above.
  - Emitted as pages are processed (or once at the end if `updateMatchesCountOnProgress` is false).

---

## Highlighting matches in a text layer

The controller computes, per page:

- `pageMatches[pageIndex]`: an array of match start positions (character offsets in the original, non-normalized text layer string)
- `pageMatchesLength[pageIndex]`: corresponding lengths for each match

How you consume these depends on how you build your text layer. A common approach is:

1. Build a page-wide text string by concatenating `textContent.items[i].str` and inserting newlines where `textItem.hasEOL` is true (this is exactly what the controller does internally).
2. Map character ranges back to DOM ranges within your text layer spans, then wrap those ranges with a highlight element.
3. When `updatetextlayermatches` fires, recompute highlights for that page.

If you maintain a `TextLayer` abstraction, handle:

```js
eventBus._on("updatetextlayermatches", ({ pageIndex }) => {
  if (pageIndex === -1) {
    // Update all active pages
    visiblePages.forEach(idx => textLayers[idx].renderMatches(findController));
  } else {
    textLayers[pageIndex].renderMatches(findController);
  }
});
```

Inside your `renderMatches` routine, use:

```js
const starts = findController.pageMatches[pageIndex] || [];
const lengths = findController.pageMatchesLength[pageIndex] || [];
// Translate (start, length) pairs into DOM ranges and wrap them.
```

To ensure the current selection is scrolled into view, the controller will set `linkService.page` as needed and also exposes:

```js
findController.scrollMatchIntoView({
  element,           // The DOM element containing the selected match
  selectedLeft: 0,   // Horizontal offset if you have overflows
  pageIndex,
  matchIndex,        // Match index within the page, used to ensure correctness
});
```

Your text layer should call `scrollMatchIntoView` after (re)rendering the selection for the active page and match indices (available via `findController.selected`).

---

## The search state you dispatch

When dispatching `find`, provide a state object with these fields:

- `type?: "again" | "highlightallchange"` (omit/undefined for initial/debounced search)
- `query: string | string[]` The search term(s). If `string[]`, the controller matches any term (OR) and prefers longer matches first.
- `caseSensitive?: boolean`
- `entireWord?: boolean` Uses a Unicode-aware heuristic to avoid partial-word matches.
- `highlightAll?: boolean` If true, compute/highlight all matches, otherwise only the current match.
- `findPrevious?: boolean` If true with `type: "again"`, navigate backward.
- `matchDiacritics?: boolean` If true, diacritics must match exactly; otherwise the controller normalizes diacritics for robust matching.

Advanced options driven by constructor and props:

- `updateMatchesCountOnProgress?: boolean` (constructor option) If false, emit counts only after all pages are processed.
- `onIsPageVisible?: (pageNumber: number) => boolean` If set, helps the controller decide whether a `findagain` should reset the search when the previously selected match is no longer visible.

---

## Programmatic navigation

To move to next/previous result without changing options:

```js
const { query } = findController.state || {};
eventBus.dispatch("find", { type: "again", query, findPrevious: false }); // Next
eventBus.dispatch("find", { type: "again", query, findPrevious: true });  // Previous
```

---

## Public getters you can read

- `findController.highlightMatches: boolean` Whether the controller wants highlights visible
- `findController.pageMatches: number[][]` Array of match-start arrays per page
- `findController.pageMatchesLength: number[][]` Array of match-length arrays per page
- `findController.selected: { pageIdx: number, matchIdx: number }` The current selection
- `findController.state: object | null` The last search state you dispatched

These are primarily useful for building your text layer/highlight views and selection styling.

---

## Performance notes

- Text extraction is done lazily, one page at a time, and cached. Searching across many pages may take time on large documents; the controller streams progress and counts if `updateMatchesCountOnProgress` is true (default).
- Initial searches are debounced by ~250ms; repeated navigation is not debounced.
- If the findbar is “closed” in your UI, you can mimic the viewer behavior by dispatching a `findbarclose` event. The controller will stop pending work, clear highlights, and reset UI state.

```js
eventBus.dispatch("findbarclose", { source: this });
```

---

## End-to-end example (wiring a simple search bar)

HTML:

```html
<form id="search">
  <input id="q" placeholder="Search" />
  <label><input id="case" type="checkbox" /> Case</label>
  <label><input id="word" type="checkbox" /> Whole word</label>
  <label><input id="hl" type="checkbox" checked /> Highlight all</label>
  <button type="button" id="prev">Prev</button>
  <button type="button" id="next">Next</button>
  <span id="count"></span>
  <span id="status"></span>
  <button type="button" id="close">Close</button>
  </form>
```

JS:

```js
const bus = new EventBus();
const find = new PDFFindController({ linkService, eventBus: bus });
find.setDocument(pdfDocument);

const $ = (id) => document.getElementById(id);

function currentState(overrides = {}) {
  return {
    query: $("q").value,
    caseSensitive: $("case").checked,
    entireWord: $("word").checked,
    highlightAll: $("hl").checked,
    findPrevious: false,
    matchDiacritics: false,
    ...overrides,
  };
}

// Debounced initial search as user types
$("q").addEventListener("input", () => bus.dispatch("find", currentState({ type: undefined })));
$("case").addEventListener("change", () => bus.dispatch("find", currentState({ type: undefined })));
$("word").addEventListener("change", () => bus.dispatch("find", currentState({ type: undefined })));
$("hl").addEventListener("change", () => bus.dispatch("find", currentState({ type: "highlightallchange" })));

$("next").addEventListener("click", () => {
  const { query } = find.state || {};
  bus.dispatch("find", { type: "again", query, findPrevious: false });
});
$("prev").addEventListener("click", () => {
  const { query } = find.state || {};
  bus.dispatch("find", { type: "again", query, findPrevious: true });
});
$("close").addEventListener("click", () => bus.dispatch("findbarclose", {}));

// Update status and counts
bus._on("updatefindcontrolstate", ({ state, matchesCount }) => {
  const status = ["FOUND", "NOT_FOUND", "WRAPPED", "PENDING"][state];
  $("status").textContent = status;
  const { current, total } = matchesCount;
  $("count").textContent = total ? `${current}/${total}` : "";
});
bus._on("updatefindmatchescount", ({ matchesCount: { current, total } }) => {
  $("count").textContent = total ? `${current}/${total}` : "";
});

// Re-render highlights for affected pages
bus._on("updatetextlayermatches", ({ pageIndex }) => {
  if (pageIndex === -1) {
    viewer.visiblePageIndices().forEach((i) => textLayers[i].renderMatches(find));
  } else {
    textLayers[pageIndex].renderMatches(find);
  }
});
```

Your `renderMatches(find)` should look up `find.pageMatches[idx]` and `find.pageMatchesLength[idx]` and wrap the corresponding character ranges within your page’s text layer DOM.

---

## Troubleshooting

- Nothing highlights: ensure you are listening for `updatetextlayermatches` and re-rendering highlights for the indicated page(s). Also verify that your `PDFDocumentProxy` was passed to `setDocument` before dispatching any `find` events.
- Navigation jumps but no selection: ensure you’re calling `renderMatches` for the selected page and calling `scrollMatchIntoView` with the selected match element. Check `find.selected` for `{ pageIdx, matchIdx }`.
- Counts stuck at zero: if you set `updateMatchesCountOnProgress: false`, counts are emitted only after all pages finish scanning. Otherwise confirm you wired `updatefindmatchescount` and `updatefindcontrolstate`.
- Whole word misses matches: the entire-word heuristic is Unicode-aware and excludes combining diacritics; confirm your text layer uses the same string segmentation as the controller (NFD with its normalization rules).

---

## Reference

- Constructor:
  - `new PDFFindController({ linkService, eventBus, updateMatchesCountOnProgress = true })`
- Methods:
  - `setDocument(pdfDocument: PDFDocumentProxy): void`
  - `scrollMatchIntoView({ element, selectedLeft = 0, pageIndex, matchIndex }): void`
  - `match(query, pageContent, pageIndex): { index: number, length: number }[] | undefined` (exposed mainly for advanced integrations/testing)
- Getters:
  - `highlightMatches: boolean`
  - `pageMatches: number[][]`
  - `pageMatchesLength: number[][]`
  - `selected: { pageIdx: number, matchIdx: number }`
  - `state: object | null`

Event names dispatched by the controller:

- `updatetextlayermatches`
- `updatefindcontrolstate`
- `updatefindmatchescount`

Event names the controller listens to:

- `find`
- `findbarclose`

This guide mirrors the behavior of `web/pdf_find_controller.js` so you can embed robust search in any custom viewer shell.


