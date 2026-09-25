# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## 0.0.3 — 2026-09-25

One test changed so that it builds with novo 0.10.0.  In that release
`list.append` writes into the list it is given.  The test that feeds two
framed messages at once now builds them with `list.concat`, which answers
a new list and leaves the framed message as it was.  No change to the
interface.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-11

The **interface**, before anyone implements it. Every signature, every
type and every effect row is published; every body is `todo()`, and the
release is stamped `NOT IMPLEMENTED — interface only`. Adding this
package works and calling it panics.

- `DnsName` holds a name's uncompressed **wire bytes**, not a dotted
  string: a label may legally contain a dot, comparison is a byte
  compare, and `Str` does not link on a device. `name_from_text` and
  `name_text` are the two functions that cross to presentation form, and
  a device build does without them.
- `DnsHeader`, `DnsFlags`, `DnsQuestion`, `DnsRecord` and `DnsMessage` —
  RFC 1035 § 4.1 as values, with an unknown opcode, rcode, type or class
  keeping its number rather than being dropped.
- `DnsRdata` for A, AAAA, CNAME, NS, PTR, MX, TXT, SRV, SOA and OPT,
  with everything else kept as `RdataUnknown` so a resolver can forward
  what it does not understand. TXT is a list of strings, which is what
  § 3.3.14 says and what every implementation that concatenated them got
  wrong.
- `DnsCompressor` and `decode_name` — compression in both directions.
  The forward-pointer rule of § 4.1.4 is the loop guard: a pointer only
  ever goes backwards, so a strictly decreasing sequence terminates, and
  the hop count is the backstop behind it.
- `DnsOpt` and the OPT pseudo-record of RFC 6891, including the twelve-
  bit `extended_rcode` a caller that read only the header would miss.
- `DnsTcpReader`, `feed_tcp`, `take_tcp` and `frame_tcp` — the state
  between a socket read and a message, as a value the caller owns one of
  per connection. A read may split a message anywhere, including between
  the two bytes of the length prefix.

One toolchain defect travels with the release rather than being designed
around: a `@value` struct may not be the payload of a `Result`, an
optional, an enum or a boxed struct, so a wire codec whose decode can
fail has no fixed-width spelling available. The README says what that
costs.

### Design notes

The interface is a port of the resolver that lived in the novo-lang
repository, and five things changed in the port. A name stopped being a
`Str`: the reference joined labels with dots, which loses a label that
contains one and puts the codec out of reach of a device. The three
ad-hoc result enums the reference carried, one per parse, became the one
`Result`. Resource records arrived: the reference parsed no RDATA at
all, so A, AAAA, CNAME, MX, TXT, SRV, NS, SOA and PTR are all new.
Compression gained its encoding half, because the reference followed
pointers and never wrote one. EDNS(0) arrived, which is what lifts the
reference's 512-byte cap and its frequent fallback to TCP.

The error type is spelled `DnsCodecError` and not `DnsError`, because
the resolver already declares a `DnsError` and a type's identity is its
name across the whole assembly. The two can be in one program today.
