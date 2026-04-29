---
description: 'Guidelines for writing self-explanatory code with minimal comments'
applyTo: '**/*.java'
---

# Self-explanatory Code

## Core Principle

**Write code that speaks for itself. Comment only when necessary to explain WHY, not WHAT.**

## Commenting Guidelines

### ❌ AVOID

- **Obvious comments**: `counter++; // Increment counter`
- **Redundant comments**: `return user.name; // Return the user's name`
- **Commented-out code**: Use git history instead
- **Changelog comments**: `// Modified by John on 2023-01-15`

### ✅ WRITE

- **Complex business logic**: Explain WHY this specific calculation or rule
- **Non-obvious algorithms**: Explain the algorithm choice
- **API constraints or gotchas**: External limitations, rate limits
- **Regex patterns**: What the pattern matches
- **Constants with reasoning**: `MAX_RETRIES = 3; // Based on network reliability studies`

### Annotations

```java
// TODO: Replace with proper implementation after security review
// FIXME: Memory leak - investigate connection pooling
// NOTE: This assumes UTC timezone for all calculations
```

## Decision Framework

Before writing a comment, ask:

1. **Is the code self-explanatory?** → No comment needed
2. **Would a better variable/method name eliminate the need?** → Refactor instead
3. **Does this explain WHY, not WHAT?** → Good comment

## Public APIs

Use Javadoc for public interfaces and service methods. Skip Javadoc for private/internal methods where the name is sufficient.

