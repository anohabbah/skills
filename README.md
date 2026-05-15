# Skills

A collection of Claude Code skills for Java null-safety tooling, focused on [JSpecify](https://jspecify.dev/) adoption and [NullAway](https://github.com/uber/NullAway) enforcement.

## Skills

| Skill | Purpose |
| --- | --- |
| [`jspecify-user-guide`](skills/jspecify-user-guide/SKILL.md) | Introduce JSpecify annotations (`@Nullable`, `@NullMarked`, `@NullUnmarked`) to a Java project, including incremental adoption and interop with unannotated libraries. |
| [`jspecify-spring-null-safety`](skills/jspecify-spring-null-safety/SKILL.md) | Spring-specific JSpecify adoption — `@Contract` on assertion methods, migrating off `org.springframework.lang` annotations, overriding and Kotlin concerns. |
| [`jspecify-spring-framework-patterns`](skills/jspecify-spring-framework-patterns/SKILL.md) | Infer `@Contract` shapes for Spring methods: boolean nullness-gates, null-preserving transforms, default-substituting and argument-returning fallbacks. |
| [`nullaway-configure`](skills/nullaway-configure/SKILL.md) | Set up NullAway in JSpecify mode for Gradle (Groovy/Kotlin DSL) or Maven, enforcing nullness at compile time. |

## Layout

```
skills/    # one directory per skill, each containing SKILL.md
docs/      # specs and implementation plans
```

## Using a skill

In Claude Code, invoke a skill by name via the `Skill` tool (or type `/<skill-name>`). Each `SKILL.md` begins with frontmatter describing when the skill applies — Claude loads the full content on demand.

### Example prompts

Configure NullAway in a build:

```
Configure NullAway on my project using the nullaway-configure skill.
```

Annotate a package and iterate on NullAway errors, capturing new rules in a skill:

```
Add JSpecify and @Contract annotations to package xxxx to make it null-safe
using all the agent skills prefixed with jspecify, run the build, analyze
NullAway error messages, fix related errors, and repeat until there is no
error.

Add potential new rules discovered to an agent skill named
jspecify-agent-loop-analysis.
```
