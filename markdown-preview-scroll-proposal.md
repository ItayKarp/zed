# Markdown preview: scrolling does O(document) work per frame

Zed commit read: `2f808b3` (2026-11-21). Findings are from reading the code, confirmed
with a benchmark (see Measurements).

## What happens on every scroll step

1. **The preview is one big element in a plain scroll container.**
   `crates/markdown_preview/src/markdown_preview_view.rs` (~line 1948) renders
   `div().overflow_y_scroll().track_scroll(..)` with a single child: one `MarkdownElement`
   for the whole document. It is not a virtualized `list()` / `uniform_list()`.

2. **Each wheel/trackpad event re-renders the view.**
   `crates/gpui/src/elements/div.rs:3484-3487`: when the scroll offset changes, the div
   calls `cx.notify(current_view)`, so `MarkdownPreviewView::render` runs again.

3. **Each render rebuilds the entire document.**
   `crates/markdown/src/markdown.rs`, `impl Element for MarkdownElement::request_layout`
   (~line 2675) walks *all* `parsed_markdown.events`, pushes every paragraph, heading,
   list, table, code block and image into a `MarkdownElementBuilder`, then runs Taffy
   layout on the whole tree (~line 3456). `prepaint` and `paint` then visit every
   element, including `paint_mouse_listeners` over all rendered text.

Parsing is already off the main thread and cached, so the per-frame cost is
building the element tree, laying it out and painting it. That grows linearly with
document length, while the visible area stays constant. On a 60 Hz display
(16.6 ms budget) a long README or doc page can miss frames, which reads as
choppy scrolling.

## Proposed fix: render root blocks through `gpui::list`

The parser already splits the document into top-level blocks
(`ParsedMarkdown::root_block_starts`, `MarkdownEvent::RootStart` / `RootEnd(ix)` in
`crates/markdown/src/parser.rs`). That maps directly onto GPUI's `list()` element,
which only builds, lays out and paints the items in (or near) the viewport and caches
the heights of the rest.

Sketch:

- Keep one `ListState` on `MarkdownPreviewView`, with item count = number of root blocks.
- The list's render callback builds a `MarkdownElement` restricted to the event range of
  root block `ix` (a new `MarkdownElement::root_block(ix)` mode, or a range filter on
  the existing builder loop).
- When a background parse finishes, diff old vs new `root_block_starts` and call
  `ListState::splice` only for blocks that changed, so the cached heights of the rest
  survive edits. Call `ListState::reset` when the width or font size changes.
- Replace `ScrollHandle` uses (page up/down, scroll to top/bottom, autoscroll to source
  index, scroll to heading) with `ListState::scroll_to(ListOffset { item_ix, offset })`,
  using `root_block_for_source_index` to find the item.

### Parts that need care

- **Selection, copy and search** assume one `RenderedMarkdown` for the whole document.
  They need to work in source offsets across blocks, which `Markdown` already stores
  (`selection` is a source range), so mostly the hit-testing side changes.
- **Text input handler** (`MarkdownInputHandler`) is registered with the full rendered
  text; it would need to cover only the visible blocks or map through source offsets.
- **Mermaid diagrams and images** change height after loading; those blocks need
  `splice` when their state updates, as the `list` docs require.
- `MarkdownElement` is shared with other features (agent panel, hover popovers), so the
  block-ranged mode should be opt-in and leave the default path unchanged.

## Smaller first step (if a full refactor is too big)

Skip building off-screen root blocks inside `MarkdownElement`: keep the per-block
heights from the previous layout, and for blocks fully outside the scroll viewport
emit an empty spacer of that height instead of the real content. This keeps one
element and one `RenderedMarkdown`, but copy/select across off-screen text would
break, so the list approach is the cleaner long-term fix.

## Measurements

New benchmark `markdown_preview_scroll` in `crates/benchmarks/benches/markdown_renderer.rs`
reproduces the preview: a 900×700 window, the document as one `MarkdownElement` inside an
`overflow_y_scroll` + `track_scroll` container, the preview's `MarkdownOptions`, and each
iteration scrolls 48 px and notifies (what a wheel event does). Documents are the existing
generated mix (headings, paragraphs, lists, tables, Rust code blocks), plus one real
304 KB / 2,256-line product spec (`markdown_preview_scroll_file`, via
`ZED_BENCH_MARKDOWN_FILE`).

Apple M1 MacBook Air (8-core, fanless), release `bench` profile, headless renderer.
"Frame interval" is GPUI's `window present interval` (time between presented frames
while scrolling continuously).

| Document | Frame interval p50 | p90 | p99 | Effective FPS (p50) |
|---|---|---|---|---|
| 5 KB generated | 0.70 ms | 0.89 ms | 1.19 ms | 60 (capped) |
| 10 KB generated | 1.00 ms | 1.28 ms | 1.69 ms | 60 (capped) |
| 50 KB generated | 5.48 ms | 6.61 ms | 9.69 ms | 60 (capped) |
| 250 KB generated | 33.98 ms | 36.50 ms | 70.26 ms | ~29 |
| 304 KB real spec | 68.75 ms | 78.97 ms | 104.33 ms | ~15 |

Per-frame cost grows roughly linearly with document size, even though the visible area
is the same 700 px in every case. The 60 Hz budget (16.7 ms) is exceeded somewhere
between 50 KB and 250 KB. The real spec is about twice as expensive as the generated
250 KB document; it has very long paragraphs (lines up to 2,746 chars), 142 table rows
and a Mermaid diagram.

Run it with:

```sh
cargo bench -p benchmarks --bench markdown_renderer --features gpui_platform/runtime_shaders -- "preview scroll"
ZED_BENCH_MARKDOWN_FILE=/path/to/file.md cargo bench -p benchmarks --bench markdown_renderer --features gpui_platform/runtime_shaders -- "preview scroll (file)"
```

With a virtualized list, the per-frame cost should stay near the 5 KB row regardless
of document length, since only the blocks in the viewport are built and laid out.

## Prototype: culling off-screen root blocks

Implemented in `crates/markdown/src/markdown.rs` as an opt-in
`MarkdownElement::cull_offscreen_root_blocks()`, enabled only by the Markdown preview:

- Each root block's laid-out height is recorded during prepaint (`MeasuredRootBlock`),
  keyed by a hash of the block's source, so edits elsewhere keep the cache. The cache is
  cleared when the container width, rem size or font family changes.
- On the next layout, root blocks whose known position lies more than one viewport
  above or below the visible area are replaced by an empty spacer of their height.
  Below the first unmeasured block, nothing is skipped, since positions are unknown.
- Root block wrappers become flex columns while culling is on, so children's margins
  stay inside the wrapper and the spacer heights add up exactly. Side effect: margins
  no longer collapse across root blocks (e.g. a heading's top margin now adds to the
  previous paragraph's bottom margin).
- Culling is suspended while a selection exists or an autoscroll / heading scroll is
  pending, because those need the text of every block.

Same benchmark, frame interval with culling on:

| Document | Before p50 | After p50 | After p99 |
|---|---|---|---|
| 5 KB generated | 0.70 ms | 0.76 ms | 2.09 ms |
| 10 KB generated | 1.00 ms | 0.72 ms | 2.44 ms |
| 50 KB generated | 5.48 ms | 0.61 ms | 1.33 ms |
| 250 KB generated | 33.98 ms | 0.95 ms | 1.77 ms |
| 304 KB real spec | 68.75 ms | 1.82 ms | 3.49 ms |

Per-frame cost no longer grows with document length. The first frame after opening (or
after a width/font change) still lays out the whole document once to measure it.

Known gaps of the prototype: search "select all" / copy of text spanning culled
blocks only works because culling pauses during selections; a full virtualized list
would avoid the margin side effect and the one-time full layout.

## Notes

- No existing upstream issue or PR covering this was found (searched "markdown preview
  slow / lag / large file / virtualize / scroll").
