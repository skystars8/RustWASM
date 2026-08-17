# Ecosystem in a Bottle

An offline artificial-life field lab powered by a real Rust/WebAssembly simulation engine. The entire application—including the compiled WASM, interface, styles, and renderer—is embedded in [`index.html`](./index.html).

## Run it

Open `index.html` in a current version of Chrome, Firefox, Safari, or Edge. No server, installation, network connection, or build step is required.

The initial world starts automatically from the seed `coral-dawn`. A seed is deterministic: the same seed and the same sequence of edits produce the same ecosystem.

## What to try

- Select **Inspect**, then click a creature to see its energy, ancestry, ecological role, and inherited traits.
- Paint **Food** or **Habitat** to sustain a population.
- Paint a **Hazard** to create a new selection pressure.
- Use **Release** to introduce a fresh lineage.
- Increase the speed and watch trait distributions change across generations.
- Raise mutation variance for faster, less stable evolution, or change the resource climate to make food scarcer or more abundant.

Creatures inherit size, speed, sensing range, metabolic efficiency, aggression, fertility, color, and several controller weights. Feeding adds energy, movement and sensing consume it, hunters can steal it, and sufficiently healthy creatures reproduce with bounded mutations. The engine caps the population at 650 to keep the single-file app responsive.

## Controls

| Input | Action |
| --- | --- |
| `Space` | Play or pause |
| `.` | Advance exactly one simulation tick |
| `1`–`6` | Select a world tool |
| `[` / `]` | Decrease / increase brush radius |
| `-` / `+` | Change simulation speed |
| `Ctrl/Cmd+Z` | Undo the last world edit |
| `I` | Toggle field notes on smaller screens |

Mouse and touch input are supported. Drag on the tank with an environmental tool to paint; the simulation itself always advances in fixed ticks, independent of rendering speed.

## Saving worlds

**Export snapshot** downloads a `.eco` file containing the complete world: creatures, terrain fields, counters, evolution settings, and random-generator state. **Import snapshot** restores it exactly and leaves the simulation paused. Corrupt, oversized, and unsupported snapshots are rejected without replacing the current world.

The browser may ask where to save the file. Snapshots are the dependable persistence mechanism because browser storage behavior can vary when a page is opened directly from the filesystem.

## Technical notes

- The deterministic simulation, mutation, environment, snapshot codec, and checksumming run in Rust compiled for `wasm32-unknown-unknown`.
- The WebAssembly binary is Base64-embedded and instantiated from memory. It is never fetched, so the page works from a `file://` URL.
- Rendering and accessible controls use dependency-free HTML, CSS, JavaScript, and Canvas 2D.
- No analytics, external fonts, CDNs, cookies, or network requests are used.
- Snapshot parsing validates its magic value, version, dimensions, counts, ranges, and finite numeric values before swapping worlds.

Everything runs locally in the browser tab. Reloading the page returns to the displayed seed unless you first export a snapshot.
