<!-- Copied from Kyogre's specification (docs/compiler-research/kyogre/stdlib.md at 6ba1e99). Relative links point into that repository and don't resolve here. -->

# Primal C: Standard Library Specification

Nov 23, 2026 · Draft

The Primal C standard library is C's library, made consistent and pleasant to use: the ISO C functions under `std::` namespaces, the widely adopted POSIX and BSD functions C programmers already reach for, and a handful of new ones where C has a gap every program fills by hand. It adds no new programming model.

It is written in Primal C and serves every unit Kyogre compiles. Primal C units call it through `std::`; C units of every dialect, from K&R to C2y and the GNU modes, call it through its C face, the root C names with their own headers, under their own data model. Whatever glibc and Microsoft's C runtime provide that real programs use, the C face provides too, with compatibility shims where Kyogre's design differs (section 18).

This document extends [`lang.md`](lang.md) §13, which maps each ISO C header to a `std::` module and states the library-wide rules. It chooses the implementations, lists what each module adds, and proposes the changes to `lang.md` that the choices need (section 17). The research behind every choice is in [`../stdlib/`](../stdlib/README.md), one file per component. Where a choice is still open, this spec states a **suggested default**; section 16 collects them.

## 1. Principles

**The prime rule.** Primal C adds no new programming model to C. Namespaces are its one new feature; its other changes spell out C's own evolution (keywords in place of the `_Keyword` spellings, C2y's features) and define what C left undefined. The library follows the same rule:

- **It is C.** Functions, structs, enums and `const` objects. No generics, no implicit context, no allocator objects threaded through calls, no result or option types, no string type with methods. Errors are a return value plus `errno`, as in C. A program written against `std::` reads like a C program with `io::` in front of `printf`.
- **Precedent first.** Every addition is taken, in order of preference, from ISO C23, accepted C2y features, POSIX.1-2024, or functions in wide use across the BSDs, glibc and musl. A function with no precedent is marked **new** here, and exists only where every nontrivial C program otherwise writes its own (a hash table, a seeded generator, UTF-8 decoding).
- **The fastest and leanest available.** Each component follows the best published design for CPU time, peak memory, allocation count, code size and start-up, and is measured against glibc 2.44 and musl (section 15).
- **The same results everywhere.** Algorithms whose output is visible are pinned by this spec: `std::`'s `rand`, the hash function, table iteration order, number conversion and the math functions give the same bits on every target and in both `libc` modes (section 2). `long double` math is the one stated exception, because its format differs by target (section 9). The C face reproduces the reference C library instead where C programs depend on its exact output, as `rand`'s sequence (section 18.2).

**Design rules.** These hold for every module.

1. **Defined for every input** (`lang.md` §13): a function gives a stated result or traps deterministically; preconditions checkable in O(1) are always checked; overlapping arguments act as if copied through a temporary.
2. **No hidden work.** No constructors, no start-up initialization, no hidden threads. A stateful component initializes from zeroed storage on first use. A program that doesn't call a function doesn't link it.
3. **No hidden allocation.** A function allocates only when allocating is its job (`malloc`, `strdup`, `asprintf`, `getline`, `open_memstream`, `table::insert`, `mergesort`). Sorting, formatting, number conversion and hashing use bounded stack space.
4. **Caller-sized output.** A function that writes into a caller's buffer takes its size and returns the length it would have written, as `snprintf` and `strlcpy` do; a result that doesn't fit is truncated and terminated, never overflowed.
5. **State is explicit or documented.** Every piece of library state is either in a caller's struct (`struct rand::state`, `struct table::table`, `struct arena::arena`) or one of a few documented globals (`errno`, the standard streams, `rand`'s seed, the environment, `getopt`'s cursor).
6. **Type-generic means builtin or type-erased.** Macros never cross `#import` (`lang.md` §10.4), so a type-generic entry is a compiler builtin, as `<tgmath.h>`, `ckd_*` and `stdc_*` already are, or a function over bytes with an element size and callbacks, as `qsort` is. The library adds no other kind.

**Naming.** ISO C, POSIX and BSD functions keep their C names in the module of their header (`str::strlcpy`, `lib::reallocarray`): those names are the domain's vocabulary. A new family gets its own module, drops the prefix the namespace already supplies, and names what each function does in plain English, not how it does it (`hash::bytes`, `hash::keyed`, `table::find`, `table::remove`, `rand::split`, `arena::alloc`). An algorithm's name appears in the documentation, not the API, so the algorithm can change without renaming anything.

## 2. Library modes

A product's `libc` (`frontend-plan.md` §9) decides who provides the C library. The `std::` interface is identical in every mode.

**2.1 `libc = "kyogre"`.** The standard library is the C library: start-up, TLS, `errno`, `setjmp`, the allocator, streams and threads, for Primal C units and C units alike. Each function has one implementation. Its `std::` name (`std.io.printf`) and its root C name (`printf`) are aliases of the same code, except where the C face needs a thin entry point of its own: the functions whose meaning depends on the data model (`%lld`, `strtoll`, `llabs`, `wchar_t`, `intmax_t`), the per-standard variants (C23's `strtol` reads `0b` prefixes, C17's doesn't), and the C-only functions and compatibility names of section 18.

**2.2 `libc = "system"`.** `std::` runs beside the platform C library:

- **Stateless modules are Kyogre's own in both modes**: `std::str`, `std::ctype`, `std::math`, `std::bit`, `std::ckd`, `std::complex`, `std::decimal`, `std::fp`, `std::bigint`, `std::uchar`, `std::wchar`, `std::hash`, `std::table`, `std::rand`, the sorting and number-conversion parts of `std::lib`, and the formatting engines of `std::io`. So `std.str.memcpy` and `std.math.sin` give the same results whatever library the product links.
- **The allocator is Kyogre's in both modes** (suggested default). The product defines `malloc` and its family process-wide, so `std::`, C units and the platform library's own allocations share one heap that can enforce `lang.md` §13's trap on invalid frees. A product that can't interpose (a plugin loaded into a host process) sets `allocator = "system"`: `std::lib`'s allocation functions then call the platform's, and trap only where it does. Section 5.3 gives the reasons.
- **Other stateful modules wrap the platform's public functions**, never its internals: `std::io::FILE` is the platform's `FILE` (one type, so C units and `std::` share streams), `errno` is the platform's, threads are the platform's `pthread_create` with Kyogre's own synchronization objects over `futex`, and clocks, signals and `setjmp` are the platform's.
- **What can't hold over a platform `FILE`** (section 7.4) is a documented difference named by library and version, as `AGENTS.md` allows for outside tools.

**2.3 `libc = "none"`.** Freestanding products get the stateless modules that need no OS (`std::str`, `std::ctype`, `std::math`, `std::bit`, `std::ckd`, `std::hash`, `std::table` without allocation, `std::rand`, sorting and number conversion) and nothing else.

**2.4 Layers.** Dependencies run one way, as the back end's stages do:

```
start          _start, auxv, static TLS, constructors, exit            (libc = "kyogre" only)
std::sys       raw system calls per target; returns -errno, never touches errno
stateless      str, ctype, math, bit, ckd, complex, fp, bigint, decimal, uchar, wchar,
               hash, table, rand; lib's sorting and conversion; io's engines
stateful       lib (allocator, exit, environment), io (FILE), err, thread, time, signal, jmp, fenv, prog
root C face    aliases, data-model and standard variants, C-only functions (libc = "kyogre" only)
compat faces   glibc's internal ABI names; Microsoft's C runtime names (section 18)
```

`start` and `std::sys` depend on nothing in `std::`; stateless modules depend only on the prelude and each other; stateful modules depend on `std::sys`; the C face depends on `std::`, and the compatibility faces on the C face; nothing depends on a face. `std::sys` is internal: it is the narrow seam a new OS ports, as mlibc's `sysdeps` is. It takes and returns UTF-8 strings and returns `-errno` values from one Kyogre table, so a Windows `std::sys` converts to UTF-16 and maps `GetLastError` at the seam and nothing above it changes (section 18.5).

**2.5 Start-up** (`libc = "kyogre"`). After the static-PIE self-relocator (`codegen.md` §8), `_start` passes the stack pointer to the library's entry. It reads `argc`, `argv`, `envp` and the auxiliary vector entries it needs (page size, `AT_RANDOM`, program headers, hardware capabilities, and the vDSO's address, recorded but not parsed); sets up static TLS and the thread pointer when the image has a `PT_TLS` segment; runs `.preinit_array` and `.init_array`; calls `main`; and calls `exit`. Nothing else runs: no stdio, allocator, locale or thread-list setup. Stream flushing at exit is a `.fini_array` entry contributed by the stream objects, so a program without stdio has none. Research: [libc architecture](../stdlib/libc-architecture.md).

## 3. Modules

| Module | From | Adds beyond ISO C (precedent) | Section |
| --- | --- | --- | --- |
| `std::str` | `<string.h>`, `<strings.h>` | `strnlen`, `stpcpy`, `stpncpy`, `strlcpy`, `strlcat`, `memmem`, `strtok_r` (POSIX 2024); `strsep` (BSD); `strchrnul`, `mempcpy`, `memrchr` (glibc, musl); `strcasecmp`, `strncasecmp` (POSIX `<strings.h>`) | 4 |
| `std::lib` | `<stdlib.h>` | `reallocarray`, `getentropy`, `qsort_r`, `setenv`, `unsetenv` (POSIX 2024); `arc4random` family (OpenBSD, glibc 2.36); `malloc_usable_size`, `malloc_trim` (glibc); `mergesort` (BSD); `bsearch_r`, `lower_bound`, `upper_bound`, `growarray`, `strshortest*`, `strfromll`, `strfromull`, `strntod`, `strntoll` (new) | 5, 6, 8 |
| `std::arena` | none | Region allocation (new; GNU obstack and Hanson's arenas) | 5.4 |
| `std::io` | `<stdio.h>` | `getline`, `getdelim`, `dprintf`, `asprintf`, `vasprintf`, `fdopen`, `fileno`, `open_memstream`, `fmemopen`, `flockfile` and the `*_unlocked` functions (POSIX 2024); `fopencookie` (glibc, musl, FreeBSD); the `%m` conversion (glibc, musl) | 7 |
| `std::math`, `std::fenv`, `std::complex` | `<math.h>`, `<fenv.h>`, `<complex.h>` | Correctly rounded functions under the plain names; `lgamma_r` (glibc, BSD); `feenableexcept` family (glibc, BSD) | 9 |
| `std::hash` | none | `bytes`, `integer`, `keyed` (new) | 10.1 |
| `std::table` | none | A hash table over bytes, shaped like `qsort` (new; replaces `hsearch`) | 10.2 |
| `std::uchar` | `<uchar.h>` | `utf8_valid_length`, `utf8_decode`, `utf8_encode`, `utf8_count` (new; Plan 9 `utf.h`); C2y `<stdmchar.h>` once published | 11 |
| `std::thread` | `<threads.h>` | Monotonic-clock waits (POSIX 2024 `*clock*` functions); read-write locks, semaphores, barriers (POSIX); attributes, names, affinity, CPU count (glibc) | 12 |
| `std::time` | `<time.h>` | `clock_gettime`, `clock_getres`, `clock_nanosleep`, `nanosleep`, `gmtime_r`, `localtime_r`, `strptime` (POSIX) | 13.2 |
| `std::rand` | none | A seeded generator on caller state (new; xoshiro256++) | 13.1 |
| `std::program` | none | `getprogname`, `setprogname`, the `err`/`warn` family (BSD); `getopt`, `getopt_long` (POSIX, GNU) | 14 |
| `std::ctype`, `std::err`, `std::limits`, `std::bit`, `std::ckd`, `std::signal`, `std::jmp`, `std::atomics`, `std::wchar`, `std::bigint`, `std::decimal`, `std::fp` | as `lang.md` §13 | Nothing beyond `lang.md` §13 in this draft | |

Declarations below use the prelude's `size_t` and Primal C's types (`lang.md` §5.1): `unsigned` is 32 bits, `long` and `unsigned long` are 64, `long long` is 128. They omit `restrict` where C has it; the library keeps C's `restrict` qualifiers, which in Primal C grant only what `lang.md` §11.1 says.

## 4. Memory and strings: `std::str`

**4.1 Contents.** All of C23's `<string.h>`, including `strdup`, `strndup`, `memccpy` and `memset_explicit`, plus:

```c
size_t strnlen(const char *s, size_t n);                          // POSIX
char  *stpcpy(char *d, const char *s);                            // POSIX
char  *stpncpy(char *d, const char *s, size_t n);                 // POSIX
size_t strlcpy(char *d, const char *s, size_t size);              // POSIX 2024, OpenBSD
size_t strlcat(char *d, const char *s, size_t size);              // POSIX 2024, OpenBSD
void  *memmem(const void *h, size_t hn, const void *n, size_t nn);// POSIX 2024
char  *strtok_r(char *s, const char *delim, char **save);         // POSIX
char  *strsep(char **sp, const char *delim);                      // BSD; C2y proposal N3667
char  *strchrnul(const char *s, int c);                           // glibc, musl
void  *mempcpy(void *d, const void *s, size_t n);                 // glibc, musl; C2y proposal N3662
void  *memrchr(const void *s, int c, size_t n);                   // glibc, musl, FreeBSD
int    strcasecmp(const char *a, const char *b);                  // POSIX <strings.h>, ASCII only
int    strncasecmp(const char *a, const char *b, size_t n);       // POSIX <strings.h>, ASCII only
```

Not included: `rawmemchr` (it can't be bounded), `strverscmp`, `strfry`, `strcasestr`, `explicit_bzero` (`memset_explicit` is the one spelling), and the Annex K `_s` functions.

**4.2 Length-carrying APIs.** `lang.md` §15 replaces Annex K with "length-carrying APIs". They are the functions above that take a size (`strlcpy`, `strlcat`, `snprintf`, `strnlen`, `memccpy`, `memmem`), together with the compiler's checking of literal formats and known destination sizes (`lang.md` §13). There is no string type and no view struct: a counted string is a pointer and a length, passed as two arguments, as `memchr` and `memmem` already take them, and printed with `%.*s`.

**4.3 Rules.**

- **Null with length 0** is a no-op for every function taking a length (`memcmp` returns 0, `memchr` returns null, `strnlen` returns 0, `strlcpy` with size 0 returns the source length). Null where a string must be read, or with a nonzero length, traps.
- **Overlap.** `memcpy` is `memmove`. The string copies (`strcpy`, `stpcpy`, `strcat`, the `n` and `l` forms, `memccpy`, `mempcpy`, `strxfrm`) measure the source, then move, so overlap acts as through a temporary.
- **Truncation.** `strlcpy` writes `size − 1` bytes and a null and returns the source's full length; it reads the whole source, as POSIX defines.
- **`strtok`** keeps its position per thread, sharing `strtok_r`'s code.
- **`memcmp`, `strcmp` and `strncmp`** return −1, 0 or 1 (suggested default), so results don't depend on the implementation's byte loop.

**4.4 Implementation.** The copy kernel follows glibc's `memmove` structure, reimplemented (glibc is LGPL): overlapping head and tail loads for small sizes, every load before any store up to eight vectors, one `dst − src` compare to pick a direction above that, and `rep movsb` and non-temporal thresholds kept as data per microarchitecture. AArch64 follows Arm optimized-routines' `memcpy` (MIT/Apache-2.0, adoptable), which already handles overlap from one entry, with the MOPS instructions where the CPU has them. Making `memcpy` overlap-safe therefore costs the library almost nothing; the cost is the optimizer's lost non-overlap fact (`ub.md`). `strstr` and `memmem` use short-needle vector filters, then Two-Way, for a linear-time guarantee. `strspn` and `strcspn` use a 256-bit set.

Each target starts with two kernel tiers (x86-64: SSE2 and AVX2 or EVEX256; AArch64: NEON and MOPS), chosen once through a function-pointer table that defaults to the baseline kernel, so a static PIE needs no `IRELATIVE` pass and `libc = "none"` needs no initialization (suggested default). A third tier is added only on a measured win; FreeBSD found AVX2 and AVX-512 `strlen` kernels slower than SSE2 on short strings.

**Reading past the end.** Fast string kernels read whole aligned vectors, which may include bytes past the terminator within the same page. That can't fault, but it touches bytes the caller doesn't own. Suggested default: library kernels may do this, sanitizer builds use masked or byte-exact kernels (AOCL-LibMem's strict-bounds option is the precedent), and Kyogre's KASAN and ASan instrumentation checks the call's declared range at the call. Research: [memory and string primitives](../stdlib/memory-string-primitives.md), [text and strings](../stdlib/text-and-strings.md).

## 5. Allocation: `std::lib` and `std::arena`

**5.1 Contents.**

```c
void  *malloc(size_t size);                                       // ISO C
void  *calloc(size_t n, size_t size);                             // ISO C
void  *realloc(void *p, size_t size);                             // ISO C
void  *reallocarray(void *p, size_t n, size_t size);              // POSIX 2024
void   free(void *p);                                             // ISO C
void   free_sized(void *p, size_t size);                          // C23; size ignored
void   free_aligned_sized(void *p, size_t align, size_t size);    // C23; both ignored
void  *aligned_alloc(size_t align, size_t size);                  // ISO C
int    posix_memalign(void **out, size_t align, size_t size);     // POSIX
size_t memalignment(const void *p);                               // C23
size_t malloc_usable_size(const void *p);                         // glibc; traps on a foreign pointer
int    malloc_trim(size_t keep);                                  // glibc; returns free pages now
inline void *growarray(void *p, size_t *cap, size_t need, size_t esize);   // new
```

`growarray` is the growable array without a type: it returns `p` when `*cap >= need`, otherwise reallocates to `max(need, 2 × *cap, 4)` elements with checked arithmetic and updates `*cap`; on failure it returns null and leaves `p` and `*cap` alone. With `reallocarray`, it is all a dynamic array needs; a struct of pointer, length and capacity is the user's own. Failures return null with `errno` set to `ENOMEM`, including sizes over `PTRDIFF_MAX`. There is no `mallinfo`, `mallopt`, `memalign`, `valloc` or `pvalloc`.

**5.2 The allocator.** Kyogre writes its own (suggested default). Every general-purpose allocator surveyed either can't tell its own pointers from foreign ones (jemalloc, rpmalloc), checks only in debug builds (jemalloc), detects double frees probabilistically (snmalloc, scudo's checksums), or detects a bad free and ignores it (mimalloc at `MI_SECURE` 2 and above, unless a handler aborts). Only hardened_malloc and musl's mallocng enforce `lang.md` §13, and both are slower than the per-thread designs. The design combines them:

- **Ownership by address.** At first use, one `PROT_NONE` reservation holds every small and medium block, so `base ≤ p < end` decides ownership; metadata lives in a second reserved range indexed by the block's 64 KiB page. There are no in-band headers.
- **Bitmap slabs.** Size classes of 16, 32, 48, 64 bytes, then four per doubling up to about 32 KiB (internal fragmentation under 20%), medium spans to 1 MiB, page runs above that, and separately mapped huge blocks in a small table. A block's state is one bit, so `free` is a range check, an index computation and a bit test-and-set: a second free finds the bit already clear and traps.
- **Per-thread heaps** with one-atomic remote frees, as in mimalloc and snmalloc: a free from another thread is one `fetch_or` on the page's remote word, and the owner merges remote bits when it runs out. A cross-thread double free of a block already back in the owner's local bitmap is caught at that merge, so the trap comes at the second `free` or, at the latest, when the owner reclaims the page (section 17).
- **Zero-size allocations** come from a never-mapped sub-range: each is a unique non-null address that costs one metadata bit, and any access faults.
- **Page return** is opportunistic, on slow paths: pages free for about a second are released with `MADV_DONTNEED`, in 2 MiB-aligned runs when transparent huge pages are on, and `malloc_trim` releases them at once.
- **Reentrancy.** A per-thread byte marks the allocator in use; a signal handler that enters it on the same thread traps (`lang.md` §11.5). POSIX.1-2024 doesn't list the allocation functions as async-signal-safe, so this turns POSIX's undefined case into a trap rather than conflicting with it.
- **Fork** takes the few global locks (page runs, huge table, heap registry) in the prepare step; the child keeps only the forking thread's heap.

The estimated size is 3,000 to 5,000 lines of C, with no start-up cost beyond one reservation on first use. The open alternative is mimalloc at `MI_SECURE` 2 with an aborting error handler plus a double-free bitmap: less code to write, at a cost to be measured (section 16). Research: [allocator](../stdlib/allocator.md).

**5.3 Why the allocator interposes in system mode.** Over glibc's `malloc`, `std::` can't detect a foreign or double free without either glibc's heuristics (incomplete), a shadow table of every live block (tens of nanoseconds per call and a second allocator), or in-band headers (16 bytes per block and still probabilistic). Defining `malloc` process-wide, which glibc's manual documents as supported, is the only option that meets §13 at full speed, and it removes the question of whose heap a pointer belongs to. Its costs: a glibc change to internals that assume its own allocator can break it (mimalloc has met one), and sanitizer runtimes, which also interpose `malloc`, replace it in sanitizer builds.

**5.4 Regions: `std::arena`.** A bump allocator for data freed together, which turns many `malloc` calls into pointer additions and one `free` per chunk. It is shaped after GNU obstack's free-to-a-mark and Hanson's arenas (*C Interfaces and Implementations*, ch. 6), minus their macros, global failure handlers and exceptions:

```c
struct arena;                                                      // opaque
struct mark { void *chunk; size_t offset; };                       // a position, by value
struct arena *create(size_t chunk_size);                           // 0 picks the default (64 KiB, doubling)
void *alloc(struct arena *a, size_t size);                         // 16-byte aligned; size 0 gives a unique pointer
void *calloc(struct arena *a, size_t n, size_t size);              // zeroed; overflow gives null and ENOMEM
void *aligned_alloc(struct arena *a, size_t align, size_t size);
struct mark mark(struct arena *a);
void  release(struct arena *a, struct mark m);                     // frees everything allocated after m; a foreign mark traps
void  reset(struct arena *a);                                      // frees everything, keeps one chunk
void  destroy(struct arena *a);                                    // null is a no-op
```

An arena belongs to one thread and has no locks. `alloc` is an inline align, compare and add. Memory released to a mark stays in a live chunk, so a stale pointer reads old bytes, as a freed `malloc` block does (`lang.md` §11.2). Chunks come from the allocator's page runs, so destroying an arena twice traps through `free`'s own rule. There is no arena `realloc`, no cleanup callbacks and no hierarchy (talloc and APR pools are the counter-examples: a second memory model).

## 6. Number conversion

One internal engine serves `printf`, `scanf`, `strto*`, `strfrom*` and `ato*`, in both name faces. Results are exact and identical on every target. Research: [number conversion](../stdlib/number-conversion.md).

**6.1 Engine.**

- **Shortest round trip** (`float`, `double`): a Schubfach-class algorithm, porting zmij's C code (MIT) or reimplementing Schubfach or Dragonbox from their papers. One table of 128-bit powers of ten, about 10.4 KB, serves both types. The JDK's Schubfach is GPL with the Classpath exception, so it is a reference only.
- **Fixed precision** (`%e`, `%f`, `%g`): the shortest engine's fast path up to 17 significant digits, then an exact base-10⁹ bignum on the stack in musl's shape: 512 bytes for `double`, about 7.4 KB for `long double`. Ryu Printf's 105 KB of tables is not worth the speed at large precisions. Output beyond `INT_MAX` characters fails with `EOVERFLOW` instead of allocating.
- **Hexadecimal** (`%a`) is exact bit manipulation.
- **Parsing**: Clinger's fast path, then Eisel–Lemire (about 10.4 KB of tables, possibly the printer's, section 16), then an exact bounded-stack fallback: either fast_float's 4,000-bit integer or llvm-libc's 800-digit decimal (section 16). The fast paths run only in round-to-nearest.
- **Integers**: digit-pair formatting in jeaiii's style up to 64 bits, with 128-bit values split at 10¹⁹; parsing eight digits at a time and `ckd_*` overflow checks.
- **`long double`** (x87 80-bit, binary128) takes the exact paths for both directions.

**6.2 Rounding mode.** Library conversions honour the dynamic rounding mode, as glibc's and llvm-libc's do (suggested default); musl and fast_float don't. The check is one control-register read per call.

**6.3 Contents.** C's `strto*`, `ato*` and C23's `strfromf`, `strfromd` and `strfroml`, with `lang.md` §13's saturation rules, plus:

```c
size_t strshortestf(char *buf, float x);         // new; buf ≥ SHORTEST_MAX_F, no terminator, returns the length
size_t strshortestd(char *buf, double x);        // new; buf ≥ SHORTEST_MAX_D
size_t strshortestl(char *buf, long double x);   // new; buf ≥ SHORTEST_MAX_L
size_t strfromll(char *buf, long long x);        // new; buf ≥ 40
size_t strfromull(char *buf, unsigned long long x);   // new; buf ≥ 40
double    strntod(const char *s, size_t n, char **end);             // new; input need not be terminated
long long strntoll(const char *s, size_t n, char **end, int base);  // new
```

C has no way to ask for the shortest representation that reads back exactly (`%g` isn't it, and `strfromd` takes only a printf-style format), and no allocation-free integer formatter. The exact spelling of shortest output (exponent threshold, `e+` form) is fixed with the `SHORTEST_MAX_*` constants in section 16.

**6.4 Testing.** All 2³² `float` values against an oracle and back through the parser (a strided sample in `make test`, the full sweep in the deep tier); random, boundary and halfway `double` cases; parsing at 1 to 800 digits and the 768-digit `double` midpoint; differential tests against glibc 2.44 in all four rounding modes; and an independent big-integer check of every generated table, since a wrong table entry is the characteristic failure.

## 7. Formatted I/O and streams: `std::io`

Research: [formatted I/O](../stdlib/formatted-io.md).

**7.1 Contents.** C23's `<stdio.h>` without `gets` and the wide functions, plus:

```c
FILE *fdopen(int fd, const char *mode);                            // POSIX
int   fileno(FILE *f);                                             // POSIX
int   fseeko(FILE *f, long off, int whence); long ftello(FILE *f); // POSIX
void  flockfile(FILE *f); int ftrylockfile(FILE *f); void funlockfile(FILE *f);   // POSIX
int   getc_unlocked(FILE *f); int putc_unlocked(int c, FILE *f);   // POSIX
int   getchar_unlocked(void); int putchar_unlocked(int c);         // POSIX
size_t fread_unlocked(void *p, size_t size, size_t n, FILE *f);    // glibc, BSD
size_t fwrite_unlocked(const void *p, size_t size, size_t n, FILE *f);   // glibc, BSD
long  getline(char **line, size_t *n, FILE *f);                    // POSIX
long  getdelim(char **line, size_t *n, int delim, FILE *f);        // POSIX
int   dprintf(int fd, const char *fmt, ...);                       // POSIX
int   vdprintf(int fd, const char *fmt, va_list ap);               // POSIX
int   asprintf(char **out, const char *fmt, ...);                  // POSIX 2024, glibc, BSD, musl
int   vasprintf(char **out, const char *fmt, va_list ap);          // POSIX 2024, glibc, BSD, musl
FILE *open_memstream(char **buf, size_t *size);                    // POSIX
FILE *fmemopen(void *buf, size_t size, const char *mode);          // POSIX
FILE *fopencookie(void *cookie, const char *mode, cookie_io_functions_t fns);   // glibc, musl, FreeBSD
```

- **`open_memstream` is the string builder.** Its stream buffer is the target buffer, so `fprintf` to it formats once, in place, with every conversion `printf` has and no new type.
- **`asprintf`** formats once into a small stack array and copies to an exact allocation, growing only for longer results, as glibc 2.37's does; on failure it sets `*out` to null.
- **`fopencookie`** gives custom streams (compressed files, sockets, ring buffers); BSD's `funopen`, which has a different callback shape, is not included.
- **`fmemopen`** follows POSIX.1-2024's text exactly, including the null written on flush in `"w"` mode; glibc's and musl's agreement with it is untested.
- **`%m`** prints `strerror(errno)` and consumes no argument, as glibc, musl and Cosmopolitan do. It is a fixed conversion, so literal formats stay checkable (suggested default).
- **Not included:** custom conversions (`register_printf_specifier`, Plan 9's `fmtinstall`). A registered letter changes what a literal format means, so it couldn't be checked at compile time, and it makes the set of conversions depend on registration order, against `lang.md` §13's rule that an unknown conversion is copied through. The C way stays: format the value into a string, or `fprintf` to an `open_memstream`.
- **Unknown conversions** are copied through whole, flags and width included, consuming no argument, which is what glibc 2.44 does; musl fails with `EINVAL`, and FreeBSD copies only the conversion character.

**7.2 Streams.** A musl-shaped `FILE` (musl's `vfprintf` is 7.9 KiB and `vfscanf` 3.0 KiB at `-Os`, against 22.7 KiB and 39.8 KiB in glibc 2.44): one allocation per stream, statically allocated `stdin`, `stdout` and `stderr`, one lock word, and a single-threaded fast path that takes no lock until a second thread exists. Buffered data and an overflowing write go out together in one `writev`. `stdout` is line-buffered on a terminal. `FILE` is opaque to Primal C code. Its first 56 bytes follow glibc's `struct _IO_FILE` (the flags word with glibc's end-of-file and error bits, then the read and write cursors in glibc's order), which are the cursors the stream already keeps, so glibc-compiled objects' inline `getc` and `putc` work against it (section 18.4); the rest of the layout is Kyogre's, including a text-translation flag that only the Microsoft face sets (section 18.5). Positions are 64-bit byte offsets. Each stream has an "in use" bit, set around compound operations, so a signal handler that enters the same stream on the same thread traps (`lang.md` §11.5).

**7.3 Engine.** One formatter writes to a small internal sink (pointer, end, running total, flush function) shared by every caller: a stream's sink is its buffer, `snprintf`'s is the caller's array (counting past the end), `asprintf`'s grows, `dprintf`'s writes. It takes an options word and a data-model descriptor (the widths of `long`, `long long`, `wchar_t`, `size_t` and pointers, and which legacy forms such as Microsoft's `%I64d` are enabled), so the C face's variants for each data model and runtime are parameters of one engine, not copies. Literal text and `%s` go out in `memcpy` runs. Numbers go through section 6. The engine is split by aspect, after llvm-libc's modular `printf`: when the compiler has checked a literal format and it has no floating-point conversion, the call goes to an integer-only entry, so a program that prints only integers doesn't link the float code. The `sprintf` overlap rule costs one range compare per `%s` argument; the engine formats into a stack buffer only when one overlaps.

**7.4 System mode.** `std::io::FILE` is the platform's `FILE`. `std::` formatting is Kyogre's engine writing into it with `fwrite_unlocked` under `flockfile`, because the rules differ from glibc's (unknown conversions, 128-bit `%lld`, `EOVERFLOW`). The layer adds the rules that are cheap and local: `fopen`'s mode check, `fgets`'s size check, and `fflush` on an input stream, which glibc already discards. Text and binary streams are already identical on Linux. The rest of `lang.md` §13's stream rules (no seek between reading and writing, positions after `ungetc` and after a failed transfer) and the §11.5 reentrancy trap hold only under `libc = "kyogre"`; under `libc = "system"` they are documented differences naming glibc and its version (suggested default; section 17).

## 8. Sorting and searching

**8.1 Contents** (`std::lib`):

```c
void  qsort(void *base, size_t n, size_t size, int (*cmp)(const void *, const void *));   // ISO C
void  qsort_r(void *base, size_t n, size_t size,
              int (*cmp)(const void *, const void *, void *), void *arg);                 // POSIX 2024, glibc order
void *bsearch(const void *key, const void *base, size_t n, size_t size,
              int (*cmp)(const void *, const void *));                                    // ISO C
inline void *bsearch_r(const void *key, const void *base, size_t n, size_t size,
              int (*cmp)(const void *, const void *, void *), void *arg);                 // new
inline void *lower_bound(const void *key, const void *base, size_t n, size_t size,
              int (*cmp)(const void *, const void *));    // new: first element not less than key, or the end
inline void *upper_bound(const void *key, const void *base, size_t n, size_t size,
              int (*cmp)(const void *, const void *));    // new: first element greater than key, or the end
int   mergesort(void *base, size_t n, size_t size, int (*cmp)(const void *, const void *));   // BSD; stable
```

**8.2 `qsort`** is an in-place introsort following ipnsort and LLVM libc's C adaptation: insertion sort for small inputs, median-of-three or recursive median pivots, branchless Lomuto partitioning with dedicated workers for 4-, 8- and 16-byte elements and a guarded Hoare scan for other sizes, equal-element detection, and heapsort after `2⌊log₂ n⌋` bad partitions. It never allocates, so its result depends only on the input and the comparator's answers, never on whether memory was available (glibc's `qsort` falls back between algorithms on `malloc` failure). Every scan is index-bounded, so an inconsistent or mutating comparator can't cause an access outside the array, and the array is a permutation of its input when `qsort` returns (`lang.md` §13). For reference, glibc's own introsort was merged in October 2023 and reverted in January 2024 before release, and the Qualys 2024 memory corruption came from the older quicksort's reliance on a sentinel.

**8.3 Specialization.** Most of `qsort`'s cost is its interface: in a one-core experiment sorting a million `int`s, fixing the element size cut the time by about 20%, and inlining the comparator by about 40% (research measurements, not independent). `qsort` and `qsort_r` are therefore `inline` forwarders in the module interface (`lang.md` §9) that check arguments and branch on a constant `size` to the dedicated workers, and the searches are short inline loops, so a known comparator can fold into the call site. Whether the optimizer clones a whole sort per call site with a constant comparator is a code-size budget question (section 16).

**8.4 `mergesort`** is the one sort that allocates (`n/2` elements), and fails with `ENOMEM` rather than falling back, so its output never depends on memory. It is optional (section 16); a stable sort without allocation is `qsort` over keys with their original index as a tiebreak. Annex K's `qsort_s` is not included. Research: [sorting and searching](../stdlib/sorting-searching.md).

## 9. Mathematics: `std::math`, `std::fenv`, `std::complex`

**9.1 The promise: the same bits everywhere.** Every function in `std::math` returns the same bits for the same arguments and rounding mode on every target, in every `libc` mode, whichever CPU features the machine has, whether the call is scalar, vectorized or folded at compile time. This is what the library guarantees; precision serves it. C leaves the accuracy of `<math.h>` implementation-defined, so neither tier below is undefined behavior: each is a stated result.

Each `float` and `double` function is in one of two tiers, and its documentation says which:

- **Correctly rounded.** The result is the exact value rounded in the current mode. Any correct implementation gives the same bits, so the implementation can change, kernels can be chosen by CPU feature, and vector lanes can fall back to scalar code without changing a result. A result that passes a sound rounding test *is* the correctly rounded value, whichever path produced it.
- **Pinned.** The result comes from one algorithm this library specifies as a fixed sequence of IEEE 754 operations, with an error under one ulp in round-to-nearest. Every target executes the same sequence: explicit `fma` operations where the algorithm uses them (emulated exactly on CPUs without FMA), no contraction or reassociation, and the same sequence lane by lane in vector code. In the directed rounding modes the same sequence runs in the current mode, so the result is still identical everywhere, at full speed, with each function's measured bound documented. Switching to round-to-nearest inside would cost every call a mode check and still couldn't guarantee the rounding direction; only correct rounding guarantees that.

The pinned tier is temporary by intent: the goal is every `float` and `double` function correctly rounded within the budget. Each pinned function has a named path to get there (section 9.3).

A function is correctly rounded when a correctly rounded design meets the speed budget (section 9.2), and pinned otherwise until one does. The functions exact by definition (`sqrt`, `fma`, `floor`, `ceil`, `trunc`, `round`, `rint`, `nearbyint`, `fabs`, `copysign`, `fmod`, `remainder`, `frexp`, `ldexp`, `scalbn`, `logb`, `nextafter`) are trivially correctly rounded. Moving a function from pinned to correctly rounded changes some of its results, so it is a versioned change listed in `CHANGES.md`; "the same bits everywhere" holds across targets and modes at one library version.

Initial assignment, from the measurements in [math performance: exp and log](../stdlib/math-performance-exp-log.md) and [pow and trigonometry](../stdlib/math-performance-pow-trig.md):

| Function | Tier | Design | Measured against glibc 2.44 (Zen 4, FMA) |
| --- | --- | --- | --- |
| `float` functions | Correctly rounded | CORE-MATH (MIT) | Often faster than glibc |
| `sin`, `cos`, `sincos` | Correctly rounded | CORE-MATH's current `sin`/`cos` (two 128-entry tables, one rounding test in the current mode) | 12–25% faster in round-to-nearest, about 3× faster in the other modes, 12× faster on huge arguments |
| `exp`, `exp2` | Correctly rounded | llvm-libc's reduction (mask-based, one 128-entry table), CORE-MATH as cross-check | Throughput 0.7× and 1.0×; latency up to 1.4× |
| `log`, `log2` | Correctly rounded | llvm-libc's design (128-entry table, degree-7 polynomial) | Throughput 0.85× and 1.0×; `log` 32% faster than CORE-MATH's |
| `pow` | Pinned | Arm optimized-routines' `pow` (MIT or Apache-2.0), which glibc ships; under 0.52 ulp | Correctly rounded `pow` measured +50% after optimization, and its hardest cases aren't fully known |
| `tan` | Pinned until a correctly rounded design meets the budget | fdlibm's `tan` as the pinned algorithm; a correctly rounded rewrite on `sin`'s design to be measured | CORE-MATH's `tan` +89% |
| The rest | Decided per function by the budget when implemented | CORE-MATH or llvm-libc where they meet it | Not measured |

**9.2 Speed budget.** Measured against glibc 2.44's public function (what a C program calls, including its wrapper), on the same machine at the same instruction-set level, with FMA hardware: reciprocal throughput within 30%, and latency within 50%, each over a named input distribution per function (its useful domain gates; the full domain and the hard-case set are reported). Each measurement also reports the slow-path rate and the bytes of code and tables (suggested default). A single "within 30%" without naming throughput or latency is meaningless here: CORE-MATH's `exp` has about 30–40% better throughput than glibc's and about 30% worse latency on the same machine.

**9.3 Implementation.**

- **Kernels by CPU feature.** Each correctly rounded function has an FMA kernel (always used on AArch64 and on x86-64 CPUs with FMA, which every Intel CPU since Haswell in 2013 and AMD CPU since Piledriver in 2012 has, except low-power Atom-class cores before 2021 and some budget models) and a no-FMA kernel using exact Dekker splitting with pre-split tables, selected once at start-up through section 4.4's table. Both give the same bits. The no-FMA `sin` measured 1.65× the FMA kernel, against 3.5× for emulating each `fma`. Pinned functions have one operation sequence; on CPUs without FMA its `fma` operations are emulated exactly, which is slow on those CPUs only.
- **Fast-path rules that measured well:** one unsigned range compare for all special cases; the rounding test compares bit patterns; slow paths are cold tail calls; the final step adds a low part to a high part in the current rounding mode, so no function reads the mode on its fast path; flag and exactness checks run only when the input bits make them possible (−9% on `pow`).
- **Compiled under `[[fenv_access]]`.** Every function in `std::math` has `[[fenv_access]]` semantics, so the optimizer never assumes round-to-nearest inside the library. Without the equivalent `-frounding-math`, GCC 16.2.1 miscompiles CORE-MATH's `cos`, `exp2` and `pow` in the directed modes; with it, the measured cost is nothing.
- **Pure functions.** With no `errno` (section 9.6), every `std::math` function is pure, so the compiler may hoist, merge and vectorize calls. It folds a call with constant arguments only to the library's own result: any correctly rounded evaluation for a correctly rounded function, and the pinned algorithm itself, run by the compile-time evaluator, for a pinned one (section 17).
- **Vector variants** run the same fast path in each lane and fall back to the scalar function for a lane whose rounding test fails, so each lane gives the scalar result. A 4-lane AVX2 `exp` matched scalar results on 80 million inputs in all four modes, with 0.22% of lanes falling back, at 1.36× the scalar throughput. They wait for a SIMD module.
- **The path to correct rounding for the pinned functions.** `tan`: rebuild it on `sin`'s design (a table of tangents, the tangent addition formula, one division, an error bound near 2⁻⁶³ with a rarely failing test), which also gives huge arguments the fast path. `pow`: after the measured −9% from gating its checks, shorten its double-double schedule (fold the `y · log x` product and the table products into fewer `fma` steps) and give `x` near 1 a cheaper path through an exact `x − 1`, where today's rounding test fails most. Each moves to the correctly rounded tier when it measures within the budget.

**9.4 Contents.** C23's `<math.h>`, including the `exp2m1`, `exp10m1`, `log2p1`, `log10p1`, `sinpi`, `cospi`, `tanpi`, `asinpi`, `acospi`, `atanpi`, `atan2pi`, `pown`, `rootn`, `compoundn`, `rsqrt`, `powr`, `fminimum` and `fmaximum` families, and the narrowing operations (`fadd`, `dsqrt`, `ffma` and the rest). C23 gives `pown`, `rootn` and `compoundn` a `long long` exponent, meaning 64 bits; in Primal C it is `long` (suggested default). `signgam` is replaced by `double lgamma_r(double x, int *sign)` (glibc, BSD). Classification and comparison are compiler builtins (`lang.md` §13), as is type-generic selection, which needs only that every function exist for each type under C's suffixes. There are no `cr_` names: C23 reserves them so a library can offer correct rounding beside a faster default, and each Kyogre function has one answer.

**9.5 Types.**

- **`long double`** functions are pinned, not correctly rounded: CORE-MATH covers only a few x87 80-bit functions, and binary128 is software on AArch64. Within a target they are deterministic like the rest; across targets they differ because the format does (x87 80-bit on x86-64, binary128 on AArch64). This is the one exception to section 9.1.
- **`std::fp`**'s `f16` and `f128` functions use C23 Annex H's names (`sinf16`, `sinf128`) and are correctly rounded; CORE-MATH's binary128 functions are 6 to 35 times faster than glibc's, and binary16 functions are tested exhaustively (65,536 inputs).

**9.6 Errors and the environment.** `std::math` never sets `errno`: `math_errhandling` is `MATH_ERREXCEPT` (suggested default; section 17), as CORE-MATH's default build already behaves. The C face's math sets `errno` for C units through wrappers on exceptional results only (section 18.3). Functions raise the IEEE 754 flags their exact results require (divide-by-zero for `log(0)`, invalid for `log(-1)`, overflow and underflow), but not inexact. They don't depend on the caller's `[[fenv_access]]`; internal helpers that need round-to-nearest (the Bessel functions) set and restore it. `std::fenv` is C23's `<fenv.h>` plus glibc and BSD's `feenableexcept`, `fedisableexcept` and `fegetexcept`, with `lang.md` §13's record of whether a call filled an `fenv_t`.

**9.7 Complex** functions follow C23 Annex G, based on FreeBSD msun (BSD license) with `cabs` as the correctly rounded `hypot`. They are pinned, and their accuracy is documented per function.

**9.8 Testing.** Exhaustive binary32 tests in four rounding modes (CORE-MATH's checker, MPFR as the oracle, in the deep tier), CORE-MATH's and Zimmermann's worst-case inputs for binary64 stored as data files with expected bits generated once, llvm-libc's tests, and Annex F and G special-value tables. Pinned functions are tested against frozen expected outputs, generated once and identical on both targets, on random and boundary inputs in all four modes, and against MPFR for their error bound. Every kernel of a correctly rounded function runs the same suites, and a test forces the no-FMA kernel on FMA hardware. MPFR (LGPL) is a test tool only. Research: [math library](../stdlib/math-library.md), [math performance techniques](../stdlib/math-performance-techniques.md).

## 10. Hashing and tables

C already has a hash table (`hsearch`), a tree (`tsearch`), a linear search (`lsearch`) and an intrusive list (`insque`), and the hash table is the one that's broken: one table per process, no deletion, no iteration. Nearly every nontrivial C program writes its own. Kyogre provides one hash function family and one table, shaped like `qsort`, and nothing else: no vector type (a struct and `growarray` suffice), no list or set type, no ordered map beyond `tsearch`. Research: [hashing and containers](../stdlib/hashing-and-containers.md).

**10.1 `std::hash`** (new).

```c
unsigned long bytes(const void *p, size_t n, unsigned long seed);              // rapidhash; p may be null when n is 0
unsigned long integer(unsigned long x, unsigned long seed);                    // one 64-bit integer, or a pointer converted to one
unsigned long keyed(const void *p, size_t n, const char key[static 16]);       // SipHash-1-3, for keys an attacker controls
```

`bytes` is rapidhash: in independent SMHasher3 results it is the fastest clean function on small keys, it is 487 bytes of code, and it needs no CPU features, so its output is the same on every target. Kyogre pins its own version of the algorithm (suggested default), because upstream rapidhash's output changes between versions and build options. Fields combine by chaining the seed: `h = hash::bytes(&b, sizeof b, hash::bytes(&a, sizeof a, seed))`. Its short-input path is `inline` in the interface, so a constant-length call compiles to a few instructions. `keyed` is SipHash-1-3 with a 16-byte secret key; it costs about five times as much per small key and is the choice for untrusted keys.

**10.2 `std::table`** (new). A flat open-addressing table in the Swiss-table layout: groups of 16 slots with a control byte each, probed with SSE2 on x86-64 and NEON on AArch64 (both baseline ISAs), and a scalar fallback that gives the same layout. An element is `esize` bytes whose first `ksize` bytes are its key.

```c
enum kind : int { KEY_BYTES, KEY_STRING, KEY_CUSTOM };
typedef unsigned long (*hash_fn)(const void *key, unsigned long seed);
typedef bool (*eq_fn)(const void *a, const void *b);
struct table { size_t count; /* private fields follow */ };

bool  init(struct table *t, size_t esize, size_t ksize, enum kind kind,
           unsigned long seed, hash_fn hash, eq_fn eq);    // hash and eq only for KEY_CUSTOM
void  destroy(struct table *t);
void  clear(struct table *t);                              // keeps capacity
bool  reserve(struct table *t, size_t n);
void *find(const struct table *t, const void *key);        // the element, or null
void *insert(struct table *t, const void *key, bool *added);   // finds or inserts; value bytes zeroed; null on ENOMEM
bool  remove(struct table *t, const void *key);           // true if the key was present
void *next(const struct table *t, size_t *pos);            // iteration; start with *pos = 0
```

- **Key kinds.** `KEY_BYTES` compares the key bytes with `memcmp`; `KEY_STRING` treats the key as a `char *` to a null-terminated string. Both need no callbacks. `KEY_CUSTOM` takes hash and equality functions, which cost about 17–19% per lookup in the research's microbenchmark, and less when the optimizer can inline them from the interface. A set is a table with `ksize == esize`.
- **Deterministic iteration.** The seed is an explicit argument: the library never randomizes on its own, so iteration order is a function of the seed and the operations, the same on every target. A program that hashes untrusted keys passes a seed from `arc4random` and hashes with `hash::keyed`.
- **Rules.** `init` checks `esize`, `ksize` and `kind` in O(1). `insert` copies a key that points into the table before growing; it may move elements, and `remove` never does. Removing the element just returned during iteration is allowed; inserting during iteration is defined but may skip or repeat elements. Over-aligned elements trap (section 16).
- **Target.** Within 1.2× of the best of Verstable, Abseil and Boost's flat map on Allan's benchmark rows for the callback-free kinds, and within 1.5× for `KEY_CUSTOM`. The best flat tables are already within 1.5× of each other there; Verstable's tombstone-free design is the fallback if removal-heavy workloads suffer.

Under `libc = "kyogre"`, `hsearch`, `tsearch`, `lsearch` and `insque` stay in the C face for C units; `std::` doesn't declare them (suggested default).

## 11. Text: `std::uchar` and `std::ctype`

Primal C is UTF-8 and the "C" locale only (`lang.md` §13). `std::uchar` keeps C23's `mbrtoc8`, `c8rtomb`, `mbrtoc16`, `c16rtomb`, `mbrtoc32` and `c32rtomb`, adopts C2y's `<stdmchar.h>` bulk conversions once C2y is published, and adds four stateless functions, because the restartable conversions need an `mbstate_t` and report through `errno`, while a lexer, terminal or renderer wants "decode the character here":

```c
size_t utf8_valid_length(const char *s, size_t n);           // new: length of the longest valid prefix; n means all valid
size_t utf8_decode(unsigned *cp, const char *s, size_t n);   // new: bytes consumed, 1 to 4; 0 only when n is 0
size_t utf8_encode(char out[4], unsigned cp);                // new: bytes written, 1 to 4
size_t utf8_count(const char *s, size_t n);                  // new: code points; each invalid byte counts as one
```

Invalid input decodes to U+FFFD, consuming the maximal invalid subpart (Unicode §3.9, as WHATWG does) and never more bytes than are present, so a decoding loop always advances; surrogates, overlong forms and values above U+10FFFF are invalid, and encoding one writes U+FFFD. `utf8_valid_length` is simdutf's validation with the error position kept and the error kind dropped. Precedent: Plan 9's `chartorune`, utf8proc's iterator.

`std::ctype` is ISO's, ASCII only, computed without tables. Unicode properties, normalization, case folding, grapheme breaking and display width are not in the library: they need large generated tables, and display width isn't portable. If they're wanted, they form a separate `std::unicode` module that the compiler's own identifier and NFC checks would share (section 16). Research: [text and strings](../stdlib/text-and-strings.md).

## 12. Threads: `std::thread`

**12.1 Contents.** C11's `<threads.h>` unchanged, with error-checking mutexes whose misuse traps (`lang.md` §11.4), plus:

```c
int  mtx_clocklock(mtx_t *m, int base, const struct timespec *abs);                // POSIX 2024 shape
int  cnd_clockwait(cnd_t *c, mtx_t *m, int base, const struct timespec *abs);      // POSIX 2024 shape
int  thrd_sleep_until(int base, const struct timespec *abs);                       // POSIX clock_nanosleep shape

typedef struct { unsigned state, wake; } rwlock_t;                                 // POSIX rwlocks
int  rwlock_init(rwlock_t *l);  void rwlock_destroy(rwlock_t *l);
int  rwlock_rdlock(rwlock_t *l);  int rwlock_tryrdlock(rwlock_t *l);
int  rwlock_wrlock(rwlock_t *l);  int rwlock_trywrlock(rwlock_t *l);
int  rwlock_clockrdlock(rwlock_t *l, int base, const struct timespec *abs);
int  rwlock_clockwrlock(rwlock_t *l, int base, const struct timespec *abs);
int  rwlock_unlock(rwlock_t *l);

typedef struct { unsigned value, waiters; } sem_t;                                 // POSIX semaphores
int  sem_init(sem_t *s, unsigned value);  void sem_destroy(sem_t *s);
int  sem_post(sem_t *s);  int sem_wait(sem_t *s);  int sem_trywait(sem_t *s);
int  sem_clockwait(sem_t *s, int base, const struct timespec *abs);

typedef struct { unsigned count, generation, waiters; } barrier_t;                 // POSIX barriers
int  barrier_init(barrier_t *b, unsigned count);  void barrier_destroy(barrier_t *b);
int  barrier_wait(barrier_t *b);               // thrd_barrier_serial in exactly one thread

typedef struct { unsigned long stack_size, guard_size; bool detached; } thrd_attr_t;
int  thrd_create_attr(thrd_t *t, const thrd_attr_t *a, thrd_start_t f, void *arg); // glibc attributes
int  thrd_setname(thrd_t t, const char *name);       // glibc; truncated to 15 bytes
int  thrd_getname(thrd_t t, char *buf, size_t n);
int  thrd_setaffinity(thrd_t t, size_t n, const unsigned char *cpuset);   // Linux sched_setaffinity
unsigned thrd_cpu_count(void);                       // affinity-aware; glibc get_nprocs
```

The `*clock*` functions take a `timespec_get` base, `TIME_UTC` or `TIME_MONOTONIC`, because C11's timed waits use calendar time and break when the clock is set. The kernel does the conversion (`FUTEX_WAIT_BITSET` with an absolute time). Cancellation is not included: a cooperative stop is an atomic flag, and asynchronous cancellation in the middle of a lock operation would make the trap rules impossible to state. Robust and priority-inheritance mutexes are not in v1; the mutex layout leaves room for both.

**12.2 Implementation.** One futex-based implementation in both modes.

- **Mutex.** One 32-bit word in the kernel's priority-inheritance futex layout: 0 when unlocked, otherwise the owner's thread ID, with a waiters bit. Lock and unlock are one compare-and-swap each; the owner check that makes relocking and unlocking an unowned mutex trap is the compare itself, so error checking adds nothing to the fast path. A second word holds the kind and recursion depth: `mtx_t` is 8 bytes (glibc's is 40). The mutex is unfair (barging), spins about 100 loads, then waits on the futex. In the research's single-core measurement an uncontended pair took 3.3 ns, against 4.1 ns for glibc's default mutex and 5.1 ns for its error-checking one.
- **Condition variable**: a sequence word and a waiter count, so `signal` with no waiters makes no system call. It must be model-checked before adoption; Rust's counter-only condition variable is the fallback (section 16).
- **`call_once`**: a state word plus the running thread's ID, one acquire load once done; a recursive call or an initializer that exits or `longjmp`s out traps. Kyogre's `call_once` is plain atomics, so ThreadSanitizer follows it, unlike glibc's (`AGENTS.md`).
- **Read-write lock**: one state word plus a wake word. A writer's unlock is owner-checked; a reader's can't be checked in O(1), so it traps only when the lock isn't held at all.
- **Threads** (`libc = "kyogre"`): one mapping holds a guard page, the stack, static TLS and the thread descriptor, created with `clone3`; `join` waits on the kernel-cleared thread ID; finished threads' mappings go to a small cache. The default stack is 1 MiB on every target (suggested default; glibc's default follows `RLIMIT_STACK`, musl's is 128 KiB). Under `libc = "system"` the platform's `pthread_create` makes threads, and Kyogre's state word beside each catches double joins.
- **TLS.** Static TLS with local-exec access; the thread pointer is the descriptor's address. On x86-64 the first bytes mirror glibc's layout where compiled code depends on it: a self-pointer at `%fs:0` and the stack-protector canary at `%fs:0x28`. `errno`, the thread ID and the §11.5 reentrancy marks share the descriptor's first cache line.
- **Signal reentrancy** (`lang.md` §11.5) costs one `or` and one `and` on a per-thread mask for each library subsystem whose invariants break mid-update (allocator, each stream). User mutexes need no mark: a handler that relocks a mutex its own thread holds already traps through the owner check.
- **POSIX threads for C units.** The C face's `pthread_*` types have glibc's sizes and static-initializer offsets per target (`pthread_mutex_t` is 40 bytes on x86-64 and 48 on AArch64), with Kyogre's 8-byte lock word at offset 0 and the mutex kind read from glibc's offset, so glibc-compiled objects and static initializers work unchanged (section 18.4). The same code serves `mtx_t` and `pthread_mutex_t`.

Research: [concurrency](../stdlib/concurrency.md).

## 13. Randomness and time

**13.1 Randomness.**

- **`rand` and `srand`** in `std::lib` are pinned to one algorithm so the sequence is the same on every target and in every mode: a 64-bit state advanced by SplitMix64's increment (`0x9E3779B97F4A7C15`) with a relaxed atomic add, output `mix(x) >> 33`, `RAND_MAX` 2³¹ − 1, and `srand(1)` as the initial state (suggested default: one global state, not per thread). The C face's `rand` is a separate function that reproduces the target's reference C library, because C programs and their tests print values after `srand` (section 18.2): glibc's TYPE_3 additive generator on the Linux rows, where `srand(1)` gives 846930886 and then 1804289383.
- **Secure randomness** (`std::lib`): `arc4random`, `arc4random_buf`, `arc4random_uniform` (OpenBSD; glibc 2.36) and `getentropy` (POSIX 2024). They never fail or return partial data; if the kernel refuses entropy, they trap. `arc4random_uniform` of 0 or 1 returns 0. Under `libc = "kyogre"` they use the Linux 6.11 vDSO `getrandom`, with the system call as fallback.
- **`std::rand`** (new) is a seeded generator on caller state, for simulations, tests and games that need reproducible streams, as `drand48`'s explicit-state functions are, with a modern algorithm (xoshiro256++, Blackman and Vigna, public domain):

```c
struct state { unsigned long s[4]; };
void          seed(struct state *g, unsigned long seed);    // expands with SplitMix64; never all zero
unsigned long next(struct state *g);                        // 64 uniform bits
unsigned long below(struct state *g, unsigned long bound);  // Lemire's method; bound 0 returns 0
double        fraction(struct state *g);                    // [0, 1): (x >> 11) × 2⁻⁵³
float         fractionf(struct state *g);                   // [0, 1): (x >> 40) × 2⁻²⁴
void          fill(struct state *g, void *buf, size_t n);
struct state  split(struct state *g);                       // a stream that never overlaps g's: returns g, then moves g 2¹²⁸ steps on
void          shuffle(struct state *g, void *base, size_t n, size_t size);   // Fisher–Yates
```

**13.2 Time** (`std::time`). C23's `timespec_get` and `timespec_getres` with `TIME_UTC`, `TIME_MONOTONIC`, `TIME_ACTIVE` and `TIME_THREAD_ACTIVE`, and `timegm`; POSIX's `clock_gettime`, `clock_getres`, `clock_nanosleep`, `nanosleep`, `gmtime_r`, `localtime_r` and `strptime`; no `asctime` or `ctime` (`lang.md` §13).

- **Clocks.** `clock_gettime` uses the vDSO, resolved on first use and never at start-up, and falls back to the system call; on this machine that is 15.5 ns against 262 ns.
- **Time zones.** `localtime` reads `TZ` and TZif files with IANA's published algorithm (public domain); a missing or unreadable zone means UTC, never an error.
- **`strftime` and `strptime`** use the "C" locale only. They add glibc and BSD's `%s` and ignore the `%E` and `%O` modifiers. Out-of-range `struct tm` fields print as given, and names that can't be looked up print as `?`.
- **Range.** `gmtime`, `localtime` and `timegm` return null, or −1, with `EOVERFLOW` when the year doesn't fit an `int`.

Research: [random, time and system](../stdlib/random-time-system.md).

## 14. Programs and the environment

**14.1 `std::program`** (new module; existing functions). The helpers BSD and GNU programs expect together, which can't live in `std::err`, the `errno` module:

```c
const char *getprogname(void);   void setprogname(const char *name);              // BSD
void err(int status, const char *fmt, ...);   void errx(int status, const char *fmt, ...);   // 4.4BSD
void warn(const char *fmt, ...);              void warnx(const char *fmt, ...);               // 4.4BSD
void verr(int status, const char *fmt, va_list ap);  void verrx(int status, const char *fmt, va_list ap);
void vwarn(const char *fmt, va_list ap);             void vwarnx(const char *fmt, va_list ap);
extern thread_local int optind, opterr, optopt;  extern thread_local char *optarg;  // POSIX, per thread
int getopt(int argc, char *const argv[], const char *opts);                        // POSIX
struct option { const char *name; int has_arg; int *flag; int val; };              // GNU, BSD
int getopt_long(int argc, char *const argv[], const char *opts,
                const struct option *longopts, int *index);                        // GNU, BSD
```

`getopt`'s cursor is per thread, like `errno`; `optind = 0` restarts a scan (glibc's convention). There is no `POSIXLY_CORRECT`. Whether `getopt_long` permutes `argv` as GNU's does is open (section 16).

**14.2 Environment** (`std::lib`). `getenv`, `setenv`, `unsetenv`, `putenv`, `clearenv`, following glibc 2.41's design: `getenv` is wait-free and async-signal-safe; writers take a lock among themselves; old strings and arrays are never freed, so memory grows by a bounded amount per distinct variable. A null name to `setenv` traps; an empty name or one containing `=` fails with `EINVAL`. `getenv` returns the environment's own writable string (`lang.md` §13). `strtonum` (OpenBSD) is optional (section 16).

## 15. Build order, gates and measurement

`plan.md`'s std-1 (XL) splits into these milestones, in dependency order:

1. **Foundation.** `std::sys` for `x86_64-linux`, `start`, static TLS, `exit`, `errno`, and the memory primitives. Gate: a program writing with `write` runs under `libc = "kyogre"`.
2. **Core.** The allocator, `std::str`, `std::ctype`, number conversion, sorting, `std::io`'s engine and `FILE`, `std::hash`, `std::table`, `std::rand`, and the C face's manifest, generated headers and entry points. Gate: libc-test's `functional` and `regression` suites against the C face; differential tests against glibc 2.44 for `strtod`, `printf`, string results and `qsort` permutations; the header gate (every header in every dialect against the reference's declared names); and the layout gate (section 18.4).
3. **Concurrency and system.** `std::thread`, locked streams, `std::time`, `std::signal`, `std::jmp`, `std::program`, the environment, and the C face's `pthread_*`. Gate: the condition variable and read-write lock model-checked; ThreadSanitizer clean with Clang (`AGENTS.md`).
4. **Mathematics.** `std::math`, `std::fenv`, `std::complex`, `std::fp`, and the C face's `errno`-setting wrappers. Gate: the worst-case files and the pinned functions' frozen outputs in `make test`, exhaustive binary32 in the deep tier, and every kernel of each correctly rounded function forced and checked.
5. **glibc compatibility.** The binary-compatibility names (section 18.4). Gate: the glibc-compiled object corpus links into a static `libc = "kyogre"` executable and passes its tests, with unresolved names counted by tier.
6. **System mode.** The `libc = "system"` wrappers over glibc, and the allocator's interposition. Gate: the same suites, through `std::`, with documented differences listed.
7. **AArch64.** `std::sys`, start-up and the kernels for `aarch64-linux`. Gate: every output that section 1 pins is bit-identical between `x86_64-linux` and `aarch64-linux` under `qemu-user`.
8. **Microsoft face on Linux.** The `libc_abi = "ms"` names (section 18.5), built and tested on `x86_64-linux` against Wine's `ucrtbase` and `msvcrt` tests, before any Windows row.

**Every milestone** reports its numbers to the PERF lane: object bytes, retired instructions, peak resident memory and allocation count for an empty `main`, `puts`, `printf("%d")`, `malloc` and `free`, sorting a million `int`s, parsing a million `double`s, and `sqlite3.c` at `-O0`, `-O2` and `-Os`, each against glibc 2.44 and musl (glibc's static empty `main` is 719 KB of text on this machine). The component budgets are the ones stated in each section: `std::math` by section 9.2, `std::table` within 1.2× of the best flat tables, mimalloc-bench and long-running peak-RSS runs for the allocator.

**Tests** follow `AGENTS.md`'s tiers: exhaustive sweeps (all `float` values for conversion and the binary32 math functions) are deep-tier, with strided samples and known hard cases in `make test`; comparator fuzzing for `qsort` (inconsistent, mutating and `longjmp`ing comparators under ASan, asserting bounds and a permutation); test suites borrowed from libc-test (MIT), llvm-libc (Apache-2.0) and CORE-MATH (MIT); glibc (LGPL) only as a differential oracle on the host.

## 16. Open questions and suggested defaults

| Question | Suggested default | Alternative |
| --- | --- | --- |
| Allocator in system mode | Kyogre's allocator interposes `malloc` process-wide; `allocator = "system"` opts out | Layer over the platform's and carve §13's trap out of system mode |
| Allocator source | Write Kyogre's own (§5.2) | mimalloc at `MI_SECURE` 2 with an aborting handler and a double-free bitmap |
| Cross-thread double free | Trap at the second `free`, or at the latest when the owner reclaims the page | Trap at the call, at the cost of an atomic per local free |
| `FILE` rules in system mode | Hold under `libc = "kyogre"`; documented differences under `system` | A wrapping layer with a mode check per operation |
| Over-reads within a page in string kernels | Allowed in library kernels; byte-exact kernels in sanitizer builds | Masked loads only |
| Kernel selection | A function-pointer table filled once, baseline by default | `IRELATIVE` relocations, or a build-time level |
| `memcmp`, `strcmp` results | −1, 0 or 1 | The byte difference, as glibc |
| Non-POSIX conveniences in `std::str` | `strsep`, `strchrnul`, `mempcpy`, `memrchr` in | POSIX 2024 only |
| Rounding mode in conversions | Honoured, as glibc and llvm-libc | Round-to-nearest only, as musl and fast_float |
| Shortest-output spelling and `SHORTEST_MAX_*` | Decided with the first implementation, matching `%g`'s exponent form | Leave `strshortest*` out |
| Parse fallback | llvm-libc's 800-digit decimal | fast_float's 4,000-bit integer |
| Shared power-of-ten tables for printing and parsing | Shared if the formats align | Two tables, about 21 KB |
| `std::math` promise | The same bits everywhere; each function correctly rounded or pinned, chosen by the budget, with correct rounding as the goal for all | Correctly rounded throughout; or pinned throughout |
| `long double` math | Pinned; deterministic per target, different across targets | Correctly rounded at a large cost |
| `std::math` speed budget | Throughput within 30% and latency within 50% of glibc's public functions, same machine and ISA level | One combined figure |
| `tan` | Pinned (fdlibm) until a correctly rounded design measures within budget | Correctly rounded at +89% |
| FMA on x86-64 | Two kernels for correctly rounded functions; exact `fma` emulation for pinned ones on CPUs without FMA | Require x86-64-v3 for `std::math` |
| `pown`, `rootn`, `compoundn` exponent | `long`, the 64-bit width C23 intends by `long long` | Primal C's 128-bit `long long` |
| `errno` from `std::math` | Never set; `math_errhandling` is `MATH_ERREXCEPT` | Set as C allows |
| `signgam` | Removed; `lgamma_r` only | A `thread_local signgam` |
| `qsort` cloning per call site | Within the optimizer's code-size budget, constant comparators only | Never |
| `mergesort` | Included, optional to use | Left out; sort with an index tiebreak |
| Permutation during `qsort` | At return | At every comparator call |
| `hash::bytes` stability | Frozen to Kyogre's own rapidhash version | Track upstream |
| Table seeds | Explicit, never randomized by the library | Random by default |
| Over-aligned table elements | Trap | An alignment parameter to `init` |
| `hsearch`, `tsearch` | C face only | Also in `std::lib` |
| UTF-8 invalid-byte policy | One U+FFFD per maximal subpart | One per byte |
| Unicode data | Not in v1; a separate `std::unicode` if wanted | Properties and case folding in `std::uchar` |
| Default thread stack | 1 MiB on every target | The platform's |
| Condition variable | Sequence plus waiter count, model-checked | Rust's counter-only design |
| `rand` state | One global, atomic | Per thread |
| `getopt_long` permutation | No permutation unless the option string starts with `-` | GNU's default permutation |
| `strtonum` | Left out | In `std::lib` |
| Arena module | `std::arena` | Functions in `std::lib` |
| `BUFSIZ` and default buffer | 4 KiB | The file's `st_blksize` |
| `%m` in `printf` | Included, as glibc and musl | Left out; `strerror` through `%s` |
| §11.5 trap on streams | Compound operations only; not the `*_unlocked` single-character functions | Every stream operation |
| C face headers | Generated from the manifest, declarations and feature gates alike; `features.h` and `sys/cdefs.h` macro machinery written by hand | Every header written by hand in glibc's structure |
| `__GLIBC__` | Not defined; `__GLIBC_PREREQ` is 0 and `__KY_LIBC__` gives the version; a product setting may define it | Defined, claiming a glibc version |
| Linking glibc-built objects | Static archives and objects only | Also glibc-built shared libraries, which need a loader and symbol versions |
| `name@GLIBC_x.y` references in static links | Resolved as `name` (a linker decision) | Rejected, as GNU ld does |
| C face `rand` | The reference C library's sequence per target | `std::`'s pinned sequence |
| C face `qsort` | Kyogre's unstable introsort; C leaves stability unspecified | A stable sort, as glibc's is in practice |
| `mbrtowc` in the `"C"` locale for C units | Rejects bytes 0x80 and above, as glibc | Maps them, as musl |
| Unicode tables for the C face's `C.UTF-8` | Generated, linked only when used | ASCII only |
| Microsoft `_snprintf` on truncation | Microsoft's contract: no terminator, returns −1 | Always terminate |
| Default text mode under the Microsoft face | Binary; a product setting enables CRLF translation | Text, as MSVC |
| UTF-8 code-page manifest in Windows executables | Embedded by default | Only on request |
| Microsoft C runtime target | The Universal CRT | Also `msvcrt.dll` |

## 17. Changes to `lang.md` this spec needs

Each needs approval as a spec change and an entry in `CHANGES.md`.

1. **§14.1, `libc = "system"`.** "`std::` is a layer over that library's allocator, `FILE` and `errno`" becomes: stateless modules are Kyogre's in every mode; the allocator is Kyogre's and defines `malloc` process-wide unless the product sets `allocator = "system"`; `FILE`, `errno`, threads and clocks are the platform's (section 2.2).
2. **§13, library-wide rules.** The stream rules and the §11.5 reentrancy trap hold under `libc = "kyogre"`; under `system`, the differences are documented by library and version (section 7.4). `fflush` on a seekable input stream also repositions the file offset to the stream's position, as POSIX.1-2024 says. Unknown conversions are copied through with their flags and width.
3. **§13, `<stdlib.h>` row.** A cross-thread double free traps at the second `free` or at the latest when the owning thread reclaims the page (section 5.2); `std::`'s `rand` is pinned, while the C face's reproduces the reference library (section 13.1).
4. **§13, `<math.h>` row.** Every function gives the same bits on every target, correctly rounded or pinned per function; `errno` is never set and `math_errhandling` is `MATH_ERREXCEPT`; `signgam` is replaced by `lgamma_r` (section 9).
5. **§13, module table.** New modules `std::hash`, `std::table`, `std::rand`, `std::arena` and `std::program`, and the additions listed in section 3. `<strings.h>`'s functions join `std::str`.
6. **§15.** "Length-carrying APIs" names the functions in section 4.2.
7. **§6.5, a new `[[alias("sym")]]` attribute.** It gives a definition a further symbol name, weak by default, as GCC's `alias` attribute does. The C face needs several names for one definition (`strtol`, `strtoll` and `strtoimax` on LP64); `[[link_name]]` gives only one.
8. **§14.1, the C face.** Under `libc = "kyogre"`, C units call the library through its C face (section 18), which follows section 18.2's boundary rule. The library's code is Primal C, so a C unit's call gets Primal C's defined result where C leaves the behavior undefined; this governs the library, while §14.1's rule that the compiler neither defines nor exploits C units' undefined behavior still governs the compiler.

Two requirements fall on other specs. Constant folding of a `std::math` call (`sema.md`, `ir.md`) must give the library's own result: any correctly rounded evaluation for a correctly rounded function, and the pinned algorithm itself, run by the compile-time evaluator, for a pinned one. And `frontend-plan.md` §9's product settings gain `allocator`, `libc_abi` and the Microsoft face's `fmode`.

## 18. Serving C units: the C face and compatibility

The library is written in Primal C and serves C units of every dialect and data model. Under `libc = "system"`, C units use the sysroot's headers and the platform library, and none of this section applies to them (`lang.md` §14.1). Research: [C face](../stdlib/c-face.md), [glibc compatibility](../stdlib/glibc-compatibility.md), [Microsoft C runtime compatibility](../stdlib/msvc-compatibility.md).

**18.1 The C face.** Primal C source with one object per entry point or alias group, linked only when referenced, so the face costs an empty `main` nothing. One manifest lists every C name with its C types, the standards and feature-test macros that declare it, and its variants. From it a generator produces the C headers, the alias list and the constants shared with `std::` (errno values, signal numbers, limits). The face imports its own generated headers, so the compiler checks every entry point against the declaration C units see (`lang.md` §14.4).

- **Three kinds of entry.** An alias of a `std::` function (most of them, through `[[alias]]`). A thin entry over a `std::` engine where the C type differs from Primal C's or a standard changes the behavior. Or a C-only or compatibility function (sections 18.3 to 18.5). Dependencies run one way: compatibility faces, then the C face, then `std::`.
- **Data models.** A generated `std::abi` module holds each target row's C types and widths, and the engines take widths and flags as parameters. On LP64, C's `long long` and `intmax_t` are Primal C's 64-bit `long`, so `strtoll`, `llabs`, `lldiv`, `imaxabs` and `strtoimax` are aliases of the `long` entries; only `std::`'s 128-bit `long long` functions need entries of their own. On ILP32 and on LLP64 Windows, the C `long` entries become the thin ones. A new data model is a new row, not new code.
- **Headers.** The compiler keeps providing the headers that describe the compiler and target (`include/`: `<stddef.h>`, `<stdarg.h>`, `<limits.h>` and the rest, `frontend-plan.md` §3). Every other header comes from the library, generated in GNU reserved spellings (`__restrict`, `__inline`, `__attribute__((__noreturn__))`) so one header serves C89 through C2y, with a separate branch for K&R. `features.h` resolves feature-test macros with glibc 2.44's precedence (`_GNU_SOURCE`, `_DEFAULT_SOURCE` when not strict, `_POSIX_C_SOURCE`, `_XOPEN_SOURCE`, `_FILE_OFFSET_BITS`, `_TIME_BITS`, `_FORTIFY_SOURCE`), and `sys/cdefs.h` supplies glibc's `__THROW`, `__nonnull`, `__REDIRECT` and `__attribute_*__` macros, which programs include directly. A compatibility directory searched last holds the headers only some programs expect (`error.h`, `sys/queue.h`), so a project's own copy wins.
- **Standard variants** are chosen in the header with `__asm__` labels, as glibc does: under C23, `strtol` and the `scanf` family bind to entries that accept `0b` prefixes; under earlier standards to ones that don't. A C17 unit and a C23 unit in one program each get their own `strtol`, with no run-time flag.
- **C's macros.** `errno` expands to `(*__errno_location())`; `assert`, `getc`, `putc`, `offsetof`, the limits and `PRId64` are macros as C expects, with `long long` meaning 64 bits; `<tgmath.h>` uses `_Generic` or the compiler's `__builtin_tgmath`.

**18.2 The boundary rule.** The library's code is Primal C, but C units and their test suites depend on C's and the reference library's behavior:

1. **Where C or POSIX defines a behavior, the face gives it exactly.**
2. **Where C leaves it undefined or unspecified and the reference library has a stable behavior programs rely on, the face reproduces it**, when that costs none of Primal C's rules. The list is closed, each item named by library and version, and tested differentially: `rand`'s sequence; `printf`'s `(nil)` for a null `%p` and `(null)` for a null `%s`; `strerror`'s message texts; `%a`'s spelling; `strftime("%c")` in the `"C"` locale; `strtol` setting `EINVAL` for a bad base; `atexit`'s order.
3. **Otherwise Primal C's rules apply, to C units too**: `memcpy` with overlap moves, `free` of a foreign pointer traps, an inconsistent `qsort` comparator stays in bounds. This never makes a correct program behave differently, and makes an incorrect one more predictable.
4. **A trap only where a correct C program can't reach it.** `free` of an invalid pointer traps (undefined in C), but `fopen(path, "rw")`, whose result POSIX leaves to the implementation, fails with `EINVAL`.
5. **What can't be reproduced is a named difference**: `long double` math, last-bit differences from the reference's math, thread interleavings, allocation addresses.

**18.3 What C units get that `std::` drops.** The face provides the full ISO C library and the POSIX interfaces programs use, on `std::`'s internals:

- **Wide characters.** Wide-character strings and classification (`wcslen`, `wmemcpy`, `towlower`, `iswalpha`, `wcwidth`), with Unicode tables generated and linked only when used; wide-character I/O (`fgetwc`, `fwprintf`, `fwide`) for completeness, though the glibc scan found it in at most 22 of 6,300 files.
- **`<locale.h>`** with the `"C"` and `"C.UTF-8"` locales (`setlocale`, `localeconv`, `nl_langinfo`, `newlocale` and `uselocale` over those two), plus a small `iconv` (UTF-8, UTF-16 and UTF-32 in both byte orders, ASCII, ISO-8859-1) and `gettext` stubs that return their argument.
- **Legacy and removed functions**: `gets` (declared below C11; the symbol always exists), `asctime`, `ctime` and their `_r` forms, `signgam`, `random`, `srandom`, `initstate`, `setstate`, the `drand48` family, `rand_r`, `hsearch`, `tsearch`, `lsearch`, `insque` and `alloca`.
- **Math that sets `errno`**: `math_errhandling` is `MATH_ERRNO | MATH_ERREXCEPT` for C units, through wrappers that act only on exceptional results, as glibc's do, over the same `std::math` functions.
- **Not provided**: Annex K's `_s` functions (optional in ISO C, and glibc lacks them too).

**18.4 glibc compatibility.** C programs written for glibc build and run against `libc = "kyogre"`, and objects and static archives compiled against glibc's headers link into Kyogre products. A scan of 6,300 programs and libraries on this machine found that 91% of them use only names musl's headers also declare; 113 distinct glibc-internal names cover the rest, and 40 of those cover 98% of files.

- **Source level, required:** all C23 and POSIX.1-2024 headers; glibc's feature-test macros; the GNU and BSD functions this spec lists; `<stdio_ext.h>`; GNU `getopt_long`; `<err.h>`, `<error.h>`, `<malloc.h>` (`mallinfo2` with real totals, `mallopt`); `<execinfo.h>` over the runtime's unwinder; `dl_iterate_phdr` and `_dl_find_object`; and the `<sys/*.h>`, `<net*/…>`, `<pthread.h>`, `<dirent.h>`, `<termios.h>`, `<regex.h>`, `<glob.h>` and other system headers behind the functions programs call. Later, as programs need them: `<fts.h>`, `<obstack.h>`, `<argp.h>`, `ucontext`, files-only name lookup (`getpwnam`, `getaddrinfo` over `/etc` and a stub resolver, without `nsswitch.conf`). Not provided: custom `printf` conversions, GNU's `re_*` regex interface, and `dlopen` of arbitrary libraries, which returns null with a `dlerror` message as a static glibc does.
- **Binary level: glibc's ABI names and layouts are Kyogre's own.** The C face defines the names glibc-compiled code calls: `__errno_location`, `__stack_chk_fail`, `__assert_fail`, `__cxa_atexit`, `__ctype_b_loc` and its tables, `__libc_single_threaded`, `_setjmp`, the `<stdio_ext.h>` names and about 30 more (about 300 lines), then the mechanical families: the `_FORTIFY_SOURCE` checks (`__memcpy_chk` and 66 more, which the compiler's fortify builtins also emit), the `__isoc99_*` and `__isoc23_*` variants, the `*64` names (aliases on LP64) and the `__xstat` family. Each is its own archive member, so a program that references none links none. The ABI-visible types take glibc's per-target layout: `FILE`'s first 56 bytes (section 7.2), the `pthread_*` types (section 12.2), `jmp_buf`, `sigset_t`, `struct sigaction`, `struct stat`, `dirent`, `struct tm` with `tm_gmtoff`, `lconv` and the rest. Choosing them now costs nothing; changing them after C units are built breaks every one.
- **Static only.** glibc-built shared libraries, `ld.so`, symbol versions and `GLIBC_PRIVATE` interfaces are out of scope: a product that must load a library built against `libc.so.6` uses `libc = "system"`.
- **Gates.** A layout gate compares `sizeof`, `alignof` and `offsetof` for every ABI-visible type with glibc 2.44's headers on both target rows, as `struct-layout-1` does for the compiler. A link gate builds a small corpus (zlib, Lua, SQLite, PCRE2) with GCC against glibc's headers, at `-O2 -D_FORTIFY_SOURCE=3 -fstack-protector-strong`, links it into a static `libc = "kyogre"` executable, and runs its tests; unresolved names are counted gaps by tier.

**18.5 Microsoft C runtime compatibility.** Windows rows come after the v1 Linux rows; the design leaves room for them now.

- **A face, not a fork.** The Microsoft names are a compatibility face (`libc_abi = "ms"`) in Primal C over `std::`, like the glibc names. It is ordinary C, so it builds and is tested on Linux first, against Wine's `ucrtbase` and `msvcrt` tests used as an oracle (Wine is LGPL, so its code is not copied), and code written for Microsoft's runtime builds on Linux too.
- **First tier**, about 150 to 200 names: the `__stdio_common_*` formatting entries and `__acrt_iob_func` that MSVC-compiled code calls, `_errno`, `_snprintf` and `_vsnprintf`, `_strdup`, `_stricmp`, `_strnicmp`, `_aligned_malloc` and `_aligned_free`, `_beginthreadex`, `errno_t`, `_set_invalid_parameter_handler`, `_set_fmode`, the `_wfopen` family converting UTF-16 paths to UTF-8, and the stack cookie and `__chkstk`. **Second tier**: Microsoft's `_s` functions with Microsoft's signatures, which differ from Annex K's; the `io.h`, `process.h`, `direct.h` and `tchar.h` layers. **Not provided**: the debug runtime, wide stream I/O, non-UTF-8 code pages and the `_mbs*` functions that depend on them, and `msvcrt.dll`'s legacy formatting quirks.
- **Two Windows modes.** `libc = "system"` sits on the Universal CRT through generated import libraries. `libc = "kyogre"` implements `std::sys` over `kernel32` and `ntdll` and exports the `/MD` names MSVC-compiled objects import.
- **Primal C's rules on Windows.** Paths and strings are UTF-8 above `std::sys`, which converts to UTF-16 at each OS call; `argv` comes from `GetCommandLineW`; executables embed the UTF-8 active-code-page manifest. Streams are binary unless a product enables text-mode translation for the Microsoft face. The invalid-parameter handler's default traps; a program that installs its own gets Microsoft's `errno` and error return when it returns. `long` is 32 bits and `wchar_t` 16 bits for Windows C units, handled by the data-model rows of section 18.1, and `%I64d` is a legacy form the engine's options enable.
- **What v1 keeps open for it.** `std::sys` takes UTF-8 and returns `-errno` (section 2.4). The formatting engine takes an options word and a data-model descriptor (section 7.3). The C face is generated per target row. `FILE` has a text-translation flag and 64-bit offsets (section 7.2). Stateful modules reach TLS, locks and `errno` through internal interfaces, not `%fs` directly. Precondition failures go through one internal trap function, which the Microsoft face can route through the invalid-parameter handler first.
