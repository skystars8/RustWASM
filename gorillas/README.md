# GORILLAS.BAS — Rust + WebAssembly edition

A faithful browser recreation of the 1991 QBASIC rooftop artillery game. The
gameplay, physics, collision detection, city generation, pixel art, craters,
wind, scoring, and 640×350 RGBA framebuffer are implemented in Rust and
compiled to WebAssembly.

The small JavaScript layer only handles browser controls, accessibility,
PC-speaker-style WebAudio cues, and copying the Rust framebuffer to the canvas.

## Play immediately

Open `index.html` in a modern browser. The compiled WebAssembly is base64
embedded in that one file, so the game works offline and from a `file://` URL.
There are no external fonts, images, scripts, or network requests.

## Original-style flow

- Enter two player names (limited to 10 characters).
- Choose the total number of scoring rounds and gravity; defaults are 3 and 9.8.
- On each turn, enter an angle and then velocity, both from 0 through 360.
- Account for the red wind arrow at the bottom of the 640×350 EGA playfield.
- One gorilla hit ends a scoring round. After the configured number of rounds,
  the highest score wins; a tie is possible.

The recreation retains several original quirks: player-two angle mirroring,
velocity below 2 causing a self-hit, strong occasional wind gusts, bananas
passing through and startling the sun, passable blast craters, and the skyline
generator's unreachable inverted-V branch.

## Build from Rust

The source crate has no third-party dependencies and does not use
`wasm-bindgen` or `wasm-pack`.

```bash
python3 scripts/build-standalone.py
```

That command compiles `rust-gorillas/src/lib.rs` for
`wasm32-unknown-unknown`, embeds the release module into
`web/gorillas.template.html`, and writes identical copies to `index.html` and
`public/gorillas.html`.

Rust checks can be run separately:

```bash
cargo test --manifest-path rust-gorillas/Cargo.toml
cargo clippy --manifest-path rust-gorillas/Cargo.toml --all-targets -- -D warnings
```

The optional Vinext wrapper serves `public/gorillas.html` in a full-screen
iframe and can still be built with the package scripts when Node dependencies
are available.
