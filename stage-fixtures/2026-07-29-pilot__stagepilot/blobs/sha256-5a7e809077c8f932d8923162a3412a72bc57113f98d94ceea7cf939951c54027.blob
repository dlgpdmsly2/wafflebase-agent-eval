# Support whole-column, whole-row, and open-ended range references (`A:A`, `1:1`, `A1:B`)

### Description:

The current `REF` and `REFRANGE` lexer tokens strictly require both column letters and row numbers (e.g. `[A-Z]+[0-9]+`). Standard Excel / Google Sheets range expressions that reference an entire column (e.g. `A:A`), an entire row (e.g. `1:1`), or an open-ended segment (e.g. `A1:B`) cannot be expressed because they fail to lex.

### What you expected to happen:

The parser should recognise and evaluate the following shapes:

- `=SUM(A:A)` — entire column A
- `=COUNT(1:1)` — entire row 1
- `=AVERAGE(B2:D)` — open-ended column range starting at B2, extending to the bottom of column D
- `=SUM(A:C)` — multi-column whole range
- `=SUM(2:5)` — multi-row whole range

Each behaves as if the range covered every populated cell in the named columns / rows (matching Excel / Google Sheets semantics, where unused cells contribute zero / nothing).

### How to reproduce it (as minimally and precisely as possible):

**Setup:** A1=`10`, A2=`20`, A3=`30`, B1=`100`, B2=`200`.

| Cell | Formula | Expected | Actual | Status |
| --- | --- | --- | --- | --- |
| D1 | `=SUM(A:A)` | `60` | `#ERROR!` | ✗ |
| D2 | `=SUM(1:1)` | `110` | `#ERROR!` | ✗ |
| D3 | `=AVERAGE(B2:B)` | `200` | `#ERROR!` | ✗ |
| D4 | `=COUNT(A:B)` | `5` | `#ERROR!` | ✗ |
| D5 | `=SUM(1:3)` | `360` | `#ERROR!` | ✗ |

All five formulas fail during the lexing/parsing phase due to the strict `REF` structural matching.