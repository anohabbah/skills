---
name: jspecify-user-guide
description: Use when introducing JSpecify nullness annotations (@Nullable, @NullMarked, @NullUnmarked) to a Java project that has no nullness annotations yet, when deciding where to place nullness annotations during incremental adoption, or when annotating code that calls unannotated/legacy libraries.
---

# JSpecify Nullness Adoption Guide

## Overview

JSpecify annotations declare whether a Java type usage can be `null`. The core adoption move is `@NullMarked`: inside a null-marked scope, every unannotated type means **non-null**, and you annotate only the exceptions with `@Nullable`. This turns "null is everywhere and undocumented" into "null is opt-in and visible."

## When to Use

- Adding nullness annotations to a codebase that currently has none
- Deciding the scope at which to apply `@NullMarked`
- Deciding where `@Nullable` / `@NonNull` belong on a given signature
- Carving out not-yet-reviewed code during incremental rollout

## The Four Nullness States

| State | Meaning |
|-------|---------|
| Nullable | `@Nullable T` — may be `null` |
| Non-null | plain `T` inside `@NullMarked` — never `null` |
| Parametric | plain type variable — nullness comes from the type argument |
| Unspecified | plain `T` outside any `@NullMarked` scope — legacy, unknown |

## Adoption Rules

1. **Apply `@NullMarked` at the widest scope you can.** Prefer module → package (`package-info.java`) → class. One annotation flips the default for everything inside it.
2. **Inside `@NullMarked`, add `@Nullable` only where `null` is genuinely possible** — nullable parameters, return values, and fields. Leave everything else unannotated; it is already non-null.
3. **You almost never need `@NonNull`.** It only matters to override a nullable default — e.g. inside an `@NullUnmarked` region. Don't sprinkle it through `@NullMarked` code.
4. **Adopt incrementally with `@NullUnmarked`.** Mark the whole project or package `@NullMarked`, then carve out *not-yet-reviewed* classes or methods with `@NullUnmarked` (not code that merely calls legacy APIs — see Calling Unannotated Dependencies). Re-nest `@NullMarked` to opt sub-regions back in.
5. **Don't annotate local variable types.** Tools infer local nullness from assignments. Annotate the API surface (parameters, returns, fields), not locals.

## Placement & Syntax Rules

JSpecify annotations are **type-use** annotations — they attach to the type, not the declaration. Position changes meaning:

| Construct | Correct form | Meaning |
|-----------|-------------|---------|
| Nested type | `Map.@Nullable Entry` | the `Entry` may be null, not the `Map` |
| Array elements | `@Nullable String[]` | elements may be null |
| Array itself | `String @Nullable []` | the array reference may be null |
| Both | `@Nullable String @Nullable []` | both may be null |
| Wildcard | `List<? extends @Nullable Number>` | allows nullable elements |

## Generics Rules

The bound controls what **callers may pass**. A `@Nullable` on a *use* of the type variable is independent of the bound.

- A type parameter with **no `@Nullable` bound rejects nullable type arguments**: `class Box<E>` makes `Box<@Nullable String>` illegal. Keep the plain bound by default.
- Add an explicit nullable bound (`<E extends @Nullable Object>`) **only when callers genuinely need to pass a nullable type argument** — e.g. a container that deliberately stores `null`. Do not widen the bound just because a method returns `null`.
- **Returning `@Nullable T` does not require a nullable bound.** `firstOrNull(List<T>)` keeps the plain `<T>` and returns `@Nullable T` — it can return `null` even though every list element is non-null. This is the common case.
- Plain `E` in a signature means "nullness follows the type argument." `@Nullable E` means "may be null regardless of the type argument."

## Calling Unannotated Dependencies

Code from libraries without JSpecify annotations has **unspecified** nullness — not `@Nullable`, not non-null. When your `@NullMarked` method delegates to it:

- Decide your own method's contract from what you actually know. If you can't guarantee the dependency won't hand you `null`, annotate *your* return `@Nullable` so your callers null-check.
- **Do not reach for `@NullUnmarked` here.** `@NullUnmarked` is for *your own code you haven't reviewed yet* — it removes the spec entirely. A reviewed method that happens to call a legacy API still gets a real, specified contract.

## Example

```java
// package-info.java — widest practical scope
@NullMarked
package com.example.strings;

import org.jspecify.annotations.NullMarked;
```

```java
package com.example.strings;

import org.jspecify.annotations.Nullable;

// @NullMarked is inherited from the package — plain types are non-null here
class Strings {
  // x defaults to non-null; only the return needs @Nullable
  static @Nullable String emptyToNull(String x) {
    return x.isEmpty() ? null : x;
  }

  // parameter is the exception, so it gets @Nullable; return is non-null by default
  static String nullToEmpty(@Nullable String x) {
    return x == null ? "" : x;
  }
}
```

## Common Mistakes

- **Annotating types before `@NullMarked` is in effect.** Outside a null-marked scope, plain types are "unspecified," so lone `@Nullable` markers carry only half the meaning. Establish `@NullMarked` first.
- **Using `@NonNull` everywhere.** Redundant inside `@NullMarked` — it signals you haven't actually flipped the default.
- **Forgetting `package-info.java`.** It is the cleanest place for package-level `@NullMarked`; class-by-class annotation is more work and easier to miss.
- **Widening a generic bound because a method returns null.** The bound governs caller-supplied type arguments, not return values. `firstOrNull` returns `@Nullable T` with a plain `<T>`. Only add `extends @Nullable Object` when callers must be able to pass nullable type arguments.
- **Using `@NullUnmarked` for code that calls legacy APIs.** It is for unreviewed code, not for reviewed code touching unannotated dependencies. See Calling Unannotated Dependencies.
- **Annotating local variables.** Wasted effort — nullness there is inferred.
