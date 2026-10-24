
---

# Part II. The Floor

## Act IV. The Symlink That Broke Everything

### 1. The question that should have been asked earlier

The `open` call had the `O_NOFOLLOW` flag, and that flag was set on the final component of the path. It gave a warm feeling of safety: a symlink in place of a file is rejected by the kernel.

But `O_NOFOLLOW` on the last component says nothing about the components before it. A symlink in place of an intermediate directory passes the check unnoticed: the kernel follows it, descends into the target directory, and then opens the file there - honestly, with `O_NOFOLLOW`, since the final component really is a file.

The question "what if the symlink is not the last component of the path" had to be asked at design time. It was asked later, by an external audit.

### 2. Reproduction

A directory outside the root was created on the router, and a symlink to it inside the root:

```text
/tmp/outside/secret.txt          -> "SECRET_OUTSIDE"
/tmp/www/escdir -> /tmp/outside  (a symlink to a directory)
```

A request through the running server:

```text
GET /escdir/secret.txt  ->  200 OK  SECRET_OUTSIDE
```

The server served a file lying outside the document root. This is not a theoretical vulnerability - it reproduced on the production build, which was serving port 80 at that moment.

### 3. Why the tests were silent

Here begins the most unpleasant part. The tests were green. All of them.

- `run_all.sh` checked the **direct file symlink** - the one that `O_NOFOLLOW` correctly blocked. The test passed and created a feeling of safety.
- A symlink to an **intermediate directory** was checked by nobody. Such a test simply did not exist.
- The project had a file `test_state_machine.c`, which re-implemented a copy of the finite state machine inside the test and checked **that copy**, not the running server. The test could not detect an error in the server by definition: it was testing itself.

The lesson this act is written for: **a green test proves nothing if it checks a copy of the logic**. It is exactly the same error as "it compiles - therefore it works", one floor up. The compiler checks syntax, a copy-test checks the test author's intent, and neither checks what actually executes.

Hence the rule now in force in the project: **a test must talk to a live process through a socket**. Not to a function, not to a copy of the state machine, not to a model - to the process that listens on the port.

### 4. The first repair attempt: a frightening price

The obvious fix is to open the path component by component, applying `O_NOFOLLOW` at every step:

```c
fd = open(root, O_RDONLY | O_DIRECTORY);
for (each component) {
    fd = openat(fd, component, O_RDONLY | O_NOFOLLOW | O_DIRECTORY);
    if (fd < 0) return 404;
}
```

On MIPS this costs a lot: a path-parsing loop, a separate `openat` for every component, error handling at every step. The first safe build came out at **5,912 bytes** instead of 3,944.

Plus 1,968 bytes - almost double. At that moment it seemed that safety and Sub-4KB were incompatible: to close one hole you had to give up the achievement itself.

At the same time it turned out that the directory listing (`sys_getdents64`, HTML generation, name escaping) would also have to go: it cost about 200 bytes, and the budget was spent. A directory without `index.html` began returning `403` instead of a file list. This is a deliberate trade: **the listing was given up for path safety**.

### 5. Compression: returning to a single page

5,912 bytes is not a sentence, it is an intermediate point. Then began the work that eventually returned the size:

| Step | Size | What was done |
| :--- | ---: | :--- |
| Original vulnerable build | 3,944 | `O_NOFOLLOW` on the last component only |
| First safe build | 5,912 | component-by-component `openat`, listing removed |
| Compression | 4,064 | dead listing code, MIME dedup, structure merging |
| Range per RFC 7233 | 4,080 | three Range fixes + CLI validation |
| **Final** | **4,092** | `Cache-Control` + 416 on overflow |

Compressing from 5,912 to 4,064 gave minus 1,848 bytes - almost as much as safety had cost. The main sources: removing the dead listing strings (`dir_head`, `tag_a` and the rest of what remained in the binary after the feature was disabled), MIME type deduplication through a shared string pool and an offset table, merging small data structures.

Then - four Range fixes that cost only **16 bytes**:

1. `bytes=-N` with `N >= file size` returned `200` instead of `206` (RFC 7233 section 2.1 requires `206` with the full file). Fixed with a single `movz` instruction clamping `N` to the size.
2. `bytes=-0` returned `200` instead of `416`. Free: it falls into the existing `from >= size` check.
3. An oversized `last-byte-pos` (for example `bytes=0-99999999999999999999`) returned `200` instead of `206` with clamping. Cause: `parse_dec` read only 9 digits and did not recognize the rest of the number as a legal continuation.
4. `--port 8080junk` was accepted as `8080`. `parse_uint` did not check the end of the string.

Four fixes - sixteen bytes. It turned out cheaper than expected, and it was the first good news of the whole act.

### 6. The last 12 bytes: caching and overflow

The final two additions required shortening the reason-phrases to fit the page.

**`Cache-Control: max-age=60`** is emitted only on cacheable responses: `200`, `206`, `301`. The check is a single instruction `sltiu $t0, $s7, 400`: codes below 400 pass, errors do not. This matters: if `Cache-Control` were put on `404`, the browser would cache the error and not see the file that appears for a minute.

Cost: 20 bytes of code + 28 bytes of string, a measured binary growth of +44 bytes.

Why not `304 Not Modified`: `max-age` removes the request itself, saving both latency and traffic. `304` saves only the body; the request still reaches the server, and it costs 100-150 bytes (it needs `If-None-Match` parsing or HTTP-date formatting). On a 16-byte budget `max-age` gives more benefit for less money.

**Range: `first-byte-pos` overflow**. `parse_dec` reads at most 9 digits. If another digit follows, the value is necessarily over a billion, which exceeds any file on the target device (128 MB RAM, 61 MB tmpfs). Such a request gets `416`:

```mips
    lbu   $t0, 0($a0)
    addiu $t1, $t0, -48
    sltiu $t1, $t1, 10
    bnez  $t1, .L_hce_416       # 10th digit -> >= 1G -> 416
    nop                         # delay slot: branching in it is forbidden
    bne   $t0, $t7, .L_do_mime  # otherwise '-' is mandatory
    addiu $a0, $a0, 1           # [delay slot]
```

Cost: 16 bytes.

### 7. The trap in the delay slot

The first version of the 416 fix crashed the server with `SIGILL` on the very first request carrying a `Range` header. The build passed silently; the tests failed obscurely.

The cause: `bnez` got a `bne` instruction in its delay slot. On MIPS the delay slot executes **before** the branch, and if it contains a branch, the processor hits undefined behavior. QEMU honestly raised `SIGILL`.

Fixed with a single `nop`. The lesson: **after every edit to branching code, check the disassembly**, not just a successful build. The assembler does not catch this - it does not know what you put in the delay slot.

### 8. What it was paid with

16 bytes were free, about 48 were needed. The reason-phrases were shortened - pure cosmetics, clients use the numeric code:

| Code | Was | Became | Saving |
| :--- | :--- | :--- | ---: |
| 206 | `Partial Content` | `Part` | 11 |
| 301 | `Moved` | `Mv` | 3 |
| 400 | `Bad Request` | `Bad Req` | 4 |
| 403 | `Forbidden` | `Denied` | 3 |
| 404 | `Not Found` | `No` | 7 |
| 416 | `Range Not Satisfiable` | `Range` | 16 |
| 501 | `Not Implemented` | `Not Impl` | 7 |
| | | **total text** | **51** |

In the binary this gave minus 48 bytes: three bytes were eaten by string alignment in `.sdata`.

It is a readability regression: `404 No` instead of `404 Not Found`. Functionality is untouched; codes and headers are in place. The alternative - removing `--daemon` (about 61 bytes) - would have preserved the full phrases, but that is losing a function, not cosmetics.

---

## Act V. The Floor: 4,092 Bytes and the Decision to Stop

### 1. What came out of it

The final build: **4,092 bytes**, a 4-byte margin to the page.

| Parameter | Value |
| :--- | :--- |
| Size | 4,092 bytes |
| Margin to 4,096 | 4 bytes |
| `.text` | 3,360 bytes |
| `.sdata` | 572 bytes |
| Data segment on disk | `filesz = 0` |
| `VmHWM` in production | 28 KB |

The full set of checks is in place: GET/HEAD, HTTP/1.0 and 1.1, Keep-Alive, pipelining, `index.html`, 301 redirect, Range with 206/416, 17 MIME extensions (14 unique types) with `charset=utf-8`, eight status codes, component-by-component `openat` with `O_NOFOLLOW`, traversal protection, dotfiles return 403, `Host` validation, CRLF injection protection, obs-fold rejection, `Transfer-Encoding` rejection, epoll, dual-stack IPv6, `SIGPIPE` ignored, `sendfile`, `Cache-Control`.

### 2. Verified on live hardware

| Check | Result |
| :--- | :--- |
| `GET /`, `GET /index.html` | 200, body correct, md5 matches the file on disk |
| `HEAD /` | 200, no body |
| Range `0-9`, `-10` | 206, 10 bytes each |
| Range overflow | 416 |
| `HTTP/1.0` | version echoed, `Connection: close` |
| 301 on a directory | `Location` with `%20` preserved |
| Symlink to a file | 404 |
| Symlink to a directory | 404 |
| Path traversal | 400 |
| Directory listing | 403 (deliberate) |
| 2,000 requests, 64 threads | 2,000 / 2,000 |
| 4 MB transfer | md5 matched |
| `VmHWM` under load | 28 KB |
| Descriptors after load | 5 |

Separately verified: what the old 3,944-byte server let through and the new one blocks:

| Attack | 3,944 | 4,092 |
| :--- | :---: | :---: |
| CRLF injection in a header | 200 | **400** |
| Missing `Host` in HTTP/1.1 | 200 | **400** |
| Duplicate `Host` | 200 | **400** |
| `Transfer-Encoding` in GET | 200 | **400** |
| `Content-Length` in GET | 200 | **400** |
| `//` in the path | 200 | **400** |

### 3. The price: comparison with the version running on the router

Measurements on one file (16 bytes), `wrk -t2 -c50`, ASUS RT-AC57U V3 router. Three builds at once on different ports, five to seven runs each, runs interleaved so drift would not skew the result:

| Build | `Connection: close` | Keep-Alive | Against 3,944 |
| :--- | ---: | ---: | :--- |
| 3,944 (vulnerable) | 540 req/s | 1,442 req/s | - |
| 5,912 (first safe) | 404 req/s | 892 req/s | **-25% / -38%** |
| **4,092 (final, on the router)** | **524 req/s** | **1,498 req/s** | **-3% / +4%** |

A separate measurement on a single connection (`wrk -t1 -c1`, keep-alive): 3,944 - 228 req/s, 5,912 - 222 req/s, 4,092 - 232 req/s. All three equal.

The intermediate 5,912 build was noticeably slower under high concurrency: minus a quarter on connection close and minus 38% in keep-alive. These were the numbers previously taken for the price of the checks.

But the single-connection measurement says otherwise: processing one request costs all three the same, which means the checks themselves cost nothing. The difference appears only at fifty concurrent connections - which means 5,912 scaled worse, not computed slower.

The final 4,092 build performs at the level of the vulnerable one: 524 versus 540 on close and 1,498 versus 1,442 in keep-alive, the difference within noise. The compression work from 5,912 to 4,092 delivered both bytes and scalability.

Hence the conclusion that flips the intuition of the sizecoding community, where everything is measured in bytes: **the most expensive check is not the one that eats bytes, but the one that eats cycles - and what eats cycles is not the check but its clumsy implementation**. The same set of validations cost 38% in one build and zero in another.

### 4. The real contract: what it can and cannot do

All the numbers above describe size and speed. But the server has one more characteristic that articles about sizecoding usually omit: **what it can actually do**. Let us list it honestly, because 4,092 bytes impose hard limits, and the reader has the right to know them before deploying the server anywhere.

#### What it can do

| Capability | Details |
| :--- | :--- |
| **Methods** | `GET`, `HEAD`. `HEAD` without a body on all codes, including 400, 403, 404 and 416 |
| **Protocol versions** | HTTP/1.0 and HTTP/1.1, the version is echoed in the response |
| **Connections** | Keep-Alive, pipelining (several requests from one buffer) |
| **Index file** | `index.html` on a directory request |
| **Directory redirect** | `301` with `Location` from the raw URL: `%20` and other escapes preserved as received |
| **Range** | One range: `bytes=A-B`, `bytes=A-`, `bytes=-N`. Responses `206` and `416` |
| **MIME** | 17 extensions, 14 unique types. `charset=utf-8` only for textual ones |
| **Caching** | `Cache-Control: max-age=60` on `200`, `206`, `301` |
| **Status codes** | `200`, `206`, `301`, `400`, `403`, `404`, `416`, `501` |
| **Path protection** | Component-by-component `openat` with `O_NOFOLLOW` at every step. Traversal -> `400`, symlink -> `404` |
| **Dotfiles** | Any component starting with a dot -> `403` |
| **Request validation** | `Host` (presence, uniqueness, non-emptiness), CRLF injections, obs-fold, `Transfer-Encoding`, `Content-Length` in GET -> `400` |
| **Multiplexing** | `epoll`, one process, no threads |
| **Network** | Dual-stack IPv6 + IPv4 on one port (`AF_INET6` + `IPV6_V6ONLY=0`) |
| **File serving** | `sendfile` with a full loop, `EINTR` and partial send handling |
| **Signals** | `SIGPIPE` ignored, a client dropping the connection does not kill the process |
| **CLI** | `--port`, `--daemon` |

Verified on live hardware: 2,000 requests at 64 threads without loss, 4 MB through `sendfile` with matching md5, `VmHWM` 28 KB, descriptors return to five after load.

#### What it cannot do

| Missing | Consequence |
| :--- | :--- |
| **TLS** | Only behind a proxy. Passwords and tokens in the clear |
| **Directory listing** | A directory without `index.html` -> `403`. No browsing files in a browser |
| **Multiple ranges** | `bytes=0-1,4-5` is ignored entirely, a `200` with the full body is returned |
| **`ETag` / `Last-Modified`** | Only `Cache-Control`. No conditional requests, no `304` |
| **`If-Modified-Since`** | The header is read and ignored, the answer is always `200` |
| **`If-Range`** | Ignored: `206` is returned even when the validator does not match |
| **Compression** | `Accept-Encoding: gzip` is ignored, the body is always as-is |
| **Authentication** | Neither Basic nor tokens |
| **chroot and privilege drop** | Runs with whatever rights it was started with |
| **Logs** | Only a startup message. No access log, no error log |
| **CGI and any backend** | `POST`, `PUT`, `OPTIONS` -> `501`. Nowhere to send forms |
| **HTTP/2, WebSocket** | No |
| **Config file** | Command-line arguments only |
| **`--addr`** | Not supported, listens on all interfaces. Unknown flags are rejected |
| **Prefork** | One process, one CPU core |
| **slowloris protection** | See below |

#### A separate word on slowloris

The server does have a garbage collector: `epoll_wait` is called with a 3-second timeout, and if there are no events, all descriptors from `listen_fd+1` to 255 are closed. This is protection against hung connections.

But the trigger condition is **zero events**. While at least one connection is active, the collector does not run. Which means a client sending one byte at a time holds the connection indefinitely.

Measured: one byte every two seconds, 80 seconds - the connection is alive, the server has not dropped it. A typical request is 51 bytes, so at that pace a single connection is held for over a hundred seconds, not counting the time to answer it.

For a trusted local network this is acceptable. For a public address it is not: a few dozen such clients would exhaust the descriptor limit.

The direction of future work is not to close descriptors but to reuse them: remove from `epoll`, reset the connection state, return to the free pool. Then the collector could cut off truly dead connections without spending descriptors on repeated `accept`.

#### A separate word on dotfiles and ACME

The dotfile ban is protection: `.git`, `.env`, `.htpasswd` are never served. But it also blocks `.well-known/acme-challenge/`, and without it Let's Encrypt HTTP-01 certificate issuance does not work.

So the server fundamentally cannot obtain a TLS certificate by itself: one protection excludes another. A certificate can be issued on another machine and a proxy placed in front of the server - but that is already external infrastructure.

#### The bottom line

This implementation is **not a general-purpose web server**, but a static file server with a hard, limited contract. Everything listed concerns only the assembly version in GAS for MIPS32 Big-Endian.

The C implementation on nolibc closes most of this list: directory listing with escaping, `--addr`, `--workers` up to 16, a full non-blocking state machine with `EPOLLOUT` resume, `ETag` and `304`, `If-Modified-Since`, `If-Range`, `OPTIONS` with `Allow`, `405` instead of `501`, `Date`, `Accept-Ranges`, access and error logs, `--index`, `--no-listing`, `--pidfile`. Still outside the contract of both versions: TLS, CGI, WebSocket, HTTP/2, gzip, authentication, chroot, a config file and multiple ranges. That is the boundary between a size reference and a product.

**Suitable for**: serving static files and a web interface on modern Linux, a trusted host or local network, a separate directory without secrets, an unprivileged user, behind a proxy if TLS is needed.

**Not suitable for**: replacing nginx or Apache, a file manager, a public address with secrets, anything requiring authentication, form processing or logs.

### 5. The floor is found

4 bytes of margin. Any further addition requires either cutting an existing feature or compressing the code further.

What has already been given up: the directory listing. What could still be given up: `--daemon` (61 bytes), the `mjs` and `md` MIME types, compressing `strstr`/`strcmp`. Every option is losing a feature or risking a bug for a few dozen bytes.

And here it became clear: **the assembly path is exhausted**. Not because 4,092 is a bad number, but because what follows is no longer engineering but squeezing. Every new check is a battle for sixteen bytes, every edit requires recounting the entire data geometry.

The assembly path demands not just perseverance. It demands giving up much: readability, familiar abstractions, the ability to fix something quickly. When the goal is not merely to write working code but to fit into an artificial one-page memory limit while honestly delivering the declared features, every byte becomes a decision and every decision a compromise. It is a discipline that teaches a lot, but one cannot live in it permanently.

4,092 bytes proved the limit is reachable. Next, a decision had to be made about what to do with that knowledge.
