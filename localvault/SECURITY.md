# Security policy

## Reporting a vulnerability

Do not open a public issue for a suspected vulnerability. Send a private report
to the security contact configured by the deployment owner. Include the affected
version, a minimal reproduction, impact, and whether you believe encrypted files
or passphrases were exposed. Deployment owners should replace this paragraph with
a monitored security address before launch.

## Supported version

The current `1.x` application and LocalVault container format v1 receive security
fixes. Encrypted format compatibility is append-only: an incompatible change must
use a new version or suite identifier, never silently reinterpret v1 bytes.

## Change requirements

Changes to cryptography or the container parser require:

- native round-trip and adversarial tests;
- a release WASM build under the minimum supported Rust version;
- review of nonce uniqueness, authenticated fields, checked length arithmetic,
  and KDF resource bounds;
- browser verification that cancellation discards partial output; and
- a dependency audit with no ignored advisories.

Never add telemetry, remote logging, password recovery, clipboard copying, or
network submission of file/password material without an explicit product and
security review plus prominent user disclosure.
