---
name: nullaway-configure
description: Use when wiring NullAway (JSpecify mode) into a Java build so nullness violations fail compilation — configuring Error Prone + NullAway in a Gradle Groovy build.gradle, a Gradle Kotlin build.gradle.kts, or a Maven pom.xml, picking the jspecifyMode/onlyNullMarked/RequireExplicitNullMarking knobs, and choosing plugin versions and the JDK/release baseline.
license: MIT
metadata:
  author: nullaway-configure
  version: "1.0"
  source: https://github.com/sdeleuze/jspecify-nullaway-demo (main + maven branch)
---

# Configuring NullAway (JSpecify mode)

NullAway is an **Error Prone plugin** that checks JSpecify nullness at compile time and
turns violations into build failures. Companion to `jspecify-userguide` /
`jspecify-spring-null-safety` (the annotation rules) — this skill is the *build wiring*
that enforces them. Versions below are from the reference demo; check for newer releases.

## Prerequisites (all build systems)

- `org.jspecify:jspecify:1.0.0` on the compile classpath.
- Code marked with `@NullMarked` (e.g. in each `package-info.java`).
- A **recent javac** (the demo uses a JDK 25 toolchain) — JSpecify mode needs JDK 22+.
  You can still target an older bytecode baseline via `release`/`java.version` (demo
  keeps `release = 17` while compiling with JDK 25).

## The four NullAway knobs

| Knob | Effect |
|------|--------|
| `jspecifyMode = true` | Full JSpecify semantics (generics, bounds, arrays). Needs recent javac. |
| `onlyNullMarked = true` | Check only `@NullMarked` code; treat unmarked code as unspecified. |
| `nullaway { error() }` | Make nullness violations **errors** (fail the build), not warnings. |
| `error('RequireExplicitNullMarking')` | Force every class to be explicitly `@NullMarked` or `@NullUnmarked`. |

Pair with `disableAllChecks = true` to run **only** NullAway and skip the rest of Error
Prone. The Gradle plugins inject the required `-XDcompilePolicy=simple -Xplugin:ErrorProne`
javac args automatically; on raw javac you must add them yourself.

## Gradle (Groovy) — `build.gradle`

```groovy
plugins {
    id 'java'
    id 'net.ltgt.errorprone' version '5.1.0'
    id 'net.ltgt.nullaway' version '3.0.0'
}

java {
    toolchain { languageVersion = JavaLanguageVersion.of(25) } // recent javac for JSpecify mode
}

tasks.withType(JavaCompile).configureEach {
    options.errorprone {
        disableAllChecks = true                  // run only NullAway
        error('RequireExplicitNullMarking')      // every type must be @NullMarked/@NullUnmarked
        nullaway { error() }                     // violations fail the build
    }
    options.release = 17                         // bytecode baseline
}

nullaway {
    onlyNullMarked = true
    jspecifyMode = true
}

dependencies {
    implementation 'org.jspecify:jspecify:1.0.0'
    errorprone 'com.google.errorprone:error_prone_core:2.42.0'
    errorprone 'com.uber.nullaway:nullaway:0.13.4'
}
```

## Gradle (Kotlin) — `build.gradle.kts`

Same setup; note the required import and the `withType<JavaCompile>` / parenthesized DSL.

```kotlin
import net.ltgt.gradle.errorprone.errorprone   // required to access options.errorprone

plugins {
    id("java")
    id("net.ltgt.errorprone") version "5.1.0"
    id("net.ltgt.nullaway") version "3.0.0"
}

java {
    toolchain { languageVersion = JavaLanguageVersion.of(25) }
}

nullaway {
    onlyNullMarked = true
    jspecifyMode = true
}

tasks.withType<JavaCompile> {
    options.errorprone {
        disableAllChecks = true
        error("RequireExplicitNullMarking")
        nullaway { error() }
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

The demo's maven branch uses the **`nullability-maven-plugin`**, which auto-wires Error
Prone + NullAway into the compiler plugin via its `configure` goal (`<extensions>true</extensions>`
lets it inject compiler args):

```xml
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
            <extensions>true</extensions>
            <executions>
                <execution>
                    <goals><goal>configure</goal></goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

If you cannot use that convenience plugin, the manual equivalent configures
`maven-compiler-plugin` with Error Prone + NullAway on `annotationProcessorPaths` and the
`-XDcompilePolicy=simple -Xplugin:ErrorProne -XepOpt:NullAway:...` compiler args — the
plugin exists precisely to avoid that boilerplate.

## Gotchas

| Symptom | Cause / fix |
|---------|-------------|
| NullAway reports nothing | Code isn't `@NullMarked`, or `onlyNullMarked = true` excludes it. Add `@NullMarked`. |
| `jspecifyMode` errors / generics not checked | javac too old; use a JDK 22+ toolchain (demo uses 25). |
| Violations are only warnings | Add `nullaway { error() }` (and run NullAway as `error`, not `warn`). |
| Flood of unrelated Error Prone findings | Set `disableAllChecks = true` and re-enable only NullAway. |
| `options.errorprone` unresolved in Kotlin DSL | Add `import net.ltgt.gradle.errorprone.errorprone`. |
| Error Prone not actually running on raw javac | Missing `-XDcompilePolicy=simple -Xplugin:ErrorProne`; the Gradle/Maven plugins add these for you. |
| Want classes flagged when neither marked nor unmarked | `error('RequireExplicitNullMarking')`. |
