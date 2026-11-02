# Pixel Babel

A [Library of Babel](https://en.wikipedia.org/wiki/The_Library_of_Babel) for pixel
images. Every 8×8, 16×16 and 32×32 image using a fixed 16-colour palette has exactly
one address, and every address decodes to exactly one image. One image, the
all-white page, has no address; see *How it works*. There is no
database and no server: an index of the image space is turned into page contents by
modular multiplication, and back again by multiplying with its inverse.

The whole program is one HTML file with no dependencies. Open `index.html` in a
modern browser. It runs a WASM pixel codec built at runtime from opcodes inside the
file, with a pure-JS reference implementation used for base32, CRC32 and verification.

## Usage

| Action | What happens |
| --- | --- |
| Show page | address → image (READ) |
| Find address | image → address (FIND) |
| Random page | draws a uniform random index |
| Paint / Fill / New blank page | edit pixels, then Find |
| Embed text | stores UTF-8 in the pixel bytes of a page |
| Colour preview upload | rescales any image to a page, maps to the 16 colours, optionally dithered. Lossy |
| Lossless archive upload | stores the file's bytes in one or more chained pages, CRC32-checked. Download restores the original file |
| Download exhibit PNG | indexed 4-bit PNG whose `tEXt` chunk carries the address |
| Download standalone exhibit | a single HTML file that decodes and renders one address |

An address is the page index in base32 (RFC4648 alphabet, no padding) followed by
seven checksum characters (base32 of the CRC32 over the index bytes). Lengths are
59, 212 and 827 characters for 8×8, 16×16 and 32×32. Input is case-insensitive;
spaces, dots, commas, slashes and dashes are ignored. A mistyped address either
fails the checksum or decodes to the same image.

## How it works

A page of `w×w` pixels, each an index 0–15 into the palette, is a base-16 number
`V` with `L = w²` digits. With `N = 16^L − 1`:

```
READ  (address → image):  value = (C · index) mod N
FIND  (image → address):  index = (I · value) mod N
```

`C` is the first odd integer ≥ `0x9E3779B97F4A7C15` coprime to `N`; `I` is its
modular multiplicative inverse, found with the extended Euclidean algorithm over
`BigInt`. Because `gcd(C, N) = 1`, multiplication by `C` permutes the residues, so
the two directions are exact inverses. This is the method used by
[libraryofbabel.app](https://github.com/tdjsnelling/babel), applied to pixels.

Indices run 1..N. The value `V = N` (every pixel the last palette entry, a pure
white page) has no index, since `(I · N) mod N = 0`. Exactly one image out of
`16^L` is unreachable; the reference implementations lose one book for the same
reason. The self-test checks this case directly.

Capacity per page is `L × 4` bits: 256, 1024 and 4096 bits for 8×8, 16×16 and
32×32. A page can be read as colour indices or as raw data, so the same mechanism
that hides a text message also archives any file bit-for-bit, chained across
pages when the file is larger than one page.

## Tests

Open `index.html?test=1` or press *Run self-tests*. The suite verifies: WASM versus
JS equality for the pixel codec, CRC32 and base32 test vectors, exhaustive bijection
properties for every size, image/address round-trips, boundary pages, checksum
rejection of typos, embedded messages, exhibit PNG round-trips and lossless archive
round-trips including multi-page and corrupted sets. Payloads are random
(property-based testing); the fixed vectors and the verdict logic are deterministic.

## Repository layout

```
index.html   the library program, its tests and the documentation
otp.html     PBOTP v1: one-time-pad / password-stream encryption with a
             self-contained DEFLATE and SHA-256 (see below)
LICENSE      GNU Affero General Public License v3 or later
README.md    this file
```

## OTP (otp.html)

`otp.html` encrypts any text or file, losslessly:

- *One-time pad mode*: a uniform random pad as long as the payload, generated
  with `crypto.getRandomValues`, shown once, stored nowhere. XOR applied after
  optional DEFLATE compression. Perfect secrecy holds under the classical pad
  rules (random, secret, at least payload length, never reused).
- *Password-stream mode*: keystream blocks `SHA-256(salt || password ||
  counter)`. Practical key transport, explicitly not perfect secrecy.
- Container `PBOTP v1`: magic, version, flags, payload length, 4-byte
  SHA-256 tag, 8-byte salt, ciphertext; transported as RFC4648 base32. Wrong
  keys, truncation and single bit flips are detected via the tag.
- Compression runs **before** encryption only. XOR with a random pad yields
  uniform bytes, which no lossless compressor can shrink; the self-test
  demonstrates this. Encrypt-then-compress is not offered.
- SHA-256 (FIPS 180-4) and DEFLATE (RFC 1951, stored/fixed/dynamic blocks)
  are implemented in the file itself. Streams written by other zlib-family
  tools decode here, and streams written here decode under Node's zlib
  (cross-implementation tests). No dependencies, works from file://.

## Prior art and credits

- [github.com/tdjsnelling/babel](https://github.com/tdjsnelling/babel) (GPL-3.0) —
  the reversible-index method this project applies to images.
- [github.com/librarianofbabel/libraryofbabel.info-algo](https://github.com/librarianofbabel/libraryofbabel.info-algo)
  (Jonathan Basile, CC BY-NC-SA) — a different invertible map, described here for
  comparison. No code from that repository is reproduced in this project.

Both are cited as algorithm references. This implementation is original JavaScript,
BigInt and hand-assembled WebAssembly.

## License

Copyright (C) 2026 moon440

Pixel Babel is free software: you can redistribute it and/or modify it under the
terms of the GNU Affero General Public License as published by the Free Software
Foundation, either version 3 of the License, or (at your option) any later version.
See [LICENSE](LICENSE) for the full text.

Pixel Babel is distributed in the hope that it will be useful, but WITHOUT ANY
WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A
PARTICULAR PURPOSE. See the GNU Affero General Public License for more details.

The AGPL differs from the GPL in section 13: if you modify this program and make it
available to users over a network, you must also make its source code available to
those users. Any page generated by this program is just pixel data and carries no
license obligation.
