# LocalVault

LocalVault is a local-first, single-page file encryption app. Files are read in
4 MiB chunks, encrypted in a Web Worker by a Rust WebAssembly module, and never
sent to a server. The project has no JavaScript package manager, runtime
framework, cookies, analytics, remote fonts, or network API.

## Build and run

Requirements:

- Rust 1.85 or newer
- the `wasm32-unknown-unknown` Rust target
- Python 3 (only for the example local server)

```sh
rustup target add wasm32-unknown-unknown
make build
make serve
```

Open <http://localhost:8080>. Do not open `index.html` through a `file://` URL:
browsers require an HTTP origin to fetch WebAssembly and start the module
worker. `make check` formats, lints, tests, and compiles the WASM target.

For a literal one-file app, open `localvault.html` directly. It embeds the CSS,
JavaScript worker, and compiled WASM and does not need a server or companion
files. `make build` regenerates it and also copies it to
`dist/localvault.html`.

An HTTP-header-ready multi-file artifact is also generated in `dist/`. When using
that deployment shape, deploy its five files atomically so the HTML,
worker, and WASM ABI stay in sync.

With Node.js 18 or newer installed, the same integration smoke tests used in CI
can be run with `node scripts/wasm-smoke.mjs` and
`node scripts/worker-smoke.mjs`. The standalone artifact can be checked with
`node scripts/standalone-smoke.mjs`. These exercise the raw WASM ABI, a real
multi-chunk round trip, worker transfer/detachment behavior, metadata recovery,
version-skew errors, and wrong-passphrase rejection.

## Security design

New files use:

- Argon2id v1.3 with 64 MiB of memory, three iterations, and one lane;
- XChaCha20-Poly1305 authenticated encryption;
- an independent 128-bit random salt and 128-bit random nonce prefix from the
  browser's `crypto.getRandomValues` implementation;
- sequential 4 MiB chunks with a unique 64-bit nonce counter;
- an authenticated, encrypted metadata header containing the original filename,
  MIME type, and modification time; and
- exact container-length, KDF-bound, header, ordering, and final-state checks.

The complete encrypted header, chunk index, chunk count, and plaintext chunk
length are authenticated with every chunk. A zero-byte file has one authenticated
empty chunk. Authentication failures intentionally do not distinguish a wrong
passphrase from corruption or tampering.

Keys and password byte buffers owned by Rust are zeroized. Worker password
buffers are overwritten before release, and cancelling terminates the entire
worker. JavaScript strings and browser-managed copies cannot be guaranteed to be
erased; the UI does not claim otherwise.

### Threat model and limitations

LocalVault protects a file at rest when the encrypted output and its passphrase
are kept separately. It does not protect against a compromised device, browser,
extension, malicious hosting origin, screen/key logger, or a passphrase an
attacker can guess. There is no account, escrow, reset link, or recovery key. A
lost passphrase means a lost file.

The header hides names and MIME types, but the encrypted file's total size leaks
the approximate plaintext size. Files are not compressed because compression can
amplify resource usage and create surprising information leaks.

Chromium-family browsers can use the File System Access API to stream partial
output into an abortable temporary write and commit it only after authentication
succeeds. Other browsers accumulate output chunks into a Blob and are capped at
256 MiB to avoid unstable memory pressure. Decryption never offers a partial Blob
for download.

## LocalVault container v1

All integers are little-endian. The public fixed header is 80 bytes:

| Offset | Size | Field |
| ---: | ---: | --- |
| 0 | 8 | binary magic `LVLT CR LF 1A LF` |
| 8 | 1 | format version (`1`) |
| 9 | 1 | cipher/KDF suite (`1`) |
| 10 | 2 | flags (must be zero) |
| 12 | 4 | total encrypted-header length |
| 16 | 4 | Argon2 memory in KiB |
| 20 | 4 | Argon2 iterations |
| 24 | 1 | Argon2 lanes |
| 25 | 3 | reserved (must be zero) |
| 28 | 4 | plaintext chunk size |
| 32 | 8 | total plaintext size |
| 40 | 16 | Argon2 salt |
| 56 | 16 | XChaCha nonce prefix |
| 72 | 4 | encrypted metadata length, including tag |
| 76 | 4 | reserved (must be zero) |

The encrypted metadata immediately follows the fixed header. Its nonce is the
16-byte prefix followed by `LE64(u64::MAX)` and its AAD is the fixed header.
Metadata plaintext is `filename_len:u16`, `mime_len:u16`, `modified_ms:u64`, then
UTF-8 filename and MIME bytes.

Each encrypted data chunk follows without a length prefix; its size is derived
from the authenticated plaintext and chunk sizes. Chunk `i` uses the nonce prefix
followed by `LE64(i)`. Its AAD is the domain string `LocalVault chunk v1`, the
complete encrypted header, `i:u64`, total chunk count `u64`, and plaintext chunk
length `u32`.

Decoder limits are deliberately stricter than integer ranges: 64 KiB–64 MiB
chunks, 8–65,536 KiB KDF memory, 1–3 iterations, 1–4 lanes, and at most 196,608
KiB-iterations of combined KDF work. The production encoder always writes the
fixed parameters documented above.

## Production deployment

For a hosted production deployment, prefer the multi-file artifact, serve it
only over HTTPS, and set `localvault.wasm` to
`Content-Type: application/wasm`. The HTML includes a restrictive CSP fallback;
prefer equivalent HTTP response headers in production, together with:

```text
Content-Security-Policy: default-src 'self'; script-src 'self' 'wasm-unsafe-eval'; style-src 'self'; img-src 'self' data:; connect-src 'self'; worker-src 'self'; object-src 'none'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'
Cross-Origin-Opener-Policy: same-origin
Referrer-Policy: no-referrer
Permissions-Policy: camera=(), microphone=(), geolocation=(), payment=(), usb=()
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
```

Use `Cache-Control: no-cache` unless the asset names are fingerprinted, and keep
the build's files in one release unit. Review `SECURITY.md` before changing the
format, KDF bounds, worker isolation, or cryptographic dependencies.

## License

MIT. See `LICENSE`.
