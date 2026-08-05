---
layout: post
title: "#103 — multi-segment OOPath end-to-end"
date: 2026-08-05
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [oopath, multi-segment, from, dataprovider, #103]
---

# #103 — multi-segment OOPath end-to-end

I wanted to know if `/persons/addresses[city == "London"]` — multi-segment OOPath — worked in drlx-parser. The answer was: the grammar parsed it, the drools runtime could execute it, but the builder in between threw the segments away.

## The grammar was already there

Claude surveyed the ANTLR grammar and found this:

```antlr
oopathExpression: QUESTION? '/' oopathRoot ('/' oopathChunk)* ;
```

The `('/' oopathChunk)*` part handles any number of segments. The `DrlxToJavaParserVisitor` correctly builds an `OOPathExpr` with multiple `OOPathChunk` nodes. But every existing test asserted `getChunks().hasSize(1)` — nobody had ever tested two segments.

We confirmed the drools-ruleunits runtime handles it too, by adding an `OOPathMultilevelTest` in `drools-ruleunits-impl` with `PersonWithAddresses` → `ReactiveList<Address>`. Passes cleanly.

## Where the pipeline broke

The gap was in `DrlxToRuleAstVisitor` and `DrlxRuleAstRuntimeBuilder`. The visitor's `collectDrlxExpressions` grabbed conditions from the last chunk only and stuffed them into a flat `PatternIR`. The intermediate segment field names — `addresses` in `/persons/addresses` — were simply lost. `PatternIR` had no field for them, and the runtime builder produced one pattern per `PatternIR`, sourced from an `EntryPointId`. No mechanism for chaining.

## The fix: segments in the IR, `From` at runtime

We added `OopathSegmentIR(field, conditions, inlineCast)` to the IR model and a `List<OopathSegmentIR> oopathSegments` field on `PatternIR`. For single-segment OOPath — the common case — the list is empty and nothing changes.

The visitor got `extractOopathSegments`, and `extractConditions` was fixed to return root conditions only (previously it returned the last chunk's conditions, which worked by accident when there was only one chunk).

The runtime builder got `buildPatternChain`. For multi-segment, it builds the root pattern from the entry point as before, then for each segment:

```java
From from = new From(new OopathFieldDataProvider(previousDecl, getter));
from.setResultPattern(segPattern);
segPattern.setSource(from);
```

`OopathFieldDataProvider` is a simple `DataProvider` — invokes the getter, returns an iterator over the result. If it's an `Iterable`, iterate it; if a single value, wrap it.

## The `From` that NPEs without a handshake

The one surprise: `From(DataProvider)` compiles fine and `pattern.setSource(from)` wires the pattern → From direction. But `From.getResultClass()` dereferences `resultPattern.getObjectType()` — and `resultPattern` is null unless you explicitly call `from.setResultPattern(pattern)`. The constructor doesn't set it. The NPE only shows up at runtime when the rete network evaluates the `FromNode`. Submitted this to the garden as a gotcha.

## What landed

Five commits on branch `103-multi-segment-oopath`: IR model extension, visitor changes, protobuf round-trip, and the runtime builder with two execution tests — one for bare `/persons/addresses[city == "London"]` and one with a root constraint `/persons[age > 25]/addresses[city == "London"]`. Full test suite green.
