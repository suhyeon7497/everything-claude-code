---
description: Fix Kotlin/Spring build errors, Gradle issues, and compiler warnings incrementally. Invokes the kotlin-build-resolver agent for minimal, surgical fixes.
---

# Kotlin Build and Fix

This command invokes the **kotlin-build-resolver** agent to incrementally fix Kotlin/Spring build errors with minimal changes.

## What This Command Does

1. **Run Diagnostics**: Execute `./gradlew compileKotlin`, `./gradlew test`
2. **Parse Errors**: Group by file and sort by severity
3. **Fix Incrementally**: One error at a time
4. **Verify Each Fix**: Re-run build after each change
5. **Report Summary**: Show what was fixed and what remains

## When to Use

Use `/kotlin-build` when:
- `./gradlew build` fails with errors
- Kotlin compiler reports type/null safety issues
- Gradle dependency resolution fails
- Spring Boot application fails to start
- After pulling changes that break the build
- KAPT annotation processing fails

## Diagnostic Commands Run

```bash
# Primary build check
./gradlew compileKotlin

# Full build with tests
./gradlew build

# Clean and rebuild
./gradlew clean build

# Dependency tree
./gradlew dependencies --configuration runtimeClasspath

# Spring Boot startup check
./gradlew bootRun
```

## Example Session

```text
User: /kotlin-build

Agent:
# Kotlin Build Resolution

## Initial Diagnostics

```bash
$ ./gradlew compileKotlin

e: src/main/kotlin/com/example/service/UserService.kt:25:15
  Unresolved reference: UserRepository
e: src/main/kotlin/com/example/controller/ApiController.kt:42:9
  Type mismatch: inferred type is String? but String was expected
e: src/main/kotlin/com/example/controller/ApiController.kt:58:2
  A 'return' expression required in a function with a block body
```

Errors found: 3

## Fix 1: Unresolved Reference

File: src/main/kotlin/com/example/service/UserService.kt:25
Error: Unresolved reference: UserRepository
Cause: Missing import

```kotlin
// Added import
import com.example.repository.UserRepository
```

```bash
$ ./gradlew compileKotlin
# 2 errors remaining
```

## Fix 2: Type Mismatch (Nullable)

File: src/main/kotlin/com/example/controller/ApiController.kt:42
Error: Type mismatch: String? vs String

```kotlin
// Changed
val name: String = request.getParameter("name")
// To
val name: String = request.getParameter("name") ?: ""
```

```bash
$ ./gradlew compileKotlin
# 1 error remaining
```

## Fix 3: Missing Return

File: src/main/kotlin/com/example/controller/ApiController.kt:58
Error: A 'return' expression required

```kotlin
fun getUser(id: Long): ResponseEntity<User> {
    val user = userService.findById(id)
    // Added missing return
    return ResponseEntity.ok(user)
}
```

```bash
$ ./gradlew compileKotlin
# Build successful!
```

## Final Verification

```bash
$ ./gradlew test
BUILD SUCCESSFUL

$ ./gradlew bootRun
# Application started on port 8080
```

## Summary

| Metric | Count |
|--------|-------|
| Build errors fixed | 3 |
| Warnings fixed | 0 |
| Files modified | 2 |
| Remaining issues | 0 |

Build Status: SUCCESS
```

## Common Errors Fixed

| Error | Typical Fix |
|-------|-------------|
| `Unresolved reference` | Add import or dependency |
| `Type mismatch` | Fix type or add null safety |
| `return expression required` | Add return statement |
| `Overrides nothing` | Fix function signature |
| `No value passed for parameter` | Add default value or argument |
| `Cannot access` | Fix visibility modifier |
| `Gradle dependency conflict` | Exclude transitive or force version |
| `KAPT error` | Fix annotation or add `correctErrorTypes` |

## Fix Strategy

1. **Compile errors first** - Code must compile
2. **Spring startup second** - Application must boot
3. **Test failures third** - Tests must pass
4. **Lint warnings fourth** - Style and best practices
5. **One fix at a time** - Verify each change
6. **Minimal changes** - Don't refactor, just fix

## Stop Conditions

The agent will stop and report if:
- Same error persists after 3 attempts
- Fix introduces more errors
- Requires architectural changes
- Missing external dependencies
- Spring Boot version incompatibility

## Related Commands

- `/kotlin-test` - Run tests after build succeeds
- `/kotlin-review` - Review code quality
- `/verify` - Full verification loop

## Related

- Agent: `agents/kotlin-build-resolver.md`
- Skill: `skills/kotlin-patterns/`
