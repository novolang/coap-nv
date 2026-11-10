# coap-nv

CoAP, the Constrained Application Protocol, is HTTP's request and
response model rebuilt for a microcontroller on a lossy network. A client
sends a GET, PUT, POST or DELETE for a path; a server answers with a code
and a payload. It runs over UDP, so the protocol carries its own
retransmission, its own duplicate detection and its own way to move a
payload larger than one datagram. This package brings CoAP over UDP to
novo-lang: [RFC 7252](https://www.rfc-editor.org/rfc/rfc7252), with the
block-wise transfer of RFC 7959 and the observe option of RFC 7641.

## What it is

A sensor that answers `GET /temperature` needs four things from a CoAP
library: to read the request, to find the handler for its path, to build
the response with the request's message ID and token, and to notice a
request it has already answered. This package is those four, and then the
parts a larger exchange needs: the retransmission of a confirmable
request, the blocks of a payload that does not fit a datagram, and a
list of clients to tell when a reading changes.

The package has no socket and no clock. A program that uses it owns the
UDP socket and the timer. The package decides what to send and when to
send it again.

**Nothing is allocated.** A message is a `Vec[u8; 1152]` in the caller's
frame. A token is two integers. The state of an exchange, of a block
transfer and of a table of observers is a `@value` the caller holds. The
whole package builds for a Cortex-M with no heap.

**A server is a table and a function.** The caller implements
`CoapResources`, a trait with four functions: how many resources there are,
the path of each, the methods each allows, and a handler that fills in the
response. `serve` takes one datagram and builds the answer, with the
message layer of RFC 7252 around the handler. The effects of `serve` are
the effects of the handler: a handler that reads a sensor makes it a `[hw]`
function, and a handler in a test makes it pure.

## Install

```
novo pkg add coap-nv
```

## Example

A temperature sensor's server. The handler adds the payload and answers
the code; `serve` does the rest, and the program prints what the client
receives. On a device the two `CoapMsg` buffers are the datagram buffers
and the call is made when the UDP stack hands over a datagram.

```novo
use coapmsg
use coapserver
use coapwire

struct Temp
    unused: Int

impl CoapResources[] for Temp
    fn res_count(self) -> Int
        1

    fn res_path(self, i: Int) -> Str
        "temperature"

    fn res_methods(self, i: Int) -> Int
        coapserver.COAP_M_GET

    fn res_handle(self, i: Int, method: Int, var req: CoapMsg, h: CoapHeader,
                  var resp: CoapMsg, var b: CoapBuilder) -> Int
        let _ = coapwire.start_payload(var resp, var b)
        let _ = coapwire.push_payload_str(var resp, var b, "22.3 C")
        coapwire.COAP_CONTENT

fn main() [io]
    // The request of RFC 7252 section 4.8: GET /temperature, confirmable,
    // message ID 0x7D34.
    var req: coapwire.CoapMsg = Vec.new()
    var b = coapwire.builder()
    let _ = coapwire.begin(var req, var b, coapwire.COAP_TYPE_CON, coapwire.COAP_GET,
                           0x7D34, coapwire.token(0, 0))
    let _ = coapwire.add_path(var req, var b, "temperature")

    var resp: coapwire.CoapMsg = Vec.new()
    var seen = coapmsg.dedup()
    let result = coapserver.serve(Temp { unused: 0 }, var req, var resp, var seen, 1)

    let h = coapwire.parse(var resp)
    println("${result} type ${h.kind} code ${h.code} id ${h.mid} payload ${coapwire.payload_len(h)}")
    // 1 type 2 code 69 id 32052 payload 6
```

Build and test with `novo pkg build` and `novo test tests`.

## What the package contains

| Module | Contents |
| --- | --- |
| `coapwire` | The message of RFC 7252 section 3: the header, the token, the options with their delta encoding, the payload marker, the codes and option numbers; a builder that writes them in order into a caller's buffer, a parser that checks them from end to end, an iterator over the options, and the helpers for Uri-Path, Uri-Query, integer options and critical options. |
| `coapmsg` | The message layer of section 4: the four types, the retransmission schedule of a confirmable request as a timer the caller drives, the piggybacked, separate and empty responses, pings and resets, and the memory of message IDs that detects a repeat. |
| `coapblock` | Block-wise transfer, RFC 7959: the Block1 and Block2 option values, the slice of a resource a block names, a receiver that takes blocks in order, and a sender for an upload. |
| `coapobs` | Observe, RFC 7641: the sequence rule of section 3.4 that decides which notification is newer, and the table of observers and sequence numbers a server keeps. |
| `coapserver` | A server's request path: the `CoapResources` trait, and `serve`, which reads a request, routes it, runs the handler and builds the response. |

Every `pub` declaration is documented where it is declared, and
[the package's page](https://novo-lang.org/packages/coap-nv) is that
reference.

## The rules a user needs

1. **A message is built in order.** Options are written in the order of
   their numbers, because each is a delta from the one before. The builder
   refuses an option out of order, one after the payload marker, and
   anything that does not fit. A refusal answers `false`, empties the
   message, and marks the `CoapBuilder` failed, so a run of calls can be
   checked once at the end with `b.ok`.
2. **A parse is checked to the end.** `parse` walks every option before it
   answers, so a message that parses has options that stay inside it and a
   payload that starts where `payload_off` says. A fault is a code in
   `status`: too short, the wrong version, a token longer than 8 bytes, a
   bad option, a payload marker with no payload, an Empty message that is
   longer than four bytes.
3. **A token is a length and a number.** Up to eight bytes, the first in
   the highest position. Two tokens are equal when their lengths and
   numbers are, and no buffer is held.
4. **The caller drives the timer.** `tx_begin` takes a draw from 0 to 1000
   that the caller makes, and sets the first wait between 2 and 3 seconds.
   `tx_tick` is told how many milliseconds passed and answers whether to
   send again. The waits double: 2, 4, 8, 16 and 32 seconds for a draw of
   0, and after four repeats the request is given up. A fixed draw makes
   the schedule repeatable in a test.
5. **A peer is a number the caller chooses.** The package sees no
   addresses. The table of observers and the memory of message IDs are
   kept for one peer at a time or are given a number to tell peers apart;
   the caller keeps one `CoapDedup` for each peer it talks to.
6. **A repeat is not run twice.** `serve` remembers the IDs of the last
   eight messages for 247 seconds and answers a repeat with
   `COAP_SERVE_DUPLICATE`. It does not keep the response; the caller sends
   the one it kept, or sends none.
7. **Block sizes are powers of two from 16 to 1024.** A block option holds
   the number, a flag for more, and an exponent. A receiver may be given a
   smaller size than it asked for, and goes on from the byte it has
   reached, so its position is an offset and not a block number.
8. **A notification is newer by a rule on a circle.** The sequence number
   is 24 bits and wraps, so `observe_newer` compares them on a circle and
   also treats anything after 128 seconds as new.
9. **A handler returns a code, or defers.** `serve` begins the response
   before the handler runs, with the type, the message ID and the token the
   request calls for. The handler adds options in order and a payload, and
   returns the code. `COAP_DEFER` makes `serve` answer with an empty
   acknowledgement; the caller sends the response later with
   `coapmsg.build_separate`. A handler that leaves the builder failed, or
   returns something that is not a response code, is answered with 5.00.
10. **Unknown critical options are refused.** A confirmable request with a
    critical option the package does not understand is answered with 4.02
    and a non-confirmable one is dropped, as section 5.4.1 says.
11. **A message that cannot be read costs a reset at most.** A confirmable
    one with the right version is reset; anything else is dropped.

## Running on a microcontroller

All five modules are `@tier(embedded)` throughout and allocate nothing, and
the package declares no effects. `tests/embedded_probe.nv` is a firmware
program for a sensor node. It answers a request over a table of two
resources and recognises the repeat, runs a confirmable request through
its retransmission schedule until it gives up, downloads a resource of
1500 bytes in blocks of 512, and numbers two notifications for an
observer. It is compiled for a Cortex-M as part of this package's checks,
so the claim is built rather than asserted, and under QEMU it prints
`PASS: coap-embedded`.

The image is 22,928 bytes of flash and 520 bytes of static RAM, including
the runtime and the board's startup code. The program's own code is 10,480
bytes of that flash. The probe's frame holds three message buffers of
1,160 bytes, a table of observers of 432 bytes and a memory of message IDs
of 144 bytes. A smaller `COAP_MSG_MAX` would be a change to the capacity of
`CoapMsg`.

The device effects are the caller's. The package reads no register and no
clock, so a call costs what its caller's datagram transport costs, and
`serve` is exactly as effectful as the handler it is given. A handler that
reads a sensor over I2C makes `serve` a `[hw]` function at the call site.

## What is not included

- **CoAP over TCP, TLS and WebSockets, RFC 8323.** It has a different
  framing and a different message layer; this package is CoAP over UDP.
- **Security.** DTLS, OSCORE and the Echo option are not here. A datagram
  arrives as bytes the caller has already decrypted, or not.
- **Resource discovery, RFC 6690.** The link format of `/.well-known/core`
  is not built or parsed. A handler for that path is the caller's.
- **Proxying and multicast.** Proxy-Uri and Proxy-Scheme are refused as
  critical options. The leisure delay for a multicast request is a
  constant, `COAP_DEFAULT_LEISURE_MS`, and nothing applies it.
- **The client side of an observation.** The package decides whether a
  notification is newer. Keeping the registration alive, and
  re-registering after `Max-Age`, are the caller's.
- **Parsing a `coap://` URI** into a host, port, path and query. The
  helpers add a path and a query to a message.
- **Block sizes above 1024 bytes,** and the Q-Block options of RFC 9177.
- **A cache for responses.** The memory of message IDs recognises a
  repeat. The response sent the first time is kept by the caller.
- **Sockets and timers.**

## Related packages

- [cbor-nv](https://novo-lang.org/packages/cbor-nv) writes and reads the
  CBOR that CoAP payloads are most often made of. The payload here is bytes
  the package does not read.

## Tests

The suite is six files under `tests/`. What they assert comes from the
RFCs rather than from an implementation.

`tests/coapwire_tests.nv` holds the message to byte strings worked out by
hand from section 3 of RFC 7252, including the request and the response of
the RFC's own example in section 4.8. It covers the extended option deltas
and lengths at 13 and 269, the integer option sizes, every parse fault and
every way a builder refuses.

`tests/coapmsg_tests.nv` follows the retransmission schedule of section 4.2
step by step, every event an exchange can see (a piggybacked response, an
empty acknowledgement and a separate response, a reset) and the memory of
message IDs, including its eviction and its expiry.

`tests/coapblock_tests.nv` holds the option value to RFC 7959's
`NUM << 4 | M << 3 | SZX`, the slice of a body at every edge, and the
receiver and sender, including a peer that lowers the size.

`tests/coapobs_tests.nv` holds the freshness rule of RFC 7641 section 3.4
at the edges of the circle, and the table of observers when it is full.

`tests/coapserver_tests.nv` runs a server over a table of resources through
every branch of the request path, and ends with three whole exchanges: a
Block2 download, a Block1 upload and two notifications to an observer.

`tests/embedded_probe.nv` is the microcontroller program described above.

## Licence

Apache-2.0. See `LICENSE`.
