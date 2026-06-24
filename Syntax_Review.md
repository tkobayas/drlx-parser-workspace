# DRLXXXX Syntax Review — Ambiguities and Error-Prone Constructs

Review date: 2026-06-22
Source: `drlx-parser/docs/DRLXXXX.md`

---

## ~~1. `[]` is heavily overloaded — 5 distinct meanings~~ (resolved)

`[]` is used for constraints, property-reactive watch, list/array access, boolean test blocks, and window parameters. However, the spec's positional rule resolves the main concern: the first `[]` is always constraints, the second is always property-reactive (`/location[][city]`). So `/location[city]` is unambiguously a constraint. The other uses occur in distinct syntactic contexts (expressions, not `/path` patterns), so the parser can disambiguate them.

---

## 2. `=` vs `==` inside `[]` constraint blocks — future task (#94)

The spec typos (`=` where `==` was intended) have been corrected. However, the parser currently accepts `=` (assignment) inside `[]` constraint blocks. The `drlxExpression` rule falls through to the general `expression` rule (Mvel3Parser.g4 line 138), which includes `=` as an assignment operator. No semantic validation rejects it.

This means `/persons[locationId = l.id]` silently parses as an assignment rather than failing or being treated as a constraint. The downstream MVEL3 compilation does catch this with a `KieMemoryCompilerException` ("incompatible types: int cannot be converted to java.lang.Boolean"), so it is not silent — but the error message doesn't hint at the likely `=`/`==` typo.

**Status:** Issue [#94](https://github.com/tkobayas/drlx-parser/issues/94) created. Spec and plan written (`specs/2026-06-23-assignment-hint-in-constraints.md`, `plans/2026-06-23-assignment-hint-in-constraints.md`). Implementation suspended — not critical for syntax review. The fix is to enhance the error message in `DrlxLambdaCompiler.compileBatch()` with a hint.

---

## ~~3. `{}` is overloaded — 6+ distinct meanings~~ (resolved)

Most `{}` uses follow standard Java conventions (rule body, action blocks, `if`/`else`, array init). The only DRLX-specific form is the 'with' setter block `t{status = RECEIVED}`, which has clear syntactic context (identifier immediately preceding `{`). Not more overloaded than Java itself.

---

## 4. `do` vs bare statement — subtle distinction with high consequences (pending clarification)

The spec says (line 658): *"If the 'do' is omitted, those statements are executed during propagation."* So `do` = agenda, bare = immediate. This is a clear design intent.

However, several earlier examples in the spec appear contradictory — they show final action blocks WITHOUT `do` in contexts that look like agenda-scheduled consequences:

- **Rules and Consequences** (lines 254, 269, 274): final `{...}` and bare statements without `do`
- **"'do' : Immediate vs Agenda"** (lines 638-641): labeled *"Normal 'do'"* but example has no `do` keyword
- **Lines 653-655**: labeled *"Same as above but {} omitted"* but adds `do` to two of three statements, changing semantics

From line 658 onward, the spec is internally consistent. The contradictions are in the introductory/overview examples — likely the spec evolved and earlier examples weren't updated.

**Status:** Checking with the document author whether the earlier examples are mistakes. The `do` vs bare distinction itself is intentional design, not an ambiguity.

---

## 5. `var` is overloaded — declaration vs out-parameter

`var` means two different things:

- **Variable declaration**: `var p : /persons` — binds iteration variable
- **Out-parameter marker**: `/trusts(a, var b)` — indicates `b` is an unbound output

The spec itself acknowledges this tension: *"'var' is used to indicate it's an 'out' parameter... However for now, I think it's best to keep it consistent elsewhere."*

A reader seeing `var b` in `/trusts(a, var b)` would naturally assume it's declaring a new variable — which it partly is, but the "out" semantics are non-obvious.

**Recommendation:** Use `out` keyword for out-parameters instead of `var`. e.g. `/trusts(a, out b)`. This gives each keyword exactly one job and has precedent in C# (`int.TryParse("123", out result)`). The spec's rationale of "consistency" is undermined by overloading `var` with two different semantics.

---

## 6. Comma rules (pending clarification)

The spec states three rules:

1. *"All statements must be comma separated"*
2. *"The last element in a sequence does not need ','"*
3. *"',' can be omitted before hard keywords, or after any {}"*

Three examples appear to have comma mistakes (checking with author):

- **Line 596:** `r : /requests    ?and(...)` — missing comma between pattern and `?and`
- **Line 603:** `not /persons[ age < 18]    {...}` — missing comma between `not` pattern and `{...}`
- **Line 736:** `var p : /products[rate == Rates.LOW],` inside else branch — trailing comma before `}` that the grammar does not accept. The `branchBody` rule (DrlxParser.g4 line 148) uses `,` as a **separator** (`branchItem (',' branchItem)*`), not a terminator. A trailing comma here would be a parse error.

The spec itself acknowledges the `if`/`match` trailing comma is awkward: *"The last ',' of 'if' or 'match' looks weird"* — see item #8.

**Status:** Checking with the document author whether lines 596/603/736 are mistakes. The comma rules themselves are a deliberate design choice, not an ambiguity.

---

## 7. `acc()` positional parameter overloading

`acc()` can have 2, 3, 4, or 5 positional parameters, where the meaning of each position changes based on the total count:

| Params | Meaning |
|--------|---------|
| 2 | pattern, function |
| 3 | pattern, init, action+result |
| 4 | pattern, init, action, result (no reverse) |
| 5 | pattern, init, action, reverse, result |

This is very error-prone. Adding or removing a comma changes which parameter is "init" vs "action" vs "result." A miscounted parameter silently reinterprets the others.

**Options:**

1. **Drop inline custom accumulate (simplest).** The 3-5 parameter forms are all inline custom accumulate. Current DRL already [discourages this](https://kie.apache.org/docs/10.2.x/drools/drools/language-reference-traditional/index.html#drl-rules-WHEN-elements-ref_drl-rules-traditional): *"This form of accumulate is supported for backward compatibility only."* Removing it eliminates the problem entirely — only the clean 2-parameter form (`acc(pattern, function)`) remains.

2. **Named sections with keywords.** If inline custom accumulate is kept, use labeled sections (same pattern as `edge(onAdd ..., onUpdate ..., onRemove ...)`):
   ```
   acc(p : /persons,
       init int s,
       action s = s + p.age,
       reverse s = s - p.age,
       result int sum = s
   )
   ```
   Optional sections (`init`, `reverse`) can simply be omitted without shifting meaning.

---

## ~~8. `if`/`match` termination is awkward~~ (resolved)

The trailing `,` after `if/else` blocks is actually consistent. The rule body is a comma-separated list of elements (patterns, if-blocks, do-blocks). The `if/else` construct is one element in that list — the `,` after `}` is just the regular separator. Omitting `,` before `do` would be the real inconsistency.

**Note for docs author:** The spec says *"The last ',' of 'if' or 'match' looks weird"* and proposes alternatives. We vote for "always separate with comma" — it's consistent with the rest of the language and doesn't need special-casing.

---

## ~~9. `#` overloading — cast vs coercion~~ (resolved)

Currently only inline cast is implemented (`#Type`). Unit coercion (`5#Litres`, `"01-01-2005"#StdDate`) is a future feature (issue #36, de-prioritized). When coercion is implemented, it will require the target type to implement a specific interface (e.g. `UnitConverter`), so the compiler can disambiguate: `#Car` (no interface → cast) vs `#Litres` (implements interface → coercion). No syntax ambiguity.

---

## ~~10. `/` path prefix edge cases~~ (resolved, tracked as #95)

`/` is currently used for DataStore references (`/persons`) and query calls (`/trusts(a, b)`) only. The "from expression" form (`/new int[] {1, 2, 3}`) is not yet implemented — tracked as issue [#95](https://github.com/tkobayas/drlx-parser/issues/95).

No ambiguity with division: `/` as OOPath only appears in rule element position (after `:` or at the start of a comma-separated element), while `/` as division only appears in expression position (inside `[]` constraints, assignments). These are distinct grammar contexts.

---

## ~~11. `|` for windows vs bitwise OR~~ (resolved)

No ambiguity. Window `|` is a `windowFilter` suffix at rule element level (after OOPath/constraints). Bitwise OR `|` is inside the `expression` rule (inside `[]` constraints). Different grammar positions — the parser distinguishes them by context.

---

## ~~12. Spec-level typos that demonstrate real user risks~~ (fixed)

The `=` vs `==` typos in the spec examples have been corrected in DRLXXXX.md. Other typos (`System.outprintln`, wrong variable names) remain as documentation-level issues, not parser concerns.

---

## 13. `t[...]` boolean test block vs list/map accessor (#93/#65)

The spec defines `[]` on a variable as a boolean test block: `t[status == RECEIVED]` returns `true`/`false`. But `[]` on a variable is also list/map access: `cheeseList[0]`, `cheeseMap["stilton"]`.

The parser cannot distinguish these because the expression inside `[]` can look identical in both cases:

- `list[0]` — list access (get element at index 0)? Or boolean test (`0` is truthy)?
- `map["key"]` — map access (get value for "key")? Or boolean test (string is truthy)?
- `list[someVar]` — list access by variable index? Or boolean test on a variable?

For `List<String>` or `Map<String, Integer>`, the expression inside `[]` is not necessarily boolean-typed, so the parser can't use the expression's return type to decide. Type information is unavailable at parse time.

**Status:** Tracked as issue [#93](https://github.com/tkobayas/drlx-parser/issues/93) / [#65](https://github.com/tkobayas/drlx-parser/issues/65). May require a syntax change to disambiguate (e.g. a different notation for boolean test blocks on plain variables).

---

## Summary

**Resolved:** #1 (`[]` overloading — positional rule disambiguates), #3 (`{}` overloading — follows Java conventions), #8 (`if`/`match` comma — consistent, vote for always-comma), #9 (`#` overloading — disambiguated by interface at compile time), #10 (`/` prefix — distinct grammar contexts, no ambiguity), #11 (`|` — distinct grammar contexts, no ambiguity), #12 (spec typos fixed)

**Future task:** #2 (`=` vs `==` — issue #94 tracks improved error hint, not critical)

**Open design concerns (not bugs — language design questions for future consideration):**
- #4 `do` vs bare statement, #5 `var` overloading, #6 comma rules, #7 `acc()` positional params, #13 `t[...]` boolean test vs list/map access (#93/#65)
