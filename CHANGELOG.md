# Changelog

All notable changes to sixlowpan-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0

The first release.

### Added

- **`lowpan`**, the dispatch space of RFC 4944 section 5.1 and RFC 6282
  section 5, the mesh header and the broadcast header, read, `inspect` for the
  headers in front of a datagram, the IPv6 address as four words, the IEEE
  802.15.4 short and extended addresses, and the interface identifier and the
  link-local address one makes.
- **`lowiphc`**, LOWPAN_IPHC of RFC 6282: `compress`, `decompress` and
  `pack_frame`, with every TF, HLIM, SAC, SAM, M, DAC and DAM combination, the
  context byte, the NHC for UDP with port compression and checksum elision, and
  the NHC for the Hop-by-Hop, Routing, Destination Options and Mobility
  headers.
- **`lowctx`**, a table of 16 contexts with `ctx_set`, `ctx_clear` and
  `ctx_find`.
- **`lowfrag`**, the fragment headers FRAG1 and FRAGN, a sender (`frag_first`,
  `frag_next`) and `Reassembly`, a buffer for one datagram of fixed capacity
  with a timer of 60 seconds that the caller drives.
- **`lowcsum`**, the Internet checksum and the UDP checksum over IPv6.
- **`tests/embedded_probe.nv`**, a firmware program for a border node, built
  for `--target=nrf52-qemu`. It prints `PASS: sixlowpan-embedded` under QEMU.
  The image is 37,036 bytes of flash and 528 bytes of static RAM.
- Eight test suites with compressed headers derived by hand, and the header
  cases of Zephyr's 6lo test.

### Not included

- Neighbour discovery for 6LoWPAN (RFC 6775).
- The IPv6-in-IPv6 NHC and the Fragment header NHC.
- Eliding the padding of Hop-by-Hop and Destination Options headers.
- Mesh routing, LOWPAN_HC1, generic header compression (RFC 7400) and fragment
  recovery (RFC 8931).
- Reassembly of a later fragment that arrives before the first.

### Known

- A `Vec` whose element is an alias imported from another module is not built,
  so the device probe spells out `Vec[Vec[u8; 128]; 4]` where the type of a
  frame is `Frame`. The filing is
  `a-vec-of-an-alias-imported-from-another-module-is-not-resolved`.

This release requires novo 0.23.0 or later.
