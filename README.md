# ssh-nv

The Secure Shell protocol runs an encrypted, authenticated session over
an insecure network. Its architecture is
[RFC 4251](https://www.rfc-editor.org/rfc/rfc4251), its transport layer
[RFC 4253](https://www.rfc-editor.org/rfc/rfc4253), its user
authentication [RFC 4252](https://www.rfc-editor.org/rfc/rfc4252) and
its connection protocol
[RFC 4254](https://www.rfc-editor.org/rfc/rfc4254). This package is an
SSH client for novo-lang. The protocol is a state machine that performs
no input or output: it answers what the caller should do next, and the
caller reads the socket, draws the random bytes and produces the
signatures. It is built on
[x25519-nv](https://novo-lang.org/packages/x25519-nv) for the key
exchange, [ed25519-nv](https://novo-lang.org/packages/ed25519-nv) for
host keys and user keys,
[chacha20-nv](https://novo-lang.org/packages/chacha20-nv) for the
cipher, [crypto-nv](https://novo-lang.org/packages/crypto-nv) for
SHA-256 and HMAC, and
[base64-nv](https://novo-lang.org/packages/base64-nv) for the key files.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What SSH is

An SSH connection is built in three layers, and each has its own
document. The transport layer opens the connection, agrees the
algorithms, authenticates the server and encrypts everything after that.
The user authentication protocol runs over it and proves who the client
is. The connection protocol runs over that and multiplexes the session
into channels.

Both sides begin by sending a version line, `SSH-2.0-` and a software
name. Everything after it is a **binary packet**: a length, a padding
length, the payload and at least four bytes of random padding, encrypted
as a whole. The **sequence number** is how many packets a side has sent
or received. It is never transmitted, and both ends count it into the
nonce and the message authentication code.

The **key exchange** agrees one algorithm from each of eight lists and
produces a shared secret. This package implements `curve25519-sha256`,
which is an X25519 exchange of ephemeral public keys. Both sides then
compute the **exchange hash**, a SHA-256 over eight fields: the two
version strings, the two `KEXINIT` payloads, the server's **host key**,
the two ephemeral public keys and the shared secret. The server signs
that hash with its host key, which is how the client knows it is talking
to the server it meant. The hash of the *first* exchange becomes the
**session id**, and it never changes afterwards.

The client decides whether to trust the host key by looking it up in
`known_hosts`, a text file of host patterns and keys that the caller
loads and this package reads. Six keys are then derived from the shared
secret and the exchange hash, two per direction: an initialisation
vector, an encryption key and an integrity key each way. This package's
cipher is `chacha20-poly1305@openssh.com`, which encrypts the packet's
length field under a second key, so a reader cannot know how many bytes
a packet holds until it has decrypted them.

Authentication is a sequence of requests, each naming a method. This
package implements `publickey` and `password`. A **channel** is then
opened for the work: a command, an interactive shell, or a **subsystem**
named by string. Each channel has a **flow-control window** in each
direction, which is how many bytes the peer will accept before it sends
a `CHANNEL_WINDOW_ADJUST` message granting more.

| Fact | Value |
| --- | --- |
| Key exchange | `curve25519-sha256`, and `curve25519-sha256@libssh.org`, which is byte-identical |
| Host key algorithm | `ssh-ed25519` |
| Cipher | `chacha20-poly1305@openssh.com`, 64 bytes of key material |
| Smallest packet | 16 bytes |
| Largest packet an implementation must accept | 35000 bytes (RFC 4253 section 6.1) |
| Smallest padding | 4 bytes |
| Bytes before a rekey is due | 1 GiB (RFC 4253 section 9) |
| Initial channel window | 2 MiB |
| Largest channel packet | 32 KiB |
| A window adjustment is owed at | half the window consumed |
| Extended data type for stderr | 1 |

## Install

```
novo pkg add ssh-nv
```

## Example

```novo
use sshconn
use sshhostkey

fn main() [io, net]
    // The known_hosts the caller loaded. An empty set means every host
    // is unknown, and an unknown host is refused unless the caller has
    // said otherwise.
    let hosts = sshhostkey.parse_known_hosts("")
    let cfg = sshconn.config("alice", hosts)

    // Open the socket and send the version line. Nothing else has
    // happened yet.
    match sshconn.dial("example.com", 22, cfg)
        Err(e) => println("could not connect: ${e.message()}")
        Ok(c) =>
            // The caller pumps the handshake. Every action is something
            // only the caller can do.
            match sshconn.poll(c)
                Err(e) => println("the handshake failed: ${e.message()}")
                Ok(action) =>
                    match action
                        SshActionWantBytes(least) =>
                            println("read at least ${least} bytes from the socket")
                        SshActionWantRandom(count, purpose) =>
                            println("draw ${count} random bytes for the ${purpose}")
                        SshActionCheckHostKey(algorithm, key, fingerprint) =>
                            println("the server offers ${algorithm} ${fingerprint}")
                        SshActionReady =>
                            println("authenticated; a channel may be opened")
                        _ =>
                            println("some other step of the handshake")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: ssh-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `sshfault` | Everything that can go wrong as a value, and RFC 4253 section 11.1's disconnect reason codes. |
| `sshpkt` | The binary packet protocol: the reader, the writer, the sequence numbers, the rekey rule, and the four wire encodings everything else is written in. |
| `sshkex` | The algorithm negotiation, the exchange hash, the shared secret, the session id and the six derived keys. |
| `sshhostkey` | `known_hosts` as a value, the verdict for a host and a key, and the fingerprint forms a person reads. |
| `sshkey` | The OpenSSH private key container, public key lines, and signing with a key held in memory. |
| `sshauth` | The authentication exchange, with the signature left to the caller. |
| `sshchan` | Channels, the requests that start work on one, and the flow-control window. |
| `sshconn` | The socket, and the pump: an action the caller performs and a function that supplies the answer. |

## How to choose an entry point

**`sshconn` is the whole client.** It opens a TCP connection, drives the
handshake and the authentication, and opens channels. `poll` answers one
`SshAction`, the caller performs it, and a `supply_*` function hands the
answer back. This is the only module that reads or writes anything.

**The other seven modules are the protocol without the socket.** A
program that speaks SSH over something other than TCP — a serial line, a
tunnel, a proxy command — takes them and writes its own pump. A test and
a fuzzer do the same. The packet layer, the key schedule and the channel
arithmetic are functions from bytes to bytes.

**`sshkey.sign` is for a key that really is in memory.** Most keys are
not: they are in an agent, on a hardware token behind a personal
identification number, or in a keychain that will produce a signature
and will not export a secret. For those, answer `SshActionSign` with
`sshconn.supply_signature`.

## The rules a user needs

1. **The flow-control window is the rule that has no error message.**
   RFC 4254 section 5.2. A sender may not send more bytes than the
   window allows, and when it runs out it must wait for the peer to
   adjust it. If the peer never does, the connection stops: no message,
   no timeout, nothing in a log. `sshchan.may_send` answers how many
   bytes may go out now, and it is the only correct answer to "can I
   write this". `sshchan.data_message` answers
   `SshFaultWindowExhausted` rather than blocking.
2. **Extended data counts against the same window.** RFC 4254 section
   5.2. A command whose output is mostly standard error runs the window
   down exactly as one writing to standard output does.
   `sshchan.consumed` takes the bytes whatever their kind, and
   `apply_event` applies the arithmetic for both.
3. **An adjustment is owed at half the window, not at zero.** An
   adjustment sent at zero arrives after the peer has already stopped,
   so every window's worth of data costs a round trip.
   `sshchan.needs_adjust` is true once half is consumed.
4. **End of file is not a close.** RFC 4254 section 5.3. A command that
   reads its standard input to the end waits forever without
   `CHANNEL_EOF`, and that hang looks exactly like the window one.
   `eof_message` sends it and the channel may still receive.
5. **The exit status arrives before the close, and a signal is a
   different message.** RFC 4254 section 6.10. `exit-status` is a
   channel request; a client that waited for `CHANNEL_CLOSE` to read it
   gets nothing. `exit-signal` is sent instead when the process was
   killed, and a client that read only the first reports success for it.
   `exit_status_of` answers `None` for a channel that reported neither.
6. **The two channel numbers are independent.** RFC 4254 section 5.1.
   Each side allocates its own, a message carries the recipient's, and a
   client that used one number for both works against a server that
   happens to agree. A channel is finished only when both sides have
   sent `CHANNEL_CLOSE`; `is_finished` is that check, and freeing the
   number earlier gives the next channel the previous one's data.
7. **The packet length is encrypted, so framing needs the key.**
   `chacha20-poly1305@openssh.com` encrypts the four-byte length field
   under a second key. `SshReader` is therefore fed with `feed` and
   asked with `step`, which takes the key material, rather than handed a
   buffer to parse.
8. **The sequence number is counted, never sent, and does not reset on a
   rekey.** RFC 4253 section 6.4. Both sides count from zero into the
   nonce and the authentication tag. A client that reset the count at a
   rekey desynchronises, and the symptom is a bad tag on the first
   packet afterwards. `needs_rekey` is true after a gibibyte, which is
   RFC 4253 section 9's rule and is what stops a nonce being reused.
9. **The padding is the caller's random bytes.** RFC 4253 section 6: at
   least four, and the total must be a multiple of the cipher's block
   size. `sshpkt.padding_len` answers how many are needed and
   `write_into` takes them as an argument. The padding is what hides the
   length of a password from an observer.
10. **An `mpint` gets one leading zero when its top bit is set.**
    RFC 4251 section 5. A shared secret whose first byte is above `0x7F`
    needs it, and a client that omits it computes an exchange hash that
    differs from the server's about half the time.
    `sshpkt.write_mpint` is that encoding.
11. **Negotiation is first match in the client's list.** RFC 4253
    section 7.1. The chosen algorithm is the first on the client's list
    that also appears on the server's. Scanning the server's list
    instead lets the server choose. `sshkex.can_complete` then refuses
    an agreed algorithm this package has no primitive for, before the
    exchange starts rather than three messages later.
12. **The exchange hash is eight fields in one order, and two of them
    are bytes that arrived.** RFC 4253 section 8. The `KEXINIT` payloads
    are hashed exactly as they were received, so a client that
    re-serialised the server's gets a different hash. A version string
    contributes without its trailing carriage return and line feed and
    without the optional comment after the first space. Getting one
    field wrong produces a bad-signature failure that points at the
    signature code. `sshkex.exchange_hash_input` builds the buffer on
    its own so it can be checked on its own.
13. **The first exchange hash is the session id forever.** RFC 4253
    section 7.2. It does not change at a rekey, every `publickey`
    signature is over it, and that is what stops a signature captured on
    one connection being replayed on another.
14. **The key derivation extends past one digest.** RFC 4253 section
    7.2: more key material is produced by re-hashing the secret, the
    hash and everything derived so far. `chacha20-poly1305@openssh.com`
    wants 64 bytes from a 32-byte hash, so the extension is exercised on
    every connection. The two directions never share key material, and
    `cipher_keys` lays the pair out payload key first, which is
    OpenSSH's order and not the one the name suggests.
15. **An all-zero shared secret is a fault and not a key.** The peer
    sent a low-order point, and continuing gives a connection whose key
    the peer chose. `sshkex.shared_secret` answers a fault.
16. **"Unknown host" and "the host key changed" are different
    situations.** Unknown is a first connection. Changed is a rebuilt
    server or the attack `known_hosts` exists to catch, and a `@revoked`
    marker is neither: it is terminal. There is no default that accepts
    an unknown host; `sshconn.accepting_unknown_hosts` is a named
    function so that every place a program trusts a host it has never
    seen can be found by searching for it. The port is part of an entry:
    `[example.com]:2222` and `example.com` are different lines. A line
    this package cannot read is skipped and counted, never fatal to the
    file.
17. **A public key is queried before it is signed.** RFC 4252 section 7
    allows a `publickey` request with no signature, meaning "would this
    key be accepted". A client that skips the query signs with each key
    in turn, which is one prompt per key on a hardware token.
18. **Partial success is not a failure.** RFC 4252 section 5.1. A
    failure message with the partial flag set means the method worked
    and another is required, which is how a two-factor server answers.
    The methods it names are what remains, not what to try instead.
19. **Message number 60 means two things, and the method in flight
    decides which.** Under `publickey` it is "the key would be
    accepted"; under `password` it is a password-change request.
    `SshAuth.in_flight` is what `read_reply` consults.
20. **A signature on the wire carries its algorithm name.** It is two
    length-prefixed strings, not the raw 64 bytes.
    `sshkey.signature_blob` builds one, and `supply_signature` expects
    one.
21. **Version 1 is refused by name.** `SSH-2.0-` and `SSH-1.99-` are
    accepted and `SSH-1.5-` is not, because version 1 is a different
    protocol with known breaks and a client that fell back to it could
    be downgraded into one.

## What is not included

- **A server.** The packet layer, the key schedule and the channel
  arithmetic are the same on both sides, but the handshake here runs in
  the client's direction only.
- **RSA, and it is the one that matters.** A large share of deployed
  servers present an `ssh-rsa` or `rsa-sha2-256` host key, and a large
  share of user keys are RSA. No package on the registry implements RSA
  yet. `sshkex.can_complete` names the algorithm before the exchange
  starts rather than failing later. `ssh-rsa` with SHA-1 will not be
  added even when RSA lands: it is deprecated and most servers refuse
  it.
- **ECDSA over the NIST P-256 curve**, for `ecdsa-sha2-nistp256` host
  keys. [p256-nv](https://novo-lang.org/packages/p256-nv) is key
  agreement only today.
- **An encrypted private key.** An OpenSSH key with a passphrase is
  `aes256-ctr` over a key derived by `bcrypt_pbkdf`, and neither is on
  the registry. `sshkey.is_encrypted` answers before anything is
  attempted and `SshFaultKeyEncrypted` names the derivation function, so
  a caller can say "this key has a passphrase and this client cannot use
  it" rather than "bad key".
- **Writing a hashed `known_hosts` line.** OpenSSH writes hashed host
  names as HMAC-SHA1. This package reads that form, and `append_line`
  writes the plain one: emitting SHA-1 is a choice, and a library should
  not make it on a program's behalf.
- **An `ssh-agent` client.** The agent is its own protocol over a Unix
  socket named by `SSH_AUTH_SOCK`. It drops in without a change here,
  because the signature was always an action.
- **Port forwarding and agent forwarding.** `direct-tcpip` and
  `tcpip-forward` are channels like any other and the machinery is here.
  The listener, the connection table and the policy about what a remote
  peer may reach are a program's decisions. Agent forwarding also hands
  a remote host the ability to sign with your key.
- **The SFTP protocol.** `sshconn.subsystem(c, "sftp")` opens the
  channel and answers its number. SFTP has its own packet format and is
  a package of its own.
- **`~/.ssh/config`.** `Host` aliases, `ProxyJump`, `IdentityFile` and
  `User` are what make a short host name work on a machine that reaches
  the server through a bastion. This package reads no configuration at
  all.
- **`keyboard-interactive`.** It is a prompt loop the library would have
  to drive, and this interface has no callback for one.
  `sshauth.method_supported` answers false, so a client can say what it
  cannot do.
- **Compression.** `zlib@openssh.com` compresses after authentication,
  and a compressor inside a security protocol is a decision with its own
  literature.
- **A deadline, and randomness.** There is no clock here: a timeout
  belongs to the caller's socket. Randomness is needed in three places —
  the `KEXINIT` cookie, the ephemeral X25519 secret and every packet's
  padding — and all three are arguments. `SshActionWantRandom` carries a
  purpose string that tells them apart.
- **Printing.** Every failure is a value with a `message()`, and
  `SshActionBanner` hands the server's banner to the caller.

## Related packages

- [x25519-nv](https://novo-lang.org/packages/x25519-nv) is the scalar
  multiplication `curve25519-sha256` is,
  [ed25519-nv](https://novo-lang.org/packages/ed25519-nv) verifies the
  server's signature over the exchange hash and signs a `publickey`
  request, and [chacha20-nv](https://novo-lang.org/packages/chacha20-nv)
  is the cipher and its Poly1305 tag.
- [crypto-nv](https://novo-lang.org/packages/crypto-nv) supplies
  SHA-256 for the exchange hash and the key derivation, the
  constant-time comparison the handshake needs, and the HMAC-SHA1 that
  reads a hashed `known_hosts` line.
- [base64-nv](https://novo-lang.org/packages/base64-nv) decodes the
  private key container and the `known_hosts` and `authorized_keys`
  lines, all three of which are base64 with a header around it.
- [tls-nv](https://novo-lang.org/packages/tls-nv) is the other secure
  transport on the registry. TLS authenticates a server with a
  certificate chain signed by an authority; SSH authenticates it with a
  key the client has seen before. Take TLS to reach a web service and
  this package to reach a shell.
- [git-nv](https://novo-lang.org/packages/git-nv) and
  [git-core-nv](https://novo-lang.org/packages/git-core-nv) are the
  other half of fetching a repository over SSH: this package carries the
  channel, and `git-upload-pack`'s request and response travel on it.
- [unixsock-nv](https://novo-lang.org/packages/unixsock-nv) is the
  transport an `ssh-agent` client would need, since the agent listens on
  a Unix-domain socket.
- `std.net` in the standard library is the TCP socket `sshconn` dials.
  Its own `Read`, `Write` and `Close` implementations are why this
  package declares `[io]` beside `[net]`.

## Tests

```bash
novo test tests/sshwire_tests.nv      # 17 tests: the packet layer, the encodings, the key exchange
novo test tests/sshsession_tests.nv   # 21 tests: the window, the host key, the authentication
```

No test opens a socket or draws a random number. The padding, the
`KEXINIT` cookie and the ephemeral secret are arguments, so a whole
handshake is a value the suite writes out and the same inputs produce
the same bytes every run. The suite asserts the rules above one at a
time: that framing needs the key, that the sequence number is counted
and not sent, that an `mpint` gains a leading zero when its top bit is
set, that negotiation takes the client's first match, that the exchange
hash covers the `KEXINIT` bytes that arrived, that a low-order shared
secret is a fault, that the derivation extends past one digest, that a
channel out of window stops without an error, that an adjustment is owed
at half, that extended data counts, that an unknown host and a changed
one are different verdicts, and that a public key is queried before it
is signed.

The tests compile today and fail at run, each on the
`not implemented: ssh-nv.<module>.<fn>` panic that is its body. They
turn green one at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| The `SSH_*` constants in `sshpkt`, `sshkex`, `sshhostkey`, `sshkey`, `sshauth` and `sshchan` | yes (they are constants) |
| `sshfault.disconnect_of_code`, `.disconnect_code`, `.retry_would_help`, `.is_configuration`, `.SshFault.message` | no |
| `sshpkt.reader`, `.writer`, `.feed`, `.step`, `.take`, `.pending_len` | no |
| `sshpkt.write_into`, `.framed_len`, `.padding_len`, `.wrote`, `.needs_rekey`, `.rekeyed_reader`, `.rekeyed_writer` | no |
| `sshpkt.msg_of_number`, `.msg_number`, `.is_method_specific` | no |
| `sshpkt.write_string`, `.read_string`, `.write_u32`, `.read_u32`, `.write_mpint`, `.mpint_len` | no |
| `sshpkt.write_name_list`, `.read_name_list`, `.write_byte`, `.write_bool` | no |
| `sshkex.kex_init`, `.write_kex_init`, `.read_kex_init`, `.negotiate`, `.can_complete` | no |
| `sshkex.exchange_hash_input`, `.exchange_hash`, `.version_for_hash`, `.version_acceptable` | no |
| `sshkex.derive_keys`, `.cipher_keys` | no |
| `sshkex.kex_ecdh_init`, `.read_kex_ecdh_reply`, `.shared_secret`, `.kex_result`, `.verify_exchange_signature` | no |
| `sshhostkey.parse_known_hosts`, `.unreadable_lines`, `.empty_known_hosts`, `.verify`, `.fault_of` | no |
| `sshhostkey.host_matches`, `.is_hashed`, `.hashed_matches` | no |
| `sshhostkey.fingerprint`, `.legacy_fingerprint`, `.algorithm_of`, `.append_line`, `.with_host`, `.supported_algorithms` | no |
| `sshkey.is_private_key_file`, `.key_info`, `.is_encrypted`, `.parse_private_key`, `.parse_private_key_at`, `.check_words_match` | no |
| `sshkey.parse_public_key`, `.public_key_line`, `.public_of`, `.sign`, `.signature_blob`, `.read_signature_blob`, `.supported_algorithms` | no |
| `sshauth.auth`, `.service_request`, `.none_request`, `.publickey_query`, `.publickey_signed_data`, `.publickey_signed` | no |
| `sshauth.password_request`, `.password_change_request`, `.read_reply`, `.after_reply` | no |
| `sshauth.method_supported`, `.attemptable`, `.is_done`, `.fault_of` | no |
| `sshchan.open_session`, `.read_open_confirmation`, `.read_open_failure` | no |
| `sshchan.exec_request`, `.shell_request`, `.pty_request`, `.subsystem_request`, `.window_change_request`, `.env_request` | no |
| `sshchan.may_send`, `.data_message`, `.sent`, `.consumed`, `.needs_adjust`, `.adjust_message` | no |
| `sshchan.read_event`, `.apply_event`, `.eof_message`, `.close_message` | no |
| `sshchan.is_finished`, `.can_send`, `.can_receive`, `.pty`, `.exit_status_of` | no |
| `sshconn.config`, `.accepting_unknown_hosts`, `.dial`, `.attach`, `.poll` | no |
| `sshconn.supply_random`, `.supply_host_verdict`, `.supply_signature`, `.supply_password`, `.offer_key` | no |
| `sshconn.exec`, `.shell`, `.subsystem`, `.write`, `.send_eof`, `.close_channel`, `.disconnect` | no |
| `sshconn.channel_of`, `.session_id_of`, `.agreed_of`, `.stream_of` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
