# @DataSource annotation for query entry point override — Design Spec

**Issue:** #61
**Date:** 2026-06-05

## Summary

Add `@DataSource("name")` annotation support on DRLX query rules to override the implicit entry point name. Currently, `rule Trusts(Object a, Object b)` creates entry point `trusts` (lowercased rule name). With `@DataSource("trustworthy")`, the entry point becomes `trustworthy` instead, so the Unit class field must be `DataSource<Trust> trustworthy`.

## DRLX syntax

```
import org.drools.drlx.annotations.DataSource;

@DataSource("trustworthy")
rule Trusts(Object a, Object b) {
    or (/trusts(a, b),
        and (/trusts(var z, b),
             /trusts(a, z)))
}
```

## Constraints

- Only allowed on query rules (rules with parameters). Error if used on non-query rules.
- Argument must be a non-empty string literal.
- No duplicate `@DataSource` on the same rule (handled by existing duplicate-detection logic).
- Grammar already supports `annotation*` on `ruleDeclaration` — no grammar changes needed.

## Changes

### 1. New annotation class

`DataSource.java` in `org.drools.drlx.annotations`, following the existing `Salience`/`Description` pattern:
- `@Retention(RUNTIME)`, `@Target(TYPE)`
- `String value()`

### 2. IR model — `DrlxRuleAstModel.java`

Add `DATASOURCE` to `RuleAnnotationIR.Kind` enum:
```java
public enum Kind { SALIENCE, DESCRIPTION, DATASOURCE }
```

### 3. Visitor — `DrlxToRuleAstVisitor.java`

- Add `DATASOURCE_FQN` constant and entry in `SUPPORTED_ANNOTATION_KINDS` map.
- Add `case DATASOURCE` in `extractAnnotationLiteral()` — validate string literal (same as `DESCRIPTION`), additionally reject empty strings.
- Update the "supported:" error message in `resolveKind()` to include `@DataSource`.

### 4. Runtime builder — `DrlxRuleAstRuntimeBuilder.java`

In the query-building loop (line ~117-125), after checking `!rule.parameters().isEmpty()`:
- Look for a `DATASOURCE` annotation in `rule.annotations()`.
- If present, use its `rawValue` as `entryPointName` instead of the default lowercased rule name.
- Keep the existing default logic as fallback.

In the non-query rule loop (line ~127-132):
- Check if any rule has a `DATASOURCE` annotation and throw an error: `"@DataSource is only allowed on query rules (rules with parameters)"`.

In `applyAnnotations()`:
- Add `case DATASOURCE -> {}` (no-op, already consumed during entry point setup).

### 5. Tests

Happy path:
- `@DataSource("trustworthy")` overrides entry point name — query results arrive via the overridden name.
- Import-based and FQN-based annotation resolution both work.
- Combine `@DataSource` with `@Salience` on a query rule.

Error cases:
- `@DataSource` on a non-query rule → error.
- `@DataSource` with empty string `""` → error.
- `@DataSource` without argument → error.
- `@DataSource` with non-string argument (e.g., integer) → error.
- Duplicate `@DataSource` on same rule → error (covered by existing logic).
