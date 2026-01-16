HAProxy Rebase to 2.8.18 for OpenShift Router

Context

**Previous release**: 2.8.10

**Release to rebase to**: [2.8.18](https://www.haproxy.org/bugs/bugs-2.8.18.html)
**Comparison**: [2.8.10-2.8.18](https://www.haproxy.org/bugs/bugs-2.8.10.html)

**Release timeline**:
- 2.8.11: 2024-09-19
- 2.8.12: 2024-11-08
- 2.8.13: 2024-12-12
- 2.8.14: 2025-01-29
- 2.8.15: 2025-04-22
- 2.8.16: 2025-10-03
- 2.8.17: 2025-12-19
- 2.8.18: 2025-12-25

-----

Overall Assessment

**Total known bugs fixed**: 335 (1 CRITICAL + 7 MAJOR + 128 MEDIUM + 199 MINOR)

**OpenShift router relevant fixes**:
- 1 CRITICAL bug: mjson DoS (no risk - not exposed)
- 3 MAJOR bugs: Stream channel analysis, mux-h1 zero-copy forwarding, and listener accounting (low risk - bug fixes)
- ~15 MEDIUM bugs in HTTP/1, HTTP/2, SSL/TLS, backend, and healthcheck areas (low risk - bug fixes)
- ~10 MINOR bugs in HTTP protocol compliance, error detection, and WebSocket handling (no risk to low risk)

**Filtered out (not relevant to OpenShift router)**:
- All QUIC protocol fixes (4 MAJOR bugs, many MEDIUM/MINOR) - not used
- All HTTP/3 fixes - not used
- All Lua scripting fixes - not used
- Prometheus exporter fixes - not enabled
- OCSP auto-update - not used

**Risk Assessment**: **Low**

All fixes are bug corrections that restore intended behavior without introducing new features or requiring configuration changes. The upgrade risk is low.

The most important bug fixes for OpenShift router are:
1. **Stream channel analysis** (BUG/MAJOR): Fixes stalled streams when errors occur during synchronous sends - affects all HTTP traffic
2. **HTTP/1 zero-copy forwarding** (BUG/MAJOR): Fixes connection leaks during graceful shutdown
3. **HTTP/1 tunnel mode** (BUG/MEDIUM): Fixes premature connection closure for WebSocket passthrough
4. **HTTP/2 idle connection** (BUG/MEDIUM): Fixes H2 backend connection reuse (if using H2 backends)
5. **SSL/TLS handshake improvements**: Multiple fixes for edge cases
6. **HTTP protocol compliance** (BUG/MINOR): Transfer-Encoding validation prevents potential HTTP request smuggling
7. **WebSocket and protocol upgrade** (BUG/MINOR): Fixes connection handling during upgrades

**Recommendation**: **Upgrade recommended**. The rebase to 2.8.18 brings important bug fixes with low risk:
- Fixes for stream processing that could cause stalled requests/responses
- WebSocket and tunnel mode reliability improvements
- HTTP protocol compliance and security hardening
- No configuration changes required
- No behavioral changes that affect user operations

All changes are bug fixes that correct broken behavior. No medium or high risk changes identified.

-----

Critical bugs (1)

  - [BUG/CRITICAL: mjson: fix possible DoS when parsing numbers](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=444144e): fixes Denial of Service vulnerability when parsing numbers with large exponents. The mjson library's strtod() implementation used an iterative loop in O(exp) time for exponent calculation, which could take a huge amount of time for unbounded exponents. The fix replaces this with shifts and squares approach computing the exponent in O(log(exp)) time. The DoS could be triggered by sending a specially crafted JSON number with a very large exponent.
      - **No risk** as mjson is primarily used for Prometheus metrics export (USE_PROMEX), which is not compiled or enabled in OpenShift router.

-----

Major bugs (7)

In chronological order:

  - [BUG/MAJOR: mux-h1: Wake SC to perform 0-copy forwarding in CLOSING state](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=1b53186): when zero-copy forwarding is enabled and the mux is in CLOSING state, the stream connector (SC) was not being woken up for I/O events. This caused the mux to ignore connection closures, leaving H1 connections blocked indefinitely waiting for shutdown timeout. Without a timeout configured, this led to connection leaks. The fix ensures SC is woken up even in CLOSING state.
      - **Low risk**. Fixes potential connection leaks during graceful shutdown. Improves stability for HTTP/1 connections. Zero-copy forwarding (splice) is a performance optimization that may be enabled in production.

  - [BUG/MAJOR: ocsp: Separate refcount per instance and per store](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=7a5ca2a): fixes OCSP response reference counting. Previously, if all ckch_inst (SSL contexts) referencing a certificate_ocsp were destroyed, the OCSP response would be removed even if the certificate remained in the system. The fix introduces two separate reference counters: one for live SSL_CTX pointers and one for ckch stores. This prevents crashes during OCSP auto-updates and ensures OCSP responses are properly managed during certificate lifecycle operations.
      - **No risk** as we don't use OCSP auto-update (ocsp-update directive on bind) in OpenShift router.

  - [BUG/MAJOR: quic: fix wrong packet building due to already acked frames](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=2d1c69d): prevents packet corruption in QUIC streams by properly handling already acknowledged frames.
      - **No risk** as QUIC is not used in OpenShift router.

  - [BUG/MAJOR: quic: reject too large CRYPTO frames](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=c090f34): security fix to prevent oversized CRYPTO frames in QUIC protocol.
      - **No risk** as QUIC is not used in OpenShift router.

  - [BUG/MAJOR: listeners: transfer connection accounting when switching listeners](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=27c07ab): when a bind_conf listens to multiple thread groups with shards, the per-listener connection count was not properly transferred when switching to another thread group. This resulted in incorrect connection counts (one listener with high values, another with negative values), causing HAProxy to stop accepting connections when maxconn is set on the bind_conf. The issue only affects configs with shards (e.g., CLI stats socket or "bind ... shards 1") and requires thread groups to be enabled.
      - **Low risk**. Only affects configurations with sharded binds and multiple thread groups. OpenShift router typically doesn't use sharded binds or multiple thread groups. If encountered, would cause connection acceptance to stop once maxconn is reached incorrectly.

  - [BUG/MAJOR: stream: Force channel analysis on successful synchronous send](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=99eb7ae): reverts a previous fix that was masking write events before synchronous sends. The previous approach could mask shutdowns performed during process_stream(), blocking the stream indefinitely when an error occurred. The new fix properly detects synchronous sends by forcing channel analysis with CF_WAKE_ONCE flag on the channel after detecting a write event from a synchronous send. This prevents streams from remaining blocked when I/O events are missed during error conditions.
      - **Low risk**. Fixes bug where streams could hang indefinitely when errors occur during data transmission. Corrects broken behavior to process streams properly. Important fix affecting all HTTP traffic.

  - [BUG/MAJOR: quic: use ncbmbuf for CRYPTO handling](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=e78271f): fixes QUIC CRYPTO frame handling by using proper buffer management.
      - **No risk** as QUIC is not used in OpenShift router.

-----

Notable medium bugs (15 out of 128)

All QUIC, HTTP/3, Lua, and Prometheus-specific bugs were filtered out. Focused on HTTP/1, HTTP/2, SSL/TLS, backend, and connection management.

**HTTP/2 and connection handling:**

  - [BUG/MEDIUM: mux-h2: make sure not to move a dead connection to idle](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=c18bf84): prevents moving dead HTTP/2 connections to idle pool.
      - Note: This was later reverted in 2.8.18 as it caused idle connection preservation issues on H2 backends.
  - [Revert "BUG/MEDIUM: mux-h2: make sure not to move a dead connection to idle"](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=2c49d15): reverts the above fix because it prevented idle connection preservation after certain accumulation thresholds, causing high backend connection rates.
      - **Low risk**. If OpenShift router doesn't use HTTP/2 on backend, no impact. For HTTP/2 backends, the revert improves connection reuse.
  - [BUG/MEDIUM: mux-h2: Properly handle connection error during preface sending](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=45edc07): fixes error handling during HTTP/2 connection initialization.
      - **Low risk**. Improves HTTP/2 error handling.

**HTTP/1:**

  - [BUG/MEDIUM: h1: prevent a crash on HTTP/2 upgrade](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=185cab7): prevents crash during protocol upgrade from HTTP/1 to HTTP/2.
      - **Low risk**. Prevents potential crash during protocol upgrades.
  - [BUG/MEDIUM: h1: Allow reception if we have early data](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=d7f77e8): fixes handling of TLS early data (0-RTT) with HTTP/1.
      - **No risk** as OpenShift router doesn't use TLS early data (0-RTT).
  - [BUG/MEDIUM: http-ana: Don't close server connection on read0 in TUNNEL mode](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=7b83b1a): prevents premature connection closure in tunnel mode (used for WebSocket and CONNECT).
      - **Low risk**. Fixes bug where WebSocket connections could close prematurely. Important for WebSocket passthrough and tunneled protocols.
  - [BUG/MEDIUM: http-ana: Report 502 from req analyzer only during rsp forwarding](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=9557a5f): fixes timing of 502 error reporting to avoid incorrect error responses.
      - **Low risk**. Improves error handling accuracy.

**SSL/TLS:**

  - [BUG/MEDIUM: ssl: Crash because of dangling ckch_store reference in a ckch instance](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=1f21378): fixes crash due to certificate store reference issues.
      - **Low risk**. Prevents potential crash during certificate operations.
  - [BUG/MEDIUM: ssl: take care of second client hello](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=324fd5c): properly handles second ClientHello messages in TLS handshakes (can occur with TLS resumption or certain client behaviors).
      - **Low risk**. Improves TLS handshake reliability.
  - [BUG/MEDIUM: ssl: ca-file directory mode must read every certificates of a file](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=09a28e9): ensures all certificates in a CA file are loaded when using ca-file directive, not just the first one.
      - **Low risk** if using ca-file. Improves certificate loading completeness.
  - [BUG/MEDIUM: ssl: create the mux immediately on early data](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=135c87c): fixes multiplexer creation timing with TLS early data (0-RTT).
      - **No risk** as OpenShift router doesn't use TLS early data.
  - [BUG/MEDIUM: ssl/clienthello: ECDSA with ssl-max-ver TLSv1.2 and no ECDSA ciphers](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=11f80a1): fixes ECDSA certificate selection when TLSv1.2 is the maximum version and no ECDSA ciphers are configured.
      - **Low risk**. Improves certificate selection logic for edge cases.

**Backend and connection management:**

  - [BUG/MEDIUM: backend: fix reuse with set-dst/set-dst-port](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=52a216f): fixes connection reuse when destination address is modified dynamically.
      - **No risk** as OpenShift router doesn't use set-dst/set-dst-port directives.
  - [BUG/MEDIUM: backend: do not overwrite srv dst address on reuse (2)](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=550a704): prevents destination address corruption during connection reuse.
      - **Low risk**. Improves connection reuse reliability.

**Healthchecks:**

  - [BUG/MEDIUM: checks: fix ALPN inheritance from server](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=6184a4f): ensures healthchecks properly inherit ALPN settings from server configuration.
      - **Low risk** if using ALPN in healthchecks. Improves healthcheck configuration consistency.
  - [BUG/MEDIUM: server: Duplicate healthcheck's alpn inherited from default server](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=496f6d4): fixes ALPN inheritance from default-server settings.
      - **Low risk** if using default-server with ALPN. Prevents configuration errors.
  - [BUG/MEDIUM: check: Set SOCKERR by default when a connection error is reported](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=7f971c6): improves error reporting for healthcheck connection failures.
      - **Low risk**. Better healthcheck error detection.
  - [BUG/MEDIUM: check: Requeue healthchecks on I/O events to handle check timeout](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=2730bd2): ensures healthcheck timeouts are properly enforced.
      - **Low risk**. Improves healthcheck timeout handling.

**Stick tables:**

  - [BUG/MEDIUM: stick-tables: Always return the good stksess from stktable_set_entry](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=d9ff6d5): fixes stick table session reference handling.
      - **No risk** if not using stick tables. OpenShift router typically doesn't use advanced stick table features.
  - [BUG/MEDIUM: stick-tables: Don't forget to dec count on failure](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=32bc627): fixes stick table reference counting.
      - **No risk** if not using stick tables.

**DNS:**

  - [BUG/MEDIUM: dns: Reset reconnect tempo when connection is finally established](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=542d058): fixes DNS reconnection backoff logic.
      - **No risk** if not using DNS resolution in HAProxy. OpenShift router typically resolves DNS externally.

-----

Notable minor bugs (10 out of 199)

Focused on HTTP protocol compliance, error detection, and operational improvements.

**HTTP protocol compliance and security:**

  - [BUG/MINOR: h2: forbid 'Z' as well in header field names checks](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=7294a29): fixes header name validation in HTTP/2 to properly reject uppercase 'Z' character. The check for forbidden uppercase letters (A-Z) in header field names was missing the 'Z' character due to incorrect range check. No real consequences but improves protocol compliance.
      - **No risk**. Protocol compliance fix.

  - [BUG/MINOR: h1: Reject empty coding name as last transfer-encoding value](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=a76e5b3): rejects malformed Transfer-Encoding headers with trailing empty values (e.g., "Transfer-Encoding: chunked,"). Previously the empty value was silently ignored.
      - **Low risk**. Improves HTTP/1 protocol compliance and request validation.

  - [BUG/MINOR: h1: Fail to parse empty transfer coding names](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=58ee718): properly rejects Transfer-Encoding headers with empty coding names.
      - **Low risk**. Prevents potential HTTP request smuggling vectors.

  - [BUG/MINOR: h2: always trim leading and trailing LWS in header values](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=8189d70): ensures HTTP/2 header values have whitespace properly trimmed for consistency.
      - **No risk**. Protocol compliance improvement, internal handling only.

**Error detection and logging:**

  - [BUG/MINOR: http-ana: Properly detect client abort when forwarding the response](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=734720b): fixes client vs server abort detection during response forwarding. Previously, a server abort could be incorrectly reported as client abort in logs (CD-- instead of SD--), especially when abortonclose option is set. The fix properly checks for both shutdown and abort flags on the back stream connector.
      - **No risk**. Improves logging accuracy for troubleshooting. Does not affect traffic handling, only log messages.

  - [BUG/MINOR: Don't report early srv aborts on request forwarding in DONE state](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=20aa0af): prevents incorrect server abort reporting when request forwarding is already complete.
      - **No risk**. Logging accuracy improvement only.

**Backend and server management:**

  - [BUG/MINOR: server: Update healthcheck when server settings are changed via CLI](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=2e94ab4): when SSL is enabled/disabled for a server via CLI, or when healthcheck address/port are updated, the healthcheck transport layer is now properly updated. Previously, healthchecks would continue using old settings until reload.
      - **Low risk**. Improves dynamic configuration updates. Useful for runtime server management.

  - [BUG/MINOR: backend: do not overwrite srv dst address on reuse](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=4de748d): prevents backend server destination address corruption during connection reuse.
      - **Low risk**. Connection reuse reliability improvement.

**Protocol upgrade and WebSocket:**

  - [BUG/MINOR: mux-h1: Don't pretend connection was released for TCP>H1>H2 upgrade](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=a5c0692): fixes connection tracking during protocol upgrades from TCP to HTTP/1 to HTTP/2.
      - **No risk**. Improves internal connection tracking during protocol upgrades.

  - [BUG/MINOR: http-ana: Disable fast-fwd for unfinished req waiting for upgrade](https://git.haproxy.org/?p=haproxy-2.8.git;a=commitdiff;h=0ff9b3f): disables fast-forward optimization for requests waiting for protocol upgrade (e.g., WebSocket). Prevents potential data loss during upgrade.
      - **Low risk**. Improves WebSocket and protocol upgrade reliability.
