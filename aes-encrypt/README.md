# File Encryptor (Rust + WASM) — Single HTML File

A complete file encryption/decryption tool that runs entirely in the browser.  
Everything is packed into **one HTML file** — just double-click it. No server, no install, no internet required after download.

## Quick Start

1. Download `file-encryptor-single.html`
2. Double-click the file (or open it in any modern browser)
3. Drop a file → enter a password → Encrypt or Decrypt

That’s it.

## Features

- **AES-256-GCM** authenticated encryption
- **Argon2** password-based key derivation (memory-hard)
- Unique salt + nonce on every encryption
- Works 100% offline
- Drag & drop support
- Single self-contained HTML file (~139 KB)
- No data ever leaves your device

## How to use

| Action   | Result                                      |
|----------|---------------------------------------------|
| Encrypt  | Creates `originalname.enc`                  |
| Decrypt  | Restores original (or `.decrypted` if no `.enc` suffix) |

**Important:** Use a strong password. There is no recovery if you forget it.

## Crypto details

| Component          | Details                                      |
|--------------------|----------------------------------------------|
| Key derivation     | Argon2 (default parameters from the `argon2` crate) |
| Cipher             | AES-256-GCM                                  |
| Output format      | `[16-byte salt][12-byte nonce][ciphertext + 16-byte GCM tag]` |
| Implementation     | Pure Rust → compiled to WebAssembly          |

## Browser support

Works in recent versions of:

- Chrome / Chromium / Edge
- Firefox
- Safari

Requires WebAssembly support (available in all modern browsers since ~2017–2018).

## Limitations

- The entire file is loaded into memory. Very large files (hundreds of MB) may cause the tab to slow down or run out of memory.
- Encryption/decryption runs on the main thread, so the UI freezes briefly on big files.
- This is a demonstration-grade tool. For highly sensitive data, prefer well-audited desktop tools (e.g. age, GPG, VeraCrypt).

## Files

```
file-encryptor-single.html   ← the only file you need
README.md                    ← this file
```

(An earlier multi-file version with a separate `pkg/` folder also exists if you prefer loading the WASM externally.)

## Building from source

If you want to rebuild or modify the Rust/WASM part:

```bash
# Requires Rust + wasm-pack
cargo new --lib file-encryptor-wasm
# (add the crypto code + wasm-bindgen bindings)
wasm-pack build --target web --release
# then embed the resulting .wasm as base64 into the HTML
```

## License

Do whatever you want with it.
