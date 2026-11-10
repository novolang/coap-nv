# Changelog

All notable changes to coap-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0

The first release.

### Added

- **`coapwire`**, the message of RFC 7252 with no allocation: the header,
  tokens of up to eight bytes (`CoapToken`), options with delta encoding
  and the extended forms at 13 and 269, the payload marker, the codes and
  the option numbers. A builder (`begin`, `add_uint`, `add_str`,
  `add_opaque`, `add_empty`, `add_path`, `add_query`, `start_payload`,
  `push_payload`) writes in order of option number and refuses what is out
  of order or does not fit. `parse` checks every option and the payload
  marker and reports a fault as a code. An iterator (`opt_iter`,
  `opt_next`) and the helpers `find_uint`, `has_option`, `count_options`,
  `path_is`, `query_has` and `unknown_critical` read a message in place.
- **`coapmsg`**, the message layer of section 4: the retransmission of a
  confirmable request as a timer the caller drives (`tx_begin`, `tx_tick`,
  `tx_on_message`), with the schedule of section 4.2, a draw supplied by the
  caller, piggybacked, separate and empty responses, pings and resets, and a
  memory of eight message IDs (`CoapDedup`) that detects a repeat within the
  exchange lifetime.
- **`coapblock`**, block-wise transfer, RFC 7959: the Block1 and Block2
  option values, `slice` for a server, a receiver (`blockrx`) for a client
  download or a server's reassembly of a request, and a sender (`blocktx`)
  for an upload. A peer that lowers the size is followed from the byte
  reached.
- **`coapobs`**, observe, RFC 7641: `observe_newer`, the freshness rule of
  section 3.4, and a table of eight observers with a sequence counter for
  each of eight resources (`CoapObservers`).
- **`coapserver`**, a server's request path: the `CoapResources` trait,
  `serve`, and `CoapNoResources`, the smallest table. A server routes by
  path, checks the method, refuses unknown critical options, answers a ping
  with a reset and recognises a repeated request.
- **`tests/embedded_probe.nv`**, a firmware program for a sensor node,
  built for `--target=nrf52-qemu`. It prints `PASS: coap-embedded` under
  QEMU. The image is 22,928 bytes of flash and 520 bytes of static RAM.
- Six test suites with vectors worked out by hand, among them the request
  and the response of RFC 7252 section 4.8.

### Not included

- CoAP over TCP, TLS and WebSockets (RFC 8323).
- DTLS, OSCORE and the Echo option.
- Resource discovery (RFC 6690).
- Proxying, multicast, and the client side of an observation beyond the
  freshness rule.
- Block sizes above 1024 bytes and Q-Block (RFC 9177).
- A response cache. The caller keeps the response it sent the first time.

### Known

- `CoapNoResources` is in the package so that `serve` and `find_resource` can
  carry `@no_alloc`: a function generic over a trait bound builds only when the
  compilation unit holds an impl of the trait. The filing is
  `no-alloc-through-a-trait-bound-is-refused-when-the-package-has-no-impl`.
