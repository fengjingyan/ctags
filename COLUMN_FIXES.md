# Column / endColumn / selectionRange fixes

Notes on the changes made on top of `feat_column_endLine_endColumn` to make
the `column`, `endColumn`, and `selection*` fields actually correct for
VSCode / LSP `DocumentSymbol` consumption.

The fixes were validated against `ctags.c` (a small reproducer with a
typedef'd anonymous struct, an anonymous enum, a macro `#define`, a
prototype invocation, and a function definition).

## 1. Bug inventory

Three independent defects were causing wrong numbers in the generated JSON.

### Bug A — off-by-one column for tokens whose first char is hex / `R`

Examples from the reproducer:

| Tag | reported `column` | expected | first char |
|---|---|---|---|
| `RED` (line 9) | 6 | 5 | `R` |
| `BLUE` (line 11) | 6 | 5 | `B` (hex digit) |
| `COLOR` (line 12) | 4 | 3 | `C` (hex digit) |
| `DEFINE_HANDLER` prototype (line 14) | 2 | 1 | `D` (hex digit) |
| `GREEN` (line 10) | 5 | 5 | `G` (correct) |

**Root cause.** Inside `cppGetc` there are multiple peek-and-unget sites:
- C++ raw literal lookahead (`R"..."`) — triggers for any token starting with `R`
- Hex digit-separator lookahead — triggers for `0-9 a-f A-F`
- `@"..."` literal lookahead
- BACKSLASH, `?`, `<`, `:`, `%`, `//` (COMMENT_CPLUS)

Each site looked like:

```c
int next = cppGetcFromUngetBufferOrFile();   // peek
if (cond)
    cppUngetc(next);                         // put back
```

The old `cppUngetc` snapshotted `getInputDisplayColumnNumber()` *after*
the peek, so the stored `ungetBuffer->columnNumber` was the column of the
peeked-past char, not the char that was just returned to the caller. The
next `cppGetInputColumnNumber()` therefore read a value one too large.

An earlier patch (`fec92f12e 修改列号错位问题`) worked around this at
three sites (`@`, `R`, `isxdigit`) with a `savedColumn`/`noBuffer`/overwrite
dance. The remaining ~6 peek sites were untouched.

### Bug B — `#define` macro tag has `column: 0, endColumn: 0`

`directiveDefine` in `cpreprocessor.c` was creating the macro tag without
ever calling `setTagColumn` / `setTagEndColumn`. After `memset`, both
fields were zero. The earlier patch already addressed this; documented
here for completeness.

### Bug C — anonymous container `endColumn` is the synthetic-name length

```
__anonde3138680108  end=6   endColumn=34    (line 6 is "} SA;" — col 34 has no text)
__anonde3138680203  end=12  endColumn=32    (line 12 is "} COLOR;" — col 32 has no text)
```

The synthetic anonymous identifier (e.g. `__anonde3138680108`, 18 chars)
was created with `iEndColumnNumber = iColumnNumber + strlen(name)`. That
gets stored as the tag's `endColumn` in `cxxRefTagBegin` and was never
updated when the matching closing `}` was parsed. (Contrast `endLine`,
which *was* being updated via `cxxParserMarkEndLineForTagInCorkQueue`.)

The same defect propagated to the `selection*` fields, which are
initialized from the same token and likewise never re-touched.

## 2. Approach

### A — centralize the rewind inside `cppUngetc`

Rather than copying the `savedColumn` boilerplate to every peek site
(error-prone, easy to miss new sites), move the adjustment one level down
to `cppUngetc` itself: when it creates a *fresh* ungetBuffer, snapshot
`getInputDisplayColumnNumber() - 1` instead of the raw value. That single
change fixes every peek-and-unget pattern in `cppGetc`.

When `cppUngetc` is called while a buffer already exists (macro
expansion replay, recursive `STRING_SYMBOL` / `CHAR_SYMBOL` expansion in
`ungetBufferUngetc`), the columnNumber must *not* be touched — it
encodes the macro arg's source position. The `if (Cpp.ungetBuffer ==
NULL)` guard preserves that.

The three per-site overwrites added by `fec92f12e` are now redundant and
were removed.

### C — symmetric "Mark end column" helper

Add `setTagEndColumnToCorkEntry` in `main/entry.{c,h}` mirroring the
existing `setTagEndLineToCorkEntry`. In the cxx layer, add
`cxxParserMarkEndColumnForTagInCorkQueue` (uses
`g_cxx.pToken->iEndColumnNumber`, intended to be called when the closing
`}` has just been consumed) and `cxxParserSetEndColumnForTagInCorkQueue`
(explicit column for callers that already have the relevant token).

Wire these in alongside every existing `MarkEndLine` / `SetEndLine` call
for **container/block** endings — but **not** at variable / prototype
end sites, so `int x;` continues to report `endColumn = (column after
x)`, matching the field's documented "column just after the tag name"
semantic.

### Anonymous-tag `selectionRange` — collapse to zero width

`cxxTokenCreateAnonymousIdentifier` was setting `iEndColumnNumber =
iColumnNumber + vStringLength(pszWord)` — i.e. it pretended the synthetic
"__anon..." string lived in the source. It doesn't. The cleanest fix is
`iEndColumnNumber = iColumnNumber`, giving a zero-width range at the
nominal start position (the `{` of the body, for anon structs/enums/etc).
That's what LSP servers conventionally emit for anonymous symbols, and
it propagates through `cxxRefTagBegin` so all four `selection*` fields
get the same collapsed value automatically.

The anonymous-parameter callers in `cxx_parser_function.c` overwrite
both `iColumnNumber` and `iEndColumnNumber` immediately after the
factory returns, so they're unaffected.

## 3. File-by-file changes

| File | Change |
|---|---|
| `parsers/cpreprocessor.c` | `cppUngetc`: store `col-1` instead of `col` when creating a fresh ungetBuffer. Removed the now-redundant `savedColumn`/`noBuffer` boilerplate at the `@`, `R`, and `isxdigit` peek sites. |
| `main/entry.h` | Declared `setTagEndColumnToCorkEntry`. |
| `main/entry.c` | Defined `setTagEndColumnToCorkEntry` (mirror of `setTagEndLineToCorkEntry`). |
| `parsers/cxx/cxx_parser_internal.h` | Declared `cxxParserMarkEndColumnForTagInCorkQueue` and `cxxParserSetEndColumnForTagInCorkQueue`. |
| `parsers/cxx/cxx_parser.c` | Defined the two new helpers. Wired `MarkEndColumn` next to `MarkEndLine` at the enum-body-close and class/struct/union/scope-body-close sites. |
| `parsers/cxx/cxx_parser_block.c` | Capture `uEndColumn = g_cxx.pToken->iEndColumnNumber` alongside `uEndPosition` for the function-body / try-block close, and call `SetEndColumn` next to `SetEndLine`. |
| `parsers/cxx/cxx_parser_function.c` | `MarkEndColumn` alongside `MarkEndLine` at the K&R function close. |
| `parsers/cxx/cxx_parser_namespace.c` | `MarkEndColumn` alongside `MarkEndLine` at the namespace close. |
| `parsers/cxx/cxx_parser_lambda.c` | `MarkEndColumn` alongside `MarkEndLine` at the lambda close. |
| `parsers/cxx/cxx_token.c` | In `cxxTokenCreateAnonymousIdentifier`, set `iEndColumnNumber = iColumnNumber` (was `iColumnNumber + vStringLength(pszWord)`). |

Intentionally **not** changed:

- `parsers/cxx/cxx_parser_variable.c` — variables keep `endColumn = "just after the name"`.
- `parsers/cxx/cxx_parser.c` prototype site (around line 1694) — prototypes likewise.

That preserves the field-description semantics for symbols that have no
body to span.

## 4. Before / after for `ctags.c`

Source (CRLF, 4-space indent):

```
 1  #define DEFINE_HANDLER(name) void name##_init(void)
 2
 3  typedef struct {
 4      int x;
 5      int y;
 6  } SA;
 7
 8  typedef enum {
 9      RED,
10      GREEN,
11      BLUE
12  } COLOR;
13
14  DEFINE_HANDLER(network);
15
16  void test() {
17      SA sa;
18      COLOR color = RED;
19  }
```

Only the columns that changed are shown.

| Tag | line | before                            | after                              |
|---|---|---|---|
| `DEFINE_HANDLER` (macro)   | 1  | `column: 0, endColumn: 0`         | `column: 9, endColumn: 23` (B)     |
| `RED`                      | 9  | `column: 6, endColumn: 9`         | `column: 5, endColumn: 8`  (A)     |
| `BLUE`                     | 11 | `column: 6, endColumn: 10`        | `column: 5, endColumn: 9`  (A)     |
| `COLOR`                    | 12 | `column: 4, endColumn: 9`         | `column: 3, endColumn: 8`  (A)     |
| `DEFINE_HANDLER` (proto)   | 14 | `column: 2, endColumn: 16`        | `column: 1, endColumn: 15` (A)     |
| `__anon…0108` (anon struct)| 3  | `endColumn: 34, selEndCol: 34`    | `endColumn: 2, selEndCol: 16` (C)  |
| `__anon…0203` (anon enum)  | 8  | `endColumn: 32, selEndCol: 32`    | `endColumn: 2, selEndCol: 14` (C)  |
| `test`                     | 16 | `end: 19, endColumn: 10`          | `end: 19, endColumn: 2`        (C) |

Unchanged: `SA`, `GREEN`, `x`, `y`, `sa`, `color`, `name`, `network`,
the file tag, etc.

## 5. Field semantics after these changes

| Field | What it means |
|---|---|
| `column` / `endColumn` | LSP-`range`-like. For container/function/lambda/namespace tags this now spans from the start of the declaration to the column just past the closing `}`. For variables, parameters, members, prototypes, enumerators, typedef names — the identifier extent ("just after the name"), unchanged. |
| `selectionStartLine`/`Column`, `selectionEndLine`/`Column` | LSP-`selectionRange`-like. The identifier itself for named tags. For anonymous tags, a zero-width range at the nominal start position (the `{`), conventionally what LSP servers emit. |

## 6. Out of scope / further work

- The multi-peek paths in `cppGetc` (`?`, `<`, trigraph fallback) still
  do >1 peek per branch. The central `col-1` fix is correct for the
  single-peek case (the vast majority) but is off by `N-1` if the
  branch ends up doing `N` peeks. None of these branches can trigger
  for C identifiers, so they don't affect tag columns in the test
  corpus; fixing them properly needs per-char column tracking inside
  the unget buffer.
- The `selection*` fields are still copied from the same token in
  `cxxRefTagBegin`. If a richer "selection range" definition is wanted
  later (e.g. always the identifier including for compound declarators),
  it would need its own update site rather than riding on the same
  token as `column`/`endColumn`.
