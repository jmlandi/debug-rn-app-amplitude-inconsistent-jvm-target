# JVM Target Mismatch in React Native (Android)

## Overview

This repository demonstrates a build failure caused by a mismatch between:

- Java compilation target version
- Kotlin JVM target version

The error reproduced in the `error` branch is:
```
Inconsistent JVM-target compatibility detected for tasks
'compileDebugJavaWithJavac' (1.8) and
'compileDebugKotlin' (17)
```

This document explains:

- What causes the error
- Why it happens
- Why it does NOT always appear in React Native projects
- How to fix it properly


------------------------------------------------------------

## What Causes This Error?

In Android builds, both Java and Kotlin code are compiled to JVM bytecode.

Each compiler has a target:

- Java → defined in `compileOptions`
- Kotlin → defined in `kotlinOptions.jvmTarget`

If these targets are different, Gradle may fail the build with:
```
Inconsistent JVM-target compatibility detected
```

Example mismatch:

Java:
    sourceCompatibility = JavaVersion.VERSION_1_8
    targetCompatibility = JavaVersion.VERSION_1_8

Kotlin:
    jvmTarget = "17"

This produces:

Java bytecode → 1.8
Kotlin bytecode → 17

Gradle considers this inconsistent.


------------------------------------------------------------

## Why This Is Confusing in React Native Projects

React Native’s Gradle Plugin automatically aligns Java and Kotlin versions by default.

That means:

Even if you configure:

    Java → 1.8
    Kotlin → 17

React Native may silently adjust one of them so they match.

Because of this automatic alignment, many developers never see the mismatch error.


------------------------------------------------------------

## How the Error Was Reproduced

To force the error, we disabled React Native’s automatic version alignment.

In `gradle.properties`:

    react.internal.disableJavaVersionAlignment=true

Then we explicitly configured:

In `app/build.gradle`:

Java:
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }

Kotlin:
    kotlinOptions {
        jvmTarget = "17"
    }

After cleaning all Gradle caches and rebuilding:

    ./gradlew clean
    ./gradlew assembleDebug

The build failed with:

    Inconsistent JVM-target compatibility detected


------------------------------------------------------------

## Why This Happens Technically

Java compiler → produces bytecode targeting a specific JVM version.
Kotlin compiler → also produces bytecode targeting a JVM version.

If:
    Java target = 1.8
    Kotlin target = 17

You end up mixing bytecode levels inside the same module.

Gradle 8+ performs stricter validation and fails the build
instead of allowing potentially incompatible bytecode.


------------------------------------------------------------

## Why Some Developers Encounter This

This issue typically appears when:

- Migrating to a newer Kotlin version
- Upgrading Android Gradle Plugin
- Upgrading Gradle
- Using Java 17 locally
- Disabling React Native alignment
- Having custom Gradle configurations


------------------------------------------------------------

## How to Fix Properly

The fix is simple:

Make Java and Kotlin targets identical.

Recommended modern configuration (Java 17):

In `app/build.gradle`:

    android {
        compileOptions {
            sourceCompatibility JavaVersion.VERSION_17
            targetCompatibility JavaVersion.VERSION_17
        }
    }

    tasks.withType(org.jetbrains.kotlin.gradle.tasks.KotlinCompile).configureEach {
        kotlinOptions {
            jvmTarget = "17"
        }
    }

After aligning both to 17:

    ./gradlew clean
    ./gradlew assembleDebug

Build succeeds.


------------------------------------------------------------

## Branch Structure in This Repository

main  → Contains the FIX (aligned Java and Kotlin versions)
error → Contains the MISCONFIGURATION that reproduces the failure


------------------------------------------------------------

## Key Takeaways

1. Java and Kotlin must compile to the same JVM target.
2. React Native normally hides this issue via automatic alignment.
3. Disabling alignment exposes real configuration mismatches.
4. Gradle 8+ enforces stricter JVM target validation.
5. Always align:
       Java compileOptions
       Kotlin jvmTarget


------------------------------------------------------------

## Final Recommendation

For modern React Native Android projects:

Use Java 17 consistently across:

- compileOptions
- kotlinOptions.jvmTarget
- Local JDK version

Avoid mixing 1.8 and 17.

Consistency prevents subtle and hard-to-debug build failures.
