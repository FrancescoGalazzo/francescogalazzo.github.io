---
layout: post
title: "Software Testing in Java: Why You Should Just Write the Tests"
date: 2026-05-20
description: A personal take on why testing should be a natural habit when writing code, and a detailed guide on how to do it in Java with JUnit 5.
tags: programming java testing junit software-engineering
categories: coding
featured: false
---

Let me be honest with you: for a long time, I treated tests as something you write *after* the code is done — a bureaucratic step you rush through before pushing to the repo. If you've ever thought the same, this post is for you. Because at some point it clicked: **testing isn't a chore you do after coding, it's part of coding itself.**

I think the root of the problem is how testing is often taught. It shows up at the end of a course, in a chapter that half the students skip. It's presented as a separate discipline — something QA people do, not developers. But that framing is wrong, and it costs teams an enormous amount of time and stress.

## The Real Cost of Not Testing

Think about the last time you changed a method, ran the app, manually clicked through a few screens, and convinced yourself everything was fine. We've all been there. It feels efficient in the moment. The problem is that manual verification doesn't scale. As the codebase grows, the surface area of things that can silently break when you touch something grows with it — and your ability to check everything manually shrinks.

A well-placed test catches a regression in milliseconds. A missing test lets it reach production, and reach your users. Beyond that, testing gives you something less tangible but equally valuable: **confidence**. When your test suite is green, you can refactor aggressively, add features without anxiety, and merge pull requests without that nagging feeling that you've broken something somewhere. That peace of mind is not a luxury — it's what allows a codebase to keep moving fast as it matures.

## What Software Testing Actually Is

At its core, software testing is the process of evaluating and verifying that a software product functions correctly, securely, and as intended. It splits into two broad families. The first is **dynamic testing** — actually running the software and checking its outputs against expected results. The second is **static testing** — reviewing code, documentation, or design artifacts without executing them. Think code reviews, linting, and static analysis tools like SpotBugs or SonarQube.

Both approaches are useful and complementary. But as developers writing application code day to day, dynamic testing — and unit testing in particular — is where most of our leverage is. It's fast, it's local, and it runs on every build.

## The Four Levels of Testing

Testing doesn't happen at a single level. It unfolds across a spectrum of abstraction, each level targeting a different scope of the system.

**Unit testing** is the most granular. It tests individual methods and classes in complete isolation from the rest of the system. This is the developer's primary responsibility. **Integration testing** comes next: it verifies that separate components work correctly when combined — that your service talks to your repository as expected, that your API layer correctly calls your business logic. **System testing** then validates the entire application against its functional requirements, typically owned by a QA team. Finally, **acceptance testing** answers the highest-level question of all: does the software actually meet the real needs of its users and stakeholders?

| Level | What It Tests | Who Typically Runs It |
|---|---|---|
| **Unit Testing** | Individual methods and classes in isolation | Developers |
| **Integration Testing** | Interaction between combined components | Developers / QA |
| **System Testing** | The entire application against requirements | QA team |
| **Acceptance Testing** | Whether the software meets real user/business needs | Stakeholders / Clients |

As developers, we own the bottom two levels. Understanding the upper levels helps us write better unit and integration tests that don't leave dangerous blind spots.

## Other Testing Types Worth Knowing

Beyond those four levels, a few specialized categories come up regularly in real projects and are worth having in your vocabulary.

**Regression testing** confirms that new changes haven't broken existing functionality. It's the bedrock of CI/CD pipelines — the suite that runs automatically on every push. **Smoke testing** is a quick sanity check that the most critical paths still work after a new build, before investing time in deeper testing. **Performance testing** evaluates how the system behaves under load — response times, throughput, and stability under stress. **Security testing** proactively identifies vulnerabilities before attackers find them. And **exploratory testing** is the unscripted, curiosity-driven kind, where a tester actively looks for unexpected behaviour with no predefined script to follow.

You won't always need all of these, but knowing they exist means you'll recognize when to reach for them.

## JUnit 5 in Practice

Now let's get concrete. **JUnit 5** — also known as JUnit Jupiter — is the standard unit testing framework for Java. It's mature, expressive, and integrates natively with Maven, Gradle, IntelliJ IDEA, and Eclipse. If you write Java professionally, you will use it.

### Setting Up

For **Maven**, add this to your `pom.xml`:

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.10.2</version>
    <scope>test</scope>
</dependency>
```

For **Gradle**:

```groovy
testImplementation 'org.junit.jupiter:junit-jupiter:5.10.2'
```

### The Test Lifecycle

One of the things I appreciate most about JUnit 5 is how explicitly it models the lifecycle of a test. Before any test runs, you can set up state. After each test, you can clean it up. The annotations that control this are:

| Annotation | When It Runs |
|---|---|
| `@BeforeAll` | Once before all tests in the class |
| `@BeforeEach` | Before every single test method |
| `@Test` | The test itself |
| `@AfterEach` | After every single test method |
| `@AfterAll` | Once after all tests in the class |

The golden rule is: **never share mutable state between tests**. Each test should start with a clean slate. `@BeforeEach` is the tool for that:

```java
class OrderServiceTest {

    private OrderService orderService;

    @BeforeEach
    void setUp() {
        orderService = new OrderService(); // fresh instance for every test
    }

    @AfterEach
    void tearDown() {
        // cleanup if needed (e.g. close resources)
    }
}
```

### Basic Tests and Assertions

Let's start with something simple — a `Calculator` class:

```java
// Calculator.java
public class Calculator {
    public int add(int a, int b)        { return a + b; }
    public int subtract(int a, int b)   { return a - b; }
    public int multiply(int a, int b)   { return a * b; }
    public int divide(int a, int b)     { return a / b; }
}
```

```java
// CalculatorTest.java
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

class CalculatorTest {

    private Calculator calculator;

    @BeforeEach
    void setUp() {
        calculator = new Calculator();
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

    @Test
    @DisplayName("Multiply result should be positive and correct")
    void testMultiplyAssertAll() {
        int result = calculator.multiply(3, 4);
        assertAll("multiply checks",
            () -> assertNotNull(result),
            () -> assertTrue(result > 0),
            () -> assertEquals(12, result)
        );
    }
}
```

Notice `assertAll()` in that last test. Unlike stacking multiple `assertEquals` calls, `assertAll()` runs every assertion inside it and reports *all* failures at once, rather than stopping at the first. When debugging a failing test, that difference is huge.

### Parameterized Tests

This is where JUnit 5 really earns its keep. Rather than duplicating a test method for every input combination you want to cover, `@ParameterizedTest` lets you express the same logic once and feed it a table of data:

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;
import org.junit.jupiter.params.provider.ValueSource;
import static org.junit.jupiter.api.Assertions.*;

class CalculatorParameterizedTest {

    private final Calculator calculator = new Calculator();

    @ParameterizedTest
    @ValueSource(ints = {1, 2, 3, 100, -5})
    @DisplayName("Adding zero should always return the same number")
    void testAddZero(int number) {
        assertEquals(number, calculator.add(number, 0));
    }

    @ParameterizedTest(name = "{0} + {1} = {2}")
    @CsvSource({
        "1,  2,  3",
        "5,  5, 10",
        "-3, 3,  0",
        "0,  0,  0"
    })
    void testAddWithMultipleInputs(int a, int b, int expected) {
        assertEquals(expected, calculator.add(a, b));
    }
}
```

`@ValueSource` handles single-argument tests cleanly. `@CsvSource` is perfect for multi-column tabular data, and the `name` attribute even formats the test name in the output, making failures immediately readable. For more complex objects, `@MethodSource` lets you supply a `Stream<Arguments>` from a static factory method.

### Nested Tests

`@Nested` classes let you group related tests under a shared context, giving your test file a logical hierarchy that mirrors your domain:

```java
@DisplayName("Calculator Tests")
class CalculatorNestedTest {

    private Calculator calculator = new Calculator();

    @Nested
    @DisplayName("Addition")
    class AdditionTests {
        @Test void addsPositiveNumbers()  { assertEquals(5,  calculator.add(2, 3));   }
        @Test void addsNegativeNumbers()  { assertEquals(-5, calculator.add(-2, -3)); }
        @Test void addsMixedNumbers()     { assertEquals(1,  calculator.add(4, -3));  }
    }

    @Nested
    @DisplayName("Division")
    class DivisionTests {
        @Test void dividesTwoNumbers()    { assertEquals(2, calculator.divide(10, 5)); }
        @Test void throwsOnDivideByZero() {
            assertThrows(ArithmeticException.class, () -> calculator.divide(5, 0));
        }
    }
}
```

When these run in your IDE, the test results tree mirrors the nested structure exactly. It reads almost like a specification document, which is precisely the point.

### Mocking with Mockito

Real-world classes rarely live in isolation. A `UserService` depends on a `UserRepository`. A `PaymentService` calls an external gateway. You don't want your unit test to spin up a real database or make real HTTP calls — so you *mock* the dependency, replacing it with a controlled fake that you can instruct to return specific values.

Mockito is the standard library for this in Java. Add it alongside JUnit 5:

```xml
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-junit-jupiter</artifactId>
    <version>5.12.0</version>
    <scope>test</scope>
</dependency>
```

Then wire it in with `@ExtendWith`:

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import java.util.Optional;
import static org.mockito.Mockito.*;
import static org.junit.jupiter.api.Assertions.*;

@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;  // fake, fully controlled by us

    @InjectMocks
    private UserService userService;        // the class under test

    @Test
    @DisplayName("Should return user when found by ID")
    void testGetUserById() {
        User user = new User(1, "Mario");
        when(userRepository.findById(1)).thenReturn(Optional.of(user));

        User result = userService.getUserById(1);

        assertEquals("Mario", result.getName());
        verify(userRepository, times(1)).findById(1);
    }

    @Test
    @DisplayName("Should throw exception when user not found")
    void testGetUserByIdNotFound() {
        when(userRepository.findById(99)).thenReturn(Optional.empty());
        assertThrows(UserNotFoundException.class, () -> userService.getUserById(99));
    }
}
```

`@Mock` creates a fake `UserRepository`. `@InjectMocks` automatically wires it into `UserService`. `when(...).thenReturn(...)` defines the fake's behaviour for a specific input. And `verify(...)` asserts that the mock was actually called the expected number of times — so you're not just checking the return value, you're checking the interaction itself.

## The TDD Mindset

All of the above becomes even more powerful when you flip the order: write the test *before* the code. This is **Test-Driven Development (TDD)**, and it follows a short loop:

1. **Red** — write a test for behaviour that doesn't exist yet; watch it fail
2. **Green** — write the *minimum* code to make it pass
3. **Refactor** — clean up the code freely, knowing the test has your back

Writing the test first forces you to think about the interface and behaviour of your code before you think about the implementation. You end up with smaller, more focused methods, clearer responsibilities, and a test suite that grows naturally alongside your production code rather than being retrofitted after the fact.

## Running Your Tests

- **IntelliJ IDEA / Eclipse**: right-click the test class or method → *Run*
- **Maven**: `mvn test`
- **Gradle**: `gradle test`

Your build tool will generate a report under `target/surefire-reports` (Maven) or `build/reports/tests` (Gradle) with a full breakdown of passed, failed, and skipped tests.

---

Writing tests is one of those habits that feels slightly slower at first and then, quietly, makes everything faster. The codebase stays cleaner, refactoring becomes less scary, and bugs get caught before users ever see them. Your future self — staring at a green CI pipeline at 11pm before a release — will be grateful.