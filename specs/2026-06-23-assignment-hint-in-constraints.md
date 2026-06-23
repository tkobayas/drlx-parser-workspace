# Spec: Hint message for assignment operator in constraint expressions

Date: 2026-06-23

## Problem

When a user writes `=` instead of `==` inside a `[]` constraint block (e.g., `/persons[locationId = l.id]`), the parser accepts it as an assignment expression. The downstream MVEL3 batch compilation then fails with a `KieMemoryCompilerException` containing a Java compiler error like "incompatible types: cannot be converted to boolean". This error is technically correct but gives no indication that the likely cause is `=` vs `==` confusion.

## Solution

Enhance the error message in `DrlxLambdaCompiler.compileBatch()` to detect this case and append a hint.

### Changes

**File:** `drlx-parser-core/src/main/java/org/drools/drlx/builder/DrlxLambdaCompiler.java`

In `compileBatch()` (line 314), wrap the `batchCompiler.compile(classLoader)` call in a try/catch for `KieMemoryCompilerException`:

1. Catch `KieMemoryCompilerException`
2. Check if the error message contains `"incompatible types"` and `"boolean"`
3. Scan the pending lambda expressions for bare `=` using regex `(?<![!=<>])=(?!=)` (matches `=` not preceded by `!`, `=`, `<`, `>` and not followed by `=`)
4. If both conditions match, re-throw with the original message plus: `"Hint: constraint expression contains '=' — did you mean '=='?"`
5. If conditions don't match, re-throw the original exception unchanged

### Test

**File:** `drlx-parser-core/src/test/java/org/drools/drlx/builder/syntax/` (new or existing test class)

Add a test that:
- Defines a rule with a constraint using `=` instead of `==` (e.g., `var p : /persons[age = 18]`)
- Asserts that the thrown exception message contains the hint text

## Out of scope

- No grammar changes
- No visitor-level validation
- No changes to MVEL3's `KieMemoryCompilerException` itself
