# #73 Runtime Support for CEP Windows (EventHandle + STREAM Mode)

**Issue:** https://github.com/tkobayas/drlx-parser/issues/73
**Epic:** #26
**Prerequisite:** #67 (basic window parsing — completed)

## Problem

Windows are correctly parsed and attached to `Pattern` as `SlidingTimeWindow` /
`SlidingLengthWindow` behaviors, but fail at runtime when facts are inserted
through `DrlxRuleUnitInstance`. Two root causes:

1. **STREAM mode missing:** `DrlxRuleBuilder.createKieBase()` hardcodes default
   `RuleBaseConfiguration` (CLOUD mode). Windows / CEP require
   `EventProcessingOption.STREAM`.

2. **`ClassObjectType.isEvent` derived from wrong signal:**
   `DrlxRuleAstRuntimeBuilder.buildPattern()` sets `isEvent` based on
   `windowType != null`. The event-ness of a type should be determined by the
   `@Role(Role.Type.EVENT)` annotation on the domain class, which is already
   picked up by `TypeDeclaration.createTypeDeclarationForBean()`.

When both issues are fixed, the fact handle creation chain works correctly:
`ClassObjectTypeConf.isEvent=true` → `RuleUnitFactHandleFactory.createEventFactHandle()`
→ `RuleUnitEventFactHandle extends DefaultEventHandle` → `WindowNode.assertObject()`
cast succeeds.

## Changes

### 1. Externalize `RuleBaseConfiguration` in `DrlxRuleBuilder`

**File:** `DrlxRuleBuilder.java`

Add overloaded methods that accept `RuleBaseConfiguration`:

| Method | Behavior |
|--------|----------|
| `createKieBase(List<KiePackage>, RuleBaseConfiguration)` | Creates KieBase with provided config |
| `createKieBase(List<KiePackage>)` | Delegates with default config (backward compatible) |
| `build(String, RuleBaseConfiguration)` | Parses DRLX then creates KieBase with provided config |
| `build(String)` | Unchanged — delegates with default config |

Callers who need STREAM mode create a `RuleBaseConfiguration`, set
`EventProcessingOption.STREAM`, and pass it in.

### 2. Fix `ClassObjectType.isEvent` derivation

**File:** `DrlxRuleAstRuntimeBuilder.java`, `buildPattern()` method

Replace:
```java
boolean isEvent = parseResult.windowType() != null;
```
With:
```java
Role roleAnnotation = type.getAnnotation(Role.class);
boolean isEvent = roleAnnotation != null && roleAnnotation.value() == Role.Type.EVENT;
```

Domain classes used with windows are expected to carry the `@Role(EVENT)`
annotation.

### 3. Session-level integration test

**File:** new `WindowTest.java` under `org.drools.drlx.builder.syntax`

Test that exercises the full runtime path:
- Builds a DRLX rule with `| length[5]` window on `withdrawals`
- Creates `KieBase` with STREAM-mode `RuleBaseConfiguration`
- Creates `DrlxRuleUnitInstance<WithdrawalUnit>`
- Inserts `Withdrawal` events via `unit.withdrawals.add(...)`
- Fires rules and asserts the rule fired (no `ClassCastException`)

## Out of Scope

- Auto-detection of STREAM mode from package content — caller is responsible
- Support for classes without `@Role(EVENT)` annotation in window patterns
- Temporal operators (`after`, `before`) — tracked in #43
