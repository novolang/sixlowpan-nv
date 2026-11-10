# sixlowpan-nv

An IEEE 802.15.4 frame carries at most 127 bytes, and an IPv6 header alone is
40. 6LoWPAN is the adaptation layer that makes IPv6 run over such a link: it
takes most of the header out where both ends can work it out, and breaks a
datagram too large for one frame into fragments. It is what Thread, Zigbee IP
and most wireless sensor networks put between the radio and the IP stack. This
package brings it to novo-lang: the formats of
[RFC 4944](https://www.rfc-editor.org/rfc/rfc4944) and the header
compression of [RFC 6282](https://www.rfc-editor.org/rfc/rfc6282).

## What it is

A border node, or a node that forwards, does five things with a frame. It
looks at the first byte to see what the frame is. It reads a mesh or
broadcast header if there is one. It expands the compressed IPv6 header. It
compresses one on the way out. It reassembles fragments, or makes them.
This package is those five, as functions over buffers the caller owns.

The package has no radio, no clock and no neighbour table. A program that
uses it owns the link and the timer. The package decides what the bytes
mean, and says when a half-built datagram is too old.

**Nothing is allocated.** A frame is a `Vec[u8; 128]` and a datagram a
`Vec[u8; 1280]`, the IPv6 minimum MTU, in the caller's frame or in static
storage. An address is four integers. The reassembly buffer is one `@value`
of 1,512 bytes. The whole package builds for a Cortex-M with no heap.

**Compression is by construction.** Each compression mode of RFC 6282 is a
function that rebuilds the address from what is in-line, the link address
and the context. The compressor picks the shortest mode whose rebuilt
address is the address it was given, so what it writes can always be read
back by the expander that is in the same package.

## Install

```
novo pkg add sixlowpan-nv
```

## Example

A UDP datagram between two link-local addresses is compressed to a few bytes
and expanded again. The link addresses are the 16-bit short addresses 1 and
2; both IPv6 addresses are derived from them, so neither travels.

```novo
use lowpan
use lowctx
use lowiphc

// The IPv6 and UDP headers of a datagram from fe80::ff:fe00:1 to
// fe80::ff:fe00:2, hop limit 64, UDP ports 5683 and 5684, two payload bytes.
fn build(var d: Vec[u8; 1280])
    for b in [0x60, 0x00, 0x00, 0x00, 0x00, 0x0a, 0x11, 0x40]
        let _a = d.push(b as u8)
    for k in 0..2
        for i in 0..16
            var b = 0
            if i == 0
                b = 0xFE
            else if i == 1
                b = 0x80
            else if i == 11
                b = 0xFF
            else if i == 12
                b = 0xFE
            else if i == 15
                b = k + 1
            let _b = d.push(b as u8)
    for b in [0x16, 0x33, 0x16, 0x34, 0x00, 0x0a, 0x12, 0x34, 0x41, 0x42]
        let _c = d.push(b as u8)

fn main() [io]
    var d: Vec[u8; 1280] = Vec.new()
    build(var d)
    var t = lowctx.ctx_table()
    let a = lowpan.ll_short(1)
    let b = lowpan.ll_short(2)

    // Out: 50 bytes of datagram become one frame of 11.
    var frame: Vec[u8; 128] = Vec.new()
    let n = lowiphc.pack_frame(var d, 50, a, b, var t, false, 127, var frame)
    println("${n} bytes: ${frame[0]} ${frame[1]} ${frame[2]}")   // 11 bytes: 126 51 240

    // In: the frame becomes the datagram again.
    var back: Vec[u8; 1280] = Vec.new()
    let r = lowiphc.decompress(var frame, 0, -1, a, b, var t, var back)
    println("status ${r.status}, ${r.size} bytes, hop limit ${back[7]}")   // status 0, 50 bytes, hop limit 64
```

Build and test with `novo pkg build` and `novo test tests --isolate`.

## What the package contains

| Module | Contents |
| --- | --- |
| `lowpan` | The dispatch byte of RFC 4944 section 5.1 and what it names; the mesh header and the broadcast header, read; `inspect`, which walks the headers in front of a datagram; the IPv6 address as four words and the IEEE 802.15.4 address, short or extended, and the interface identifier one makes. |
| `lowiphc` | LOWPAN_IPHC, RFC 6282: `compress`, `decompress` and `pack_frame`. Traffic class and flow label, hop limit, source and destination in every mode, unicast and multicast, stateless and with a context; the NHC for UDP with all four port forms and the checksum elided or in-line; the NHC for Hop-by-Hop, Routing, Destination Options and Mobility headers. |
| `lowctx` | The table of 16 contexts: `ctx_set`, `ctx_clear`, `ctx_find`, and the address a context and an identifier make. |
| `lowfrag` | The fragment headers FRAG1 and FRAGN, the sender (`frag_first`, `frag_next`), and `Reassembly`, a buffer for one datagram with the 60-second timer as a step the caller drives. |
| `lowcsum` | The Internet checksum and the UDP checksum over IPv6, which an elided checksum is put back with. |

Every `pub` declaration is documented where it is declared, and
[the package's page](https://novo-lang.org/packages/sixlowpan-nv) is that
reference.

## The rules a user needs

1. **A frame starts at the dispatch byte.** The bytes of the IEEE 802.15.4
   header are the radio driver's. `inspect` walks a mesh header, a broadcast
   header and says where the fragment or IPv6 dispatch starts. A frame whose
   first byte is `00xxxxxx` is not a LoWPAN frame, and `inspect` says so.
2. **The compressor writes the headers, not the payload.** `compress`
   answers how many bytes of the datagram its output stands for. The payload
   after that is the caller's to append, or the fragmenter's. `pack_frame`
   does both for a datagram that fits one frame.
3. **A link address is an argument.** An identifier that equals the one the
   link address makes is elided. A frame travelling with a short address
   elides `fe80::ff:fe00:XXXX`, one with an extended address elides the
   EUI-64 with its universal/local bit inverted. With no link address given
   the identifier travels.
4. **A context is state the caller keeps.** The package has no neighbour
   discovery, so the table is filled by `ctx_set` when the caller learns a
   context and emptied by `ctx_clear` when its lifetime ends. A context with
   the compression flag clear expands an address and is never used to
   compress one.
5. **The UDP checksum is elided only on the caller's word.** `compress` takes
   an `elide` flag that the upper layer authorises, verifies the checksum
   first, and refuses a wrong one with `IPHC_E_CHECKSUM`. RFC 6282 section
   4.3.2 also asks for an integrity check at the link layer; that is the
   caller's.
6. **The sizes come from the datagram, not the frame.** The IPv6 payload length
   and the UDP length are not in the compressed form, and `decompress` fills
   them from the size of the datagram: the frame's own length for a datagram
   in one frame, the fragment header's `datagram_size` for the first
   fragment.
7. **Fragments count the uncompressed datagram.** `datagram_size` and every
   offset are in bytes of the datagram as it is after expansion, in units of
   eight. Every fragment but the last is a multiple of eight bytes, which is
   why `frag_first` tells how far into the datagram it reached.
8. **A reassembly starts with its first fragment.** The first fragment is
   what says how far the compressed header expands, so a later fragment that
   arrives before it is not kept (`SIXLO_FRAG_NO_FIRST`). After it the later
   fragments come in any order. A copy of a fragment held is
   `SIXLO_FRAG_DUPLICATE`; one that overlaps another and differs discards the
   datagram, as RFC 4944 section 5.3 says.
9. **The caller keeps the identity and the clock.** A fragment belongs to a
   datagram by the sender and destination (the `key` the caller makes from
   the link addresses), the tag and the size. `reasm_tick` is told how many
   milliseconds passed and frees the buffer 60 seconds after the first
   fragment.
10. **A datagram complete is read from the buffer.** `SIXLO_FRAG_COMPLETE`
    leaves the datagram in `Reassembly.buf` with `Reassembly.size` bytes, its
    elided UDP checksum put back. It stays until `reasm_reset`.

## Running on a microcontroller

All five modules are `@tier(embedded)` and `@no_alloc` throughout, and the
package declares no effects. `tests/embedded_probe.nv` is a firmware program
for a border node. It reads a frame with a mesh header and a compressed UDP
datagram, expands the datagram and checks it, compresses it again and checks
that the same bytes come out, sends a 300-byte datagram as four fragments,
takes the fragments back in a different order into a reassembly buffer, and
checks the datagram and the timeout. It is compiled for a Cortex-M as part of
this package's checks, so the claim is built rather than asserted, and under
QEMU it prints `PASS: sixlowpan-embedded`.

The image is 37,036 bytes of flash and 528 bytes of static RAM, including the
runtime and the board's startup code. The program's own code is 24,034 bytes
of that flash. The probe's frame holds about 6.5 KB of buffers: two datagrams
of 1,288 bytes, the reassembly buffer of 1,512 bytes, a table of contexts of
776 bytes and the frames. A node that only forwards single frames needs one
datagram buffer and no reassembly buffer.

The device effects are the caller's. The package reads no register and no
clock, so a call costs what its caller's radio costs.

## What is not included

- **Neighbour discovery, RFC 6775.** Router Advertisements with the 6LoWPAN
  context option, the address registration, and duplicate address detection
  are not implemented. A context is put into the table by the caller, and
  `lowctx` is the interface a 6CO option would call.
- **The IPv6-in-IPv6 NHC (EID 7) and the Fragment header NHC (EID 2).** The
  compressor leaves a datagram that has either one uncompressed from that
  header on, and the expander refuses a packet that uses one with
  `IPHC_E_UNSUPPORTED`.
- **Eliding the padding of Hop-by-Hop and Destination Options headers.**
  RFC 6282 section 4.2 allows a compressor to; this one does not, and the
  expander restores padding it is given a header without.
- **Mesh routing.** The mesh header is read and its fields are given. Forwarding
  on them, the hop limit and the broadcast duplicate suppression are the
  caller's.
- **LOWPAN_HC1 and HC2 of RFC 4944.** The dispatch is recognised and the
  frame is not decoded; RFC 6282 replaced both.
- **Generic header compression, RFC 7400**, and the fragment recovery of RFC
  8931.
- **Reassembly of a later fragment before the first.** See rule 8.
- **Several datagrams in one `Reassembly`.** A node that takes several keeps
  several buffers and gives a fragment to the one whose key, tag and size
  match, and to a free one if none does.
- **The link layer.** The IEEE 802.15.4 frame, its addresses and its
  integrity check.
- **A routing header's effect on the UDP pseudo-header.** The checksum uses
  the destination in the IPv6 header.

## Related packages

- [cobs-nv](https://novo-lang.org/packages/cobs-nv) frames a byte stream on a
  serial link, the other way to carry a datagram on a constrained link.
- [coap-nv](https://novo-lang.org/packages/coap-nv) is the application protocol
  most 6LoWPAN nodes speak over the UDP this package carries.

## Tests

The suite is eight files under `tests/`. What they assert comes from the RFCs
rather than from an implementation: each compressed header is derived by hand
from RFC 6282 and the derivation is in the test's comment.

`tests/lowpan_tests.nv` holds the dispatch space at every edge, the mesh
header in each of its forms including the deep hops left, and the walk over
the headers in front of a datagram.

`tests/lowiphc_tests.nv` holds the header compression vector by vector: the
four forms of the traffic class and flow label, each hop limit, every source
and destination mode, the multicast forms at their edges, contexts including
a prefix of 112 bits and a stateful multicast address, the UDP port forms, the
elided checksum with its verification, and the extension headers singly and in
a chain. Each vector is run backwards.

`tests/lowiphc_zephyr_tests.nv` takes the headers of Zephyr's `6lo` test, at
commit `9d38cb9cfc636ff4853bd8368855a49390fc79e6`, and checks each compresses
to the size Zephyr expects and expands to the datagram it began with.

`tests/lowiphc_cut_tests.nv` expands every strict prefix of a header and
requires a short frame, for each in-line field.

`tests/lowfrag_tests.nv` derives the fragment headers by hand, cuts a
300-byte datagram into four fragments on eight-byte boundaries, and reassembles
it in order and out of order, with duplicates, overlap, a second datagram,
a changed size, every malformed fragment, an elided checksum put back, and the
timer.

`tests/lowctx_tests.nv` and `tests/lowcsum_tests.nv` hold the contexts and
the checksum, the latter with a UDP checksum summed by hand.

`tests/embedded_probe.nv` is the microcontroller program described above.

## Licence

Apache-2.0. See `LICENSE`.
