# Changelog

All notable changes to ssh-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

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

### Design notes

Recorded here because the 0.0.2 README no longer carries them.

**The `ssh-core-nv` split, and the recommendation to take it.** Seven
modules out of eight declare no effects — about three thousand lines of
framing, key schedule, host-key policy, key parsing, authentication and
channel arithmetic — against `sshconn`, which is a hundred lines of
moving bytes between a socket and a state machine.  Four consumers want
the pure half and cannot take this package: a test and a fuzzer, which
want the packet layer at a million cases a second; a client over
something that is not a TCP socket, such as a serial line, a WebSocket,
a QUIC stream or a proxy command; an implementation of the server side,
which shares the packet layer, the key schedule and the channel
arithmetic and has no use for `std.net`; and a device fetching firmware
over SSH.  The split is clean because it was designed to be: `sshconn`
depends on the other seven and nothing depends on it.  Two notes for
whoever takes it.  `SshConn` holds a `TcpStream` and would stay behind,
while `SshAction` names `SshChannelEvent` and `SshHostVerdict` and would
move with the core, so the action enum and the pump end up in different
packages — which works because the enum is the contract.  And no
embedded claim is made: `sshkey` and `sshhostkey` name `Str` throughout,
so a device claim would cover the packet layer and the key schedule only
and wants measuring before it is written down.

**Why randomness is an argument in three places.** SSH needs random
bytes for the `KEXINIT` cookie, for the ephemeral X25519 secret and for
the padding of every single packet.  All three are parameters:
`sshkex.kex_init(cookie)`, `sshkex.kex_ecdh_init` with the secret the
caller's, and `sshpkt.write_into(..., padding, ...)`.  The padding is
the interesting one, because it is drawn per packet and taking it as an
argument is more work for the caller than a hidden effect would have
been.  It buys two things: the whole handshake becomes reproducible, so
a recorded exchange produces identical bytes every run, and a caller
whose randomness comes from a hardware source is an ordinary case rather
than a fork.  The pattern is worth naming — a module with no effects
takes its randomness as an argument, and the host's effect row stays
narrow.

**What `novo pkg add` would call.** The dependency driver resolves a git
dependency by shelling out to `git ls-remote --tags` and
`git clone --depth 1`.  For an `ssh://` or `git@host:owner/repo` URL
that starts `git`, which starts `ssh`, which reads the user's
`~/.ssh/config`, their agent socket and their keys.  With this package
it would be `sshconn.dial`, then
`sshconn.exec(c, "git-upload-pack 'owner/repo.git'")`, then the version
2 `ls-refs` and `fetch` exchanges read off the channel, with git-nv's
pack reader over what comes back, and `sshhostkey.verify` against the
user's own `known_hosts`.  Three things that port would have to decide:
where the key comes from, which for a developer machine is `ssh-agent`
and therefore an `ssh-agent-nv` row; that nothing reads `~/.ssh/config`,
so `ProxyJump` and `IdentityFile` would have to be the driver's; and
that the pack protocol's request and response framing is git-nv's row
rather than this package's.

**What changes from the reference implementations.** `russh` for the
shape and `paramiko` for the surface, with OpenSSH as the
implementation on the other end.  Four things differ.  `russh` owns a
task per connection and `paramiko` a thread; here `sshconn.poll` answers
an action and the caller's loop decides.  Both references hide the
window — `paramiko`'s `Channel.send` blocks until there is room and
`russh`'s returns a future — so a reader of either never learns the
window exists; here `may_send` is the first thing a caller meets.
`paramiko` reads `~/.ssh/id_rsa` by default and `russh` takes a key
pair; here the signature is an action, so an agent and a hardware token
are the ordinary cases.  And `paramiko`'s automatic host-key policy is
one line away from the default; here there is no policy object at all
and `sshconn.accepting_unknown_hosts` is a named function.
