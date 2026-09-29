# mod_protect

Apache HTTP Server 2.4 module that limits concurrent requests and request rates per client IP, URI and virtual host. A request over a limit is rejected with `429 Too Many Requests` (no response body is generated).

## Requirements

- Apache HTTP Server 2.4
- `mod_status` with `ExtendedStatus On` (needed for the concurrent limits)
- APR shared memory and process-shared mutex support (needed for the rate limits)

## Build and install

```
apxs -c -I. mod_protect.c protect_scoreboard.c protect_rate.c
apxs -i -a mod_protect.la
```

Or use the included Makefile: `make && make install`.

```
LoadModule protect_module modules/mod_protect.so
```

## Overview

| Directive | Limits | Counted per |
|---|---|---|
| `ProtectMaxConcurrentPerIP` | simultaneous requests | client IP, all virtual hosts together |
| `ProtectMaxConcurrentPerVHost` | simultaneous requests | virtual host, all clients together |
| `ProtectRatePerIPVHost` | requests per time window | client IP and virtual host |
| `ProtectRatePerIPURI` | requests per time window | client IP, virtual host and URI |
| `ProtectRatePerIPDynamicURI` | requests per time window, dynamic requests only | client IP, virtual host and URI |
| `ProtectLog` | | additional log of rejected requests |

All limit directives take a non-negative integer. `0` (the default) disables the limit.

All directives are valid in server config and `<VirtualHost>` context only.

**Inheritance:** limits configured in the main server context are inherited by virtual hosts. A directive explicitly set to `0` in a virtual host disables the inherited limit for that virtual host. `ProtectLog` is also inherited.

## Concurrent limits

Count requests that are being processed at this moment, taken from the Apache scoreboard (requests that are being read, written or waiting for a DNS lookup). Idle keep-alive connections are not counted. These are not TCP connection limits and not rate limits.

A limit of `N` allows `N` simultaneous requests; the next one is rejected.

### ProtectMaxConcurrentPerIP

```
ProtectMaxConcurrentPerIP number
```

Maximum number of simultaneous requests from one client IP, counted across all virtual hosts.

### ProtectMaxConcurrentPerVHost

```
ProtectMaxConcurrentPerVHost number
```

Maximum number of simultaneous requests handled by the virtual host, from all clients together.

## Rate limits

Rate limits use fixed time windows. The window starts with the first request. Up to `Count` requests are allowed in the window and further requests are rejected until the window ends; then the next request starts a new window. Rejected requests are not counted.

Each rate directive takes two arguments: the maximum number of requests and the fixed window length in seconds. If the request count is `0`, that limit is off.

### ProtectRatePerIPVHost

```
ProtectRatePerIPVHost number seconds
```

Maximum number of requests from one client IP to one virtual host, across all URIs, within the specified number of seconds.

### ProtectRatePerIPURI

```
ProtectRatePerIPURI number seconds
```

Maximum number of requests from one client IP to one URI within the specified number of seconds. All requests are counted, static and dynamic.

The query string is ignored: `/a?x=1` and `/a?x=2` are the same URI.

### ProtectRatePerIPDynamicURI

```
ProtectRatePerIPDynamicURI number seconds
```

Like `ProtectRatePerIPURI`, but counts only dynamic requests. When configured, dynamic requests use this limit instead of `ProtectRatePerIPURI`; they do not consume the normal URI counter. If this limit is not configured, dynamic requests fall back to `ProtectRatePerIPURI`.

A request is dynamic if its handler is one of `cgi-script`, `fcgid-script`, `proxy-server`, `application/x-httpd-php`, or starts with `proxy:` (for example PHP-FPM through `SetHandler "proxy:unix:..."`). The URI and file extension are not checked.

## Logging

### ProtectLog

```
ProtectLog /path/to/file
```

Appends every rejected request to the given file, in addition to the Apache error log. The file is created if it does not exist. Each line has the form:

```
[timestamp] [concurrent] ip=192.0.2.10 vhost=www.example.com count_ip=21/20 count_vhost=5/80 directive=ProtectMaxConcurrentPerIP uri=/index.php
[timestamp] [rate] ip=192.0.2.10 vhost=www.example.com uri=/api/test count=101/100 directive=ProtectURICount
```

### Error log

Rejections are also written to the Apache error log at `notice` level, so the default `LogLevel warn` hides them. Enable them with:

```
LogLevel protect:notice
```

Each message names the directive that caused the rejection and shows `current/limit`:

```
mod_protect: ProtectMaxConcurrentPerIP exceeded: ip=192.0.2.10 vhost=www.example.com count_ip=21/20 count_vhost=5/80
mod_protect: ProtectURICount exceeded: ip=192.0.2.10 uri=/api/test count=101/100
```

If both concurrent limits are exceeded, both directive names are listed. For rate limits the current count is always the limit plus one.

`LogLevel protect:debug` additionally logs when the scoreboard is unavailable.

## Behavior

- Limits are checked late in request processing (fixups phase), only for the initial request. Subrequests and internal redirects are not counted, and neither are requests already rejected earlier (for example by access control).
- Concurrent limits are checked first, then rate limits in this order: site, URI, dynamic. The first exceeded limit rejects the request and is the one reported.
- Fail open: if the scoreboard is unavailable, or a rate-limit table is full, the request is allowed.
- Rate counters are kept in shared memory and are reset when Apache is restarted or reloaded.

## Implementation notes

Concurrent counts come from Apache's public scoreboard API. mod_protect keeps no counter of its own and does not query `/server-status`.

Rate counters are shared between Apache processes through APR shared memory, protected by a process-shared mutex:

```
logs/protect-rates.shm
logs/protect-rates.lock
```

Each rate category (site, URI, dynamic URI) has 16384 entries. Keys are hashed with FNV-1a (64-bit) and stored with linear probing.

## Configuration example

```
LoadModule status_module modules/mod_status.so
LoadModule protect_module modules/mod_protect.so

ExtendedStatus On
LogLevel protect:notice
ProtectLog /var/log/apache2/protect.log

<VirtualHost *:443>
    ServerName www.example.com

    # max 10 simultaneous requests per IP, 20 per virtual host
    ProtectMaxConcurrentPerIP 10
    ProtectMaxConcurrentPerVHost 20

    # max 300 requests per IP to this vhost in 1 s
    ProtectRatePerIPVHost 300 1

    # max 100 requests per IP to one URI in 10 s
    ProtectRatePerIPURI 100 10

    # max 20 dynamic requests per IP to one URI in 5 s
    ProtectRatePerIPDynamicURI 20 5
</VirtualHost>
```
