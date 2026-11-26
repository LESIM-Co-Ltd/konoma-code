# konoma-code-core

Syntax highlighting into [`ratatui`](https://crates.io/crates/ratatui) 0.30 lines, as used by
[konoma](https://github.com/LESIM-Co-Ltd/konoma): [syntect](https://crates.io/crates/syntect) with
the pure-Rust `fancy-regex` engine (no oniguruma) and the syntaxes and 32 themes of
[two-face](https://crates.io/crates/two-face).

It returns styled text only: foreground color, bold, italic and underline. It does not set
background colors, draw anything, or add line numbers.

```rust
use std::path::Path;

use konoma_code_core::{expand_tabs, has_named_syntax, highlight};

fn main() {
    let path = Path::new("hello.rs");
    let source = "fn main() {\n\tprintln!(\"hi\");\n}\n";

    // One `ratatui::text::Line` per input line, foreground colors only.
    let lines = highlight(source, path, "Nord");
    let lines = expand_tabs(lines, 4);
    assert_eq!(lines.len(), 3);
    for line in &lines {
        println!("{line}");
    }

    // Decided from the name alone; the file is not read.
    assert!(has_named_syntax(Path::new("Makefile")));
    assert!(!has_named_syntax(Path::new("notes.txt")));
}
```

## Functions

| Function | Purpose |
|---|---|
| `highlight(src, path, theme)` | Highlight text, choosing the grammar from `path` |
| `highlight_lang(src, lang, theme)` | Highlight by a language token such as a code fence's `rust` or `py` |
| `highlight_line_by_ext(line, ext, theme)` | Highlight one line on its own (for diffs); nothing carries over between lines |
| `has_named_syntax(path)` | Whether the name alone selects a grammar |
| `expand_tabs(lines, tab_width)` | Replace tabs with a dim `→` padded to the next tab stop |
| `clamp_tab_width(raw)` | `0` (leave tabs) and `1..=16` are kept; larger values become `4` |
| `warm_file(ext, path)` | Compile a grammar ahead of time from the first 64 KiB of a file |
| `warm_dir(root)` | `warm_file` for the extensions under a folder, most frequent first |
| `is_ext_warm(ext)` | Whether that has happened for an extension |

`highlight` picks the grammar from the extension, then the file name (also without its leading
dot), then a few aliases (`*ignore` files, `jsonc`, `json5`), then the first line, and otherwise
uses plain text.

Theme names ignore case, spaces, `-`, `_`, `.` and parentheses. An empty or unknown name gives
TwoDark.

## Behaviour to know about

- Grammars compile on first use, which can take noticeably long. `warm_file` and `warm_dir` are
  blocking calls; run them on a worker thread.
- `warm_dir` looks at no more than 20,000 files, skips hidden entries and does not follow symlinked
  directories.
- A line that syntect fails or panics on is returned as plain text. Catching panics goes through
  `konoma-base`, which installs a process-wide panic hook (it forwards other panics to the previous
  hook).
- `highlight` and `highlight_lang` have **no input size limit**. Bound large or untrusted input
  yourself, for example by highlighting only the visible lines.

No feature flags. Rust 1.92 or newer; tested in CI on macOS and Linux. MIT licensed.
