# Rust + WASM File Encryptor


A single-page web app that encrypts and decrypts files entirely in the browser using **Rust compiled to WebAssembly**.

## Features

- **AES-256-GCM** authenticated encryption
- **Argon2** password-based key derivation (memory-hard)
- Salt + unique nonce per encryption
- Zero server contact — everything runs locally
- Drag & drop file support
- Clean modern UI

## Files

```
index.html          ← the app
pkg/
  file_encryptor_wasm.js
  file_encryptor_wasm_bg.wasm
  ...
```

## How to run

Because browsers restrict loading `.wasm` modules from `file://`, serve the folder with any static server:

```bash
# Python
python3 -m http.server 8080

# or Node
npx serve .

# or PHP
php -S localhost:8080
```

Then open http://localhost:8080

## Crypto details

- **KDF**: Argon2 (default parameters from the `argon2` crate)
- **Cipher**: AES-256-GCM
- **Output format**:
  ```
  [16-byte salt] [12-byte nonce] [ciphertext + 16-byte GCM tag]
  ```
- Encrypted files get a `.enc` extension by convention

## Building from source

If you want to rebuild the WASM module:

```bash
# (requires Rust + wasm-pack)
cd file-encryptor-wasm   # if you keep the source
wasm-pack build --target web --release --out-dir ../pkg
```

The Rust source lives in the original crate (AES-GCM + Argon2 + wasm-bindgen).
