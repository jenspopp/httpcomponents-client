# Apache HttpClient 5.5 Complete Timeout Configuration Guide

## Executive Summary

Apache HttpClient 5.5 provides many distinct timeout-related configuration settings. They are distributed across six configuration levels. This document provides a comprehensive list of all timeout configurations, 
their interactions, and override behaviors.


## Complete Timeout Inventory (Sorted by Configuration Level)

### Level 1: IOReactorConfig (Async Client - Most Generic)

**Package:** `org.apache.hc.core5.reactor`

#### 1.1 setSoTimeout(Timeout)
- **Default:** null (undefined)
- **Configuration:** `IOReactorConfig.Builder` → `HttpAsyncClients.custom().setIOReactorConfig()`
- **What Is Affected:** Maximum time for single I/O operation (read or write) on async socket. Time waiting for network data to arrive or be sent during async I/O operations. Applies to individual read/write operations, NOT entire request duration.
- **Phase:** Data Transfer (Async Read/Write)
- **Status:** Active (Async clients only)

#### 1.2 setSelectInterval(TimeValue)
- **Default:** 1 second
- **Configuration:** `IOReactorConfig.Builder` → `HttpAsyncClients.custom().setIOReactorConfig()`
- **What Is Affected:** How frequently the I/O reactor event loop checks for socket timeouts and session requests. Controls granularity of timeout detection. Lower value = more responsive timeout detection but higher CPU usage.
- **Phase:** I/O Event Loop (Background)
- **Status:** Active (Async clients only)
- **Note:** Not a timeout itself but controls timeout detection frequency

---

### Level 2: TlsConfig (TLS Strategy)

**Package:** `org.apache.hc.client5.http.config`

#### 2.1 setHandshakeTimeout(Timeout)
- **Default:** null (uses ConnectTimeout)
- **Configuration:** `TlsConfig.Builder` → `ClientTlsStrategyBuilder.create().setTlsConfig()`
- **What Is Affected:** Time to complete TLS handshake (ClientHello, ServerHello, certificate exchange, key exchange, Finished messages). Starts AFTER TCP connection established. Does NOT include TCP handshake or DNS resolution.
- **Phase:** TLS Handshake (after TCP, before HTTP)
- **Interaction:** Provides fine-grained control over TLS portion. ConnectionConfig.setConnectTimeout still applies to total time including TLS. If both set, shorter timeout triggers first.
- **Status:** Active (since 5.1)

***

### Level 3: SocketConfig (Socket Level - Classic Client)

**Package:** `org.apache.hc.core5.http.io`

#### 3.1 setSoTimeout(Timeout)
- **Default:** 3 minutes (classic client)
- **Configuration:** `SocketConfig.Builder` → `PoolingHttpClientConnectionManagerBuilder.setDefaultSocketConfig()`
- **What Is Affected:** Maximum inactivity period for socket read operations (SO_TIMEOUT). Time waiting for next data packet to arrive on socket. Applies ONLY during data reading (response headers/body). Measures inactivity between consecutive packets, NOT total duration. Timer resets on each packet received.
- **Phase:** Data Transfer (Socket Read - waiting for packets)
- **Interaction:** **OVERRIDDEN by RequestConfig.setResponseTimeout when ResponseTimeout is set to non-null value**
- **Status:** Active (Classic client)
- **TODO:** Not interaction with IOReactorConfig (async vs. classic client)
***

### Level 4: ConnectionConfig (Connection Manager)

**Package:** `org.apache.hc.client5.http.config`

#### 4.1 setConnectTimeout(Timeout)
- **Default:** 3 minutes
- **Configuration:** `ConnectionConfig.Builder` → `PoolingHttpClientConnectionManagerBuilder.setDefaultConnectionConfig()`
- **What Is Affected:** Total time to establish new connection from scratch:
  1. **TCP 3-way handshake** (SYN/SYN-ACK/ACK)
  2. **TLS handshake** if HTTPS
- **Phase:** Connection Establishment (TCP + TLS) - DNS is not included!
- **Status:** Active (Preferred since 5.2 - replaces deprecated RequestConfig.setConnectTimeout)
- **Critical Issue:** DNS can take 15+ seconds at OS level, potentially exceeding configured timeout

#### 4.2 setSocketTimeout(Timeout)
- **Default:** null (undefined)
- **Configuration:** `ConnectionConfig.Builder` → `PoolingHttpClientConnectionManagerBuilder.setDefaultConnectionConfig()`
- **What Is Affected:** Maximum inactivity period between data packets. Time waiting for next data packet during I/O operations. Each packet received resets the timer.
- **Phase:** Data Transfer (Read Operations)
- **Interaction:** **OVERRIDDEN by RequestConfig.setResponseTimeout when ResponseTimeout is set to non-null value**
- **Status:** Active

#### 4.3 setValidateAfterInactivity(TimeValue)
- **Default:** null (no validation)
- **Configuration:** `ConnectionConfig.Builder` → `PoolingHttpClientConnectionManagerBuilder.setDefaultConnectionConfig()`
- **What Is Affected:** Time a pooled connection can sit idle before requiring validation (stale check) before next use. NOT a timeout - it's a duration that triggers action. Helps detect connections closed by server or network issues.
- **Phase:** Connection Pool (Before Reuse)
- **Type:** Duration trigger (not timeout)
- **Status:** Active

#### 4.4 setTimeToLive(TimeValue)
- **Default:** null (unlimited)
- **Configuration:** `ConnectionConfig.Builder` → `PoolingHttpClientConnectionManagerBuilder.setDefaultConnectionConfig()`
- **What Is Affected:** Maximum age of connection from creation until forced closure. Total lifetime regardless of activity. Prevents long-lived connection issues (DNS changes, load balancer timeouts).
- **Phase:** Connection Lifecycle (Birth to Death)
- **Type:** Absolute duration (not timeout)
- **Status:** Active

***

### Level 5: RequestConfig (Client Default or Per-Request)

**Package:** `org.apache.hc.client5.http.config`

#### 5.1 setConnectionRequestTimeout(Timeout)
- **Default:** 3 minutes
- **Configuration:** `RequestConfig.Builder` → `HttpClients.custom().setDefaultRequestConfig()` OR `HttpRequest.setConfig()`
- **What Is Affected:** Time spent waiting for available connection from connection pool. If pool at max capacity and all connections in use, request waits up to this timeout. Does NOT include connection establishment or request execution time.
- **Phase:** Connection Pool (Lease/Acquisition)
- **Status:** Active

#### 5.2 setResponseTimeout(Timeout)
- **Default:** null (infinite)
- **Configuration:** `RequestConfig.Builder` → `HttpClients.custom().setDefaultRequestConfig()` OR `HttpRequest.setConfig()`
- **What Is Affected:** Total time from request start to response completion. Includes:
  1. Sending request headers/body,
  2. Server processing time,
  3. Receiving response headers/body. Starts when request execution begins (after connection obtained).
- **Phase:** Request Execution (Send + Process + Receive)
- **Override Behavior:** **"This parameter overrides the socket timeout setting applied at the connection management or I/O layers for the duration of a message exchange."**
- **HTTP/2 Caveat:** May not be fully supported with HTTP/2 multiplexing
- **Status:** Active

#### 5.3 setConnectTimeout(Timeout) ⚠️ DEPRECATED
- **Default:** 3 minutes
- **Configuration:** `RequestConfig.Builder` → `HttpClients.custom().setDefaultRequestConfig()` OR `HttpRequest.setConfig()`
- **What Is Affected:** DEPRECATED - Use ConnectionConfig.setConnectTimeout instead
- **Status:** DEPRECATED (use ConnectionConfig instead, since 5.2)

#### 5.4 setConnectionKeepAlive(TimeValue)
- **Default:** 3 minutes
- **Configuration:** `RequestConfig.Builder` → `HttpClients.custom().setDefaultRequestConfig()` OR `HttpRequest.setConfig()`
- **What Is Affected:** How long idle connection remains in pool available for reuse. Used when server does not send Keep-Alive header. Server can override with shorter Keep-Alive header value.
- **Phase:** Connection Pool (Idle Connection Management)
- **Type:** Duration (not timeout)
- **Status:** Active

***

### Level 6: ConnectionEndpoint (Runtime - Most Specific)

**Package:** `org.apache.hc.client5.http.io`

#### 6.1 setSocketTimeout(Timeout)
- **Default:** Inherited from config
- **Configuration:** Direct method call → `ConnectionEndpoint.setSocketTimeout()` during request execution
- **What Is Affected:** Runtime modification of socket read timeout on active connection. Changes timeout immediately for subsequent read operations. Useful for adjusting timeout mid-request (e.g., longer timeout after receiving 100-Continue).
- **Phase:** Data Transfer (Runtime Adjustment)
- **Interaction:** Highest priority - overrides all configured socket timeouts for this connection instance
- **Status:** Active

---

## Critical Timeout Interactions

### 1. ResponseTimeout OVERRIDES SocketTimeout

**Official Documentation:** *"This parameter overrides the socket timeout setting applied at the connection management or I/O layers for the duration of a message exchange."*

| Configuration | ResponseTimeout | Effective Timeout | Explanation |
|---------------|----------------|-------------------|-------------|
| SocketTimeout: 30s<br>ResponseTimeout: null | null (default) | **30s socket timeout** | Socket timeout applies when ResponseTimeout not set |
| SocketTimeout: 30s<br>ResponseTimeout: 10s | 10s | **10s response timeout** | ResponseTimeout overrides (shorter) |
| SocketTimeout: 10s<br>ResponseTimeout: 30s | 30s | **30s response timeout** | ResponseTimeout overrides (longer) |
| SocketTimeout: 30s<br>ResponseTimeout: DISABLED | DISABLED | **Infinite** | ResponseTimeout DISABLED overrides completely |

**Key Point:** The override is **complete replacement**, not "whichever is shorter wins."

### 2. DNS Resolution Critical Issue

**Problem:** DNS resolution is **NOT controlled by ConnectTimeout** and happens at the OS/JVM level.

#### Scenario: ConnectTimeout = 5 seconds, DNS takes 6 seconds

**Timeline:**
```
0-6 seconds: DNS resolution (OS-controlled, ConnectTimeout NOT counting)
6 seconds:   DNS completes, returns IP
6-11 seconds: TCP + TLS handshake (ConnectTimeout NOW counting, 5-second limit)
```

**Result:** Total connection time = 6 + 5 = **11 seconds** (not 5 seconds!)

**The connection will hang for 6 seconds during DNS** before ConnectTimeout even starts.

#### DNS Timeout Defaults
- **Linux/macOS:** 5-15 seconds per nameserver
- **Windows:** 5-15 seconds
- **Multiple nameservers:** Can multiply (3 nameservers × 5 seconds = 15 seconds)


## Complete Configuration Hierarchy and Override Rules

### Override Precedence (Highest to Lowest)

**For Socket Read Timeout:**
1. **ConnectionEndpoint.setSocketTimeout()** (Runtime - highest)
2. **RequestConfig.setResponseTimeout()** (OVERRIDES all socket timeouts when non-null)
3. **ConnectionConfig.setSocketTimeout()** (Connection manager)
4. **SocketConfig.setSoTimeout()** (Socket default)

**For Connect Timeout:**
1. **ConnectionConfig.setConnectTimeout()** (Preferred, 5.2+)
2. ~~RequestConfig.setConnectTimeout()~~ (DEPRECATED)

**For Response Timeout:**
1. **Per-request RequestConfig** (`HttpRequest.setConfig()`) (Highest)
2. **Client default RequestConfig** (`HttpClients.custom().setDefaultRequestConfig()`)
3. **null (infinite)** (Default)

### Configuration Scope

| Level | Scope | Applied To | Override Behavior |
|-------|-------|-----------|-------------------|
| Level 6: ConnectionEndpoint | Single connection instance | Runtime modification | Highest priority, immediate effect |
| Level 5: RequestConfig (per-request) | Single request | One HTTP request | Completely replaces client defaults (no inheritance) |
| Level 5: RequestConfig (client default) | All client requests | All requests from client | Used when request has no explicit config |
| Level 4: ConnectionConfig | All connections | Connection manager | Applies to all connections from manager |
| Level 3: SocketConfig | All sockets | Socket operations | Classic clients only |
| Level 2: TlsConfig | All TLS operations | TLS handshakes | Specific to TLS phase |
| Level 1: IOReactorConfig | All async I/O | Async client | Async clients only |

***

## Timeout Lifecycle Coverage

### Connection Establishment Phases

```
[Pool Wait] → [DNS] → [TCP] → [TLS] → [HTTP Traffic]
     ↓          ↓       └──┴────┘          ↓
ConnectionRequest  ⚠️ OS   ConnectTimeout  ResponseTimeout
Timeout         Uncontrolled             (overrides SocketTimeout)
```

| Phase | Duration | Timeout Applied | Can Be Isolated? | Notes |
|-------|----------|----------------|------------------|-------|
| **1. Pool Request** | Instant to timeout | ConnectionRequestTimeout | Yes | Only if pool exhausted |
| **2. DNS Resolution** | 0-15s (typical) | ⚠️ **NONE (OS-controlled)** | ❌ No | **Critical gap - requires custom DnsResolver** |
| **3. TCP Connect** | Milliseconds to seconds | ConnectTimeout | No (combined) | Part of connection establishment |
| **4. TLS Handshake** | 100ms-5s | ConnectTimeout + TlsHandshakeTimeout | Partially | TlsConfig provides fine control |
| **5. Request Send** | Milliseconds | ResponseTimeout (if set) | No | Part of request-response |
| **6. Server Processing** | Variable | ResponseTimeout (if set) | No | Part of request-response |
| **7. Response Receive** | Variable | ResponseTimeout (if set) | No | Part of request-response |
| **8. Connection Reuse** | Up to KeepAlive/TTL | N/A | N/A | Lifecycle management |

---

## Production Configuration Examples

TODO

**Note:** HTTP/2 ResponseTimeout may not be fully supported. Rely on IOReactorConfig and connection-level timeouts.

***
