# Support ECMA byte-oriented text functions (`LEFTB`, `RIGHTB`, `MIDB`, …)

### Description:

The byte-oriented text variants used by Excel / Google Sheets in DBCS (CJK) workbooks are not recognised by the formula engine and resolve to `#NAME?`: `LEFTB`, `RIGHTB`, `MIDB`, `LENB`, `FINDB`, `SEARCHB`, `REPLACEB`, `ASC`. For pure ASCII the result of a B-function equals its non-B sibling, but for multibyte text they differ — so users authoring CJK spreadsheets cannot mirror Excel/Sheets behaviour today.

### What you expected to happen:

- `LEFTB(s, n)` returns the longest prefix of `s` whose UTF-8 byte length is ≤ `n`, split at character boundaries.
- `LENB(s)` returns the UTF-8 byte length of `s`.
- `RIGHTB` / `MIDB` / `FINDB` / `SEARCHB` / `REPLACEB` use the same byte-counting helper.
- `ASC(s)` converts full-width characters (U+FF01–U+FF5E, full-width katakana) to their half-width equivalents.

### How to reproduce it (as minimally and precisely as possible):

**Setup:** Empty sheet.

| Cell | Formula | Expected | Actual | Status |
| --- | --- | --- | --- | --- |
| A1 | `=LEFTB("Hello", 3)` | `Hel` | `#NAME?` | ✗ |
| A2 | `=LENB("café")` | `5` | `#NAME?` | ✗ |
| A3 | `=LENB("Hello")` | `5` | `#NAME?` | ✗ |
| A4 | `=MIDB("Hello", 2, 3)` | `ell` | `#NAME?` | ✗ |
| A5 | `=ASC("ＨＥＬＬＯ")` | `HELLO` | `#NAME?` | ✗ |

`LENB("café") = 5` because `é` is 2 bytes in UTF-8. For pure ASCII, B-functions match the non-B versions; for multibyte text they differ.