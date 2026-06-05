---
layout: post
title: "#61 — @DataSource annotation and the end of epic #77"
date: 2026-06-05
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [queries, annotations, datasource, epic-77]
---

# #61 — @DataSource annotation and the end of epic #77

The last issue in the query enhancements epic: `@DataSource("trustworthy")` on a query rule overrides the implicit entry point name. Without it, `rule Trusts(...)` creates entry point `trusts` (lowercased rule name). With the annotation, the Unit class matches whatever name the annotation declares. Four commits, eight new tests, epic #77 closed.

## Extending the annotation pipeline

The infrastructure was already in place from `@Salience` and `@Description`. Claude and I followed the same pipeline: new `DataSource.java` annotation class, `DATASOURCE` added to the `Kind` enum, FQN mapping in the visitor's `SUPPORTED_ANNOTATION_KINDS`, string-literal validation in `extractAnnotationLiteral()`. One thing the plan missed — the protobuf serialization layer also switches on `Kind`, so the proto definition and both `fromProtoKind`/`toProtoKind` methods needed updating too. The exhaustive switch caught it at compile time.

The runtime builder change looked simple: read the `@DataSource` value from the annotation list and use it as the `queryRegistry` key instead of the default lowercased name. One `stream().filter().findFirst().orElse()` replacement.

## The silent failure nobody asked for

The first happy-path test came back with empty results. No exception, no warning — the query just matched nothing. Claude traced it to the self-reference resolution path in `buildLhs`. When the query `Trusts` is registered under `trustworthy`, its own body pattern `/trusts(x, y)` calls `queryRegistry.get("trusts")` and gets `null`. That makes it fall through to the regular `buildPattern` path instead of `buildSelfReferencePattern`.

The difference matters: `buildSelfReferencePattern` wraps each constraint with `DrlxUnificationConstraint`, which skips evaluation when a parameter is unbound (the `Variable.v` sentinel). Without the wrapper, the constraint evaluates `a == Variable.v` literally, which always fails. Empty results, zero noise.

The fix is dual-registration — the query goes into `queryRegistry` under both the override name (for external invocation by other rules) and the default lowercased name (for self-reference resolution in the query body):

```java
queryRegistry.put(entryPointName, query);
if (!entryPointName.equals(defaultName)) {
    queryRegistry.put(defaultName, query);
}
```

## What landed

| Commit | Change |
|--------|--------|
| `06fb88b` | `DataSource.java` annotation + `DATASOURCE` IR kind + proto |
| `b53e088` | Visitor resolution and validation |
| `c67d4da` | Runtime builder entry point override with dual-registration |
| `d7569af` | 8 tests: 3 happy-path, 5 error-case |

Epic #77 (Query enhancements) is complete — six issues shipped across passive invocation, result binding, handle access, named access, and now `@DataSource` override. The `@Rule` annotation for Java Unit fields was split to #84 under the de-prioritized epic #80, since it requires drools-side changes.
