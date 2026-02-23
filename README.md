# 🔧 JVM Target Mismatch in React Native (Android)

> **Who is this for?** Developers who are not deeply familiar with React Native's Android build system and need to understand — and fix — a JVM target compatibility error.

---

## 📋 Table of Contents

- [What Is the Error?](#-what-is-the-error)
- [Background: How Android Builds Work](#-background-how-android-builds-work)
- [What Causes the Mismatch?](#-what-causes-the-mismatch)
- [Why You Might Never Have Seen This Before](#-why-you-might-never-have-seen-this-before)
- [How to Reproduce It](#-how-to-reproduce-it)
- [How to Fix It](#-how-to-fix-it)
- [When Does This Typically Appear?](#-when-does-this-typically-appear)
- [Key Takeaways](#-key-takeaways)

---

## 🚨 What Is the Error?

When building your React Native Android app, you may encounter:

```
Inconsistent JVM-target compatibility detected for tasks
'compileDebugJavaWithJavac' (1.8) and
'compileDebugKotlin' (17)
```

This means your **Java code** and **Kotlin code** are being compiled to different versions of JVM bytecode, which Gradle (the Android build tool) refuses to allow.

---

## 👀 Background: How Android Builds Work

To understand this error, it helps to know what happens when you build an Android app:

```
Your Java/Kotlin source code
        │
        ▼
  Android Compiler
  ┌─────────────────────────────────┐
  │  javac  → compiles Java code    │  ← targets a JVM version
  │  kotlinc → compiles Kotlin code │  ← targets a JVM version
  └─────────────────────────────────┘
        │
        ▼
  .class bytecode files (bundled into your APK)
```

Both compilers must output bytecode targeting the **same JVM version**. Think of it like two people writing a document — they need to use the same file format, or nobody can read it consistently.

---

## 🔴 What Causes the Mismatch?

In `app/build.gradle`, you configure each compiler separately:

**Java compiler** — controlled by `compileOptions`:
```groovy
android {
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8  // Java targets JVM 1.8
        targetCompatibility JavaVersion.VERSION_1_8
    }
}
```

**Kotlin compiler** — controlled by `kotlinOptions`:
```groovy
android {
    kotlinOptions {
        jvmTarget = "17"  // Kotlin targets JVM 17
    }
}
```

The result:

| Compiler | Bytecode Target |
|----------|----------------|
| Java (`javac`) | `1.8` |
| Kotlin (`kotlinc`) | `17` |

❌ **These don't match → Gradle fails the build.**

> **Gradle 8+ is especially strict** about this. Older versions may have silently allowed mismatches, but modern Gradle enforces consistency.

---

## 🤔 Why You Might Never Have Seen This Before

React Native includes a Gradle plugin that **automatically aligns** the Java and Kotlin versions. This means even if your config is mismatched, RN quietly corrects it behind the scenes.

```
app/build.gradle (mismatched config)
        │
        ▼
React Native Gradle Plugin
  "Oh, Java is 1.8 and Kotlin is 17?
   Let me fix that silently..."
        │
        ▼
Build succeeds ✅ (but the misconfiguration still exists)
```

This is why many developers have mismatched configs without ever noticing — until something breaks the automatic alignment.

---

## 🧪 How to Reproduce It

To force the error and see it yourself:

**Step 1 — Disable RN's automatic alignment** in `gradle.properties`:

```properties
react.internal.disableJavaVersionAlignment=true
```

**Step 2 — Set mismatched versions** in `app/build.gradle`:

```groovy
android {
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }

    kotlinOptions {
        jvmTarget = "17"
    }
}
```

**Step 3 — Clean and rebuild**:

```bash
./gradlew clean
./gradlew assembleDebug
```

💥 The build will fail with the inconsistency error.

---

## ✅ How to Fix It

The fix is simple: **make both targets identical**.

The modern recommended approach is to align everything to **Java 17**:

```groovy
// app/build.gradle

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
```

Then rebuild:

```bash
./gradlew clean
./gradlew assembleDebug
```

✅ Build succeeds.

> **Pro tip:** Also make sure your **local JDK** is version 17. You can check with `java -version` in your terminal. A mismatch between your local JDK and your build config can cause subtle issues.

---

## 🔍 When Does This Typically Appear?

You're most likely to hit this error after:

- Migrating to a **newer Kotlin version**
- Upgrading the **Android Gradle Plugin (AGP)**
- Upgrading **Gradle** itself
- Installing **Java 17** locally (changes defaults)
- Disabling React Native's alignment (e.g., for debugging or CI)
- Applying **custom Gradle configurations** from third-party libraries

---

## 🏁 Key Takeaways

| # | Rule |
|---|------|
| 1 | Java and Kotlin **must** compile to the same JVM target |
| 2 | React Native **hides** this issue with automatic version alignment |
| 3 | Disabling alignment **exposes** real config mismatches |
| 4 | Gradle 8+ **enforces** strict JVM target validation |
| 5 | Always align `compileOptions`, `kotlinOptions.jvmTarget`, **and** your local JDK |

### Branch structure in this repository

| Branch | Purpose |
|--------|---------|
| `main` | ✅ Contains the **fix** (aligned versions) |
| `error` | ❌ Contains the **misconfiguration** (reproduces the failure) |

---

## 📌 Final Recommendation

For all modern React Native Android projects, use **Java 17 consistently** across your entire setup:

```
✅ compileOptions          → JavaVersion.VERSION_17
✅ kotlinOptions.jvmTarget → "17"
✅ Local JDK               → Java 17 (verify with: java -version)
```

Avoid mixing `1.8` and `17`. Consistency prevents subtle, hard-to-debug build failures.
