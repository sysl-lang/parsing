# parsing

The parts of a hand-written parser that every grammar rewrites, for
[sysl](https://github.com/sysl-lang/sysl).

A program that has to read a config format, a wire text format or a source language starts at the
byte cursor and builds the same things every time: a scanner, a span, literal reading with its escape
rules, and a way to say where the mistake was. This package is those, plus the two that are fiddly
enough to be worth writing once — the layout pass an indentation-sensitive grammar needs, and the
binding-power loop an expression grammar needs.

**It is not a parser generator and not a combinator library.** A grammar stays hand-written recursive
descent — which is what Clang, GCC's C++ front end, rustc, Go, TypeScript, V8, Roslyn, Swift, Zig and
javac all are, several of them having started on a generator and moved off. Recognising valid input
is not the hard part; what happens on *invalid* input is, and that is exactly what a table-driven
parser cannot tell you about. What this removes is the four hundred lines underneath the grammar that
are the same in all of them.

```
sh/sysl/parsing/
    source.sysl         spans, positions, and the input with its line-start table
    scan.sysl           the byte and character cursor
    classes.sysl        the byte tests, as predicates and as 256-bit sets
    literal.sysl        numbers, quoted text, escapes
    layout.sysl         indentation as structure, brackets included
    tokens.sysl         a value with its span, and a cursor over a lexed token list
    pratt.sysl          expressions, by binding power
    diag.sysl           diagnostics, carets, and a report that truncates
    tests.sysl          what all of it claims, run by `sysl test .`
    tests_json.sysl     a whole JSON reader, as the proof a consumer can be written
package.hocon           who this package is, and what it needs of the machine
```

The module is **`sh.sysl.parsing`**, and the three directories are that name: a dotted module name
mirrors its path from the library root. The prefix is the reverse-DNS of `sysl.sh`, so that a package
claims a name nobody else will mint rather than the top-level word `parsing`.

## Using it

Name it in your project's `package.hocon` and `sysl build` fetches it:

```hocon
dependencies {
  parsing { git = "github.com/sysl-lang/parsing", version = "0.6.0" }
}
```

The coordinate is an identity rather than a URL, so it carries no `https://`, and `version` is the
tag `v0.3.0` here. It needs sysl 0.0.76 or newer, for the reason `package.hocon` gives.

Or build it into an artifact and compile against that, which needs no fetching:

```
sysl build-lib . -o /tmp/parsing.syslib
sysl run yourprogram.sysl --lib /tmp/parsing.syslib
```

## A first lexer

```sysl
import sh.sysl.parsing.*

main()
    var s = scanner_of("let x = 42  // the answer")

    s.skip_blanks()

    val name = s.read_ident(is_ident_start, is_ident_cont)

    print(name)                     // Some(0..<3)

    s.skip_blanks()
    print(s.read_ident(is_ident_start, is_ident_cont))
    s.skip_blanks()
    print(s.accept(u8('=')))        // true
    s.skip_blanks()
    print(read_integer(&s))         // Ok(42)
    s.skip_blanks()
    print(s.skip_line_comment("//".bytes))
```

Everything above works in bytes, and that is not an oversight: every structural character of every
text format on the machine is ASCII, and no ASCII byte can appear inside a UTF-8 character. Reach for
`next_char` where the *content* is text — an identifier that may be Greek, a string being copied out
— and pay for one decode per character that needed one rather than one per byte of the file.

## Saying where the mistake was

```sysl
val src = source_of("config.json", text)
val d = expected(span, "a value", found_at(src, span))
    .with_note("a value is a number, a string, `true`, `false`, `null`, an array or an object")

print(d.render(src))
```

```
error: expected a value
 --> config.json:3:8
  |
3 |   "b": ,
  |        ^ found `,`
  |
  = note: a value is a number, a string, `true`, `false`, `null`, an array or an object
```

The caret is placed by **display width** rather than by byte count, so a line with a tab or a Chinese
character in it still points at the right thing. A `Report` gathers many, sorts them by where they
are, and prints the first five with `showing the first 5 of 23` — because a parser that resumes
produces a great many diagnostics and the twentieth is never read.

Nothing here is coloured. ANSI belongs to the program that knows whether its output is a terminal.

## Expressions

`pratt` replaces the nine near-identical functions a precedence-climbing recursive-descent parser
would need with one number per operator:

```sysl
private power_of(t: Tok) -> Power
    t match
        Plus -> left_assoc(10)
        Star -> left_assoc(20)
        Caret -> right_assoc(30)          // 2 ^ 3 ^ 2 is 512, not 64
        Bang -> left_assoc(40)            // postfix: its callback reads nothing
        _ -> left_assoc(0)                // not an infix operator, so the loop stops
```

Associativity is the difference between two numbers and nothing else — recurse at the operator's own
power and it groups to the left, one below it and it groups to the right. Prefix operators, postfix
operators, indexing, calls and the ternary all fall out of the callbacks being free to read what they
like, so nothing about them is in the loop.

## Indentation, brackets included

A bracket suspends the off-side rule — `f(a,\n  b)` is one logical line, and every language with both
features implements it by counting brackets in the lexer. What that costs, if it is the whole rule,
is that a construct whose body is an indented block cannot be written as an argument.

`layout` lets a grammar say otherwise. `opens_block` marks the token that opens one, and from there
the block's own lines count until the bracket that surrounds it closes — at which point
`close_bracket` answers how many blocks it closed, so the lexer emits their `DEDENT`s before the `)`:

```
print(n match
    0 -> "none"
    1 -> "one")
```

**Which tokens open a block is the grammar's to say, and is not guessable here.** sysl's are `match`
and `->` and deliberately nothing else; another language's might be `:`, `do` or `of`. What this
module owns is the bookkeeping that makes such a block possible.

## What is deliberately not here

**No grammar DSL and no combinators.** Closure-chained combinators allocate per alternative, and the
type noise is the kind sysl's inference rules exist to keep out. The stronger argument is first-hand:
the sysl compiler's own front end is built on parser combinators, and controlling its diagnostics
cost a run of hard-won rules about failure ranking that are not bugs in the library — they are what
it feels like to steer error reporting through a mechanism not built to expose it.

**No generic AST.** Every grammar's tree is its own type, and a `Spanned[T]` is as far as this goes.

**No error-recovery policy.** `skip_until`, `skip_until_set` and `skip_balanced` are primitives, and
where to resume is the grammar's decision — a statement parser resumes at a newline, a JSON reader at
a `,` or a `}`, an argument list at a `)`.

Each of those is how a library of this kind turns into a framework.

## Building on it

`tests_json.sysl` is a complete JSON reader in about a hundred and thirty lines: values, nesting,
escapes with surrogate pairs, JSON's narrower rules about numbers, and messages that quote the line
and point at the place. Nothing in it is scaffolding this package provides — it is what a caller
writes, and it is the shortest answer to what the package is for.

## Licence

ISC.
