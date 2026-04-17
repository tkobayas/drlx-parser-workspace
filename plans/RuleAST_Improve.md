# RuleAST Improvement Analysis

## Summary

`RULE_AST` is a useful optimization because it removes ANTLR reparsing and tree
walking at `KieBase` build time. In this repository, that clearly improves the
cache-based build path.

The main downside is not the raw runtime cost of loading the snapshot. The
larger downside is that `RULE_AST` currently introduces a second semantic build
path with reduced fidelity compared to reparsing the DRLX source and using the
normal visitor.

That means the current tradeoff is:

- Faster cached build path
- Higher risk of semantic drift, stale cache interpretation, and feature gaps

## Current Downsides

### 1. Reduced fidelity compared to reparsing

The snapshot only stores a compact subset of rule information:

- package name
- imports
- rule names
- pattern type name
- binding name
- entry point
- condition expressions as strings
- consequence block as a string

See [src/main/proto/drlx_rule_ast.proto](/home/tkobayas/usr/work/mvel3-development/drlx-parser/src/main/proto/drlx_rule_ast.proto) and [src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java](/home/tkobayas/usr/work/mvel3-development/drlx-parser/src/main/java/org/drools/drlx/builder/DrlxRuleAstParseResult.java).

Anything not captured in that model cannot affect rebuild later. Reparsing does
not have that limitation because the full grammar is interpreted again.

### 2. Two semantic paths must stay aligned

The normal path builds rules through
[src/main/java/org/drools/drlx/builder/DrlxToRuleImplVisitor.java](/home/tkobayas/usr/work/mvel3-development/drlx-parser/src/main/java/org/drools/drlx/builder/DrlxToRuleImplVisitor.java).

The cached `RULE_AST` path builds rules through
[src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java](/home/tkobayas/usr/work/mvel3-development/drlx-parser/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java).

Even though `DrlxRuleAstRuntimeBuilder` reuses some helper methods by extending
`DrlxToRuleImplVisitor`, it is still a separate reconstruction path. Any new
syntax or semantics must be implemented in both places or the two-step build can
drift from the normal build.

### 3. Cache invalidation is too narrow

The current invalidation key is based on source hash only. If DRLX text stays
the same but parser semantics, builder behavior, or snapshot interpretation
changes, an old snapshot can still be loaded.

This is the major downside of any IR cache compared with reparsing. Reparsing
always uses the current parser and builder semantics.

### 4. Current reconstruction uses heuristics

`DrlxRuleAstRuntimeBuilder` detects referenced bindings by regex matching
variable names inside condition strings. See
[src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java](/home/tkobayas/usr/work/mvel3-development/drlx-parser/src/main/java/org/drools/drlx/builder/DrlxRuleAstRuntimeBuilder.java).

That is workable for the current narrow syntax, but it is more fragile than
reusing fully parsed semantic information.

### 5. Debuggability is worse

With reparsing, the full parse tree and token stream are still available during
build. With `RULE_AST`, the source has already been compressed into a narrower
representation, so some debugging and future tooling options are lost.

### 6. Performance ceiling is elsewhere

The performance notes show that after moving to `RULE_AST`, cache load itself is
cheap and the dominant remaining cost is evaluator hidden-class loading rather
than protobuf decode or AST restore. See
[PERF_ANALYSIS.md](/home/tkobayas/usr/work/mvel3-development/drlx-parser/PERF_ANALYSIS.md).

That means `RULE_AST` is useful, but further optimization in this area alone
will eventually hit diminishing returns.

## Suggested Direction

The cleanest way to keep the benefit of `RULE_AST` while reducing the downsides
is to make it a serialization of a canonical semantic model, not a separate
reconstruction strategy.

Target shape:

1. Parse DRLX source once
2. Build a canonical semantic model
3. Either:
   - build `KieBase` directly from that model, or
   - serialize that same model for later reuse
4. On cache load, deserialize the same model and run the same downstream build

This removes the biggest problem: duplicated semantics.

## Concrete Suggestions

### 1. Introduce a canonical semantic IR

Create an internal semantic representation that sits between parse tree walking
and `RuleImpl` construction.

This IR should contain enough resolved structure to express the supported DRLX
subset without needing heuristics at load time. For example:

- package/imports
- rules
- ordered rule items
- pattern identity
- resolved type name
- bind name
- entry point
- explicit constraint kind
- explicit referenced bindings for beta constraints
- consequence input bindings

Then both paths use it:

- normal build: parse tree -> semantic IR -> `KieBase`
- cached build: snapshot -> semantic IR -> `KieBase`

This makes one builder path authoritative.

### 2. Move snapshot generation after semantic resolution

Do not serialize raw strings as the only semantic carrier if more structure is
already known during prebuild.

If the prebuild phase already knows which bindings are referenced and which
constraint form should be created, persist that resolved information directly.

That avoids recomputing semantics from strings during cache load.

### 3. Strengthen cache invalidation

Add explicit version fields to the snapshot header, for example:

- snapshot format version
- grammar version
- runtime builder version
- semantic IR version

Then reject cached snapshots when any of those versions differ.

Optional follow-up:

- include relevant environment fingerprinting if import/type resolution changes
  can alter the result even when DRLX text is unchanged

This is the main safeguard against stale cache interpretation.

### 4. Eliminate regex-based binding detection

`findReferencedBindings()` is an implementation shortcut, not a robust long-term
foundation.

Instead, persist referenced bindings explicitly in the snapshot or the semantic
IR. That gives deterministic reconstruction and avoids false positives or false
negatives from string matching.

### 5. Keep reparsing as the correctness fallback

The non-cache strategy remains the safest reference implementation.

If snapshot loading fails validation, if versions mismatch, or if a future rule
feature is not supported by the snapshot format, fall back to reparsing
immediately.

This keeps the optimized path opportunistic instead of making it mandatory.

### 6. Add differential equivalence tests

For each supported rule shape, compare:

- normal build with reparsing
- `RULE_AST` build from prebuilt artifacts

The tests should verify both:

- structural equivalence where practical
- behavioral equivalence by firing rules against representative facts

This converts semantic drift into a test failure instead of a latent production
bug.

Recommended coverage:

- alpha constraints
- beta constraints
- multi-join rules
- multiple imports
- multiple rules per unit
- consequence blocks using multiple bindings
- edge cases in identifier naming

### 7. Treat `RULE_AST` as a compatibility boundary

Any time DRLX syntax support expands, update all of the following together:

- parser handling
- semantic IR
- snapshot format
- cache loader
- equivalence tests

That process discipline matters more than the protobuf format choice.

### 8. Shift further performance work toward evaluator loading

Once `RULE_AST` correctness is solid, the next larger performance opportunity is
likely not more cache compression. The perf analysis already shows the dominant
remaining cost is hidden-class definition for precompiled evaluators.

So the likely optimization order should be:

1. fix semantic duplication
2. harden cache invalidation
3. remove heuristic reconstruction
4. expand equivalence tests
5. investigate lambda/evaluator loading overhead

## Recommended Implementation Plan

### Phase 1: Safety

- add snapshot version fields
- reject stale snapshots aggressively
- expand tests for `RULE_AST` vs normal build equivalence

This gives immediate risk reduction without a large refactor.

### Phase 2: Model unification

- introduce a semantic IR
- move normal build to parse tree -> IR -> `KieBase`
- move `RULE_AST` load to snapshot -> IR -> `KieBase`

This is the highest-value architectural change.

### Phase 3: Remove heuristics

- persist referenced bindings explicitly
- remove regex-based inference from runtime rebuild
- persist any other information currently reconstructed indirectly from strings

### Phase 4: Next performance focus

- profile evaluator loading and hidden-class creation
- decide whether class definition, class reuse, or lambda packaging can be
  improved further

## Bottom Line

`RULE_AST` is a good direction, but it should not remain a separate semantics
path indefinitely.

If it becomes a versioned serialization of a canonical semantic IR, most of the
current downsides go away:

- lower drift risk
- stronger invalidation
- better maintainability
- less heuristic rebuild logic

At that point, the main remaining tradeoff versus reparsing is normal cache
complexity, which is acceptable if the performance gain is worth it.
