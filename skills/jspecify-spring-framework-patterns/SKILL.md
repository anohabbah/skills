---
name: jspecify-spring-framework-patterns
description: Use when you must DECIDE where @Nullable / @NonNull / @Contract belong while annotating un-annotated Java/Spring code and want the inference heuristics Spring Framework's own spring-core module uses — reading a method's implementation or Javadoc to infer a nullable return, parameter, or field, choosing the right @Contract clause for assertions/predicates/transforms, and deciding between @Nullable, @SuppressWarnings("NullAway.Init") and @SuppressWarnings("NullAway").
license: MIT
metadata:
  author: jspecify-spring-framework-patterns
  version: "1.0"
  source: https://github.com/spring-projects/spring-framework (spring-core + spring-web module analysis)
---

# Spring Framework Nullness Patterns (inference heuristics)

**Companion skill — read `jspecify-userguide` and `jspecify-spring-null-safety` first.**
Those cover the JSpecify rules, placement, generics, migration, and `@Contract`
syntax. This skill answers the harder question: **given un-annotated code, how do
you infer *where* `@Nullable` and which `@Contract` clause belong** — distilled from
how `spring-core` and `spring-web` actually annotate themselves. Evidence cited by
class + method.

## Inferring a `@Nullable` RETURN

Annotate the return `@Nullable` when **any** of these signals is present:

| Signal | Evidence (spring-core) |
|--------|------------------------|
| A `return null;` literal on some path | `StringUtils.getFilenameExtension`, `ClassUtils.resolvePrimitiveClassName`, `ReflectionUtils.findMethod` |
| Javadoc `@return ... or {@code null} if not found / if none / cannot be resolved` | `CollectionUtils.findFirstMatch`, `AnnotationUtils.getAnnotation`, `CollectionUtils.firstElement` |
| Name implies a lookup that can miss: `find*`, `get*`, `resolve*`, `firstElement`/`lastElement` | `ReflectionUtils.findField`, `CollectionUtils.findValueOfType`, `AnnotationUtils.getValue` |
| Return delegates to an already-`@Nullable` method (`opt.orElse(null)`, a `Map.get`, another `@Nullable` API) | `ObjectUtils.unwrapOptional` returns `optional.orElse(null)` |
| Method carries `@Contract("null -> null")` / `"...-> null"` | `ClassUtils.resolvePrimitiveClassName`, `CollectionUtils.firstElement` |

**Anti-pattern — do NOT mark `@Nullable`:** Spring deliberately returns an **empty
collection / array / string** instead of `null`. `StringUtils.tokenizeToStringArray`
returns `EMPTY_STRING_ARRAY`; `ClassUtils.getAllInterfacesForClassAsSet` returns an
empty set. Prefer-empty-over-null methods stay non-null. Also: `boolean`/primitive
returns are never `@Nullable` (use `@Contract` instead — see below).

## Inferring a `@Nullable` PARAMETER

| Signal | Evidence |
|--------|----------|
| Javadoc says `(may be {@code null})` / `or {@code null}` | `MimeType(String,String,@Nullable Map)` param `parameters` |
| Body guards it: `if (param != null)` | `Assert.nullSafeGet(@Nullable Supplier)` → `messageSupplier != null ? ... : null` |
| Ternary fallback to a default: `param != null ? param : default` | `CustomizableThreadCreator(@Nullable String prefix)` |
| A `@Nullable`-accepting overload exists, or the value is later passed to a `@Nullable` slot | common across `Assert.*` |

## Inferring a `@Nullable` FIELD (vs deferred init)

- **`@Nullable` field** when the value is legitimately absent: not assigned (or
  assigned `null`) in the constructor, can be reset to `null`, or is a lazily-computed
  cache. Real: `StopWatch.currentTaskName`, `MimeType.resolvedCharset`
  (`transient @Nullable`), `MimeType.toStringValue` (`volatile @Nullable`),
  `SingletonSupplier.singletonInstance` (`volatile @Nullable`, double-checked lock).
- **`@SuppressWarnings("NullAway.Init")` on a NON-null field** when init is *deferred
  but certain before use* — framework/container injection, or a stateful object that
  sets the field behind a guard flag before reading it. Real:
  `MergedAnnotationPredicates.FirstRunOfPredicate.lastValue` (set on first `test()`
  call, guarded by `hasLastValue`). Decision rule: **legitimately absent → `@Nullable`;
  always set before read → `NullAway.Init`.**

## Inferring nullness inside generics, collections, and varargs

The `@Nullable` does not always go on the outer type — often the *element* or *value*
is what can be null. Signals seen across `spring-web`:

| Situation | Annotation | Evidence (spring-web) |
|-----------|------------|------------------------|
| A `Map`/collection whose **values** may legitimately be null (e.g. URI variables) | `@Nullable` on the value type argument | `Map<String, ? extends @Nullable Object> uriVariables` (`RestOperations.getForObject`, `RequestEntity.DefaultBodyBuilder`) |
| Varargs whose **individual elements** may be null | `@Nullable T...` | `@Nullable Object... uriVariables`, `@Nullable Object... providedArgs` |
| A **method-level** type parameter that callers may substitute with a nullable type | bound it `<T extends @Nullable Object>` | `RestClient.RequestHeadersSpec.exchange(...)`, `ExchangeFunction<T extends @Nullable Object>` |
| The whole map/collection is itself optional **and** its values are nullable | both: `@Nullable Map<String, ? extends @Nullable Object>` | `RequestEntity` `uriVarsMap` field |

Rule: ask *can the container be absent?* (outer `@Nullable`) **separately** from *can
an entry inside be null?* (inner `@Nullable` on the type argument / varargs element).
Method type variables follow the same bound rule as class type variables — add
`extends @Nullable Object` when a nullable substitution must be allowed.

## `@Contract` clause → method category

> **Where `@Contract` is worth it:** it concentrates in low-level utilities
> (`spring-core` `Assert`/`*Utils`, predicates, transforms). `spring-web` uses **no
> `@Contract` at all** — ordinary service/web code relies on plain `@Nullable` plus the
> narrow suppressions below, and builder setters just declare a non-null self-return
> without a `_ -> this` clause. Reach for `@Contract` on reusable assertion/predicate/
> transform helpers, not on every method.

`@Contract` (`org.springframework.lang.Contract`) is declarative metadata for tools
(NullAway/IDE); the implementation must match it exactly. Clause DSL:
`args -> effect`, clauses separated by `;`. **Args:** `null`, `!null`, `true`,
`false`, `_` (any). **Effects:** `fail` (throws / never returns normally), `null`,
`!null`, `true`, `false`, `new`, `this`, `paramN` (returns the Nth arg, 1-indexed), `_`.

Infer the clause from what the method does:

| Method category | Clause | Real example |
|-----------------|--------|--------------|
| Assert: throw if arg null (trailing message/supplier `_`) | `null, _ -> fail` | `Assert.notNull`, `Assert.hasText` |
| Assert: throw if arg NOT null | `!null, _ -> fail` | `Assert.isNull` |
| Assert: throw if boolean false | `false, _ -> fail` | `Assert.state`, `Assert.isTrue` |
| Always throws (rethrowers) | `_ -> fail` | `ReflectionUtils.rethrowRuntimeException` |
| Assert on a non-first arg | `_, null, _ -> fail` / `_, null, null -> fail` | `Assert.doesNotContain`, `ReflectionUtils.findField` (name or type required) |
| Predicate, null means "no/invalid" | `null -> false` | `StringUtils.hasText`, `hasLength`; `ReflectionUtils.isEqualsMethod` |
| Predicate, null means "empty/missing" | `null -> true` | `ObjectUtils.isEmpty`, `CollectionUtils.isEmpty` |
| Two-arg predicate, either null ⇒ false | `null, _ -> false; _, null -> false` | `StringUtils.startsWithIgnoreCase`, `PatternMatchUtils.simpleMatch` |
| Null-safe equals (both null ⇒ true) | `null, null -> true; null, _ -> false; _, null -> false` | `ObjectUtils.nullSafeEquals` |
| Asymmetric two-arg predicate (order matters) | `_, null -> true; null, _ -> false` | `TypeUtils.isAssignableBound` |
| Transform: may return null only when given null | `null -> null` | `ObjectUtils.unwrapOptional`, `SerializationUtils.deserialize` |
| Nullness-preserving transform (strongest) | `null -> null; !null -> !null` | `StringUtils.quote`, `getFilename`; `TypeDescriptor.forObject` |
| Filter: first arg null ⇒ null result | `null, _ -> null` / `null, _, _ -> null` | `CollectionUtils.findValueOfType` |
| Coalesce: return one of the args | `null, _ -> param2; _, null -> param1` | `ClassUtils.determineCommonAncestor`, `StringUtils.concatenateStringArrays` |

**Choosing `null -> false` vs `null -> true`:** read the method's *meaning for null*.
"Does it have content / match / qualify?" → `false`. "Is it empty / absent?" → `true`.

**Single vs both transform clauses:** use `null -> null` alone when the method *only*
returns null for null input but may legitimately return non-null otherwise; use
`null -> null; !null -> !null` only when a non-null input is *guaranteed* to yield a
non-null result (true nullness preservation).

## `@Nullable` and `@Contract` together

They state different things and are often both needed:
- `@Nullable` on the param/return = "this slot can hold null."
- `@Contract` = the input→output relationship a checker uses downstream.

Example: `public static void notNull(@Nullable Object object, String message)` with
`@Contract("null, _ -> fail")` — the param *can* be null, but after a successful call
NullAway treats `object` as non-null. A nullable-returning transform needs both the
`@Nullable` return *and* `@Contract("null -> null; !null -> !null")` so callers
passing a non-null value get a non-null result. `@Contract` alone suffices for
`void`/`boolean` methods (assertions, predicates) where there is no nullable slot to mark.

## Choosing the suppression (last resort)

| Situation | Use |
|-----------|-----|
| Value is genuinely sometimes absent | `@Nullable` (not a suppression) |
| Non-null field, init deferred but certain (injection, guarded lazy set) | `@SuppressWarnings("NullAway.Init")` |
| Checker can't follow the dataflow though code is correct | `@SuppressWarnings("NullAway") // Dataflow analysis limitation` |
| A null-guard can't be tracked across a lambda/closure boundary | `@SuppressWarnings("NullAway") // Lambda` (e.g. `CorsConfiguration.addAllowedOrigin`, `AbstractStreamingClientHttpRequest`) |
| Non-null guaranteed by a delegated helper the checker can't see into | `@SuppressWarnings("NullAway") // Not null assertion performed in <Helper#method>` (`ResourceRegionHttpMessageConverter` → `StreamUtils#copyRange`) |
| A domain invariant the checker can't know (value never null here) | `@SuppressWarnings("NullAway") // <field/value> is never null` (`InvalidMediaTypeException.getMessage`) |
| Overriding a super method that isn't annotated yet | `@SuppressWarnings("NullAway") // Null-safety of Java super method not yet managed` |
| Known checker bug | `@SuppressWarnings("NullAway") // <issue URL>` |

Always suppress narrowly with a reason comment; never widen scope to silence warnings.

## Common mistakes

| Mistake | Fix |
|---------|-----|
| Marking a "prefer empty over null" method `@Nullable` | Leave it non-null; it returns empty, not null. |
| Putting `@Nullable` on a `boolean`/primitive predicate | Express null behavior with `@Contract("null -> true/false")` instead. |
| Using `NullAway.Init` for a value that's truly optional | Use `@Nullable`; `NullAway.Init` is only for *certain-before-use* init. |
| `@Contract` clause that the implementation doesn't actually honor | The contract must match real behavior exactly, or it misleads tools. |
| Adding `@Contract` but omitting `@Nullable` on a nullable param/return | Add both — they encode different facts. |
| `null -> null; !null -> !null` on a method that can return null for non-null input | Use single-clause `null -> null`; the strong form asserts preservation. |
