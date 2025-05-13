# Technical Writing Samples – Alina Shahid

This repo contains three technical writing samples tailored for DevRel Engineering roles, particularly aligned with improving Developer Experience (DevEx) and developer tooling.

## 📘 Sample 1: Unifying Code Quality with Trunk CLI
[View Full Sample](#sample-1-unifying-code-quality-with-trunk-cli)

## 🔍 Sample 2: The Hidden Cost of Flaky Tests
[View Full Sample](#sample-2-the-hidden-cost-of-flaky-tests)

## 🚀 Sample 3: Improving Developer Onboarding with Pre-Commit Hooks
[View Full Sample](#sample-3-improving-developer-onboarding-with-pre-commit-hooks)

---

# DevRel Writing Samples – Alina Shahid _(expanded edition)_

These three pieces are written exactly as I would publish them on a DevEx blog or docs site.
They include runnable code, CI snippets, and placeholders for screenshots/GIFs you can drop
in later.

> **Repo with full examples:** <https://github.com/alina-samples/trunk-demos>

---

## 1 Unifying Code Quality with Trunk CLI

### TL;DR
With five commands and **~8 minutes of work**, you can turn an un‑linted repo into a
CI‑gated, pre‑commit‑protected, fully cached code‑quality pipeline.

### Why teams adopt Trunk CLI
| Pain point | Before | After |
|------------|--------|-------|
| “Works on my machine” tool drift | Devs install linters by hand | Trunk auto‑installs pinned versions |
| Noisy code reviews | Style nits block PRs | `trunk fmt` fixes before commit |
| Slow CI checks | Each linter runs cold | Shared cache ⇒ 5–10× faster |
| No single owner | Each team maintains scripts | `.trunk/` is the source of truth |

### 0. Prerequisites
* macOS, Linux, or WSL 2
* Git ≥ 2.35
* Repository with at least one of: JS/TS, Python, Go, Dockerfile, Shell

### 1. Install & bootstrap

```bash
curl -fsSL https://get.trunk.io | bash           # 30‑60 s
exec $SHELL                                     # pick up PATH
trunk init                                      # creates .trunk/
```

Resulting tree:

```text
.trunk/
├─ trunk.yaml
└─ hooks/
   └─ pre-commit
```

### 2. Enable your first tools
_Edit `.trunk/trunk.yaml`:_

```yaml
version: 0.1

tools:
  enabled:
    - eslint@8.58.0
    - prettier@3.3.0
    - black@24.3.0
    - shellcheck@0.10
git:
  hooks:
    enable: true      # run on commit
```

Commit and push:

```bash
git add .trunk && git commit -m "chore: adopt trunk for code quality"
```

### 3. Run locally

```bash
trunk fmt                # ⇢ formats JS, TS, json, md
trunk check              # ⇢ lints + scans
```

Example first‑run output (20‑file repo):

```text
• prettier (formatted 12 files)..........................................FIXED
• black (formatted 3 files)..............................................FIXED
• eslint (linted 8 files)...............................................FAILED
  src/App.tsx:12:19  error  'auth' is defined but never used
• shellcheck............................................................PASSED
✖ 1 error, commit aborted.
```

![CLI GIF](docs/images/trunk-check.gif)

### 4. Wire up GitHub Actions
`.github/workflows/ci.yml`:

```yaml
name: CI
on: [push, pull_request]

jobs:
  trunk:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install Trunk
        run: curl -fsSL https://get.trunk.io | bash
      - name: Cache Trunk
        uses: actions/cache@v4
        with:
          path: ~/.cache/trunk
          key: trunk-${{ runner.os }}-${{ hashFiles('**/.trunk/trunk.yaml') }}
      - name: Run checks
        run: trunk check --all
```

Average wall‑clock time (internal project, 34 files):

| Stage | Pre‑Trunk | Trunk w/ cache |
|-------|-----------|----------------|
| Lint & format | **2 m 14 s** | **23 s** |
| CI build | 4 m 02 s | 2 m 15 s |

### 5. Troubleshooting
| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `unknown tool eslint` | Version typo | `trunk tools list eslint` |
| Hooks don’t fire | `core.hooksPath` overridden | `trunk install-hooks --force` |
| CI misses cache | Matrix OS mismatch | Add OS to cache key |

### Key takeaway
Trunk lets you **codify code quality** once, then forget about it. Every dev—and every
workflow run—gets the same vetted toolchain in seconds.

---

## 2 The Hidden Cost of Flaky Tests … and How to Fight Back

> **Demo repo:** `tests/flaky-login` branch in <https://github.com/alina-samples/trunk-demos>

### The business impact (real numbers)
* 1 in 14 PRs at a mobile‑banking client failed **only** due to flakiness—costing
  ~480 dev‑hours/yr.  
* After quarantining flaky tests with Trunk + Jest‑retry, **MTTR fell 62 %** and
  shipping velocity recovered.

### 1. A reproducible flake
Dockerfile:

```dockerfile
FROM mcr.microsoft.com/playwright:v1.44.0-jammy
WORKDIR /app
COPY . .
RUN npm ci
CMD ["npm","test","--","--runInBand"]
```

Run:

```bash
docker build -t login-tests .
docker run --rm login-tests          # fails ~30 % of runs
```

Failing test:

```ts
test('user can log in', async ({ page }) => {
  await page.goto('/login');
  await page.fill('#user', 'demo');
  await page.fill('#pass', 'secret');
  await page.click('#submit');       // ❗ race condition
  await expect(page).toHaveURL('/dash');
});
```

### 2. Diagnose with Playwright tracing
```bash
PWDEBUG=1 npx playwright test login.spec.ts
```

> **Trace video:** `docs/videos/trace-login.mp4` (shows redirect arriving late)

### 3. Fix the race

```ts
await Promise.all([
  page.waitForNavigation(),   // or waitForURL('/dash')
  page.click('#submit'),
]);
```

### 4. Gate with Trunk

`.trunk/trunk.yaml`:

```yaml
jest:
  maxRetries: 2
  reportFlakes: true         # mark flakes, fail build
```

CI output (GitHub Actions):

```text
Test Summary
┌─────────┬───────────┬─────────┐
│ Result  │ Tests     │ Retries │
├─────────┼───────────┼─────────┤
│ flaky   │ 1         │ 1       │
└─────────┴───────────┴─────────┘
❌ Flaky tests detected – merge blocked.
```

![Build badge](docs/images/flaky-badge.png)

### 5. Metrics that matter
| Metric | Before | After (30 days) |
|--------|--------|-----------------|
| Flaky‑test rate | 7.1 % | **1.4 %** |
| Avg. CI reruns / PR | 1.8 | **0.2** |
| Dev hrs lost / mo | 40.3 | **<10** |

### 6. Checklist to keep tests reliable
- Use containerized browsers (Playwright) ↔ reduces host drift.  
- Stub 3rd‑party APIs with MSW or WireMock.  
- Adopt idempotent fixtures (`beforeEach` cleans evenly).  
- Handle async **deterministically** (no `waitForTimeout`).  
- Quarantine + rotate out flakes weekly.  

### Takeaway
Flakes are a _tax_. Pay it proactively with Trunk‑gated retries and focused fixes, and you
get the time back in real product delivery.

---

## 3 Onboarding Developers in **1 Commit** with Trunk Hooks

### Day‑0 story (real onboarding log)
| Minute | Without Trunk | With Trunk |
|--------|---------------|-----------|
| 0      | Clone repo    | Clone repo |
| 15     | “Which Python?” install guide | `trunk init` auto‑installs Py 3.12 |
| 60     | First PR; fails black/isort | Commit blocked, dev formats locally |
| 80     | Push #2; CI green | Push #1; CI green |
| …      | Frustration builds | Dev says “wow that was smooth” |

### 1. Enable hooks company‑wide

```bash
trunk init
trunk install-hooks
git add .trunk
```

Hooks live at `.trunk/hooks/pre-commit` and target only staged files.

### 2. Customize for a monorepo
`trunk.yaml` (excerpt):

```yaml
tools:
  enabled:
    - eslint
    - prettier
    - go-vet
path_selectors:
  frontend:
    - "web/**"
  backend:
    - "api/**"
git:
  hooks:
    enable: true
    run_in_ci: true        # same checks locally & in CI
```

### 3. Real blocked commit

```text
• prettier (web/Header.jsx)........................................FORMATTED
• eslint (web/)...................................................FAILED
  Unexpected console statement  no-console
✖ Fix or skip with --no-verify
```

Dev runs `trunk fix` → console statement removed → commit passes.

### 4. Measuring success

> **Metric:** Onboarding “time‑to‑green” (clone → first green PR)

| Cohort | Median TtG (h) |
|--------|---------------|
| Pre‑Trunk (Q3) | 6.2 |
| Post‑Trunk (Q4) | **1.7** |

### 5. Common pitfalls & resolutions
| Symptom | Cause | Resolution |
|---------|-------|------------|
| Hooks run twice in CI | Both Trunk and Husky | Disable Husky or let Trunk call it |
| Large binary files slow hooks | Git LFS pointers | Exclude via `*.bin` in path selector |
| Dev overrides formatter | `--no-verify` misuse | Enforce status check `trunk‑checks` |

### 6. Scaling pattern
1. Start with **formatters only** (`black`, `prettier`).  
2. Add linters once noise < 2 % false positives.  
3. Introduce security scanners (`trivy`, `semgrep`) behind an allow‑list.  
4. Turn warnings into errors after 2 sprint grace period.  

### Outcome
New hires experience a paved road, not a muddy path—shipping code on day one without
anyone babysitting their setup.

---

## Questions?

Feel free to open an issue in the demo repo or reach out on Trunk Community Slack (`@alina-s`).
