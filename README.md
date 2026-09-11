# dns-codec-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

RFC 1035's wire format as a codec with no socket: the twelve-byte
header, the question section, the resource records a resolver meets (A,
AAAA, CNAME, MX, TXT, SRV, NS, SOA, PTR), name compression in both
directions — encoding with pointer reuse, decoding with the loop guard —
EDNS0's OPT pseudo-record (RFC 6891), and a feed-and-drain reader for
TCP's length-prefixed framing.

It turns bytes into messages and messages into bytes, and every decision
about what to DO with one belongs to the caller.  The same codec serves
a stub resolver, an authoritative server, a capture tool and a device
doing service discovery on a local link, because it has no idea which
one it is.

## Adding it, and checking it

```bash
novo pkg add dns-codec-nv          # into your novo.toml
novo pkg build                     # type- and effect-check the package
novo test tests/dns_codec_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion below the first constants fails with `not implemented:
dns_codec.<fn>`.  They turn green one at a time as bodies land.

## The one example that will work

```novo
use dns_codec

// A resolver's read path: bytes from a socket, an answer or nothing.
fn on_datagram(bytes: [u8], id: Int) -> ?DnsMessage
    match dns_codec.decode_message(bytes)
        Err(_)  => None
        Ok(msg) =>
            if msg.header.id != id
                return None
            if msg.header.flags.truncated
                return None       // ask again over TCP
            Some(msg)
```

## The layer, and why

`core`.  A message in, a message out.  The socket, the retry policy, the
cache and the clock that expires it are the resolver's, and none of them
is here.

`tests/embedded_probe.nv` is that claim in a form that either builds or
does not, and **it builds**: `novo build --target=nrf52-qemu` produces a
Cortex-M4 ELF.  This package has no dependencies, so the shard audit's
`core-embedded` row runs that build itself and passes.

Two functions are **not** in the probe and cannot be on a device:
`name_from_text` and `name_text`.  They take or return a `Str`, and the
embedded runtime defines no `novo_str_byte_at`, so a device build does
without them.  That is not a limitation the package works around — it is
the reason the package is shaped the way it is.

## The load-bearing interface

**A name is its wire bytes.**

```novo
pub struct DnsName
    wire: [u8]
```

Not a dotted string.  Three reasons, and the third is the one that
decided it:

- a label may legally contain a dot, so a dotted string is **lossy** —
  `a\.b.example.com` is two labels and every string representation has
  to escape its way back out of that;
- comparison is a byte compare and compression is a **suffix match**,
  and both of those go through a split and a rejoin if the name is text;
- `Str` does not link at `@tier(embedded)`, so a name held as text puts
  the whole package out of reach of the device that most wants it.

`name_from_text` and `name_text` are the two functions that cross to
presentation form, and a device build does without them.

The second decision is the pair that reads a compressed name:

```novo
pub struct DnsNameRead
    name: DnsName
    next: Int
```

`next` is **not** where the name ended when the name ended in a pointer.
It is the offset just past the pointer, because that is where the
record's own next field is.  Confusing those two is the § 4.1.4 bug
every implementation writes once, and a named field is cheaper than a
second return value nobody reads the order of.

The third is the loop guard, and it is a rule rather than a set:

```novo
    DnsForwardPointer(from: Int, to: Int)
    DnsPointerLoop(offset: Int)
```

§ 4.1.4 only ever writes a pointer to an **earlier** name.  Enforcing
that makes a loop impossible — a strictly decreasing sequence of offsets
terminates — and it costs one comparison rather than a visited set.
`DnsPointerLoop` is the backstop for the hop count, kept because a
defence with one layer is a defence with a bug in it.

## Where the no-allocation shape ran out

Every decoded value here is a boxed struct or an enum with a payload, and
none of it is a `@value` struct.  That is not an oversight: a `@value`
struct may not be the payload of a `Result`, an optional, an enum or a
boxed struct (E2015), so the moment a decode can fail — which is every
decode in a wire codec — the fixed-width spelling is unavailable.  The
defect is filed from this lane and the types stay as they are; a package
that changed its shape to fit a compiler limitation would be publishing
the limitation as a design.

What the package does instead is what a `core` package can always do:
hold no buffer of its own, take every byte from the caller, and keep the
one piece of state a streaming codec needs — `DnsTcpReader` — as a value
the caller owns one of per connection.

## What is not here, and what a consumer should expect

**The resolver.**  Everything `orbit/dns` keeps: the socket, the UDP
retry and backoff, the TCP fallback a truncated response triggers, the
cache and the TTL clock that expires it, the root hints and the
iterative walk down from them.  All of those need `[net]` or `[time]`,
and a `core` package has neither.

**DNSSEC.**  `dnssec_ok` is a bit in the OPT record and this package
carries it; RRSIG, DNSKEY, DS and NSEC are `RdataUnknown` and their
validation is a cryptographic job for a package above this one.

**Presentation-format zone files.**  A zone file is text, and the
functions that read it belong in a host package beside the server that
loads one.

## The reference implementation

`orbit/dns/src/main.nv` — 1,048 lines, of which roughly 430 are the
codec and the rest are the resolver and its tests.  What moves is
`parse_header`, `parse_name_step`, `parse_question`, the `serialize_*`
family, `build_query`, `frame_tcp` and `unframe_tcp`.  What stays is
`Resolver`, `resolve`, `resolve_udp`, `resolve_with_retries`,
`exchange_tcp` and `recv_tcp_body` — every one of them `[net]` — along
with `zone_lookup` and `handle_query`, which are a demonstration server
rather than a library.

Five things change in the port, and each is a thing the reference could
not have or did not do.

- **A name stops being a `Str`.**  `parse_name` returns
  `NameOk(name: Str, next: Int)` with the labels joined by dots, which
  loses a label that contains one and puts the resolver out of reach of
  a device.
- **The ad-hoc result enums go.**  `HeaderResult`, `QuestionResult` and
  `NameResult` each exist because the file predates `Result`; three
  types become one.
- **Resource records arrive.**  The reference parses no RDATA at all —
  it serialises an A record for its demonstration zone and reads none —
  so A, AAAA, CNAME, MX, TXT, SRV, NS, SOA and PTR are all new, and TXT
  is a list of strings rather than one string because § 3.3.14 says so.
- **Compression gains its encoding half.**  `parse_name_step` follows
  pointers; `serialize_name` never writes one, so every name in a
  message the reference builds is written in full.  `DnsCompressor` is
  that missing half.
- **EDNS0 arrives**, which is why the reference is capped at 512 bytes
  and falls back to TCP more often than it needs to.

One thing the reference got right and this package keeps: the forward-
pointer rule.  `parse_name_step` refuses `target >= off` and caps the
hop count at 128, which is the correct defence and the cheap one.

One name collides and the split has to resolve it: `orbit/dns` declares
`error_kind DnsError`, and a type's identity is its name across the
whole assembly.  This package deliberately spells its own
`DnsCodecError`, so the two can be in one program today; the resolver
deletes its own the day it adopts this one.

RFC 1035, RFC 2782, RFC 3596 and RFC 6891 are the sources of the test
vectors, and `tests/dns_codec_tests.nv` lays RFC 1035 § 4.1.4's
compression example out at the offsets the RFC itself uses.

## Status

| item | implemented |
| --- | --- |
| `HEADER_LEN`, `LABEL_MAX`, `NAME_MAX`, `UDP_MESSAGE_MAX`, `EDNS_PAYLOAD_DEFAULT`, `POINTER_TAG`, `EDNS_VERSION` | yes — they are constants |
| `dns_codec.root_name`, `.name_of_labels`, `.name_labels`, `.name_label_count`, `.name_eq` | no |
| `dns_codec.name_from_text`, `.name_text` | no |
| `dns_codec.encode_header`, `.decode_header`, `.encode_flags`, `.decode_flags` | no |
| `dns_codec.type_code`, `.type_of_code`, `.class_code`, `.class_of_code` | no |
| `dns_codec.compressor`, `.encode_name`, `.encode_name_uncompressed`, `.decode_name` | no |
| `dns_codec.encode_question`, `.decode_question`, `.encode_record`, `.decode_record` | no |
| `dns_codec.encode_message`, `.encode_message_limited`, `.decode_message` | no |
| `dns_codec.opt_record`, `.read_opt`, `.find_opt`, `.extended_rcode` | no |
| `dns_codec.frame_tcp`, `.scan_tcp`, `.tcp_reader`, `.feed_tcp`, `.take_tcp`, `.tcp_pending_len` | no |
| `dns_codec.DnsCodecError.message` | no |
