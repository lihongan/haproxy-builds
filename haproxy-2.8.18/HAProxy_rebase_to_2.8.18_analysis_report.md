HAProxy Rebase to 2.8.18 for OCP 4.22

Context

**Previous release**: 2.8.10

**Release to rebase to**: [2.8.18](https://www.haproxy.org/bugs/bugs-2.8.18.html)

**Comparison**: [2.8.10-2.8.18](https://www.haproxy.org/bugs/bugs-2.8.10.html)

**Test PR**: https://github.com/openshift/router/pull/718

-----

Critical bugs (1)

  - [BUG/CRITICAL: mjson: fix possible DoS when parsing numbers](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=444144e): fixes DoS vulnerability when parsing JSON numbers with large exponents by replacing O(exp) iterative loop with O(log(exp)) shifts and squares approach.
      - **No risk** as mjson is used for Prometheus metrics export (USE_PROMEX), which is not compiled or enabled in OpenShift router.

-----

Major bugs (7)

  - [BUG/MAJOR: mux-h1: Wake SC to perform 0-copy forwarding in CLOSING state](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=1b53186): fixes connection leaks when zero-copy forwarding is enabled. The stream connector was not woken up for I/O events in CLOSING state, leaving H1 connections blocked indefinitely.
      - **Low risk**. Zero-copy forwarding (splice) may be enabled in production.

  - [BUG/MAJOR: ocsp: Separate refcount per instance and per store](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=7a5ca2a): fixes OCSP response reference counting by introducing two separate counters.
      - **No risk** as we don't use OCSP auto-update in OpenShift router.

  - [BUG/MAJOR: quic: fix wrong packet building due to already acked frames](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=2d1c69d): prevents packet corruption in QUIC streams.
      - **No risk** as QUIC is not used in OpenShift router.

  - [BUG/MAJOR: quic: reject too large CRYPTO frames](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=c090f34): security fix to prevent oversized CRYPTO frames in QUIC.
      - **No risk** as QUIC is not used in OpenShift router.

  - [BUG/MAJOR: listeners: transfer connection accounting when switching listeners](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=27c07ab): fixes per-listener connection count when using shards with multiple thread groups. Incorrect counts caused HAProxy to stop accepting connections.
      - **Low risk**. OpenShift router doesn't use sharded binds or multiple thread groups.

  - [BUG/MAJOR: stream: Force channel analysis on successful synchronous send](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=99eb7ae): fixes streams hanging indefinitely when errors occur during data transmission. Forces channel analysis with CF_WAKE_ONCE flag after synchronous sends.
      - **Low risk**. Important fix affecting all HTTP traffic.

  - [BUG/MAJOR: quic: use ncbmbuf for CRYPTO handling](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=e78271f): fixes QUIC CRYPTO frame handling.
      - **No risk** as QUIC is not used in OpenShift router.

-----

Notable medium bugs (10 out of 128)

All QUIC, HTTP/3, Lua, Prometheus, and non-router features were filtered out.

  - [BUG/MEDIUM: mux-h2: Properly handle connection error during preface sending](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=45edc07): fixes error handling during HTTP/2 connection initialization.
      - **Low risk**. Improves HTTP/2 error handling.
  - [BUG/MEDIUM: h1: prevent a crash on HTTP/2 upgrade](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=185cab7): prevents crash during protocol upgrade from HTTP/1 to HTTP/2.
      - **Low risk**. Prevents potential crash during protocol upgrades.
  - [BUG/MEDIUM: http-ana: Don't close server connection on read0 in TUNNEL mode](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=7b83b1a): prevents premature connection closure in tunnel mode (WebSocket and CONNECT).
      - **Low risk**. Important for WebSocket passthrough.
  - [BUG/MEDIUM: http-ana: Report 502 from req analyzer only during rsp forwarding](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=9557a5f): fixes timing of 502 error reporting.
      - **Low risk**. Improves error handling accuracy.
  - [BUG/MEDIUM: ssl: Crash because of dangling ckch_store reference in a ckch instance](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=1f21378): fixes crash due to certificate store reference issues.
      - **Low risk**. Prevents potential crash during certificate operations.
  - [BUG/MEDIUM: ssl: take care of second client hello](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=324fd5c): properly handles second ClientHello messages in TLS handshakes.
      - **Low risk**. Improves TLS handshake reliability.
  - [BUG/MEDIUM: ssl: ca-file directory mode must read every certificates of a file](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=09a28e9): ensures all certificates in a CA file are loaded, not just the first one.
      - **Low risk**. Improves certificate loading completeness.
  - [BUG/MEDIUM: ssl/clienthello: ECDSA with ssl-max-ver TLSv1.2 and no ECDSA ciphers](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=11f80a1): fixes ECDSA certificate selection edge case.
      - **Low risk**. Improves certificate selection logic.
  - [BUG/MEDIUM: backend: do not overwrite srv dst address on reuse (2)](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=550a704): prevents destination address corruption during connection reuse.
      - **Low risk**. Improves connection reuse reliability.
  - [BUG/MEDIUM: check: Set SOCKERR by default when a connection error is reported](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=7f971c6): improves error reporting for healthcheck connection failures.
      - **Low risk**. Better healthcheck error detection.

-----

Notable minor bugs (8 out of 199)

  - [BUG/MINOR: h1: Reject empty coding name as last transfer-encoding value](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=a76e5b3): rejects malformed Transfer-Encoding headers with trailing empty values.
      - **Low risk**. Improves HTTP/1 protocol compliance.
  - [BUG/MINOR: h1: Fail to parse empty transfer coding names](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=58ee718): properly rejects Transfer-Encoding headers with empty coding names.
      - **Low risk**. Prevents potential HTTP request smuggling.
  - [BUG/MINOR: http-ana: Properly detect client abort when forwarding the response](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=734720b): fixes client vs server abort detection in logs.
      - **No risk**. Logging accuracy only.
  - [BUG/MINOR: Don't report early srv aborts on request forwarding in DONE state](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=20aa0af): prevents incorrect server abort reporting.
      - **No risk**. Logging accuracy only.
  - [BUG/MINOR: server: Update healthcheck when server settings are changed via CLI](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=2e94ab4): properly updates healthcheck transport layer when SSL or healthcheck settings are changed via CLI.
      - **Low risk**. Improves dynamic configuration updates.
  - [BUG/MINOR: backend: do not overwrite srv dst address on reuse](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=4de748d): prevents backend server destination address corruption during connection reuse.
      - **Low risk**. Connection reuse reliability.
  - [BUG/MINOR: mux-h1: Don't pretend connection was released for TCP>H1>H2 upgrade](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=a5c0692): fixes connection tracking during protocol upgrades.
      - **No risk**. Internal connection tracking.
  - [BUG/MINOR: http-ana: Disable fast-fwd for unfinished req waiting for upgrade](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=0ff9b3f): disables fast-forward optimization for requests waiting for protocol upgrade (WebSocket).
      - **Low risk**. Improves WebSocket reliability.
