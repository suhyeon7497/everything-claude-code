---
description: Comprehensive Kotlin/Spring code review for idiomatic patterns, null safety, coroutine correctness, and security. Invokes the kotlin-reviewer agent.
---

# Kotlin Code Review

This command invokes the **kotlin-reviewer** agent for comprehensive Kotlin/Spring-specific code review.

## What This Command Does

1. **Identify Kotlin Changes**: Find modified `.kt`/`.kts` files via `git diff`
2. **Run Static Analysis**: Execute `detekt`, `ktlint`, compiler warnings
3. **Security Scan**: Check for SQL injection, command injection, missing auth
4. **Coroutine Review**: Analyze coroutine safety, scope management, dispatcher usage
5. **Spring Review**: Verify DI patterns, transaction management, N+1 queries
6. **Generate Report**: Categorize issues by severity

## When to Use

Use `/kotlin-review` when:
- After writing or modifying Kotlin code
- Before committing Kotlin/Spring changes
- Reviewing pull requests with Kotlin code
- Onboarding to a new Kotlin/Spring codebase
- Learning idiomatic Kotlin patterns

## Review Categories

### CRITICAL (Must Fix)
- SQL/Command injection vulnerabilities
- Hardcoded credentials
- Force unwrap (`!!`) on untrusted data
- Missing `@PreAuthorize` on sensitive endpoints
- Blocking calls in reactive/coroutine context
- Shared mutable state without synchronization

### HIGH (Should Fix)
- Excessive `!!` usage instead of safe calls
- GlobalScope coroutine leaks
- Field injection instead of constructor injection
- Missing `@Transactional` on multi-step operations
- N+1 query patterns
- Missing input validation

### MEDIUM (Consider)
- Not using Kotlin idioms (let, run, apply, etc.)
- `var` where `val` suffices
- Missing KDoc on public APIs
- Large functions (>50 lines)
- Unnecessary mutable collections

## Automated Checks Run

```bash
# Static analysis
./gradlew detekt
./gradlew ktlintCheck

# Compile warnings
./gradlew compileKotlin -Werror

# Test + coverage
./gradlew test jacocoTestReport

# Dependency vulnerability
./gradlew dependencyCheckAnalyze
```

## Example Usage

```text
User: /kotlin-review

Agent:
# Kotlin Code Review Report

## Files Reviewed
- src/main/kotlin/com/example/service/AuthService.kt (modified)
- src/main/kotlin/com/example/controller/UserController.kt (modified)

## Static Analysis Results
✓ detekt: No issues
⚠ ktlint: 2 formatting issues (auto-fixable)

## Issues Found

[CRITICAL] Force Unwrap on User Input
File: src/main/kotlin/com/example/controller/UserController.kt:35
Issue: `!!` used on request parameter
```kotlin
val userId = request.getParameter("id")!!.toLong()  // Crashes on null!
```
Fix: Use safe call with error response
```kotlin
val userId = request.getParameter("id")?.toLongOrNull()
    ?: return ResponseEntity.badRequest().body("Invalid user ID")
```

[HIGH] Field Injection
File: src/main/kotlin/com/example/service/AuthService.kt:12
Issue: Using @Autowired field injection
```kotlin
@Autowired
lateinit var tokenProvider: TokenProvider
```
Fix: Constructor injection
```kotlin
@Service
class AuthService(
    private val tokenProvider: TokenProvider
)
```

## Summary
- CRITICAL: 1
- HIGH: 1
- MEDIUM: 0

Recommendation: ❌ Block merge until CRITICAL issue is fixed
```

## Approval Criteria

| Status | Condition |
|--------|-----------|
| ✅ Approve | No CRITICAL or HIGH issues |
| ⚠️ Warning | Only MEDIUM issues (merge with caution) |
| ❌ Block | CRITICAL or HIGH issues found |

## Integration with Other Commands

- Use `/kotlin-test` first to ensure tests pass
- Use `/kotlin-build` if build errors occur
- Use `/kotlin-review` before committing
- Use `/code-review` for non-Kotlin specific concerns

## Related

- Agent: `agents/kotlin-reviewer.md`
- Skills: `skills/kotlin-patterns/`, `skills/kotlin-testing/`
