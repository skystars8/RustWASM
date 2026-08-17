# Atomic Lockbox

Atomic Lockbox is a self-contained HTML utility that encrypts or decrypts a local file with a password. Its Rust cryptography core is compiled to WebAssembly and embedded directly in [`atomic-lockbox.html`](./atomic-lockbox.html), together with the release-path Rust source.

The app makes no network requests after loading. File bytes and passwords stay inside the browser process.

> [!CAUTION]
> Encryption replaces the selected file. A forgotten password cannot be recovered. Keep a separate backup until you have successfully tested decryption.

## Quick start

Use a current desktop version of Chrome or Edge. Direct file replacement depends on the File System Access API and will not work through an ordinary upload control.

You can try opening `atomic-lockbox.html` directly. If the browser does not consider the local page a secure context, serve the directory through localhost:

```bash
cd /path/to/atomic-lockbox
python3 -m http.server 8000
```

Then open:

```text
http://localhost:8000/atomic-lockbox.html
```

### Encrypt a file

1. Choose the local file.
2. Select **Encrypt**.
3. Enter and confirm a passphrase of at least 12 characters.
4. Select **Encrypt & replace original** and grant read/write access.
5. Keep the page open until it reports **Encryption committed**.

The filename and pathname remain unchanged. The file contents become a PWCRYPT v1 encrypted envelope.

### Decrypt a file

1. Choose a file previously encrypted by Atomic Lockbox.
2. The app should detect PWCRYPT v1 and select **Decrypt** automatically.
3. Enter the original password.
4. Select **Decrypt & replace encrypted file**.
5. Keep the page open until it reports **Decryption committed**.

Wrong passwords, corrupted data, truncated files, and unsupported encrypted files all produce the same authentication error. Nothing is intentionally committed in those cases.

## What “atomic” means here

Atomic Lockbox first tries to open an exclusive staged writer on the same browser file handle using `createWritable({ keepExistingData: false, mode: "exclusive" })`. Browsers that reject the experimental `mode` option fall back to the standard staged writer. The app writes the complete result to browser-managed staging storage and calls `close()` only after processing and authentication succeed.

- Before `close()`, staged changes are not visible at the selected pathname.
- Cancellation or failure before commit calls `abort()`.
- Success is reported only after `close()` resolves.
- A same-origin Web Lock reduces races with another Atomic Lockbox tab.
- File size and modification time are checked again immediately before commit.

This is the strongest same-path replacement workflow exposed to a browser page, but it is still best-effort. The [File System Standard](https://fs.spec.whatwg.org/#api-filesystemfilehandle-createwritable) does not provide the crash-durability guarantees of a native application using explicit temporary files, `fsync`, atomic rename, and directory `fsync`.

It also does not provide secure erasure. Old data may survive in filesystem journals, snapshots, backups, cloud version history, SSD wear-leveling, browser temporary files, or other storage layers. External programs are not excluded by the browser’s writer lock.

## Cryptography

PWCRYPT v1 uses:

- **Password KDF:** Argon2id v1.3
- **KDF settings:** 64 MiB memory, 3 passes, 1 lane, 32-byte output
- **Authenticated cipher:** XChaCha20-Poly1305
- **Salt:** 16 random bytes per encryption
- **Nonce prefix:** 16 random bytes per encryption, followed by a 64-bit record number
- **Chunk size:** 4 MiB
- **Authentication tag:** 16 bytes for the header and for every data chunk

The 96-byte prefix is authenticated before decryption begins or plaintext is staged. Each chunk’s associated data binds it to that complete prefix, its zero-based position, and its expected plaintext length. Reordering, duplication, truncation, trailing bytes, header changes, and ciphertext changes are therefore rejected.

Decryption uses two passes over the encrypted chunks:

1. Authenticate the header and every chunk without staging plaintext.
2. Decrypt again and stream plaintext to the uncommitted writer.

The selected file is committed only after both passes complete successfully.

## PWCRYPT v1 file layout

All integers are unsigned little-endian values.

| Offset | Size | Field |
| ---: | ---: | --- |
| 0 | 8 | Magic: `PWCRYPT\0` |
| 8 | 2 | Version: `1` |
| 10 | 2 | Prefix length: `96` |
| 12 | 2 | Cipher/KDF suite: `1` |
| 14 | 2 | Flags: `0` |
| 16 | 4 | Argon2 memory in KiB |
| 20 | 4 | Argon2 passes |
| 24 | 4 | Argon2 lanes |
| 28 | 4 | Plaintext chunk size |
| 32 | 8 | Exact original byte length |
| 40 | 16 | Argon2 salt |
| 56 | 16 | XChaCha nonce prefix |
| 72 | 24 | Reserved, all zero |
| 96 | 16 | Header authentication tag |
| 112 | variable | Authenticated ciphertext chunks |

For plaintext length `S`, chunk size `C`, and chunk count `N = ceil(S / C)`—or zero chunks when `S` is zero—the encrypted size is:

```text
112 + S + 16 × N
```

The format is custom to Atomic Lockbox and is not compatible with age, GPG, ZIP encryption, or other file-encryption tools.

## Security limits

- Password guessing is fully offline. Argon2id raises its cost but cannot rescue a weak or reused password.
- Filenames, paths, timestamps, and exact original file size are not encrypted.
- JavaScript strings, browser memory, Wasm memory, and operating-system caches cannot be guaranteed to wipe perfectly.
- A compromised page, browser, extension, device, or embedded Wasm module can capture passwords and plaintext.
- The app cannot detect rollback to an older, otherwise valid encrypted copy.
- Processing temporarily requires disk space for both the old file and the complete replacement.
- The primitives are established, but this application and custom file format have not received an independent professional security audit. Prefer a mature native tool for high-risk or long-term archival use.

## Browser compatibility

The app requires all of the following:

- `showOpenFilePicker()` and writable `FileSystemFileHandle` support
- A top-level secure context, normally HTTPS or localhost
- WebAssembly, Web Workers, BigInt, and secure random generation

Web Locks are used when available to prevent two Atomic Lockbox tabs from transforming files concurrently.

Desktop Chrome and Edge are the intended browsers. Firefox and Safari do not currently expose a compatible arbitrary-local-file picker for this same-path workflow. The page feature-detects its requirements and disables processing when they are unavailable.

## Troubleshooting

### “Choose file” is disabled

Open the page in desktop Chrome or Edge through HTTPS or localhost. Embedded frames and ordinary insecure HTTP origins cannot request direct local-file access.

### The file is already in use

Close other Atomic Lockbox tabs and programs that may be writing the file, then select it again.

### The staged write could not finish

Check available disk space, filesystem permissions, cloud-sync status, and whether another program is using the file. The temporary replacement may require approximately as much space as the original.

### The password is correct but decryption fails

Atomic Lockbox intentionally does not distinguish a wrong password from damaged or unsupported ciphertext. Restore a known-good encrypted backup and try again.

### Commit or cleanup could not be confirmed

Do not retry blindly. Re-select the file and inspect whether Atomic Lockbox detects it as encrypted, plain, or damaged.

## Validation performed

The implementation was checked for:

- Rust encryption/decryption round trips
- The production Argon2id settings
- Wrong passwords and wrong keys
- Modified ciphertext, authentication tags, nonces, and associated data
- Empty files and sizes at `1`, `C−1`, `C`, and `C+1`
- Truncation and trailing-data rejection
- Wasm validity, expected exports, and absence of imports
- Byte-for-byte parity between the embedded Wasm and the release build
- Main-thread and Worker JavaScript syntax and initialization
- Cancellation immediately before commit
- Accessible live status, focus, contrast, and disabled states

The operating-system file picker and permission prompt require a real user gesture, so they should receive a final smoke test in the target browser and filesystem environment.

## Project contents

```text
atomic-lockbox.html  Self-contained application, embedded Wasm, and Rust source
README.md            Usage, format, security, and compatibility notes
```
