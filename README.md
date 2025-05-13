# Technical Writing Samples – Alina Shahid

This repo contains three technical writing samples tailored for DevRel Engineering roles, particularly aligned with improving Developer Experience (DevEx) and developer tooling.

## 📘 Sample 1: Unifying Code Quality with Trunk CLI
[View Full Sample](#sample-1-unifying-code-quality-with-trunk-cli)

## 🔍 Sample 2: The Hidden Cost of Flaky Tests
[View Full Sample](#sample-2-the-hidden-cost-of-flaky-tests)

## 🚀 Sample 3: Improving Developer Onboarding with Pre-Commit Hooks
[View Full Sample](#sample-3-improving-developer-onboarding-with-pre-commit-hooks)

---

## Sample 1: Unifying Code Quality with Trunk CLI

Maintaining consistent code quality across teams is one of the most common developer experience (DevEx) challenges in modern software development. As teams scale, so does the risk of misaligned linter configurations, toolchain drift, and code review fatigue. Trunk CLI solves this problem by centralizing and enforcing code standards using a single declarative configuration and smart automation.

### Why This Matters

Code quality enforcement shouldn’t depend on each developer remembering to run five different tools. Trunk centralizes these tools, provides auto-installation, and supports consistent results locally and in CI.

### Setting Up Trunk in Your Repository

**Install Trunk CLI**

```bash
curl -fsSL https://get.trunk.io -o install_trunk.sh && bash install_trunk.sh
```

**Initialize Trunk**

```bash
trunk init
```

**Enable tools**

```yaml
tools:
  enabled:
    - eslint
    - black
    - shellcheck
    - trivy
```

**Run checks locally**

```bash
trunk check
trunk fmt
```

### Use in CI Pipelines

```yaml
- name: Install Trunk
  run: curl -fsSL https://get.trunk.io | bash
- name: Run checks
  run: trunk check --all
```

### Benefits

- One config for all tools
- Auto-installation of exact versions
- Pre-commit and CI parity
- Cached runs for fast feedback

### Conclusion

Trunk eliminates the friction of maintaining consistent tooling across environments, making code quality effortless and enforceable from day one.

---

## Sample 2: The Hidden Cost of Flaky Tests

Flaky tests — those that fail randomly — are among the most frustrating DevEx issues. They erode developer trust, delay releases, and often go unresolved.

### Why Flaky Tests Are Dangerous

- Wasted CI time
- Lowered developer confidence
- Merge delays and blocked deploys

### Common Causes

- Async timing issues
- Environment drift between CI and local
- Shared global state between tests

### Reproducible Example

```javascript
test('user can log in', async () => {
  await page.goto('/login');
  await page.click('#submit'); // race condition
  expect(await page.url()).toBe('/dashboard');
});
```

### Solution

```javascript
await Promise.all([
  page.waitForNavigation(),
  page.click('#submit'),
]);
```

### Strategy

- Retry failed tests in CI
- Mark and quarantine flaky tests
- Run `trunk check` locally to surface failures early

### Conclusion

Flaky tests are a DevEx problem. Reducing them improves trust, release confidence, and team morale.

---

## Sample 3: Improving Developer Onboarding with Pre-Commit Hooks

Onboarding friction often stems from invisible standards — formatting, linting, and security checks. Trunk helps enforce them automatically with pre-commit hooks.

### The Problem

- Devs make first commit → CI fails
- Time wasted debugging unshared conventions

### Trunk Setup for Hooks

```bash
trunk init
trunk install-hooks
```

```yaml
git:
  hooks:
    enable: true
    run_in_ci: true
```

### Example Output

```text
• prettier (formatted 3 files)..........................FIXED
• eslint (found 2 issues)...............................FAILED
✖ Commit blocked
```

### Dev Journey Impact

| Stage      | Without Hooks   | With Trunk Hooks     |
|------------|------------------|-----------------------|
| Activation | CI failures      | Pre-commit feedback   |
| Adoption   | Manual setup     | Auto-configured tools |
| Retention  | Tool drift       | Consistent workflow   |

### Conclusion

Pre-commit hooks with Trunk enforce team standards before CI ever runs — making onboarding smooth, predictable, and fast.

---
