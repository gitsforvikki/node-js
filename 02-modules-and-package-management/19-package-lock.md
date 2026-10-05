# Lesson 19 — package-lock.json

## What is package-lock.json?

`package-lock.json` records the exact dependency tree resolved by npm.

If `package.json` says:

```json
{
  "dependencies": {
    "express": "^5.1.0"
  }
}
```

that is a range.

The lockfile records the exact version npm selected, together with transitive dependencies and integrity information.

## Why It Exists

Without a lockfile:

```text
Developer A installs today
Developer B installs next month
CI installs later
```

They may receive different dependency versions.

With a lockfile, npm can reproduce the resolved graph more reliably.

## Direct vs Transitive Dependencies

Direct:

```text
your app
   |
   v
express
```

Transitive:

```text
your app
   |
 express
   |
 another package
   |
 another dependency
```

The lockfile captures the larger tree.

## Why Commit package-lock.json?

For applications, generally commit it.

Benefits:
- reproducible builds
- stable CI
- easier debugging
- security auditing
- predictable deployments

## npm ci and Lockfiles

```bash
npm ci
```

depends heavily on the lockfile.

If `package.json` and lockfile disagree, CI can fail instead of silently changing the dependency graph.

That is desirable.

## Integrity Information

Lockfiles can contain integrity hashes used to verify downloaded package content.

This improves package-install consistency and supply-chain protection.

## Do Not Manually Edit It

Avoid hand-editing `package-lock.json`.

Use npm commands:

```bash
npm install
npm uninstall
npm update
```

and allow npm to maintain the file.

## Common Mistake

Deleting the lockfile whenever there is a dependency issue.

That can hide the original problem and produce a completely different dependency graph.

Delete and regenerate it only when you understand why.

## package.json vs package-lock.json

```text
package.json
  declares desired dependency ranges

package-lock.json
  records exact resolved dependency tree
```

## Interview Question

### Why should package-lock.json be committed?

Because it improves reproducibility by preserving the exact dependency graph used by the project.

## Summary

The lockfile is a key production artifact, not useless generated noise.

It helps make:

```text
local
CI
Docker
production
```

use the same dependency graph.
