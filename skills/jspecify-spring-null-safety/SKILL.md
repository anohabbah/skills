---
name: jspecify-spring-null-safety
description: Use when introducing JSpecify and Spring's @Contract annotations to a Spring Framework 6.2+/Boot 3.4+ (ideally Spring 7 / Boot 4) project not yet annotated — covers Spring-specific rules layered on top of plain JSpecify, migrating off the deprecated org.springframework.lang nullness annotations or JSR-305 jsr305, expressing nullness post-conditions with @Contract, and the NullAway/IDE setup that consumes them.
license: MIT
metadata:
  author: jspecify-spring-null-safety
  version: "1.0"
  source: https://docs.spring.io/spring-framework/reference/core/null-safety.html
---

# Spring Null-Safety: JSpecify + @Contract

**Companion skill — read `jspecify-userguide` first.** That skill covers the core
JSpecify rules (the four nullness states, `@Nullable`/`@NonNull`/`@NullMarked`/
`@NullUnmarked`, type-use placement, arrays, nested types, generics, locals). This
skill adds only the **Spring-specific** rules and the `@Contract` annotation. It
does not repeat the core rules.

## Spring context

Spring Framework 7 declares nullability with **JSpecify** (`org.jspecify.annotations.*`).
The goal is to catch `NullPointerException` at build time via explicit nullability.
Spring expects ecosystem libraries (Reactor, Micrometer, Spring portfolio) to do the same.

## Migrate off the old nullness annotations

Spring deprecated `org.springframework.lang.@Nullable`, `@NonNull`, `@NonNullApi`,
and `@NonNullFields` (JSR-305 semantics). This project also carries the legacy
`com.google.code.findbugs:jsr305` dependency. When annotating:

- Use `org.jspecify.annotations.Nullable`, **not** the Spring `org.springframework.lang`
  or `javax.annotation` (JSR-305) variants.
- The key difference: old annotations target fields/params/returns as a whole;
  JSpecify annotations target **type usage**, so they migrate position too.

```java
// Old (deprecated)              // New (JSpecify, type-use position)
@Nullable private String field;  private @Nullable String field;
@Nullable public String get() {} public @Nullable String get() {}
```

## Spring style convention

Place type-use annotations on the **same line, immediately preceding the type**
(not on their own line as old field-level annotations were written):

```java
private @Nullable String fileEncoding;

public @Nullable String buildMessage(@Nullable String message,
                                     @Nullable Throwable cause) { ... }
```

## @Contract — express nullness post-conditions

`org.springframework.lang.Contract` declares complementary semantics that JSpecify
alone cannot express, so analysis tools stop emitting false warnings. Use it on
methods whose effect on nullness depends on their arguments — assertions, guards,
and nullness-preserving transforms.

```java
import org.springframework.lang.Contract;

// After a successful call, the argument is known non-null.
@Contract("null, _ -> fail")
public static void notNull(@Nullable Object object, String message) { ... }

// Guard that returns whether the value is present.
@Contract("null -> false")
public static boolean hasText(@Nullable String str) { ... }

// Nullness-preserving transform: null in -> null out, non-null in -> non-null out.
@Contract("null -> null; !null -> !null")
public static @Nullable String trimWhitespace(@Nullable String str) { ... }
```

Contract clause syntax: `args -> effect`, clauses separated by `;`.
- **Args**: `null`, `!null`, `true`, `false`, `_` (any), positional and comma-separated.
- **Effect**: `fail` (throws), `null`, `!null`, `true`, `false`, `new`, `this`, `_`.

Without `@Contract`, a tool cannot know that code after `Assert.notNull(x, ...)` may
treat `x` as non-null.

## Tooling that consumes these annotations

`@Contract` is only meaningful to a checker. Spring's recommended NullAway config:

```properties
NullAway:OnlyNullMarked=true                                  # check only @NullMarked packages
NullAway:CustomContractAnnotations=org.springframework.lang.Contract
NullAway:JSpecifyMode=true                                    # optional, needs JDK 22+
```

IDE support: IntelliJ IDEA has full JSpecify support; Eclipse needs manual config.
In Kotlin, JSpecify annotations are auto-translated to Kotlin null safety.

## Legitimate `@SuppressWarnings` cases

When NullAway cannot prove safety but the code is correct, suppress narrowly with a
reason comment — do not weaken the annotations themselves:

```java
@SuppressWarnings("NullAway.Init")  // field initialized lazily / by container
@SuppressWarnings("NullAway")       // dataflow limitation, lambda, reflection, well-known map key
```

## Runtime check

`org.springframework.core.Nullness` detects the nullness of a type usage, field,
return type, or parameter at runtime — supports JSpecify, Kotlin null safety, and a
pragmatic check against any `@Nullable` annotation regardless of package.

## Common mistakes (Spring-specific)

| Mistake | Fix |
|---------|-----|
| Importing `org.springframework.lang.Nullable` or JSR-305 `@Nullable` | Import `org.jspecify.annotations.Nullable`. |
| Keeping `@Nullable` on its own line (old field style) | Put it inline, immediately before the type. |
| Assertion/guard methods cause false NPE warnings downstream | Add `@Contract` so the checker learns the post-condition. |
| Suppressing warnings broadly to silence NullAway | Suppress narrowly (`NullAway.Init` / `NullAway`) with a reason; fix the annotation if it's actually wrong. |
| Expecting `@Contract` to do anything on its own | It is metadata for tools (NullAway must list it via `CustomContractAnnotations`). |
