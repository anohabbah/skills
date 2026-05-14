# Design: jspecify-spring-framework-patterns skill

## Goal

Create an agent skill, `jspecify-spring-framework-patterns`, that teaches how to
**infer** JSpecify nullness annotations (`@Nullable`) and `@Contract` from a real
unannotated implementation or its Javadoc — the recognition layer that the two
existing JSpecify skills do not cover.

Source of the patterns: detailed analysis of the Spring Framework `spring-core`
module (https://github.com/spring-projects/spring-framework) — `Assert`,
`ObjectUtils`, `StringUtils`, `CollectionUtils`, `ClassUtils`, and peers.

## Boundary with the existing skills

Three sibling skills, no duplication:

- `jspecify-user-guide` — the general *rules*: the four nullness states,
  `@NullMarked` scoping, where `@Nullable` belongs, type-use placement, generics,
  calling unannotated deps.
- `jspecify-spring-null-safety` — the Spring-specific *syntax and migration*:
  `@Contract` cookbook, `org.springframework.lang.@Contract` vs JetBrains',
  migrating off deprecated Spring annotations, override non-inheritance, Kotlin.
- `jspecify-spring-framework-patterns` (this skill) — the *inference* layer: how
  to recognize, from what the code and Javadoc actually do, that a specific
  return / parameter / field should be `@Nullable`, or that a method should carry
  `@Contract`.

Both existing skills are cross-referenced as **REQUIRED BACKGROUND**. This skill
does **not** re-explain placement rules or `@Contract` syntax — it points to them.

## Scope

### In scope (the inference signals — not in the other skills)

1. A **catalog of inference signals**, organized by annotation target
   (returns, parameters, fields, `@Contract`). Each signal is concrete: a piece
   of evidence in the implementation or Javadoc that maps to a specific
   annotation conclusion, plus the counter-signal that means "leave it plain".
2. The **precedence rule** for when signals disagree:
   code-structure > Javadoc > naming idiom. The implementation is ground truth.
3. A **short class-review procedure** that walks the targets in order and applies
   the catalog.

### Out of scope

- General adoption mechanics (`@NullMarked` scoping, placement syntax, generics,
  unannotated deps) — `jspecify-user-guide`, cross-referenced.
- `@Contract` string syntax, the Spring-vs-JetBrains choice, legacy migration,
  override non-inheritance, Kotlin interop — `jspecify-spring-null-safety`,
  cross-referenced.
- NullAway / build tooling setup, `@SuppressWarnings` patterns.
- Inline citation of specific `spring-core` classes (see "Grounding").

## Skill type

Pattern skill — mental models for recognizing where annotations belong.
Self-contained `SKILL.md`, no supporting files. Cross-references
`jspecify-user-guide` and `jspecify-spring-null-safety` by name as REQUIRED
BACKGROUND; does not `@`-link them.

## Grounding

Each signal is illustrated with one tight **synthetic** snippet that distills the
pattern. `spring-core` is the research source but is **not** cited inline — this
keeps the skill portable and avoids tying it to a source version. The synthetic
examples are distilled from a real read of `spring-core` during implementation.

## Structure

**Frontmatter `description`** (triggers only, no workflow summary):

> Use when deciding whether a specific return, parameter, or field should be
> @Nullable — or whether a method should carry @Contract — by reading its
> implementation or Javadoc rather than from placement rules alone. Spring
> Framework / JSpecify; complements jspecify-user-guide and
> jspecify-spring-null-safety.

**Sections:**

1. **Overview** — the inference idea: annotations are *derived* from what the
   code does, not guessed. States the **precedence rule** — when signals
   disagree, code-structure beats Javadoc beats naming idiom; the implementation
   is ground truth, stale Javadoc and conventional names lose to the body.

2. **When to Use** — symptom bullets (annotating an existing unannotated class,
   reviewing whether an annotation is correct, unsure if a return is nullable).
   When NOT to use: for the placement *rules* → `jspecify-user-guide`; for
   `@Contract` *syntax* → `jspecify-spring-null-safety`.

3. **Inferring `@Nullable` returns** — signals:
   - `return null;` literal on any path (strongest, direct evidence)
   - `@return` Javadoc mentions `null` ("or `null` if not found", "may be `null`")
   - directly returns a known-`@Nullable` call (`return map.get(key);`)
   - returns `optional.orElse(null)`
   - naming idiom: `find*`, `*IfAvailable`, `*OrNull`, `resolve*` (weak — body wins)
   - counter-signal: every path returns a fresh object / literal / `this` /
     non-null param → leave plain

4. **Inferring `@Nullable` parameters** — signals:
   - body *tolerates* null: `if (p != null)`, `p == null ? fallback : p`
   - param passed onward to a `@Nullable`-accepting method with no prior check
   - Javadoc: "`null` to use the default"
   - overload family: `doX(s)` delegates `doX(s, null)` → delegated param is `@Nullable`
   - counter-signal: param is immediately dereferenced or `Assert.notNull`'d →
     **non-null** (the check enforces a non-null contract, it does not tolerate null)

5. **Inferring `@Nullable` fields** — signals:
   - lazy-initialized field (no initializer; set inside an `if (f == null)`
     getter or an `init()` method)
   - field assigned only by an optional setter / configured after construction
   - cached / memoized field
   - counter-signal: `final` field assigned non-null in every constructor → leave plain

6. **Inferring `@Contract`** — recognizing method *shapes* (for the string
   syntax, points to `jspecify-spring-null-safety`):
   - passthrough / identity: returns its argument unchanged → `_ -> param1`
   - null-preserving transform: `x == null ? null : f(x)` → `null -> null`
     (plus `!null -> !null` when `f` never returns null)
   - **boolean nullness-gate**: `isEmpty(x)` true-when-null → `null -> true`;
     `hasText(x)` false-when-null → `null -> false` (neither existing skill
     shows the `null -> true` / `null -> false` forms)
   - default-substituting: `x != null ? x : dflt` → `!null, _ -> param1`
   - throws-on-null → one-line pointer to `jspecify-spring-null-safety`
     (already covered there as the assertion-method rule)

7. **Reviewing a class — short procedure** — ordered walk: confirm `@NullMarked`
   is in effect → fields → returns → parameters → `@Contract` shapes → apply the
   precedence rule on any conflict.

8. **Common Mistakes** — table:
   - trusting the method name over the body (`getX` assumed non-null though the
     body has `return null`)
   - trusting stale Javadoc over the implementation
   - marking an `Assert.notNull`'d parameter `@Nullable` (the check is a non-null
     contract, not null tolerance)
   - missing `@Contract` on boolean nullness-gate methods — callers then can't narrow
   - treating a lazy field as non-null because "it's always set before use" —
     it is null between construction and first init; `@Nullable` is correct

**Examples:** one tight synthetic snippet per signal, inline in its section.

## File location

`skills/jspecify-spring-framework-patterns/SKILL.md` (sibling of
`jspecify-user-guide` and `jspecify-spring-null-safety`).

## Verification

Per the writing-skills TDD process:

1. **`spring-core` analysis** — before writing, read representative `spring-core`
   files (`Assert`, `ObjectUtils`, `StringUtils`, `CollectionUtils`, `ClassUtils`)
   to validate and refine the signal list; the synthetic examples are distilled
   from this pass.
2. **RED** — baseline-test with subagents: give them `spring-core`-style
   unannotated code and ask them to add JSpecify annotations, WITHOUT the skill.
   Expected gaps: missed boolean `@Contract` (`null -> true/false`), trusting
   method names over the body, missed lazy `@Nullable` fields. Record verbatim.
3. **GREEN** — write `SKILL.md` addressing the documented gaps; re-test the same
   scenarios with the skill; confirm agents now apply the signals.
4. **REFACTOR** — close any remaining gaps; re-test until the scenarios pass.
