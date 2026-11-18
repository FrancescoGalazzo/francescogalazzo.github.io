---
layout: post
title: "Solving Exercism's Secret Handshake in Java"
date: 2026-05-22
description: "A compact walkthrough of the Secret Handshake exercise: how to read bits, build the answer, and why the whole thing runs in constant time."
categories: [Java, Exercism, coding]
tags: [java, exercism, algorithms, bit-manipulation]

featured: false
related_posts: false
---

# The Problem

[Exercism' Secret Handshake](https://exercism.org/tracks/java/exercises/secret-handshake)  exercise looks mysterious at first: you receive an integer, and you must return a list of handshake *actions*. The trick is that the answer is hidden in the **binary representation** of the number, not in the number itself. Here is my [code](https://github.com/FrancescoGalazzo/secret-handshake).

The five least-significant bits each have a meaning:

| Bit | Value | Action |
|:---:|:---:|:---|
| 0 | 1 | wink |
| 1 | 2 | double blink |
| 2 | 4 | close your eyes |
| 3 | 8 | jump |
| 4 | 16 | **reverse** the list |

The first four bits add their action to the list, in that fixed order. The fifth bit does not add anything: if it is set, the final list gets reversed.

A few examples to make it concrete:

- `1` → `00001` → bit 0 set → `[wink]`
- `3` → `00011` → bits 0 and 1 → `[wink, double blink]`
- `19` → `10011` → bits 0, 1, 4 → `[double blink, wink]` (reversed)
- `26` → `11010` → bits 1, 3, 4 → `[jump, double blink]` (reversed)

## My Approach

The key insight is recognizing what kind of problem this is. Each bit is simply a **yes/no flag**: "is this action in the list or not?" There is no arithmetic, no recursion, no parsing. Once you see the input as five independent flags, the solution almost writes itself.

The approach is three steps:

1. **Mask the input** to keep only the five bits we care about. Anything above bit 4 is irrelevant, so I clear it away immediately with `number & 0b1_1111` (that is 31, i.e. 2⁵ − 1). This also guarantees the value is between 0 and 31 no matter what integer the caller passes.
2. **Test each bit** with the bitwise AND. The expression `(code & 0b00010) != 0` asks a single, precise question: "is bit 1 set?" If yes, add the matching action to the list.
3. **Reverse if bit 4 is set**, using the standard library's `Collections.reverse()`.

Why `(code & mask) != 0` works: AND compares two numbers bit by bit and keeps a 1 only where *both* have one. So AND-ing with a mask that has a single 1 isolates exactly that bit — the result is non-zero only if the bit is set:

```
  26 = 11010
&  2 = 00010   ← mask for "double blink"
──────────────
       00010   ← non-zero: bit 1 IS set

  26 = 11010
&  1 = 00001   ← mask for "wink"
──────────────
       00000   ← zero: bit 0 is NOT set
```

## The Solution

The exercise expects a `HandshakeCalculator` class and a `Signal` enum. I like keeping the actions in an enum rather than raw strings — the compiler then guarantees there are no typos and no invalid values:

```java
enum Signal {
    WINK, DOUBLE_BLINK, CLOSE_YOUR_EYES, JUMP
}
```

```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

class HandshakeCalculator {

    List<Signal> calculateHandshake(int number) {
        List<Signal> signals = new ArrayList<>();
        int code = number & 0b1_1111;

        if ((code & 0b00001) != 0) {
            signals.add(Signal.WINK);
        }
        if ((code & 0b00010) != 0) {
            signals.add(Signal.DOUBLE_BLINK);
        }
        if ((code & 0b00100) != 0) {
            signals.add(Signal.CLOSE_YOUR_EYES);
        }
        if ((code & 0b01000) != 0) {
            signals.add(Signal.JUMP);
        }
        if ((code & 0b10000) != 0) {
            Collections.reverse(signals);
        }

        return signals;
    }
}
```

Tracing `26` through it: `26 & 31` is still `26` (`11010`). Bit 0 → off, skip. Bit 1 → set, add `DOUBLE_BLINK`. Bit 2 → off, skip. Bit 3 → set, add `JUMP`. Bit 4 → set, so reverse: `[JUMP, DOUBLE_BLINK]`. Done.

## Complexity

**Time: O(1).** The method does one masking AND, four fixed bit checks, at most four list additions, and possibly one reversal of at most four elements. That is the *same* amount of work whether the input is `1` or `2,147,483,647`. There are no loops over the input and nothing that grows with its value.

**Space: O(1).** The result list holds at most four elements regardless of the input, so the extra memory is bounded by a constant.

Constant time and space is possible here because the problem has a **fixed, bounded universe**: exactly five relevant bits and four possible actions. There is nothing to scale. A tempting alternative — converting the number to a binary *string* and reading characters — would also pass the tests, but it hides the intent behind string allocations and character parsing, while each bitwise check here maps to a single CPU instruction.

## Takeaways

- The right representation makes code trivial: `26` looks arbitrary, `11010` tells the whole story.
- `(value & (1 << i)) != 0` is the idiomatic bit test in Java — worth knowing by heart.
- `Collections.reverse()` reverses in place, in O(n), and the standard library is the right place to look before writing your own.
- When a problem has a fixed number of flags, expect O(1): recognizing *why* is more valuable than the solution itself.
