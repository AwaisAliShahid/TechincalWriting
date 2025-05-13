# TechincalWriting

# DevRel Writing Samples – Awais Ali Shahid

---

## 1  Unifying Code Quality with Trunk CLI

### Overview
Fast-moving teams quickly drift into **inconsistent code quality**—different linter versions, forgotten formatters, tool sprawl.  
**Trunk CLI** offers one unified interface to install, run, and cache every check locally and in CI.

### Why This Matters
* One command (`trunk check`) replaces “remember to run five tools.”
* Consistent results eliminate noisy code-review comments.
* Auto-install keeps onboarding to a single step.

### Prerequisites
* macOS, Linux, or WSL
* Git repo with at least one language Trunk supports
* CI provider of your choice (optional)

### Step&nbsp;1 Install Trunk CLI
```bash
curl -fsSL https://get.trunk.io -o install_trunk.sh && bash install_trunk.sh
```

### Step&nbsp;2 Initialize in Your Repository
```bash
trunk init
```
Trunk creates a **`.trunk/`** directory and pre-commit hooks.

### Step&nbsp;3 Configure Tools
`.trunk/trunk.yaml`:
```yaml
version: 0.1

tools:
  enabled:
    - eslint@8.58.0   # JavaScript/TypeScript
    - black@24.3.0    # Python formatter
    - shellcheck@0.10 # Shell script linter
    - trivy@0.51.0    # Container/image scanning
```

### Step&nbsp;4 Run Checks
```bash
# Run every enabled tool
trunk check

# Auto-format staged files
trunk fmt
```

**Sample output**:
```text
• black...........................................................PASSED
• eslint (8 files, cached 6)......................................FAILED
  src/App.tsx:12:19  error  'auth' is defined but never used
• shellcheck......................................................PASSED
• trivy (Dockerfile)..............................................PASSED
```

### Benefits for Teams
| Benefit                 | Impact on DevEx                         |
|-------------------------|-----------------------------------------|
| Single source of truth  | No “works on my machine” issues         |
| Zero onboarding steps   | New devs commit in minutes              |
| Caching + parallel exec | Checks finish > 5× faster in CI         |
| Native CI integrations  | One-line add for GitHub Actions, Circle |

### Next Steps
* **Docs:** <https://docs.trunk.io>  
* **Example repo:** <https://github.com/alina-samples/trunk-demo>

---

## 2  The Hidden Cost of Flaky Tests — And How to Fight Back

### Overview
Flaky tests undermine developer confidence and delay releases. This post shows **how to reproduce, diagnose, and fix** a real flaky test—and how Trunk can gate CI to catch it early.

### Why Flaky Tests Are Dangerous
* **Erodes trust:** developers ignore red builds.  
* **Slows CI/CD:** wasted debug hours.  
* **Hurts morale:** “green build” loses meaning.

### Reproducing a Flaky Login Test (Jest + Playwright)
`login.spec.ts`:
```ts
test('user can log in', async ({ page }) => {
  await page.goto('/login');
  await page.fill('#user', 'demo');
  await page.fill('#pass', 'secret');
  await page.click('#submit');           // ❗ fails intermittently
  await expect(page).toHaveURL('/dash');
});
```

Typical failure output:
```text
expect(received).toHaveURL('/dash')
Received: '/login?redirect=/dash'
```

Root cause: race condition—navigation completes before server redirect.

### Fix: Stabilize with `waitForURL`
```ts
await Promise.all([
  page.waitForURL('/dash'),
  page.click('#submit'),
]);
```

### Catching Flakes with Trunk
1. Enable **Jest retry** plug‑in in `.trunk/trunk.yaml`:
   ```yaml
   jest:
     maxRetries: 2
   ```
2. Trunk marks the test as **flaky** if retries pass, quarantining it and failing CI until fixed.

> **Result:** build stays green, and devs see an actionable flaky‑test report.

### Strategies to Eliminate Flakiness
1. **Isolate external services** (stub network calls).  
2. **Lock environments** (Docker, `.nvmrc`, `.python-version`).  
3. **Retry at framework level** (temporary mitigation).  
4. **Gate merges** with Trunk’s flaky‑test detection.

### Conclusion
Treat flaky tests as a **DevEx bug**. By catching and fixing them early—especially with Trunk’s automated gating—you protect velocity, code quality, and team morale.

---

## 3  Improving Developer Onboarding with Pre‑Commit Hooks

### Overview
The first commit experience shapes how fast a new hire becomes productive. Pre‑commit hooks via Trunk provide **guardrails, not gates**, blocking bad code before it hits CI.

### The Problem
A new developer:  
1. Clones the repo.  
2. Commits code.  
3. CI fails on formatting they didn’t know about.

### Quick Fix with Trunk
```bash
trunk init            # installs hooks automatically
trunk install-hooks   # re‑install if needed
```
Now every `git commit` triggers the configured checks.

**Sample blocked commit**:
```text
• black...........................................................FAILED
  src/utils.py: formatted
✖ Commit aborted – fix issues and re‑stage files.
```

### Full `.trunk/trunk.yaml` Snippet
```yaml
tools:
  enabled:
    - prettier        # JS/TS/CSS/MD
    - black
    - go-vet
    - hadolint
git:
  hooks:
    enable: true      # run on commit
```

### Developer Journey Impact
| Stage       | Without Trunk                              | With Trunk + Hooks                         |
|-------------|--------------------------------------------|-------------------------------------------|
| Activation  | “Wait, what tools do I need?”             | `trunk init` installs everything          |
| Adoption    | Push breaks CI; frustration                | Issues caught locally, quick feedback     |
| Retention   | Trust in tooling erodes over time          | Devs feel supported, velocity stays high  |

### Conclusion
Pre‑commit hooks via Trunk turn onboarding into a **one‑command setup**, catching issues at the earliest, cheapest stage. New developers ship code on day one—confident and unblocked.
