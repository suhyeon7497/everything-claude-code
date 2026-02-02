---
name: kotlin-patterns
description: Idiomatic Kotlin and Spring Boot patterns, best practices, and conventions for building robust, efficient, and maintainable applications.
---

# Kotlin/Spring Development Patterns

Idiomatic Kotlin and Spring Boot patterns for building robust, efficient, and maintainable applications.

## When to Activate

- Writing new Kotlin code
- Reviewing Kotlin/Spring code
- Refactoring existing Kotlin code
- Designing Spring Boot services

## Core Principles

### 1. Null Safety First

Kotlin's type system prevents null pointer exceptions at compile time. Use it fully.

```kotlin
// Good: Null-safe chain
fun getUserDisplayName(user: User?): String {
    return user?.profile?.displayName ?: "Anonymous"
}

// Bad: Force unwrap
fun getUserDisplayName(user: User?): String {
    return user!!.profile!!.displayName  // NPE risk
}
```

### 2. Prefer Immutability

Use `val` over `var`, immutable collections over mutable.

```kotlin
// Good: Immutable
data class User(
    val id: Long,
    val name: String,
    val email: String
)

val users = listOf(user1, user2)
val updated = users.map { it.copy(name = it.name.uppercase()) }

// Bad: Mutable
var name = "Alice"
val mutableUsers = mutableListOf<User>()
mutableUsers.add(user1)
```

### 3. Data Classes for DTOs

```kotlin
// Good: Data class gives equals, hashCode, toString, copy
data class CreateUserRequest(
    @field:NotBlank val name: String,
    @field:Email val email: String,
    @field:Size(min = 8) val password: String
)

// Good: Sealed class for responses
sealed class ApiResult<out T> {
    data class Success<T>(val data: T) : ApiResult<T>()
    data class Error(val message: String, val code: Int) : ApiResult<Nothing>()
}
```

### 4. Extension Functions for Utility

```kotlin
// Good: Extension function - reads naturally
fun String.toSlug(): String =
    lowercase()
        .replace(Regex("[^a-z0-9\\s-]"), "")
        .replace(Regex("\\s+"), "-")
        .trim('-')

// Usage
val slug = "Hello World!".toSlug() // "hello-world"

// Good: Extension on domain type
fun User.toResponse(): UserResponse =
    UserResponse(
        id = id,
        name = name,
        email = email,
    )
```

## Spring Boot Patterns

### Constructor Injection (Always)

```kotlin
// Good: Constructor injection (Kotlin primary constructor)
@Service
class UserService(
    private val userRepository: UserRepository,
    private val passwordEncoder: PasswordEncoder,
    private val eventPublisher: ApplicationEventPublisher,
)

// Bad: Field injection
@Service
class UserService {
    @Autowired lateinit var userRepository: UserRepository
}
```

### Repository Pattern with Spring Data

```kotlin
interface UserRepository : JpaRepository<User, Long> {
    fun findByEmail(email: String): User?
    fun findByNameContainingIgnoreCase(name: String): List<User>

    @Query("SELECT u FROM User u WHERE u.createdAt > :since")
    fun findRecentUsers(@Param("since") since: LocalDateTime): List<User>

    @Query("SELECT u FROM User u JOIN FETCH u.roles WHERE u.id = :id")
    fun findByIdWithRoles(@Param("id") id: Long): User?
}
```

### Service Layer with Transaction Management

```kotlin
@Service
class UserService(
    private val userRepository: UserRepository,
    private val passwordEncoder: PasswordEncoder,
) {
    @Transactional(readOnly = true)
    fun findById(id: Long): User =
        userRepository.findById(id)
            .orElseThrow { EntityNotFoundException("User not found: $id") }

    @Transactional
    fun createUser(request: CreateUserRequest): User {
        require(userRepository.findByEmail(request.email) == null) {
            "Email already exists: ${request.email}"
        }

        val user = User(
            name = request.name,
            email = request.email,
            passwordHash = passwordEncoder.encode(request.password),
        )
        return userRepository.save(user)
    }

    @Transactional
    fun updateUser(id: Long, request: UpdateUserRequest): User {
        val user = findById(id)
        val updated = user.copy(
            name = request.name ?: user.name,
            email = request.email ?: user.email,
        )
        return userRepository.save(updated)
    }
}
```

### REST Controller with Validation

```kotlin
@RestController
@RequestMapping("/api/v1/users")
class UserController(
    private val userService: UserService,
) {
    @GetMapping("/{id}")
    fun getUser(@PathVariable id: Long): ResponseEntity<UserResponse> {
        val user = userService.findById(id)
        return ResponseEntity.ok(user.toResponse())
    }

    @PostMapping
    fun createUser(
        @Valid @RequestBody request: CreateUserRequest
    ): ResponseEntity<UserResponse> {
        val user = userService.createUser(request)
        val location = URI.create("/api/v1/users/${user.id}")
        return ResponseEntity.created(location).body(user.toResponse())
    }

    @ExceptionHandler(EntityNotFoundException::class)
    fun handleNotFound(e: EntityNotFoundException): ResponseEntity<ErrorResponse> =
        ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(ErrorResponse(message = e.message ?: "Not found"))
}
```

## Error Handling Patterns

### Sealed Class for Domain Errors

```kotlin
sealed class DomainException(message: String) : RuntimeException(message) {
    class NotFound(entity: String, id: Any) :
        DomainException("$entity not found: $id")

    class AlreadyExists(entity: String, field: String, value: Any) :
        DomainException("$entity with $field=$value already exists")

    class InvalidOperation(reason: String) :
        DomainException("Invalid operation: $reason")
}

// Usage
throw DomainException.NotFound("User", userId)
```

### Global Exception Handler

```kotlin
@RestControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(DomainException.NotFound::class)
    fun handleNotFound(e: DomainException.NotFound) =
        ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(ErrorResponse(e.message ?: "Not found"))

    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun handleValidation(e: MethodArgumentNotValidException) =
        ResponseEntity.badRequest().body(
            ErrorResponse(
                message = "Validation failed",
                details = e.bindingResult.fieldErrors.associate {
                    it.field to (it.defaultMessage ?: "Invalid")
                }
            )
        )
}

data class ErrorResponse(
    val message: String,
    val details: Map<String, String>? = null,
)
```

### Result Pattern (No Exceptions for Expected Failures)

```kotlin
sealed class Result<out T> {
    data class Success<T>(val value: T) : Result<T>()
    data class Failure(val error: String) : Result<Nothing>()

    fun getOrNull(): T? = when (this) {
        is Success -> value
        is Failure -> null
    }

    fun <R> map(transform: (T) -> R): Result<R> = when (this) {
        is Success -> Success(transform(value))
        is Failure -> this
    }
}

// Usage
fun findUser(id: Long): Result<User> {
    val user = userRepository.findById(id).orElse(null)
        ?: return Result.Failure("User not found: $id")
    return Result.Success(user)
}
```

## Coroutine Patterns (Spring WebFlux)

### Suspend Controller

```kotlin
@RestController
@RequestMapping("/api/v1/users")
class UserController(
    private val userService: UserService,
) {
    @GetMapping("/{id}")
    suspend fun getUser(@PathVariable id: Long): ResponseEntity<UserResponse> {
        val user = userService.findById(id)
        return ResponseEntity.ok(user.toResponse())
    }
}
```

### Structured Concurrency

```kotlin
suspend fun fetchDashboard(userId: Long): Dashboard = coroutineScope {
    val userDeferred = async { userService.findById(userId) }
    val ordersDeferred = async { orderService.findByUserId(userId) }
    val statsDeferred = async { statsService.getUserStats(userId) }

    Dashboard(
        user = userDeferred.await(),
        orders = ordersDeferred.await(),
        stats = statsDeferred.await(),
    )
}
```

### Dispatcher Usage

```kotlin
// CPU-bound work
suspend fun computeHash(data: String): String = withContext(Dispatchers.Default) {
    MessageDigest.getInstance("SHA-256")
        .digest(data.toByteArray())
        .joinToString("") { "%02x".format(it) }
}

// IO-bound work
suspend fun readFile(path: String): String = withContext(Dispatchers.IO) {
    File(path).readText()
}
```

## JPA Entity Patterns

### Entity with Kotlin

```kotlin
@Entity
@Table(name = "users")
class User(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @Column(nullable = false)
    var name: String,

    @Column(nullable = false, unique = true)
    val email: String,

    @Column(name = "password_hash", nullable = false)
    var passwordHash: String,

    @Column(name = "created_at", nullable = false, updatable = false)
    val createdAt: LocalDateTime = LocalDateTime.now(),

    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    val orders: List<Order> = emptyList(),
) {
    // Required for JPA (use kotlin-jpa plugin for auto no-arg)
    override fun equals(other: Any?): Boolean {
        if (this === other) return true
        if (other !is User) return false
        return id != 0L && id == other.id
    }

    override fun hashCode(): Int = javaClass.hashCode()
}
```

### build.gradle.kts Plugins for JPA

```kotlin
plugins {
    kotlin("plugin.spring")   // Opens Spring-annotated classes
    kotlin("plugin.jpa")      // Generates no-arg constructors
    kotlin("plugin.allopen")  // Opens @Entity classes
}

allOpen {
    annotation("jakarta.persistence.Entity")
    annotation("jakarta.persistence.MappedSuperclass")
    annotation("jakarta.persistence.Embeddable")
}
```

## Scope Function Guide

| Function | Object ref | Return value | Use case |
|----------|-----------|-------------|----------|
| `let` | `it` | Lambda result | Null checks, transformations |
| `run` | `this` | Lambda result | Object config + compute |
| `with` | `this` | Lambda result | Grouping calls on object |
| `apply` | `this` | Object itself | Object configuration |
| `also` | `it` | Object itself | Side effects (logging) |

```kotlin
// let: Null-safe transformation
val length = name?.let { it.trim().length } ?: 0

// apply: Builder-style configuration
val user = User().apply {
    name = "Alice"
    email = "alice@example.com"
}

// also: Side effects
userRepository.save(user).also {
    logger.info("Created user: ${it.id}")
}

// run: Object configuration + result
val result = connection.run {
    prepareStatement(sql)
    executeQuery()
    // result is returned
}
```

## Collection Patterns

```kotlin
// Filtering and mapping
val activeUserNames = users
    .filter { it.isActive }
    .map { it.name }

// Grouping
val usersByRole = users.groupBy { it.role }

// First or null (safe)
val admin = users.firstOrNull { it.role == Role.ADMIN }

// Associate to map
val userMap = users.associateBy { it.id }

// Sequence for large collections (lazy evaluation)
val result = hugeList.asSequence()
    .filter { it.isValid }
    .map { it.transform() }
    .take(10)
    .toList()
```

## Configuration Pattern

```kotlin
@ConfigurationProperties(prefix = "app")
data class AppProperties(
    val name: String = "MyApp",
    val maxRetries: Int = 3,
    val database: DatabaseProperties = DatabaseProperties(),
) {
    data class DatabaseProperties(
        val url: String = "",
        val poolSize: Int = 10,
    )
}

// application.yml
// app:
//   name: MyApp
//   max-retries: 5
//   database:
//     url: jdbc:postgresql://localhost:5432/mydb
//     pool-size: 20
```

## Anti-Patterns to Avoid

```kotlin
// Bad: Using !! everywhere
val name = user!!.name!!.trim()

// Bad: Java-style static utility class
object StringUtils {
    fun isEmpty(s: String?) = s.isNullOrEmpty()
}
// Good: Use stdlib or extension functions
val empty = name.isNullOrEmpty()

// Bad: Mutable where immutable works
var items = mutableListOf<Item>()
// Good
val items = listOf(item1, item2)

// Bad: When without exhaustive check on sealed class
when (result) {
    is Success -> handle(result.data)
    // Missing Failure case!
}
// Good: Compiler enforces exhaustive when
when (result) {
    is Success -> handle(result.data)
    is Failure -> handleError(result.error)
}

// Bad: Not using named arguments for clarity
createOrder(1L, 2L, "pending", true, false)
// Good
createOrder(
    userId = 1L,
    productId = 2L,
    status = "pending",
    isPriority = true,
    isGift = false,
)
```

## Quick Reference: Kotlin Idioms

| Idiom | Description |
|-------|-------------|
| Null safety (`?.`, `?:`, `?.let`) | Prevent NPE at compile time |
| `val` over `var` | Prefer immutability |
| Data classes | Auto equals/hashCode/toString/copy |
| Extension functions | Add behavior without inheritance |
| Sealed classes | Exhaustive type hierarchies |
| Scope functions | `let`, `run`, `with`, `apply`, `also` |
| Named arguments | Clarity for multi-param functions |
| String templates | `"Hello, $name"` |
| Destructuring | `val (id, name) = user` |
| `when` expression | Pattern matching |

**Remember**: Idiomatic Kotlin is concise, safe, and expressive. Leverage the type system, use null safety, prefer immutability, and write code that reads like documentation.
