
---

# Part III. The Ascent

## Act VI. The Move to C: nolibc, a Home-Grown wsock.h, and Porting the MIPS Version

### 1. Why not assembly

Having hit the floor on MIPS32, the project ran into the problem of scaling protocol logic. The assembly server is blazingly fast and weightless, but what if industrial-grade mechanisms need to be added?

- A full non-blocking finite state machine with `EPOLLOUT` resume (so that while serving a large file to a slow client the process does not block in `sendfile` but asynchronously switches between hundreds of connections);
- HTTP/1.1 pipelining support (batch processing of coalesced requests from one buffer with a `memmove` shift);
- A ring timeout collector and slowloris protection;
- Extending the MIME table and new status codes.

Each of these additions in assembly is a battle for bytes that no longer exist. Hand-writing a multi-phase state machine in assembly inevitably duplicates the code base.

### 2. Is C to blame

And here the circle closes: **is the C language itself to blame for bloated binaries?**

Not at all. The GCC compiler is by nature a translator from C text to GNU Assembler (GAS) text. All the excess weight and the megabyte memory appetite comes from the standard library libc.

Dropping the standard library entirely (`-nostdlib`, `-fno-builtin`) makes it possible to combine pure C (Freestanding C99), the official single-file header **`nolibc.h`** from the Linux kernel source tree (`tools/include/nolibc`, written by HAProxy author Willy Tarreau specifically for building ultra-compact system utilities), and a thin home-grown layer of networking syscalls, **`wsock.h`**.

This is not "synthesizing assembly tricks into C". The MIPS version was ported to C, and the network layer, which `nolibc` lacks, was written separately. A deliberate step: regain the ability to develop the code, paying for it with size.

### 3. The anatomy of wsock.h: taming MIPS and x86 incompatibilities

Stock `nolibc.h` has no networking or signal support. The `wsock.h` header (9 syscalls and 4 structures) isolates all the architectural specifics of porting between x86-64 and MIPS32:

- **Inverted socket constants in the MIPS ABI**: in MIPS Linux `SOCK_STREAM` is 2 (while on x86 and ARM it is 1 - on MIPS a one creates a UDP socket!), `SOL_SOCKET` is 0xFFFF, and the non-blocking flag `O_NONBLOCK` is 0x0080.
- **Kernel structure geometry**: on x86-64 `epoll_event` is packed to 12 bytes, on MIPS it is aligned to 16. In the MIPS `sigaction`, `sa_flags` comes first and the `sa_handler` pointer second.
- **Zero-Stack syscalls for MIPS O32**: the standard `nolibc` macros reserve 32 bytes of stack before every syscall (`addiu $sp, $sp, -32 / +32`). Since the Linux kernel does not write into this area of the user stack, replacing them with stackless assembly stubs `ws_syscall0..4` saves 2 instructions (8 bytes) on each of the program's 31 calls.

### 4. The httpd.c architecture: an academic state machine without malloc

On top of `wsock.h` sits a single cross-platform source `httpd.c`, compiled without a single `#ifdef` for both x86-64 and MIPS32 Big-Endian:

- **Zero allocation**: no `malloc`, `calloc` or `free` calls at all. All state arrays and buffers are static in `.bss`, allocated by the kernel on demand (demand paging).
- **A safe string builder (`struct buf`)**: the structure `{ char *p; char *end; }` (the high-level analogue of the assembly version's end-to-end `$a0` pointer) makes it physically impossible to write past the end of a buffer.
- **A non-blocking pipeline**: when `sendfile` returns `EAGAIN`, the descriptor is switched to waiting on `WS_EPOLLOUT`, preserving the `body_sent` offset.

### 5. Fixing a protocol bug in Range (206 Partial Content)

An audit of `httpd.c` found and fixed a subtle bug in building the 206 Partial Content response: the base function `build_header()` terminated the header with a double CRLF `\r\n\r\n`, after which the Range handler appended `Content-Range: bytes ... \r\n\r\n` outside the now-closed header block. The client then read the `Content-Range` line as the first 35 bytes of the file body.

Rolling the write pointer back 2 bytes before inserting `Content-Range` gave 100% RFC 7233 compliance:

```c
/* Insert Content-Range strictly before the closing \r\n of the header block */
hb.p = g_hdr[fd] + g_hlen[fd] - 2;
```

### 6. What sizecoding means in C

Moving to C does not cancel the size question. It reframes it.

In assembly, size is a property of every line. Want it smaller - write different instructions. In C there is no such direct link: between the source and the machine code stands the optimizer, and it is often smarter than the author. The only way to check was experiment.

The largest function was taken - `prepare()`, request parsing, 3.8 KB on x86-64 and 4.6 KB on MIPS - and an attempt was made to shrink it six ways.

| Attempt | Idea | Result |
| :--- | :--- | ---: |
| 1 | A single exit via `goto` for 15 `error_response(400)` calls | **+32** |
| 2 | A `switch` on the first letter of the header name instead of an `iequal` chain | **+64** |
| 3 | Version check via `memcmp("HTTP/1.",7)` instead of two `strcmp` | **+32** |
| 4 | A `tchar[128]` table instead of `strchr` for name validation | **+112** |
| 5 | Moving directory handling into a separate `resolve_dir()` function | **+96** |
| 6 | A byte-by-byte `memcmp` scan at every position -> `strstr` | **-64** |

Five of the six attempts made the code bigger. The reasons are concrete:

- The single exit was unneeded: GCC already merges identical tails into a common block, and the label added a jump.
- A `switch` over five `case`s generates a jump table, and it costs more than a comparison chain.
- A 128-byte table costs more than the `strchr` calls it saves.
- Extracting a function added a prologue and duplication, and the separation gains did not cover the overhead.

The only win came not from replacing a construct with a "cleverer" one but from **eliminating redundant work**. The end-of-headers search was written like this:

```c
for(i=0;i+3<c->used;i++)if(!memcmp(r+i,"\r\n\r\n",4)){end=r+i;break;}
```

One `memcmp` call with four comparisons per request byte. For a 500-byte request that is 500 calls. The buffer is already NUL-terminated after each `read`, so `strstr(r,"\r\n\r\n")` is enough. Minus 64 bytes and noticeably less work: throughput rose from 24,700 to 25,200 requests per second.

Hence a conclusion worth remembering for anyone coming to sizecoding from assembly: **in C, size is determined not by how cleverly a line is written but by how much needless work the program does**. The optimizer beats the author in micro-optimizations almost always. It loses only where the author sees the whole algorithm: an extra pass over a string, an extra call per byte, an extra copy.

Sizecoding in C is not a battle for bytes but a hunt for the unnecessary.

This review also exposed a bug no test had seen: the `c->options` field was not reset between requests on a keep-alive connection. After an `OPTIONS`, the next `GET` was handled as `OPTIONS` and returned `200` with an empty body. The bug survived several revisions because every test opened a fresh connection and the `memset` at `accept` zeroed the structure. It surfaced only when connection state was gathered in one place - and it became visible that one flag was missing there.

### 7. What the C version can do now

The easiest way to explain the difference: the assembly version answers the question "can a server fit into one memory page". The C version answers "what must a server be so that people will use it".

Here is what it does.

#### It serves files

This is the foundation. A file request gets an answer with the correct content type. The browser receives `text/html` for a page, `image/png` for a picture, `application/javascript` for a script, `application/wasm` for WebAssembly. Seventeen extensions in all, and textual types automatically get `charset=utf-8` appended, so Russian text does not turn into mojibake.

A directory without a file name serves `index.html`. If there is no index file, it shows a file list where names are escaped, so a file called `<script>` in the list is harmless. The listing can be disabled with `--no-listing`, and then a directory without an index honestly answers `403`. The index file can be renamed: `--index home.html`.

If a directory is requested without a trailing slash, the server answers with a `301` redirect - the browser adds the slash and repeats the request itself. Escapes in the address are preserved as received, so a directory with a space in its name works.

#### It saves traffic

Three mechanisms, all three matter on a slow link.

**Caching.** The server tells the browser: this file need not be re-asked for a minute (`Cache-Control: max-age=60`). Error codes do not carry this header - otherwise the browser would remember a `404` and miss the file that appeared a second later.

**Conditional requests.** Every file has a fingerprint (`ETag`) and a modification date (`Last-Modified`). When the browser returns, it sends the fingerprint, and if the file has not changed the server answers `304 Not Modified` - no body. The entire file volume is saved; only headers travel. It works by fingerprint and by date alike: old clients send the date, new ones the fingerprint. If both arrive, the fingerprint wins, as the standard requires.

**Resumption.** A `Range: bytes=1000-2000` request returns exactly that piece with code `206`, not the whole file. This is what download managers and video players use when seeking. If the range is impossible (the file is shorter), a `416` arrives with the real size. If the file changed since the download began, the client sends `If-Range`, and the server sends the whole file instead of the piece - otherwise the client would stitch the old and new versions together.

#### It does not give away what it should not

**Hidden files are closed.** Any file or directory starting with a dot answers `403`. This protects `.git`, `.env`, `.htpasswd` and everything else that must not leak onto the network. The single exception is `/.well-known/`, because without it a Let's Encrypt certificate cannot be issued. An escape through that exception is closed too: `/.well-known/../.env` gets `403`.

**Escaping the root is closed.** The path is opened one component at a time, and following symlinks is forbidden at every step. A symlink to a file, a symlink to a directory, an escape attempt through `..` - all of it `403`. This is exactly where the hole in the 3,944-byte version was.

**Garbage in the request is rejected.** The server checks `Host` (mandatory in HTTP/1.1, must not be duplicated), rejects attempts to smuggle a line break into a header, rejects the obsolete way of folding headers onto the next line, rejects `Transfer-Encoding` and `Content-Length` where they must not be. All of that gets `400`. This closes attacks that try to mix in their own response through a header.

#### It works with modern clients

`GET` and `HEAD` are normal requests and requests without a body. `OPTIONS` is the question "what can you do", and the answer lists the allowed methods. `POST`, `PUT`, `DELETE`, `PATCH` get `405` with the same list: the method is known, but a static server does not perform it. An unknown method gets `501`.

HTTP/1.0 and HTTP/1.1, a connection can be kept open (`keep-alive`) and several requests sent without waiting for answers (`pipelining`). Works over IPv6 and IPv4 simultaneously on one port; the bind address is set with `--addr`.

Every response carries `Date` - without it some proxies and clients behave strangely. There is `Accept-Ranges: bytes` - from it the client learns that resumption is supported.

#### It serves many clients

Four processes by default, up to sixteen with `--workers`. Each holds up to 128 connections, which on four workers is a little over five hundred. Inside is a non-blocking `epoll` loop: while one client slowly receives a file, the server serves the others instead of waiting.

Why 128 and not more. On a router running kernel 3.3.8, memory for connection buffers is not an abstraction: with a hundred open connections the process takes 488 KB, and with 256 slots it would take 624 KB. A 136 KB difference per worker, times four workers - half a megabyte. Five hundred connections is manifestly enough for a router's web interface, so extra slots are memory given away for nothing.

Three timeouts close hung connections: 15 seconds to receive a request, 30 seconds of inactivity, 5 seconds on keep-alive between requests. A client that opened a connection and went silent is disconnected and does not occupy a slot forever.

#### It tells you what is going on

The `--log FILE` flag enables a journal. It is off by default - a router does not need extra writes to flash. When enabled, the journal writes two things: a line per request in the standard format (client address, time, request, code, size) and error messages with a startup version line. The file is rewritten when it exceeds 256 KB, so the space does not grow without bound.

The `--pidfile FILE` flag writes the process number - startup scripts need it to stop the server cleanly later. `--daemon` sends the server to the background.

#### It answers questions about itself

`whitehttpd --version` prints the version, author, license and project address. `whitehttpd --help` prints full help with launch examples. Every response header carries `Server: whitehttpd/1.0.0`.

#### What it still does not do

An honest list, because it matters more than the list of capabilities.

No TLS - encryption is terminated outside, in `stunnel` or `nginx`. No CGI - nowhere to send forms. No WebSocket, no HTTP/2. No on-the-fly compression. No authentication - if a password is needed, a proxy does it. No `chroot`, no privilege drop. No config file, only command-line arguments. Multiple ranges in one request are not supported, which the standard permits.

This is not a replacement for nginx and not a universal server. It is a static file server that does its job well and predictably.

#### What it looks like in numbers

| | Assembly version | C version |
| :--- | :---: | :---: |
| Size | 4,092 bytes | 17,008 (x86-64) / 20,908 (MIPS) |
| Directory listing | no | yes |
| `Cache-Control` | yes | yes |
| `ETag`, `Last-Modified`, `304` | no | yes |
| `Range` with `206` and `416` | yes | yes |
| `If-Range` | no | yes |
| `OPTIONS` and `405` | no | yes |
| `Date` and `Accept-Ranges` | no | yes |
| Journal | no | yes |
| Index and listing configuration | no | yes |
| PID file | no | yes |
| IPv6 | yes | yes |
| Path protection | yes | yes |
| Runs on kernel 3.3.8 | yes | yes |

The four-to-five-fold difference in size is the price of the list above. Not "sprawl" but work the server began to do.

---

## Act VII. Detective Debugging of the C Server on a Live Router: Linux 3.3.8 Traps

On an Ubuntu x86-64 workstation the C server `httpd.c` built in a fraction of a second and passed every test. But cross-compiling for MIPS32 Big-Endian (`mipsel-linux-gnu-gcc -EB`) and running the binary on an ASUS RT-AC57U V3 running Linux 3.3.8 exposed three fundamental system-level traps.

### 1. The Bus Error mystery (SIGBUS, code 138)

On the very first launch with `--help` the process aborted:

```text
admin@RT-AC57U_V3:/tmp# ./httpd.mipsbe --help
Bus error
```

Disassembling `main` revealed the cause:

```mips
addiu a1, gp, -18404
sw    v0, -18408(gp)
```

When compiled with `-G 65536`, GCC generated accesses to global variables through the **`$gp` (Global Pointer)** register. In ordinary C programs `$gp` is initialized by the `crt1.o` startup code from libc. But in a pure freestanding `nolibc` environment `crt1.o` is absent, and `$gp` holds unaligned garbage.

Unlike x86, the MIPS processor, on an attempt to store a 32-bit word (`sw`) at an address not a multiple of 4, raises a hardware exception and kills the process with `SIGBUS`.

*Fix*: compile the C code with **`-G 0`**, forcing GCC to use direct 32-bit addressing with `lui + addiu` pairs.

### 2. The Linux 4.11 time paradox (`sys_statx` vs `sys_stat64`)

With SIGBUS gone the server ran, but returned `404 Not Found` for every existing file.

The cause was inside `nolibc.h`: the library wrapper for `stat()` unconditionally called **`sys_statx` (number 4366)**, which appeared in Linux only in version 4.11 (2017). On the router's Linux 3.3.8 (2012) the call returned `-ENOSYS` (-38). The assembly `whitehttpd.s` never had the problem, because it used the classic `sys_stat64` from the start.

*Fix*: overriding `stat()` in `wsock.h` via **`sys_stat64` (number 4213)**, supported by all MIPS Linux kernels since 2.4.

### 3. The 2038 bug (`clock_gettime64` vs `gettimeofday`)

The `time()` call in the modern `nolibc.h` went to the 64-bit **`sys_clock_gettime64` (number 4403)**, added in Linux 5.1 to solve the year-2038 problem. On kernel 3.3.8 the call likewise returned `-ENOSYS`, so `time()` returned uninitialized stack values and closed connections early by timeout.

*Fix*: redirecting `time()` in `wsock.h` to the classic **`sys_gettimeofday` (number 4078)**.

After these three fixes in `wsock.h`, the cross-compiler produced a monolithic static executable of **10,532 bytes** (after `strip -s`), working flawlessly on any generation of Linux kernels.

---

## Act VIII. Field Trials and the Price of Checks

### 1. The measurement arsenal and methodology

Final verification was performed directly on the **ASUS RT-AC57U V3** hardware router, through the physical network segment.

1. **`sshpass` + `scp`**: automated delivery of builds into the router's RAM-backed tmpfs (`/tmp`), sparing the NOR flash from wear.
2. **`wrk` (v4.2.0)**: a multi-threaded HTTP load generator for measuring peak RPS, latency and channel throughput.
3. **`curl` (`-i`, `--path-as-is`)**: header validation against [RFC 7230](https://datatracker.ietf.org/doc/html/rfc7230) and [RFC 7233](https://datatracker.ietf.org/doc/html/rfc7233) (Range 206), checking 301 redirects and blocking path traversal attacks (`/../../../../etc/passwd`).
4. **`python3` (raw `socket`)**: testing HTTP pipelining and socket reuse (Keep-Alive).
5. **Kernel telemetry (`netstat -tln`, `ps`, `/proc/$PID/status`)**: monitoring descriptors and measuring real physical memory consumption (`VmHWM`).

### 2. wrk field benchmark results

Both servers were launched on the router with an identical document root `/tmp/www_test`:

- **Port 18081**: the assembly `whitehttpd` (3,944 bytes);
- **Port 18082**: the C server `nolibc_httpd` (10,532 bytes).

#### Test 1. Peak RPS (`index.html` file, 25 bytes, 20 connections, 2 threads)

```text
=== Benchmark 1: whitehttpd (GAS MIPS, port 18081) ===
Running 5s test @ http://192.168.5.220:18081/index.html
  2 threads and 20 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    12.92ms    7.21ms  79.18ms   70.46%
    Req/Sec   794.71    219.57     1.48k    75.00%
  7912 requests in 5.00s, 1.12MB read
Requests/sec:   1581.53
Transfer/sec:    228.58KB

=== Benchmark 2: nolibc_httpd (Freestanding C, port 18082) ===
Running 5s test @ http://192.168.5.220:18082/index.html
  2 threads and 20 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    12.45ms    6.66ms  49.94ms   79.54%
    Req/Sec   825.59    235.14     1.25k    56.00%
  8217 requests in 5.00s, 1.16MB read
Requests/sec:   1642.21
Transfer/sec:    237.35KB
```

#### Test 2. Pushing a heavy file (`big.bin`, 1,048,576 bytes, `sendfile`)

```text
=== Throughput: whitehttpd (GAS MIPS, port 18081) ===
Running 3s test @ http://192.168.5.220:18081/big.bin
  2 threads and 4 connections
  Requests/sec:     10.32
  Transfer/sec:     10.32MB/s (~82.6 Mbit/s)

=== Throughput: nolibc_httpd (Freestanding C, port 18082) ===
Running 3s test @ http://192.168.5.220:18082/big.bin
  2 threads and 4 connections
  Requests/sec:     11.66
  Transfer/sec:     12.15MB/s (~97.2 Mbit/s)
```

**Analysis of the results:**

1. On requests per second (RPS), pure assembly (1,581 RPS) and freestanding C (1,642 RPS) showed full parity. The slight advantage of the C version comes from its 4 preforked processes and non-blocking `EPOLLOUT` resume.
2. On transferring the megabyte file, the `nolibc` server pushed **12.15 MB/s (97.2 Mbit/s of useful traffic)**, fully saturating the router's 100BASE-TX physical channel. The 82.6 Mbit/s figure for the assembly version was obtained over a Wi-Fi bridge (where the 802.11 radio overhead shows); on a direct Ethernet cable the kernel's `sys_sendfile` delivers the same **100 Mbit/s** on both implementations.

### 3. Range header verification (RFC 7233)

Verification of partial-content responses confirmed the precision of the header builder:

```text
$ curl -i -H "Range: bytes=0-4" http://192.168.5.220:18082/hello.txt
HTTP/1.1 206 Partial Content
Server: whitehttpd
Connection: keep-alive
Content-Type: text/plain; charset=utf-8
Content-Length: 5
Content-Range: bytes 0-4/29

Hello
```

The `Content-Range: bytes 0-4/29` header sits strictly inside the HTTP header block, and the body contains exactly the 5 requested bytes (`Hello`).

### 4. The price list of checks

Over the course of the work, exact numbers accumulated: what each check costs. This is the most useful table in the article, because it answers the question that usually goes unanswered.

| Check | Price | Where measured |
| :--- | ---: | :--- |
| Component-by-component `openat` instead of one `open` | **+1,968** | MIPS, 3,944 -> 5,912 |
| Compression after it (dead listing, MIME dedup) | **-1,848** | MIPS, 5,912 -> 4,064 |
| Range per RFC 7233 (3 fixes) + CLI validation | **+16** | MIPS, 4,064 -> 4,080 |
| `Cache-Control` | **+44** | MIPS, 4,096 -> 4,140 (20 code + 28 string) |
| 416 on `first-byte-pos` overflow | **+16** | MIPS |
| Shortening reason-phrases | **-48** | MIPS, 4,140 -> 4,092 (51 bytes of text, 3 eaten by alignment) |
| `openat2` with `RESOLVE_BENEATH` instead of a manual walk | **-140** | x86-64, kernel 5.6+ |
| `Host` + CRLF + framing checks | **0 bytes** | MIPS, no cycle cost found in the final build |

Two rows of this table deserve a separate word.

**`openat2` is 140 bytes cheaper than the manual walk.** On x86-64 with kernel 5.6+ the component-by-component opening loop can be replaced with a single call carrying the `RESOLVE_BENEATH | RESOLVE_NO_SYMLINKS` flags, and the kernel enforces the protection. This is the rare case where the safer solution is also smaller. The price: on kernels older than 5.6 the call returns `ENOSYS` and every request gets 403. That is why the MIPS profile (kernel 3.3.8) keeps the portable component-by-component walk.

**The `Host` check is free in size.** A few string-comparison instructions per request are zero bytes in the binary. In the first safe build the whole validation set cost 38% of throughput, but after the code compression the price vanished: 4,092 performs at the level of 3,944. What eats cycles is not the check as such but its clumsy implementation - and that is worth remembering for anyone who measures only bytes.

### 5. The price is the same in all worlds

Security checks cost roughly the same wherever they are made. This proves the point is not the language but the nature of the checks:

| World | Before audit | After audit | Difference |
| :--- | ---: | ---: | ---: |
| FASM x86-64 | 4,062 | 4,090 | +28 |
| GAS MIPS | 3,944 | **4,092** | +148 |
| nolibc x86-64 | 8,480 | 11,104 | +2,624 |
| nolibc MIPS BE | 10,532 | 13,680 | +3,148 |

Assembly paid 28 bytes on x86-64 (+0.7%) and 148 bytes on MIPS (+3.8%). Freestanding C paid 2,624 and 3,148 bytes (+30-31% over its original size). The difference in absolute numbers is explained by the C version containing more features (listing, IPv6, `--addr`, the non-blocking state machine), and every check applies to more code.

The numbers in the "After audit" column are the state at the moment the vulnerabilities were closed. Afterwards the two products diverged: the assembly version stayed at 4,092 and did not change, while the C version kept developing as a product. The current sizes are in the final comparison below.

---

## The Final Comparison of Worlds

| Characteristic | 1. Classic C | 2. Hand-written FASM | 3. Hand-written GAS MIPS | **4. Freestanding C** |
| :--- | :---: | :---: | :---: | :---: |
| **Directory in the repo** | not in the tree (external `darkhttpd`) | not in the tree (historical stage) | `final-4092/` | `whitehttpd_c/` |
| **Main source file** | `darkhttpd.c` (external) | `whitehttpd.asm` (lost) | `final-4092/whitehttpd.s` | `whitehttpd_c/src/httpd.c` + `wsock.h` |
| **Binary name** | `darkhttpd` (external) | `whitehttpd-fasm-x86_64` (lost) | `whitehttpd-mips32be` | `whitehttpd-nolibc-x86_64`<br>`whitehttpd-nolibc-mips32be`<br>`whitehttpd-nolibc-mips32el` |
| **Server response header** | `Server: darkhttpd` | `Server: whitehttpd` | `Server: whitehttpd` | `Server: whitehttpd/1.0.0` |
| **libc dependencies** | Requires `glibc` / `musl` | **0 (pure kernel)** | **0 (pure kernel)** | **0 (pure kernel)** |
| **On-disk binary size** | 56 KB (x86) / 927 KB (MIPS, static) | **4,090 bytes**<br>(Sub-4KB, x86, historical) | **4,092 bytes**<br>(Sub-4KB, MIPS) | **17.0 KB (x86) / 20.9 KB (MIPS)** |
| **RAM in production** | 128 KB - 916 KB | ~16-20 KB per worker | **28 KB (`VmHWM`)** | **24 KB - 80 KB (per worker)** |
| **Source portability** | High | x86-64 only | MIPS32 BE only | **Absolute (one C source for x86, MIPS, ARM, RISC-V)** |
| **I/O architecture** | Non-blocking `poll` | `epoll` + prefork<br><span class="nobr">(4 workers)</span> | `epoll` (1 process) | **Full `epoll` FSM + prefork (4 workers), pipelining, Range 206, timeouts** |
| **Directory listing** | yes | no (403) | no (403) | **yes, with escaping, disabled by `--no-listing`** |
| **Validators and caching** | `ETag`, `Last-Modified` | no | `Cache-Control` | **`ETag`, `Last-Modified`, `304`, `If-None-Match`, `If-Modified-Since`, `If-Range`, `Cache-Control`** |
| **Logs** | present | no | no | **`--log`, CLF, access + error** |
| **Entry and maintenance threshold** | Low | High | Extreme (`.set noreorder`, `$gp`) | **Low (clean academic C99)** |

*A note on FASM*: the table shows the historical 4,090-byte prototype from Acts I-II. It does not contain the security checks added in Act IV (component-by-component `openat`, strict `Host` validation, CRLF, framing) and has no `Cache-Control`. Porting the final MIPS version to FASM x86-64 is future work. FASM is shown as a historical stage, not a current product.

*A note on repository structure*: the tree holds two products - `final-4092/` (assembly, 4,092 bytes, MIPS32 BE) and `whitehttpd_c/` (freestanding C, three binaries: x86-64 and MIPS32 BE/LE). The intermediate GAS x86-64 port (6,023 bytes) discussed in Acts I-II and the FASM prototype are absent from the tree: they are historical stages whose numbers are preserved in the text as chronicle but can no longer be checked against sources.

---

## Epilogue: The Circle Turned Out to Be a Spiral

The research began with the question of what lies at the very bottom of computer architecture - below the C language.

The descent to pure FASM and GAS assembly proved that a full-featured modern web server with epoll, sendfile, Keep-Alive, IPv6 support and protection against attacks can be packed into **4,092 bytes** - one hardware memory page of a router, consuming 28 KB of RAM by `VmHWM` under load.

But on the way something unexpected happened: the smallest binary turned out to be the most wrong one. The 3,944 bytes were faster and lighter, and at the same time they served files outside the document root, passed CRLF injections and did not validate `Host`. Size and correctness diverged, and a separate act of work was needed to bring them back together.

The half-step ascent back up - to **freestanding C on nolibc and wsock.h** - closed the architectural circle. But it is not the same circle the research started from.

| | Start | End |
| :--- | :---: | :---: |
| Language | C | C |
| libc | yes | no |
| MIPS | 927 KB (static) | 20.9 KB |
| x86-64 | 56 KB | 17.0 KB |
| How we know what is excess | guesses | the descent to the bottom |

The research returned to the C it started from. But it is a different C: the one demanded 927 KB and libc; this one demands 20.9 KB and nothing. The difference came not from the compiler but from the descent to the bottom, from where you can see what can be lived without.

The C version grew from 13.7 to 20.9 KB for a reason. After the audit it stopped being a minimal port and became a product: dotfile blocking, `ETag` with `304`, `If-Modified-Since`, `If-Range`, `Date`, `Accept-Ranges`, `OPTIONS` with `Allow`, `405` instead of `501`, a configurable index, a disableable listing, a PID file, access and error logs, `MAXFD 128` instead of 64. Every item answers a real need, and every item costs bytes. This is not sprawl but the price of the work the server began to do.

The true code-size difference between hand-written reference assembly and an optimizing C compiler without libc is not 30x, as it seems when looking at standard builds, but **5.1x (4,092 bytes versus 20,908 bytes)** on the same MIPS architecture. And that is a fair price for full portability and for a feature set the assembly version does not have.

4,092 bytes proved the limit is reachable. 20,908 proved you do not have to pay it if you intend to live with the code.

The circle turned out to be a spiral: the same point horizontally, a level higher vertically. And now it is visible what lies at the bottom.

---

## Appendix:## Appendix: What Is in the Tree Today

Everything above is chronicle. What follows is the state at the moment of publication, taken from the tree itself rather than from memory. A reader who wants to verify the numbers should look here.

### The assembly profile

| Parameter | Value |
| :--- | ---: |
| Directory | `final-4092/` |
| Source | `whitehttpd.s` (GNU as, MIPS32r2 Big-Endian) |
| Binary | `whitehttpd-mips32be` |
| Size | **4,092 bytes** |
| `Server` header | `whitehttpd` (no version: `/1.0.0` does not fit in the remaining 4 bytes) |
| Kernel requirement | Linux 3.3.8 and later |

The historical 3,944-byte build is absent here and cannot be restored: it read files outside the root through a symlink to an intermediate directory. The 148-byte difference is the price of path validation. The project considers it mandatory.

### The C profiles

| Binary | Architecture | Size |
| :--- | :--- | ---: |
| `whitehttpd-nolibc-x86_64` | x86-64 | **17,008** |
| `whitehttpd-nolibc-mips32be` | MIPS32 BE | **20,892** |
| `whitehttpd-nolibc-mips32el` | MIPS32 LE | **20,908** |

Directory `whitehttpd_c/`, build with `make build-c` (for MIPS - `make build-c-mips`). This is freestanding C: no libc, no heap, every kernel call through the thin `nosyscall` layer.

### Re-measurement on a live router

The old 3,944-byte build survives in `old/whitehttpd_mips/`, so both versions could be placed side by side on one router and measured with one instrument. A caveat worth knowing: the 3,944 build **does not accept** `--port` or `--addr`. With `--port` it listens on loopback only and resets connections; without flags it listens on `8080` on all interfaces. The measurement therefore ran on `8080`.

Conditions: ASUS RT-AC57U V3, Linux 3.3.8, MIPS 74Kc, `wrk -t2 -c50 -d10s` from an external host over the network.

| Scenario | 3,944 bytes | 4,092 bytes | Change |
| :--- | ---: | ---: | ---: |
| Keep-Alive, 16 B file | **1,538 req/s**, 209 KB/s | **1,421 req/s**, 230 KB/s | -8% RPS |
| `Connection: close`, 16 B file | not measured | 520 req/s | - |
| 256 KB file | 33.9 req/s, 8.28 MB/s | **39.3 req/s**, 9.61 MB/s | +16% |

The new build is 8% slower in Keep-Alive and 16% faster on the large file. That is exactly the price described in the security fix section: every request gained `Host` and framing checks plus component-by-component path opening. On a large body, the payoff from the hardened descriptor handling wins instead.

### What is verified on hardware

| Check | Result |
| :--- | :--- |
| GET, HEAD, 404, 403 | correct, body and codes |
| 301 on a directory without a slash | `Location: /space%20dir/` |
| IPv6 dual-stack | `:::port`, requests from `::1` and `127.0.0.1` both served |
| Range | `206` with `Content-Range`, past end of file - `416` |
| Symlink to a file and to a directory | `404` |
| Root escape (`/../etc/passwd`, `/%2e%2e/`) | `400` |
| Directory listing | `403`, absent by contract |
| Dotfiles (`.htpasswd`) | `403` |
| `Transfer-Encoding`, duplicate `Content-Length`, obs-fold, HTTP/1.1 without `Host` | `400` |
| Request header limit | 8,192 bytes, above - `400` |
| Idle connection timeout | 3 s |

### What is not in the public tree

- The FASM implementation (4,062 bytes) and the intermediate GAS x86-64 port (6,023 bytes) from Acts I-II. The sources of these intermediate stages are preserved in the development archives, but not published in the release tree: the public repository focuses on the final, verified profiles.
- The directory listing in the assembly profile.
- The `Date`, `ETag`, `Last-Modified` headers in the assembly profile. It serves `Server`, `Connection`, `Cache-Control`, `Content-Type`, `Content-Length`.

The current reference for the assembly profile: `final-4092/README.md` and the man page `final-4092/whitehttpd.1`.

---

## References and Source Publications

### Base projects and tooling

1. **darkhttpd** - the original minimalist C web server (Emil Mikulic): [https://unix4lyfe.org/darkhttpd/](https://unix4lyfe.org/darkhttpd/)
2. **nolibc.h** - the Freestanding C header from the Linux kernel tree (Willy Tarreau): `tools/include/nolibc/`
3. **wrk** - a modern HTTP benchmark: [https://github.com/wg/wrk](https://github.com/wg/wrk)
4. **Flat Assembler (FASM)**: [https://flatassembler.net/](https://flatassembler.net/)
5. **GNU Binutils (GAS, ld, objcopy)**: [https://www.gnu.org/software/binutils/](https://www.gnu.org/software/binutils/)

### Classic works on extreme ELF binary shrinking (sizecoding)

1. **A Whirlwind Tutorial on Creating Really Teensy ELF Executables for Linux** (Brian Raiter): [https://www.muppetlabs.com/~breadbox/software/tiny/teensy.html](https://www.muppetlabs.com/~breadbox/software/tiny/teensy.html)
2. **ELF: Executable and Linkable Format** - the specification: [https://refspecs.linuxfoundation.org/elf/elf.pdf](https://refspecs.linuxfoundation.org/elf/elf.pdf)
3. **sstrip** - an ELF section table stripping utility: [https://github.com/BR903/ELFkickers](https://github.com/BR903/ELFkickers)

### MIPS architecture, ABI and Linux networking specifications

1. **MIPS32 Architecture For Programmers** - official volumes: [https://www.mips.com/products/architectures/](https://www.mips.com/products/architectures/)
2. **System V Application Binary Interface - MIPS RISC Processor Supplement**: [https://refspecs.linuxfoundation.org/abi/](https://refspecs.linuxfoundation.org/abi/)
3. **Linux MIPS syscall table (O32 ABI)**: `arch/mips/kernel/syscalls/syscall_o32.tbl`
4. **RFC 7230** - HTTP/1.1: Message Syntax and Routing: [https://datatracker.ietf.org/doc/html/rfc7230](https://datatracker.ietf.org/doc/html/rfc7230)
5. **RFC 7233** - HTTP/1.1: Range Requests: [https://datatracker.ietf.org/doc/html/rfc7233](https://datatracker.ietf.org/doc/html/rfc7233)
6. **epoll(7)** - Linux man page: [https://man7.org/linux/man-pages/man7/epoll.7.html](https://man7.org/linux/man-pages/man7/epoll.7.html)
7. **openat2(2)** - Linux man page: [https://man7.org/linux/man-pages/man2/openat2.2.html](https://man7.org/linux/man-pages/man2/openat2.2.html)


*The whitehttpd project is distributed under a dual-licensing model: open-source under AGPLv3 and a commercial OEM license for network hardware manufacturers. Full legal documentation, licensing terms, and the Clean-Room declaration are published in the [official GitHub repository](https://github.com/Svarkovsky/whitehttpd#5-license-and-legal-model).*
