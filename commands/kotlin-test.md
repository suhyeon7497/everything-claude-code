---
description: Enforce TDD workflow for Kotlin/Spring. Write tests first with JUnit 5 and MockK, then implement. Verify 80%+ coverage with JaCoCo.
---

# Kotlin TDD Command

This command enforces test-driven development methodology for Kotlin code using JUnit 5, MockK, and Spring Boot Test patterns.

## What This Command Does

1. **Define Types/Interfaces**: Scaffold function signatures first
2. **Write Tests First**: Create comprehensive test cases (RED)
3. **Run Tests**: Verify tests fail for the right reason
4. **Implement Code**: Write minimal code to pass (GREEN)
5. **Refactor**: Improve while keeping tests green
6. **Check Coverage**: Ensure 80%+ coverage with JaCoCo

## When to Use

Use `/kotlin-test` when:
- Implementing new Kotlin functions or Spring services
- Adding test coverage to existing code
- Fixing bugs (write failing test first)
- Building critical business logic
- Learning TDD workflow in Kotlin/Spring

## TDD Cycle

```
RED     → Write failing test with JUnit 5
GREEN   → Implement minimal code to pass
REFACTOR → Improve code, tests stay green
REPEAT  → Next test case
```

## Example Session

```text
User: /kotlin-test I need a service to validate email addresses

Agent:
# TDD Session: Email Validator

## Step 1: Define Interface

```kotlin
// src/main/kotlin/com/example/validator/EmailValidator.kt
package com.example.validator

@Service
class EmailValidator {
    fun validate(email: String): ValidationResult {
        TODO("Not implemented")
    }
}

data class ValidationResult(
    val isValid: Boolean,
    val error: String? = null
)
```

## Step 2: Write Tests First (RED)

```kotlin
// src/test/kotlin/com/example/validator/EmailValidatorTest.kt
package com.example.validator

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Nested
import org.junit.jupiter.api.Test
import org.junit.jupiter.params.ParameterizedTest
import org.junit.jupiter.params.provider.ValueSource

class EmailValidatorTest {

    private val validator = EmailValidator()

    @Nested
    inner class ValidEmails {
        @ParameterizedTest
        @ValueSource(strings = [
            "user@example.com",
            "user@mail.example.com",
            "user+tag@example.com",
            "first.last@example.com"
        ])
        fun `should accept valid email addresses`(email: String) {
            val result = validator.validate(email)
            assertThat(result.isValid).isTrue()
            assertThat(result.error).isNull()
        }
    }

    @Nested
    inner class InvalidEmails {
        @Test
        fun `should reject empty string`() {
            val result = validator.validate("")
            assertThat(result.isValid).isFalse()
            assertThat(result.error).isEqualTo("Email cannot be empty")
        }

        @ParameterizedTest
        @ValueSource(strings = [
            "userexample.com",
            "user@",
            "@example.com",
            "user@@example.com",
            "user @example.com"
        ])
        fun `should reject invalid email formats`(email: String) {
            val result = validator.validate(email)
            assertThat(result.isValid).isFalse()
        }
    }
}
```

## Step 3: Run Tests - Verify FAIL

```bash
$ ./gradlew test --tests '*EmailValidatorTest*'

EmailValidatorTest > ValidEmails > should accept valid email addresses(String) FAILED
    kotlin.NotImplementedError: Not implemented

FAILED (6 tests, 0 passed, 6 failed)
```

✓ Tests fail as expected (NotImplementedError).

## Step 4: Implement Minimal Code (GREEN)

```kotlin
// src/main/kotlin/com/example/validator/EmailValidator.kt
package com.example.validator

import org.springframework.stereotype.Service

@Service
class EmailValidator {

    private val emailRegex = Regex("^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$")

    fun validate(email: String): ValidationResult {
        if (email.isBlank()) {
            return ValidationResult(isValid = false, error = "Email cannot be empty")
        }
        if (!emailRegex.matches(email)) {
            return ValidationResult(isValid = false, error = "Invalid email format")
        }
        return ValidationResult(isValid = true)
    }
}
```

## Step 5: Run Tests - Verify PASS

```bash
$ ./gradlew test --tests '*EmailValidatorTest*'

PASSED (6 tests, 6 passed)
```

✓ All tests passing!

## Step 6: Check Coverage

```bash
$ ./gradlew test jacocoTestReport

Coverage: 100% (EmailValidator.kt)
```

✓ Coverage: 100%

## TDD Complete!
```

## Test Patterns

### Parameterized Tests
```kotlin
@ParameterizedTest
@CsvSource(
    "1, true",
    "0, false",
    "-1, false"
)
fun `should validate positive numbers`(input: Int, expected: Boolean) {
    assertThat(isPositive(input)).isEqualTo(expected)
}
```

### MockK for Mocking
```kotlin
@ExtendWith(MockKExtension::class)
class UserServiceTest {

    @MockK
    lateinit var userRepository: UserRepository

    @InjectMockKs
    lateinit var userService: UserService

    @Test
    fun `should return user by id`() {
        every { userRepository.findById(1L) } returns Optional.of(User(1L, "Alice"))

        val user = userService.findById(1L)

        assertThat(user.name).isEqualTo("Alice")
        verify(exactly = 1) { userRepository.findById(1L) }
    }
}
```

### Spring Boot Integration Test
```kotlin
@SpringBootTest
@AutoConfigureMockMvc
class UserControllerIntegrationTest {

    @Autowired
    lateinit var mockMvc: MockMvc

    @Test
    fun `GET users should return 200`() {
        mockMvc.get("/api/users") {
            accept = MediaType.APPLICATION_JSON
        }.andExpect {
            status { isOk() }
            jsonPath("$.length()") { value(greaterThan(0)) }
        }
    }
}
```

### Test Helpers
```kotlin
fun createTestUser(
    id: Long = 1L,
    name: String = "Test User",
    email: String = "test@example.com"
) = User(id = id, name = name, email = email)
```

## Coverage Commands

```bash
# Run tests with coverage
./gradlew test jacocoTestReport

# View HTML report
# build/reports/jacoco/test/html/index.html

# Check coverage threshold
./gradlew jacocoTestCoverageVerification

# Run specific test class
./gradlew test --tests '*UserServiceTest*'

# Run specific test method
./gradlew test --tests '*UserServiceTest.should return user*'
```

## Coverage Targets

| Code Type | Target |
|-----------|--------|
| Critical business logic | 100% |
| Service layer | 90%+ |
| General code | 80%+ |
| Generated code | Exclude |

## TDD Best Practices

**DO:**
- Write test FIRST, before any implementation
- Run tests after each change
- Use `@Nested` to organize related tests
- Use parameterized tests for comprehensive coverage
- Include edge cases (empty, null, max values)

**DON'T:**
- Write implementation before tests
- Skip the RED phase
- Use `@SpringBootTest` for unit tests (use MockK instead)
- Use `Thread.sleep` in tests
- Ignore flaky tests

## Related Commands

- `/kotlin-build` - Fix build errors
- `/kotlin-review` - Review code after implementation
- `/verify` - Run full verification loop

## Related

- Skill: `skills/kotlin-testing/`
- Skill: `skills/tdd-workflow/`
