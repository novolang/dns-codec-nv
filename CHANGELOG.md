# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

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
