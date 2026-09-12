# Changelog

All notable changes to chacha20-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-12

### Changed — The README is rewritten in plain technical-writer prose; no signature changed.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `chacha20` — `Cc20State`, `Cc20Block` and `Cc20Sub`, the quarter
  round, the double round, the twenty-round block function and
  HChaCha20, all at `@tier(embedded)` with no `Bytes` in the module.
- `poly1305` — `Cc20Poly` and `Cc20Chunk`, the clamp in the
  constructor, `absorb` and `absorb_last`, the AEAD framing steps
  `absorb_pad` and `absorb_lengths`, and a device-tier constant-time
  tag comparison.
- `cc20bytes` — the bridge: width checks that answer `?Cc20Error`,
  `Bytes` into words, words into a caller's `Cursor`, and the one-shot
  Poly1305.
- `cc20aead` — the raw stream cipher over a caller's buffer, the
  one-time key derivation, `seal` and `open` with their length
  functions, and the § 2.8 tag over data this package did not encrypt.
- `cc20x` — HChaCha20's subkey, the derived state, and
  XChaCha20-Poly1305's `xseal` and `xopen`.
- `cc20err` — one error type, with the authentication failure told
  apart from a caller's own mistake.

### Known

- **The load-bearing shape is that a fallible call answers a COUNT and
  the state moves by a pure function.** A `@value` cannot be a `Result`
  payload, so `apply_into` answers bytes written and `advance` moves the
  counter; the same constraint makes the width checks their own
  functions rather than refusing constructors.
- **The device claim covers `chacha20` and `poly1305` and nothing
  else**, and `tests/embedded_probe.nv` builds them for a Cortex-M4.
  The three modules that speak `Bytes` are the host's half, and the
  README says which is which.
- **What is constant time is written down**, function by function, and
  so is what is not and why that is not secret.
- **The nonce is never this package's to choose.** Two disciplines are
  named; a repeated one is catastrophic, not degrading.
- **One dependency, crypto-nv**, for `digest.ct_eq` alone — one
  constant-time comparison on the grid rather than one per package.
- **XChaCha20 is a draft rather than an RFC**, and the README says so.
- ChaCha8, ChaCha12, the 2008 paper's 64-bit nonce and Salsa20 are
  deliberately outside.
