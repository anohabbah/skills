# Design: jspecify-spring-null-safety skill

## Goal

Create an agent skill, `jspecify-spring-null-safety`, that defines the
Spring-specific rules for introducing JSpecify and `@Contract` annotations to a
Spring Framework project. It complements the existing `jspecify-user-guide`
skill and deliberately does **not** repeat its content.

Source: https://docs.spring.io/spring-framework/reference/core/null-safety.html

## Scope

### In scope (rules not in jspecify-user-guide)

1. **`@Contract` on assertion/precondition methods** — annotating nullness-validating
   helpers so NullAway treats arguments as non-null after a successful call.
2. **Migrating off deprecated Spring annotations** — `org.springframework.lang.@Nullable`,
   `@NonNull`, `@NonNullApi`, `@NonNullFields` → JSpecify equivalents, including
   repositioning and the array-nullness migration gotcha.
3. **Overridden methods do not inherit nullness** — JSpecify annotations must be
   copied onto overriding methods.
4. **Kotlin interop** — how `@NullMarked` / `@Nullable` Java maps to Kotlin
   null-safety, and that gaps become hard Kotlin compile errors.

### Out of scope

- General JSpecify adoption mechanics (`@NullMarked` scoping, where `@Nullable`
  goes, type-use placement, generics, calling unannotated deps) — already in
  `jspecify-user-guide`, cross-referenced as REQUIRED BACKGROUND.
- NullAway build/tooling setup (`OnlyNullMarked`, `CustomContractAnnotations`,
  `JSpecifyMode`) and `@SuppressWarnings` patterns — explicitly excluded by the user.
- Runtime `org.springframework.core.Nullness` API.

## Skill type

Spring-specific technique skill. Self-contained `SKILL.md`, no supporting files
(content fits comfortably inline). Cross-references `jspecify-user-guide` by
name as REQUIRED BACKGROUND; does not `@`-link it.

## Structure

**Frontmatter `description`** (triggers only, no workflow summary):
> Use when introducing JSpecify nullness annotations to a Spring Framework
> project: annotating assertion/precondition methods with @Contract, migrating
> off deprecated org.springframework.lang annotations (@NonNullApi,
> @NonNullFields), handling overridden methods, or Kotlin interop.

**Sections:**

1. **Overview** — Spring 7 exposes APIs via JSpecify; this skill adds the
   Spring-specific rules on top of the general adoption rules in
   `jspecify-user-guide`.
2. **When to Use** — symptom bullets; when NOT to use (general adoption → user guide).
3. **@Contract on assertion/precondition methods** — core rule. `Assert.notNull(x)`-style
   helpers don't change a type, but after a successful call the argument is known
   non-null; JSpecify alone can't express that, NullAway needs
   `org.springframework.lang.@Contract`. Rule: methods whose job is to
   validate/guarantee nullness get `@Contract`; ordinary methods don't. Syntax
   cookbook: `@Contract("null -> fail")`, `@Contract("null, _ -> fail")`,
   `@Contract("!null -> !null")`. Payoff: callers keep a `@Nullable` field
   non-null after the assert instead of adding redundant checks or suppressions.
4. **Migrating off deprecated Spring annotations** — mapping table:
   `org.springframework.lang.@Nullable` → `org.jspecify.annotations.@Nullable`;
   `@NonNull` → usually delete; `@NonNullApi` → `@NullMarked`; `@NonNullFields` →
   delete (covered by `@NullMarked`). Gotchas: reposition declaration → type-use
   position; old `@Nullable Object[]` must become `Object @Nullable []` to
   preserve "nullable array" meaning.
5. **Overridden methods don't inherit nullness** — copy `@Nullable` onto the
   override; omitting it silently re-declares param/return as non-null.
6. **Kotlin interop** — `@NullMarked` Java → Kotlin non-null `String`; `@Nullable`
   → `String?`. Missing `@Nullable` becomes a hard Kotlin compile error, not a
   warning. Get the API surface right before Kotlin depends on it.
7. **Common Mistakes** — table: un-`@Contract`-ed assertion helpers; keeping
   `@NonNullApi`/`@NonNullFields` next to `@NullMarked`; mechanical array
   migration; forgetting to copy `@Nullable` onto overrides; `@Contract` noise on
   methods with no nullness contract.

**Example:** one Java snippet — a `@Contract`-annotated assertion helper plus a
caller that benefits from it.

## File location

`skills/jspecify-spring-null-safety/SKILL.md` (sibling of `jspecify-user-guide`).

## Verification

Per writing-skills TDD process: baseline-test with subagents (retrieval scenarios
on Spring null-safety questions) WITHOUT the skill to document gaps, then write
the skill, then re-test to confirm agents apply the rules correctly.
