# chacha20-nv

ChaCha20 is a stream cipher and Poly1305 is a one-time message authenticator.
Together they form the authenticated encryption scheme specified in
[RFC 8439](https://www.rfc-editor.org/rfc/rfc8439). This package brings both to
novo-lang, along with XChaCha20-Poly1305, the variant with a 192-bit nonce
described in
[draft-irtf-cfrg-xchacha](https://datatracker.ietf.org/doc/draft-irtf-cfrg-xchacha/).

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it is
implemented. Version 0.1.0 will be the first working release.

## What ChaCha20-Poly1305 is

A *stream cipher* turns a key and a nonce into a long pseudorandom byte string,
the *keystream*, and encrypts by XOR-ing the plaintext with it. Decryption is
the same operation. ChaCha20 produces its keystream 64 bytes at a time by
running twenty rounds of addition, XOR and rotation over sixteen 32-bit words.

A *message authentication code* is a short value that proves a message was not
altered. Poly1305 is a *one-time* authenticator: its key must never be reused,
so RFC 8439 derives a fresh one from the cipher's own keystream for each
message. The arithmetic is modulo the prime 2¹³⁰ − 5.

*Authenticated encryption with associated data* (AEAD) combines the two in one
operation. It encrypts the plaintext, authenticates the ciphertext together with
some additional data that travels in the clear, and appends a 16-byte tag.
Decryption checks the tag first and returns nothing if it does not match.

| Value | ChaCha20-Poly1305 | XChaCha20-Poly1305 |
| --- | --- | --- |
| Key | 32 bytes | 32 bytes |
| Nonce | 12 bytes | 24 bytes |
| Tag | 16 bytes | 16 bytes |
| Block | 64 bytes | 64 bytes |
| Rounds | 20 | 20 |
| Longest message under one key and nonce | 274877906880 bytes (2³⁸ − 64) | the same |

ChaCha20 has no substitution table and no data-dependent array index anywhere in
its round function, which is what makes it the cipher to reach for on a
processor with no AES instructions: an nRF52 peripheral, a hardware wallet, a
RISC-V core in a sensor, and equally a server that would rather have one cipher
whose timing does not depend on a cache.

## Install

```
novo pkg add chacha20-nv
```

## Example

```novo
use std.bytes
use cc20aead

fn protect(key: Bytes, nonce: Bytes, aad: Bytes, plaintext: Bytes) -> Result<Bytes, Cc20Error>
    var out = bytes.cursor_le(bytes.zeros(cc20aead.seal_len(bytes.len(plaintext))))
    let n = cc20aead.seal(key, nonce, aad, plaintext, out)!
    Ok(bytes.slice(out.finish(), 0, n))
```

`seal_len` gives the exact size the output needs, the caller owns the buffer,
and the tag is the last sixteen bytes of what comes back. Nothing in this
package allocates a result.

Build and test with:

```
novo pkg build
novo test
```

Today `novo test` fails on purpose: every test reaches a
`not implemented: chacha20-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents | Builds for a microcontroller |
| --- | --- | --- |
| `chacha20` | The cipher at the lowest level: the sixteen-word state, the quarter round, the double round, the block function and HChaCha20. | yes |
| `poly1305` | The authenticator at the lowest level, and the AEAD framing steps of RFC 8439 section 2.8. | yes |
| `cc20bytes` | The boundary between `Bytes` and words: the width checks, the constructors, and the functions that write a block or a chunk into a cursor. | no |
| `cc20aead` | ChaCha20-Poly1305 over buffers: `seal`, `open`, the keystream functions, and the sizing helpers `seal_len` and `open_len`. | no |
| `cc20x` | XChaCha20-Poly1305 over buffers: `xseal`, `xopen`, the subkey step and the 24-byte nonce check. | no |
| `cc20err` | The error type `Cc20Error`, its message, and the two questions `describe` and `is_authentication_failure`. | — |

Six public types, 58 public functions, 15 public constants and one trait
implementation.

## How to choose an entry point

**Most programs call `cc20x.xseal` and `cc20x.xopen`**, the 24-byte-nonce form.
See "The rules a user needs" for why the nonce width matters.

**Call `cc20aead.seal` and `cc20aead.open`** when a protocol specifies the
12-byte nonce of RFC 8439, and you have a counter that cannot repeat.

**Call `cc20aead.apply_into` and `cc20aead.advance`** to encrypt or decrypt in
pieces. `apply_into` answers how many bytes it wrote; `advance` moves the
keystream position by that many bytes and hands back the new state.

```novo
let n  = cc20aead.apply_into(st, chunk, dst)!   // bytes written
let st = cc20aead.advance(st, n)                // where the keystream is now
```

The two facts a streaming cipher has to keep straight — how much went out, and
where the keystream is — are two values on two lines. Because `advance` is a
separate function, a caller can also move the state *without producing output*,
which is what seeking into the middle of an encrypted file needs.
`with_counter(st, k)` starts at block `k` of the keystream with nothing before
it computed.

**Firmware calls `chacha20` and `poly1305` directly.** See "Running on a
microcontroller".

## The rules a user needs

1. **A nonce must never repeat under one key.** A repeat destroys
   ChaCha20-Poly1305 completely rather than weakening it. Two ciphertexts under
   the same nonce XOR to the XOR of their plaintexts, and because the Poly1305
   key is derived from the key *and* the nonce, two tags under a repeated nonce
   leak the authentication key itself. After that anything can be forged.
2. **Do not pick a 12-byte nonce at random.** 96 bits is too few: the chance of
   a collision becomes real at about 2⁴⁸ messages, which a busy service reaches.
   Either keep a counter that survives restarts, replicas and restored backups,
   or use the 24-byte nonce, where the same bound is 2⁹⁶ and nothing reaches it.
   That is the whole reason XChaCha20 exists.
3. **This package cannot choose a nonce for you.** It draws no randomness and
   keeps no counter. The key, the nonce and the block counter all arrive as
   arguments.
4. **At most 274877906880 bytes — 2³⁸ − 64, which is 256 GiB less one block —
   may be encrypted under one key and nonce.** Past that the block counter wraps
   and the keystream repeats. `apply_into` refuses with
   `Cc20MessageTooLong(got, limit)` before it writes anything, which is why
   `advance` cannot fail. A caller with more data than that needs a second
   nonce, which is what age-nv's chunking does.
5. **A failed `open` says nothing but `Cc20TagMismatch`, and writes nothing to
   the caller's buffer.** No partial plaintext is produced and no detail about
   where the mismatch was is reported.
6. **Size the output buffer with `seal_len` or `open_len` first.** A buffer with
   fewer bytes left than the call needs is `Cc20OutputTooSmall(want, have)`.
   `open` on fewer than 16 bytes is `Cc20CiphertextTooShort(got, want)`, since
   there is not even a tag there.
7. **The width checks are separate functions.** `cc20bytes.check_key`,
   `cc20bytes.check_nonce` and `cc20x.check_xnonce` each answer `?Cc20Error`.
   Every function a program ordinarily calls — `seal`, `open`, `xseal`, `xopen`
   — performs them itself, so the only way to reach an unchecked constructor is
   to call it deliberately.

A word on the shape of the streaming interface. `Cc20State` is a plain value
struct: sixteen words in the caller's own stack frame, with no heap cell, no
header and no reference count. The language does not admit such a struct as a
`Result` payload (SPEC section 14.5), so `apply_into` cannot hand back the
advanced state. `advance` is what replaces that, and the same constraint is why
`cc20bytes.state_from` cannot refuse a 16-byte key and so does not pretend to.

## Running on a microcontroller

novo-lang lets a package state which of its modules can run on a device with no
heap allocator, and the registry checks that claim by compiling a probe program
for a Cortex-M4. Here the claim covers `chacha20` and `poly1305`.

Those two modules take no `Bytes`, no `Str` and no list at all. A cipher state
is `[Int; 16]` inside a plain value struct, a Poly1305 block is `[Int; 4]`
inside one, and every function between them is word algebra. Sixty-four bytes of
stack for the cipher, a hundred and twelve for the authenticator, and the heap
untouched. `tests/embedded_probe.nv` builds them for the device on every test
run, so the claim is built rather than asserted.

The other three modules speak `Bytes` and `Cursor`, which are heap objects. A
device that wants the whole AEAD drives it out of the first two: `poly_key_for`
is a keystream block, and RFC 8439 section 2.8's framing is `absorb_pad` and
`absorb_lengths`, both public for exactly that reason.

## Timing behaviour

**Constant time by construction.** `block`, `quarter_round`, `double_round` and
`hchacha20` use addition, XOR and rotation by constant amounts. No branch
depends on a key bit, no table is consulted and no array is indexed by secret
data. `absorb`, `absorb_last` and the Poly1305 arithmetic are multiply, add and
carry, with no branch on message or key content.

**Tag comparison.** `tag_matches` is crypto-nv's `digest.ct_eq`, and
`poly1305.chunk_eq` is its counterpart for the device modules: four word
comparisons folded with OR, never short-circuited. An `&&` chain returns early
on the first differing word and tells an attacker how many they have guessed.

**Indexed by the caller, not by a secret.** `state_word`, `block_word`,
`block_byte`, `chunk_word` and `chunk_byte` index an inline array with an index
the caller chose. Nothing in this package passes a secret as an index.

**Message lengths are not secret.** `absorb_last` takes one, `seal_len` returns
one, and every construction here reveals the plaintext length anyway.

**A requirement on the implementation.** The final reduction in `tag` performs a
conditional subtraction of 2¹³⁰ − 5, and it must be done with a mask rather than
an `if`. A signature cannot carry that requirement, so it is written in
`poly1305`'s module header.

**Buffers are not zeroed after use.** A `Bytes` belongs to the caller, who
decides its lifetime. A plain value state lives in a stack frame the compiler
owns, and there is no portable way to promise that frame is scrubbed. A program
that needs that guarantee needs it from a layer that can give it.

## What is not included

- **ChaCha8 and ChaCha12.** RFC 8439 specifies twenty rounds, and offering fewer
  would offer a choice a caller has no way to make.
- **The original ChaCha20 with a 64-bit nonce**, from the 2008 paper. Only a
  legacy protocol wants it.
- **Salsa20**, the same author's earlier design. A separate package if anyone
  needs it.
- **Key derivation.** That is [hkdf-nv](https://novo-lang.org/packages/hkdf-nv).
- **A random number generator or a nonce counter.** See rule 3.
- **Buffer zeroing.** See "Timing behaviour".

## Related packages

- [crypto-nv](https://novo-lang.org/packages/crypto-nv) is the only dependency,
  and supplies one function: `digest.ct_eq`, the constant-time byte comparison
  that `cc20aead.open` uses to check a tag. A tag comparison that returns early
  is the classic forgery oracle, and there should be one such loop on the
  registry for a reviewer to read rather than one per package. Nothing else is
  taken from crypto-nv; this package hashes nothing.
- Four packages are written against this one: cookie-nv for its AEAD, smp-nv for
  Bluetooth pairing, age-nv whose payload is this construction in chunks, and
  paseto-nv whose `v4.local` token is `cc20x.xseal`. All four pass their own
  buffers, and none of them allocates in the cipher.

## Test vectors

RFC 8439 is the oracle, section by section, so a failing assertion names the
paragraph the implementation disagrees with: section 2.1.1's quarter round,
section 2.3.2's block, section 2.4.2's encryption, section 2.5.2's MAC,
section 2.6.2's one-time key and section 2.8.2's AEAD. XChaCha20's vectors come
from draft-irtf-cfrg-xchacha section 2.2.2.

RustCrypto's `chacha20poly1305` and `chacha20` crates are the reference for the
split between a raw cipher and an AEAD, and libsodium for the XChaCha20
construction as it is actually deployed.

XChaCha20 is an Internet-Draft rather than an RFC, and has been for years. It is
stable, unchanged and deployed: libsodium, age and PASETO v4 all ship it. This
is said here rather than left for a reader to discover from a dead link.

## Implementation status

Every function is a `todo()`. `novo test --isolate` runs the suite and every
assertion reaches `not implemented: chacha20-nv.<module>.<fn>`.

| Module | Public types | Public items | Implemented |
| --- | --- | --- | --- |
| `chacha20` | `Cc20State`, `Cc20Block`, `Cc20Sub` | 6 constants, 15 functions | no |
| `poly1305` | `Cc20Poly`, `Cc20Chunk` | 3 constants, 15 functions | no |
| `cc20bytes` | — | 9 functions | no |
| `cc20aead` | — | 4 constants, 11 functions | no |
| `cc20x` | — | 2 constants, 6 functions | no |
| `cc20err` | `Cc20Error` (with `impl Error`) | 2 functions | no |

## Licence

Apache-2.0.
