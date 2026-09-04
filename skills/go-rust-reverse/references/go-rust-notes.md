# Go/Rust Tips (Go/Rust 提示)

Go: first find `runtime.main` / `main.main`, then recover symbols via the pclntab.
Rust: first collect `src/` path strings and `Option`/`Result` handling blocks.
Both: prefer string-driven analysis; avoid getting lost in the runtime libraries.
