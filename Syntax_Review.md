# DRLXXXX Syntax Review — Ambiguities and Error-Prone Constructs

Review date: 2026-06-22
Source: `drlx-parser/docs/DRLXXXX.md`

---

## 1. `[]` is heavily overloaded — 5 distinct meanings

`[]` is used for:

- **Constraints**: `/persons[age > 18]`
- **Property-reactive watch list**: `/location[][city]`
- **List/array access**: `cheeseList[0]`
- **Boolean test block** ('with' style): `t[status == RECEIVED]`
- **Window parameters**: `length[5]`, `time[5s]`

The critical ambiguity is between constraints and property-reactive. Given `/location[city]` — is this a constraint (testing that `city` is truthy) or a property watch? The spec distinguishes them by position (`[][]` — first is constraint, second is watch), but a single `[city]` is ambiguous to the reader and potentially to the parser.

---

## 2. `=` vs `==` inside `[]` constraint blocks

The spec states: *"Within [] blocks, and [] blocks only, == is mapped to 'equals'"*. But many examples use `=` inside `[]` where `==` was clearly intended:

- `var p : /persons[locationId = l.id]`
- `Person p : /persons[locationId = l.id, var a : age]`
- `type = r.productType`

If `=` is assignment and `==` is equality inside `[]`, then `[locationId = l.id]` would be an assignment, not a constraint — likely a spec typo, but it's exactly the kind of confusion users will have. A single `=` vs `==` typo silently changes semantics from "test equality" to "assign value."

---

## 3. `{}` is overloaded — 6+ distinct meanings

`{}` appears in:

- Rule body: `rule R1 { ... }`
- Immediate action block: `{ System.out.println(a); }`
- 'with' setter block: `t{status = RECEIVED, timestamp = new Date()}`
- `if`/`else` branch body
- `match` case body
- Java array init: `new int[] {1, 2, 3}`
- `acc` init block: `{ var count = 0; var total = 0; }`

The 'with' setter block `t{...}` is especially risky — `t{status = RECEIVED}` looks almost identical to a typo where someone forgot a comma before an action block.

---

## 4. `do` vs bare statement — subtle distinction with high consequences

The spec says:

- `do System.out.println(a)` — scheduled on the **agenda**
- `System.out.println(a)` — executed **immediately** during propagation

This is a major semantic difference (ordering, transactional behavior), yet the only syntactic signal is the presence/absence of a tiny keyword. This is very error-prone — omitting or adding `do` by accident silently changes execution semantics.

Additionally, the `when {} then {}` form introduces a third way to express consequences, and it's unclear whether `then { ... }` is equivalent to `do { ... }` or something else.

---

## 5. `var` is overloaded — declaration vs out-parameter

`var` means two different things:

- **Variable declaration**: `var p : /persons` — binds iteration variable
- **Out-parameter marker**: `/trusts(a, var b)` — indicates `b` is an unbound output

The spec itself acknowledges this tension: *"'var' is used to indicate it's an 'out' parameter... However for now, I think it's best to keep it consistent elsewhere."*

A reader seeing `var b` in `/trusts(a, var b)` would naturally assume it's declaring a new variable — which it partly is, but the "out" semantics are non-obvious.

---

## 6. Comma rules are inconsistent and self-contradictory

The spec states three rules:

1. *"All statements must be comma separated"*
2. *"The last element in a sequence does not need ','"*
3. *"',' can be omitted before hard keywords, or after any {}"*

The spec itself is unhappy about this: *"The last ',' of 'if' or 'match' looks weird"*

Examples are inconsistent — some have trailing commas, some don't, some have commas after `}`, some don't. This will make the grammar hard to learn and error messages hard to produce.

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

---

## 8. `if`/`match` termination is awkward

The spec requires a trailing `,` after `if`/`else` blocks:

```
if (cond) {
   ...
} else {
   ...
},    // <-- this comma looks wrong
do System.out.println(c + " " + p)
```

Three alternative syntaxes are proposed without resolution. The "comma after closing brace" pattern conflicts with every C-family language's muscle memory.

---

## 9. `#` overloading — cast vs coercion

`#` means:

- **Inline cast**: `/objects#Car[speed > 80]` — casts to Car
- **Unit coercion**: `5#Litres`, `"01-01-2005"#StdDate` — converts literal

These look similar but have very different semantics. Is `value#Integer` a cast or a coercion? Context-dependent answers are a source of bugs.

---

## 10. `/` path prefix edge cases

`/` is used for:

- DataStore: `/persons`
- Expression ('from'): `/new int[] {1, 2, 3}`
- Query call: `/trusts(a, b)`

The "from expression" form `/new int[] {1, 2, 3}` is especially odd. The `/` before an arbitrary Java expression is unlike anything in Java or XPath. It will surprise users and create parser ambiguity with `/` as division.

---

## 11. `|` for windows vs bitwise OR

Windows use `|`: `/withdrawals | length[5]`

But `|` is also Java's bitwise OR. Inside constraints like `/data[flags | MASK > 0]`, the parser must distinguish "window operator" from "bitwise OR." The spec doesn't address this.

---

## 12. Spec-level typos that demonstrate real user risks

Several examples have what appear to be typos, but they demonstrate the kind of mistakes users will make:

- `locationId == i.id=` — wrong variable `i` (should be `l`) and trailing `=`
- Same `i.id=` typo repeated in the next example
- `System.outprintln(a)` — missing dot
- `/trusts(var a, z)` — `a` is a parameter, `var a` as out makes no sense

---

## Summary: Top 3 risks

1. **`=` vs `==` inside `[]`** — single-character difference between constraint and assignment, no guard rail
2. **`do` vs bare statement** — presence/absence of a keyword silently changes execution semantics (agenda vs immediate)
3. **`acc()` positional parameter count** — meaning shifts based on comma count, very fragile
