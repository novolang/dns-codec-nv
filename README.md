# dns-codec-nv

The Domain Name System carries its queries and its answers in one
message format, specified in
[RFC 1035](https://www.rfc-editor.org/rfc/rfc1035), section 4. This
package brings that format to novo-lang as a codec with no socket under
it. It reads bytes into messages and writes messages back into bytes,
and every decision about what to do with one belongs to the caller.
[mdns-nv](https://novo-lang.org/packages/mdns-nv) is built on it: that
package is multicast DNS and service discovery on the local link, and
it uses these messages on a different port under different rules. This
package depends on nothing.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What a DNS message is

A message is a twelve-byte **header** followed by four sections: the
questions, the answers, the authority records and the additional
records. The header carries a count for each section, so a reader knows
how many entries to expect before it reads any. Section 4.1 defines the
layout. Every multi-byte field is big-endian, which is the DNS
convention and the opposite of most wire formats a reader will have met.

A **domain name** on the wire is a sequence of **labels**. Each label is
one length byte followed by that many bytes, and the sequence ends with
a zero length byte. `www.example.com` is seventeen bytes:
`03 77 77 77 07 65 78 61 6D 70 6C 65 03 63 6F 6D 00`. The root is the
single byte `00`. A label holds bytes and not characters, so a label may
contain a dot, and the dotted spelling is a presentation form rather
than the name itself.

A **resource record** is a name, a type, a class, a lifetime in seconds
and a block of type-specific data called the **RDATA**. Section 4.1.3
defines it. The type says what the RDATA means: an A record holds four
octets of IPv4 address, a TXT record holds a list of byte strings, an
SRV record holds a host and a port.

**Name compression** is what keeps a message small. Section 4.1.4 lets
any name in a message end in a two-byte **pointer** to a name that
appeared earlier, marked by the top two bits `11`. A reader that meets
one jumps to that offset and carries on reading labels there. A pointer
is an offset from the start of the whole message, so a reader given
only a slice cannot follow one.

**EDNS(0)** is the extension mechanism of
[RFC 6891](https://www.rfc-editor.org/rfc/rfc6891). It adds one
pseudo-record, the **OPT record**, to the additional section. That
record's class field holds the largest reply the sender can accept, and
its lifetime field holds eight more bits of response code, a version and
a flags word. It is how a sender says it can take a reply larger than
512 bytes.

These are the numbers the format fixes.

| Constant | Value | What it is |
| --- | --- | --- |
| `HEADER_LEN` | 12 bytes | The header, which is never any other length (section 4.1.1) |
| `LABEL_MAX` | 63 bytes | The longest a single label may be (section 2.3.4) |
| `NAME_MAX` | 255 bytes | The longest a name may be in its wire form, terminator included (section 2.3.4) |
| `UDP_MESSAGE_MAX` | 512 bytes | The largest message plain UDP carries without EDNS(0) (section 4.2.1) |
| `EDNS_PAYLOAD_DEFAULT` | 1232 bytes | The reply size to advertise on a path you know nothing about (RFC 6891 section 6.2.5) |
| `POINTER_TAG` | 0xC0 | The two top bits that mark a compression pointer (section 4.1.4) |
| `EDNS_VERSION` | 0 | The EDNS version this package writes and understands |

This package performs no input or output. It holds no buffer of its own,
opens no socket and consults no clock. Every function is arithmetic over
bytes the caller already holds, so the same code serves a stub resolver,
an authoritative server, a capture tool reading a file, and a device
doing service discovery on a local link.

## Install

```
novo pkg add dns-codec-nv
```

## Example

```novo
use dns_codec

fn main() [io]
    // One response datagram, as it came off the socket.
    let datagram: [u8] = [0x12 as u8, 0x34 as u8, 0x81 as u8, 0x80 as u8,
                          0x00 as u8, 0x01 as u8, 0x00 as u8, 0x01 as u8,
                          0x00 as u8, 0x00 as u8, 0x00 as u8, 0x01 as u8]

    // The header is the first twelve bytes, and it says whether the
    // rest of the answer arrived.
    match dns_codec.decode_header(datagram)
        Err(e)  => println("the header is wrong: ${e.message()}")
        Ok(hdr) =>
            if hdr.flags.truncated
                println("the answer did not fit: ask again over TCP")

    // Read the whole message, then look at what each answer holds.
    match dns_codec.decode_message(datagram)
        Err(e)  => println("the message is wrong: ${e.message()}")
        Ok(msg) =>
            for record in msg.answers
                match record.data
                    RdataA(octets)        => println("${list.len(octets)} bytes of address")
                    RdataCname(target)    => println("an alias of ${dns_codec.name_label_count(target)} labels")
                    RdataTxt(strings)     => println("${list.len(strings)} text strings")
                    RdataUnknown(code, _) => println("type ${code}, kept as its bytes")
                    _                     => println("some other record")

            // The OPT pseudo-record, if the responder sent one. It
            // carries the largest reply this path will accept.
            match dns_codec.find_opt(msg)
                Err(e)    => println("the OPT record is wrong: ${e.message()}")
                Ok(found) =>
                    match found
                        None      => println("no EDNS0: ${dns_codec.UDP_MESSAGE_MAX} bytes is the limit")
                        Some(opt) => println("EDNS0: up to ${opt.payload_size} bytes")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a `not implemented: dns_codec.<fn>`
panic. The tests are the specification the implementation will have to
satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `dns_codec` | The whole surface. The sizes the format fixes, names as wire bytes, the header and its flags, questions and resource records, the ten record types this package models, name compression in both directions, whole-message encoding and decoding, the EDNS(0) OPT record, and a feed-and-drain reader for TCP's length-prefixed framing. |

## How to choose an entry point

There are two ways to read an incoming message, and they differ in how
much of the message you want at once.

**`decode_message` reads the whole message.** It reads each section to
the count the header declared and hands back every question and every
record. Use it in a resolver, a responder, or anything that acts on an
answer.

**`decode_header`, `decode_question`, `decode_record` and `decode_name`
read one piece each.** Each takes the whole message and an offset, and
each answers where the reader should carry on. Use them in a capture
tool, or on a device that wants the header before it decides whether the
rest is worth parsing.

There are also two ways to write a message. **`encode_message` writes
all of it**, compressing names as it goes. **`encode_message_limited`
writes as much as fits**, drops answers from the end, and sets the
truncated bit. An authoritative server answering over UDP wants the
second one.

Framing over TCP has the same pair. **`scan_tcp` reads only the length
prefix** and answers how many bytes at the front of a buffer are one
framed message, copying nothing. **`DnsTcpReader` buffers for you**:
feed it what the socket handed over, then drain it until it stops
producing messages. Keep one reader per connection.

## The rules a user needs

1. **A name is its wire bytes, not a dotted string.** `DnsName` holds
   the uncompressed length-prefixed encoding. A label may legally
   contain a dot, so a dotted string is lossy, and section 5.1's
   presentation form escapes a dot inside a label as `\.` and any byte
   as `\DDD`. `name_from_text` and `name_text` are the only two
   functions that cross to text.
2. **A label may not exceed 63 bytes and a name's wire form may not
   exceed 255** (section 2.3.4). `name_of_labels` refuses both, which is
   where a caller building a name out of user input is stopped.
3. **Names compare without regard to ASCII case** (section 2.3.3).
   `EXAMPLE.COM` and `example.com` are the same name. Use `name_eq`
   rather than comparing the wire bytes.
4. **`decode_name` takes the whole message, not a slice.** A compression
   pointer is an offset from the message's first byte, so a reader
   handed a slice cannot follow one (section 4.1.4).
5. **`DnsNameRead.next` is past the pointer, not past the name.** When a
   name ends in a pointer, the record's own next field begins two bytes
   after the pointer started, not where the expanded name ended.
   Confusing the two is the classic section 4.1.4 error.
6. **A pointer only ever points backwards.** Section 4.1.4 writes a
   pointer to an earlier name, so `decode_name` refuses a forward or
   self-referential pointer with `DnsForwardPointer`. `DnsPointerLoop`
   is the hop-count backstop behind that rule.
7. **A pointer reaches only the first 16,383 bytes of a message.** The
   offset is fourteen bits. `encode_name` writes a name past that in
   full and does not remember it.
8. **`encode_name`'s `offset` argument must be where the bytes will
   land.** The compressor remembers offsets, and a caller that writes
   the result somewhere else produces a message whose pointers name the
   wrong names.
9. **Plain UDP carries 512 bytes** (section 4.2.1). A larger answer
   comes back truncated with the TC bit set, and the resolver asks again
   over TCP. Sending an OPT record that advertises 1232 bytes is what
   avoids that round trip (RFC 6891 section 6.2.5).
10. **An OPT record's name is always the root, and there is at most one
    per message** (RFC 6891 section 6.1.1). `read_opt` and `find_opt`
    refuse anything else with `DnsBadOptRecord`.
11. **The response code is twelve bits when an OPT record is present**
    (RFC 6891 section 6.1.3). The header holds the low four and the OPT
    record the high eight. Call `extended_rcode`, because a caller
    reading only the header sees `DnsNotImplemented` where the responder
    said BADVERS, and retries forever.
12. **A TXT record is a list of byte strings, not one string**
    (section 3.3.14). `RdataTxt` holds the list, and joining them
    changes what the record says.
13. **Names inside RDATA are compressed only for the record types
    RFC 1035 itself defines.** Every type specified later, SRV among
    them, writes its names in full. `encode_name_uncompressed` is that
    case.
14. **Over TCP a message is preceded by its own two-byte big-endian
    length** (section 4.2.2). A socket read may split a message
    anywhere, including between the two bytes of that prefix.
    `feed_tcp` hands back one message per call, so a read that completed
    two leaves the second inside and `take_tcp` is what gets it.
15. **An unknown opcode, response code, type or class keeps its
    number.** `DnsTypeOther`, `DnsRcodeOther` and `RdataUnknown` hold
    what arrived, so a message a resolver forwards comes out the way it
    went in.

## Running on a microcontroller

novo-lang lets a package state which of its modules can run on a device
with no heap allocator, and the compiler checks that claim on every
build. Here the claim covers the whole package. Every function is
arithmetic over bytes the caller supplies, and the package has no
dependencies.

```bash
novo build --target=nrf52-qemu src/main.nv
```

`tests/embedded_probe.nv` is that claim as a program that either builds
or does not. It builds, producing a Cortex-M4 executable.

Two functions are outside the claim. `name_from_text` and `name_text`
take or return a `Str`, and the device runtime has no string support, so
a device build does without them. This is why a name is its wire bytes
everywhere else here: on a device a name arrives in a packet, is
compared with another name from a packet, and goes back out in a packet,
and it is never typed in.

## What is not included

- **A resolver.** No socket, no UDP retry or backoff, no TCP fallback
  when an answer comes back truncated, no cache and no clock to expire
  one, no root hints and no iterative walk down from them. All of those
  need network and clock access, and this package has neither.
  `std.net` is the socket.
- **DNSSEC validation.** `dnssec_ok` is a bit in the OPT record and this
  package carries it. RRSIG, DNSKEY, DS and NSEC arrive as
  `RdataUnknown`, and checking a signature is a cryptographic job for a
  package above this one.
- **Zone files.** A zone file is presentation-format text, and the
  functions that read one belong beside the server that loads it.
- **A decode that allocates nothing.** Every decoded value here is a
  boxed struct or an enum with a payload. A fixed-width value struct may
  not be the payload of a `Result`, an optional or an enum on this
  toolchain, and every decode in a wire codec can fail. The encode path
  and the constants are unaffected.

## Related packages

- [mdns-nv](https://novo-lang.org/packages/mdns-nv) is the host half a
  reader wants when the name is on the local link. It is multicast DNS
  and DNS-based service discovery: the probe that claims a name, the
  responder, the querier and its cache, and the multicast socket. It
  reads and writes these messages, so a program that needs to find a
  printer wants that package and gets this one with it. A program that
  needs the wire format itself, for a resolver or a capture tool, wants
  this one alone.
- [ipaddr-nv](https://novo-lang.org/packages/ipaddr-nv) turns the four
  octets of an A record and the sixteen of an AAAA record into an
  address value that prints and parses. This package hands back the
  octets and stops there.
- [http-codec-nv](https://novo-lang.org/packages/http-codec-nv) is the
  same shape for HTTP/1.1: a codec with no socket, for a caller that
  owns the transport.
- `std.net` in the standard library is the socket. A resolver is this
  package for the bytes and `std.net` for the datagram.

## Tests

```bash
novo test tests/dns_codec_tests.nv       # 48 tests against the signatures
```

The byte strings the suite asserts against are the specifications' own.
RFC 1035 section 4.1.4's compression example is laid out at the offsets
the RFC itself uses, so the three compressed forms it shows are read
back from the bytes it prints: a name that ends in a pointer, a pointer
into the middle of an earlier name, and the root. RFC 2782 supplies the
SRV record, RFC 3596 the AAAA record and RFC 6891 section 6.1.2 the OPT
record.

The suite checks that a name survives the crossing to text and back,
that a header round trips, that a forward pointer and a self-pointer are
refused, that an encoder reuses a pointer for a repeated name, that a
record whose RDLENGTH disagrees with its contents is refused, that a
limited encode truncates and says so, that the response code is read as
twelve bits, and that a message split between the two bytes of a TCP
length prefix is reassembled.

The tests compile today and fail at run, each on the `not implemented:
dns_codec.<fn>` panic that is its body. That is the expected state of an
interface release. They turn green one at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| The `HEADER_LEN`, `LABEL_MAX`, `NAME_MAX`, `UDP_MESSAGE_MAX`, `EDNS_PAYLOAD_DEFAULT`, `POINTER_TAG` and `EDNS_VERSION` constants | yes (they are constants) |
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

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
