---
name: jspecify-user-guide
description: Use when introducing JSpecify nullness annotations to a Java project that is not yet annotated, or when deciding where @Nullable / @NonNull / @NullMarked / @NullUnmarked belong, how to place them on generics, arrays, and nested types, or where null is allowed in a signature.
license: MIT
metadata:
  author: jspecify-user-guide
  version: "1.0"
  source: https://jspecify.dev/docs/user-guide/
---

# Introducing JSpecify Nullness Annotations

## Overview

JSpecify annotations declare whether a type reference can hold `null`. They are
**type-use** annotations (`org.jspecify.annotations.*`) and require the `org.jspecify`
artifact on the classpath. Core principle: `@NonNull String` is a *subtype* of
`@Nullable String` — a non-null value is assignable to a nullable slot, not vice versa.

## The four nullness states

| State | Meaning |
|-------|---------|
| Nullable | The value can be `null`; callers must handle it. |
| Non-nullable | The value will not be `null`; callers may assume so. |
| Parametric | On a type variable `E` — nullable only if the type argument is. |
| Unspecified | Unknown. The default before annotation; avoid leaving code here. |

## The annotations

| Annotation | Where | Effect |
|------------|-------|--------|
| `@Nullable` | A specific type usage | That usage may be `null`. |
| `@NonNull` | A specific type usage | That usage is never `null` (rarely needed inside `@NullMarked`). |
| `@NullMarked` | module / package / class / method | Unannotated types in scope mean `@NonNull` by default. |
| `@NullUnmarked` | package / class / method already inside `@NullMarked` | Reverts to unspecified — the incremental-adoption escape hatch. |

## Adoption strategy (un-annotated project)

1. **Turn on `@NullMarked` broadly.** Apply it per package via `package-info.java`
   (or per class). Inside the scope, a bare `String` *means* `@NonNull String`, so
   you only annotate the exceptions.
2. **Add `@Nullable` only where `null` is genuinely possible** — return types,
   parameters, and fields that can hold `null`. Everything else stays bare.
3. **Defer hard areas with `@NullUnmarked`** instead of annotating everything at
   once. Adopt package by package; unmarked code reads as unspecified, not wrong.

```java
package-info.java:
@NullMarked
package com.example.text;
import org.jspecify.annotations.NullMarked;
```

```java
@NullMarked
class Strings {
  // param is @NonNull by default; only the nullable return is annotated
  static @Nullable String emptyToNull(String x) {
    return x.isEmpty() ? null : x;
  }
}
```

## Placement & syntax rules

- **Do not annotate local-variable root types.** Nullness is inferred from the
  assigned value. Annotate params, returns, fields, and type arguments — not locals.
- **Arrays** — the position decides what is nullable:
  - `@Nullable String[]` → elements may be null.
  - `String @Nullable []` → the array reference may be null.
  - `@Nullable String @Nullable []` → both.
- **Nested types** — annotate the part that is nullable: `Map.@Nullable Entry`.

## Generics rules

- In `@NullMarked`, a bare `<E>` means `<E extends @NonNull Object>`. To allow
  nullable type arguments, give an explicit nullable bound:
  `<E extends @Nullable Object>`. Then both `List<String>` and `List<@Nullable String>`
  are legal.
- A bare `E` in a signature follows the type argument's nullness; `@Nullable E`
  can be null even when `E` cannot:

```java
@NullMarked
interface List<E extends @Nullable Object> {
  boolean add(E element);
  E get(int index);          // nullness follows E
  @Nullable E getFirst();    // may be null even for List<String>
}
```

- A wildcard `<?>` inherits the bound of its type variable: for
  `List<E extends @Nullable Object>`, `List<?>` permits nullable elements; for
  `Box<E>` (i.e. `extends @NonNull Object`) it does not.

## Common mistakes

| Mistake | Fix |
|---------|-----|
| Annotating local variables | Leave locals bare; nullness is inferred. |
| `@Nullable` on a non-null thing inside `@NullMarked` | Bare already means `@NonNull`; only mark real nullables. |
| Sprinkling `@NonNull` everywhere | Inside `@NullMarked` it is the default — usually redundant. |
| Type variable can take null but bound is bare | Use `<E extends @Nullable Object>`. |
| Wrong array/nested placement | `@Nullable String[]` (elements) vs `String @Nullable []` (array); `Map.@Nullable Entry`. |
| Leaving large areas unannotated silently | Mark them `@NullUnmarked` so "unspecified" is explicit and intentional. |
