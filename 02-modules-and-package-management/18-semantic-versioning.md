# Lesson 18 — Semantic Versioning

## What is Semantic Versioning?

Semantic Versioning uses:

```text
MAJOR.MINOR.PATCH
```

Example:

```text
3.7.4
```

Meaning:

```text
3 -> major
7 -> minor
4 -> patch
```

## PATCH

Bug fixes that should remain backward compatible.

```text
1.2.3 -> 1.2.4
```

## MINOR

New backward-compatible functionality.

```text
1.2.3 -> 1.3.0
```

## MAJOR

Breaking changes.

```text
1.2.3 -> 2.0.0
```

## Version Ranges

### Exact

```json
"express": "5.1.0"
```

Only that exact version.

### Caret

```json
"express": "^5.1.0"
```

Usually allows compatible changes within the current major version.

Conceptually:

```text
>=5.1.0 and <6.0.0
```

### Tilde

```json
"some-package": "~2.4.1"
```

Usually allows patch updates:

```text
>=2.4.1 and <2.5.0
```

## Why SemVer Matters

Imagine a dependency releases:

```text
2.3.1
```

then:
- 2.3.2 should be a fix
- 2.4.0 should add backward-compatible functionality
- 3.0.0 may break existing usage

This helps consumers estimate upgrade risk.

## Real-World Reality

SemVer is a contract convention, not a magical guarantee.

Packages can:
- make mistakes
- accidentally introduce regressions
- misuse version numbers

That is why lockfiles, tests, and CI are essential.

## Pre-release Versions

Examples:

```text
2.0.0-alpha.1
2.0.0-beta.2
2.0.0-rc.1
```

These indicate pre-release versions.

## Common Production Strategy

```text
package.json
   |
   v
version range
   |
package-lock.json
   |
   v
exact resolved dependency graph
```

This gives flexibility in package declarations but reproducibility in actual installs.

## Interview Questions

### What does ^1.2.3 mean?

It usually allows compatible updates that do not cross the next major version.

### Why isn't SemVer enough for reproducible builds?

Because ranges can resolve to newer versions over time. A lockfile records the exact dependency graph used.

## Interview-Ready Summary

```text
MAJOR -> breaking change
MINOR -> backward-compatible feature
PATCH -> backward-compatible fix
```

SemVer helps communicate compatibility expectations.
