# tui — Terminal UI toolkit for Wyn

A small, pretty terminal-UI library, pure Wyn over the native `Terminal` and
`Color` builtins. The headline feature is **display-width-aware layout**: boxes
and menus stay aligned even when their content is colored or contains multibyte
UTF-8, because widths are measured in *visible columns*, not bytes.

```bash
wyn run examples/box_demo.wyn
wyn run examples/menu_demo.wyn
wyn run examples/menu_interactive.wyn   # live arrow-key menu
wyn test tests/
```

## Why `display_width`

`len(s)` counts **bytes**. `"\e[32mhi\e[0m"` is 11 bytes but occupies **2**
columns; `"café"` is 5 bytes but **4** columns. Laying out a border with `len()`
would leave colored or accented rows ragged. `tui::display_width` skips ANSI
`\x1b[…m` escape sequences and counts UTF-8 codepoints:

```wyn
import tui
tui::display_width(Color::green("hi"))   // => 2   (not 11)
tui::display_width("café")               // => 4   (not 5)
tui::display_width("│─┌")                // => 3   (not 9)
```

Every other helper (`pad_to`, `pad_left`, `truncate`, the box and menu
renderers) is built on it, so colored content never breaks a border:

```
┌─ Status ───────────────────────────┐
│ Plain ASCII line                   │
│ Green success line                 │   <- green, still aligned
│ Red bold error                     │   <- red + bold, still aligned
│ Accented: café résumé naïve        │   <- multibyte, still aligned
│ Cyan — the widest line in this box │
└────────────────────────────────────┘
```

## API (Stage 1)

All functions are free functions in the `tui` module (see the note on the
compiler below for why there are no exported structs/enums yet).

### Width & text
- `display_width(s) -> int` — visible column count (skips ANSI, counts codepoints).
- `pad_to(s, width) -> string` — right-pad to `width` visible columns.
- `pad_left(s, width) -> string` — left-pad (right-align) to `width` columns.
- `truncate(s, width) -> string` — cut to `width` visible columns, closing any
  open color with a reset.
- `repeat_str(c, n) -> string`.

### Boxes
- `render_box(title, body) -> string` — a titled, bordered box. `body` is the
  content lines **joined by `"\n"`**.
- `content_width(title, body) -> int`, `box_top(title, w)`, `box_bottom(w)`,
  `box_row(content, w)` — the pieces, if you want to compose your own.
- Glyphs: `glyph_tl/tr/bl/br/h/v()`.

### Menu widget
- `menu_render(title, items, selected) -> string` — a framed menu; `items` is a
  `"\n"`-joined string, `selected` is highlighted in green with a `›` marker.

  ```
  ┌─ Wyn ─────────┐
  │ › Dashboard   │   <- selected row (green)
  │   Examples    │
  │   Packages    │
  │   Concurrency │
  │   Quit        │
  └───────────────┘
  ```
- `menu_row(item, is_selected) -> string`, `menu_up(selected, count) -> int`,
  `menu_down(selected, count) -> int` (both wrap around).

### Screen & keyboard (for interactive apps)
- `screen_begin()` / `screen_end()` — raw-mode enter / restore.
- `screen_render(frame)` — clear, home the cursor, write a full frame.
- `classify_key(code) -> int` — fold a raw `Terminal::read_key()` code into a
  stable category; compare against `key_up/down/left/right/enter/esc/char/other()`.
- `key_name(kind) -> string`, `is_quit(code) -> bool`.

For the design doc's **event-loop via `match` over an `Event` enum** pattern,
see `examples/menu_interactive.wyn` (a live arrow-key menu).

## Examples

| File | What it shows |
|------|----------------|
| `examples/box_demo.wyn` | Titled boxes with colored / multibyte / truncated content, all aligned. |
| `examples/menu_demo.wyn` | The `Menu` widget rendered at several selection positions (non-interactive). |
| `examples/menu_interactive.wyn` | A live `↑/↓`/`j`/`k` + Enter menu driven by `Terminal::read_key()`, dispatched with `match` over an `Event` enum. |

## What's implemented / what's next

**Stage 1 (this release):** `display_width`, width-aware text helpers, titled
box drawing, the `Menu` widget, screen wrappers, keyboard classification, and an
interactive menu loop. 30 unit tests (`wyn test tests/`).

**Next (per `internal-docs/TUI_DESIGN.md`):**
- S2 — more widgets: Text/Paragraph, Input/TextField, ProgressBar, StatusBar,
  and a `Rect` + `split_h/split_v` layout splitter.
- S3 — a double-buffered diff renderer (flicker-free) + Tabs / SplitPane / ScrollView.
- S4 — the `wyn tui` dashboard assembling it all.

## Note on the current compiler (v1.17.0)

The reusable API is intentionally **free functions over primitive types**. The
compiler currently miscompiles a few cross-module constructs, which shaped the
API:

- **`struct` / payload `enum` defined in one module, used from another** emit
  duplicate C typedefs — so `Menu`/`Key` are functions + an int category rather
  than exported types. They work fine *within* a file, which is why
  `menu_interactive.wyn` is self-contained and uses the `Event` enum directly.
- **Array-typed *parameters* don't survive a module boundary** (they arrive
  typed as `long long`). String params work perfectly, so the multi-line APIs
  take a `"\n"`-joined string instead of `[string]`.
- Two smaller checker/RC quirks are worked around inside `src/tui.wyn` (a `==`
  used directly as a bool argument, and `return f(local)` releasing a locally
  built string too early — both fixed by binding to a `var` first; see the
  comments there).

When these are fixed, a `Menu` struct and `Key` enum can wrap these functions
directly with no change to behavior.
