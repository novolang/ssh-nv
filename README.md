# ssh-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

An SSH **client** (RFC 4251 to 4254), written in novo-lang as a sans-IO
state machine: the binary packet protocol, curve25519-sha256 key
exchange, the ChaCha20-Poly1305 cipher whose packet length is itself
encrypted, Ed25519 host keys checked against a `known_hosts` the caller
loads, `publickey` and `password` authentication with signing as an
ACTION rather than a function, the OpenSSH private key format parsed
with no filesystem in sight, and channels with the flow-control window
that decides whether a large transfer finishes or hangs.

Client only.  There is no server, no port forwarding, no agent
forwarding and no SFTP protocol — the section at the bottom says why for
each.

## Adding it, and checking it

```bash
novo pkg add ssh-nv            # into your novo.toml
novo pkg build                 # type- and effect-check the package
novo test --isolate tests/sshwire_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: ssh-nv.<module>.<fn>`.  They turn
green one at a time as bodies land.

## The one example that will work

```novo
use sshchan
use sshconn
use sshhostkey

// Run one command on a server whose key is in a known_hosts the caller
// loaded, and answer the channel it is running on.
//
// The signature over the authentication request is an ACTION the caller
// performs — see "The load-bearing interface".  So is every random
// draw.
fn run(host: Str, user: Str, hosts: Str, command: Str) -> Result<Int, SshFault> [io, net]
    let cfg = sshconn.config(user, sshhostkey.parse_known_hosts(hosts))
    let c = sshconn.dial(host, 22, cfg)!
    sshconn.exec(c, command)
```

## The layer, and why

`host`, and seven of the eight modules declare nothing.

| module | row | why |
| --- | --- | --- |
| `sshfault` | `[]` throughout | the faults and the disconnect codes |
| `sshpkt` | `[]` throughout | the binary packet protocol and four wire encodings |
| `sshkex` | `[]` throughout | negotiation, the exchange hash, the six keys |
| `sshhostkey` | `[]` throughout | `known_hosts` and the verdict |
| `sshkey` | `[]` throughout | the OpenSSH private key container |
| `sshauth` | `[]` throughout | the authentication exchange, signature excluded |
| `sshchan` | `[]` throughout | channels and the window |
| `sshconn` | `[io, net]` | the socket, and about a hundred lines of pumping |

**`[io, net]` and nothing else.**  No `[time]`: a deadline belongs to
the caller's socket, and a library that owned one would own the
caller's cancellation too.  No `[rand]`, though SSH needs randomness in
three places — see "Where a row wanted to widen".

`layer = "host"` and **not** `layer = "core"` with `host_modules`, for
the same reason tls-nv gave: the package's subject is a connection.  The
next section but two is the argument for the split, and it is the
strongest one this lane makes.

## The load-bearing interface

**The flow-control window**, and the reason it is load-bearing is that
getting it wrong **does not produce an error**.

Every SSH channel has a window in each direction.  A sender may not send
more bytes than the window allows; when it runs out it must wait for the
peer to send `CHANNEL_WINDOW_ADJUST`.  If the peer never does — because
the client forgot to adjust as it consumed data — **the connection
simply stops**.  No message, no timeout, no disconnect, nothing in any
log.

The shape that produces is unmistakable once you have seen it and
baffling before:

- `ssh host cat small.txt` works.
- `ssh host cat big.log` hangs after about two megabytes, which is the
  initial window.
- A command whose output is mostly **stderr** hangs too, because
  extended data counts against the **same** window and a client that
  only adjusted on `CHANNEL_DATA` runs it down without noticing.
- And a client that adjusts at **zero** rather than at half transfers at
  one window per round-trip time — about a tenth of the speed it should
  be on a transatlantic link, slow and never obviously broken.

So `sshchan` makes the window impossible to ignore, and four functions
carry it:

| | |
| --- | --- |
| `sshchan.may_send` | how many bytes may go out RIGHT NOW: the only correct answer to "can I write this" |
| `sshchan.consumed` | what the caller calls as it drains, and it takes extended data too |
| `sshchan.needs_adjust` | true at half the window, for the reason above |
| `SshFaultWindowExhausted` | what this package answers instead of blocking forever — the one thing the protocol does not provide |

`sshchan.apply_event` applies the arithmetic for both kinds of data,
which is the whole reason to call it rather than to update the fields by
hand.

The second decision is **`SshActionSign`: the signature is an action**.
A developer's SSH key is usually not in a file this process can read —
it is in `ssh-agent`, on a hardware token behind a PIN, or in a keychain
that will do a signature and will not export a secret.  A library whose
authentication took a key would work for one of those cases and be
unusable for the others, and the one it works for is the least common.
So `sshauth` never signs: `publickey_signed_data` builds the bytes,
the host signs them however it can, and `publickey_signed` puts the
result back in.  `sshkey.sign` is there for a caller whose key really is
in memory, and it is one function rather than the shape of the API.

Two smaller rules follow from the same discipline, and both are in the
tests:

- **The two-phase `publickey` query.**  A client may ask "would this key
  be accepted" without signing.  A client that skipped it signs with
  each key in turn, which is five prompts on a token and five taps on a
  YubiKey — and is what makes some clients unusable with hardware.
- **Partial success is not a failure.**  `USERAUTH_FAILURE` with the
  partial flag set means "that one worked, now do another", which is how
  every two-factor server answers; a client that read it as a refusal
  cannot log in to one at all.

## What `novo pkg add` would call

`novo pkg add` resolves a git dependency by shelling out: `git
ls-remote --tags <url>` to find the tag a version requirement names, and
`git clone --depth 1` to fetch it.  For an `https://` URL that is fine.
For a `git@github.com:owner/repo.git` URL it means the toolchain starts
`git`, which starts `ssh`, which reads the user's `~/.ssh/config`, their
agent socket and their keys — three processes and a configuration
language, to fetch a tarball's worth of source.

This package is what that becomes without a subprocess:

| what the driver does today | what it would call |
| --- | --- |
| `git ls-remote --tags git@host:owner/repo` | `sshconn.dial`, then `sshconn.exec(c, "git-upload-pack 'owner/repo.git'")`, then read the v2 `ls-refs` response off the channel |
| `git clone --depth 1` | the same channel, with `fetch` and `deepen 1`, and git-nv's `GitPackReader` over the pack that comes back |
| the host key check | `sshhostkey.verify` against the user's own `known_hosts`, which the driver reads |
| the key | `SshActionSign` — and the answer for most developers is an `ssh-agent` client, which is the one case a file-reading library could not serve |

Three things that port would have to decide, and they are why this is a
design note rather than a patch.

**Where the key comes from.**  The honest answer for a developer machine
is `ssh-agent`, which is its own protocol over a Unix socket named by
`SSH_AUTH_SOCK`.  That is an `ssh-agent-nv` row and not this package;
what this package does is make it a drop-in, because the signature was
always an action.

**`~/.ssh/config` is not read by anybody.**  `Host` aliases,
`ProxyJump`, `IdentityFile` and `User` are what make `git@github.com`
work on a machine where GitHub is reached through a bastion.  This
package reads no configuration at all, so the driver would have to —
another row, and one worth naming before somebody assumes it is here.

**The pack protocol is git-nv's.**  `git-nv` publishes a pack reader and
no wire protocol; what travels over this channel is
`git-upload-pack`'s v2 request and response framing, which is a row on
that package rather than on this one.

## `ssh-core-nv` should be a row, and this is the strongest case

**Take it.**

Seven modules out of eight are `[]` — about three thousand lines of
framing, key schedule, host-key policy, key parsing, authentication and
channel arithmetic — against `sshconn`, which is a hundred lines of
moving bytes between a socket and a state machine.  That is not a
package with a pure corner; it is a `core` package with a socket adapter
stapled to it, and more so than tls-nv, which made the same argument at
six modules out of seven.

Four consumers want the `core` half and cannot have this package:

- **A test and a fuzzer.**  Every rule above is testable against a
  recorded exchange with no socket and no randomness, which is exactly
  what `tests/sshwire_tests.nv` does — and what a fuzzer would want to
  do against the packet layer at a million cases a second.
- **A client over something that is not a TCP socket.**  SSH over a
  serial line, over a WebSocket, over a QUIC stream, or through a
  `ProxyCommand` — all of them want the protocol and none of them wants
  `std.net`.
- **An implementation of the SERVER side.**  The packet layer, the key
  schedule and the channel arithmetic are shared; only the handshake's
  direction differs.  A server built on this package would take a
  `std.net` dependency it has no use for.
- **A device.**  Less obviously than for mdns-nv, but an embedded client
  fetching firmware over SSH is a real thing, and it is `[]` modules or
  nothing.

The split is clean because it was designed to be: `sshconn` depends on
the other seven and nothing depends on it.  `ssh-core-nv` is `sshfault`,
`sshpkt`, `sshkex`, `sshhostkey`, `sshkey`, `sshauth` and `sshchan`
unchanged, and `ssh-nv` keeps `sshconn` and takes a dependency on it.

Two notes for whoever takes it.  `SshConn` holds a `TcpStream` and would
stay behind; `SshAction` names `SshChannelEvent` and `SshHostVerdict`
and would move with the core, which means the action enum and the pump
end up in different packages — the same arrangement tls-nv's `TlsAction`
and `tlsconn` have, and it works because the enum is the contract.  And
no `@tier(embedded)` claim is made here: `sshkey` and `sshhostkey` name
`Str` throughout, so a device claim would cover the packet layer and the
key schedule only and wants to be measured before it is written down.

## What is missing, by name

**RSA, and it is the one that matters.**  A large share of deployed SSH
servers still present an `ssh-rsa` or `rsa-sha2-256` host key, and a
large share of deployed user keys are RSA.  Nothing on this grid
implements RSA: crypto-nv 0.1.2 has no public-key arithmetic, jwt-nv
declares a key type over no implementation, and tls-nv and dkim-nv have
both named the same gap.  The row is **`rsa-nv`**, crypto/core over
bigint-nv — PKCS#1 v1.5 signing and verification at minimum.  Until it
lands, `sshkex.can_complete` answers a fault naming the algorithm before
the exchange starts, rather than failing three messages later.

**ECDSA over P-256, for `ecdsa-sha2-nistp256` host keys.**  p256-nv
0.0.2 is ECDH-only; this is the same row tls-nv and acme-nv name from
their own sides.

**`bcrypt_pbkdf` and AES-256-CTR, for an encrypted private key.**  An
OpenSSH key with a passphrase is `aes256-ctr` over a key derived by
`bcrypt_pbkdf`, and neither is on this grid — crypto-nv has no block
cipher at all, which tls-nv has already named.  `sshkey.is_encrypted`
answers before anything is attempted and `SshFaultKeyEncrypted` names
the KDF, so a caller can say "this key has a passphrase and this client
cannot use it" rather than "bad key".

**`ssh-agent`, which is a protocol and not a primitive.**  A Unix socket
named by `SSH_AUTH_SOCK`, a request and a signature back.  It is the
answer for most real callers of this package and it is a row of its own:
`ssh-agent-nv`, `host`/`networking`, and small.

**SHA-1, for the hashed `known_hosts` form.**  OpenSSH writes hashed
hostnames as HMAC-SHA1, and crypto-nv publishes `hmac_sha1`, so this
package can READ that form.  It does not WRITE a new one: emitting SHA-1
is a choice, and a library that made it quietly would be adding SHA-1 to
a program that had none.  `sshhostkey.append_line` writes the plain
form.

## Where a row wanted to widen

**`[rand]`, in three places, and the design answered by not taking it.**
SSH needs randomness for the KEXINIT cookie, for the ephemeral X25519
secret, and for the padding of **every single packet**.  All three are
arguments: `sshkex.kex_init(cookie)`, `sshkex.kex_ecdh_init(public)` with
the secret the caller's, and `sshpkt.write_into(..., padding, ...)`.

The third is the interesting one.  Padding is drawn per packet, so a
library that took it as an argument is a library whose caller has to
draw random bytes in its send path — which is more work than a hidden
`[rand]` would have been.  It is still the right trade, for two reasons:
the whole handshake becomes reproducible, so a recorded exchange is a
test that produces identical bytes every run; and a caller whose
randomness comes from a hardware source, or who is on a platform where
`std.rand` is not what they want, is an ordinary case rather than a
fork.  `SshActionWantRandom` carries a `purpose` string so a caller can
tell the three apart.

That is the cohort's **fourth** instance of this pattern — tls-nv's
handshake randomness, ntp-nv's nonce, mdns-nv's three jitters, and these
three — and it is worth naming as a pattern rather than rediscovering it
per package: **a `core`-shaped module takes its randomness as an
argument, and the host's row stays narrow.**

**No effect parameter was wanted at all.**  Unlike tls-nv, which wanted
two and could have one, this package's pump is a concrete `TcpStream`
and its signature checker is an action rather than a trait.  That is a
consequence of `SshActionSign`: because signing is an action, there is
no trait to be polymorphic over, and the one-parameter limit never comes
up.  Worth recording as evidence that the action-enum shape avoids the
limit that the trait shape runs into.

**`[io]` beside `[net]`, which the plan's row already said.**
`std.net`'s own `Read`, `Write` and `Close` impls for `TcpStream`
declare `[io, net]`, so a package that reads a socket through them
carries both.  Nothing here widened past what the plan wrote.

## What this does not do, on purpose

- **No server.**  The row is client-only, and the packet layer and key
  schedule are the same on both sides — which is an argument for
  `ssh-core-nv` rather than for putting a server here.
- **No port forwarding, local or remote.**  `direct-tcpip` and
  `tcpip-forward` are channels like any other and the machinery is
  here; what is not here is the listener, the connection table and the
  policy about what a remote peer may ask to reach, which is a program's
  decision and not a library's.
- **No agent forwarding**, which is port forwarding plus handing a
  remote host the ability to sign with your key.  Out on purpose and
  worth saying why: it is the SSH feature most often enabled by people
  who would not enable it if they had read what it does.
- **No SFTP protocol.**  `sshconn.subsystem(c, "sftp")` opens the
  channel and hands back a number; SFTP is its own packet format and its
  own row.  Declared and scoped, which is the useful half.
- **No `~/.ssh/config`.**  Named above as a row, because a driver that
  needs `ProxyJump` needs it and this package reads no configuration at
  all.
- **No `keyboard-interactive`.**  It is a prompt loop the library would
  have to drive, and the row for it is a caller-supplied prompt callback
  this interface does not have.  `sshauth.method_supported` answers
  false so a client can say what it cannot do.
- **No compression.**  `zlib@openssh.com` compresses after
  authentication; flate-nv is on the grid and could be a later row, but
  a compressor in a security protocol is a decision with its own
  literature and not one to take by default.
- **No `ssh-rsa` with SHA-1**, ever, even when RSA lands: it is
  deprecated and most servers built this decade refuse it.
- **It does not print.**  Every failure is a value with a `message()`,
  and `SshActionBanner` hands the server's banner to the caller rather
  than writing it to a terminal the library does not own.

## The reference implementation

`russh` for the shape and `paramiko` for the surface; OpenSSH is the
reference implementation in the sense that matters, which is that it is
what the other end is.  RFC 4251 through 4254 are the specifications,
and `chacha20-poly1305@openssh.com` is OpenSSH's own extension with its
own document.

Four things change in the port.

`russh` is async and owns a task per connection; `paramiko` owns a
thread. Here there is neither: `sshconn.poll` answers an action and the
caller's loop decides, for the reasons the `ssh-core-nv` section gives.

Both references hide the window.  `paramiko`'s `Channel.send` blocks
until there is room and `russh`'s returns a future; both are correct and
both mean that a reader of the code never learns the window exists.
Here `may_send` is the first thing a caller meets, because the failure
it prevents has no error message.

`paramiko` reads `~/.ssh/id_rsa` by default and `russh` takes a
`KeyPair`.  Here the signature is an action, so an agent and a hardware
token are the ordinary cases rather than the awkward ones.

And `paramiko`'s `AutoAddPolicy` is one line away from the default.
Here there is no policy object at all: `sshhostkey.verify` answers a
verdict, `SshHostUnknown` is a fault by default, and
`sshconn.accepting_unknown_hosts` is a named function so that a reviewer
grepping for it finds every place a program decided to trust a host it
had never seen.

## Status

| item | implemented |
| --- | --- |
| `sshfault` — `SshDisconnect`, `SshFault` | types only |
| `sshfault.disconnect_of_code`, `.disconnect_code`, `.retry_would_help`, `.is_configuration`, the `message` impl | no |
| `sshpkt` — `SshMsg`, `SshPacket`, `SshReader`, `SshWriter`, `SshReadStep` | types only |
| `sshpkt.SSH_PACKET_MIN`, `.SSH_PACKET_MAX`, `.SSH_PADDING_MIN`, `.SSH_LENGTH_BYTES`, `.SSH_REKEY_BYTES` | yes — they are constants |
| `sshpkt.reader`, `.writer`, `.feed`, `.step`, `.take`, `.pending_len` | no |
| `sshpkt.write_into`, `.framed_len`, `.padding_len`, `.wrote`, `.needs_rekey`, `.rekeyed_reader`, `.rekeyed_writer` | no |
| `sshpkt.msg_of_number`, `.msg_number`, `.is_method_specific` | no |
| `sshpkt.write_string`, `.read_string`, `.write_u32`, `.read_u32`, `.write_mpint`, `.mpint_len` | no |
| `sshpkt.write_name_list`, `.read_name_list`, `.write_byte`, `.write_bool` | no |
| `sshkex` — `SshKexInit`, `SshAgreed`, `SshKexResult`, `SshKeys` | types only |
| `sshkex.SSH_VERSION_PREFIX`, `.SSH_KEX_CURVE25519`, `.SSH_KEX_CURVE25519_LIBSSH`, `.SSH_CIPHER_CHACHA20`, `.SSH_HOSTKEY_ED25519`, `.SSH_CHACHA_KEY_BYTES` | yes — they are constants |
| `sshkex.kex_init`, `.write_kex_init`, `.read_kex_init`, `.negotiate`, `.can_complete` | no |
| `sshkex.exchange_hash_input`, `.exchange_hash`, `.version_for_hash`, `.version_acceptable` | no |
| `sshkex.derive_keys`, `.cipher_keys` | no |
| `sshkex.kex_ecdh_init`, `.read_kex_ecdh_reply`, `.shared_secret`, `.kex_result`, `.verify_exchange_signature` | no |
| `sshhostkey` — `SshKnownHost`, `SshKnownHosts`, `SshHostVerdict` | types only |
| `sshhostkey.SSH_FINGERPRINT_PREFIX` | yes — it is a constant |
| `sshhostkey.parse_known_hosts`, `.unreadable_lines`, `.empty_known_hosts`, `.verify`, `.fault_of` | no |
| `sshhostkey.host_matches`, `.is_hashed`, `.hashed_matches` | no |
| `sshhostkey.fingerprint`, `.legacy_fingerprint`, `.algorithm_of`, `.append_line`, `.with_host`, `.supported_algorithms` | no |
| `sshkey` — `SshPrivateKey`, `SshKeyInfo` | types only |
| `sshkey.SSH_PRIVATE_KEY_BEGIN`, `.SSH_PRIVATE_KEY_END`, `.SSH_KEY_MAGIC`, `.SSH_KEY_CIPHER_NONE` | yes — they are constants |
| `sshkey.is_private_key_file`, `.key_info`, `.is_encrypted`, `.parse_private_key`, `.parse_private_key_at`, `.check_words_match` | no |
| `sshkey.parse_public_key`, `.public_key_line`, `.public_of`, `.sign`, `.signature_blob`, `.read_signature_blob`, `.supported_algorithms` | no |
| `sshauth` — `SshAuthReply`, `SshAuth` | types only |
| `sshauth.SSH_SERVICE_USERAUTH`, `.SSH_SERVICE_CONNECTION`, `.SSH_AUTH_NONE`, `.SSH_AUTH_PUBLICKEY`, `.SSH_AUTH_PASSWORD` | yes — they are constants |
| `sshauth.auth`, `.service_request`, `.none_request`, `.publickey_query`, `.publickey_signed_data`, `.publickey_signed` | no |
| `sshauth.password_request`, `.password_change_request`, `.read_reply`, `.after_reply` | no |
| `sshauth.method_supported`, `.attemptable`, `.is_done`, `.fault_of` | no |
| `sshchan` — `SshChannel`, `SshChannelState`, `SshChannelEvent`, `SshChannelPurpose`, `SshPty` | types only |
| `sshchan.SSH_WINDOW_DEFAULT`, `.SSH_MAX_PACKET_DEFAULT`, `.SSH_WINDOW_ADJUST_AT`, `.SSH_EXTENDED_STDERR` | yes — they are constants |
| `sshchan.open_session`, `.read_open_confirmation`, `.read_open_failure` | no |
| `sshchan.exec_request`, `.shell_request`, `.pty_request`, `.subsystem_request`, `.window_change_request`, `.env_request` | no |
| `sshchan.may_send`, `.data_message`, `.sent`, `.consumed`, `.needs_adjust`, `.adjust_message` | no |
| `sshchan.read_event`, `.apply_event`, `.eof_message`, `.close_message` | no |
| `sshchan.is_finished`, `.can_send`, `.can_receive`, `.pty`, `.exit_status_of` | no |
| `sshconn` — `SshConn`, `SshAction`, `SshConfig` | types only |
| `sshconn.config`, `.accepting_unknown_hosts`, `.dial`, `.attach`, `.poll` | no |
| `sshconn.supply_random`, `.supply_host_verdict`, `.supply_signature`, `.supply_password`, `.offer_key` | no |
| `sshconn.exec`, `.shell`, `.subsystem`, `.write`, `.send_eof`, `.close_channel`, `.disconnect` | no |
| `sshconn.channel_of`, `.session_id_of`, `.agreed_of`, `.stream_of` | no |
