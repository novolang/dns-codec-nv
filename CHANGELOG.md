# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## 0.1.1 — 2026-10-25

A call that passes a variable to a `var` parameter now writes the `var` marker, which the 0.21.1 compiler asks for (E2050).  The package needs Novo 0.21.1 or later.  No signature and no behaviour changed.

## 0.1.0 — 2026-09-27

First implementation of the interface published as 0.0.1.

- Every function has a body: names and their presentation form, the
  header and its flags, questions and the ten modelled record types,
  compression in both directions, whole messages with and without a
  size limit, the EDNS(0) OPT record, and TCP framing.
- `DnsCodecError` gains `DnsEmptyLabel(index)`. `name_of_labels`
  refuses an empty label with it, because an empty label in the middle
  of a name is a terminator and would end the name early. A `match`
  over the error type needs the new arm. This is the one breaking
  change.
- `DnsPointerLoop` now means a pointer that lands inside the labels the
  same walk has already read, or a walk longer than 255 steps.
  `DnsForwardPointer` still refuses a pointer that goes forwards or at
  itself.
- `encode_message_limited` keeps records in wire order, answers then
  authority then additional, and drops the first record that does not
  fit and every record after it.
- `encode_record` writes a TXT string longer than 255 bytes as
  consecutive strings of at most 255.
- `name_text` escapes each of `"().;\@$` with a backslash and every
  byte outside printable ASCII as `\DDD`, so its answer always reads
  back through `name_from_text`.
- Every function is `@tier(rt)`, and the whole module, the two text
  functions included, builds for `--target=nrf52-qemu`.
- Three of the published API tests asserted wrong numbers and are
  corrected: the expanded length of `FOO.F.ISI.ARPA` is 16 bytes, not
  18, and the A and SRV record tests read their fields one byte late.
- Two new suites: `tests/wire_tests.nv`, whole messages in hex checked
  by hand against the RFCs, and `tests/roundtrip_tests.nv`, properties
  over generated messages. `tests/coverage.sh` measures line coverage
  over `src/`.
- The toolchain floor is 0.13.0.

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
