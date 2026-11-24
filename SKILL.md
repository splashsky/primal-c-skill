---
name: primal-c
description: Write, read, review and port Primal C, the C dialect of the Kyogre toolchain (`ky`). Use when a file begins with `#namespace` or uses `#import <std::...>`, when code calls the `std::` library (`io::printf`, `str::strlcpy`, `table::find`), when porting C to Primal C, or when asked about Primal C's syntax, types, modules, defined behavior or standard library.
---

# Writing Primal C

Primal C (revision 1) is C23 with C2y's additions, modules with namespaces instead of headers, and every behavior C
leaves undefined given a meaning. It is still C: same declarators, operators, calling convention and idioms.

**Read [`GUIDE.md`](GUIDE.md)** for the language: it is organized so you can jump to the part you need (modules §2,
literals §3, types §4, `const` and attributes §5, expression semantics §6, `defer` and named loops §7, the
preprocessor §9, the standard library §12, C interop §13, porting §14, style §15, pitfalls §16, a quick reference
§17). The full specification is [`reference/lang.md`](reference/lang.md); where the guide and it differ, the
specification wins. The library's draft specification is [`reference/stdlib.md`](reference/stdlib.md).

## The rules that catch people

1. **Module structure is directives, without semicolons.** `#namespace a::b` is the file's first directive (once,
   never inside `#if`); `#import <std::io>` (or `as name`, `"path"`, `"header.h"`) comes before every declaration.
   `using a::b::f;` is a declaration and does take a semicolon.
2. **File scope is order-independent within a module**: no prototypes or forward declarations. **Everything not
   `static` is exported.**
3. **Fixed widths**: `char` is the *unsigned* byte, `long` is 64 bits, **`long long` is 128 bits**. One spelling per
   type: no `unsigned char`, `int64_t`, `uint8_t`, `_Bool`; use `char`, `long`, `bool`, `maxof(T)`/`minof(T)`.
4. **Clean keywords**: `bool`, `nullptr`, `atomic`, `generic`, `countof`, `defer`, `static_assert`, `thread_local`,
   `alignas`, `typeof`; no `_Keyword` spellings. `NULL` is `nullptr`.
5. **Literals**: `0o17` (not `017`, an error), `0b1010`, `1'000'000`; `'a'` has type `char`; no `u8` or `L` prefixes;
   string literals are read-only.
6. **Every object starts zeroed**; `[[uninit]]` opts out. `int x;` at file scope is a definition.
7. **One meaning per expression**: left-to-right evaluation; signed overflow wraps; shift counts are masked;
   signed/unsigned comparisons compare mathematical values (`-1 < 0u` is true); float-to-int saturates; division by
   zero and null dereference trap; no strict aliasing.
8. **`const` replaces macros**: `const` objects are compile-time constants of any type; a function with a
   `const`-qualified return type (`const int f(...)`) can run at compile time. Macros never cross `#import`.
9. **Statements**: `if (int x = f(); x > 0)`, `case 1 ... 9:`, `break outer;` on a labeled loop, and `defer stmt;`
   (runs at block exit, in reverse order).
10. **Library**: C's names in namespaced modules (`io::printf`, `lib::malloc`, `str::strlcpy`, `err::errno`); new
    functions have plain English names (`hash::bytes`, `table::insert`, `rand::below`). Library calls are defined for
    every input: bad arguments trap, overlap is safe, `memcpy` behaves as `memmove`.

## When writing code

- Start each file with `#namespace`, then imports grouped (standard library, project modules, C headers), then
  declarations.
- Qualify library calls (`io::printf`); use `using` only for names a file uses constantly.
- Put `defer` right after the acquisition it undoes.
- Use `const` objects and functions instead of `#define` constants and function-like macros.
- Prefer length-carrying calls (`snprintf`, `strlcpy`, `memmem`, `%.*s`) and check results as in C.
- Mark helpers `static`, or put them in an `internal` namespace when several files of a module share them.
- When porting, follow GUIDE.md §14 and watch the 128-bit `long long`, unsigned `char`, `char`-typed character
  constants and mathematical signed/unsigned comparisons.

## When reviewing code

Check for: a semicolon after `#namespace`/`#import`; imports after declarations; C type spellings (`uint32_t`,
`unsigned char`, `_Bool`); leading-zero octal; `long long` used where 64 bits was meant; reliance on C's
signed/unsigned conversion in comparisons; macros where `const` would do; helpers accidentally exported; cleanup
paths that `defer` would make simpler; missing `return` on a path (an error in Primal C).
