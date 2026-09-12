# chacha20-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

RFC 8439 — the ChaCha20 stream cipher, the Poly1305 one-time
authenticator, and the AEAD that combines them — plus
XChaCha20-Poly1305 with its 192-bit nonce.  Every value the cipher
carries is a `@value` struct of machine words, and everything that
produces bytes writes them into a buffer the caller already owns.

It is for the program that has no AES instructions to lean on: an nRF52
peripheral, a hardware wallet, a RISC-V core in a sensor, and equally a
server that would rather have one cipher whose timing does not depend
on a table.  ChaCha20 is addition, XOR and rotation by constant
amounts; there is no S-box to put in a cache and no data-dependent
index anywhere in the round function.

```
novo pkg add chacha20-nv
novo pkg build
novo test
```

## The one example that will work

```novo
use std.bytes
use cc20aead

fn protect(key: Bytes, nonce: Bytes, aad: Bytes, plaintext: Bytes) -> Result<Bytes, Cc20Error>
    var out = bytes.cursor_le(bytes.zeros(cc20aead.seal_len(bytes.len(plaintext))))
    let n = cc20aead.seal(key, nonce, aad, plaintext, out)!
    Ok(bytes.slice(out.finish(), 0, n))
```

`seal_len` prices the call, the caller owns the buffer, and the tag is
the last sixteen bytes of what comes back.  Nothing in this package
allocates a result.

## The load-bearing interface: a fallible call answers a COUNT, and the state moves by a pure function

`Cc20State` is a `@value` struct — sixteen words in the caller's frame,
no cell, no header, no reference count — and SPEC § 14.5 keeps an
unboxed struct out of a `Result` payload.  So `apply_into` cannot hand
back the advanced state, and there is no version of this package in
which it could.

What replaces it is better than what it could not have:

```novo
let n  = cc20aead.apply_into(st, chunk, dst)!   // bytes written
let st = cc20aead.advance(st, n)                // where the keystream is now
```

The two facts a streaming cipher has to keep straight — how much went
out, and where the keystream is — are two values on two lines rather
than one tuple that has to be destructured correctly.  And because
`advance` is separate, a caller may move the state **without producing
output**, which is exactly what seeking into the middle of an encrypted
file needs and what a fused call could not express:
`with_counter(st, k)` is block `k` of the keystream and nothing before
it has to be computed.

The same constraint shapes the constructors.  `cc20bytes.state_from`
cannot refuse a 16-byte key, so it does not pretend to: the width check
is its own function — `check_key`, `check_nonce`, `cc20x.check_xnonce`,
each answering `?Cc20Error` — called once, before anything is built.
Every function a program actually calls (`seal`, `open`, `xseal`,
`xopen`) checks for itself, so the only way to reach an unchecked
constructor is to call it deliberately.

## The two halves, and which one a device gets

| module | what it is | a device can use it |
| --- | --- | --- |
| `chacha20` | the state, the rounds, the block function, HChaCha20 | **yes** |
| `poly1305` | the authenticator, the AEAD framing steps | **yes** |
| `cc20bytes` | `Bytes` in, words out, and back | no |
| `cc20aead` | the stream cipher and the AEAD over buffers | no |
| `cc20x` | XChaCha20 over buffers | no |
| `cc20err` | `Cc20Error` | — |

The first two take no `Bytes`, no `Str` and no list at all: a state is
`[Int; 16]` inside a `@value`, a Poly1305 block is `[Int; 4]` inside
one, and every function between them is word algebra.  Sixty-four bytes
of stack for the cipher, a hundred and twelve for the authenticator,
and the arena untouched.  `tests/embedded_probe.nv` builds them for a
Cortex-M4 and the `core-embedded` shard row runs it, so the claim is
**built rather than asserted**.

The other three speak `Bytes` and `Cursor`, which are heap objects.  A
device that wants the whole AEAD drives it out of the first two:
`poly_key_for` is a keystream block, and RFC 8439 § 2.8's framing is
`absorb_pad` and `absorb_lengths`, both public for exactly that reason.

The split is crypto-nv's — `sha256_core` beside `hashing` — under this
package's names, and it is why the device claim is honest rather than
the whole package waving at a tier.

## What is constant time, and what is not

A crypto package that does not say is a crypto package a reviewer
cannot use.

**Constant time.**  `block`, `quarter_round`, `double_round` and
`hchacha20` are constant time *by construction*: addition, XOR and
rotation by constant amounts, no branch on a key bit, no table, no
data-dependent index.  `absorb`, `absorb_last` and the Poly1305
arithmetic are multiply-add-carry with no branch on message or key
content.  `tag_matches` is crypto-nv's `digest.ct_eq` and
`poly1305.chunk_eq` is its device-tier counterpart — four word
comparisons folded with OR, never short-circuited, because an `&&`
chain returns early on the first differing word and tells an attacker
how many they have guessed.

**Not constant time, and not secret.**  `state_word`, `block_word`,
`block_byte`, `chunk_word` and `chunk_byte` index an inline array with
an index the *caller* chose.  Nothing in this package passes a secret
as an index.  Message *lengths* are not secret either — `absorb_last`
takes one, `seal_len` returns one, and every construction here reveals
the plaintext length by construction.

**A requirement on the implementation, not on the signature.**  The
final reduction in `tag` does a conditional subtraction of 2^130 - 5,
and it has to be done with a mask rather than an `if`.  A signature
cannot carry that, so it is written in `poly1305`'s module header where
the lane that writes the body will read it.

**What this package does not do.**  It does not zero a buffer after
use.  `Bytes` is the caller's and the caller decides its lifetime; a
`@value` state is in a stack frame the compiler owns and there is no
portable way to promise the frame is scrubbed.  A program that needs
that needs it at a layer that can guarantee it.

## The nonce is never this package's to choose

A repeated nonce under one key destroys ChaCha20-Poly1305 completely —
not degrades it.  Two ciphertexts under the same nonce XOR to the XOR
of their plaintexts, and because the Poly1305 key is derived from the
key *and the nonce*, two tags under a repeated nonce leak the
authentication key itself.  After that anything can be forged.

There is no counter hidden in this package that could stop that, and no
`[rand]` row that could pick one: a `core` package has neither.  The
two disciplines that work:

- **A counter you keep**, for the 96-bit nonce.  It has to survive
  restarts, replicas and restored backups, which is the operational
  problem that produces nonce reuse in the field.
- **A 192-bit nonce picked at random**, with `cc20x`.  96 bits is too
  few to pick at random — the birthday bound is around 2^48 messages,
  which a busy service reaches — and 192 bits is not: 2^96, which
  nothing reaches.  That is the whole reason XChaCha20 exists, and why
  paseto-nv's `v4.local` and age-nv's stanzas use it.

`CC20P_MAX_MESSAGE_BYTES` is the other ceiling: 2^38 - 64 bytes, 256
GiB less a block, under one key and nonce.  Past it the block counter
wraps and the keystream repeats.  `apply_into` refuses before it writes
rather than wrapping, which is why `advance` cannot fail.

## The layer, and why

`core` — no effects.  The key, the nonce and the counter all arrive as
arguments; nothing here reads entropy, consults a clock or opens a
file.  That is what lets the same code run in firmware and in a request
handler, and it is why the entropy question above is answered by the
caller rather than hidden.

One dependency, `crypto-nv`, for one function: `digest.ct_eq`.  A tag
comparison that returns early is the classic AEAD forgery oracle, and
there should be **one** such loop on the grid for a reviewer to read
rather than one per package.  crypto-nv is `core`, so `dep-layer`
holds.

## The reference implementation

RustCrypto's `chacha20poly1305` and `chacha20` for the surface split
between a raw cipher and an AEAD, and libsodium for the XChaCha20
construction as it is actually deployed.  The oracle is RFC 8439
itself: § 2.1.1's quarter round, § 2.3.2's block, § 2.4.2's encryption,
§ 2.5.2's MAC, § 2.6.2's one-time key and § 2.8.2's AEAD are the test
suite, section by section, so a failing assertion names the paragraph
the implementation disagrees with.  XChaCha20's vectors are
draft-irtf-cfrg-xchacha § 2.2.2's.

**XChaCha20 is a draft, not an RFC**, and has been for years.  It is
stable, unchanged and deployed — libsodium, age and PASETO v4 all ship
it — and the README says so here rather than letting a reader discover
it from a dead link.

Deliberately not ported: ChaCha8 and ChaCha12, because RFC 8439
specifies twenty rounds and a package offering fewer would be offering
a choice a caller has no way to make; the original 64-bit-nonce
ChaCha20 of the 2008 paper, which only a legacy protocol wants;
Salsa20, which is the same author's earlier design and a different
package if anyone ever needs it; and key derivation, which is
hkdf-nv's.

## Status

Every function is `todo()`.  `novo test --isolate` runs the API suite
and every assertion reaches `not implemented: chacha20-nv.<module>.<fn>`.

| module | public types | public items | implemented |
| --- | --- | --- | --- |
| `chacha20` | `Cc20State`, `Cc20Block`, `Cc20Sub` | 6 consts, 15 fns | no |
| `poly1305` | `Cc20Poly`, `Cc20Chunk` | 3 consts, 15 fns | no |
| `cc20bytes` | — | 9 fns | no |
| `cc20aead` | — | 4 consts, 11 fns | no |
| `cc20x` | — | 2 consts, 6 fns | no |
| `cc20err` | `Cc20Error` (+ `impl Error`) | 2 fns | no |

Six public types, 58 public functions, 15 public constants and one
trait impl.

**The consumers are named rows, not hypotheticals.**  cookie-nv named
this package as its missing AEAD; smp-nv's BLE pairing wants one;
age-nv's payload is this construction chunked; paseto-nv's `v4.local`
is `cc20x.xseal`.  All four are `core`, all four pass their own
buffers, and none of them allocates in the cipher.

## Licence

Apache-2.0.
