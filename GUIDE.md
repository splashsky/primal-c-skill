# Primal C: A Language Guide

Revision 1 · November 2026

Primal C is C, made consistent. It keeps C's syntax, types, operators and calling convention, takes C23 as its base
and C2y's accepted features on top, and changes two things: **modules with namespaces replace headers**, and **every
behavior C leaves undefined is given a meaning** (a defined result, a compile-time error, or a deterministic trap).
Its one new feature is namespaces; its other changes spell out C's own evolution (`bool` instead of `_Bool`,
`atomic` instead of `_Atomic`) and remove its traps. It has no runtime, no garbage collector and no exceptions, and
it links with C code through the platform C ABI.

Its toolchain is **Kyogre** (command `ky`), which compiles Primal C and every C dialect side by side. This guide is
for writing Primal C. The authoritative text is [`reference/lang.md`](reference/lang.md); where this guide and the
specification disagree, the specification wins. Choices the specification marks as a *suggested default* may
still change between revisions.

---

## 1. A first program

```c
#namespace hello

#import <std::io>

int main(void) {
	io::printf("Hello, %s!\n", "world");
}
```

- `#namespace hello` says which module this file belongs to. It is a directive, like `#include`: no semicolon.
- `#import <std::io>` makes the standard library's `<stdio.h>` module available as `io::`.
- `main` falls off its end and returns 0, as in C.

Nothing else is needed: no header, no prototype, no `#include <stdio.h>`.

---

## 2. Modules, namespaces and imports

### Files and modules

- A **module** is every file that declares the same namespace. All Primal C files in one directory declare the
  same namespace (or all declare none). A namespace may span several directories.
- `#namespace a::b` must be the file's **first directive**, before anything but comments and blank lines, **once**,
  and **never inside `#if`**. A file without one is in the **root namespace**, and its names are emitted exactly as
  in C.
- Inside a module, **file-scope declarations are visible in every file, in any order**. You never write a forward
  declaration or prototype at file scope. Inside a function body, C's declare-before-use rule still applies.

### Visibility

- Everything at file scope that isn't `static` is **visible to importers**. There is no `export` keyword.
- `static` keeps its C meaning: this file only. It is the only way to hide a name from importers. Helpers several
  files of a module share but that aren't meant for importers are often put in an internal namespace
  (`mylib::internal::parse`) to signal intent.

### Importing

```c
#namespace app

#import <std::io>             // a namespace: names are io::printf, io::FILE ...
#import <std::io> as out      // the same namespace under another name: out::printf
#import <geo::mesh>           // any namespace in the build, found by its declaration
#import "../util/strings"     // one file or directory by relative path
#import "sqlite3.h"           // a C header, relative to this file
#import <zlib.h>              // a C header from the include paths

#if target.os == "linux"
#import <std::sys::linux>     // imports may be conditional
#endif

using std::io::puts, std::io::fputs;   // bring single names into file scope unqualified
```

- `#import` takes **no semicolon**, and its operand is **never macro-expanded**.
- Imports and `using` declarations come **before every other declaration** in the file.
- A namespace import binds its **last component** (`io::`), or the name after `as`.
- Imports are **not re-exported**: importing a module doesn't give you that module's imports.
- There is **no glob `using`** (`using a::b::*`). Two imports or `using`s that bring in different things under the
  same name are an error; resolve namespace clashes with `as`.
- `using` is a declaration, not a directive: it ends with a semicolon.

### Qualified names

- `a::b::x` names `x` in namespace `a::b`, reachable through an import or the current namespace.
- A leading `::` names the root namespace, where C headers' declarations live: `::sqlite3_open(...)`. You may also
  name them unqualified.
- Tags and enumerators belong to their namespace: `struct net::addr`, `gfx::RED`.

### Symbols

- An exported `a::b::f` becomes the linker symbol `a.b.f`. C can't spell a `.` in an identifier, so Primal C and C
  symbols never collide.
- `[[link_name("sym")]]` chooses the symbol yourself, to bind to an existing C function or to give a C-callable
  name.
- There is no overloading, so symbols never encode types.

### C headers

`#import "file.h"` preprocesses and parses the header **as C**, exactly as the C code that owns it sees it, and puts
its declarations in the root namespace. The header's macros (`SQLITE_OK`) are visible in the importing file only;
this is the one place macros cross files. Macros that would collide with Primal C keywords or prelude names
(`NULL`, `bool`, `true`, `false`) are not imported.

`#include` still exists, for plain textual inclusion of Primal C text into the including file.

---

## 3. Lexical changes

- Source is **UTF-8** (a leading BOM is ignored; invalid UTF-8 is an error). LF or CRLF line endings.
- **No trigraphs, no digraphs.** A `/*` inside a block comment is a warning.
- Identifiers: Unicode XID plus `_`, no `$`, normalized (NFC). **Names beginning with `__` are reserved** for the
  compiler; nothing else is reserved.

### Keywords

| Group | Keywords |
| --- | --- |
| From C | `auto break case char const continue default do double else enum extern float for goto if inline int long register restrict return short signed sizeof static struct switch typedef union unsigned void volatile while` |
| Clean spellings | `alignas alignof atomic bool false generic nullptr static_assert thread_local true typeof typeof_unqual` |
| C2y and TS 25755 | `countof defer` |
| New | `using assert maxof minof` |

No `_Keyword` spelling exists: `_Bool`, `_Atomic`, `_Generic`, `_Static_assert`, `_Thread_local`, `_Alignas`,
`_Alignof`, `_Noreturn`, `_Countof`, `_Imaginary`, `_BitInt`, `_Complex`, `_Decimal*` and `_Float*` are gone. Use
`bool`, `atomic`, `generic`, `static_assert`, `thread_local`, `alignas`, `alignof`, `[[noreturn]]`, `countof`, and
the library types for bit-precise integers, complex, decimal and half/quad floats. `constexpr` is not a keyword:
`const` covers it (section 6).

### Literals

| Form | Example | Notes |
| --- | --- | --- |
| Decimal, hex, binary, octal | `123`, `0x7F`, `0b1010`, `0o17` | **`017` is an error**: write `0o17`. `0` alone is zero. |
| Digit separators | `1'000'000`, `0xFFFF'0000` | |
| Integer suffixes | `u`, `l`, `ul`, `ll`, `ull`, `z`, `uz` | `z`/`uz` are `ptrdiff_t`/`size_t` |
| Floats | `1.5`, `0x1.8p3`, `2.0f`, `1.0l` | No complex or decimal suffixes |
| Characters | `'a'`, `u'x'`, `U'😀'` | **`'a'` has type `char`**, not `int`. `u` is UTF-16 (`unsigned short`), `U` UTF-32 (`unsigned int`). `u8` and `L` are errors. Multi-character constants are errors. |
| Strings | `"text"`, `u"text"`, `U"text"` | Type `char[N]`, **read-only: writing traps**. Plain strings are UTF-8. |
| Escapes | `\x{1F}`, `\o{17}`, `\u{1F600}` | Delimited escapes; an escape too large for its type is an error |

An unsuffixed integer literal is the first of `int`, `long`, `long long` that fits (hex, binary and octal also try
the unsigned types). A literal too large for `unsigned long long` (128 bits) is an error.

---

## 4. Types

### Integers: one width everywhere

| Type | Bits | Range | Replaces |
| --- | --- | --- | --- |
| `char` | 8 | 0 to 255 | `unsigned char`, `uint8_t` |
| `signed char` | 8 | −128 to 127 | `int8_t` |
| `short` | 16 | −32,768 to 32,767 | `int16_t` |
| `unsigned short` | 16 | 0 to 65,535 | `uint16_t`, `char16_t` |
| `int` | 32 | −2³¹ to 2³¹−1 | `int32_t` |
| `unsigned int` | 32 | 0 to 2³²−1 | `uint32_t`, `char32_t` |
| `long` | **64** | −2⁶³ to 2⁶³−1 | `int64_t`, `intmax_t` |
| `unsigned long` | 64 | 0 to 2⁶⁴−1 | `uint64_t` |
| `long long` | **128** | −2¹²⁷ to 2¹²⁷−1 | `__int128` |
| `unsigned long long` | 128 | 0 to 2¹²⁸−1 | `unsigned __int128` |

- **`char` is unsigned** on every target, and is the byte type. `signed char` is the signed byte.
- **`long long` is 128 bits.** In C it is 64. This is the most important difference when porting.
- **One spelling per type**: `unsigned char`, `long int`, `short int`, `signed int`, `int32_t`, `uint8_t` and the
  other `<stdint.h>` names are errors with a fix-it. `unsigned` alone still means `unsigned int`.
- `size_t` and `ptrdiff_t` follow the target's pointer width (`unsigned long` and `long` on 64-bit targets) and stand
  in for `uintptr_t`, `intptr_t` and `ssize_t`.
- **`maxof(T)` and `minof(T)`** give any integer type's limits as constants. They replace `INT_MAX`, `SIZE_MAX` and
  the rest.

### Other types

- **`bool`** is a built-in type (no header): 1 byte holding 0 or 1. Converting a scalar gives `false` for zero or
  null, `true` otherwise.
- **`float`** and **`double`** are IEEE 754 binary32 and binary64. `long double` follows the target (x87 80-bit on
  x86-64, binary128 on AArch64 Linux).
- **`nullptr`** and `nullptr_t` as in C23; **`NULL` is `nullptr`**.
- Bit-precise integers, complex numbers, decimal floats and `f16`/`f128` are **library types** (`std::bigint`,
  `std::complex`, `std::decimal`, `std::fp`), not language types.
- **Enumerations** may fix their underlying type: `enum color : char { RED, GREEN, BLUE };`. There are no scoped
  enums; namespaces already prevent collisions.
- **Arrays** decay to pointers as in C. `countof(a)` gives an array's element count (an error on a non-array).
  `int a[0]` is an error (use a flexible array member `[]`).
- **Structs** lay out as the platform ABI says. `[[packed]]` gives GCC-compatible packing. An empty struct has size 0.
  Reading a union member other than the last written reinterprets the bytes (that is how you bit-cast).

### Inference

```c
auto n = items_count(list);        // C23 auto
auto *p = &table[0];               // through a pointer declarator
typeof(n) total = 0;
typeof_unqual(*p) copy = *p;
```

`generic` selection (C11's `_Generic`) also accepts a type name as its controlling operand.

### Conversions

- Integer to a narrower signed type keeps the low bits.
- **Float to integer saturates** at the target type's limits; NaN converts to 0.
- Pointer to integer and back is the identity on the address.
- Implicit conversion between incompatible pointer types is an error; `void *` converts implicitly to and from any
  object pointer, as in C.
- The integer promotions and the usual arithmetic conversions are C's, unchanged.

---

## 5. Declarations and initialization

- **Every automatic object starts zeroed**, at block entry. `[[uninit]] char buffer[4096];` opts out; reading it
  before writing gives whatever bytes are there.
- `= {}` zeroes the whole object, padding included.
- Designated initializers as in C99, including **ranges**: `int table[16] = { [0 ... 7] = 1 };`. A repeated designator
  is allowed (the last wins) with a warning.
- Compound literals may be `static`, `thread_local` or `register`: `(static int[]){ 1, 2, 3 }`.
- **No tentative definitions**: `int x;` at file scope *defines* a zero `x`, and a second definition anywhere is an
  error.
- `register` is accepted and deprecated.
- `extern` is only needed for foreign C symbols with no header to import.
- `alignas` and `alignof` (`alignof` also takes an expression).

### `const`: compile-time constants and functions

`const` replaces most macros:

```c
const int BUFFER_SIZE = 4096;                   // a compile-time constant, typed and namespaced
const struct point ORIGIN = { 0, 0 };            // of any type

const int square(int x) {                        // a const function: return type const-qualified
	return x * x;
}

const int CELLS = square(64);                    // evaluated at compile time
static char grid[CELLS];
static_assert(CELLS == 4096, "grid size");
```

- A `const` object with a constant initializer is a compile-time constant of any type: it can size arrays, label
  cases, set bit-field widths and enumerator values, and appear in `static_assert`.
- A function whose **return type is `const`-qualified at the top level** is a `const` function. It runs at compile
  time where a constant is required and its arguments are constant; anywhere else it's an ordinary call. (For a
  pointer result put `const` on the pointer: `char *const f(...)`. `const char *f(...)` is an ordinary function
  returning a pointer to `const char`.)
- A `const` function may use locals, loops, other `const` functions and `const` objects; `static` locals, non-`const`
  globals, `volatile`, `asm`, allocation, I/O and non-`const` calls are compile errors.
- Exported `const` objects and `const` function bodies are visible to importers' constant expressions.

### Attributes

C23's `[[...]]` syntax. Unknown attributes are warnings.

| Attribute | Meaning |
| --- | --- |
| `deprecated`, `fallthrough`, `maybe_unused`, `nodiscard`, `noreturn` | As in C23 |
| `unsequenced`, `reproducible` | As in C23, granting the optimizer nothing |
| `assume(expr)` | The optimizer may assume `expr` (not evaluated; a false assumption is like reaching `unreachable()`) |
| `uninit` | Skip zero-initialization |
| `packed` | No padding, GCC's layout |
| `link_name("sym")` | The emitted symbol name |
| `export(...)` | Export from a shared library product |
| `weak`, `section("name")` | As GCC's |
| `always_inline`, `noinline`, `cold` | Optimization hints |
| `wraps` | Marks intentional wraparound, so the overflow lint stays quiet |
| `fp_contract` | Allow FMA contraction in this function |
| `fenv_access` | This function reads or changes the floating-point environment |

---

## 6. Expressions: one meaning each

Operators and precedence are C's, with `::` added at the top. What changes is that every expression has exactly one
meaning.

| Situation | C | Primal C |
| --- | --- | --- |
| Order of evaluation | Unspecified, sometimes undefined | **Left to right**, function arguments included; each operand's side effects finish before the next starts |
| `i = i++ + 1`, `a[i] = i++` | Undefined | Defined (left operand's address, then the right side, then the store), with a warning |
| Signed overflow | Undefined | **Wraps** (two's complement) |
| Division or `%` by zero | Undefined | **Traps** |
| `INT_MIN / -1`, `INT_MIN % -1` | Undefined | `INT_MIN` and `0` |
| Shift by a variable count ≥ width or negative | Undefined | **Count masked** to `width − 1` bits; a constant out-of-range count in source is an error |
| Left shift of a negative value | Undefined | Operates on the bit pattern; right shift of a negative value is arithmetic |
| `-1 < 0u` | `false` (converts to unsigned) | **`true`: signed/unsigned comparisons compare the mathematical values** |
| Float to int out of range, NaN | Undefined | Saturates; NaN gives 0 |
| Pointer arithmetic out of bounds, comparing unrelated pointers | Undefined | Address arithmetic; any result may be computed and compared |
| `NULL + 0`, `NULL - NULL` | Undefined (C23) | `NULL` and `0` |
| Dereferencing null | Undefined | **Traps** (member access through null too, whatever the offset) |
| Reading uninitialized automatic memory | Indeterminate | Zero (automatics start zeroed) |
| Strict aliasing | Undefined to violate | **No strict aliasing**: any object may be accessed through any lvalue type |

- **Floating point** is IEEE 754 throughout. There is **no FMA contraction** unless a function has `[[fp_contract]]`,
  every operation rounds to its type, and fast-math is never a default. Code that runs under another rounding mode
  or with traps enabled marks itself `[[fenv_access]]`.
- `&p->m` through a null `p` is address arithmetic and touches nothing.
- Casting away `const` is allowed; writing through the result to a static `const` object traps (it lives in
  read-only memory).
- **No statement expressions** (`({ ... })`): use a `static inline` function.
- Checked arithmetic: `ckd_add`, `ckd_sub`, `ckd_mul` from `std::ckd`.

---

## 7. Statements

### `if` and `switch` declarations (C2y)

```c
if (int fd = os::open(path, os::RDONLY); fd < 0) {
	return -1;
} else {
	use(fd);              // fd is in scope in every branch
}
```

### `switch`

- **Case ranges**: `case 'a' ... 'z':`. A range with low > high, or overlapping cases, is an error.
- A case value that changes when converted to the controlling type is an error; so are duplicates.
- Falling into a non-empty case without `[[fallthrough]]` is a warning.

### Named loops (C2y)

```c
outer: for (int i = 0; i < h; i++) {
	for (int j = 0; j < w; j++) {
		if (grid[i][j] == target)
			break outer;       // continue outer also works
	}
}
```

### `defer` (TS 25755)

```c
int copy(const char *src, const char *dst) {
	io::FILE *in = io::fopen(src, "rb");
	if (!in)
		return -1;
	defer io::fclose(in);

	io::FILE *out = io::fopen(dst, "wb");
	if (!out)
		return -1;             // closes in
	defer io::fclose(out);

	// ... copy ...
	return 0;                  // closes out, then in
}
```

- A deferred statement runs when control leaves the **enclosing block**: at its end, or by `return`, `break`,
  `continue` or `goto`. Deferred statements run in reverse order. A `return` expression is evaluated first.
- They don't run on `longjmp`, `exit` or a trap. A deferred statement can't itself `return`, `break`, `continue` or
  `goto` out.
- Jumping forward past a `defer` into its block is an error.

### Other rules

- Labels may precede declarations and close a block (C23).
- `goto` is kept. Jumping past an initialized declaration is allowed (the object holds its zero value); jumping
  into a VLA's scope is an error.
- **A non-`void` function must return a value on every path** (or reach a `[[noreturn]]` call); otherwise it's an
  error. `main` falling off its end returns 0.
- Unreachable code after `return`, `break`, `continue` or `goto` is a warning.

---

## 8. Functions

- Every function has a prototype; **`f()` means `f(void)`**. K&R definitions are errors. Parameters may be unnamed.
- **No overloading**, default arguments, references, nested functions or lambdas. Type-generic interfaces use
  `generic` selection.
- Variadics use `...`; `f(...)` with no named parameter is allowed. `va_list`, `va_start(ap)`, `va_arg`, `va_copy`
  and `va_end` are built in (no header). Reading a missing or mistyped argument reads that slot's bytes; a
  `va_list` never started traps.
- `inline` is a hint. An exported `inline` function is emitted once, by its module; importers may inline its body.
- `[[noreturn]]`: a reachable return is an error, and the function traps if it ever reaches its end.
- **Stack overflow traps** (guard page plus probes).
- `main` is `int main(void)` or `int main(int argc, char **argv)`, in any module; returning `n` calls `exit(n)`.
- **Inline assembly** uses named bindings: each operand is bound with `in`, `out` or `in out` to a register,
  `register`, `memory` or `constant`, and `changes` lists what else the block alters.

---

## 9. The preprocessor

The preprocessor keeps C's token-level macros and conditional compilation and drops most of the historical
machinery.

| Directive | Purpose |
| --- | --- |
| `#namespace` | The file's namespace: first directive, once, never in a conditional group |
| `#import` | A namespace, path or C header; allowed in conditional groups; never macro-expanded |
| `#include` | Textual inclusion of Primal C text |
| `#if`, `#elif`, `#else`, `#endif`, `#ifdef`, `#ifndef` | Conditional compilation |
| `#define`, `#undef`, `#`, `##` | Macros, stringification, pasting |
| `#embed` | Binary data as an initializer (C23) |
| `#error`, `#warning`, `#line` | Diagnostics and locations |
| `#pragma once` | The only pragma |

- **Not available**: `#elifdef`, `#elifndef`, `#include_next`, `#ident`, `_Pragma`, and every pragma but `once`.
  Pragmas give way to attributes (`#pragma pack` → `[[packed]]`) and build configuration.
- **Build configuration** is typed and read-only: `target.os`, `target.arch`, `target.abi`, `build.type`, and user
  values `env.<name>` from the build file. Compare strings with `==`: `#if target.os == "linux"`. The same values
  are visible to code as `env::name` constants.
- **An unknown identifier in `#if` is an error**, not 0.
- **Macros are per file**: they never cross `#import`. A macro is visible from its definition to the end of the file
  and in files that `#include` it.
- Use **`const` objects and `const` functions** for constants and computation; keep macros for token-level work.
- `__FILE__`, `__LINE__`, `__COUNTER__`, `__STDC__`, `__STDC_VERSION__` (`202311L`), `__PRIMAL_C__` (the revision,
  `1`) and `__NAMESPACE__` exist; **`__DATE__`, `__TIME__` and `__TIMESTAMP__` don't** (builds are reproducible).

---

## 10. Memory, concurrency and signals

- Memory is **bytes at addresses**. A load returns the bytes that are there. Integer-to-pointer round trips work for
  any object whose address has been exposed.
- **No strict aliasing.** Misaligned access works (except for atomics, where it traps).
- Struct assignment copies every byte, padding included; an assignment between overlapping objects acts as if
  through a temporary.
- `restrict` is an explicit opt-in, with C11's formal meaning.
- Accessing memory outside its lifetime (a dangling pointer) or out of bounds is **Bounded**: it does whatever the
  loads and stores do, but the compiler never assumes it can't happen.
- `volatile`: each access is exactly one load or store, never merged, split or elided.
- **Atomics**: C11's memory model with `atomic` and the operations in `std::atomics`. Atomics of 1–8 bytes are
  lock-free; 16-byte atomics are library calls.
- **Data races** are *Erroneous*: a racing load returns some stored value; the compiler never invents stores or
  re-loads a value it already read.
- **Threads** are C11's in `std::thread`, with **error-checking mutexes**: unlocking a mutex you don't own, or joining
  twice, traps.
- **Signals**: a handler may call any `std::` function; re-entering a subsystem the handler interrupted on the same
  thread (the allocator, the same stream) traps instead of deadlocking.
- **`setjmp`** has true returns-twice semantics; locals keep their last stored values after `longjmp`.

---

## 11. What "defined" means

Primal C replaces each of C's undefined behaviors with one of five outcomes:

| Term | Meaning | Example |
| --- | --- | --- |
| **Defined** | The specification states the result | Signed overflow wraps |
| **Error** | Compile-time diagnostic; compilation stops | `017`, a missing return, a negative constant shift |
| **Trap** | The program terminates at the faulting operation; nothing after it runs | Division by zero, null dereference, double `free` |
| **Erroneous** | Wrong, but bounded and stable; a checked build may trap | Reading an `[[uninit]]` variable, a data race |
| **Bounded** | The only remaining undefined behavior: limited to what the emitted loads and stores do to memory | A dangling pointer, out-of-bounds access |

Two rules follow from this: the compiler never infers anything from a fault (it can't delete or reorder code because
an operation "must" have been valid), and observable behavior before a fault really happens before it. The only ways
to grant the optimizer undefined-behavior-style freedom are explicit: `restrict`, `unreachable()` and `[[assume]]`.

---

## 12. The standard library

The library is C's library under namespaces, with the BSD and POSIX functions C programmers already use, and a
handful of new ones. **The library's specification is a draft** ([`reference/stdlib.md`](reference/stdlib.md)), so
names outside ISO C may still change.

### Conventions

- **ISO C, POSIX and BSD functions keep their C names**, in the module of their header: `io::printf`,
  `str::strlcpy`, `lib::qsort_r`, `err::EPERM`.
- **New functions have plain English names** in modules of their own, without prefixes: `hash::bytes`,
  `table::find`, `rand::split`, `arena::alloc`.
- **Everything is defined for every input**: preconditions checkable in constant time trap if violated, and
  overlapping arguments act as if copied through a temporary (`memcpy` behaves as `memmove`; `sprintf` into its own
  argument works).
- **Caller-sized buffers** take a size and return the length they would have written, as `snprintf` and `strlcpy`
  do.
- **No hidden allocation**: only functions whose job is allocating allocate.
- **The same results everywhere**: `rand`, hashing, table iteration order, number formatting and parsing, and the
  math functions give the same bits on every target.

### Modules

| Module | From C | Highlights |
| --- | --- | --- |
| `std::io` | `<stdio.h>` | `printf` family (formats checked at compile time; `%lld` is 128-bit), `getline`, `asprintf`, `dprintf`, `open_memstream` (the string builder), `fmemopen`, `fopencookie`, `%m`. No `gets`, no wide I/O. Text and binary streams are the same. |
| `std::lib` | `<stdlib.h>` | `malloc` family (bad, double and foreign frees trap; `malloc(0)` is a unique pointer), `reallocarray`, `qsort` and `qsort_r` (never allocate, never go out of bounds), `bsearch_r`, `lower_bound`, `upper_bound`, `mergesort`, `strto*` (saturating), `strshortestd`, `arc4random`, `getenv`/`setenv`, `strdup` |
| `std::str` | `<string.h>`, `<strings.h>` | All of C23's plus `strlcpy`, `strlcat`, `memmem`, `strnlen`, `stpcpy`, `strtok_r`, `strsep`, `strchrnul`, `mempcpy`, `strcasecmp`; comparisons return −1, 0 or 1 |
| `std::math` | `<math.h>` | Correctly rounded or pinned to one algorithm per function: the same bits on every target. Never sets `errno`. `lgamma_r` replaces `signgam`. |
| `std::fenv`, `std::complex`, `std::fp` | `<fenv.h>`, `<complex.h>` | Floating-point environment; complex numbers as structs; `f16`/`f128` |
| `std::ctype`, `std::uchar` | `<ctype.h>`, `<uchar.h>` | ASCII classification; UTF-8 helpers `utf8_decode`, `utf8_encode`, `utf8_count`, `utf8_valid_length` |
| `std::err`, `std::limits` | `<errno.h>`, `<limits.h>` | `errno` (a `thread_local int`), `err::EPERM`…, `strerror`; float limits as constants (integer limits are `maxof`/`minof`) |
| `std::time` | `<time.h>` | C23's clocks and `timegm`, POSIX clocks and sleeps, `strftime`, `strptime`, time zones |
| `std::thread`, `std::atomics` | `<threads.h>`, `<stdatomic.h>` | C11 threads plus monotonic timed waits, read-write locks, semaphores, barriers, thread names; atomics as builtins |
| `std::bit`, `std::ckd` | `<stdbit.h>`, `<stdckdint.h>` | `stdc_*` bit functions and `ckd_*` checked arithmetic, as builtins |
| `std::hash`, `std::table` | — | `hash::bytes`, `hash::keyed`; a flat hash table shaped like `qsort` (`init`, `find`, `insert`, `remove`, `next`) |
| `std::rand` | — | A seeded generator on your own state (`seed`, `next`, `below`, `fraction`, `split`, `shuffle`) |
| `std::arena` | — | Region allocation (`create`, `alloc`, `mark`, `release`, `reset`, `destroy`) |
| `std::program` | — | `getprogname`, `err`/`warn`, `getopt`, `getopt_long` |
| `std::bigint`, `std::decimal` | `_BitInt`, `_Decimal*` | Fixed-size big integers and decimal floats as library types |

`<stddef.h>`, `<stdarg.h>`, `<stdint.h>`, `<stdbool.h>` and `<assert.h>` have no module: their contents are built in
(`size_t`, `ptrdiff_t`, `nullptr_t`, `max_align_t`, `va_*`, `offsetof`, `unreachable()`, `NULL`) or are keywords
(`bool`, `assert`). A built-in prelude name may be shadowed, with a warning.

### Example

```c
#namespace wordcount

#import <std::io>
#import <std::lib>
#import <std::table>
#import <std::str>

struct count { char *word; long n; };

int main(int argc, char **argv) {
	struct table::table words;
	if (!table::init(&words, sizeof(struct count), sizeof(char *), table::KEY_STRING, 0, nullptr, nullptr))
		return 1;
	defer table::destroy(&words);

	char *line = nullptr;
	size_t capacity = 0;
	defer lib::free(line);
	while (io::getline(&line, &capacity, io::stdin) > 0) {
		char *rest = line;
		for (char *w; (w = str::strsep(&rest, " \t\n")) != nullptr;) {
			if (*w == 0)
				continue;
			bool added;
			struct count *c = table::insert(&words, w, &added);
			if (added)	// the key is the element's first member: point it at a copy that outlives the line
				c->word = str::strdup(w);
			c->n++;
		}
	}

	size_t pos = 0;
	for (struct count *c; (c = table::next(&words, &pos)) != nullptr;)
		io::printf("%8ld %s\n", c->n, c->word);
}
```

---

## 13. Working with C

Kyogre compiles C files as **C units** in their own standard (K&R to C2y and the GNU modes), next to Primal C files,
in one program. Calls between them use the platform C ABI.

- **Primal C calling C**: `#import "header.h"`, then call the functions unqualified or with `::`. A hand-written
  `extern` declaration with no header is trusted.
- **C calling Primal C**: give the function a C-spellable symbol with `[[link_name("geo_add")]]`, or generate a C
  header for a module with `ky header`.
- **Types across the boundary** keep their C identity. C's `long long` (64-bit) is Primal C's `long`; Primal C's
  128-bit `long long` crosses as `__int128` where the target has it, and can't cross where it doesn't.
- **C's plain `char`** maps to `char`; its signedness follows the C side's ABI at the boundary.
- C units keep C's rules: their undefined behavior is neither defined nor exploited.

---

## 14. Porting a C file

1. Add `#namespace a::b` as the first directive (or leave it out to stay in the root namespace). Every Primal C file in
   a directory must make the same choice.
2. Replace each `#include <x.h>` with `#import <std::x>` (`<stdio.h>` → `std::io`, `<stdlib.h>` → `std::lib`,
   `<string.h>` → `std::str`, `<math.h>` → `std::math`, `<errno.h>` → `std::err`...), and project headers with
   `#import` of the module that defines them.
3. Qualify library calls (`io::printf`) or add `using` declarations.
4. Turn object-like macros into `const` objects and function-like ones into `const` or `inline` functions.
5. Delete forward declarations and prototypes that only existed for ordering.
6. Rewrite `017`-style octal as `0o17`, and fix the other spellings the compiler flags (`unsigned char` → `char`,
   `int64_t` → `long`, `_Bool` → `bool`, `INT_MAX` → `maxof(int)`).
7. Build with `--warn port` and review what it reports: **`long long` and `%lld` widths (128 bits now)**, plain `char`
   signedness (unsigned now), character constant types (`char` now), and signed/unsigned comparisons (mathematical
   now).

---

## 15. Style

- **Plain names.** Name functions and types for what they mean or do, in plain English. Keep C's names where C
  already has one.
- **One module per directory**, one public function (or a tight group) per file in libraries. Shared helpers that
  aren't API go in an `internal` namespace.
- **Imports at the top**, grouped: standard library, then project modules, then C headers.
- **Qualify library calls** (`io::printf`) rather than adding many `using` declarations; reserve `using` for names
  used constantly in a file.
- **`defer` for cleanup**, right after the acquisition it undoes.
- **`const` over macros**, `const` functions over function-like macros.
- **Length-carrying calls**: prefer `snprintf`, `strlcpy`, `memchr`/`memmem` with lengths, and `%.*s` for counted
  strings.
- **Let zero-initialization work**: don't write `= 0` or `memset(&x, 0, ...)` for locals; use `[[uninit]]` only for
  large buffers on measured hot paths.
- **Check results**: allocation, I/O and conversion functions report failure through their return value and
  `errno`, as in C.
- Prefer `bool`, `nullptr`, `countof`, `maxof`/`minof` and `static_assert` over their old spellings.

---

## 16. Pitfalls

- **`#namespace` and `#import` take no semicolon**, and `#namespace` must be the first directive.
- **Imports go before every declaration**, including conditional ones.
- **`long long` is 128 bits** and **`char` is unsigned**: watch format strings, serialization and bit tricks ported
  from C.
- **`'a'` is a `char`**, so `sizeof 'a'` is 1.
- **`017` is an error**: octal is `0o17`.
- **`f()` means `f(void)`.**
- **`int x;` at file scope is a definition**, not a tentative one: declare shared objects once.
- **Non-`static` means exported.** Use `static` (or an internal namespace) for helpers.
- **Signed/unsigned comparisons are mathematical** (`-1 < 0u` is true), which differs from C.
- **Shift counts are masked**: `x << 64` on a 64-bit `x` shifts by 0, not "to zero". Write the edge case explicitly.
- **A C header's macros expand inside your file**, including after `::`. If you import a header that defines a macro
  with the same name as a qualified identifier you use (for example `E2BIG` with `err::E2BIG`), the macro wins. Avoid
  importing such headers into files that use the qualified names (a known issue under discussion).
- **No `__DATE__`/`__TIME__`**, no statement expressions, no pragmas but `#pragma once`.

---

## 17. Quick reference

```c
#namespace app::tools                      // file's namespace (first directive)

#import <std::io>                          // module import: io::
#import <std::lib> as lib                  // explicit alias
#import "legacy.h"                         // C header: root namespace (::)

using std::io::printf;                     // one name, unqualified

const int LIMIT = 1'000;                   // compile-time constant
const int twice(int x) { return 2 * x; }   // compile-time-capable function

enum state : char { IDLE, BUSY };

static int helper(long n);                 // optional: order never matters at file scope

int run(const char *path) {
	if (auto *f = io::fopen(path, "r"); f == nullptr) {
		return -1;
	} else {
		defer io::fclose(f);
		long total = 0;                    // 64-bit
		for (int c; (c = io::fgetc(f)) != io::EOF;)
			total += c == '\n';
		printf("%ld lines (max %d)\n", total, maxof(int));
	}
	return 0;
}

static int helper(long n) {
	switch (n) {
	case 0 ... 9:   return 1;
	case 10 ... 99: return 2;
	default:        return countof("xx");  // 3
	}
}
```
