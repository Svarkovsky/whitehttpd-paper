# Beneath C: A 4,092-Byte Web Server, the Price of Hardening, and Returning Without libc

**Author**: Ivan Svarkovsky (<ivansvarkovsky@gmail.com>)
**GitHub**: [https://github.com/Svarkovsky/whitehttpd](https://github.com/Svarkovsky/whitehttpd)
**License**: GNU Affero General Public License v3 ([`SPDX-License-Identifier: AGPL-3.0-or-later`](https://spdx.org/licenses/AGPL-3.0-or-later.html))
**Support / Donations**: [https://svarkovsky.github.io/donate/](https://svarkovsky.github.io/donate/)

---

> **A note to the reader**: this is a long, engineering-heavy article - disassembly, Linux kernel structures, measurements on real hardware.

> **On dates and numbers**: the article is written as a chronicle. The measurements and binary sizes in the text are snapshots taken at the corresponding stage: 3,944 bytes (MIPS, before the security fixes), 4,062 bytes (FASM x86-64, a past branch). Both builds are absent from the current tree. The current state is in the section "Appendix: what is in the tree today" at the end: 4,092 bytes on MIPS and three C profiles. The historical numbers were not rewritten, so as not to destroy the account of how they were obtained.

---

## Prologue: The Illusion of "Portable Assembly" and the Choice of Weapon

By the end of 2026 the industry had fully settled into the idea that a microservice in a container weighs 300 megabytes, and a plain "Hello World" binary eats megabytes of memory at startup. It is taken for granted that memory is cheap, compilers are smarter than humans, and writing low-level code by hand is pointless.

But the picture changes radically when you step down from cloud servers to embedded systems - to the place where hundreds of millions of home routers, IoT devices and industrial controllers run on MIPS and ARM chips with tiny 8-16 MB of flash memory.

C is traditionally called "portable assembly". In practice, however, between a line of C code and the processor's silicon lies a colossal layer of abstractions: calling conventions (ABI), stack frames, locale tables, heap allocators, and the standard library (glibc, musl or uClibc). Even before the first instruction of `main()` executes, the operating system's dynamic loader unfolds a runtime weighing close to a megabyte into memory.

This research begins with a fundamental question: **what lies below C, and what is the real price paid for the abstractions of the standard library?**

The reference testing ground chosen is the well-known minimalist web server **darkhttpd** (written in pure C). The goal of the experiment is to recreate its capabilities in pure assembly, break the physical barrier of a single hardware memory page (Sub-4KB), and then climb back up to academic C, completely free of libc and external dependencies.

The path turns out to be not a straight line but a spiral: starting from C, it will descend to the bottom of assembly, hit a limit from which not a single byte can be squeezed out without losing correctness, and only then return to C - but by a different road.

---

# Part I. The Descent

## Act I. Cold Shower on x86-64: When the Prototype Loses to Architecture

At the first stage, straightforward implementations of the web server were written in **FASM** and **GAS** for Linux x86-64 (through raw `syscall` invocations without a single external library) and set against the original `darkhttpd` in C under load testing with [`wrk`](https://github.com/wg/wrk) (2 threads, 20 concurrent connections).

The results of the first measurement were quite telling:

| Parameter | FASM (v1, raw prototype) | GAS (v1, no -N and sstrip) | Native C (`darkhttpd.c`) |
| :--- | :---: | :---: | :---: |
| **Binary size** | **7.4 KB** | 12 KB | 56 KB (+ `glibc`) |
| **Peak RAM** | **~16 KB** (4 pages)* | **~16 KB*** | 916 KB (`VmHWM` at idle) |
| **RPS (1 KB file)** | 5,118 req/s | 5,695 req/s | **10,403 req/s** |
| **Throughput (256 KB)** | 641 MB/s | 638 MB/s | **1,040 MB/s** |

***A note on re-measured values**: the RPS rows and "C wins" are the original measurements of the first stage. "C wins" in them is really the architecture winning: `darkhttpd` had `poll()`, the prototypes had a linear `accept`. Why this matters - see the conclusions below and the control measurement at the end of the act.*

*\*A note on physical memory*: in superficial measurements the `VmRSS` counter in `/proc/$PID/status` on modern x86-64 kernels may briefly show 4 KB due to deferred batch page allocation (demand paging). The real peak physical memory usage (`VmHWM`) is measured separately under load.

The lesson of the first act was unexpected. The raw assembly prototype **lost** to C by almost a factor of two in speed, while being 7-13 times smaller. And here it is important to name the cause precisely, because it is easy to confuse: it was not the assembly that lost, but the prototype's architecture. It had a linear `accept` loop with blocking reads, which could not hold twenty concurrent connections as effectively as `poll()` inside `darkhttpd`. The language had nothing to do with it: in C with the same linear loop the numbers would have been the same. What `darkhttpd` had going for it was the architecture, not the compiler.

Two conclusions grew out of this, and both defined all further work:

1. **Size by itself means nothing.** A small binary that cannot hold connections is useless. Sub-4KB without multiplexing would not be worth anything.
2. **A language's defeat cannot be blamed on the language.** The first reaction to the table above is "assembly is slow". The control measurement at the end of this act shows otherwise: when both sides have the same architecture, assembly is on par or faster. The rake was the missing `epoll`, and Act II is about stepping off it.

### Control measurement of the final builds

The table above is the first measurement of raw prototypes. When the article was being prepared for publication, all three implementations were rebuilt and measured under identical conditions: `wrk -t2 -c50 -d10s`, keep-alive, 1 KB file, Intel Core i5-3230M, Linux 6.1.0. `darkhttpd` 1.17 was built with `cc -O2` from the sources in `old/darkhttpd_c/`. Each server was run alone, so the measurements did not compete for cores.

| Implementation | Size | RPS (1 KB) | RPS (256 KB) | RPS (16 B, close) |
| :--- | ---: | ---: | ---: | ---: |
| `darkhttpd` 1.17, C + glibc | 56,256 B | **18,000** (17.7-19.2k) | 2.13 GB/s, 9,131 req/s | 4,039 |
| C profile without libc | 17,008 B | **41,733*** (24.6-43.7k) | 1.12 GB/s, 4,694 req/s | - |
| FASM x86-64 | 4,062 B | **23,000-44,000**** | 1.22 GB/s, 5,126 req/s | 830 |

\*The C profile gave 43,735, 41,733, 40,844, 43,494 req/s in four of five runs and 24,603 in one; the median of five is 41,733. The first run on a fresh process is systematically low.
\**FASM does a prefork of 4 workers, which compete with `wrk` itself for the host's 4 cores. The spread of 23-44k with a median of 36k is the scheduler, not code instability. On a host with more cores the spread disappears.

The control measurement closes two questions left by the first act:

1. **The defeat of Act I was a defeat of architecture, not language.** On the small file FASM is twice as fast as `darkhttpd`, and the C profile without libc is on par or faster. When both sides have `epoll`, assembly does not fall behind: the gap from the first table disappeared together with the linear `accept` loop.
2. **The size difference does not produce a speed difference on a small file.** 4,062 and 56,256 bytes on a 1 KB file hit the same wall - the cost of system calls and parsing the request line. On the large file `darkhttpd` is faster: 2.13 GB/s versus 1.22, because it makes fewer system calls per byte of payload. Size wins not in speed but in memory: 916 KB of RAM for `darkhttpd` versus tens of kilobytes for assembly, and on embedded hardware with 128 MB of RAM that matters more than any throughput percentages.

The lesson of the first act was unexpected - but it was a lesson about architecture, not about languages.

---

## Act II. Triumph on x86-64: Sizecoding, epoll, prefork and TCP_NODELAY

If the diagnosis from Act I is correct and the defeat was in the architecture, then the cure is architecture, not rewriting the loop in a different flavor of assembly instructions. That is what was done: the same mechanisms `darkhttpd` stood on were ported into the assembly implementations, plus two it did not have. Four key changes:

1. **Sizecoding and a single RWX segment**: all uninitialized buffers moved out of the data section into `.bss` (`rb` in FASM), which zeroed their on-disk size. The linker flag `ld -N` removed the 4 KB inter-section page alignment, and stripping the section table (`sstrip`) compressed the binary to 3.9 KB, fitting it entirely into the processor's level-1 instruction cache (L1i).
2. **Hardware prefork in 14 bytes of code**: two consecutive `sys_fork` calls (syscall 57 in the x86-64 ABI) right after `sys_listen` split the server process into 4 parallel workers matching the CPU core count, without the pthread library. Memory works in Copy-on-Write mode (~16-20 KB of physical RAM per worker: the code page, the stack page, and the request buffer pages).
3. **The kernel multiplexer `sys_epoll`**: replacing the linear accept loop with the kernel's non-blocking event queue - raw syscalls `epoll_create1` (291 in the x86-64 ABI), `epoll_ctl` (233) and `epoll_wait` (232), keeping Keep-Alive sessions open.
4. **Removing Nagle's handbrake (`TCP_NODELAY`)**: disabling Nagle's algorithm via `setsockopt` eliminated the 40-millisecond TCP Delayed ACK penalty when sending HTTP headers (`write`) and the file body (`sendfile`) separately.

The control test results on x86-64 (`wrk`, keep-alive on):

| Parameter | FASM<br><span class="nobr">(`epoll` + prefork)</span> | Native C<br><span class="nobr">(`darkhttpd`)</span> | Result |
| :--- | :---: | :---: | :---: |
| **RPS (1 KB file)**, original measurement | **27,719 req/s** | 11,077 req/s | **Assembly 2.5x faster (+150%)** |
| **Median latency** | **473 us (0.47 ms)** | 1.50 ms | **3.1x more responsive** |
| **Throughput (4.4 KB file), re-measured** | **120.9 MB/s** (27,372 req/s) | 43.0 MB/s (9,495 req/s) | **Assembly 2.8x faster** |
| **Binary size** | **4,062 bytes (Sub-4KB!)**, FASM, historical stage | 56 KB | **13.8x smaller** |
| **RAM (Peak RSS)** | **~16-20 KB (per worker)** | 916 KB (`VmHWM`) | **45-55x less memory** |

Here assembly took its revenge: 27,719 requests per second against 11,077 for C, with three times lower latency and a fourteen times smaller size. The Sub-4KB barrier was taken on x86-64.

On re-measurement the gap narrowed. `darkhttpd` 1.17 on the same host in the same mode gives 17.7-19.2k req/s, FASM - from 23 to 44k depending on how the scheduler divides the 4 cores between the server's 4 workers and the `wrk` threads. That is, assembly is faster, but by 1.3-2.3x, not 2.5x. The value of 11,077 came out lower because in that measurement `darkhttpd` ran under different conditions. Both numbers are kept: the original as chronicle, the re-measured as verifiable.

But x86-64 is a server in a VM. The real test waited on real silicon.

---

## Act III. Moving to Real Silicon: MIPS32 Big-Endian and the Battle for Sub-4KB

After the concept was proven on x86-64, development moved to the hardware platform of the home router **ASUS RT-AC57U V3** (Qualcomm Atheros QCA9563 SoC, MIPS 74Kc Big-Endian core, Linux 3.3.8). On this architecture the original statically linked C `darkhttpd` with glibc weighs 927 KB (`mipsel-linux-gnu-gcc -static -Os -s`); the figure of 119.2 KB from an early draft of this article does not reproduce with any available library and has been replaced here with the measured value. The dynamically linked variant takes 56 KB on x86-64 but requires `glibc` in the router firmware, which is not guaranteed on OpenWrt or Padavan. It is precisely the size of the static build that is the reason writing without libc makes sense on MIPS.

For MIPS the only real tool is the **GNU Assembler (GAS)**. The project was named **`whitehttpd`**.

The first working port in GAS MIPS32 showed a size of 6,608 bytes. The task was set: break the physical barrier of **4,096 bytes** (Sub-4KB - exactly one hardware page of virtual memory and one TLB entry), preserving all of the functionality of that moment: IPv6/IPv4 dual-stack, epoll, sendfile, Keep-Alive, urldecode, 301 redirects and the dynamic HTML directory listing (`sys_getdents64`). The listing would be removed later, when the budget ran out (see the section on the security fix).

### Four steps of compression to 3,944 bytes

1. **The `$gp` register and the 64 KB barrier (6,608 -> 6,072 B)**: in the MIPS RISC architecture every instruction weighs 4 bytes, so loading a 32-bit address with the `la` pseudo-instruction expands into a `lui + addiu` pair (8 bytes). Placing string constants and small buffers (< 18 KB) in the `.sdata` and `.sbss` sections made it possible to replace two-word loads with single 4-byte instructions with an offset from the Global Pointer register (`%gp_rel`), saving 536 bytes. The large buffers (`dents_buf` at 8 KB and `page_buf` at 64 KB) were moved to `.bss` and addressed through a single base register `$s7`.
2. **The `.set noreorder` pipeline and the end-to-end `$a0` chain (6,072 -> 4,616 B)**: manually placing useful instructions into the hardware branch delay slots freed the code from automatic `nop` padding. At the same time `strcpy` was redesigned so that after copying, the `$a0` register pointed exactly at the written terminal `\0`, and `uint_to_str` ended with a tail jump into `strcpy`. This made it possible to assemble complex HTTP headers in a streaming fashion, completely eliminating the `strlen` and `strcat` functions.
3. **"Brainstorm Ultra" structural synthesis (4,616 -> 4,344 B)**: merging the `sigpipe_act` structure (used once at startup), the socket constants `opt_zero` (0) and `opt_val` (1), and the `epoll_event` structure template (with the `EPOLLIN = 1` flag pre-written) into a single 24-byte data array, plus deduplicating the `; charset=utf-8` suffix in the MIME table.
4. **ELF header cleanup, in-place decoding and elimination of stack frames (4,344 -> 3,944 B)**: pre-removing the `.MIPS.abiflags` and `.reginfo` service sections from the object file with `objcopy -R`, cutting off the section table (`sstrip`), removing stack frames in `get_mime_type` and `send_error`, replacing `print_str` with direct streaming assembly, and inlining URL decoding directly into the `path_buf` buffer.

The executable took the barrier at **3,944 bytes** on disk and **20 KB of physical RAM** in production mode (while serving a WebGL 3D application through `sendfile`, thanks to the kernel's lazy demand paging). On a 20-thread stress test the server showed 309 RPS (8.71 MB/s of payload), hitting the physical limit of the router's 100-megabit port.

That measurement concerns serving a large body. On the re-shoot while preparing this article the same build on a small file gave something else entirely: 1,538 req/s in Keep-Alive mode and 208 KB/s on a 16-byte file. There is no contradiction - these are different questions. 309 RPS at 8.71 MB/s measures how many bytes the network passes, and the bottleneck there is the port. 1,538 RPS on 16 bytes measures how many requests the processor can parse, and the bottleneck there is the server itself. Both numbers are correct and do not contradict each other.

3,944 bytes. One memory page. Twenty kilobytes of RAM. The server holds epoll, IPv6, Keep-Alive, sendfile, the directory listing and 301 redirects. The listing would vanish a few months later, when the symlink root escape came to light; the reader will learn of it in the section on the price of security.

It seemed the goal was reached and all that remained was to write the article about it.
