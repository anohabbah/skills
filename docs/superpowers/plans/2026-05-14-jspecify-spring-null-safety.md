# jspecify-spring-null-safety Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create the `jspecify-spring-null-safety` agent skill — the Spring-specific rules for introducing JSpecify and `@Contract` annotations, complementing the existing `jspecify-user-guide` skill.

**Architecture:** Skill creation follows the writing-skills TDD cycle: RED (baseline-test the gaps with subagents *before* the skill exists), GREEN (write `SKILL.md`), verify (re-test with the skill), REFACTOR (close any remaining gaps). The skill is a single self-contained `SKILL.md` that cross-references `jspecify-user-guide` as REQUIRED BACKGROUND and does not duplicate it.

**Tech Stack:** Markdown skill file with YAML frontmatter. Testing via `Agent` tool (general-purpose subagents) running retrieval/application scenarios. Domain: Java, Spring Framework 7, JSpecify, `org.springframework.lang.@Contract`.

---

## File Structure

- Create: `skills/jspecify-spring-null-safety/SKILL.md` — the entire skill, inline. No supporting files; content fits comfortably inline.
- Reference (read-only, do not modify): `skills/jspecify-user-guide/SKILL.md` — the complement skill this one builds on.
- Spec: `docs/superpowers/specs/2026-05-14-jspecify-spring-null-safety-design.md`.

---

## Task 1: RED — Baseline-test the gaps without the skill

Per the writing-skills Iron Law, document how agents behave *without* the skill before writing it. This proves the skill teaches something agents don't already do.

**Files:**
- None created. Record findings in the task notes / commit message of Task 5.

- [ ] **Step 1: Dispatch baseline subagent for the @Contract scenario**

Use the `Agent` tool (`subagent_type: general-purpose`) with this exact prompt:

```
You are reviewing Java code in a Spring Framework 7 project. The whole package is @NullMarked (JSpecify). This class has its own assertion helper:

    private static void requireLoaded(@Nullable Config c) {
        if (c == null) throw new IllegalStateException("config not loaded");
    }

    private @Nullable Config config;

    void refresh() {
        requireLoaded(config);
        config.reload();   // <-- NullAway reports: dereference of @Nullable field 'config'
    }

NullAway flags the `config.reload()` line. Fix the code so the checker accepts it WITHOUT weakening the assertion or suppressing the warning. Show the corrected code and explain.
```

Expected baseline behavior (the gap): the agent adds a redundant `if (config != null)`, calls `Objects.requireNonNull`, reassigns to a local, or applies `@SuppressWarnings` — instead of adding `@Contract("null -> fail")` to `requireLoaded`. Record verbatim what it does.

- [ ] **Step 2: Dispatch baseline subagent for the legacy-migration scenario**

Use the `Agent` tool (`subagent_type: general-purpose`) with this exact prompt:

```
Migrate this Spring Framework class from the deprecated org.springframework.lang nullness annotations to JSpecify. Show the migrated package-info.java and class.

// package-info.java
@NonNullApi
@NonNullFields
package com.example.cache;
import org.springframework.lang.NonNullApi;
import org.springframework.lang.NonNullFields;

// CacheStore.java
package com.example.cache;
import org.springframework.lang.Nullable;

class CacheStore {
    @Nullable private Object[] buffer;
    @Nullable Object get(String key) { ... }
}
```

Expected baseline behavior (the gaps): the agent keeps `@Nullable Object[] buffer` unchanged (silently flipping "nullable array" to "nullable elements" under JSpecify), and/or keeps a `@NonNullFields` equivalent, and/or leaves `@Nullable` in declaration position. Record verbatim.

- [ ] **Step 3: Dispatch baseline subagent for the overriding scenario**

Use the `Agent` tool (`subagent_type: general-purpose`) with this exact prompt:

```
A Spring Framework interface declares (with JSpecify annotations):

    @Nullable String getHeader(String name);

You are implementing this interface in a class whose package is @NullMarked. Write the getHeader override.
```

Expected baseline behavior (the gap): the agent writes `public String getHeader(String name)` — omitting `@Nullable`, assuming the annotation is inherited from the interface. Record verbatim.

- [ ] **Step 4: Record the baseline gaps**

Write a short summary (3-5 lines) of which gaps reproduced. These are the failing tests this skill must make pass. Keep this summary for the Task 5 commit message. If a scenario did NOT show a gap (the agent already did the right thing), note it — that section of the skill may be unnecessary.

---

## Task 2: GREEN — Write SKILL.md

**Files:**
- Create: `skills/jspecify-spring-null-safety/SKILL.md`

- [ ] **Step 1: Create the skill file**

Create `skills/jspecify-spring-null-safety/SKILL.md` with exactly this content:

````markdown
---
name: jspecify-spring-null-safety
description: Use when introducing JSpecify nullness annotations to a Spring Framework project — annotating assertion/precondition methods with @Contract, migrating off deprecated org.springframework.lang annotations (@NonNullApi, @NonNullFields, @Nullable), handling overridden methods, or Kotlin interop. Spring-specific; complements jspecify-user-guide.
---

# JSpecify Null-Safety for Spring Projects

## Overview

Spring Framework 7 exposes its APIs through JSpecify annotations. This skill covers the **Spring-specific** rules for adopting JSpecify in a Spring project. It assumes the general adoption mechanics — `@NullMarked` scoping, where `@Nullable` belongs, type-use placement, generics — are already handled.

**REQUIRED BACKGROUND:** Use superpowers:jspecify-user-guide for the general adoption rules. This skill only adds what is Spring-specific.

## When to Use

- Annotating assertion / precondition helpers so a nullness checker trusts them
- Migrating a Spring project off the deprecated `org.springframework.lang` nullness annotations
- Overriding a Spring (or any) method that carries nullness annotations
- Exposing a `@NullMarked` Java API that Kotlin code will consume

When NOT to use: for general "where do I put `@Nullable`" questions, use jspecify-user-guide instead.

## @Contract on Assertion / Precondition Methods

An assertion helper like `Assert.notNull(x)` does not change `x`'s type, but after it returns successfully `x` is known non-null. JSpecify annotations alone cannot express this — a nullness checker still treats `x` as `@Nullable`. Spring's `org.springframework.lang.@Contract` bridges the gap.

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
2. **Arrays change meaning.** Old Spring `@Nullable Object[] cache` meant "the array may be null." Under JSpecify that exact position means "the *elements* may be null." To preserve the original meaning, migrate it to `Object @Nullable [] cache`. A mechanical import swap silently flips array nullness to element nullness.

## Overridden Methods Do Not Inherit Nullness

JSpecify annotations are **not** inherited from an overridden method. When you override a Spring method (or any annotated method), copy its `@Nullable` annotations onto your override. Omit them and, inside `@NullMarked`, the parameter or return is silently re-declared non-null — changing the contract without warning.

```java
// Spring declares: @Nullable String getHeader(String name);
@Override
public @Nullable String getHeader(String name) {   // @Nullable must be repeated
    ...
}
```

## Kotlin Interop

A correctly annotated `@NullMarked` Java API gives Kotlin callers real null-safety for free: plain Java types become Kotlin non-null (`String`), `@Nullable` types become Kotlin nullable (`String?`).

The consequence: a *missing* `@Nullable` on a Java return that Kotlin consumes is not a warning — Kotlin trusts the Java type as non-null, so the `null` surfaces as a runtime `NullPointerException` at the Kotlin call site. Get the Java API surface right before Kotlin code depends on it.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Assertion helper left un-`@Contract`-ed; callers add redundant null checks or `@SuppressWarnings` | Add `@Contract` to the helper |
| Keeping `@NonNullApi` / `@NonNullFields` alongside `@NullMarked` | Delete both — `@NullMarked` replaces both |
| Mechanically swapping `@Nullable Object[]` imports | Migrate to `Object @Nullable []` to keep "nullable array" meaning |
| Overriding an annotated method without repeating `@Nullable` | Copy the annotations onto the override |
| `@Contract` on methods with no nullness contract | Remove it — only nullness-validating methods need it |
````

- [ ] **Step 2: Verify frontmatter and word count**

Run: `head -4 skills/jspecify-spring-null-safety/SKILL.md && echo "---" && wc -w skills/jspecify-spring-null-safety/SKILL.md`

Expected: frontmatter shows `name:` and `description:` fields; `name` uses only letters/hyphens. Word count should be roughly 600-700 words — acceptable for a multi-topic reference skill. If it substantially exceeds ~750, note it for trimming in Task 4.

---

## Task 3: GREEN verification — Re-test the scenarios with the skill

**Files:**
- None created.

- [ ] **Step 1: Re-run the @Contract scenario with the skill**

Use the `Agent` tool (`subagent_type: general-purpose`) with the Task 1 Step 1 prompt, prefixed with:

```
First read the skill at skills/jspecify-spring-null-safety/SKILL.md and apply it.

[then the exact @Contract scenario prompt from Task 1 Step 1]
```

Expected: the agent adds `@Contract("null -> fail")` to `requireLoaded` and explains that the checker then treats `config` as non-null after the call. PASS.

- [ ] **Step 2: Re-run the legacy-migration scenario with the skill**

Use the `Agent` tool the same way, with the Task 1 Step 2 prompt prefixed with the same "read the skill first" instruction.

Expected: `package-info.java` uses only `@NullMarked` (both `@NonNullApi` and `@NonNullFields` removed); `@Nullable` imports become `org.jspecify.annotations.Nullable` in type-use position; `buffer` becomes `Object @Nullable [] buffer`. PASS.

- [ ] **Step 3: Re-run the overriding scenario with the skill**

Use the `Agent` tool the same way, with the Task 1 Step 3 prompt prefixed with the same "read the skill first" instruction.

Expected: the override is written `public @Nullable String getHeader(String name)` — `@Nullable` repeated. PASS.

- [ ] **Step 4: Record results**

Note which scenarios PASS and which still show the baseline gap. Any non-PASS feeds Task 4.

---

## Task 4: REFACTOR — Close remaining gaps

**Files:**
- Modify: `skills/jspecify-spring-null-safety/SKILL.md` (only if Task 3 found a non-PASS)

- [ ] **Step 1: Triage**

If all three scenarios PASSed in Task 3, skip to Task 5 — there is nothing to refactor.

If any scenario did not PASS: identify what the agent missed or misread. The cause is usually one of — the relevant rule is buried, the wording is ambiguous, or a rationalization is not explicitly countered.

- [ ] **Step 2: Edit SKILL.md to close the specific gap**

Make a targeted edit to the section that failed: sharpen the rule, move it earlier, or add an explicit counter to the rationalization the agent used (quote the agent's reasoning pattern). Do not rewrite unrelated sections.

- [ ] **Step 3: Re-test the previously-failing scenario(s)**

Re-dispatch only the scenario(s) that did not PASS, using the Task 3 procedure. Repeat Steps 2-3 until all three PASS.

---

## Task 5: Commit

**Files:**
- Commit: `skills/jspecify-spring-null-safety/SKILL.md`

- [ ] **Step 1: Stage and commit the skill**

```bash
git add skills/jspecify-spring-null-safety/SKILL.md
git commit -m "feat: add jspecify-spring-null-safety skill

Spring-specific JSpecify adoption rules complementing jspecify-user-guide:
@Contract on assertion methods, legacy org.springframework.lang migration,
override non-inheritance, Kotlin interop.

Baseline-tested without the skill (RED): <summary of gaps from Task 1 Step 4>.
Verified with the skill (GREEN): all three scenarios pass."
```

Replace the `<summary of gaps...>` placeholder with the actual Task 1 Step 4 findings before committing.

- [ ] **Step 2: Verify the commit**

Run: `git log --oneline -1 && git status`
Expected: the commit appears; working tree clean.

---

## Self-Review

- **Spec coverage:** `@Contract` rules → Task 2 §"@Contract on Assertion / Precondition Methods" + Task 1/3 scenario A. Legacy migration (incl. array gotcha, repositioning) → Task 2 §"Migrating Off Deprecated Spring Annotations" + scenario B. Override non-inheritance → Task 2 §"Overridden Methods..." + scenario C. Kotlin interop → Task 2 §"Kotlin Interop" (reviewed via doc inspection in Task 2 Step 2; not subagent-tested, as interop behavior is hard to reproduce in a text scenario). Common Mistakes table → Task 2. Cross-reference to `jspecify-user-guide` as REQUIRED BACKGROUND → Task 2 Overview. All spec sections map to tasks.
- **Placeholder scan:** one intentional fill-in — the commit-message gap summary in Task 5 Step 1 — with an explicit instruction to replace it from Task 1 Step 4 findings. No other placeholders.
- **Type consistency:** scenario prompts in Task 1 and Task 3 are referenced by exact step numbers and reused verbatim; the `SKILL.md` content is given in full once in Task 2 and only edited (not redefined) in Task 4.
