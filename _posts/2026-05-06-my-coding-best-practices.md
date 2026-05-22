---
layout: post
title: My Coding Best Practices
date: 2026-05-14
description: The principles, habits, and tools I follow every time I write code — language-agnostic and built from real-world experience.
tags: programming best-practices clean-code software-engineering
categories: coding
featured: false
related_posts: false
---

Every developer, at some point, opens a file they wrote six months ago and wonders what they were thinking. The code works — technically — but it's hard to read, harder to change, and terrifying to touch. I've been there too.

Over time, I built a set of practices that I apply every time I sit down to write code. They're not rules I follow blindly — they're habits I developed by making mistakes, reading other people's code, and slowly learning what makes a codebase a pleasure to work in rather than a burden. This post is my attempt to write them all down in one place.

None of these are language-specific. Whether you write Java, Python, or JavaScript, the underlying principles are the same.

---

## 1. Use a Proper IDE

Before writing a single line of code, set yourself up with the right environment. An **IDE (Integrated Development Environment)** is not just a text editor — it's a tool that understands your code. It autocompletes method names and imports as you type, highlights errors before you even run the program, lets you rename a variable across the entire codebase in one operation, and integrates with Git so you never have to leave the editor to commit or review changes.

That removed friction adds up. The less time you spend fighting your tools, the more mental space you have for logic and design — which is where the real work happens.

Here are the tools worth knowing:

| Language | IDE | Notes |
|---|---|---|
| Any | [VS Code](https://code.visualstudio.com/) | Free, open source, highly extensible via plugins |
| Java | [IntelliJ IDEA Community](https://www.jetbrains.com/idea/) | Free, best-in-class Java support |
| Python | [PyCharm Community](https://www.jetbrains.com/pycharm/) | Free, excellent Python tooling |
| Any | [Eclipse](https://www.eclipse.org/) | Free, open source, mature ecosystem |
| Any | [Neovim](https://neovim.io/) | Free, keyboard-driven, highly configurable |

If you use VS Code, three extensions are worth installing from day one:
- [GitLens](https://marketplace.visualstudio.com/items?itemName=eamodio.gitlens) — rich Git history and inline blame annotations
- [SonarLint](https://www.sonarsource.com/products/sonarlint/) — real-time code quality and security analysis
- [Error Lens](https://marketplace.visualstudio.com/items?itemName=usernamehw.errorlens) — displays errors and warnings inline, right next to the code

---

## 2. Write Code for Humans First

The machine doesn't care what you name your variables. Your colleagues — and your future self — very much do. Code is read far more often than it is written, so always optimize for the reader.

```java
// Hard to read — what does this do?
public double calc(double a, int b, double c) {
    return a * b * (1 - c);
}

// Clear — reads almost like a sentence
public double calculateFinalPrice(double unitPrice, int quantity, double discountRate) {
    return unitPrice * quantity * (1 - discountRate);
}
```

The second version needs no comment to be understood. That clarity is a choice, and it's one worth making every time.

A few rules of thumb to keep in mind:
- Variable names should explain what they store, not how (`userList`, not `data`)
- Method names should describe what they do (`sendWelcomeEmail()`, not `process()`)
- Keep methods short — if a method doesn't fit on one screen, it probably does too much

Tools like [SonarQube Community Edition](https://www.sonarsource.com/open-source/), [PMD](https://pmd.github.io/), and [Checkstyle](https://checkstyle.sourceforge.io/) can help enforce these standards automatically across a Java project.

---

## 3. The SOLID Principles

As a codebase grows, it tends to rot in predictable ways: classes that accumulate responsibilities over time, code that breaks when you try to extend it, dependencies that are impossible to swap out for testing. **SOLID** is a set of five design principles that address exactly these failure modes.

| Principle | What It Means | Practical Example |
|---|---|---|
| **S**ingle Responsibility | A class should do one thing only | `UserValidator` validates; it does not save to the database |
| **O**pen/Closed | Open for extension, closed for modification | Add new behaviour via new classes, not by editing existing ones |
| **L**iskov Substitution | Subclasses must be usable wherever their parent is used | A `Square` subclass should not break code written for a `Rectangle` |
| **I**nterface Segregation | Prefer many small, focused interfaces over one large one | `Printable` and `Serializable` as separate interfaces, not one `Utility` |
| **D**ependency Inversion | Depend on abstractions, not concrete implementations | Inject a `DatabaseInterface`, not a specific `MySQLDatabase` |

Each of these principles deserves more space than a table can give them. I'll go deeper into each one with practical Java examples in a dedicated follow-up post.

---

## 4. Don't Repeat Yourself (DRY)

If the same logic appears in two places, it will eventually be updated in one and forgotten in the other. This is one of the most common sources of subtle, hard-to-trace bugs.

```java
// Bad — the same logic duplicated in two different places
String email = input.trim().toLowerCase(); // in the registration flow
String email = input.trim().toLowerCase(); // in the password reset flow

// Good — extracted once, reused everywhere
private String normalizeEmail(String input) {
    return input.trim().toLowerCase();
}
```

Every piece of logic should have a single, authoritative home in the codebase. When it needs to change — and it will — you change it in exactly one place and can be confident the fix propagates everywhere it needs to.

---

## 5. Comment Why, Not What

The code already shows *what* it does. A good comment explains *why* a non-obvious decision was made. Commenting obvious things creates noise that drowns out the comments that actually matter.

```java
// Bad — the code already says this
i = i + 1; // increment i

// Good — explains a non-obvious constraint
i += 1; // offset by 1 because the external API uses 1-based indexing
```

Three rules to follow:
- Comment on business rules, edge cases, and non-obvious algorithmic choices
- Never comment out dead code — delete it. Git keeps the history
- Always update a comment when you update the code it describes. A wrong comment is worse than no comment

---

## 6. Handle Errors Explicitly

Silent failures are a debugging nightmare. When something goes wrong and nothing is logged, you have no idea where to start looking.

```java
// Bad — the error is swallowed silently
try {
    result = fetchUserData(userId);
} catch (Exception e) {
    // nothing
}

// Good — caught specifically, logged with context, re-thrown meaningfully
try {
    result = fetchUserData(userId);
} catch (NetworkException e) {
    logger.error("Failed to fetch user {}: {}", userId, e.getMessage(), e);
    throw new ServiceUnavailableException("User data temporarily unavailable", e);
}
```

Key habits:
- Always catch the most specific exception type you can — a bare `catch (Exception e)` hides bugs
- Log errors with enough context to reconstruct what happened: include IDs, inputs, and the original exception
- Never expose raw stack traces to end users
- Define custom exception classes to distinguish different failure modes from each other

---

## 7. Use Git — and Use It Well

Git is not just a backup tool. It is a communication tool. Your commit history tells the story of your project and lets anyone — including future you — understand *why* a decision was made months after the fact.

```bash
# Bad — tells you nothing
git commit -m "fix stuff"

# Good — tells you exactly what changed and why
git commit -m "fix: handle null user in authentication flow"
```

[Conventional Commits](https://www.conventionalcommits.org/) is a lightweight standard worth adopting. The prefixes you'll use most often are `feat:` for new features, `fix:` for bug fixes, `refactor:` for code restructuring without behaviour change, `docs:` for documentation updates, and `test:` for anything test-related.

### Pull Requests

A **Pull Request (PR)** is a formal proposal to merge a branch of code into another — usually `main`. When you open one, you give teammates the chance to review your changes line by line, ask questions, and run automated checks via CI/CD before anything reaches the main branch. Even on solo projects, opening a PR and reviewing your own diff before merging catches a surprising number of obvious mistakes.

Keep PRs small and focused — 200 to 400 lines is a good target. Large PRs don't get reviewed carefully; they get rubber-stamped. And three non-negotiable rules: never commit directly to `main`, never commit secrets or passwords, and always add sensitive files to `.gitignore`.

Two tools worth adding to your workflow:
- [pre-commit](https://pre-commit.com/) — run linting, formatting, and secret scanning automatically before every commit
- [act](https://github.com/nektos/act) — run your GitHub Actions pipeline locally before pushing

---

## 8. Test Your Code

Untested code is code you cannot safely change. Tests are what give you the confidence to refactor, upgrade dependencies, and ship features without fear.

There are three levels of testing:
- **Unit tests** — test individual methods in isolation. Fast and precise. This is a developer's primary responsibility
- **Integration tests** — verify that separate components work correctly together (a service correctly calling a repository, for example)
- **End-to-end tests** — validate the full user flow from input to output

A quick example in Java with JUnit:

```java
@Test
public void testCalculateFinalPrice_basicCase() {
    // $10 item, 3 units, 10% discount → $27.00
    assertEquals(27.00, calculator.calculateFinalPrice(10.0, 3, 0.10), 0.001);
}

@Test
public void testCalculateFinalPrice_noDiscount() {
    assertEquals(30.00, calculator.calculateFinalPrice(10.0, 3, 0.0), 0.001);
}

@Test
public void testCalculateFinalPrice_fullDiscount() {
    assertEquals(0.0, calculator.calculateFinalPrice(10.0, 3, 1.0), 0.001);
}
```

If you're afraid to refactor something, that's a signal you need more tests. Tests should be fast, isolated, and deterministic — no random data, no network calls, no shared state between test cases.

| Language | Testing Framework | Mocking Library |
|---|---|---|
| Java | [JUnit 5](https://junit.org/junit5/) | [Mockito](https://site.mockito.org/) |
| JavaScript | [Jest](https://jestjs.io/) | Built-in |
| Python | [pytest](https://pytest.org/) | [unittest.mock](https://docs.python.org/3/library/unittest.mock.html) |

---

## 9. Automate Formatting and Linting

Inconsistent formatting is cognitive noise. When a codebase mixes tabs and spaces, or uses different brace styles across different files, every code review becomes a style debate instead of a logic discussion. The solution: automate it and never discuss it again.

The two tools you need are a **formatter** and a **linter**, and they do different things.

A **formatter** is purely cosmetic. It rewrites your code to conform to a consistent visual style — indentation, spacing, line breaks, brace placement. It doesn't care about what your code does, only how it looks. You run it, and the file comes out formatted the same way every time, regardless of who wrote it.

A **linter** goes deeper. It analyzes your source code without executing it, looking for things that are technically valid but likely wrong: an unused variable, a function that's grown too complex, a pattern known to cause bugs, a potential null pointer, a security vulnerability. It works by parsing your code into a structured representation and checking it against a set of rules — then reporting exactly which line violates which rule and why. Think of it as a second pair of eyes that never gets tired and never misses the same mistake twice.

Together, a formatter handles *style* and a linter handles *quality*. Both run automatically, no human judgement required.

| Language | Formatter | Linter |
|---|---|---|
| Java | [google-java-format](https://github.com/google/google-java-format) | [Checkstyle](https://checkstyle.sourceforge.io/) |
| JavaScript | [Prettier](https://prettier.io/) | [ESLint](https://eslint.org/) |
| Python | [Black](https://github.com/psf/black) | [Ruff](https://docs.astral.sh/ruff/) |

Add both to your CI/CD pipeline so they run automatically on every PR. If they're not automated, they won't be consistent.

---

## 10. Refactor Continuously

> "Always leave the code cleaner than you found it."

Good code is not written once and left alone — it is refined over time. Small, continuous refactors prevent technical debt from accumulating into an unmaintainable mess.

The signals are usually obvious: a method with more than three or four parameters, the same logic in multiple places, a class that has grown past two or three hundred lines, or that vague anxiety you feel before touching a particular file. When you notice any of these, take a few minutes to clean it up before moving on.

You don't need a dedicated refactoring sprint. A few minutes of cleanup every time you touch a file is enough to keep a codebase healthy over the long term.

---

## 11. Don't Optimize Prematurely

> "Premature optimization is the root of all evil." — Donald Knuth

The correct priority order is always: **make it work → make it clear → make it fast**. Optimizing before you have a measured performance problem leads to complex, hard-to-read code that solves a problem you don't actually have.

Write the clearest solution first. Then, if performance becomes an issue, use a profiler to find where the real bottleneck is, and optimize only that specific part.

```
Don't guess. Measure.
```

Profiling tools worth knowing: [VisualVM](https://visualvm.github.io/) and [async-profiler](https://github.com/async-profiler/async-profiler) for Java, Chrome DevTools Performance panel for JavaScript, and [py-spy](https://github.com/benfred/py-spy) for Python.

---

## 12. Separate Your Concerns

Keep different responsibilities in different parts of your code. The classic pattern is a three-layer architecture:

```
Presentation layer   →  handles input and output (UI, API endpoints)
Business logic layer →  contains the rules of your application
Data layer           →  handles storage and retrieval
```

When these layers bleed into each other — database queries inside UI components, business rules scattered across API handlers — you end up with classes that are impossible to test, reuse, or safely change. The boundary between layers doesn't need to be rigid, but it does need to exist and be respected.

---

## 13. Never Hardcode Configuration

Hardcoding URLs, credentials, feature flags, or environment-specific values in source code is both a security risk and a maintenance problem.

```java
// Bad — credentials in source code
Connection db = DriverManager.getConnection(
    "jdbc:postgresql://prod-server/mydb", "admin", "secret123"
);

// Good — read from the environment at runtime
String dbUrl = System.getenv("DATABASE_URL");
Connection db = DriverManager.getConnection(dbUrl);
```

Store secrets in `.env` files locally — always listed in `.gitignore`. In production, use environment variables or a dedicated secret manager. Useful libraries for loading `.env` files: [dotenv-java](https://github.com/cdimascio/dotenv-java) for Java, [dotenv](https://github.com/motdotla/dotenv) for JavaScript, and [python-dotenv](https://github.com/theskumar/python-dotenv) for Python. For production secrets at scale, [HashiCorp Vault](https://www.vaultproject.io/) is the open source standard.

---

## 14. Take Code Review Seriously

Code reviews exist not just to catch bugs, but to share knowledge and raise the quality standard of the whole team. How you participate on both sides matters.

**As an author:** keep PRs small and focused, explain non-obvious choices in the PR description, and review your own diff before requesting a review — you'll catch most obvious mistakes yourself.

**As a reviewer:** ask questions instead of issuing directives. *"Could this be null here?"* lands better than *"This will crash."* Approve good-enough code — perfect is the enemy of shipped — and make a point of noting good patterns, not only criticising bad ones.

---

## 15. How to Approach a Problem Before Writing Code

One of the most common mistakes developers make — especially early on — is opening the editor before they fully understand the problem. The result is code that solves the wrong thing, or solves the right thing in the wrong way, and has to be thrown away.

Here is the process I follow every time I need to solve something and translate it into code:

1. **Understand the problem fully before touching the keyboard.** Read the requirements carefully. What is the input? What is the expected output? What are the edge cases — empty input, null values, very large data, invalid formats? Write these down. You cannot write a correct solution to a problem you haven't fully understood.

2. **Break it down into smaller sub-problems.** Almost every problem is a composition of simpler ones. Decompose it until each piece is small enough that you can describe it in one sentence. A function that does one clear, describable thing is a function you can write, test, and reason about.

3. **Think about the solution in plain language first.** Before writing a single line of code, describe your approach in plain English — or pseudocode if you prefer. What data structures will you use? What is the order of operations? Where could things go wrong? If you can't explain it clearly in words, you don't understand it well enough to code it yet.

4. **Write the test before the implementation.** Define what "correct" looks like before you try to be correct. Writing the test first forces you to think about the interface and behaviour of your code from the outside — which is exactly how users and other parts of the system will interact with it.

5. **Write the simplest solution that works.** Resist the urge to be clever. The goal of the first version is correctness, not elegance. Get it working, verify it against your tests and edge cases, then look at whether it needs to be made clearer or faster.

6. **Refactor once it's working.** With a passing test suite as your safety net, you can now clean up freely. Extract repeated logic, rename things for clarity, simplify control flow. This is the step where good code is actually made.

The discipline here is resisting the pull to jump straight to step five. Spending five minutes thinking clearly at the start saves hours of confused debugging later.

---

## Concepts Worth Understanding Deeply

Beyond day-to-day practices, these are the foundational ideas that will genuinely level up how you think about software. You don't need to master all of them at once, but each one is worth spending time with:

- **Big O notation** — reason about algorithm performance before profiling
- **Design patterns** — Factory, Singleton, Observer, Strategy, Decorator. Shared vocabulary for common solutions
- **CAP theorem** — fundamental trade-offs in distributed systems
- **ACID vs BASE** — the two competing models for database transaction reliability
- **REST vs GraphQL** — trade-offs between the two dominant API design philosophies
- **Immutability and pure functions** — the foundation of functional programming and a powerful tool for eliminating side-effect bugs
- **Concurrency and race conditions** — essential for writing correct multi-threaded or async code
- **The Twelve-Factor App** — a methodology for building scalable, portable software ([12factor.net](https://12factor.net/))

---

These are not rules to follow blindly. They are principles developed by generations of developers through experience — and, more importantly, through mistakes. Start with the ones most relevant to your current project, apply them consistently, and build the habit from there.

The goal is not a perfect codebase. It's a codebase that you and your team can read, understand, and confidently change six months from now. That's the whole game.
