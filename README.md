# Primal C skill

An agent skill for writing, reading, reviewing and porting **Primal C**, the C dialect of the Kyogre toolchain
(`ky`). Primal C is C23 with C2y's additions, modules with namespaces
instead of headers, and every behavior C leaves undefined given a defined result, a compile-time error or a
deterministic trap.

| File | What it is |
| --- | --- |
| [`SKILL.md`](SKILL.md) | The skill: when to use it, the rules that matter most, checklists for writing and reviewing |
| [`GUIDE.md`](GUIDE.md) | A language guide: modules, literals, types, `const`, expression semantics, statements, the preprocessor, the standard library, C interop, porting, style and pitfalls |
| [`reference/lang.md`](reference/lang.md) | The language specification, revision 1 (authoritative) |
| [`reference/stdlib.md`](reference/stdlib.md) | The standard library's specification (draft) |

## Install

The skill is a directory holding `SKILL.md`. Put it where your agent looks for skills.

**OpenCode** (all projects):

```sh
git clone https://github.com/splashsky/primal-c-skill ~/.config/opencode/skills/primal-c
```

**Claude Code** (all projects):

```sh
git clone https://github.com/splashsky/primal-c-skill ~/.claude/skills/primal-c
```

**One project only**: clone it into the project's `.opencode/skills/primal-c` or `.claude/skills/primal-c` instead.

Update with `git pull` in that directory.

## Status

Primal C is at **revision 1**. Its compiler, Kyogre, is in development: the language specification is settled
enough to write code against, and choices it marks as *suggested defaults* may still change in a later revision.
The standard library's specification is a draft, so library names outside ISO C may change. The reference
documents are copies of Kyogre's specification; their relative links point into that repository.

## License

MIT. See [`LICENSE`](LICENSE).
