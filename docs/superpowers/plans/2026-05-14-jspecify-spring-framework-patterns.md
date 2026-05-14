# jspecify-spring-framework-patterns Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create the `jspecify-spring-framework-patterns` agent skill — the *inference* layer for JSpecify annotations: how to recognize, from a real implementation or its Javadoc, that a return/parameter/field should be `@Nullable` or that a method should carry `@Contract`.

**Architecture:** Skill creation follows the writing-skills TDD cycle: RED (baseline-test the gaps with subagents *before* the skill exists), GREEN (write `SKILL.md`), verify (re-test with the skill), REFACTOR (close remaining gaps). The skill is a single self-contained `SKILL.md` that cross-references `jspecify-user-guide` and `jspecify-spring-null-safety` as REQUIRED BACKGROUND and does not duplicate either. The signal catalog has already been validated against `spring-core` source (see findings below); its synthetic examples are distilled from that read.

**Tech Stack:** Markdown skill file with YAML frontmatter. Testing via the `Agent` tool (general-purpose subagents) running annotate-this-code scenarios. Domain: Java, Spring Framework `spring-core`, JSpecify, `org.springframework.lang.@Contract`.

---

## spring-core Analysis Findings (reference — already performed)

The signal catalog in Task 2 is derived from reading `spring-core` (`Assert`, `ObjectUtils`, `StringUtils`, `CollectionUtils`, `ClassUtils`, `MethodParameter`). Verified patterns:

- **`@Nullable` return + `@return` Javadoc naming null** — `StringUtils.getFilenameExtension` (`/** @return ... or {@code null} if none */ ... @Nullable String`), `StringUtils.parseLocale` ("or `null` if none"). The word `null` in `@return` reliably co-occurs with a `@Nullable` return.
- **Null-preserving transform** — `StringUtils.getFilename`/`quote`/`quoteIfString` carry `@Contract("null -> null; !null -> !null")`; bodies are `if (x == null) return null; ... return nonNull;`.
- **`null -> null` (one-directional)** — `ObjectUtils.unwrapOptional`, `StringUtils.getFilenameExtension` — `@Nullable` return, returns null when the arg is null but may also return null otherwise.
- **Argument-returning fallback** — `StringUtils.concatenateStringArrays` carries `@Contract("null, _ -> param2; _, null -> param1")` — returns one argument when the other is null.
- **Boolean nullness-gate** — `ObjectUtils.isEmpty` → `@Contract("null -> true")`; `ObjectUtils.isArray` → `@Contract("null -> false")`; `Assert.hasText`-family validate non-null. `null -> true` / `null -> false` are heavily used and absent from both existing skills.
- **Assertion / throws-on-null** — `Assert.notNull` → `@Contract("null, _ -> fail")`, `Assert.isNull` → `@Contract("!null, _ -> fail")`. Covered by `jspecify-spring-null-safety`; this skill only points to it.
- **Lazy `@Nullable` field with non-null getter** — `MethodParameter` has `private volatile @Nullable Class<?> parameterType;` and a getter `public Class<?> getParameterType()` (return is **non-null**) that lazily computes and caches. The field is `@Nullable`; the getter return is not.

---

## File Structure

- Create: `skills/jspecify-spring-framework-patterns/SKILL.md` — the entire skill, inline. No supporting files; content fits comfortably inline.
- Reference (read-only, do not modify): `skills/jspecify-user-guide/SKILL.md`, `skills/jspecify-spring-null-safety/SKILL.md` — the complement skills this one builds on.
- Spec: `docs/superpowers/specs/2026-05-14-jspecify-spring-framework-patterns-design.md`.

---

## Task 1: RED — Baseline-test the gaps without the skill

Per the writing-skills Iron Law, document how agents behave *without* the skill before writing it. This proves the skill teaches something agents don't already do. Each scenario targets one expected gap from the spec.

**Files:**
- None created. Record findings in the Task 1 Step 4 notes and carry them into the Task 5 commit message.

- [ ] **Step 1: Dispatch baseline subagent for the boolean-`@Contract` scenario**

Use the `Agent` tool (`subagent_type: general-purpose`) with this exact prompt:

```
This is a Spring Framework 7 project; the package is @NullMarked (JSpecify). Add JSpecify nullness annotations to this utility method so its contract is fully expressed for a nullness checker like NullAway. Show the annotated method.

public static boolean isEmpty(Object obj) {
    if (obj == null) {
        return true;
    }
    if (obj instanceof CharSequence cs) {
        return cs.isEmpty();
    }
    return false;
}

A caller does:
    if (!ObjectUtils.isEmpty(value)) {
        value.toString();   // wants the checker to treat value as non-null here
    }
```

Expected baseline behavior (the gap): the agent adds `@Nullable` to the `obj` parameter but does **not** add `@Contract("null -> true")`, so the caller's `value` is not narrowed in the `else` branch. Record verbatim what it does.

- [ ] **Step 2: Dispatch baseline subagent for the name-over-body scenario**

Use the `Agent` tool (`subagent_type: general-purpose`) with this exact prompt:

```
This is a Spring Framework 7 project; the package is @NullMarked (JSpecify). Add JSpecify nullness annotations to this method. Show the annotated method.

/**
 * Return the configured executor.
 * @return the executor instance
 */
public Executor getExecutor() {
    if (this.executor == null) {
        return null;
    }
    return this.executor;
}
```

Expected baseline behavior (the gap): the agent leaves the return type plain (non-null) because the method is named `getExecutor` and the Javadoc says "the executor instance" — trusting the name/Javadoc over the body, which plainly has `return null`. Record verbatim.

- [ ] **Step 3: Dispatch baseline subagent for the lazy-field scenario**

Use the `Agent` tool (`subagent_type: general-purpose`) with this exact prompt:

```
This is a Spring Framework 7 project; the package is @NullMarked (JSpecify). Add JSpecify nullness annotations to the field(s) and method(s) in this class. Show the annotated class.

class TypeDescriptor {

    private volatile Class<?> resolvedType;

    private final Field field;

    TypeDescriptor(Field field) {
        this.field = field;
    }

    public Class<?> getResolvedType() {
        Class<?> type = this.resolvedType;
        if (type == null) {
            type = this.field.getType();
            this.resolvedType = type;
        }
        return type;
    }
}
```

Expected baseline behavior (the gap): the agent leaves `resolvedType` plain (non-null) because "it is always set before it is returned", and/or marks `getResolvedType()`'s return `@Nullable`. The correct reading: the field is `@Nullable` (null between construction and first call); the getter return is non-null. Record verbatim.

- [ ] **Step 4: Record the baseline gaps**

Write the verbatim findings for each scenario into this step as `**RED findings (YYYY-MM-DD):**` with one bullet per scenario, each marked `GAP` or `NO GAP`. If a scenario shows NO GAP, note it — the corresponding section stays in the skill as reference but is not GREEN-verified in Task 3.

---

## Task 2: GREEN — Write SKILL.md

**Files:**
- Create: `skills/jspecify-spring-framework-patterns/SKILL.md`

- [ ] **Step 1: Create the skill file**

Create `skills/jspecify-spring-framework-patterns/SKILL.md` with exactly this content:

````markdown
---
name: jspecify-spring-framework-patterns
description: Use when deciding whether a specific return, parameter, or field should be @Nullable — or whether a method should carry @Contract — by reading its implementation or Javadoc rather than from placement rules alone. Spring Framework / JSpecify; complements jspecify-user-guide and jspecify-spring-null-safety.
---

# Inferring JSpecify Annotations from Spring Code

## Overview

`@NullMarked` sets the *default*; it does not tell you which signatures are the exceptions. This skill is the recognition layer: given a real unannotated method or field, what in its **implementation or Javadoc** tells you a `@Nullable` or `@Contract` annotation belongs. The patterns are distilled from Spring Framework's `spring-core` module.

**REQUIRED BACKGROUND:** Use jspecify-user-guide for the placement rules and jspecify-spring-null-safety for `@Contract` syntax and the Spring-vs-JetBrains annotation choice. This skill only adds how to *infer* what to place.

**Precedence rule — when signals disagree:** code-structure > Javadoc > naming idiom. The body is ground truth: a `getX()` whose body has `return null` is `@Nullable` despite the name; Javadoc claiming "never null" loses to a body that returns null.

## When to Use

- Annotating an existing unannotated Spring class
- Reviewing whether an existing `@Nullable` / `@Contract` is correct
- Unsure whether a given return, parameter, or field is nullable

When NOT to use: for *where* the annotation token goes (type-use position, arrays, generics) → jspecify-user-guide; for `@Contract` *string syntax* → jspecify-spring-null-safety.

## Inferring `@Nullable` Returns

| Signal | Conclusion |
|--------|-----------|
| Body has `return null;` on any path | `@Nullable` return |
| `@return` Javadoc mentions null ("or `null` if none", "`null` if not found") | `@Nullable` return |
| Returns a known-`@Nullable` call directly (`return map.get(k);`) | `@Nullable` return |
| Returns `optional.orElse(null)` | `@Nullable` return |
| Name idiom: `find*`, `*IfAvailable`, `resolve*` | *weak* hint — confirm against the body |
| Every path returns a fresh object, literal, `this`, or a non-null param | leave plain |

```java
// @return Javadoc names null AND the body returns null → @Nullable return
/** @return the filename extension, or {@code null} if none */
static @Nullable String getFilenameExtension(@Nullable String path) {
    if (path == null) {
        return null;
    }
    // ...
}
```

## Inferring `@Nullable` Parameters

| Signal | Conclusion |
|--------|-----------|
| Body *tolerates* null: `if (p != null)`, `p == null ? fallback : p` | `@Nullable` param |
| Param passed onward to a `@Nullable`-accepting method with no prior check | `@Nullable` param |
| `@param` Javadoc: "may be `null`", "`null` to use the default" | `@Nullable` param |
| Overload `doX(s)` delegates `doX(s, null)` | the delegated param is `@Nullable` |
| Param dereferenced immediately, or `Assert.notNull(p)` at the top | leave plain — **non-null** |

The counter-signal matters: an `Assert.notNull(p)` *enforces* a non-null contract, it does not tolerate null. That parameter stays plain.

## Inferring `@Nullable` Fields

| Signal | Conclusion |
|--------|-----------|
| Lazy field: no initializer, set inside an `if (f == null)` getter or an `init()` method | `@Nullable` field |
| Field assigned only by an optional setter / configured after construction | `@Nullable` field |
| Cached / memoized field | `@Nullable` field |
| `final` field assigned non-null in every constructor | leave plain |

A lazy field is `@Nullable` even when its getter's return is non-null — the field is null between construction and first use; the getter computes and caches a value:

```java
private volatile @Nullable Class<?> parameterType;   // lazy → @Nullable field

public Class<?> getParameterType() {                 // return is non-null
    Class<?> type = this.parameterType;
    if (type == null) {
        type = compute();
        this.parameterType = type;
    }
    return type;
}
```

## Inferring `@Contract`

Recognize the method *shape*; get the string syntax from jspecify-spring-null-safety.

| Shape in the body | Contract |
|-------------------|----------|
| `x == null ? null : f(x)` — returns null iff the input is null | `null -> null` (add `; !null -> !null` when `f` never returns null) |
| Returns one argument when another argument is null | `null, _ -> param2; _, null -> param1` |
| Boolean method whose result decides an arg's nullness — true when null | `null -> true` |
| Boolean method — false when null (`hasText`, `hasLength`) | `null -> false` |
| Picks the first non-null of its arguments (`x != null ? x : dflt`) | `!null, _ -> param1` |
| Throws when an argument is null | see jspecify-spring-null-safety (assertion-method rule) |

The boolean nullness-gate is the most-missed: `isEmpty(x)` returning `true` for null earns `@Contract("null -> true")`, so a caller's `if (!isEmpty(x))` branch can treat `x` as non-null.

## Reviewing a Class — Procedure

1. Confirm `@NullMarked` is in effect (apply it per jspecify-user-guide if not).
2. **Fields** — apply the field signals to each.
3. **Returns** — check each method body for `return null` / `@Nullable` delegation / a null-naming `@return`.
4. **Parameters** — check each for null-tolerance vs. immediate-dereference / assertion.
5. **`@Contract`** — match each method against the shape table.
6. On any conflict between signals, apply the precedence rule (body > Javadoc > name).

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Trusting the method name over the body (`getX` assumed non-null though it returns null) | Read the body — code-structure wins |
| Trusting stale Javadoc over the implementation | The body is ground truth |
| Marking an `Assert.notNull(p)`'d parameter `@Nullable` | The check enforces non-null — leave the param plain |
| Missing `@Contract` on boolean nullness-gates (`isEmpty`, `hasText`) | Add `null -> true` / `null -> false` so callers can narrow |
| Treating a lazy `@Nullable` field as non-null because "it is always set before use" | It is null before first init — `@Nullable` is correct; the getter return can still be non-null |
````

- [ ] **Step 2: Verify frontmatter and word count**

Run: `head -4 skills/jspecify-spring-framework-patterns/SKILL.md && echo "---" && wc -w skills/jspecify-spring-framework-patterns/SKILL.md`

Expected: frontmatter shows `name:` and `description:` fields; `name` uses only letters and hyphens. Word count should be roughly 600–750 words — acceptable for a multi-section pattern/reference skill (siblings are in the same range). If it substantially exceeds ~800, note it for trimming in Task 4.

---

## Task 3: GREEN verification — Re-test the proven-gap scenarios with the skill

Re-run each Task 1 scenario that showed a `GAP`, now with the skill available. Skip any scenario that showed `NO GAP` in Task 1 Step 4.

**Files:**
- None created.

- [ ] **Step 1: Re-run the boolean-`@Contract` scenario with the skill**

Use the `Agent` tool (`subagent_type: general-purpose`) with the Task 1 Step 1 prompt, prefixed with:

```
First read the skill at skills/jspecify-spring-framework-patterns/SKILL.md and apply it.

[then the exact boolean-@Contract scenario prompt from Task 1 Step 1]
```

Expected (PASS): the agent adds `@Nullable` to `obj` **and** `@Contract("null -> true")` to the method, and explains the caller's `value` is then non-null in the `!isEmpty` branch.

- [ ] **Step 2: Re-run the name-over-body scenario with the skill**

Use the `Agent` tool the same way, with the Task 1 Step 2 prompt prefixed with the same "read the skill first" instruction.

Expected (PASS): the agent annotates the return `@Nullable String` (technically `@Nullable Executor`) because the body has `return null`, explicitly noting that the body overrides the method name and Javadoc.

- [ ] **Step 3: Re-run the lazy-field scenario with the skill**

Use the `Agent` tool the same way, with the Task 1 Step 3 prompt prefixed with the same "read the skill first" instruction.

Expected (PASS): the agent marks `resolvedType` as `@Nullable` (lazy field) and leaves `getResolvedType()`'s return **non-null**, explaining the field is null only between construction and first use.

- [ ] **Step 4: Record results**

Note which scenarios PASS and which still show the baseline gap. Any non-PASS feeds Task 4.

---

## Task 4: REFACTOR — Close remaining gaps

**Files:**
- Modify: `skills/jspecify-spring-framework-patterns/SKILL.md` (only if Task 3 found a non-PASS)

- [ ] **Step 1: Triage**

If every re-tested scenario PASSed in Task 3, skip to Task 5 — there is nothing to refactor.

If any scenario did not PASS: identify what the agent missed or misread. The cause is usually one of — the relevant signal is buried, the wording is ambiguous, or a rationalization is not explicitly countered (e.g. "the field is always set before use, so it is effectively non-null").

- [ ] **Step 2: Edit SKILL.md to close the specific gap**

Make a targeted edit to the section that failed: sharpen the signal row, move it earlier, or add an explicit counter to the rationalization the agent used (quote the agent's reasoning pattern in the Common Mistakes table). Do not rewrite unrelated sections.

- [ ] **Step 3: Re-test the previously-failing scenario(s)**

Re-dispatch only the scenario(s) that did not PASS, using the Task 3 procedure. Repeat Steps 2–3 until all re-tested scenarios PASS.

---

## Task 5: Commit

**Files:**
- Commit: `skills/jspecify-spring-framework-patterns/SKILL.md`

- [ ] **Step 1: Stage and commit the skill**

```bash
git add skills/jspecify-spring-framework-patterns/SKILL.md
git commit -m "feat: add jspecify-spring-framework-patterns skill

Inference layer for JSpecify annotations, distilled from spring-core:
how to recognize from an implementation or its Javadoc that a return,
parameter, or field should be @Nullable, or that a method should carry
@Contract. Complements jspecify-user-guide and jspecify-spring-null-safety.

Baseline-tested without the skill (RED): <summary of gaps from Task 1 Step 4>.
Verified with the skill (GREEN): <summary of Task 3 results>."
```

Replace both `<summary ...>` placeholders with the actual Task 1 Step 4 and Task 3 findings before committing.

- [ ] **Step 2: Verify the commit**

Run: `git log --oneline -1 && git status`
Expected: the commit appears; working tree clean.

---

## Self-Review

- **Spec coverage:**
  - Inference signal catalog by target → Task 2 §"Inferring `@Nullable` Returns", §"Inferring `@Nullable` Parameters", §"Inferring `@Nullable` Fields", §"Inferring `@Contract`".
  - Precedence rule (code > Javadoc > name) → Task 2 §"Overview" + §"Reviewing a Class" step 6 + Common Mistakes rows 1–2.
  - Short class-review procedure → Task 2 §"Reviewing a Class — Procedure".
  - Cross-references to both existing skills as REQUIRED BACKGROUND → Task 2 §"Overview" and §"When to Use".
  - Synthetic grounding, no inline spring-core citations → Task 2 examples are synthetic; the spring-core read is recorded in the plan's "Analysis Findings" section, not the skill.
  - Verification: spring-core analysis → done (Analysis Findings section); RED → Task 1; GREEN → Tasks 2–3; REFACTOR → Task 4.
  - All spec sections map to tasks.
- **Placeholder scan:** two intentional fill-ins — the `<summary ...>` gaps in the Task 5 commit message — each with an explicit instruction to replace from Task 1 Step 4 / Task 3 findings. The Task 1 Step 4 "RED findings" block is to be filled during execution (standard for the RED phase). No other placeholders; all skill content and all scenario prompts are given in full.
- **Type consistency:** the `SKILL.md` content is given in full exactly once (Task 2 Step 1) and only edited, not redefined, in Task 4. Scenario prompts are defined in Task 1 and referenced by exact step number in Task 3. Section names used in the Self-Review match the headings in the Task 2 content verbatim.
