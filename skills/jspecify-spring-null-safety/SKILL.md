---
name: jspecify-spring-null-safety
description: Use when introducing JSpecify nullness annotations to a Spring Framework project — annotating assertion/precondition methods with @Contract so NullAway trusts them, migrating off deprecated org.springframework.lang annotations (@NonNullApi, @NonNullFields, @Nullable), and Spring-specific overriding/Kotlin concerns. Spring-specific; complements jspecify-user-guide.
---

# JSpecify Null-Safety for Spring Projects

## Overview

Spring Framework 7 exposes its APIs through JSpecify annotations. This skill covers the **Spring-specific** rules for adopting JSpecify in a Spring project. It assumes the general adoption mechanics — `@NullMarked` scoping, where `@Nullable` belongs, type-use placement, generics — are already handled.

**REQUIRED BACKGROUND:** Use jspecify-user-guide for the general adoption rules. This skill only adds what is Spring-specific.

## When to Use

- Annotating assertion / precondition helpers so a nullness checker trusts them
- Migrating a Spring project off the deprecated `org.springframework.lang` nullness annotations
- Overriding annotated Spring methods, or exposing a `@NullMarked` Java API to Kotlin

When NOT to use: for general "where do I put `@Nullable`" questions, use jspecify-user-guide instead.

## @Contract on Assertion / Precondition Methods

An assertion helper like `Assert.notNull(x)` does not change `x`'s type, but after it returns successfully `x` is known non-null. JSpecify annotations alone cannot express this — a nullness checker still treats `x` as `@Nullable`. Spring ships **`org.springframework.lang.@Contract`** for exactly this.

**Use Spring's `@Contract`, not JetBrains'.** `org.jetbrains.annotations.Contract` has the same string syntax and NullAway recognizes it too, but a Spring project should use `org.springframework.lang.Contract` — no extra dependency, and it is the annotation Spring's own codebase and docs use.

**Rule:** any method whose purpose is to validate or guarantee nullness gets `@Contract`. Ordinary methods do not — `@Contract` on a method with no real nullness contract is noise.

| Contract | Meaning |
|----------|---------|
| `@Contract("null -> fail")` | throws if the (single) argument is null |
| `@Contract("null, _ -> fail")` | throws if the first argument is null |
| `@Contract("!null -> !null")` | returns non-null when given non-null, null when given null |
| `@Contract("_ -> param1")` | returns its first argument unchanged |

```java
import org.jspecify.annotations.Nullable;
import org.springframework.lang.Contract;

@Contract("null -> fail")
public static void notNull(@Nullable Object value) {
    if (value == null) {
        throw new IllegalArgumentException("must not be null");
    }
}

void use(@Nullable String name) {
    notNull(name);
    name.trim();   // checker now treats name as non-null — no redundant check
}
```

Without `@Contract`, callers are forced into a redundant `if (name != null)` or a `@SuppressWarnings` — both hide the real contract.

## Migrating Off Deprecated Spring Annotations

The `org.springframework.lang` nullness annotations are deprecated as of Spring Framework 7. Replace them:

| Deprecated | Replacement |
|------------|-------------|
| `org.springframework.lang.@Nullable` | `org.jspecify.annotations.@Nullable` |
| `org.springframework.lang.@NonNull` | delete it — redundant inside `@NullMarked` |
| `@NonNullApi` (in `package-info.java`) | `@NullMarked` |
| `@NonNullFields` (in `package-info.java`) | delete it — `@NullMarked` already covers fields |

Two migration gotchas:

1. **Reposition to type-use.** The old Spring `@Nullable` sat in *declaration* position (`@Nullable private String f;`). JSpecify `@Nullable` is a type-use annotation and goes immediately before the type (`private @Nullable String f;`). Same for parameters and return types.
2. **Do not broaden array nullness.** Old Spring `@Nullable private Object[] buffer` meant exactly one thing: *the array reference* may be null. Migrate it to `private Object @Nullable [] buffer` — and nothing more. Do **not** "upgrade" it to `@Nullable Object @Nullable []`: that also marks the *elements* nullable, a contract the original code never had. Migrate the meaning that existed, not a meaning you imagine.

## Other Spring-Specific Notes

**Overriding annotated methods.** JSpecify annotations are not inherited from the overridden method. When you override an annotated Spring method, repeat its `@Nullable` annotations on your override (e.g. `@Override public @Nullable String getHeader(String name)`).

**Kotlin interop.** A correctly annotated `@NullMarked` Java API maps to Kotlin null-safety automatically: plain Java types become Kotlin non-null (`String`), `@Nullable` types become Kotlin nullable (`String?`). A *missing* `@Nullable` on a Java return is not a warning — Kotlin trusts the type as non-null, so the `null` surfaces as a runtime `NullPointerException` at the Kotlin call site. Get the Java API surface right before Kotlin depends on it.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using `org.jetbrains.annotations.Contract` in a Spring project | Use `org.springframework.lang.Contract` — no extra dependency |
| Assertion helper left un-`@Contract`-ed; callers add redundant null checks or `@SuppressWarnings` | Add `@Contract` to the helper |
| Keeping `@NonNullApi` / `@NonNullFields` alongside `@NullMarked` | Delete both — `@NullMarked` replaces both |
| Migrating `@Nullable Object[]` to `@Nullable Object @Nullable []` | Use `Object @Nullable []` — don't broaden element nullness |
| `@Contract` on methods with no nullness contract | Remove it — only nullness-validating methods need it |
