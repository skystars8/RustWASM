# LocalHash

LocalHash is a private, browser-based utility for generating cryptographic
hashes from text. It uses a hashing engine written in Rust and compiled to
WebAssembly.

## Only `index.html` is needed

The entire runnable application is contained in [`index.html`](./index.html).
The Rust WebAssembly engine, JavaScript, CSS, and interface are all embedded in
that one file.

You can download `index.html` and open it directly in a current browser on
Linux, Windows, or macOS. It does not require installation, a web server, an
internet connection, or any supporting files.

LocalHash does not send the entered text anywhere. Hashing happens entirely
inside the browser on the local device.

## Features

- SHA3-512 as the default algorithm
- SHA3-384, SHA3-256, and SHA3-224
- SHA-512, SHA-384, and SHA-256
- BLAKE3
- Exact UTF-8 input, including spaces and line breaks
- Lowercase hexadecimal output
- Selectable hash results with a **Copy hash** button
- Fully offline operation
- Responsive interface for desktop and mobile browsers

## Using the app

1. Open [`index.html`](./index.html) in a browser.
2. Type or paste text into the input area.
3. Select a hashing algorithm.
4. Copy the generated hexadecimal result.

Even an empty input has a valid hash. Capitalization, spaces, Unicode
characters, and line breaks all affect the result.

## Source code

The complete source code is provided as a ZIP archive in the
[`/source`](./source/) directory.

The archive intentionally excludes `node_modules/` and generated Rust build
artifacts. After extracting it, install the dependencies with `npm ci`; they do
not need to be stored in this repository.

The source contains:

- The Rust hashing engine
- WebAssembly build scripts
- The TypeScript/React interface
- The standalone HTML generator
- Hash-vector and application tests
- Complete compilation instructions in `BUILDING.md`

## Important note

A cryptographic hash is a one-way fingerprint, not encryption. Fast hashes
such as those offered here are useful for integrity checks and comparisons,
but should not be used directly for password storage. Passwords should use a
dedicated password-hashing algorithm such as Argon2id.

