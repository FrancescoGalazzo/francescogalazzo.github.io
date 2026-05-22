---
layout: post
title: "Solving Exercism's Secret Handshake in Java"
date: 2026-05-22
categories: [Java, Exercism, coding]
tags: [java, gradle, intellij, exercism, algorithms, bit-manipulation, collections]
featured: false
related_posts: false
---

## Introduction

Exercism is a free, open-source platform that offers hundreds of coding exercises across dozens of programming languages. Each exercise is designed to push you toward idiomatic, clean code — not just a solution that passes the tests, but one that demonstrates an understanding of the language and its standard library. The Java track is particularly rich because it forces you to think about types, generics, and the JVM ecosystem from the very first exercise.

This post walks through the **Secret Handshake** exercise end-to-end: what the problem is, how the project is structured, how Gradle orchestrates the build and the tests, how to open everything correctly in IntelliJ IDEA, and finally how to arrive at a clean solution using binary operations and Java collections. Before the solution, there is a dedicated section on the theory of binary numbers — unsigned, signed, and two's complement — because understanding this foundation is what makes bitwise code feel natural rather than mysterious. The post closes with a rigorous analysis of time and space complexity.

***

## What Is the Problem?

The Secret Handshake exercise asks you to write a function that takes an integer and returns a list of actions — a "secret handshake" — based on the **binary representation** of that number. The five least-significant bits of the input each encode a specific meaning:

| Bit position | Decimal value | Binary mask | Action           |
|:---:|:---:|:---:|:---|
| 0 | 1  | `00001` | wink             |
| 1 | 2  | `00010` | double blink     |
| 2 | 4  | `00100` | close your eyes  |
| 3 | 8  | `01000` | jump             |
| 4 | 16 | `10000` | **reverse** the order of the other actions |

If bit 0 is set, you add `wink` to the list. If bit 1 is set, you add `double blink`, and so on. The first four bits build the list in a fixed order (wink → double blink → close your eyes → jump). The fifth bit is special: if it is set, you reverse the entire list before returning it.

A few examples to make this concrete:

- `1` → binary `00001` → only bit 0 is set → `[wink]`
- `3` → binary `00011` → bits 0 and 1 are set → `[wink, double blink]`
- `19` → binary `10011` → bits 0, 1, and 4 are set → start with `[wink, double blink]`, then reverse → `[double blink, wink]`
- `26` → binary `11010` → bits 1, 3, and 4 are set → start with `[double blink, jump]`, then reverse → `[jump, double blink]`

The elegance of this problem is that it maps perfectly onto bitwise AND operations — there is no need to parse strings, loop over digits, or use recursion. The processor already knows how to interrogate individual bits in constant time.

***

## Project Structure

When you download the exercise from Exercism or browse the [exercism/java](https://github.com/exercism/java) repository on GitHub, the directory for this exercise looks like this:

```
secret-handshake/
├── build.gradle                          ← Gradle build script
├── gradlew                               ← Unix Gradle Wrapper script
├── gradlew.bat                           ← Windows Gradle Wrapper script
├── gradle/
│   └── wrapper/
│       ├── gradle-wrapper.jar            ← Wrapper bootstrap JAR
│       └── gradle-wrapper.properties     ← Wrapper configuration (Gradle version)
└── src/
    ├── main/
    │   └── java/
    │       └── SecretHandshake.java      ← your implementation
    └── test/
        └── java/
            └── SecretHandshakeTest.java  ← provided tests (do not modify)
```

Notice that there is **no** `settings.gradle` file. This is intentional: the exercise is a single, self-contained module. There are no sub-projects, no module dependencies, and no Kotlin build scripts. It is as minimal as a Gradle project can be while still being fully functional.

Your only job is to fill in the body of `SecretHandshake.java`. The test file is given to you and you should never modify it.

***

## What Is Gradle and Why Does This Project Use It?

Gradle is an open-source **build automation tool** designed primarily for JVM-based projects. It handles three things that every non-trivial Java project needs: dependency management, compilation, and test execution.

Historically, Java projects used either **Ant** (which required you to write every step of the build by hand in XML) or **Maven** (which introduced a declarative, convention-based approach using a `pom.xml` file). **Gradle** combines the best of both worlds: it follows Maven's conventions (so you rarely need to configure anything), but it uses a Groovy or Kotlin DSL instead of XML, making build scripts dramatically more readable and flexible.

For Exercism exercises, Gradle serves two practical purposes. First, it downloads and caches the test dependencies (JUnit 5 and AssertJ) automatically from Maven Central, so you never have to manually manage JAR files. Second, it provides the `test` task, which compiles your solution alongside the test file and runs all the test cases, reporting which ones pass and which ones fail.

### The Gradle Wrapper

The `gradlew` script (and `gradlew.bat` on Windows) is called the **Gradle Wrapper**. Its purpose is to make the project self-contained with respect to the Gradle version: when you run `./gradlew test`, the wrapper reads `gradle/wrapper/gradle-wrapper.properties` to find out which Gradle version the project expects, downloads it automatically if it is not already on your machine, and then delegates to it. This means you do not need Gradle installed globally — just a working JDK and the wrapper scripts are enough.

### The `build.gradle` File

Here is the complete build script for this exercise:

```groovy
plugins {
    id "java"
}

repositories {
    mavenCentral()
}

dependencies {
    testImplementation platform("org.junit:junit-bom:5.10.0")
    testImplementation "org.junit.jupiter:junit-jupiter"
    testImplementation "org.assertj:assertj-core:3.25.1"

    testRuntimeOnly "org.junit.platform:junit-platform-launcher"
}

test {
    useJUnitPlatform()

    testLogging {
        exceptionFormat = "full"
        showStandardStreams = true
        events = ["passed", "failed", "skipped"]
    }
}
```

Let us walk through it section by section.

**`plugins { id "java" }`** — This applies the Java plugin, which teaches Gradle how to compile `.java` files, how to find source files in `src/main/java/` and test files in `src/test/java/`, and how to produce a compiled output. Without this plugin, Gradle would not know this is a Java project.

**`repositories { mavenCentral() }`** — This tells Gradle where to download dependencies from. Maven Central is the world's largest public repository of JVM libraries, hosted at `https://repo1.maven.org/maven2`. When you declare a dependency, Gradle will look it up here by its group ID, artifact ID, and version number.

**`dependencies { ... }`** — This block declares which external libraries the project needs. Notice that all four entries use either `testImplementation` or `testRuntimeOnly`, never `implementation`. This means these libraries are only available during compilation and execution of tests — they are never bundled into any production artifact. The production code (your solution) has **no external dependencies at all**.

The four dependency entries are:

- **`platform("org.junit:junit-bom:5.10.0")`** — A Bill of Materials (BOM). Instead of specifying the version of every JUnit 5 component individually, the BOM manages all versions as a consistent set. This prevents subtle incompatibility bugs between JUnit sub-modules.
- **`org.junit.jupiter:junit-jupiter`** — The JUnit 5 API. This is what provides `@Test`, `@DisplayName`, `@ParameterizedTest`, and all the annotations you see in the test file.
- **`org.assertj:assertj-core:3.25.1`** — AssertJ is a fluent assertion library. Instead of writing `assertEquals(expected, actual)`, AssertJ lets you write `assertThat(actual).containsExactly("wink", "double blink")`, which reads almost like English and produces much more informative error messages on failure.
- **`org.junit.platform:junit-platform-launcher`** (runtime only) — The launcher is the bridge between Gradle and JUnit 5. It discovers and executes tests at runtime. It is declared as `testRuntimeOnly` because it is not needed at compile time.

**`test { useJUnitPlatform() }`** — This tells the Gradle `test` task to use the JUnit Platform as its test discovery engine. Without this line, Gradle would default to the older JUnit 4 runner and would not be able to find or execute JUnit 5 tests.

***

## Opening the Project Correctly in IntelliJ IDEA

This is the step that causes the most confusion, and it is worth dwelling on. The golden rule is: **always open the exercise folder itself — never a parent folder.**

If you open a parent directory (for example, the entire `exercism/java` repository root), IntelliJ will detect the top-level `settings.gradle` that aggregates all exercises as Gradle sub-projects. IntelliJ then tries to run a Kotlin build script analysis task on every sub-project, including `secret-handshake`, which does not use Kotlin DSL. This produces the following misleading error:

```
Task 'prepareKotlinBuildScriptModel' not found in 
project ':practice:secret-handshake'.
BUILD FAILED
```

The fix is straightforward:

1. **File → Close Project** (if another project is already open)
2. **File → Open** → navigate to `secret-handshake/` and open that specific directory
3. IntelliJ detects `build.gradle` and shows a popup: *"Trust and open Gradle project?"* → click **Trust**
4. The IDE syncs the project, and Gradle downloads JUnit 5 and AssertJ in the background
5. After syncing, the Gradle tool window (View → Tool Windows → Gradle) shows the task tree with `verification → test` available

If you have already cloned the full repository and prefer not to copy the folder, you can create a minimal `settings.gradle` directly inside `secret-handshake/`:

```groovy
rootProject.name = 'secret-handshake'
```

Then, in the Gradle tool window, click the **+** icon (*Link Gradle project*) and point it to the `build.gradle` file inside `secret-handshake/`. IntelliJ will treat it as the root of a new, independent Gradle project and will not inherit the parent module structure.

***

## How to Approach This Type of Exercise

Before writing a single line of code, the most productive habit is to **read the test file carefully**. The test file is the specification. It tells you exactly what method signature you need to implement, what types are involved, what the expected output is for every meaningful input, and what edge cases the exercise author considered important.

For this exercise, the test file reveals:

- The class name must be `SecretHandshake` (not `Solution`, not `Main`)
- The method signature is something like `List<String> calculateHandshake(int number)` (or using a `Signal` enum if the test file imports one)
- The tests cover a variety of inputs, including `0` (empty list), single-bit numbers, multi-bit numbers, and numbers where bit 16 triggers a reversal

Once you understand the interface, the next step is to think about the **simplest possible representation** of the problem. Here, the problem is inherently binary. The input is a number; each bit is a yes/no flag. The output is a list of strings whose contents and order depend on which flags are active. There is no recursion, no graph traversal, no dynamic programming. The challenge is purely about reading bits correctly and building a list in the right order.

***

## Binary Numbers: Theory You Actually Need

Before diving into the Java-specific implementation, it is worth building a solid mental model of how binary numbers work — both in the unsigned (non-negative only) sense and in the signed two's complement representation that Java uses for all integer types. This section is self-contained: if you already know this material, skip ahead; if not, everything in the implementation section will make much more sense after reading it.

### The Positional Value System

The decimal system we use every day is a **base-10 positional system**: each digit position represents a power of 10. Reading from right to left, the positions represent 10⁰ = 1, 10¹ = 10, 10² = 100, and so on.

Binary is a **base-2 positional system**: each digit (called a *bit*) represents a power of 2. Reading from right to left, the positions represent 2⁰ = 1, 2¹ = 2, 2² = 4, 2³ = 8, 2⁴ = 16, and so on. A bit can only be `0` or `1`.

Converting from binary to decimal is a matter of summing the positional values for every bit that is `1`. For example:

```
Binary:  1  0  1  1  0
         │  │  │  │  └── 2⁰ = 1  × 0 =  0
         │  │  │  └───── 2¹ = 2  × 1 =  2
         │  │  └──────── 2² = 4  × 1 =  4
         │  └─────────── 2³ = 8  × 0 =  0
         └────────────── 2⁴ = 16 × 1 = 16
                                      ────
                                      22
```

So `10110` in binary equals `22` in decimal.

### How Many Values Can N Bits Represent?

This is one of the most useful formulas to internalize. With N bits, where each bit can be either `0` or `1`, and all combinations are independent, the total number of distinct patterns is:

```
2^N
```

This follows directly from the multiplication principle of combinatorics: two choices for the first bit, two for the second, two for the third, and so on — multiplied together gives 2 × 2 × … × 2 = 2^N.

| N bits | Distinct patterns (2^N) | Unsigned range   |
|:---:|:---:|:---|
| 1  | 2          | 0 to 1                     |
| 2  | 4          | 0 to 3                     |
| 3  | 8          | 0 to 7                     |
| 4  | 16         | 0 to 15                    |
| 5  | 32         | 0 to 31                    |
| 8  | 256        | 0 to 255                   |
| 16 | 65 536     | 0 to 65 535                |
| 32 | 4 294 967 296 | 0 to 4 294 967 295      |

For the Secret Handshake exercise, only 5 bits matter. Those 5 bits produce 2⁵ = **32 distinct patterns**, ranging from `00000` (= 0, no actions) to `11111` (= 31, all four actions reversed). Any bits beyond position 4 are irrelevant and should be masked away before processing.

### Unsigned vs. Signed Integers

An **unsigned** integer uses all N bits to represent magnitude. With N bits, unsigned integers cover the range from 0 to 2^N − 1.

A **signed** integer needs to represent negative values as well. The simplest scheme, **sign-magnitude**, uses the most significant bit (MSB) as a sign flag (0 = positive, 1 = negative) and the remaining N−1 bits for magnitude. However, sign-magnitude has two representations of zero (`+0` and `−0`), which complicates arithmetic in hardware.

Modern computers — and Java — use **two's complement** instead.

### Two's Complement

Two's complement is the standard representation for signed integers in virtually every modern architecture. It solves the double-zero problem and makes addition and subtraction work with the same hardware circuit regardless of sign.

In two's complement with N bits:

- The **most significant bit** (MSB, leftmost bit) has a **negative weight** of −2^(N−1), while all other bits keep their positive weights
- All N bits still represent 2^N distinct values, split across negative and positive numbers
- There is exactly **one representation of zero**

The formula for the value of a two's complement N-bit number is:

```
value = −b_(N−1) × 2^(N−1)  +  Σ (b_i × 2^i)  for i from 0 to N−2
```

where b_i is the value of bit i.

#### Range of Two's Complement

With N bits in two's complement:

- **Minimum value**: −2^(N−1)  (the bit pattern `100...0`, MSB = 1, all others = 0)
- **Maximum value**: 2^(N−1) − 1  (the bit pattern `011...1`, MSB = 0, all others = 1)

| N bits | Min value       | Max value        | Total range |
|:---:|---:|---:|---:|
| 4  | −8              | 7                | 16 values   |
| 8  | −128            | 127              | 256 values  |
| 16 | −32 768         | 32 767           | 65 536 values |
| 32 | −2 147 483 648  | 2 147 483 647    | ~4.3 billion values |
| 64 | −9.2 × 10^18    | 9.2 × 10^18      | ~1.8 × 10^19 values |

Notice that the total number of distinct values is always 2^N — the range is just shifted to include negative numbers.

#### A Concrete 4-bit Example

With 4 bits in two's complement, the MSB has weight −2³ = −8:

```
Bit pattern  MSB weight  +  remaining  =  value
 0000          0             0             0
 0001          0             1             1
 0010          0             2             2
 0011          0             3             3
 0100          0             4             4
 0101          0             5             5
 0110          0             6             6
 0111          0             7             7
 1000         -8             0            -8   ← most negative
 1001         -8             1            -7
 1010         -8             2            -6
 1011         -8             3            -5
 1100         -8             4            -4
 1101         -8             5            -3
 1110         -8             6            -2
 1111         -8             7            -1
```

Key observations: the pattern `0000` is zero, the pattern `1000` is the most negative value (−8), and the pattern `1111` is −1 (not 15 as it would be unsigned). Positive numbers have MSB = 0; negative numbers have MSB = 1.

#### How to Negate a Number in Two's Complement

To negate any N-bit two's complement value, there is a simple two-step procedure:

1. Flip all bits (bitwise NOT, operator `~` in Java)
2. Add 1

Example: negate `0011` (= 3):
```
Step 1 — flip all bits: 0011 → 1100
Step 2 — add 1:         1100 + 0001 = 1101
Result: 1101 = −8 + 4 + 1 = −3  ✓
```

This procedure works in both directions: applying it to a negative number gives the corresponding positive value.

#### Why One More Negative Than Positive?

There is always one more representable negative value than positive value in two's complement. This is because zero "occupies" one of the positive-side slots (the MSB = 0 half), leaving 2^(N−1) − 1 strictly positive values and 2^(N−1) negative values. The minimum value (−2^(N−1)) has no positive counterpart — attempting to negate it overflows back to itself, which is a classic source of bugs in programs that do `Math.abs(Integer.MIN_VALUE)` in Java.

### Java's Integer Types

Java has four signed integer primitive types, all using two's complement:

| Type   | Bits | Min value               | Max value              |
|:---|:---:|---:|---:|
| `byte`  | 8   | −128                    | 127                    |
| `short` | 16  | −32 768                 | 32 767                 |
| `int`   | 32  | −2 147 483 648          | 2 147 483 647          |
| `long`  | 64  | −9 223 372 036 854 775 808 | 9 223 372 036 854 775 807 |

Java has **no unsigned integer primitive**. If you need unsigned 32-bit arithmetic, you either use a `long` and mask the upper 32 bits, or use the static helper methods in `Integer` added in Java 8 (`Integer.toUnsignedString`, `Integer.compareUnsigned`, etc.).

For the Secret Handshake exercise, the input is a plain `int` (32-bit two's complement). Since the problem only cares about the 5 least-significant bits, the sign of the number is irrelevant after applying the mask `number & 0b1_1111`. That mask clears all bits above position 4, so the resulting `code` is always in the range [1] regardless of whether the original input was negative.

### Useful Bit-Count Formulas at a Glance

| Goal | Formula | Example (N=8) |
|:---|:---|:---|
| Total distinct patterns | 2^N | 256 |
| Unsigned range | 0 to 2^N − 1 | 0 to 255 |
| Two's complement range | −2^(N−1) to 2^(N−1) − 1 | −128 to 127 |
| Maximum positive value | 2^(N−1) − 1 | 127 |
| Minimum negative value | −2^(N−1) | −128 |
| Value of bit at position i | 2^i | bit 3 = 8 |
| Mask for the lowest N bits | 2^N − 1 (or `(1 << N) - 1`) | `0xFF` for N=8 |

***

## Binary Numbers and Bit Manipulation in Java

Before implementing the solution, it helps to understand how Java represents integers and how bitwise operators work.

### How Integers Are Stored

In Java, an `int` is a 32-bit signed two's complement integer. This means every `int` value occupies exactly 32 binary digits in memory. When we write the number `26`, the computer stores:

```
00000000 00000000 00000000 00011010
                              ^^^^^
                              ||||└── bit 0 = 0 (wink off)
                              |||└──── bit 1 = 1 (double blink ON)
                              ||└────── bit 2 = 0 (close your eyes off)
                              |└─────── bit 3 = 1 (jump ON)
                              └──────── bit 4 = 1 (reverse ON)
```

We only care about the five least-significant bits (bits 0 through 4), and we can ignore all the others.

### Java's Binary Literal Syntax

Java allows you to write integer literals in binary using the `0b` prefix:

```java
int mask = 0b00001;  // same as writing 1
int mask = 0b10000;  // same as writing 16
```

The binary literal is purely a readability feature — the compiler converts it to the same integer value as the decimal equivalent. However, when working with bit masks, writing `0b00001` instead of `1` makes the intent immediately obvious to anyone reading the code.

Java also allows underscores inside numeric literals for readability:

```java
int allFiveBits = 0b1_1111;  // same as 0b11111 = 31 = 2^5 - 1
```

### The Bitwise AND Operator (`&`)

The bitwise AND operator compares two integers **bit by bit**. For each position, the result is `1` only if **both** operands have a `1` in that position; otherwise the result is `0`.

```
  26 = 00011010
&  2 = 00000010   ← mask for "double blink" (bit 1)
─────────────────
       00000010   ← result is non-zero, so bit 1 IS set
```

```
  26 = 00011010
&  1 = 00000001   ← mask for "wink" (bit 0)
─────────────────
       00000000   ← result is zero, so bit 0 is NOT set
```

In Java code this looks like:

```java
if ((number & 0b00010) != 0) {
    actions.add("double blink");
}
```

The expression `(number & mask) != 0` is the idiomatic Java way to test whether a specific bit is set. It is a single CPU instruction and runs in O(1) time regardless of the value of `number`.

### Other Bitwise Operators (For Reference)

While this exercise only requires `&`, the full set of Java bitwise operators is worth knowing:

| Operator | Name | Behavior |
|:---:|:---|:---|
| `&` | AND | 1 only if both bits are 1 |
| `\|` | OR | 1 if at least one bit is 1 |
| `^` | XOR | 1 if the bits differ |
| `~` | NOT | flips all bits (unary) |
| `<<` | Left shift | shifts bits left, fills with 0 on the right |
| `>>` | Signed right shift | shifts bits right, fills with the sign bit on the left |
| `>>>` | Unsigned right shift | shifts bits right, always fills with 0 on the left |

The left-shift operator is particularly useful for building masks dynamically. Instead of writing `0b00100` to check bit 2, you can write `(1 << 2)`. More generally, `(1 << i)` produces a mask with exactly bit i set — a pattern you will see frequently in flag-based code.

### Isolating the Relevant Bits

As a defensive measure, it is good practice to mask the input to only the five bits we care about before doing anything else:

```java
int code = number & 0b1_1111;  // 0b11111 = 31 = 2^5 - 1, keeps only bits 0–4
```

This ensures that if the caller passes a number like `255` (`11111111`), the higher bits (`111` above bit 4) do not influence the result. After this one operation, `code` is guaranteed to be in the range `[0, 31]`, exactly the 32 distinct values representable in 5 unsigned bits.

***

## Java Collections: ArrayList and Collections.reverse

The solution builds the result as a `List<String>` (or `List<Signal>` if the exercise uses an enum). Java's `List` interface represents an ordered, index-accessible sequence of elements. For building a list incrementally by appending elements, the right implementation is `ArrayList`.

### Why ArrayList?

`ArrayList` is backed by a dynamically resizing array. Appending an element to the end (`add(element)`) is amortized O(1) — meaning that while an occasional resize operation takes O(n) time to copy all elements to a larger array, the average cost per append over many operations is constant. Since we are adding at most four elements, the resize never actually happens here, and each `add` is a simple O(1) array write.

The alternative, `LinkedList`, would also work, but it uses more memory (each node carries two pointers in addition to the value) and has worse cache locality. For small, fixed-size lists like this one, `ArrayList` is always the better choice.

### Reversing with Collections.reverse

Java's standard library provides `Collections.reverse(List<?>)`, which reverses a list **in place** — it modifies the existing list rather than creating a new one. For an `ArrayList`, this works by swapping elements from the two ends toward the center:

```
Before: [wink, double blink, jump]
Step 1: swap index 0 (wink) with index 2 (jump)   → [jump, double blink, wink]
Step 2: index 1 is the center, no swap needed
After:  [jump, double blink, wink]
```

This operation takes O(n) time, where n is the length of the list. Since our list has at most 4 elements, this is effectively O(1) in practice.

***

## The Solution

With all the background in place, the implementation is remarkably concise.

### `SecretHandshake.java`

```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

class SecretHandshake {

    List<String> calculateHandshake(int number) {
        List<String> actions = new ArrayList<>();
        int code = number & 0b1_1111;  // isolate bits 0–4 (range 0 to 31)

        if ((code & 0b00001) != 0) actions.add("wink");
        if ((code & 0b00010) != 0) actions.add("double blink");
        if ((code & 0b00100) != 0) actions.add("close your eyes");
        if ((code & 0b01000) != 0) actions.add("jump");
        if ((code & 0b10000) != 0) Collections.reverse(actions);

        return actions;
    }
}
```

### Tracing Through an Example

Let us manually trace `number = 26` step by step.

```
number = 26

Step 1 — isolate 5 bits (2^5 - 1 = 31 = 0b11111):
  code = 26 & 0b11111
       = 0b11010 & 0b11111
       = 0b11010
       = 26  (already within [0, 31], no change)

Step 2 — check bit 0 (wink):
  code & 0b00001 = 0b11010 & 0b00001 = 0b00000 = 0  → skip

Step 3 — check bit 1 (double blink):
  code & 0b00010 = 0b11010 & 0b00010 = 0b00010 ≠ 0  → add "double blink"
  actions = ["double blink"]

Step 4 — check bit 2 (close your eyes):
  code & 0b00100 = 0b11010 & 0b00100 = 0b00000 = 0  → skip

Step 5 — check bit 3 (jump):
  code & 0b01000 = 0b11010 & 0b01000 = 0b01000 ≠ 0  → add "jump"
  actions = ["double blink", "jump"]

Step 6 — check bit 4 (reverse):
  code & 0b10000 = 0b11010 & 0b10000 = 0b10000 ≠ 0  → reverse!
  actions = ["jump", "double blink"]

Result: ["jump", "double blink"] ✓
```

### The Variant with an Enum

If the test file imports and uses a `Signal` enum instead of strings, you would define a companion file:

```java
public enum Signal {
    WINK,
    DOUBLE_BLINK,
    CLOSE_YOUR_EYES,
    JUMP
}
```

And update the method accordingly:

```java
List<Signal> calculateHandshake(int number) {
    List<Signal> signals = new ArrayList<>();
    int code = number & 0b1_1111;

    if ((code & 0b00001) != 0) signals.add(Signal.WINK);
    if ((code & 0b00010) != 0) signals.add(Signal.DOUBLE_BLINK);
    if ((code & 0b00100) != 0) signals.add(Signal.CLOSE_YOUR_EYES);
    if ((code & 0b01000) != 0) signals.add(Signal.JUMP);
    if ((code & 0b10000) != 0) Collections.reverse(signals);

    return signals;
}
```

The logic is identical — only the element type changes.

***

## Complexity Analysis

Understanding the complexity of a solution is not just an academic exercise. It tells you how the solution will behave under different inputs and whether it can be relied upon in contexts where performance matters.

### Time Complexity

The method performs the following operations:

1. One bitwise AND to isolate 5 bits: **O(1)**
2. Four conditional bitwise AND checks: **O(1)** each, **O(1)** total (there are always exactly 4 checks, regardless of input)
3. At most four `ArrayList.add()` calls: **O(1)** amortized each, **O(1)** total
4. One optional `Collections.reverse()` call on a list of at most 4 elements: **O(1)** (constant input size)

**Overall time complexity: O(1)**

The entire method executes in constant time. There are no loops, no recursion, and no operations whose cost scales with the value of the input. Whether `number` is `1` or `2,147,483,647` (Integer.MAX_VALUE), exactly the same number of instructions are executed.

This is the best possible time complexity for a function — it cannot be improved further.

### Space Complexity

The method allocates:

- One `ArrayList` to hold the results: at most **4 elements**, so O(1) extra space
- One `int` variable `code`: O(1)
- No recursive stack frames, no auxiliary data structures

**Overall space complexity: O(1)**

The output list itself is returned to the caller, so it is not counted as "extra" space in the standard definition of auxiliary space complexity. Even if we count the output, the list is bounded by a constant size of 4, so it remains O(1) in all cases.

### Why Constant Complexity Is Possible Here

The reason both time and space complexity are O(1) is that **the problem has a fixed, bounded universe**. There are exactly 5 relevant bits, exactly 4 possible actions, and the reversal either happens or it does not. The problem does not scale with input size — there is no array to sort, no graph to traverse, no string to parse of variable length. The input is a single integer, and the output is a list bounded by a small constant.

This stands in contrast to a naive alternative approach that might convert the integer to its binary string representation and parse each character. That approach would still be O(1) in theory (since the string is at most 32 characters for a Java `int`), but it would be far slower in practice due to string allocation, `Integer.toBinaryString()` overhead, and character-by-character parsing. The bitwise solution avoids all of that and maps each check directly to a single CPU instruction.

***

## Running the Tests

From IntelliJ, right-click `SecretHandshakeTest.java` in the Project panel and select **Run 'SecretHandshakeTest'**. The test runner opens in the bottom panel and shows a green tree when all tests pass.

From the terminal, inside the `secret-handshake/` directory:

```bash
./gradlew test
```

A successful run produces output similar to:

```
> Task :test

SecretHandshakeTest > wink_for_1() PASSED
SecretHandshakeTest > double_blink_for_2() PASSED
SecretHandshakeTest > close_your_eyes_for_4() PASSED
SecretHandshakeTest > jump_for_16() PASSED
SecretHandshakeTest > combine_two_actions() PASSED
SecretHandshakeTest > reverse_two_actions() PASSED
SecretHandshakeTest > reversing_one_action_gives_the_same_action() PASSED
SecretHandshakeTest > reversing_no_actions_still_gives_no_actions() PASSED
SecretHandshakeTest > all_possible_actions() PASSED
...

BUILD SUCCESSFUL in 3s
2 actionable tasks: 2 executed
```

***

## Key Takeaways

Working through this exercise illustrates several principles that apply well beyond Exercism.

The most important is that **choosing the right representation makes the code trivial**. The number `26` looks arbitrary as a decimal. As a binary value, its meaning is self-evident once you know the encoding scheme. Recognizing when a problem maps naturally onto bit flags is a skill that takes practice, but it pays dividends in embedded systems, networking, operating systems, and performance-sensitive application code.

Understanding **two's complement and the 2^N formula** is foundational. It explains why Java's `int` goes up to 2 147 483 647, why there is one more negative value than positive value, and why masking with `0b1_1111` (= 2⁵ − 1 = 31) reliably isolates exactly five bits. These are not arbitrary facts — they follow directly from the mathematics of positional binary representation.

The second engineering takeaway is that **the standard library is your friend**. `ArrayList` and `Collections.reverse()` are not exotic choices — they are the right tools for the job, and they come with well-understood performance characteristics. Knowing when to reach for `ArrayList` versus `LinkedList`, when `Collections.reverse()` is in-place versus when you need a new list, and how the BOM pattern in Gradle prevents dependency version conflicts — these details compound into real engineering competence over time.

Finally, the project setup itself is instructive. A minimal Gradle project needs exactly one `build.gradle`, a Gradle Wrapper, and a conventional `src/` directory layout. Adding complexity beyond that — extra modules, nested `settings.gradle` files, plugin configurations — is a choice that should be driven by a real need, not by habit or template copying.