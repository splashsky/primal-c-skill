<!-- Copied from Kyogre's specification (docs/compiler-research/kyogre/lang.md at 6ba1e99). Relative links point into that repository and don't resolve here. -->

# Primal C: Language Specification, revision 1

Nov 17, 2026 · @Sky Johnson

Primal C unifies C99, C11, C23 and C2y into one dialect: no headers, namespaced modules, and every undefined behavior defined, diagnosed or bounded, with no runtime.

## 1. Overview

C23 is the base text. Accepted C2y features are adopted, and C99/C11 rules carry through unchanged unless this spec says otherwise. Where a choice is still open, the spec states a **suggested default**; all of them are collected in section 16.

**Goals**

- Reads as C: same declarators, operators, precedence, braces and semicolons.
- No headers for declarations: modules, namespaces and `#import` replace them. `#include` stays only as plain textual inclusion.
- No undefined behavior that can be defined, diagnosed or trapped without a runtime.
- No runtime: no GC, no exceptions, no hidden allocation, no implicit bounds checks on raw pointers.
- Platform C ABI (SysV x86-64 first), so existing C objects and libraries link unchanged.
- Human-first diagnostics: every error, warning and log line is friendly, accurate and consistent, with dense formats for tools that ingest it (Front End Spec: Lexer, Preprocessor, Parser, section 10).

**Baseline**

| Standard | What this spec takes |
| --- | --- |
| C99 | Designated initializers, `for` declarations, `restrict`, `//` comments, compound literals, flexible array members, VLAs (section 5). |
| C11 | `generic` selection (ISO C's `_Generic`), anonymous structs and unions, atomics, threads, `alignas`, `static_assert`. Annex K is dropped. |
| C23 | Base text: clean keyword spellings (`bool`, `nullptr`, `typeof`, `static_assert`), `auto` inference, `[[attributes]]`, `#embed`, binary literals, digit separators, checked integer and bit utilities. `_BitInt`, decimal and interchange float types move to the library, and `constexpr` gives way to compile-time `const`. |
| C2y draft | `countof`, named loops, `if` declarations, case ranges, `0o` octals, zero-length null operations ([GCC C2y status](https://gcc.gnu.org/projects/c-status.html)), plus `defer` from [TS 25755](https://www.open-std.org/JTC1/SC22/WG14/www/docs/n3928.pdf). |

**Semantic contract.** This is the rule everything else in Primal C units follows from; C units follow the contract in section 14.1.

- No inference from faults: the compiler may not assume an operation was valid in order to delete or reorder other code.
- No time travel: observable behavior sequenced before a fault happens before it.
- Concrete memory: memory is bytes at addresses, and a load returns the bytes that are there. No `undef` or `poison` values exist in the semantics.
- Assumptions are opt-in and local: `restrict`, `unreachable()` and `[[assume]]` are the only ways to grant the optimizer UB-based freedom. A false `[[assume]]` grants the same freedom as reaching `unreachable()`.

**Defining undefined behavior.** Every behavior ISO C leaves undefined is defined in Primal C. Among the definitions that keep the program meaningful, Primal C takes the least restrictive one that is also the most obvious and predictable. Each ruling follows one of four clauses:

1. **Obvious result.** Where the result is obvious, it is defined, never rejected or trapped: integers wrap, an operation gives the plain machine or IEEE 754 result, and an overlapping copy acts as if made through a temporary.
2. **No meaning.** An Error only where a construct has no sensible meaning to define, so rejecting it is the obvious behavior. Typical cases are malformed declarations and the constraints C2y itself adds. Where C2y replaces an undefined behavior with a constraint, Primal C takes the constraint, unless this specification gives the construct a meaning.
3. **Corrupted state.** A Trap where continuing could only corrupt state: a use after `free` the library can see, such as a block freed twice; a deadlock that would hang silently; a mutex unlocked by a thread that doesn't own it.
4. **Wild access.** Bounded only for the memory effects of wild accesses the compiler can't see: loads, stores and calls through pointers to memory the program doesn't own.

The rule is for Primal C units. C units' undefined behavior is neither defined nor exploited (section 14.1).

**Conformance terms**

- **Error**: a translation-time diagnostic that stops compilation.
- **Trap**: the program terminates at the faulting operation; nothing after it executes.
- **Erroneous**: the program is wrong, but the result is bounded and stable; a checked build may trap.
- **Bounded**: the only remaining undefined behavior, limited to what the emitted loads and stores do to memory.

A ruling that names none of these is **Defined**: this specification states the result. Erroneous is a defined result for a program that is almost certainly wrong, so it falls under clause 1. Error, Trap and Bounded are clauses 2, 3 and 4.

Every rule here that replaces an ISO C undefined behavior is catalogued in [Defining C's Undefined Behavior](ub.md), with its clause, rationale and optimizer cost.

## 2. Lexical structure

Source is UTF-8 and tokens are C23's, plus a few new keywords for modules and control flow, with the lexical traps removed.

**2.1 Source text**

- Files are UTF-8; a leading BOM is ignored and invalid UTF-8 is an error.
- LF or CRLF line endings. A missing final newline is accepted; a file ending in backslash-newline is an error.
- Comments are `/* */` and `//`. A `/*` inside a block comment is a warning.
- Trigraphs don't exist (C23). Digraphs are removed (suggested default).

**2.2 Identifiers**

- Unicode XID\_Start / XID\_Continue per C23, plus `_`. No `$`. An identifier not in Unicode Normalization Form C is an error (C2y).
- Every character is significant; there is no length limit.
- Names beginning with `__` are reserved for the compiler: declaring one, `__func__` included, or defining or undefining a macro so named, is an error. Nothing else is reserved: namespaces prevent stdlib collisions.

**2.3 Keywords**

- From C: `auto break case char const continue default do double else enum extern float for goto if inline int long register restrict return short signed sizeof static struct switch typedef union unsigned void volatile while`
- Clean spellings: `alignas alignof atomic bool false generic nullptr static_assert thread_local true typeof typeof_unqual`. `atomic` and `generic` respell C11's `_Atomic` and `_Generic`. C23's `constexpr` is not a keyword: `const` covers it (section 6.3).
- From C2y and TS 25755: `countof` (C2y spells it `_Countof` behind a header; here it's a plain keyword) and `defer`.
- New: `using`, `assert`, `maxof`, `minof`.
- No `_Keyword` spelling exists in Primal C code: `_Alignas`, `_Alignof`, `_Atomic`, `_BitInt`, `_Bool`, `_Complex`, `_Countof`, `_Decimal32/64/128`, `_FloatN`, `_Generic`, `_Imaginary`, `_Noreturn`, `_Static_assert` and `_Thread_local` are all gone. Bit-precise integers, complex, decimal and half/quad floats are library types (section 13). Legacy units keep the spellings of their own standard (section 14).

**2.4 Integer literals**

- Decimal `123`, hex `0x7F`, binary `0b1010`, octal `0o17`.
- A `0` followed by digits (`017`) is an error (suggested default; C2y only obsoletes it). `0` alone is decimal zero.
- Digit separators: `1'000'000`, `0xFFFF'0000`.
- Suffixes: `u`, `l`, `ul`, `ll`, `ull`, and `uz`/`z` for `size_t` and `ptrdiff_t` (suggested default, matching C++23).
- An unsuffixed literal takes the first of `int`, `long` and `long long` that fits; hex, binary and octal also try the unsigned types, as in C. A literal too large for `unsigned long long` is an error.

**2.5 Floating literals**

- Decimal and hex (`0x1.8p3`) forms; suffix `f` for `float`, `l` for `long double`, none for `double`.
- No complex or decimal literal suffixes: those are library types (section 13).

**2.6 Character and string literals**

- Prefixes: none (`char`), `u` (`unsigned short`, UTF-16) and `U` (`unsigned int`, UTF-32). A character constant has its prefix's type, so `'a'` is a `char`, not an `int` as in C. `u8` is redundant because plain literals are already UTF-8, and `L` has a target-dependent width; both are errors with a fix-it.
- Delimited escapes `\x{1F}`, `\o{17}`, `\u{1F600}` (C2y). An escape whose value doesn't fit its character type is an error, and so is an undelimited `\x` followed by more digits than fit.
- Undelimited octal escapes other than `\0` are deprecated.
- Multi-character constants (`'ab'`) are an error in module code.
- Adjacent string literals concatenate; mixing different encoding prefixes is an error.
- String literals have type `char[N]` and live in read-only memory: writing to one traps.

## 3. Modules and namespaces

A module is every file that declares the same namespace, and all files in one directory share one. `#import` brings in declarations, as a directive beside `#include`, which stays for plain textual inclusion, `::` qualifies names, and C's own linkage rules decide what importers can see.

**3.1 Namespace declaration**

- A file joins a namespace with the directive `#namespace a::b`, which ends at the end of the line and takes no semicolon. It must be the file's first directive, before any token but comments and blank lines, at most once per file, and never inside a conditional group: a file's namespace can't depend on configuration, because the namespace index and the skim pass read it without evaluating `#if`. In text pasted by `#include`, `#namespace` is an error: the file's own line decides (suggested default). A file with no `#namespace` is still Primal C code, in the root namespace: its names are emitted unmangled, as in C. Whether a file is Primal C or legacy C is set by the build configuration, never inferred from its contents (section 14).
- All Primal C files in one directory must declare the same namespace, or all declare none; mixing namespaces in a directory is an error. Each subdirectory follows the same rule on its own and may declare a different namespace.
- A namespace may span several directories, and their files form one module. `#import <geo::mesh>` finds them wherever they live under the build unit's root (section 3.3), so namespace paths don't have to match directory paths.
- `::` is already a C23 token (attribute prefixes like `[[gnu::packed]]`), so the grammar gains no new punctuation.

**3.2 Visibility**

- File-scope declarations are visible across every file of the module, in any order. No forward declarations or prototypes are needed at file scope; inside function bodies, C's declare-before-use rule stands.
- `static` keeps its C meaning: visible in this file only.
- Everything else is visible to importers. There is no `export` keyword, and `static` is the only way to hide a name from them. Dynamic export from a product is separate: `exports = "default"` or `"list"` and `[[export]]` (section 14.4).
- A name may be declared more than once only if every declaration agrees: compatible types with identical qualifiers, the same alignment specifier and the same `[[noreturn]]`. A disagreeing declaration is an error, whichever file holds it.
- Struct, union and enum tags and their enumerators belong to the namespace like any other name: `struct net::addr`, `gfx::RED`.

**3.3 Import and using**

- `#import <std::io>` imports a namespace by name. Like every module-structure line it is a directive: it ends at the end of the line, takes no semicolon, and its operand is never macro-expanded (unlike `#include`'s computed form), so the index and tools read imports without preprocessing. It may sit inside a conditional group (`#if target.os == "linux"` then `#import <std::sys::linux>`), since the preprocessor decides which source exists. Angle-bracket names are discovered: the compiler indexes every Primal C file under the build unit's root by its declared namespace, so `<foo::bar>` works from any file in the unit, wherever `foo::bar` lives. The standard library resolves the same way from its own root, and `std` is reserved for it.
- A namespace import binds under its last path component (`io::printf`), or under an alias: `#import <std::io> as out` gives `out::printf`.
- `#import "../foo/bar"` imports by path, relative to the importing file. A directory imports that directory's namespace. A file (`"../util/strings"` or `"../util/strings.c"`) imports just that file's names, under its own namespace, or in the root namespace if it declares none. A path that names both a directory and a file is an error.
- C headers use the same two forms: `#import "sqlite3.h"` relative to the importing file, and `#import <zlib.h>` from the build unit's root, then its include paths, then system directories. Angle brackets holding a `/` or `.` mean a header; a bare `a::b` means a namespace (section 14.2). `#include` resolves paths the same way (section 10.1).
- A file import never creates a separate module. The file still compiles with the rest of its namespace; the importer just sees that one file's names.
- `using std::io::printf, std::io::puts;` brings individual names into file scope unqualified. There is no glob form (suggested default).
- Collisions are errors: two different declarations reachable under the same name, such as two root-namespace imports that both define `init`, or two imports binding the same namespace name. Reaching one declaration by two routes is not a collision, and `as` resolves namespace-name clashes.
- Imports are never re-exported: importing a file or module doesn't make its own imports visible.
- Imports and `using` declarations are per file and must precede all other declarations. An `#import` that follows a declaration is an error wherever the preprocessor placed it, so a conditional group of imports is fine at the top and not after the first declaration. `using` stays a keyword and a declaration, since it names declarations, not files.
- GCC's own `#import` (a deprecated include-once directive) keeps its meaning in C units' profiles, which follow their standard and GNU mode; only the `primal` profile gives `#import` the meaning here.
- Import cycles are an error (suggested default).

```c
#namespace app

#import <std::io>
#import <geo::mesh>		// discovered anywhere under the build unit's root
#import "../util/strings"	// one file by relative path: names are util::...
#import "config"		// no namespace: names land in the root namespace
#import "sqlite3.h"		// a C header, relative to this file

#if target.os == "linux"
#import <std::sys::linux>
#endif
```

**3.4 Name lookup**

1. Block scopes, innermost first.
2. File scope of the current module (all its files).
3. Names brought in by `using`, and import aliases.
4. The prelude (section 12).

Steps 2 and 3 are one level for collisions: a name declared in the module's file scope that a `using` or a root-namespace import also brings in, as a different declaration, is an error at that `using` or import (section 3.3).

A qualified name `a::b::x` resolves only through the file's imports, or the current namespace. A leading `::` names the root (legacy C) namespace: `::sqlite3_open`. Two `using` declarations of the same name are an error; a local shadowing an import alias is a warning. Macros are not namespaced (section 10).

**3.5 Symbol names**

- An exported `a::b::f` is emitted as the symbol `a.b.f` (suggested default). C identifiers can't contain `.`, so these never collide with C symbols, and debuggers show readable names.
- Root-namespace names are emitted exactly as in C. Every symbol takes the container's user-label prefix (`_` on Mach-O and i686 COFF, none on ELF and on x86-64 or AArch64 COFF), and `main` follows section 9.1.
- A root-namespace definition with a C library function's name, such as `malloc`, replaces the library's under the platform's interposition rules, as a C definition would, with a warning.
- `[[link_name("sym")]]` overrides the emitted name, for binding to existing C symbols or exporting a C-callable name.
- There is no overloading, so mangling never encodes types.

**3.6 Example**

```c
// geo/vec.c
#namespace geo

typedef struct { float x, y; } vec2;

vec2 add(vec2 a, vec2 b) {
	return (vec2){ a.x + b.x, a.y + b.y };
}
```

```c
// geo/len.c
#namespace geo

#import <std::math>

float len(vec2 v) {
	return math::sqrtf(dot(v, v));	// dot is defined below; no prototype needed
}

static float dot(vec2 a, vec2 b) {
	return a.x * b.x + a.y * b.y;
}
```

```c
// app/main.c
#namespace app

#import <geo>
#import <std::io>

int main(void) {
	geo::vec2 v = geo::add((geo::vec2){ 1, 2 }, (geo::vec2){ 3, 4 });
	io::printf("%f\n", geo::len(v));
}
```

## 4. Compilation model

The module, not the file, is the unit of compilation. The compiler reads a whole module directory, resolves file-scope names across its files, and writes one object plus one interface file.

&#91;embedded content: build model · modules, interfaces, legacy units\]

Importers read only the interface, so a changed function body rebuilds one module, while a changed interface also rebuilds its importers. Legacy C units take the classic preprocessor path into the root namespace. `buildconfig.toml` separates units from products: by default `ky build` links each `[[product]]` in-process with the built-in linker, and `object` and `static` products write standard objects and archives that any system linker accepts.

- The interface (`.ci`, suggested default) holds exported types, function signatures, compile-time `const` values, and the bodies of exported `inline` and `const` functions. It is versioned and keyed by its transitive inputs (sources and headers, profile and options, target and runtime ABI, and the compiler schema), and in whole-program builds it is only a cache.
- `#import <a::b>` finds every directory whose files declare `#namespace a::b`, using a namespace index built from the `#namespace` line of each Primal C file under the build unit's root; the stdlib has its own root.
- Files within a module may still be compiled and cached separately, as long as name resolution sees the whole module.

## 5. Types

The type system is C23's with every representation pinned down: 8-bit bytes, two's complement, IEEE 754 and the platform ABI's sizes. Integer promotions and the usual arithmetic conversions are unchanged; they are part of what makes C read like C.

**5.1 Arithmetic types**

| Type | Bits | Range | Replaces |
| --- | --- | --- | --- |
| `char` | 8 | 0 to 255 | `unsigned char`, `uint8_t`, `char8_t` |
| `signed char` | 8 | −128 to 127 | `int8_t` |
| `short` | 16 | −32,768 to 32,767 | `short int`, `int16_t` |
| `unsigned short` | 16 | 0 to 65,535 | `uint16_t`, `char16_t` |
| `int` | 32 | −2³¹ to 2³¹ − 1 | `signed`, `signed int`, `int32_t` |
| `unsigned int` | 32 | 0 to 2³² − 1 | `uint32_t`, `char32_t` |
| `long` | 64 | −2⁶³ to 2⁶³ − 1 | `long int`, `int64_t`, `intmax_t` |
| `unsigned long` | 64 | 0 to 2⁶⁴ − 1 | `uint64_t`, `uintmax_t` |
| `long long` | 128 | −2¹²⁷ to 2¹²⁷ − 1 | `__int128`, `int128_t` |
| `unsigned long long` | 128 | 0 to 2¹²⁸ − 1 | `unsigned __int128`, `uint128_t` |

Integer widths are the same on every target, so promotions, overflow and `sizeof` give the same answer everywhere. `char` is the unsigned byte and `signed char` the signed one. Each type has one spelling: `unsigned` alone still means `unsigned int`, and every spelling in the last column is an error with a fix-it that rewrites it.

Two prelude aliases follow the target's pointer width: `size_t` and `ptrdiff_t`, which are `unsigned long` and `long` on every 64-bit target. They also stand in for `uintptr_t`, `intptr_t` and `ssize_t`. `maxof(T)` and `minof(T)` give any integer type's limits as constants, replacing `INT_MAX`, `SIZE_MAX` and the rest. `long long` is 128 bits, always 16 bytes and 16-byte aligned, and crosses a C boundary as the target row's `c_int128` (section 14.4); wider integers come from `std::bigint`. In C, `long long` means 64 bits, so `--warn port` flags each use in code being ported (section 14.5).

| Type | Definition |
| --- | --- |
| `bool` | Intrinsic type, like `int`: no header, no `_Bool`. 1 byte holding 0 or 1. Converting any scalar gives `false` for zero or null and `true` otherwise; a load treats any nonzero byte as `true`. `true` and `false` are constants of type `bool`, and `bool` promotes to `int` in arithmetic, as in C23. |
| Bit-precise, complex, decimal, half and quad floats | Not language types. `std::bigint`, `std::complex`, `std::decimal` and `std::fp` provide them (section 13). |
| `float`, `double` | IEEE 754 binary32 and binary64. |
| `long double` | Per target row: x87 80-bit on x86-64, binary128 on AArch64 Linux. |

**5.2 Conversions**

- Integer to narrower signed integer keeps the low bits.
- Float to integer saturates at the target's limits; NaN converts to 0.
- Pointer to integer and back is the identity on the address. A pointer converted to a narrower integer type keeps the address's low bits, with a warning.
- Implicit conversion between incompatible pointer types is an error. `void *` still converts implicitly to and from any object pointer.
- Usual arithmetic conversions: operands narrower than `int` promote to `int`; then the narrower operand converts to the wider type, and at equal width the signed one converts to unsigned. That is C's rank rule, and with fixed widths it picks the same type on every target.

**5.3 Pointers**

- One flat address space; all object pointers have the same size and representation, and null is all-zero bits.
- `nullptr` and `nullptr_t` (C23); `NULL` is a predefined alias for `nullptr`. A `nullptr_t` object reads as `nullptr`, whatever its bytes.
- Explicit casts between function and object pointers are the identity on the address.
- Qualifiers: `const`, `volatile`, `restrict`, `atomic`.

**5.4 Arrays**

- Arrays decay to pointers, as in C.
- `countof(a)` gives the element count; applied to a non-array it is an error.
- A negative constant subscript of an array, as opposed to a pointer, is an error (C2y).
- `int a[static 10]` parameters: passing a visibly shorter array is an error. A null pointer passed to a `[static N]` parameter is an error too.
- Variably modified types (`int (*p)[n]`) are mandatory. Automatic VLA objects are allowed (suggested default), but a size of 0 or less traps and frames are stack-probed.
- Array types whose sizes differ only at run time are compatible: a conversion between them keeps the address, and accesses follow section 7.5. A composite type whose VLA size was never evaluated takes the other type's length (C2y).
- `int a[0]` is an error; use a flexible array member `[]`.

**5.5 Structures and unions**

- Anonymous members, flexible array members and bit-fields as in C11. Layout follows the ABI.
- Plain `int` bit-fields are signed; `bool` bit-fields are allowed.
- A struct with a flexible array member can't be an array element or a non-final member: error.
- A structure or union with no named members is laid out as GCC lays it out; an empty one has size 0.
- A structure or union used where its size or members are needed must be completed in some file of the module; otherwise it is an error.
- `[[packed]]` gives GCC-compatible packing.
- Reading a union member other than the last written reinterprets the bytes.
- Types are identified by namespace and name, so cross-file type compatibility is never in question.

**5.6 Enumerations**

- C23 fixed underlying types: `enum color : char { RED, GREEN, BLUE };`.
- Without one, the underlying type matches GCC for interop.
- Values outside the enumerator list are valid, as in C. There are no scoped enums (suggested default): namespaces already solve the collision problem.

**5.7 Inference and generic selection**

- `auto x = expr;`, `typeof`, `typeof_unqual` (C23).
- `auto` also infers through a pointer declarator: `auto *p = &x;` gives `p` the type of `&x`, which must be a pointer. Several declarators in one `auto` declaration must infer the same type. An `auto` declaration with no declarator, or with an array or function declarator, is an error.
- `generic` selection (ISO C's `_Generic`) accepts a type name as its controlling operand (C2y), as well as an expression.
- Compile-time `const` objects and `const` functions; see section 6.3.

## 6. Declarations and initialization

Declarations keep C's syntax. Two things change: every object starts initialized, and file-scope order stops mattering.

**6.1 Storage classes**

- `static`, `extern`, `thread_local`, `typedef`, and `auto` as type inference (C23). `const` stays a type qualifier, with the compile-time meaning in section 6.3.
- `register` is accepted and deprecated; taking the address of a `register` object is still an error.
- In module code, `extern` is only needed for foreign C symbols, optionally with `[[link_name]]`. A block-scope `extern` declaration that would give a name external linkage while a file-scope `static` declaration gives it internal linkage is an error (C2y).
- No tentative definitions in Primal C: `int x;` at file scope is a zero-initialized definition, and a second definition anywhere is an error. C units keep their standard's tentative definitions (section 14.1).

**6.2 Initialization**

- Automatic objects without an initializer are zero-initialized at block entry, so a `goto` or `switch` that jumps past the declaration still sees zero.
- `[[uninit]]` opts a variable out; reading it before a store is Erroneous (returns the bytes present).
- `= {}` zeroes the whole object, padding included (C23).
- Designated initializers as in C99, including nested and array designators. Range designators `[0 ... 9] = 1` are allowed, matching case ranges (suggested default).
- Duplicate designators for one member: the last wins, with a warning.
- An initializer has the form its object needs: one expression, optionally braced, or `{}` for a scalar; a braced list or one expression of its type for a structure or union; a braced list, or a string literal, optionally parenthesized, for a character array (C2y). Any other form is an error.
- Compound literals may carry `static`, `thread_local` or `register`: `(static int[]){ 1, 2, 3 }` (C23). Any other storage class on one is an error (C2y).
- Static and thread-local initializers must be constant expressions. They may refer to file-scope objects in other files of the module; a cycle is an error. An address converted to a pointer-width integer (`(long)&x`) is a constant there, emitted as a relocation; converted to a narrower integer it is not, and is an error.

**6.3 Const objects and const functions**

`const` objects and `const` functions replace most preprocessor use: unlike macros, they're typed, scoped, namespaced and visible to importers.

- A `const` object whose initializer is a constant expression is a compile-time constant, of any type: scalars, pointers to static objects or functions, arrays, structs and unions. It can size arrays, label cases, set bit-field widths and enumerator values, initialize statics, and appear in `static_assert`.
- A `const` object with a run-time initializer is an ordinary read-only object, exactly as in C.
- A function whose return type is `const`-qualified at the top level is a `const` function: `const int square(int x)`. For a pointer result the qualifier goes on the pointer, `char *const f(...)`; `const char *f(...)` returns a pointer to `const char` and stays an ordinary function. C ignores qualifiers on return types, so this spelling had no other meaning; in C units it keeps C's meaning, and `const` functions are Primal C only. A qualifier applied to a function type through a `typedef` (`const F f;`) is an error: it reads as neither.
- A call to a `const` function runs at compile time when it appears where a constant expression is required (array sizes, case labels, static initializers, `const` initializers, `static_assert`) and all its arguments are constant. Anywhere else it's an ordinary call, which the optimizer may still fold.
- A `const` function may use parameters and locals of any object type, every statement, arithmetic, other `const` functions, `const` objects, and pointers into its own locals. `static` or `thread_local` locals, non-`const` globals, `volatile`, `asm`, allocation, I/O and calls to non-`const` functions are compile errors that name the construct.
- Compile-time evaluation follows the run-time rules exactly: integers wrap and a shift count computed during evaluation is masked, each with a warning; a constant out-of-range count written in source is an error (section 7.3). A trap (division by zero, null dereference) or an out-of-bounds access during evaluation is a compile error, never undefined behavior. Floating-point folding uses round-to-nearest.
- Evaluation is bounded at 10 million steps and a recursion depth of 1,024 (suggested defaults, adjustable per build); exceeding either is a compile error.
- Exported `const` objects and the bodies of exported `const` functions go into module interfaces, so importers can use them in constant expressions too.
- `static_assert(expr)` takes an optional message.

```c
const int BUFFER_SIZE = 4096;

const int square(int x) {
	return x * x;
}

const int CELLS = square(64);	// evaluated at compile time
static char grid[CELLS];
static_assert(CELLS == 4096, "grid size");
```

**6.4 Alignment**

- `alignas` and `alignof` (C23 keywords). `alignof` also accepts an expression (suggested default, matching GCC).
- Over-aligned statics up to the page size; heap over-alignment through `aligned_alloc`.

**6.5 Attributes**

C23 `[[...]]` syntax throughout. Unknown attributes are a warning; `gnu::` attributes with an equivalent below are mapped to it.

| Attribute | Meaning |
| --- | --- |
| `deprecated`, `fallthrough`, `maybe_unused`, `nodiscard`, `noreturn` | As in C23. |
| `unsequenced`, `reproducible` | As in C23, but they grant the optimizer nothing: a call through such a type to a function that lacks the property is an ordinary call. |
| `assume(expr)` | The optimizer may assume `expr` is true. `expr` is not evaluated, and side effects in it are an error. A false assumption grants freedom as `unreachable()` does, so it is not Bounded; a checked build evaluates it once and traps if it is false. |
| `export`, `export("V2")`, `export("V1", default = false, as = "foo")` | Export from a product with `exports = "list"`; a version string exports it as that version, the default one unless `default = false`, and `as` names the versioned symbol, so `export("V1", default = false, as = "foo")` gives the non-default `foo@V1` (section 14.4). |
| `uninit` | Skip default zero-initialization. |
| `packed` | No padding; GCC-compatible layout. |
| `link_name("sym")` | Override the emitted symbol name. |
| `weak`, `section("name")` | As GCC's. |
| `always_inline`, `noinline`, `cold` | Optimization hints. |
| `wraps` | Marks intentional wraparound so the overflow lint stays quiet. |
| `fp_contract` | Allow FMA contraction in this function; replaces `#pragma STDC FP_CONTRACT`. |
| `fenv_access` | This function reads or changes the floating-point environment; replaces `#pragma STDC FENV_ACCESS`. |

## 7. Expressions

Operators and precedence are exactly C's, with `::` added at the top. What changes is that every expression has exactly one meaning: operands evaluate left to right, integers wrap, and pointers are addresses.

**7.1 Precedence** (highest first; all left-associative except levels 3, 14 and 15)

1. `::`
2. Postfix: `()` `[]` `.` `->` `++` `--`, compound literals
3. Prefix: `++` `--` `+` `-` `!` `~` `*` `&`, casts, `sizeof`, `countof`, `alignof`
4. `*` `/` `%`
5. `+` `-`
6. `<<` `>>`
7. `<` `<=` `>` `>=`
8. `==` `!=`
9. `&`
10. `^`
11. `|`
12. `&&`
13. `||`
14. `?:`
15. `=` and compound assignment
16. `,`

**7.2 Evaluation order**

- Operands evaluate strictly left to right, including function arguments. Each operand's side effects complete before the next operand starts.
- Assignment evaluates the left operand's address, then the right operand, then stores. `i = i++ + 1` and `a[i] = i++` are therefore defined (both still draw a warning).
- Compound assignment evaluates its left operand's address once.
- `&&`, `||`, `?:` and `,` keep their C short-circuit and sequencing rules.

**7.3 Integer arithmetic**

- Signed `+ - *`, `++`, `--` and unary `-` wrap (two's complement).
- `/` or `%` by zero traps. `INT_MIN / -1` wraps to `INT_MIN`; `INT_MIN % -1` is 0.
- A variable shift count is masked to `width - 1` bits (suggested default); a constant out-of-range count written in source is an error, while one computed during `const` evaluation is masked with a warning (section 6.3). C units follow their reference compiler's constant evaluation (section 14.1).
- Left shifts operate on the bit pattern, negative values included. Right shift of a negative value is arithmetic.
- Checked arithmetic: `ckd_add`, `ckd_sub`, `ckd_mul` (C23), available from `std::ckd`.
- Comparing a signed integer operand with an unsigned one by `<`, `<=`, `>`, `>=`, `==` or `!=` compares their mathematical values: `-1 < 0u` is true, so `i < n` works for a negative `int i` and a `size_t n` (suggested default). Pointer comparisons are unchanged, and arithmetic still converts as in section 5.2.

**7.4 Floating-point arithmetic**

- IEEE 754 results throughout, including division by zero.
- No contraction into FMA unless a function opts in with `[[fp_contract]]`; fast-math is never a default.
- Every operation rounds to its type (`FLT_EVAL_METHOD == 0`).
- Outside `[[fenv_access]]` functions the environment is taken to be the default. Code there that runs under another rounding mode or with traps enabled is Erroneous: each operation gives either its round-to-nearest result, where it was folded, or its result in the dynamic mode, and the flags it raises may or may not be seen.

**7.5 Pointer arithmetic and comparison**

- Pointer arithmetic is address arithmetic in units of the pointee size. Any result may be computed, stored and compared; only dereference can fault.
- Relational operators compare addresses across any two pointers. Subtracting pointers into different objects gives the address difference divided by the element size; the difference wraps in `ptrdiff_t`, as integers do.
- `NULL + 0`, `NULL - 0` and `NULL - NULL` give null, null and 0 (C2y).
- Dereferencing null traps. Dereferencing out-of-bounds, freed or dead memory is Bounded.
- A member access through a null pointer traps whatever the member's offset. `&p->m` is address arithmetic and touches nothing: through a null `p` it gives the member's offset as an address.

**7.6 Other operators**

- `sizeof` evaluates a VLA operand, as in C. `countof` and `alignof` are covered in sections 5 and 6.
- Casts as in C. Casting away `const` is allowed; writes through the result follow section 11.
- The value of a `void` expression can't be used or converted to anything but `void`, and the value of a non-array lvalue of incomplete type can't be used: both are errors (C2y).
- There are no statement expressions; use `static inline` functions (suggested default).

## 8. Statements

Statements are C's plus four additions from C2y and TS 25755: `if`/`switch` declarations, case ranges, named loops and `defer`. Each lowers to plain jumps with no runtime support.

**8.1 Blocks and labels**

- Labels may precede declarations and close a block (C23).
- Unreachable code after `return`, `break`, `continue` or `goto` is a warning.

**8.2 `if` and `switch` declarations** (C2y)

The declared name is in scope for the condition and every branch, `else` included.

```c
if (int fd = os::open(path, os::RDONLY); fd < 0) {
	return -1;
} else {
	use(fd);
}
```

**8.3 `switch`**

- Case ranges: `case 'a' ... 'z':` (C2y). A range with low > high, or overlapping cases, is an error.
- `case` values convert to the promoted type of the controlling expression. A conversion that changes the value is an error, and so are duplicates after conversion.
- Falling into a non-empty case without `[[fallthrough]]` is a warning; fallthrough stays legal.

**8.4 Loops and named loops** (C2y)

`break` and `continue` may name any enclosing loop or `switch` by its label:

```c
outer: for (int i = 0; i < h; i++) {
	for (int j = 0; j < w; j++) {
		if (grid[i][j] == target)
			break outer;
	}
}
```

The label syntax follows N3355 (suggested default); an alternative `for outer (...)` syntax has been proposed and may still change.

**8.5 `goto`**

- Kept. Jumping into the scope of a VLA is an error, as in C.
- Jumping forward past a `defer` into its block is an error.
- Jumping past an initialized declaration is allowed; the object holds its zero-initialized value.

**8.6 `defer`** (TS 25755)

- `defer statement;` runs the statement when control leaves the enclosing block: at its end, or by `return`, `break`, `continue` or `goto`.
- Deferred statements run in reverse order of declaration. A `return` expression is evaluated before they run.
- They don't run on `longjmp`, `exit` or a trap.
- A deferred statement may not itself `return`, `break`, `continue` or `goto` out of the defer.
- `defer` is a Primal C keyword. C units get TS 25755's `_Defer` only where their pinned reference compiler enables it, which the v1 target rows don't.

```c
int copy(const char *src, const char *dst) {
	io::FILE *in = io::fopen(src, "rb");
	if (!in)
		return -1;
	defer io::fclose(in);

	io::FILE *out = io::fopen(dst, "wb");
	if (!out)
		return -1;
	defer io::fclose(out);

	// ... copy ...
	return 0;
}
```

**8.7 `return`**

A non-`void` function must return a value on every path, or reach a `[[noreturn]]` call. Otherwise it's an error, except in `main`, where falling off the end returns 0 (section 9.1).

## 9. Functions

Functions stay as C defines them: prototyped, never overloaded, and called through the platform ABI. Type-generic interfaces use `generic`.

- Every function has a prototype; `f()` means `f(void)` (C23). Old-style K&R definitions are an error.
- Parameters may be unnamed in a definition (C23).
- No overloading, default arguments, references, nested functions or lambdas.
- Variadics use `...`, and `f(...)` with no named parameter is allowed (C23). `va_list`, `va_start(ap)`, `va_arg`, `va_copy` and `va_end` are prelude builtins:
	- `va_arg(ap, T)` reads the next argument slot the ABI gives `T`. A missing argument, or one of another type, is Erroneous: the slot's bytes are read as `T`. Primal C callers widen each integer variadic argument to a full slot, so a literal `0` reads as a null pointer.
	- A `T` that promotes (`char`, `short`, `float`) reads the promoted type and converts it back. An array, incomplete or function `T` is an error.
	- A `va_list` no `va_start` has set is zero (section 6.2), and reading through it traps. A list not ended is ended when its function returns, and `va_start` or `va_copy` on such a list restarts or replaces it.
	- After a `va_list` is passed to a function, the caller's list continues as the ABI leaves it: where the callee stopped on x86-64, where it was on AArch64. `va_copy` before the call gives the second result on both.
- `inline` is a hint. An `inline` function is emitted once, by its defining module; importers may inline its body from the module interface, but always refer to the defining module's static storage and never emit a copy. C's separate inline/external-definition rule is gone.
- `[[noreturn]]`: a reachable return is an error, and the compiler emits a trap at the function's end. A call to a `[[noreturn]]` function that returns anyway, which only a foreign C function can do, traps at the call. C units' `[[noreturn]]` and `_Noreturn` get a warning instead, and no trap.
- Calls through a function pointer of another type use that pointer's signature under the ABI. A call through a null function pointer traps; a call through a pointer that addresses no function is Bounded.
- Stack overflow from recursion or large frames traps (guard page plus stack probes).

`const` functions (section 6.3) are ordinary functions that can also run at compile time; they follow every rule here.

**9.1 `main`**

- `int main(void)` or `int main(int argc, char **argv)`.
- Exactly one non-`static` `main` per executable product, in any module, always emitted as the platform's C `main` symbol (`_main` on Mach-O) whichever namespace defines it (suggested default). More than one is an error, and so is none in an executable product that doesn't name its own `entry`; the built-in linker reports it, naming the product. Freestanding units and other products follow the ordinary rules.
- Falling off the end returns 0; returning `n` calls `exit(n)`.

**9.2 Inline assembly**

- `asm` is a Primal C keyword with named bindings, as in `assembler.md` §7: each operand is bound with `in`, `out` or `in out` to a register, `register`, `memory` or `constant`, and `changes` lists what else the block alters.
- There is no `goto` form in v1, and `const` functions may not contain `asm` (section 6.3).
- C units keep GNU extended `asm`, including `asm goto`.

## 10. Preprocessor

The Primal C preprocessor keeps C's token-level metaprogramming and conditional compilation, and drops most of the historical machinery around them. It decides which source exists and performs explicit token transformations; the compiler decides what that source means, and neither depends on the other's internals. This section defines the `primal` profile; C units keep their own standard's preprocessor (10.7).

**Invariants**

1. Isolation: preprocessing never needs the parser, types, symbols or the backend.
2. Determinism: identical explicit inputs produce identical output.
3. Immutability: source code can't alter build configuration.
4. Compatibility: traditional C preprocessing stays available through explicit profiles.
5. Transparency: macro expansions and conditional decisions can be inspected.
6. Restraint: metaprogramming stays token-oriented; there is no compile-time interpreter.
7. Stewardship: every feature and optimization must justify its maintenance cost.

**10.1 Directives**

| Directive | Purpose |
| --- | --- |
| `#if`, `#elif`, `#else`, `#endif` | Conditional compilation |
| `#ifdef`, `#ifndef` | Conveniences; `defined()` and configuration expressions are preferred |
| `#define`, `#undef` | Object-like, function-like and variadic macros |
| `#`, `##` | Stringification and token pasting, inside replacement lists |
| `#include` | Textual inclusion |
| `#namespace` | The file's namespace: first directive, once, never in a conditional group (section 3.1) |
| `#import` | Imports a namespace, a path or a C header; allowed in conditional groups (section 3.3) |
| `#embed` | Binary data as an initializer (C23) |
| `#error`, `#warning` | Diagnostics |
| `#line` | Diagnostic source locations |
| `#pragma once` | The only pragma |

- `#namespace` and `#import` exist in the `primal` profile only. Their operands are never macro-expanded. C profiles keep GCC's `#import` (a deprecated include-once `#include`) and have no `#namespace`.
- Not in the `primal` profile: `#elifdef`, `#elifndef`, `#include_next`, `#ident`, the `_Pragma` operator, and every pragma except `once`.
- Pragmas give way to attributes and build configuration: `pack` becomes `[[packed]]`, `STDC FP_CONTRACT` and `STDC FENV_ACCESS` become `[[fp_contract]]` and `[[fenv_access]]`, and warning control moves to compiler flags. Any other pragma is an error that names its replacement.
- `#include` pastes text, with the same path rules as `#import`: `"path"` relative to the including file, `<path>` from the build unit's root, then its include paths, then system directories. The included text is Primal C code, parsed in the including file's namespace, and its macros stay visible to the includer. C headers use `#import`, which keeps their own standard (section 14.2).
- Every translation-time undefined behavior in ISO C that has no sensible meaning (bad token pasting, `defined` produced by expansion, directive-like macro arguments, bad `#line` values) is an error. Two have an obvious one and are defined: a header or resource name starting with a digit is an ordinary path, and a macro named like a standard attribute is an ordinary macro (10.4).

**10.2 Build configuration**

The preprocessor receives immutable, typed values from the compiler and `buildconfig.toml`.

| Identifier | Type | Meaning |
| --- | --- | --- |
| `target.os` | string | Target operating system, such as `"linux"` |
| `target.arch` | string | Target architecture, such as `"x86_64"` |
| `target.abi` | string | The target triple's third part, or the platform default (`"sysv"` on x86-64 Linux) |
| `build.type` | string | `"debug"`, `"release"`, `"checked"`, or a custom type |
| `env.<name>` | bool, integer or string | User values from `[env]`; as `env::name` objects, booleans are `bool`, integers `long` and strings `const char` arrays |

- Only declared configuration is visible; host environment variables are never exposed.
- `target`, `build` and `env` are reserved in the preprocessor: defining, undefining or shadowing them with a macro is an error.
- Primal C code can also read `[env]` values as compile-time `const` objects: `env::max_connections`.

**10.3 Conditional expressions**

- Operands: boolean, integer and string literals; configuration identifiers; macros that expand to those; `defined(NAME)`; and `__has_import`, `__has_include`, `__has_embed` and `__has_c_attribute`.
- Operators: arithmetic, bitwise, comparisons, `&&`, `||`, `!` and parentheses. Integers evaluate in `long long`; strings support only `==` and `!=`; mixing types is an error.
- An undefined configuration identifier is an error. So is any other identifier that isn't a macro, instead of silently counting as 0 (suggested default; C profiles keep 0).
- Evaluation is local to the preprocessor and never asks the compiler to resolve a symbol.
- Inactive groups are dropped before parsing, but their directive nesting must still be valid.

```c
#if target.os == "linux"
	// Linux implementation
#elif target.os == "windows"
	// Windows implementation
#else
	#error "Unsupported platform"
#endif

#if build.type == "debug" && env.logging
	#define TRACE(...) log_trace(__VA_ARGS__)
#endif
```

**10.4 Macros**

- Object-like, function-like and variadic macros, `__VA_ARGS__`, `__VA_OPT__`, `#` and `##`, conditional definitions, and nested expansion.
- A macro never expands inside its own expansion, so expansion always terminates; unrestricted recursion isn't supported.
- Macros see only preprocessing tokens: never types, AST nodes, symbols or runtime values. There is no hygiene, since that would need name resolution. Attribute names are ordinary tokens too: a macro named `deprecated` expands inside `[[...]]` as anywhere else.
- A macro is visible from its definition to the end of its file, and in any file that includes it textually. Macros never cross a namespace or file import; a C header import is the one exception (section 14.2). A unit's `defines` and `--define` settings apply to every file in it.
- Ordinary constants and computation belong in the language, as `const` objects and `const` functions (section 6.3). Macros are for token-level work:

```c
#define STRINGIFY(x)			#x
#define CONCAT(a, b)			a##b
#define DECLARE_FIELD(type, name)	type name;
```

**10.5 Diagnostics**

- Every expanded token keeps its definition and invocation locations, and diagnostics print a readable expansion trace.
- A warning flags a parameter used more than once when its argument has side effects (`++`, `--`, an assignment or a call).
- Invalid pasting and malformed expansions are errors, and conditional compilation never loses source locations.
- `ky preprocess` shows the expanded output, and `--trace preprocessor` logs each conditional decision with its value and each macro expansion.

**10.6 Predefined macros**

- `__FILE__`, `__LINE__`, `__COUNTER__`, `__STDC__` (1), `__STDC_VERSION__` (`202311L`), and the suggested `__PRIMAL_C__` and `__NAMESPACE__`. `__func__` remains a predefined identifier.
- `__DATE__`, `__TIME__` and `__TIMESTAMP__` don't exist in the `primal` profile, since identical inputs must produce identical output. C profiles keep them.

**10.7 Compatibility profiles**

- Each C standard a unit can name (`knr`, `c89` to `c2y`, `gnu89` to `gnu2y`) is a profile with that standard's own inclusion, expansion, conditional and predefined-macro rules. `knr` uses the traditional pre-ANSI preprocessor.
- Profiles share the lexer, diagnostics and macro machinery. Profile-specific rules stay isolated, and every profile has its own regression suite.

**10.8 What replaces the standard library's macros**

| ISO C macro | Replacement |
| --- | --- |
| `NULL` | Predefined alias for `nullptr`; an imported C header never overrides it. |
| `INT_MAX`, `DBL_EPSILON` and other limits | `const` objects in `std::limits`. |
| `EOF`, `SEEK_SET`, `BUFSIZ` | `const` objects in `std::io`. |
| `assert` | The `assert(expr)` keyword, Primal C only, whose operand must be scalar, as an `if` condition must; removed with `--assertions off`, which defines `NDEBUG` in C units. |
| `offsetof`, `va_*`, `unreachable` | Prelude builtins. |
| `errno` | A `thread_local int` in `std::err`. |
| `setjmp` | A compiler-recognized builtin in `std::jmp`. |
| `bool`, `true`, `false`, `alignas`, `static_assert` | Keywords. `bool` is an intrinsic type (section 5.1). |
| `countof` | Keyword. |
| `<tgmath.h>`, `ckd_*`, `stdc_*` bit utilities | Type-generic builtins in their modules. |
| `PRId64` and other format macros | C23 `%w64d`-style length modifiers. |

## 11. Memory and concurrency model

Memory is bytes at addresses. The only undefined behavior left is touching memory the program doesn't own, and a data race yields a stale value, never a license to miscompile.

**11.1 Objects and access**

- Every object occupies contiguous bytes. Any object may be read or written through any lvalue type; there is no strict aliasing.
- Provenance follows PNVI-ae-udi (ISO TS 6010): integer-to-pointer round trips work for any object whose address has been exposed.
- Struct assignment copies every byte, padding included. An assignment between overlapping objects stores the value read, as if copied through a temporary.
- Pointers may be misaligned; misaligned access works, except for atomics, where it traps. In C units a misaligned atomic gets no promise: it does what the hardware or library does.
- `restrict` is kept as an explicit opt-in and follows C11 6.7.3.1's formal definition: association, based-on, and objects modified during the block's execution. The optimizer uses only facts derived from those rules, never a blanket no-alias property. Violating it is Bounded.

**11.2 Lifetime and initialization**

- Automatic objects live from block entry to block exit; allocated objects from allocation to `free`.
- Accessing an object outside its lifetime is Bounded. A pointer to a dead object keeps its value: copying and comparing it is fine.
- An object with temporary lifetime, such as the array member of a structure returned by value, is an ordinary writable object until the end of its full expression.
- Automatic objects are zero-initialized (section 6). `[[uninit]]` objects and heap memory return the bytes present (Erroneous).
- `const` objects with static storage live in read-only memory, so writes trap in Primal C; C units get no trap. Writes to an automatic `const` object through a cast happen, but reads may see the constant-folded value (Erroneous).

**11.3 `volatile`**

Each access is exactly one load or store of the object's width when aligned. Volatile accesses are never merged, split or elided, and keep their order relative to each other.

**11.4 Atomics and threads**

- The C11 memory model is unchanged: `atomic`, memory orders and fences, with operations in `std::atomics`.
- An atomic operation with an order its kind forbids (an acquire store, a release load), or a compare-exchange whose failure order is stronger than its success order, uses `memory_order_seq_cst`, with a warning when the order is a constant.
- `.` and `->` on an atomic structure or union are an error (C2y): copy it to a non-atomic object first.
- Atomics of 1, 2, 4 and 8 bytes are lock-free on both v1 target rows (x86-64 and AArch64 Linux). 16-byte atomics, so every `atomic long long` operation, are calls to the libatomic-compatible `__atomic_*` interface there (`__atomic_fetch_add_16`). `atomic_is_lock_free` and `ATOMIC_*_LOCK_FREE` report per type and object, and `--warn port` flags an `atomic long long` used in a signal handler.
- A data race on a non-atomic object is Erroneous. The racing load returns some stored value, possibly torn above 8 bytes or when misaligned; the compiler never invents stores or re-loads a value it already read.
- Threads and mutexes are the C11 API in `std::thread`. Misuse (unlocking an unowned mutex, joining twice) traps.

**11.5 Signals and non-local jumps**

- Handlers may access `volatile sig_atomic_t` and lock-free atomics; any other access falls under the race rule. Each stdlib function documents whether it is async-signal-safe.
- A handler may call any `std::` function. Each `std::` subsystem with shared state, such as the allocator or a stream, marks the thread while it runs; a call from a handler that interrupted the same subsystem on the same thread traps instead of deadlocking or corrupting it.
- `signal` sets a per-process disposition, in a program with threads too. Delivery nests as on POSIX: a handler may `raise`, including a handler for `abort` or `raise`, and may itself be interrupted.
- A handler for a fault (`SIGFPE`, `SIGILL`, `SIGSEGV`, `SIGBUS`) that returns resumes at the faulting instruction, which runs again.
- `setjmp` has true returns-twice semantics: a function that calls it or any other returns-twice function keeps its locals in stack slots, in C units too, and after `longjmp` they hold their last stored values. `setjmp` may appear in any expression.
- `longjmp` to a frame that has returned is Bounded; checked builds of Primal C units trap via a cookie in the `jmp_buf`.
- `longjmp` from a signal handler, nested or not, or from a termination handler, is an ordinary `longjmp`. Out of a termination handler to a live frame, it abandons termination until the next `exit`.
- `longjmp` back into a scope that declared a VLA, after control left that scope, restores the stack pointer `setjmp` saved, so the VLA's storage is the frame's again; its elements hold whatever bytes are there (Erroneous).

## 12. Core prelude

A small prelude is visible in every module without `#import`. It holds what C programs used to pull from `<stddef.h>`, `<stdarg.h>` and `<stdcountof.h>`, and nothing that allocates or does I/O. The `<stdint.h>` and `<uchar.h>` types are gone: the core integer types replace them (section 5.1).

| Group | Names |
| --- | --- |
| Size and pointer types | `size_t`, `ptrdiff_t`, `nullptr_t`, `max_align_t` |
| Variadics | `va_list`, `va_start`, `va_arg`, `va_copy`, `va_end` |
| Builtins | `offsetof(type, member)`, `unreachable()` |
| Constants | `NULL` |

Prelude names are ordinary identifiers: a module may shadow one, with a warning. An imported C header never overrides the prelude's `NULL`, which is `nullptr` (section 14.2). `<stdint.h>` names such as `int32_t` aren't declared in Primal C code; using one gives an error with a fix-it to its core type.

`offsetof(type, member)` is address arithmetic: an index past an array member's bounds, as in `offsetof(struct s, data[n])`, gives the member's offset plus the index times the element size. A bit-field, or a designator that names no member, is an error.

## 13. Standard library

Each ISO C header becomes one module under `std`, and every function keeps its name (suggested default), so porting is mechanical: `#include <stdio.h>` becomes `#import <std::io>` and `printf` becomes `io::printf`. This section, its removals and its library-wide rules apply to `std::` only; C units use the C library their product's `libc` selects (section 14.1).

| Header | Module | Notes |
| --- | --- | --- |
| `<stdio.h>` | `std::io` | `FILE` keeps its name. No `gets`, and no wide conversions (`%lc`, `%ls`, `%l[`). Format strings are checked at compile time when literal. `std::` formatting and scanning are distinct symbols (`std.io.printf`), where `%lld` is the 128-bit `long long`; the root C `printf` keeps its exact C signature and the C unit's data model. |
| `<stdlib.h>` | `std::lib` | The allocator traps on a pointer it didn't return, or already freed, passed to `free` or `realloc`; `free_sized` and `free_aligned_sized` free as `free` does, whatever size they're given. A zero size to `malloc`, `calloc`, `aligned_alloc` or `realloc` gives a unique non-null pointer, and `realloc(p, 0)` frees `p`'s bytes. `atoi`, `atol` and `atoll` saturate, and `atof` returns what `strtod` returns. `abs(INT_MIN)` wraps to `INT_MIN`. |
| `<string.h>` | `std::str` | `memcpy` behaves as `memmove`. Null with length 0 is a no-op. `strtok`'s position is per thread. `strerror` returns a static read-only message, valid for the whole run. |
| `<math.h>`, `<tgmath.h>` | `std::math` | Type-generic forms are builtins; the classification and comparison builtins convert an integer operand to `double`. Adds `uabs` family (C2y, N3349) and type-generic `min`/`max`. |
| `<ctype.h>` | `std::ctype` | Accepts -128 to 255. ASCII only. |
| `<errno.h>` | `std::err` | `errno` is a `thread_local int`; codes are `const` objects. |
| `<limits.h>`, `<float.h>` | `std::limits` | `maxof(T)` and `minof(T)` replace the integer limits; the `<float.h>` values are compile-time `const` objects. |
| `<inttypes.h>` | None | Not needed in `std::`: `%d`, `%ld` and `%lld` always match `int`, `long` and `long long`, so the `PRI` and `SCN` macros go. `imaxabs` and `strtoimax` become `labs` and `strtol`. |
| `<time.h>` | `std::time` | `asctime` and `ctime` dropped; use `strftime`. `gmtime` and `localtime` results are per thread. `TIME_THREAD_ACTIVE` always measures the calling thread. |
| `<threads.h>` | `std::thread` | Error-checking mutexes. |
| `<stdatomic.h>` | `std::atomics` | C11 operations as builtins. Plural because `atomic` is a keyword. |
| `<stdbit.h>` | `std::bit` | Type-generic `stdc_*` as builtins, plus C2y bit utilities. A signed operand is read as its bit pattern; a `bool` or enumeration operand is an error. |
| `<stdckdint.h>` | `std::ckd` | `ckd_add`, `ckd_sub`, `ckd_mul`. A `bool` or enumeration operand or result is an error. |
| `<setjmp.h>` | `std::jmp` | Compiler-recognized; see section 11. |
| `<signal.h>` | `std::signal` | Signal-safety documented per function. |
| `<fenv.h>` | `std::fenv` | `fenv_t` and `fexcept_t` record that a call filled them: `fesetenv` and `feupdateenv` trap on an environment no call produced, and `fesetexceptflag` clears a flag its object didn't save. |
| `<uchar.h>` | `std::uchar` | UTF-8, 16 and 32 conversions. C2y's `<stdmchar.h>` joins it once C2y is published. |
| `<wchar.h>`, `<wctype.h>` | `std::wchar` | Encoding conversions only: the restartable conversions, `btowc`, `wctob` and `mbsinit`. An `mbstate_t` holds a partial UTF-8 character's pending bytes. Wide I/O, the wide string utilities and wide classification are dropped (suggested default). |
| `<complex.h>` | `std::complex` | `cfloat`, `cdouble`, `cldouble` structs with arithmetic functions. `cfloat` and `cdouble` carry the identity of C's `_Complex float` and `_Complex double` in signatures, aggregates, calls, returns and varargs, and are passed exactly as those types under the target's ABI. |
| `_BitInt` (C23 keyword) | `std::bigint` | Fixed-size `i256`, `u256`, `i512` and `u512` structs, plus limb-array functions for any width. 128-bit integers are the core `long long` types. |
| `_Decimal32/64/128` (C23 keywords) | `std::decimal` | `d32`, `d64`, `d128` in IEEE 754 BID encoding, with arithmetic, conversion and formatting functions. |
| `_Float16`, `_Float128` (C23 Annex H) | `std::fp` | `f16` and `f128` storage types with conversion and arithmetic functions. |
| `<stddef.h>`, `<stdint.h>`, `<stdarg.h>`, `<stdcountof.h>`, `<assert.h>` | prelude | See section 12; `assert` is a keyword. `<stdint.h>` names become the core integer types (section 5.1). |
| `<stdbool.h>`, `<stdalign.h>`, `<stdnoreturn.h>`, `<iso646.h>` | removed | Not needed. C units get them from the compiler-provided headers (section 14.1). |
| `<locale.h>` | removed | "C" locale and UTF-8 only (suggested default). |
| Annex K (`_s` functions) | removed |  |

**Library-wide rules**

- Every function either has defined behavior for every input or traps deterministically, except that accesses through pointers to memory the program doesn't own are Bounded. Preconditions checkable in O(1) are always checked.
- A function that reads some arguments and writes others acts as if it read every input before writing any output, so overlapping arguments act as if copied through a temporary: `strcpy`, `sprintf` into its own argument, `strftime`, `mbstowcs`.
- A literal format (the `printf`, `scanf`, `strftime` and `strfromd` families) is checked against the call at compile time, as a prototype is. A mismatch is an error, and so is a shortest possible output, with its null, longer than a destination of known size. A `%s` or `%[` with no width is allowed; with a known destination it draws a warning.
- With a run-time format, arguments are read from their ABI slots as the format says (Erroneous where they don't match). A flag, precision or length modifier a conversion doesn't use is ignored, and a conversion specification the function doesn't know is copied to the output unchanged and consumes no argument; in `scanf` it is a matching failure. Bytes that aren't valid UTF-8 are ordinary characters. Output over `INT_MAX` characters fails with `EOVERFLOW`, and a `scanf` count over `INT_MAX` items saturates. A `scanf` result its object can't hold saturates and sets `ERANGE`.
- `qsort` never reads out of bounds, even with an inconsistent or mutating comparator, and leaves the array a permutation of its elements; `bsearch` on unsorted input returns some match or null.
- `fflush` on an input stream discards buffered input; switching between reading and writing needs no explicit seek.
- Text and binary streams are the same, and positions are byte offsets: `fseek` and `fsetpos` accept any offset, failing with `EINVAL` where no position exists. After `ungetc` at position 0, `ftell` fails with `EINVAL` until the character is read again. After a failed transfer the position is just past the last byte moved. `fopen` with an invalid mode fails with `EINVAL`, and a literal invalid mode draws a warning. A `FILE` after `fclose` is freed memory (section 11.2).
- Library-owned strings: `getenv` returns the environment's own writable string, so a write within it changes what later calls return; `strerror`'s messages are read-only, so a write traps. A result that doesn't fit (`strxfrm`, `strftime`) leaves the first `n − 1` bytes and a null.
- Termination: a second `exit` or `quick_exit` never returns. From a termination handler, termination continues, each handler still running once, and ends with the second call's status; from another thread, the call waits for the program to end. A signal during `quick_exit` arrives as at any other time. `system` runs its command and returns its status, whatever the command does.
- The full per-function list is in the companion UB catalogue ([`ub.md`](ub.md) §5 and §6).

## 14. C interop and legacy code

Existing C compiles unchanged as legacy units, and Primal C code reaches it by importing its header. SQLite and Lua keep building while new code moves to modules.

**14.1 Legacy units**

- A legacy unit (a C unit) is a file built under a C standard (`knr`, `c89` to `c2y`, `gnu89` to `gnu2y`) in `buildconfig.toml`. It uses that standard's syntax and keywords, `#include`, and the classic preprocessor pipeline (the traditional pre-ANSI one for `knr`). Every file belongs to exactly one build unit, and its standard comes from that unit, never from its contents.
- Its names live in the root namespace and are emitted unmangled.
- C units use the headers of the C library their product's `libc` selects (`system`, the default for hosted builds; `kyogre`; or `none`, the default for freestanding builds and `image` products), after the compiler-provided headers such as `<stddef.h>` and `<stdbool.h>`. With `libc = "system"` the sysroot's headers are used unmodified, and `std::` is a layer over that library's allocator, `FILE` and `errno`.
- Each unit is compiled under one profile, taken from its build unit.

  **C units** follow their standard and GNU mode exactly, as ISO C and the unit's pinned reference compiler define them. Every implementation-defined and ABI-visible choice comes from the target row and reference profile (`frontend-plan.md` §3.2). The compiler neither defines nor exploits their undefined behavior: where it detects undefined behavior it warns, inserts no check or trap, promises no result, and derives no optimization from it. Constant evaluation that the language requires follows the reference compiler.

  **Primal C units** follow this specification.

  Calls between the two use the platform C ABI.

**14.2 Importing a C header from module code**

```c
#namespace app

#import "sqlite3.h"

int open_db(const char *path, sqlite3 **db) {
	return ::sqlite3_open(path, db) == SQLITE_OK ? 0 : -1;
}
```

- `#import "file.h"` runs the preprocessor over the header and adds its declarations to the root namespace. They can be named unqualified or with a leading `::`. The header is preprocessed and parsed entirely as C, exactly as the C unit that owns it sees it (its standard, defines, include paths, target and pragmas), so Primal C code sees the same declarations the implementation was compiled against. Ownership is inferred from paths with no extra configuration, and an ambiguous owner must be named in `buildconfig.toml`; headers no unit owns, like system headers, use `gnu23` with the product's `libc` headers.
- The header's macros (`SQLITE_OK`) are visible in the importing file only. This is the single place macros cross files (suggested default). They cross as preprocessing-token sequences that keep their C spelling and provenance, so `0400` stays octal and `LL` stays C `long long`, and their types map through section 14.4. Macros that collide with Primal C keywords or prelude names (`NULL`, `bool`, `true`, `false`) are not imported, and a macro whose expansion isn't valid Primal C at the use site is an error there, with a hint to write a C wrapper.
- The header's `inline` functions (`static inline`, GNU `extern inline`, C99 `inline`) compile as C in that configuration, once per importing Primal C file, so each importer gets its own copy of their static objects, as C gives each translation unit.

**14.3 Calling module code from C**

- Exported names are symbols like `geo.add`, which C can't spell. Give a function a C name with `[[link_name("geo_add")]]`.
- `ky header` generates a C header for a module, for a named consumer profile. Its declarations use fully prefixed asm labels (`__asm__("geo.add")`) where the consumer supports them, so GCC and Clang code can call the module directly; root C-linkage functions for MSVC consumers; and otherwise a named C wrapper, or a precise "not representable" diagnostic. `[[link_name]]` names replace the module names in the header.
- It maps Primal C types back to C: `char` is `unsigned char`, `signed char` stays, `short` and `int` are the row's 16- and 32-bit types, `long` is `long long` (or `int64_t`), `long long` is the row's `c_int128` or not representable, `bool` is `_Bool` (C99 and later), and `struct a::b::S` is `struct a_b_S` or its `[[link_name]]`. `std::complex` types are `_Complex` where the consumer supports it, and a wrapper otherwise.

**14.4 ABI and linking**

- Struct layout, calling convention, varargs and `bool` follow the platform ABI (SysV x86-64 first).
- By default `ky build` links each `[[product]]` in-process with the built-in linker. `object` and `static` products write standard ELF, Mach-O or COFF objects and archives that any system linker accepts, and static and shared libraries work as usual.
- Dynamic export is per product: `exports = "default"` exports every non-`static` symbol not marked hidden (C units' `__attribute__((visibility))` and a unit's `visibility = "hidden"` apply), and `exports = "list"` exports only `[[export]]` symbols and names in the product's `export_file`. A single `[[export]]` never switches the mode. Versions come from `[[export("V2")]]` and C units' `symver`, with at most one default per name; containers without ELF-style versioning reject them.
- Types in an imported C header map to Primal C types by the target row's typedefs, by width and signedness, never by OS name. C `long long` is `long` and `__int128` is `long long`; a 64-bit C `long` is `long` and a 32-bit one is `int`; `wchar_t` is the row's type, such as `int` on x86-64 Linux, `unsigned int` on AArch64 Linux and `unsigned short` on Windows x64; each `<stdint.h>` name is its core type. The mapping is for spelling only: at the boundary each declaration keeps its C type identity, so `char`, `signed char` and `unsigned char` stay distinct, and so do `long` and `long long` at equal width.
- `long long` crosses a C boundary as the row's `c_int128`: `__int128` in register pairs on LP64 ELF, Apple arm64, x32 and arm64_32, and Clang's `__int128` on Windows x64 (by pointer, returned in `xmm0`) and Windows ARM64 (register pairs). On a row without one, such as i686, Primal C passes it as a struct of two 64-bit integers, and crossing a C boundary is an error with no type-changing fix-it; the diagnostic suggests an explicit wrapper (`lo` and `hi`, unsigned 64-bit) with conversion helpers.
- C's plain `char` maps to `char`. By-value `char` arguments and results keep their bits, and the backend extends them as the C ABI expects (signed on x86-64, unsigned on AArch64).
- C types with no Primal C spelling, such as `_BitInt(N)`, make a declaration unusable from Primal C, with an error at the use.
- A Primal C declaration of a C entity is checked against the imported header, and a mismatch is an error. A hand-written `extern` with no header to check is trusted: calls and accesses use its type under the platform ABI, as calls through a function pointer of another type do (section 9), and they work at the object's real alignment, whatever alignment the declaration states. An access past the foreign object's real size is Bounded.

**14.5 Porting a C file to a module**

1. Add `#namespace a::b` as the first directive, or leave it out to stay in the root namespace. Every Primal C file in the same directory must make the same choice.
2. Replace each `#include <x.h>` with `#import <std::x>` (section 13), and project headers with `#import` of the module that defines them.
3. Qualify stdlib calls (`io::printf`) or add `using` declarations.
4. Turn exported object-like macros into `const` objects and function-like ones into `const` or `inline` functions.
5. Delete forward declarations and prototypes that only existed for header ordering.
6. Rewrite `017`-style octal literals as `0o17`.
7. Build with `--warn port` and fix what it reports: `long long` and `%lld` widths, plain `char` signedness, character constant types and signed/unsigned comparisons.

## 15. Removed and changed from ISO C

Only syntax and library features are listed here; every behavior change is in sections 7 and 11 and in the companion UB catalogue ([`ub.md`](ub.md)). The table is Primal C's; C units keep their standard (section 14.1).

| Feature | Status | Use instead |
| --- | --- | --- |
| `#include` for sharing declarations | Replaced; textual inclusion stays | `#import` |
| Header files and forward declarations for ordering | Unneeded | Order-independent file scope |
| K&R definitions, unprototyped declarations | Removed (as in C23) | Prototypes |
| Implicit `int`, implicit function declarations | Removed (as in C99) |  |
| Trigraphs | Removed (as in C23) |  |
| Digraphs | Removed (suggested) |  |
| Leading-zero octal (`017`) | Error | `0o17` |
| Multi-character constants | Error in modules | Integer constants |
| Tentative definitions, COMMON symbols | Removed | One definition |
| `register` | Deprecated | Nothing |
| `_Bool`, `<stdbool.h>`, `_Noreturn`, `_Imaginary` | Removed | `bool` (intrinsic), `[[noreturn]]` |
| Every `_Keyword` spelling (`_Alignas`, `_Atomic`, `_Generic`, `_Static_assert`, `_Thread_local`, `_Countof`) | Removed | `alignas`, `atomic`, `generic`, `static_assert`, `thread_local`, `countof` |
| `_BitInt`, `_Complex`, `_Decimal32/64/128`, `_Float16`, `_Float128` | Moved to the library | `std::bigint`, `std::complex`, `std::decimal`, `std::fp` |
| Exported macros | Removed | `const` objects, `const` and `inline` functions |
| `gets`, `asctime`, `ctime` | Removed | `fgets`, `strftime` |
| Annex K `_s` functions | Removed | Length-carrying APIs |
| `<locale.h>`, wide-character I/O, `<iso646.h>` | Removed (suggested) | UTF-8 |
| `constexpr` | Removed | `const` objects and functions |
| `long int`, `short int`, `signed int`, `unsigned char` and other alternate spellings | Error with a fix-it | One spelling per type (section 5.1) |
| `<stdint.h>` and `<uchar.h>` types, `intmax_t`, `intptr_t`, `wchar_t` | Removed | Core integer types, `size_t`, `ptrdiff_t` |
| `u8` and `L` prefixes | Error with a fix-it | Plain literals, `u`/`U` |
| Integer limit macros (`INT_MAX`, `SIZE_MAX`, …) | Replaced | `maxof(T)`, `minof(T)` |

## 16. Open questions and suggested defaults

Each default below is what the spec assumes today. The alternative column lists the strongest competing choice.

| Question | Suggested default | Alternative |
| --- | --- | --- |
| Language name and file extension | "Primal C"; keep `.c` | A new extension for module files |
| Namespace declaration form | The directive `#namespace a::b`, with `#import` beside `#include` | A file-level `namespace a::b;` keyword declaration and `import` keyword; C++-style `namespace a { }` blocks |
| Symbol names | `a.b.f`, with the container's user-label prefix | `a__b__f`, or Itanium-style mangling |
| Glob imports | None | `using a::b::*;` |
| Import cycles | Error | Allowed via a declarations-first pass |
| Macro export | Never; type-generic stdlib entries are builtins | `#export` plus qualified macro names |
| Macros from imported C headers | Visible in the importing file as C token sequences; macros colliding with keywords or prelude names are not imported | `const` objects and functions only |
| Oversized variable shifts | Mask the count | Go-style 0 or sign fill |
| Signed overflow | Wrap; trapping is an opt-in lint | Trap by default |
| Automatic VLAs | Allowed, checked and stack-probed | Removed; variably modified types only |
| Digraphs | Removed | Kept |
| Locale and wide I/O | Removed; UTF-8 and the "C" locale | Kept |
| `defer` | Primal C keyword, per TS 25755; C units get `_Defer` only where the reference compiler enables it | Wait for C2y |
| Named loop syntax | Labels (N3355) | `for name (...)` |
| Entry point | One non-`static` `main` per executable product, in any module, emitted as the platform's C `main` | Root namespace only |
| Interface file | Compiler-private binary `.ci`, versioned and keyed by its inputs; only a cache in whole-program builds | Textual, human-readable interface |
| Version macros | `__STDC_VERSION__` = `202311L` plus `__PRIMAL_C__` | Report the C2y value once published |
| Bit-precise integers | Library structs (`i256` to `u512`) and limb-array functions | A compiler-builtin `bitint(N)` with operators |
| Atomics module name | `std::atomics` | `std::sync` |
| Mixing standards | One standard per build unit, set in `buildconfig.toml`; a file in two units is an error | Per-file overrides or detection from contents |
| Unknown identifiers in Primal C `#if` | Error | 0, as in C |
| `#embed` in the primal profile | Kept | Removed, with binary data loaded through the build |
| Compile-time evaluation limits | 10 million steps, recursion depth 1,024 | No limit beyond memory |
| Plain `char` signedness | Unsigned on every target | Follow the ABI (signed on x86-64) |
| Signed/unsigned comparisons | Compare mathematical values (integer operands only, never pointers) | C's conversion to unsigned, plus a warning |
| Alternate integer spellings | Errors with fix-its | Accepted, with a warning |
| Null pointer constants | A literal `0` (an integer constant expression with value 0) still converts to a pointer, as in C23 | Only `nullptr` and `NULL` |

## 17. Sources

- [GCC C status](https://gcc.gnu.org/projects/c-status.html): the C2y features adopted here and their proposals (N3260, N3298, N3349, N3353, N3355, N3356, N3369, N3370, N3322).
- [N3322: Allow zero length operations on null pointers](https://open-std.org/JTC1/sc22/wg14/www/docs/n3322.pdf).
- [ISO/DIS TS 25755: defer](https://www.open-std.org/JTC1/SC22/WG14/www/docs/n3928.pdf).
- C23 (ISO/IEC 9899:2024), C11 and C99 from general knowledge. C2y is still a working draft, so details may move before publication.
