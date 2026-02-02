---
name: kotlin-testing
description: Kotlin/Spring testing patterns including JUnit 5, MockK, Spring Boot Test, parameterized tests, and test coverage. Follows TDD methodology with idiomatic Kotlin practices.
---

# Kotlin/Spring Testing Patterns

Comprehensive testing patterns for Kotlin/Spring Boot applications using JUnit 5, MockK, and AssertJ.

## When to Activate

- Writing new Kotlin functions or Spring services
- Adding test coverage to existing code
- Creating integration tests for Spring Boot
- Following TDD workflow in Kotlin projects

## TDD Workflow for Kotlin

### The RED-GREEN-REFACTOR Cycle

```
RED     → Write a failing test first
GREEN   → Write minimal code to pass the test
REFACTOR → Improve code while keeping tests green
REPEAT  → Continue with next requirement
```

### Step-by-Step TDD in Kotlin

```kotlin
// Step 1: Define the interface/signature
// Calculator.kt
package com.example.math

fun add(a: Int, b: Int): Int = TODO("Not implemented")

// Step 2: Write failing test (RED)
// CalculatorTest.kt
package com.example.math

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test

class CalculatorTest {
    @Test
    fun `add should return sum of two numbers`() {
        assertThat(add(2, 3)).isEqualTo(5)
    }
}

// Step 3: Run test - verify FAIL
// ./gradlew test -> NotImplementedError

// Step 4: Implement minimal code (GREEN)
fun add(a: Int, b: Int): Int = a + b

// Step 5: Run test - verify PASS
// Step 6: Refactor if needed
```

## JUnit 5 with Kotlin

### Basic Test Structure

```kotlin
class UserServiceTest {

    private val userRepository = mockk<UserRepository>()
    private val userService = UserService(userRepository)

    @Test
    fun `should find user by id`() {
        // Given
        val user = User(id = 1L, name = "Alice")
        every { userRepository.findById(1L) } returns Optional.of(user)

        // When
        val result = userService.findById(1L)

        // Then
        assertThat(result.name).isEqualTo("Alice")
        verify(exactly = 1) { userRepository.findById(1L) }
    }

    @Test
    fun `should throw when user not found`() {
        every { userRepository.findById(any()) } returns Optional.empty()

        assertThatThrownBy { userService.findById(999L) }
            .isInstanceOf(EntityNotFoundException::class.java)
            .hasMessageContaining("999")
    }
}
```

### Nested Tests for Organization

```kotlin
class UserServiceTest {

    private val repo = mockk<UserRepository>()
    private val service = UserService(repo)

    @Nested
    inner class FindById {
        @Test
        fun `returns user when found`() { /* ... */ }

        @Test
        fun `throws when not found`() { /* ... */ }
    }

    @Nested
    inner class CreateUser {
        @Test
        fun `creates user with valid input`() { /* ... */ }

        @Test
        fun `rejects duplicate email`() { /* ... */ }

        @Test
        fun `hashes password before saving`() { /* ... */ }
    }

    @Nested
    inner class DeleteUser {
        @Test
        fun `deletes existing user`() { /* ... */ }

        @Test
        fun `throws when user not found`() { /* ... */ }
    }
}
```

### Parameterized Tests

```kotlin
class EmailValidatorTest {

    private val validator = EmailValidator()

    @ParameterizedTest
    @ValueSource(strings = [
        "user@example.com",
        "user+tag@example.com",
        "first.last@example.com",
    ])
    fun `should accept valid emails`(email: String) {
        assertThat(validator.validate(email).isValid).isTrue()
    }

    @ParameterizedTest
    @ValueSource(strings = [
        "",
        "invalid",
        "user@",
        "@example.com",
    ])
    fun `should reject invalid emails`(email: String) {
        assertThat(validator.validate(email).isValid).isFalse()
    }

    @ParameterizedTest
    @CsvSource(
        "1, 2, 3",
        "0, 0, 0",
        "-1, 1, 0",
        "100, 200, 300",
    )
    fun `should add numbers correctly`(a: Int, b: Int, expected: Int) {
        assertThat(add(a, b)).isEqualTo(expected)
    }

    @ParameterizedTest
    @MethodSource("provideTestCases")
    fun `should process input correctly`(input: String, expected: String) {
        assertThat(process(input)).isEqualTo(expected)
    }

    companion object {
        @JvmStatic
        fun provideTestCases() = listOf(
            Arguments.of("hello", "HELLO"),
            Arguments.of("world", "WORLD"),
        )
    }
}
```

## MockK Patterns

### Basic Mocking

```kotlin
// Create mock
val repository = mockk<UserRepository>()

// Stub behavior
every { repository.findById(1L) } returns Optional.of(testUser)
every { repository.save(any()) } answers { firstArg() }
every { repository.deleteById(any()) } just Runs

// Verify interactions
verify { repository.findById(1L) }
verify(exactly = 1) { repository.save(any()) }
verify(exactly = 0) { repository.deleteById(any()) }
```

### Capturing Arguments

```kotlin
@Test
fun `should save user with hashed password`() {
    val slot = slot<User>()
    every { userRepository.save(capture(slot)) } answers { firstArg() }

    userService.createUser(CreateUserRequest(
        name = "Alice",
        email = "alice@example.com",
        password = "plain123",
    ))

    val savedUser = slot.captured
    assertThat(savedUser.name).isEqualTo("Alice")
    assertThat(savedUser.passwordHash).isNotEqualTo("plain123")
    assertThat(savedUser.passwordHash).startsWith("$2a$")
}
```

### Relaxed Mocks

```kotlin
// All methods return default values
val logger = mockk<Logger>(relaxed = true)

// Only Unit-returning methods return Unit
val service = mockk<UserService>(relaxUnitFun = true)
```

### Coroutine Mocking

```kotlin
@Test
fun `should fetch user asynchronously`() = runTest {
    coEvery { userRepository.findById(1L) } returns testUser

    val result = userService.findById(1L)

    assertThat(result.name).isEqualTo("Alice")
    coVerify { userRepository.findById(1L) }
}
```

## Spring Boot Test Patterns

### Slice Tests (Focused)

```kotlin
// Controller layer only
@WebMvcTest(UserController::class)
class UserControllerTest {

    @Autowired
    lateinit var mockMvc: MockMvc

    @MockkBean
    lateinit var userService: UserService

    @Test
    fun `GET users_id should return user`() {
        every { userService.findById(1L) } returns testUser

        mockMvc.get("/api/v1/users/1") {
            accept = MediaType.APPLICATION_JSON
        }.andExpect {
            status { isOk() }
            jsonPath("$.name") { value("Alice") }
            jsonPath("$.email") { value("alice@example.com") }
        }
    }

    @Test
    fun `POST users should validate input`() {
        mockMvc.post("/api/v1/users") {
            contentType = MediaType.APPLICATION_JSON
            content = """{"name": "", "email": "invalid"}"""
        }.andExpect {
            status { isBadRequest() }
        }
    }
}
```

### Repository Test

```kotlin
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class UserRepositoryTest {

    companion object {
        @Container
        val postgres = PostgreSQLContainer("postgres:16-alpine")

        @DynamicPropertySource
        @JvmStatic
        fun properties(registry: DynamicPropertyRegistry) {
            registry.add("spring.datasource.url") { postgres.jdbcUrl }
            registry.add("spring.datasource.username") { postgres.username }
            registry.add("spring.datasource.password") { postgres.password }
        }
    }

    @Autowired
    lateinit var userRepository: UserRepository

    @Test
    fun `should find user by email`() {
        val user = User(name = "Alice", email = "alice@example.com", passwordHash = "hash")
        userRepository.save(user)

        val found = userRepository.findByEmail("alice@example.com")

        assertThat(found).isNotNull
        assertThat(found!!.name).isEqualTo("Alice")
    }
}
```

### Full Integration Test

```kotlin
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class UserIntegrationTest {

    @Autowired
    lateinit var restTemplate: TestRestTemplate

    @Autowired
    lateinit var userRepository: UserRepository

    @BeforeEach
    fun setup() {
        userRepository.deleteAll()
    }

    @Test
    fun `full user lifecycle`() {
        // Create
        val createResponse = restTemplate.postForEntity(
            "/api/v1/users",
            CreateUserRequest("Alice", "alice@example.com", "password123"),
            UserResponse::class.java,
        )
        assertThat(createResponse.statusCode).isEqualTo(HttpStatus.CREATED)
        val userId = createResponse.body!!.id

        // Read
        val getResponse = restTemplate.getForEntity(
            "/api/v1/users/$userId",
            UserResponse::class.java,
        )
        assertThat(getResponse.body!!.name).isEqualTo("Alice")

        // Delete
        restTemplate.delete("/api/v1/users/$userId")

        // Verify deleted
        val notFound = restTemplate.getForEntity(
            "/api/v1/users/$userId",
            ErrorResponse::class.java,
        )
        assertThat(notFound.statusCode).isEqualTo(HttpStatus.NOT_FOUND)
    }
}
```

## Test Fixtures & Helpers

### Factory Functions

```kotlin
object TestFixtures {
    fun createUser(
        id: Long = 0L,
        name: String = "Test User",
        email: String = "test@example.com",
        passwordHash: String = "hashed",
        createdAt: LocalDateTime = LocalDateTime.now(),
    ) = User(
        id = id,
        name = name,
        email = email,
        passwordHash = passwordHash,
        createdAt = createdAt,
    )

    fun createUserRequest(
        name: String = "Test User",
        email: String = "test@example.com",
        password: String = "password123",
    ) = CreateUserRequest(
        name = name,
        email = email,
        password = password,
    )
}
```

### Custom AssertJ Assertions

```kotlin
class UserAssert(actual: User) : AbstractAssert<UserAssert, User>(actual, UserAssert::class.java) {

    fun hasName(expected: String): UserAssert {
        assertThat(actual.name).isEqualTo(expected)
        return this
    }

    fun hasEmail(expected: String): UserAssert {
        assertThat(actual.email).isEqualTo(expected)
        return this
    }

    companion object {
        fun assertThat(actual: User) = UserAssert(actual)
    }
}

// Usage
UserAssert.assertThat(user)
    .hasName("Alice")
    .hasEmail("alice@example.com")
```

## Test Coverage

### JaCoCo Configuration

```kotlin
// build.gradle.kts
plugins {
    jacoco
}

tasks.jacocoTestReport {
    reports {
        xml.required.set(true)
        html.required.set(true)
    }
}

tasks.jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                minimum = 0.80.toBigDecimal()
            }
        }
        rule {
            element = "CLASS"
            excludes = listOf(
                "*.config.*",
                "*.Application*",
            )
            limit {
                minimum = 0.80.toBigDecimal()
            }
        }
    }
}

tasks.test {
    finalizedBy(tasks.jacocoTestReport)
}

tasks.check {
    dependsOn(tasks.jacocoTestCoverageVerification)
}
```

### Running Coverage

```bash
# Run tests with coverage
./gradlew test jacocoTestReport

# View HTML report
# build/reports/jacoco/test/html/index.html

# Verify coverage meets threshold
./gradlew jacocoTestCoverageVerification
```

### Coverage Targets

| Code Type | Target |
|-----------|--------|
| Critical business logic | 100% |
| Service layer | 90%+ |
| Controller layer | 80%+ |
| General code | 80%+ |
| Config/Application classes | Exclude |
| Generated code | Exclude |

## Testing Best Practices

**DO:**
- Write tests FIRST (TDD)
- Use `@Nested` to organize test classes
- Use parameterized tests for multiple inputs
- Use MockK over Mockito for Kotlin
- Use Testcontainers for database tests
- Use `assertThat` (AssertJ) over JUnit assertions
- Give tests descriptive names with backticks

**DON'T:**
- Use `@SpringBootTest` for unit tests
- Use `Thread.sleep` in tests (use Awaitility instead)
- Share mutable state between tests
- Mock everything (prefer integration tests for Spring beans)
- Ignore flaky tests (fix or quarantine them)
- Test private functions directly

## build.gradle.kts Test Dependencies

```kotlin
dependencies {
    // Test framework
    testImplementation("org.springframework.boot:spring-boot-starter-test") {
        exclude(module = "mockito-core")
    }

    // MockK (Kotlin-native mocking)
    testImplementation("io.mockk:mockk:1.13.8")
    testImplementation("com.ninja-squad:springmockk:4.0.2")

    // Assertions
    testImplementation("org.assertj:assertj-core:3.25.1")

    // Testcontainers
    testImplementation("org.testcontainers:junit-jupiter:1.19.3")
    testImplementation("org.testcontainers:postgresql:1.19.3")

    // Coroutine testing
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3")

    // Awaitility (async assertions)
    testImplementation("org.awaitility:awaitility-kotlin:4.2.0")
}
```

## CI Integration

```yaml
# GitHub Actions
test:
  runs-on: ubuntu-latest
  services:
    postgres:
      image: postgres:16-alpine
      env:
        POSTGRES_DB: testdb
        POSTGRES_USER: test
        POSTGRES_PASSWORD: test
      ports:
        - 5432:5432
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-java@v4
      with:
        distribution: 'temurin'
        java-version: '21'

    - name: Run tests
      run: ./gradlew test jacocoTestReport

    - name: Check coverage
      run: ./gradlew jacocoTestCoverageVerification

    - name: Upload coverage report
      uses: actions/upload-artifact@v4
      with:
        name: coverage-report
        path: build/reports/jacoco/
```

**Remember**: Tests are documentation. They show how your code is meant to be used. Use descriptive names, follow Arrange-Act-Assert, and keep them focused and independent.
