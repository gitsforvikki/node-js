# Lesson 147 — Dependency Vulnerabilities in Node.js

## 1. What is a dependency?

A **dependency** is code your application installs from another package. For example:

```bash
npm install express mongoose
```

Your app directly uses Express and Mongoose, but they also depend on other packages called **transitive dependencies**.

```text
Your Node.js app
  |
  +-- express (direct dependency)
  |      +-- other packages (transitive)
  |
  +-- mongoose (direct dependency)
         +-- MongoDB driver
         +-- other packages
```

A security bug in a direct **or transitive** dependency may affect your application.

## 2. What is a vulnerability?

A vulnerability is a weakness that could allow unintended behavior, such as data leakage, denial of service, code execution, or bypassing a security check.

A vulnerability report usually includes:
- **Affected package and version range**
- **Severity** (low, moderate, high, critical)
- **Conditions required for exploitation**
- **Patched version or mitigation**
- **Advisory/CVE/GHSA identifier**

**Severity alone is not the full risk.** Ask whether your app actually uses the affected feature and whether the vulnerable code is reachable.

## 3. Inspect installed packages

```bash
npm ls --depth=0     # Direct dependencies
npm ls               # Full dependency tree
npm outdated         # Available newer versions
```

These commands help you understand what is installed and which packages need review.

## 4. Run npm audit

```bash
npm audit
```

This checks your dependency tree against known advisories available to npm.

For machine-readable output:

```bash
npm audit --json
```

To focus on production dependencies:

```bash
npm audit --omit=dev
```

**Important:** A clean audit is **not proof** that your app is secure. Unknown vulnerabilities, malicious packages, and application-level bugs may not be detected.

## 5. How to fix vulnerabilities safely

Recommended workflow:

1. Read the advisory: which package/version is affected?
2. Find how the package entered your app (`npm explain package-name`).
3. Check whether a fixed version exists.
4. Update the direct dependency or the relevant parent dependency.
5. Run tests and verify behavior.
6. Re-run audit.
7. Deploy the patched version and monitor.

```bash
npm explain some-package
npm update
npm audit
npm test
```

Avoid automatically running `npm audit fix --force` without review. It may install breaking major versions and cause application failures.

## 6. package.json vs package-lock.json

`package.json` declares acceptable version ranges. `package-lock.json` records the resolved dependency tree so teammates/CI can install more reproducibly.

Example:

```json
{
  "dependencies": {
    "express": "^5.0.0"
  }
}
```

The caret `^` usually permits compatible minor/patch updates within the same major version for stable releases. The lockfile pins what actually gets installed.

For CI:

```bash
npm ci
```

`npm ci` installs according to the lockfile and fails when it is out of sync with `package.json`. Commit the lockfile for applications.

## 7. Supply-chain risks

Not every risk is a known CVE. Packages can be compromised or intentionally malicious.

Examples:
- Typosquatting: installing a similarly named malicious package.
- Compromised maintainer account: malicious version published.
- Install scripts: package executes code during installation.
- Abandoned packages: unpatched vulnerabilities.

Mitigations:
- Check package name, publisher, maintenance and release history.
- Keep dependencies minimal.
- Review unexpected dependency changes in pull requests.
- Use lockfiles and restricted CI permissions.
- Use automated dependency update tools and secret scanning.

## 8. Automated updates with Dependabot

GitHub Dependabot can propose security and version updates automatically.

Example `.github/dependabot.yml`:

```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
```

Review each pull request and run tests before merging. Security alerts and automatic update features depend on repository settings and availability.

## 9. CI security checks

Example GitHub Actions workflow:

```yaml
name: Node security check
on: [push, pull_request]
jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: "npm"
      - run: npm ci
      - run: npm audit --audit-level=high
      - run: npm test --if-present
```

**Note:** This is a learning example. In production, review/pin action versions appropriately, set minimal GitHub token permissions, and decide which advisories should block a build. Audit availability and exit behavior can depend on the registry/network.

## 10. Prioritize what to fix

| Situation | Suggested action |
| --- | --- |
| Exploitable vulnerability in production request path | Urgent patch/mitigation |
| High-severity vulnerability in unused dev tooling | Assess reachability and patch appropriately |
| Patched version is a breaking major release | Plan tests, migration and temporary mitigation |
| No fix available | Reduce exposure, replace package, or isolate feature |

Track exceptions with an owner and review date. Don't permanently ignore warnings without investigation.

## 11. Common mistakes

1. Ignoring transitive dependencies.
2. Running `npm audit fix --force` without testing.
3. Assuming a clean audit means zero risk.
4. Committing `node_modules` instead of using a lockfile.
5. Installing similarly named packages without checking them.
6. Never reviewing CI dependencies and GitHub Actions.
7. Updating packages without deploying the updated build.

## 12. Practice

1. Run `npm ls --depth=0`, `npm outdated`, and `npm audit` in a practice project.
2. Explain direct vs transitive dependencies.
3. Compare `npm install` and `npm ci`.
4. Add Dependabot configuration to a test repository.
5. Explain why dependency updates require both security review and regression testing.

**Interview answer:** Dependency vulnerabilities are weaknesses in third-party packages, including transitive packages. Reduce risk with lockfiles, vulnerability scanning, controlled updates, minimal permissions, CI checks, and prompt patching guided by exploitability and impact.
