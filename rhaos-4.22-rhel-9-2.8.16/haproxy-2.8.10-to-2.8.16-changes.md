# HAProxy 2.8.10 to 2.8.16 Changes Documentation

## Overview

This document summarizes all changes between HAProxy versions 2.8.10 and 2.8.16, covering releases 2.8.11 through 2.8.16.

**Release Timeline:**
- 2.8.11: Released 2024
- 2.8.12: Released 2024
- 2.8.13: Released 2024
- 2.8.14: Released 2024
- 2.8.15: Released 2025-04-22
- 2.8.16: Released 2025-10-03

**Overall Statistics:**
- **150 files changed**
- **4,758 insertions**
- **1,546 deletions**
- **422 commits** with bug fixes, improvements, and documentation updates

---

## Critical Security Fixes

### BUG/CRITICAL: mjson DoS vulnerability (2.8.16)
- **Impact**: Possible Denial of Service when parsing numbers
- **Component**: mjson parser
- **Severity**: CRITICAL
- **Action**: Immediate upgrade recommended

---

## Major Bug Fixes

### Listeners and Connection Management
- **BUG/MAJOR**: Transfer connection accounting when switching listeners
  - Fixed connection accounting issues during listener switches

- **BUG/MAJOR**: Wake SC to perform 0-copy forwarding in CLOSING state (mux-h1)
  - Proper handling of stream connector wake-ups

### QUIC Protocol
- **BUG/MAJOR**: Fix wrong packet building due to already acked frames
  - Prevents packet corruption in QUIC streams

- **BUG/MAJOR**: Reject too large CRYPTO frames
  - Security fix to prevent oversized CRYPTO frames

### OCSP and SSL
- **BUG/MAJOR**: Separate refcount per instance and per store (OCSP)
  - Fixed reference counting to prevent crashes during OCSP updates

---

## Medium Severity Bug Fixes

### HTTP/2 and HTTP/3
- **Header validation**: Reject forbidden characters in Host header and :authority pseudo-header
- **Header limits**: Properly limit and increase max number of headers when sending/receiving
- **Whitespace handling**: Trim whitespaces in header values (H3)
- **Transfer-Encoding**: Reject empty Transfer-encoding headers (H1)
- **Protocol upgrades**: Fixed TCP>H1>H2 upgrade handling
- **Response handling**: Multiple fixes for early responses, 1XX interim responses

### QUIC Transport
- **Connection management**: Support wait-for-handshake, prevent conn freeze on 0RTT
- **Stream handling**: Handle retransmit for standalone FIN STREAM
- **CRYPTO parsing**: Prevent crashes due to CRYPTO parsing errors
- **Transport parameters**: Multiple validation fixes for TPs
- **Packet handling**: Repeat packet parsing to deal with fragmented CRYPTO

### SSL/TLS
- **0-RTT**: Fixed server-side 0-RTT functionality
- **Certificate selection**: Correct certificate using RSA-PSS with TLSv1.3
- **ECDSA**: Fixed ECDSA with ssl-max-ver TLSv1.2 and no ECDSA ciphers
- **AWS-LC compatibility**: Fixed build issues
- **OCSP**: Fixed crash when updating OCSP during ongoing updates
- **CA files**: ca-file directory mode must read every certificate in a file

### Connection Handling and Multiplexing
- **mux-h1**: Fixed timeout application, empty message handling, connection release
- **mux-h2**: Proper connection error handling, RST_STREAM fixes, header frame handling
- **mux-quic**: Timeout handling, stream closure, MAX_STREAMS fixes
- **mux-pt**: Never fully close connection on shutdown

### Backend and Load Balancing
- **Server reuse**: Fixed address collision with set-dst/set-dst-port
- **Connection reuse**: Do not overwrite srv dst address on reuse
- **Healthchecks**: ALPN inheritance, requeue on I/O events, timeout handling
- **Server maintenance**: Fixed server stuck in maintenance after FQDN change

### Queue Management
- **Queue logic**: Deal with rare TOCTOU in assign_server_and_queue()
- **Dequeuing**: Always dequeue backend when redistributing last server
- **Connection limits**: Never queue when no more served connections
- **Process improvements**: Make process_srv_queue return number of streams

### HTTP Analysis and Client
- **L7 retries**: Fixed request flag reset for L7 retries
- **Client aborts**: Proper detection during response forwarding
- **HTTP client**: Multiple data transfer, wake-up, and response handling fixes
- **Fast-forward**: Disable for unfinished requests waiting for upgrade

### Lua Integration
- **Memory safety**: Fixed UAF in cli applet, Queue:pop_wait()
- **Context handling**: Make hlua_ctx_renew() safe
- **Sample functions**: Properly handle errors in hlua_run_sample_{fetch,conv}()
- **Socket reporting**: Report to SC when data consumed/blocked
- **Channel methods**: Fix Channel:data() and Channel:line() to respect docs
- **httpclient**: Throw error if lua httpclient instance is reused

### Clock and Timing
- **Time jumps**: Detect and cover jumps during execution
- **Date offset**: Update date offset on time jumps
- **Expiration**: Always apply offsets to now_ms in various components
- **TICK_ETERNITY**: Ensure now_ms cannot be TICK_ETERNITY
- **Busy polling**: Fix time reporting

### Pattern Matching and ACLs
- **UAF prevention**: Prevent UAF on reused pattern expressions
- **Uninitialized reads**: Prevent in pat_match_{str,beg}
- **Const samples**: Prevent tampering in pat_match_beg()

### DNS and Resolvers
- **Reconnection**: Reset reconnect tempo when connection established
- **Connection tempo**: Add tempo between connection attempts
- **Resolution**: Insert non-executed resolution in front of wait list
- **FQDN**: Always normalize FQDN from response

### Stream Connections (stconn)
- **Shutdown**: Don't forward shut for SC in connecting state
- **Timers**: Only consider I/O timers to update expiration date
- **Error reporting**: Report blocked send if sends blocked by error
- **Pipe handling**: Request wake-up when pipe is full

### CLI and Management
- **Deadlock**: Fixed deadlock when setting frontend maxconn
- **Pipelined commands**: Fix pipelined modes on master CLI
- **FD transfer**: Wait for last ACK when FDs transferred
- **Thread display**: Fix "show threads" crashing with low thread counts

### Debugging and Diagnostics
- **Thread dump**: Close race between thread dump and panic()
- **STUCK flag**: Don't set from debug_handler()
- **Memory profiling**: Clean stale pool info on pool_destroy()
- **Tainted flag**: Add when ha_panic() is called

---

## Minor Bug Fixes (Selected Highlights)

### QUIC
- Proper error codes for various TP validation failures
- Fix room check if padding requested
- Do not emit probe data if CONNECTION_CLOSE requested
- Do not increase congestion window if app limited
- Reject NEW_TOKEN frames from clients
- Fix CRYPTO payload size calculation
- Reserve length field for long header encoding

### HTTP Processing
- Be able to use %ID alias anytime during stream evaluation
- Avoid recursive evaluation for unique-id based on itself
- Adjust server status before L7 retries
- Report -1 for %Tr for invalid response only

### Server Management
- Update healthcheck when server settings changed via CLI
- Don't warn fallback IP is used during init-addr resolution
- Fix dynamic server leak with check on failed init
- Make sure HMAINT state is part of MAINT

### Configuration Parsing
- Fix NULL ptr dereference in cfg_parse_peers
- Fix allowed args number for setenv
- Support one 'users' option for 'group' directive
- Relax LSTCHK_NETADM checks for non-root

### JWT
- Copy input and parameters in dedicated buffers in jwt_verify
- Clear SSL error queue on error when checking signature
- Don't try to load files with HMAC algorithm
- Fix variable initialization

### Stick Tables
- Cap sticky counter idx with tune.nb_stk_ctr
- Fix big-endian compatibility in smp_to_stkey()
- Fix crash for src_inc_gpc() without stkcounter
- Fix missing lock on some table converters

### Signal Handling
- Register default handler for SIGINT in signal_init()
- Fix soft-stop without multithreading support

### Namespace and Network
- Handle possible strdup() failure
- Delete fd from fdtab if listen() fails (TCP, UXST)
- Keep error msg if listen() fails
- Really assign distinct IDs to shards

---

## Features and Improvements

### MEDIUM Enhancements
- **SSL stack**: Initialize SSL stack explicitly
- **FD limits**: Set default for fd_hard_limit via DEFAULT_MAXFD
- **Lua**: Add function to change body length of HTTP Message
- **H1**: Accept invalid T-E values with accept-invalid-http-response option

### MINOR Enhancements
- **Applet**: Add appctx_schedule() macro
- **Channel**: Implement ci_insert() function
- **Compiler**: Add __nonstring macro, thread-safe variable declarations
- **Debug**: Improved thread dump mechanisms, memory profiling
- **QUIC**: Extended return values, better error traces, handshake notifications
- **Task**: Add thread-safe notification variants
- **Tools**: Improved symbol resolution without dl_addr

### Optimizations
- **Check**: Do not delay MUX for ALPN if SSL not active

---

## Documentation Updates

### Configuration Documentation
- Add details on prefer-client-ciphers
- Clarify json_query() converter limitations
- Recommend disabling libc-based resolution with resolvers
- Restore default values for resolvers hold directive
- Recommend single quoting passwords
- Improve http-keep-alive section
- Add tune.lua.burst-timeout to global parameters
- Update maxconn description
- Explain quotes and spaces in conditional blocks
- Add example for server "track" keyword

### Management Documentation
- Clarify usage of -V with -c
- Add missed -dR and -dv options
- Rename "dns" domain to "resolvers"
- More details about master-worker mode

### API Documentation
- Fix yield-dependent methods expected contexts (lua)
- Add note about httpclient object reuse
- Fix HTTPMessage.set_body_len() typos
- Clarify htx_xfer_blks() mark parameter

### Other Documentation
- List missing global QUIC settings
- Fix jwt_verify converter doc
- Refer to newer RFC5424 for ring
- Note unreliable sockpair@ on macOS

---

## Build and Compatibility

### Build Fixes
- Silence possible null deref warning in parse_acl_expr()
- Always set _POSIX_VERSION to ease comparisons
- Move ASSUME_NONNULL() for QUIC variables
- Avoid build warnings on gcc-4.8
- Silence build warning when USE_THREAD=0

### Platform-Specific
- **macOS**: Disable workaround to load libgcc_s
- **macOS**: Note about unreliable sockpair@
- **musl**: Backtrace support (reverted)
- **AWS-LC**: SSL build compatibility

### Thread Support
- Use pthread_self() not ha_pthread[tid] in set_affinity
- Fix soft-stop without multithreading support
- Improve thread-safe operations across components

---

## Regression Testing

### Test Suite Updates
- Rely on VTest2 to run regression tests
- Make conditional set-var compatible with Vtest2
- Explicitly allow failing shell commands in some scripts
- Fix random failures in wrong_ip_port_logging.vtc
- Update H1/H2 protocol upgrade tests
- Test pipelined commands on master CLI
- Never reuse server connection in truncated.vtc

---

## Reverted Changes

1. **BUILD**: Enable backtrace by default on musl (reverted)
   - Caused compatibility issues

2. **BUG/MINOR**: Reject QUIC addresses (reverted)
   - Needed further refinement

---

## Component Impact Summary

### Most Affected Components
1. **QUIC** (60+ fixes): Transport parameters, stream handling, connection management
2. **HTTP/2 & HTTP/3** (40+ fixes): Header validation, multiplexing, protocol compliance
3. **SSL/TLS** (20+ fixes): Certificate selection, OCSP, 0-RTT, compatibility
4. **Lua** (15+ fixes): Memory safety, API compliance, integration
5. **Backend/Queue** (15+ fixes): Connection reuse, load balancing, queuing logic
6. **Stream Connections** (10+ fixes): State management, error handling
7. **CLI** (10+ fixes): Command handling, FD transfer, deadlock prevention

### Cross-Cutting Improvements
- **Memory Safety**: Multiple UAF, leak, and crash fixes
- **Time Handling**: Consistent now_ms offset handling across components
- **Error Handling**: Better error reporting and recovery
- **Thread Safety**: Improved synchronization and race condition fixes
- **Protocol Compliance**: Stricter validation for HTTP headers and QUIC frames

---

## Upgrade Recommendations

### Critical
- Upgrade immediately for **mjson DoS vulnerability** fix

### High Priority
- QUIC users: Multiple stability and security fixes
- HTTP/2 & HTTP/3 users: Header validation and protocol compliance
- SSL/TLS users: 0-RTT and certificate selection fixes
- Lua users: Memory safety improvements

### Recommended for All
- General stability improvements across all components
- Better error handling and diagnostics
- Performance optimizations

---

## Migration Notes

### Breaking Changes
None identified. All changes are backward compatible bug fixes.

### Configuration Changes
- Consider enabling stricter HTTP validation if not already enabled
- Review SSL/TLS configuration for 0-RTT if used
- Check QUIC transport parameter configurations

### Testing Recommendations
1. Test QUIC connections thoroughly if using QUIC
2. Verify HTTP/2 and HTTP/3 header handling
3. Check SSL/TLS certificate selection
4. Test Lua integrations if using Lua
5. Verify backend connection reuse behavior
6. Test healthcheck and server maintenance operations

---

## References

- **Source Repository**: https://git.haproxy.org/?p=haproxy-2.8.git
- **GitHub Mirror**: https://github.com/haproxy/haproxy (main repository)
- **Official Website**: https://www.haproxy.org/
- **Documentation**: https://docs.haproxy.org/2.8/

---

## Detailed File Changes

### Source Files Modified (Top 20 by lines changed)
1. `doc/configuration.txt` (+629 lines)
2. `src/quic_conn.c` (+568 lines)
3. `CHANGELOG` (+422 lines)
4. `src/h3.c` (+231 lines)
5. `src/hlua.c` (+224 lines)
6. `src/stick_table.c` (+198 lines)
7. `src/mux_quic.c` (+196 lines)
8. `src/debug.c` (+186 lines)
9. `src/quic_tp.c` (+168 lines)
10. `src/ssl_ocsp.c` (+144 lines)
11. `src/tools.c` (+126 lines)
12. `src/haproxy.c` (+120 lines)
13. `src/http_ana.c` (+116 lines)
14. `src/pattern.c` (+113 lines)
15. `src/queue.c` (+108 lines)
16. `src/cli.c` (+108 lines)
17. `src/http_client.c` (+106 lines)
18. `src/ssl_sock.c` (+103 lines)
19. `src/mux_h1.c` (+85 lines)
20. `src/quic_stream.c` (+86 lines)

### Header Files Modified (Top 10)
1. `include/haproxy/http.h` (+61 lines)
2. `include/haproxy/task.h` (+62 lines)
3. `include/haproxy/quic_conn.h` (+31 lines)
4. `include/haproxy/compiler.h` (+28 lines)
5. `include/haproxy/defaults.h` (+26 lines)
6. `include/haproxy/stream.h` (+22 lines)
7. `include/haproxy/task-t.h` (+18 lines)
8. `include/haproxy/server-t.h` (+15 lines)
9. `include/haproxy/quic_conn-t.h` (+7 lines)
10. `include/haproxy/quic_tp-t.h` (+7 lines)

---

## Summary

HAProxy 2.8.16 represents a significant stability and security improvement over 2.8.10, with particular focus on:

- **Security**: Critical mjson DoS fix, header validation improvements
- **QUIC maturity**: Extensive transport layer fixes and improvements
- **HTTP/2 & HTTP/3**: Better protocol compliance and header handling
- **SSL/TLS**: Improved 0-RTT, certificate selection, and OCSP handling
- **Reliability**: Memory safety, race condition, and error handling fixes
- **Documentation**: Comprehensive updates across configuration and APIs

The upgrade from 2.8.10 to 2.8.16 is **strongly recommended** for all users, especially those using QUIC, HTTP/2, HTTP/3, or Lua integrations.
