---
name: jspecify-spring-framework-patterns
description: Use when a Spring `@Nullable`-annotated method's callers still hit redundant null-check warnings, when annotating `isEmpty`/`hasText`/`getFilename`-style utilities, or when deciding whether `@Contract("null -> true" | "null -> false" | "null -> null; !null -> !null" | "null, _ -> param2")` belongs on a Spring method. Keywords: boolean nullness-gate, null-preserving transform, default-substituting, argument-returning fallback. Complements jspecify-user-guide and jspecify-spring-null-safety.
---

# Inferring `@Contract` from Spring Method Shapes

## Overview

`@Nullable` describes the *types* on a signature; it does not describe how the
return depends on which arguments were null. Without `@Contract`, a caller's
`if (!ObjectUtils.isEmpty(value))` cannot narrow `value`, and a caller of
`getFilename(path)` cannot rely on a non-null path producing a non-null result.
This skill is the recognition layer for the four `@Contract` shapes that recur
in `spring-core`-style utilities.

**REQUIRED BACKGROUND:** jspecify-user-guide for placement; jspecify-spring-null-safety
for `@Contract` string syntax, the Spring vs JetBrains choice, and assertion methods.

## When to Use

- Deciding whether an already-`@Nullable`-annotated Spring method also needs `@Contract`
- A checker forces redundant null checks at call sites of a `@Nullable`-annotated API
- The return depends conditionally on which arguments were null

When NOT to use: for *where* `@Nullable` belongs (placement, arrays, generics)
→ jspecify-user-guide. For `@Contract` *string syntax* and assertion methods
(`Assert.notNull`-style) → jspecify-spring-null-safety.

## The Four Shapes

`@Contract` is shape-driven: read the method body and match it to one of these.
For each match, callers can narrow at the call site — that narrowing is the
payoff and the reason the contract is worth writing.

### 1. Boolean nullness-gate — the most-missed shape

A boolean method whose result is determined by whether an argument is null:

```java
@Contract("null -> true")
public static boolean isEmpty(@Nullable Object obj) {        // true when null
    if (obj == null) {
        return true;
    }
    // ...
    return false;
}

@Contract("null -> false")
public static boolean hasText(@Nullable String str) {        // false when null
    if (str == null || str.isEmpty()) {
        return false;
    }
    // ...
    return true;
}
```

**Body signal:** the first branch is `if (arg == null) return <constant>;`.
**Payoff:** the caller can narrow in the opposite branch.
```java
if (!ObjectUtils.isEmpty(value)) {
    value.toString();   // checker now treats value as non-null
}
```

Without the contract, the JSpecify `@Nullable` alone leaves the caller writing
a redundant `if (value != null)` or a `@SuppressWarnings`.

### 2. Null-preserving transform — propagates non-null-ness

A method whose return tracks its input's nullness one-to-one:

```java
@Contract("null -> null; !null -> !null")
public static @Nullable String getFilename(@Nullable String path) {
    if (path == null) {
        return null;
    }
    int sep = path.lastIndexOf('/');
    return (sep != -1 ? path.substring(sep + 1) : path);
}
```

**Body signal:** `if (arg == null) return null;` followed by a path that
**always** returns non-null. The `!null -> !null` half is what callers usually
need — a non-null input proves a non-null output.

**Use only `null -> null`** (omit the `!null -> !null` half) when the
non-null path can itself produce null:

```java
@Contract("null -> null")
public static @Nullable Object unwrapOptional(@Nullable Object obj) {
    if (obj instanceof Optional<?> o) {
        return o.orElse(null);   // can return null even though obj was non-null
    }
    return obj;
}
```

### 3. Default-substituting — picks the first non-null

```java
@Contract("!null, _ -> param1")
public static <T> T firstNonNull(@Nullable T primary, T fallback) {
    return primary != null ? primary : fallback;
}
```

**Body signal:** `arg != null ? arg : other` returning one of its arguments
unchanged. Callers receive whichever argument was selected — `_ -> param1`,
`null, _ -> param2`, and combinations encode this.

### 4. Argument-returning fallback — passes one argument through when another is null

```java
@Contract("null, _ -> param2; _, null -> param1")
public static String @Nullable [] concatenateStringArrays(
        String @Nullable [] array1, String @Nullable [] array2) {
    if (array1 == null) {
        return array2;
    }
    if (array2 == null) {
        return array1;
    }
    // ... merge into a result; can only be null when both inputs were null
}
```

**Body signal:** chained `if (argN == null) return otherArg;` guards. The
contract names *which* argument is returned, so a caller passing a known-non-null
`array1` knows the result is non-null when `array2` is non-null too.

Throws-on-null methods (`Assert.notNull(x)` and peers) earn
`@Contract("null, _ -> fail")` — fully covered by jspecify-spring-null-safety.

## Recognition Procedure

For a method already carrying `@Nullable` annotations:

1. Throws when an arg is null → assertion-method rule (jspecify-spring-null-safety).
2. Boolean return, first branch is `if (arg == null) return <true|false>;` → shape 1.
3. `if (arg == null) return null;` then a path that always returns non-null → shape 2 (with `!null -> !null`).
4. `if (arg == null) return null;` then a path that may itself return null → shape 2 (without `!null -> !null`).
5. `arg != null ? arg : other`, or `if (argN == null) return otherArg;` → shape 3 or 4.
6. None of the above → no `@Contract`. Noise on methods with no null-dependent contract.

## Quick Reference: `@Nullable` Signals

Reliable signals agents already apply correctly — checklist, not teaching:

| Position | Signal that means `@Nullable` |
|----------|-------------------------------|
| Return | `return null` on any path; `@return` Javadoc says "or null if …"; returns a `@Nullable`-yielding call (`map.get(k)`, `optional.orElse(null)`) |
| Parameter | Body tolerates null (`if (p != null)`, `p == null ? … : p`); `@param` Javadoc says "may be null" |
| Field | Lazy (no initializer, set inside `if (f == null)` or `init()`); set only by an optional setter; cached/memoized |

Counter-signal: an `Assert.notNull(arg)` at the top *enforces* a non-null
contract — the parameter stays plain, not `@Nullable`.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| `@Nullable` added, `@Contract` not — caller has to write a redundant null check | Match the body to one of the four shapes; add the matching `@Contract` |
| Boolean nullness-gate left without `@Contract` — caller cannot narrow | Add `@Contract("null -> true")` or `"null -> false"` |
| Using `@Contract("null -> null; !null -> !null")` on a method whose non-null path can return null | Drop the `!null -> !null` half; the contract lies otherwise |
| `@Contract` on a method with no null-dependent contract | Remove it — noise, and a maintenance hazard |
