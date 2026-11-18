---
layout: post
title: "Software Testing in Java: A Practical Guide to JUnit 5"
date: 2026-05-20
description: A practical, example-driven guide to testing in Java — the testing landscape, JUnit 5 basics, assertions, parameterized tests, Mockito, and the TDD loop.
tags: [programming, java, testing, junit, software-engineering]
categories: [coding]
featured: false
---

I will be honest with you: for a long time I treated tests as something you write *after* the code is done — a boring step you rush through before pushing to the repo. If you ever thought the same, this post is for you.

At some point it clicked for me: **testing is not a chore you do after coding. It is part of coding itself.** This post is the guide I wish I had read back then — short on theory, long on examples.

## Why Bother Writing Tests?

Picture the typical "no tests" workflow: you change a method, run the app, click through a few screens, and convince yourself everything works. It feels fast. The problem is that it does not scale:

- **Manual checking gets slower over time.** Every new feature adds more things you should re-check by hand.
- **A well-written test catches a bug in milliseconds.** A missing test lets that bug reach production and your users.
- **Tests give you confidence.** When the suite is green, you can refactor and add features without the fear of silently breaking something.

The habit feels slower at first, then quietly makes everything faster.

## The Map: Four Levels of Testing

Testing happens at different levels. Knowing the map keeps you oriented:

| Level | What It Tests | Example | Who Runs It |
|---|---|---|---|
| **Unit** | One method or class, in isolation | "does `add(2, 3)` return 5?" | Developers |
| **Integration** | Components working together | "does `UserService` save via the repository correctly?" | Developers |
| **System** | The whole application against requirements | "can a user complete checkout end to end?" | QA team |
| **Acceptance** | Real user/business needs | "does this actually solve the user's problem?" | Stakeholders |

As a developer, you own the first two levels. That is where this post — and JUnit — live.

A few extra terms you will hear: **regression testing** (re-running the suite to catch anything broken by new changes), **smoke testing** (a quick sanity check on critical paths after a build), **performance testing** (behavior under load), and **exploratory testing** (unscripted, curiosity-driven probing). You do not need them daily, but recognizing the words helps.

## Setting Up JUnit 5

JUnit 5 is the standard testing framework for Java. One dependency and you are ready.

**Maven** (`pom.xml`):

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.10.2</version>
    <scope>test</scope>
</dependency>
```

**Gradle** (`build.gradle`):

```groovy
testImplementation 'org.junit.jupiter:junit-jupiter:5.10.2'
```

## Your First Test

Here is the class we want to test:

```java
public class Calculator {
    public int add(int a, int b)      { return a + b; }
    public int divide(int a, int b)   { return a / b; }
}
```

And here is its test class:

```java
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

class CalculatorTest {

    private Calculator calculator;

    @BeforeEach
    void setUp() {
        calculator = new Calculator();   // fresh object for every test
    }

    @Test
    @DisplayName("4 + 6 should equal 10")
    void testAdd() {
        assertEquals(10, calculator.add(4, 6));
    }

    @Test
    @DisplayName("Dividing by zero should throw ArithmeticException")
    void testDivideByZero() {
        assertThrows(ArithmeticException.class, () -> calculator.divide(10, 0));
    }
}
```

Reading it line by line:

- `@Test` marks a method as a test — you can have as many as you want.
- `@DisplayName` gives the test a human-readable name in the report.
- `assertEquals(expected, actual)` compares two values.
- `assertThrows(Exception.class, () -> ...)` checks that running that code throws the given exception.
- `@BeforeEach` runs **before every single test**, giving each one a fresh `Calculator`.

The golden rule behind `@BeforeEach`: **never share mutable state between tests.** Each test should start from a clean slate, so a failure always points at one specific problem.

## The Full Lifecycle

JUnit models the life of a test explicitly. The five annotations, in order:

| Annotation | When It Runs | Typical Use |
|---|---|---|
| `@BeforeAll` | Once, before all tests in the class | Expensive setup (start a server) |
| `@BeforeEach` | Before every test method | Fresh objects and state |
| `@Test` | The test itself | Your assertions |
| `@AfterEach` | After every test method | Cleanup |
| `@AfterAll` | Once, after all tests | Expensive teardown |

## The Assertions You Actually Need

Assertions are the vocabulary of tests. These six cover almost everything:

| Assertion | Question It Answers | Example |
|---|---|---|
| `assertEquals(x, y)` | Are these equal? | `assertEquals(10, calc.add(4, 6));` |
| `assertTrue(cond)` | Is this true? | `assertTrue(result > 0);` |
| `assertFalse(cond)` | Is this false? | `assertFalse(list.isEmpty());` |
| `assertNull(x)` / `assertNotNull(x)` | Is this null / not null? | `assertNotNull(user);` |
| `assertThrows(Ex.class, () -> ...)` | Does this throw? | `assertThrows(ArithmeticException.class, () -> calc.divide(1, 0));` |
| `assertAll(...)` | Do all of these checks hold? | see below |

One nuance worth knowing — plain `assertEquals` calls stop at the first failure, so you fix one thing at a time. `assertAll()` runs *every* assertion inside it and reports **all** failures at once:

```java
@Test
void testMultiply() {
    int result = calculator.multiply(3, 4);
    assertAll("multiply checks",
        () -> assertNotNull(result),
        () -> assertTrue(result > 0),
        () -> assertEquals(12, result)
    );
}
```

When a test fails, seeing all three broken expectations at once saves you two extra run-fix cycles.

## Parameterized Tests: One Test, Many Inputs

Copying and pasting a test method for every input combination gets old fast. `@ParameterizedTest` writes the logic once and feeds it data:

```java
@ParameterizedTest(name = "{0} + {1} = {2}")
@CsvSource({
    "1,  2,  3",
    "5,  5, 10",
    "-3, 3,  0",
    "0,  0,  0"
})
void testAdd(int a, int b, int expected) {
    assertEquals(expected, calculator.add(a, b));
}
```

The `@CsvSource` table maps each row to the method parameters: the first row runs `testAdd(1, 2, 3)`, the second `testAdd(5, 5, 10)`, and so on. Four test runs, one method.

For single-argument tests, `@ValueSource` is even simpler:

```java
@ParameterizedTest
@ValueSource(ints = {1, 2, 3, 100, -5})
void testAddZero(int number) {
    assertEquals(number, calculator.add(number, 0));
}
```

Rule of thumb: **the same logic with different data → parameterized test. Different logic → different test method.**

## Grouping with @Nested

When a test class grows, `@Nested` classes group related tests into a readable hierarchy:

```java
@DisplayName("Calculator")
class CalculatorNestedTest {

    private final Calculator calculator = new Calculator();

    @Nested
    @DisplayName("addition")
    class AdditionTests {
        @Test void positiveNumbers() { assertEquals(5, calculator.add(2, 3));  }
        @Test void negativeNumbers() { assertEquals(-5, calculator.add(-2, -3)); }
    }

    @Nested
    @DisplayName("division")
    class DivisionTests {
        @Test void dividesNumbers() { assertEquals(2, calculator.divide(10, 5)); }
        @Test void divideByZero()   { assertThrows(ArithmeticException.class,
                                                  () -> calculator.divide(5, 0)); }
    }
}
```

In the IDE's test tree this renders as *Calculator → addition → positiveNumbers* — it reads like a specification.

## Mocking with Mockito

Real classes have dependencies: a `UserService` needs a `UserRepository`. In a unit test you do not want a real database — so you replace the dependency with a **mock**: a fake object you fully control.

Mockito is the standard library for this:

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;  // fake repository

    @InjectMocks
    private UserService userService;        // real service, with the fake injected

    @Test
    void getUserById_returnsUser() {
        User user = new User(1, "Mario");

        // given: the mock returns Mario when asked for id 1
        when(userRepository.findById(1)).thenReturn(Optional.of(user));

        // when + then
        User result = userService.getUserById(1);

        assertEquals("Mario", result.getName());
        verify(userRepository, times(1)).findById(1);  // interaction was really used
    }
}
```

Four pieces to remember:

- `@Mock` creates the fake.
- `@InjectMocks` puts the fake into the real class under test.
- `when(...).thenReturn(...)` scripts the fake's behavior.
- `verify(...)` checks the fake was actually called — testing the *interaction*, not just the result.

## The TDD Loop

Everything above becomes even more powerful when you flip the order: write the test **before** the code. That is Test-Driven Development, and the loop is just three steps:

1. **Red** — write a test for behavior that does not exist yet; watch it fail.
2. **Green** — write the minimum code needed to make it pass.
3. **Refactor** — clean up freely, knowing the test protects you.

Writing the test first forces you to think about *what* the code should do before thinking *how*. You end up with smaller methods, clearer responsibilities, and a suite that grows with the code instead of being retrofitted later.

## Running Your Tests

- **IntelliJ / Eclipse**: right-click the test class (or single method) → *Run*.
- **Maven**: `mvn test` — report in `target/surefire-reports`.
- **Gradle**: `gradle test` — report in `build/reports/tests`.

## Cheat Sheet

| I want to... | Use |
|---|---|
| Check a value | `assertEquals(expected, actual)` |
| Check an exception | `assertThrows(Ex.class, () -> ...)` |
| Run many input combinations | `@ParameterizedTest` + `@CsvSource` / `@ValueSource` |
| Reset state before each test | `@BeforeEach` |
| See all failures at once | `assertAll(...)` |
| Group related tests | `@Nested` |
| Replace a dependency with a fake | `@Mock` + `@InjectMocks` + `when(...)` |
| Check a dependency was called | `verify(mock)` |
| Structure development | Red → Green → Refactor |

Writing tests is one of those habits that feels slower at first and then quietly makes everything faster. The codebase stays cleaner, refactoring stops being scary, and bugs get caught before anyone else sees them. Your future self — staring at a green pipeline at 11pm before a release — will be grateful.
