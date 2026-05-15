---
name: nullaway-configure
description: Use when setting up NullAway with JSpecify mode in a Java project's build — Gradle (Groovy DSL), Gradle (Kotlin DSL), or Maven — to enforce `@NullMarked`/`@Nullable` annotations at compile time.
---

# Configure NullAway with JSpecify

## Overview

[NullAway](https://github.com/uber/NullAway) is an Error Prone-based static analyzer that enforces null safety at compile time. In **JSpecify mode** it consumes the standard `org.jspecify.annotations` (`@NullMarked`, `@NullUnmarked`, `@Nullable`) and treats violations as build errors.

A working setup needs four things wired together:

1. **JSpecify annotations** on the classpath (`org.jspecify:jspecify`).
2. **Error Prone** as the compiler plugin host.
3. **NullAway** as an Error Prone plugin, configured with `jspecifyMode = true`.
4. A **recent JDK** (24+, the demo uses 25) to run javac — NullAway's JSpecify mode needs current javac internals. You can still target an older bytecode level via `--release`.

The reference demo this skill mirrors: <https://github.com/sdeleuze/jspecify-nullaway-demo> (Gradle on `main`, Maven on the `maven` branch).

## When to Use

- Adding null-safety enforcement to a new or existing Java module.
- Migrating from `org.springframework.lang.Nullable` / `javax.annotation.Nullable` to JSpecify and wanting compile-time checking.
- Setting up CI to fail on unannotated APIs (`RequireExplicitNullMarking`).
- Pair with [[jspecify-user-guide]] for annotation semantics and [[jspecify-spring-framework-patterns]] for Spring-specific patterns.

Not for: runtime null checks (use `Objects.requireNonNull` / Bean Validation), Kotlin code (the compiler already does this), or non-JVM languages.

## Key Configuration Knobs

| Knob | Purpose | Demo value |
|------|---------|-----------|
| `nullaway.onlyNullMarked` | Only analyze code inside `@NullMarked` scopes | `true` |
| `nullaway.jspecifyMode` | Interpret JSpecify annotations per [JSpecify spec](https://jspecify.dev/) | `true` |
| `errorprone.disableAllChecks` | Run *only* NullAway, skip other Error Prone checks | `true` |
| `errorprone.error('RequireExplicitNullMarking')` | Fail build if any package/class lacks `@NullMarked` or `@NullUnmarked` | enabled |
| `errorprone.nullaway { error() }` | Promote NullAway warnings to errors | enabled |
| `java.toolchain.languageVersion` | JDK used to *run* javac (needs recent version) | `25` |
| `options.release` | Bytecode target (can stay on LTS) | `17` |

## Gradle (Groovy DSL) — `build.gradle`

```groovy
plugins {
    id 'java'
    id 'net.ltgt.errorprone' version '5.1.0'  // gradle-errorprone-plugin
    id 'net.ltgt.nullaway'  version '3.0.0'   // gradle-nullaway-plugin
}

repositories { mavenCentral() }

java {
    toolchain {
        // Recent javac required for NullAway JSpecify mode
        languageVersion = JavaLanguageVersion.of(25)
    }
}

tasks.withType(JavaCompile).configureEach {
    options.errorprone {
        disableAllChecks = true                    // run only NullAway
        error('RequireExplicitNullMarking')        // every type must be @NullMarked or @NullUnmarked
        nullaway {
            error()                                // promote to error
        }
    }
    options.release = 17                           // keep bytecode on an LTS baseline
}

nullaway {
    onlyNullMarked = true
    jspecifyMode   = true
}

dependencies {
    implementation 'org.jspecify:jspecify:1.0.0'
    errorprone     'com.google.errorprone:error_prone_core:2.42.0'
    errorprone     'com.uber.nullaway:nullaway:0.13.4'
}
```

## Gradle (Kotlin DSL) — `build.gradle.kts`

Identical setup; only differences are the `import` for the `errorprone` extension accessor and Kotlin syntax.

```kotlin
import net.ltgt.gradle.errorprone.errorprone

plugins {
    java
    id("net.ltgt.errorprone") version "5.1.0"
    id("net.ltgt.nullaway")  version "3.0.0"
}

repositories { mavenCentral() }

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(25)
    }
}

nullaway {
    onlyNullMarked = true
    jspecifyMode   = true
}

tasks.withType<JavaCompile> {
    options.errorprone {
        disableAllChecks = true
        error("RequireExplicitNullMarking")
        nullaway {
            error()
        }
    }
    options.release = 17
}

dependencies {
    implementation("org.jspecify:jspecify:1.0.0")
    errorprone("com.google.errorprone:error_prone_core:2.42.0")
    errorprone("com.uber.nullaway:nullaway:0.13.4")
}
```

## Maven — `pom.xml`

The demo's `maven` branch uses [am.ik.maven:nullability-maven-plugin](https://github.com/making/nullability-maven-plugin), which auto-wires Error Prone + NullAway + JSpecify mode into `maven-compiler-plugin` so you don't hand-write all the `<annotationProcessorPath>` / `<compilerArg>` boilerplate.

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>my-app</artifactId>
    <version>0.0.1-SNAPSHOT</version>

    <properties>
        <java.version>25</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.jspecify</groupId>
            <artifactId>jspecify</artifactId>
            <version>1.0.0</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.14.1</version>
            </plugin>
            <plugin>
                <groupId>am.ik.maven</groupId>
                <artifactId>nullability-maven-plugin</artifactId>
                <version>0.3.0</version>
                <extensions>true</extensions>           <!-- required so it can patch the compiler plugin -->
                <executions>
                    <execution>
                        <goals>
                            <goal>configure</goal>      <!-- injects Error Prone + NullAway config -->
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

Set `<java.version>` to the JDK actually running Maven; the plugin adds the `--release` flag for you. To override defaults (e.g. allow legacy `javax.annotation.Nullable`), see the plugin README.

## Minimum source-level setup

Once the build is configured, you still need at least one `@NullMarked` scope, otherwise NullAway has nothing to check (with `onlyNullMarked = true`) and `RequireExplicitNullMarking` will fail. The conventional place is a `package-info.java`:

```java
@NullMarked
package org.example;

import org.jspecify.annotations.NullMarked;
```

Then mark individual nullable returns/parameters with `@Nullable`:

```java
public @Nullable String extractToken(String input) {
    return input.contains("token") ? "token" : null;
}
```

## Verifying

- `./gradlew build` (or `./mvnw verify`) — should compile clean on annotated code, fail on a deliberate `null` returned from a non-`@Nullable` method.
- Remove `@NullMarked` from `package-info.java` — `RequireExplicitNullMarking` should fail the build.
- Reintroduce the leak from the demo: have `DefaultTokenExtractor.extractToken` return `null` but drop `@Nullable` from the signature; NullAway should flag the dereference site.

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Old JDK running javac | `IllegalAccessError` from NullAway, or JSpecify mode silently weakened | Set Gradle `toolchain.languageVersion` (or Maven JDK) to 24+ |
| Forgot `errorprone` configuration entries for the deps | `error_prone_core` / `nullaway` on `implementation` instead of `errorprone` | Use the dedicated `errorprone` configuration (Gradle) — Maven plugin handles this |
| No `@NullMarked` anywhere | Build passes but nothing is checked | Add `package-info.java` with `@NullMarked` (or annotate at type/module level) |
| Mixing `jakarta`/`javax`/Spring `@Nullable` with JSpecify | NullAway ignores foreign annotations in `jspecifyMode` | Migrate imports to `org.jspecify.annotations.Nullable` |
| Targeting a newer `--release` than needed | Downstream consumers on LTS break | Decouple toolchain (run javac on 25) from `options.release` (emit 17) |

## References

- Demo repo (Gradle): <https://github.com/sdeleuze/jspecify-nullaway-demo>
- Demo repo (Maven branch): <https://github.com/sdeleuze/jspecify-nullaway-demo/tree/maven>
- NullAway: <https://github.com/uber/NullAway>
- gradle-errorprone-plugin: <https://github.com/tbroyer/gradle-errorprone-plugin>
- gradle-nullaway-plugin: <https://github.com/tbroyer/gradle-nullaway-plugin>
- nullability-maven-plugin: <https://github.com/making/nullability-maven-plugin>
- JSpecify: <https://jspecify.dev/>
