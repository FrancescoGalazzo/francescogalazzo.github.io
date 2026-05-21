---
layout: post
title: Coding Best Practices II
date: 2026-05-14
description: Test.
tags: programming best-practices clean-code software-engineering
categories: coding
featured: false
---

In my experience, these are the practices that every developer should follow, regardless of language or framework. If you're early in your career, internalizing these habits will accelerate your growth faster than any new library or tool. If you're experienced, this is a useful checklist to audit yourself against.

The difference between code that's a joy to work with and code that makes you dread opening the file isn't usually about clever algorithms or exotic frameworks — it's about a handful of habits applied consistently.

---

## 1. Use an IDE

Before writing a single line of code, you need the right environment to write it in. An **IDE (Integrated Development Environment)** is a software tool that combines a code editor, debugger, file explorer, terminal, and smart code assistance into a single application.

Unlike a plain text editor, an IDE understands your code. It can:
- **Autocomplete** method names, variable names, and imports as you type.
- **Highlight errors** before you even run the code.
- **Refactor safely** — rename a variable everywhere it is used in one operation.
- **Debug interactively** — pause execution, inspect variables, and step through code line by line.
- **Integrate with Git** — stage, commit, and review diffs without leaving the editor.

A good IDE does not make you a better developer automatically, but it removes friction and surfaces problems earlier, which gives you more mental space to focus on logic and design.

**Recommended IDEs:**

| Language | IDE | Notes |
|---|---|---|
| Any | [VS Code](https://code.visualstudio.com/) | Free, open source, highly extensible via plugins |
| Java | [IntelliJ IDEA Community](https://www.jetbrains.com/idea/) | Free community edition, best-in-class Java support |
| Python | [PyCharm Community](https://www.jetbrains.com/pycharm/) | Free community edition, excellent Python tooling |
| Any | [Eclipse](https://www.eclipse.org/) | Free, open source, mature ecosystem |
| Any | [Neovim](https://neovim.io/) | Free, open source, keyboard-driven, highly configurable |

**Useful open source VS Code extensions:**
- [GitLens](https://marketplace.visualstudio.com/items?itemName=eamodio.gitlens) — rich Git history and blame annotations.
- [SonarLint](https://www.sonarsource.com/products/sonarlint/) — real-time code quality and security analysis.
- [Error Lens](https://marketplace.visualstudio.com/items?itemName=usernamehw.errorlens) — inline display of errors and warnings.

---

## 2. Write Code for Humans First

The machine doesn't care what you name your variables. Your colleagues — and your future self — very much do. Code is read far more often than it is written, so optimize for the reader.

**Bad:**
```java
public double calc(double a, int b, double c) {
    return a * b * (1 - c);
}
```

**Good:**
```java
public double calculateFinalPrice(double unitPrice, int quantity, double discountRate) {
    return unitPrice * quantity * (1 - discountRate);
}
```

The second version reads almost like a sentence. No comment needed to understand what it does.

**Rules of thumb:**
- Variable names should explain what they store, not how they are stored (`userList` not `data`).
- Method names should describe what they do (`sendWelcomeEmail()` not `process()`).
- Keep methods short — if it doesn't fit on one screen, it probably does too much.

**Open source tools:**
- [SonarQube Community Edition](https://www.sonarsource.com/open-source/) — static code analysis for readability and quality issues.
- [PMD](https://pmd.github.io/) — open source static analyzer for Java and other languages.
- [Checkstyle](https://checkstyle.sourceforge.io/) — enforces coding standards in Java projects.

---

## 3. The SOLID Principles

SOLID is a set of five design principles for writing maintainable object-oriented code. Large codebases tend to rot in predictable ways: classes that do too much, code that breaks when you extend it, dependencies impossible to swap out. SOLID is the antidote.

| Principle | Meaning | Practical example |
|---|---|---|
| **S**ingle Responsibility | A class does one thing | `UserValidator` only validates, not saves |
| **O**pen/Closed | Open for extension, closed for modification | Add behavior via new classes, not by editing existing ones |
| **L**iskov Substitution | Subclasses must be replaceable by their parent | A `Square` should not break code written for a `Rectangle` |
| **I**nterface Segregation | Many small interfaces beat one large one | `Printable` and `Serializable` separately, not one giant `Utility` |
| **D**ependency Inversion | Depend on abstractions, not concrete implementations | Inject a `DatabaseInterface`, not a specific `MySQLDatabase` |

I will go deeper into each of these principles with practical Java examples in a dedicated follow-up post — they each deserve more space than a table can give them.

---

## 4. Don't Repeat Yourself (DRY)

If the same logic appears in two places, it will eventually be updated in one and forgotten in the other. This is one of the most common sources of subtle, hard-to-find bugs.

**Bad:**
```java
// In user registration
String email = input.trim().toLowerCase();

// In password reset
String email = input.trim().toLowerCase();
```

**Good:**
```java
private String normalizeEmail(String input) {
    return input.trim().toLowerCase();
}
```

Extract repeated logic into methods, utilities, or shared classes. When the logic needs to change — and it will — you change it in exactly one place.

---

## 5. Write Meaningful Comments — Explain Why, Not What

The code already shows *what* it does. A good comment explains *why* a non-obvious decision was made. Over-commenting obvious code creates noise that hides the comments that actually matter.

**Bad:**
```java
i = i + 1; // increment i
```

**Good:**
```java
i += 1; // offset by 1 because the external API uses 1-based indexing
```

**Golden rules:**
- Comment on business rules, edge cases, and non-obvious algorithmic choices.
- Never comment out dead code — delete it. Git keeps the history.
- Update comments when you update the code. A wrong comment is worse than no comment.

---

## 6. Handle Errors Explicitly and Gracefully

Silent failures are a debugging nightmare. When something goes wrong and no error is logged, you have no idea where to start looking.

**Bad:**
```java
try {
    result = fetchUserData(userId);
} catch (Exception e) {
    // silently ignored
}
```

**Good:**
```java
try {
    result = fetchUserData(userId);
} catch (NetworkException e) {
    logger.error("Failed to fetch user {}: {}", userId, e.getMessage(), e);
    throw new ServiceUnavailableException("User data temporarily unavailable", e);
}
```

**Key habits:**
- Be specific about the exception type you catch — a bare `catch (Exception e)` hides bugs.
- Log the error with enough context to debug it later (include IDs, inputs, and the original error).
- Never expose raw stack traces to end users.
- Define custom exception classes to distinguish different failure modes.

---

## 7. Version Control: Git Done Right

Git is not just a backup tool — it is a communication tool. Your commit history tells the story of your project and lets you understand *why* a decision was made months later.

**Write meaningful commit messages:**
```bash
# Bad
git commit -m "fix stuff"

# Good
git commit -m "fix: handle null user in authentication flow"
```

Follow [Conventional Commits](https://www.conventionalcommits.org/) for a consistent structure:
- `feat:` — new feature
- `fix:` — bug fix
- `refactor:` — code restructuring without behavior change
- `docs:` — documentation changes
- `test:` — adding or updating tests

### What is a Pull Request?

A **Pull Request (PR)** is a formal proposal to merge a branch of code into another (usually `main`). When you finish working on a feature or fix in your branch, you open a PR to:

1. **Show what changed** — reviewers can see the exact diff, line by line.
2. **Request feedback** — teammates can comment on specific lines, ask questions, and suggest improvements.
3. **Run automated checks** — CI/CD pipelines run tests, linters, and security scans automatically on the PR before it is merged.
4. **Keep history clean** — the PR becomes a permanent record of *what* was changed and *why*, linked to the task or issue it solved.

PRs are the standard collaboration unit in professional software development. Even on solo projects, opening a PR forces you to review your own work before merging — a habit that catches many mistakes.

**Other habits:**
- Keep PRs small and focused — 200 to 400 lines is a good target. Large PRs don't get reviewed carefully.
- Never commit directly to `main`.
- Never commit secrets, passwords, or API keys. Use `.gitignore` and environment variables.

**Open source tools:**
- [pre-commit](https://pre-commit.com/) — run linting, formatting, and secret scanning automatically before every commit.
- [act](https://github.com/nektos/act) — run your GitHub Actions CI pipeline locally before pushing.

---

## 8. Test Your Code

Untested code is code you cannot safely change. Tests are what give you the confidence to refactor, upgrade dependencies, and ship features without fear.

**The three levels of testing:**
- **Unit tests** — test individual methods in isolation. Fast and precise.
- **Integration tests** — test how components work together (e.g., a service calling a database).
- **End-to-end tests** — test the full user flow from input to output.

**Example unit tests in Java with JUnit:**
```java
@Test
public void testCalculateFinalPrice_basicCase() {
    // $10 item, 3 units, 10% discount → $10 * 3 * 0.9 = $27.00
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

**Rules of thumb:**
- If you are afraid to refactor something, you need more tests.
- Tests should be fast, isolated, and deterministic — no random data, no network calls, no shared state.
- 80% coverage on critical paths is far more valuable than 100% coverage on trivial code.

**Open source tools:**

| Language | Testing framework | Mocking library |
|---|---|---|
| Java | [JUnit 5](https://junit.org/junit5/) | [Mockito](https://site.mockito.org/) |
| JavaScript | [Jest](https://jestjs.io/) | Built-in |
| Python | [pytest](https://pytest.org/) | [unittest.mock](https://docs.python.org/3/library/unittest.mock.html) |

---

## 9. Use Consistent Formatting and Linting

Inconsistent formatting is cognitive noise. When a codebase mixes tabs and spaces, or uses four different brace styles, every review degenerates into style arguments instead of logic discussions.

The solution: automate it and never discuss it again.

| Language | Formatter | Linter |
|---|---|---|
| Java | [google-java-format](https://github.com/google/google-java-format) | [Checkstyle](https://checkstyle.sourceforge.io/) |
| JavaScript | [Prettier](https://prettier.io/) | [ESLint](https://eslint.org/) |
| Python | [Black](https://github.com/psf/black) | [Ruff](https://docs.astral.sh/ruff/) |

Add these to your CI/CD pipeline so the check runs automatically on every PR. If it's not automated, it won't be consistent.

---

## 10. Refactor Continuously — The Boy Scout Rule

> "Always leave the code cleaner than you found it."

Good code is not written once — it is refined over time. Small, continuous refactors prevent technical debt from accumulating into an unmaintainable mess.

**Signs that code is asking to be refactored:**
- A method has more than 3–4 parameters.
- The same logic appears in multiple places.
- You need to read it twice to understand what it does.
- A class has grown past 200–300 lines.
- You feel anxious before changing it.

You don't need a dedicated "refactoring sprint." A few minutes of cleanup every time you touch a file is enough to keep a codebase healthy over the long term.

---

## 11. Avoid Premature Optimization

> "Premature optimization is the root of all evil." — Donald Knuth

The correct priority order is always: **make it work → make it clear → make it fast**.

Optimizing before you have a measured performance problem leads to complex, clever code that solves a problem you don't actually have. Write the clearest solution first, then profile to find where the actual bottleneck is, and only then optimize that specific part.

```
Don't guess. Measure.
```

**Open source profiling tools:**
- Java: [VisualVM](https://visualvm.github.io/) (free, bundled with JDK), [async-profiler](https://github.com/async-profiler/async-profiler)
- JavaScript: Chrome DevTools Performance panel (built-in)
- Python: [py-spy](https://github.com/benfred/py-spy)

---

## 12. Separate Concerns

Keep different responsibilities in different parts of your code. The classic pattern is the **three-layer architecture**:

```
Presentation layer   →  handles input and output (UI, API endpoints)
Business logic layer →  contains the rules of your application
Data layer           →  handles storage and retrieval
```

When these layers bleed into each other — database queries inside UI components, business rules scattered across API handlers — you end up with "God classes" that are impossible to test, reuse, or change safely.

---

## 13. Use Environment Variables for Configuration

Never hardcode URLs, credentials, feature flags, or environment-specific values in source code. This is both a security risk and a maintenance problem.

**Bad:**
```java
Connection db = DriverManager.getConnection(
    "jdbc:postgresql://prod-server/mydb", "admin", "secret123"
);
```

**Good:**
```java
String dbUrl = System.getenv("DATABASE_URL");
Connection db = DriverManager.getConnection(dbUrl);
```

Store secrets in `.env` files locally (always in `.gitignore`). In production, use environment variables or a dedicated secret manager.

**Open source tools:**
- [dotenv-java](https://github.com/cdimascio/dotenv-java) — Java `.env` file loader.
- [dotenv](https://github.com/motdotla/dotenv) — JavaScript.
- [python-dotenv](https://github.com/theskumar/python-dotenv) — Python.
- [HashiCorp Vault](https://www.vaultproject.io/) — production-grade open source secret management.

---

## 14. Code Review Culture

Code reviews exist not just to catch bugs, but to share knowledge and raise the quality standard of the whole team.

**As an author:**
- Keep PRs small and focused (see: section on Git).
- Explain *why* you made non-obvious choices in the PR description.
- Review your own diff before requesting a review — you'll catch most obvious mistakes yourself.

**As a reviewer:**
- Ask questions instead of issuing directives: *"Could this be null here?"* lands better than *"This will crash."*
- Approve good-enough code — perfect is the enemy of shipped.
- Praise good patterns, not only critique bad ones.

---

## Concepts Worth Knowing Deeply

Beyond day-to-day practices, these are the foundational ideas that will level up your understanding of software:

- **Big O notation** — reason about algorithm performance before profiling.
- **Design patterns** — Factory, Singleton, Observer, Strategy, Decorator. Shared vocabulary for common solutions.
- **CAP theorem** — fundamental trade-offs in distributed systems.
- **ACID vs BASE** — the two competing models for database transaction reliability.
- **REST vs GraphQL** — trade-offs between the two dominant API design philosophies.
- **Immutability and pure functions** — the foundation of functional programming and a powerful tool for eliminating side-effect bugs.
- **Concurrency and race conditions** — essential for writing correct multi-threaded or async code.
- **The Twelve-Factor App** — a methodology for building scalable, portable software ([12factor.net](https://12factor.net/)).

---

These are not rules to follow blindly. They are principles developed by generations of developers through experience and, more importantly, through mistakes. Start with the ones most relevant to your current project, apply them consistently, and build the habit from there. The goal is not a perfect codebase — it is a codebase that you and your team can read, understand, and confidently change six months from now.

That's the whole game.
