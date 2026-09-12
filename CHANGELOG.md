# Changelog

All notable changes to ssh-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.1 — 2026-09-12

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `sshfault` — the faults and RFC 4253 § 11.1's disconnect reasons,
  `[]` throughout: "no matching key exchange method" and "host key not
  verifiable" are a configuration problem and an attack, and a client
  that reported "connection closed" for both told its user nothing.
- `sshpkt` — the binary packet protocol and the four wire encodings,
  `[]` throughout: the packet length is ENCRYPTED under a second key so
  the reader is fed and drained rather than handed a buffer, the
  sequence number is counted and never sent and does NOT reset on a
  rekey, the padding is the caller's random bytes, and `mpint`'s leading
  zero is a named function because getting it wrong fails half of all
  handshakes.
- `sshkex` — negotiation, the exchange hash and the six keys, `[]`
  throughout: first match in the CLIENT's order, the exchange hash
  input built as its own function because that is what goes wrong and
  the failure points elsewhere, the first exchange's hash as the session
  id forever, and a key derivation that has to extend past one digest
  because ChaCha20-Poly1305 wants 64 bytes from SHA-256.
- `sshhostkey` — `known_hosts` and the verdict, `[]` throughout: unknown
  and changed are DIFFERENT variants, a revoked marker is terminal, an
  unreadable line is skipped and counted rather than failing the file,
  the port is part of an entry, and there is no default that accepts an
  unknown host.
- `sshkey` — the OpenSSH private key container, `[]` throughout: not
  PEM and not ASN.1, the two check integers as the password check, and
  an encrypted key named as encrypted rather than reported as corrupt.
- `sshauth` — the authentication exchange, `[]` throughout and THE
  SIGNATURE EXCLUDED: the two-phase `publickey` query so a client with
  five keys does not make five taps on a token, partial success as not
  a failure, and message 60 read according to the method in flight.
- `sshchan` — channels and the flow-control window, `[]` throughout:
  `may_send` as the only correct answer to "can I write this",
  extended data counted against the same window, an adjustment owed at
  half rather than at zero, and `SshFaultWindowExhausted` as this
  package's answer to the one failure the protocol does not report.
- `sshconn` — the host half: `[io, net]` and nothing else, with
  `SshAction` as the enum the host pumps and `SshActionSign` as the
  variant that makes an agent and a hardware token ordinary cases.
- API tests in `tests/sshwire_tests.nv` — a whole handshake's
  arithmetic with no socket and no randomness — and
  `tests/sshsession_tests.nv`.  Red until the bodies land.

### Named as missing

**RSA, and it is the one that matters**: a large share of deployed
servers present an `ssh-rsa` or `rsa-sha2-256` host key and nothing on
this grid implements RSA.  The row is `rsa-nv`, crypto/core over
bigint-nv; tls-nv and dkim-nv name the same gap.  Also named: ECDSA over
P-256 (the same row tls-nv and acme-nv name), `bcrypt_pbkdf` and
AES-256-CTR for an encrypted private key, and `ssh-agent-nv` — which is
a protocol rather than a primitive and is the answer for most real
callers, and which drops in unchanged because the signature was always
an action.

**`ssh-core-nv`**, which the README recommends taking and which is the
strongest such case this lane found: seven modules and about three
thousand lines of `[]` against a hundred lines of pumping, with four
consumers that cannot take this package.
