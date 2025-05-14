## Sample 2: The Hidden Cost of Flaky Tests

### 2 The Hidden Cost of Flaky Tests … and How to Fight Back

> **Demo repo:** [`tests/flaky-login`](https://github.com/alina-samples/trunk-demos/tree/tests/flaky-login)

---

### Table of Contents
1. [Why flakes matter](#1-why-flakes-matter)  
2. [A reproducible flake](#2-a-reproducible-flake)  
3. [Diagnosing with Playwright tracing](#3-diagnosing-with-playwright-tracing)  
4. [Root-cause analysis](#4-root-cause-analysis)  
5. [Fixing the race condition](#5-fixing-the-race-condition)  
6. [Quarantining & gating with Trunk](#6-quarantining--gating-with-trunk)  
7. [Measuring the impact](#7-measuring-the-before-and-after-impact)  
8. [A sustainable anti-flake checklist](#8-a-sustainable-anti-flake-checklist)  
9. [Lessons learned & next steps](#9-lessons-learned--next-steps)  
10. [Appendix A – reference pipeline YAML](#appendix-a--reference-pipeline-yaml)  
11. [Appendix B – suggested reading](#appendix-b--suggested-reading)

---

### The business impact (real numbers 📊)

| Metric                      | Raw Value | Annualized Cost |
|-----------------------------|-----------|-----------------|
| **Flaky PRs**               | 1 in 14   | —               |
| Dev hrs lost / PR rerun     | 1.5 h     | —               |
| **Total dev hrs / year**    | —         | **≈ 480 h**     |
| Eng. cost (blended $70/h)   | —         | **≈ $33.6 k**   |
| ⏱️ MTTR before              | 2 h 45 m  | —               |
| ⏱️ MTTR after               | **1 h 3 m (-62 %)** | — |

> Real data from a 32-dev mobile-banking team (Q1–Q2 2025).

---

## 1. Why flakes matter

*Flaky tests* fail nondeterministically, eroding trust in CI and hiding real regressions.

* Lost **flow state** while rerunning CI  
* **Merge paralysis** → larger PRs → harder reviews  
* **Hidden defects** buried by noise  
* Morale drain — nobody wants to babysit red builds  

---

## 2. A reproducible flake

Hermetic repro: Playwright in a container; fails ≈ 30 % of runs.

```dockerfile
FROM mcr.microsoft.com/playwright:v1.44.0-jammy
WORKDIR /app
COPY . .
RUN npm ci
CMD ["npm","test","--","--runInBand"]
```

```bash
docker build -t login-tests .
docker run --rm login-tests   # fails ~30 % of runs
```

```ts
test('user can log in', async ({ page }) => {
  await page.goto('/login');
  await page.fill('#user', 'demo');
  await page.fill('#pass', 'secret');
  await page.click('#submit');   // ❗ race condition
  await expect(page).toHaveURL('/dash');
});
```

---

## 3. Diagnosing with Playwright tracing

```bash
PWDEBUG=1 npx playwright test login.spec.ts
```

| Artifact                 | Purpose                     |
|--------------------------|-----------------------------|
| `trace.zip`              | Full trace bundle           |
| `trace.html`             | Self-contained viewer       |
| `videos/trace-login.mp4` | Slow-mo redirect delay demo |

---

## 4. Root-cause analysis

```
click(#submit) ─┬─> /api/auth (XHR) ─┬─> 302 /dash
                │                    │
                │      (slow) ───────┘
                └── assertion fires here (flaky)
```

* Auth API spikes → redirect arrives late  
* URL asserted **before** navigation completes  

---

## 5. Fixing the race condition

### A. Event-driven wait (✅ preferred)

```ts
await Promise.all([
  page.waitForNavigation({ url: '/dash' }),
  page.click('#submit'),
]);
```

### B. Retry assertion (🛠 stop-gap)

```ts
await page.click('#submit');
await expect(page).toHaveURL('/dash', { timeout: 5000 });
```

---

## 6. Quarantining & gating with Trunk

```yaml
# .trunk/trunk.yaml
jest:
  maxRetries: 2
  reportFlakes: true
  quarantineLabel: flaky
```

```
Test Summary
┌─────────┬───────────┬─────────┐
│ Result  │  Tests    │ Retries │
├─────────┼───────────┼─────────┤
│ flaky   │     1     │    1    │
└─────────┴───────────┴─────────┘
❌ Flaky tests detected – merge blocked
```

---

## 7. Measuring the impact

| KPI               | Before | After | Δ      |
|-------------------|--------|-------|-------:|
| Flaky-test rate   | 7.1 %  | **1.4 %** | ▼ 80 % |
| CI reruns / PR    | 1.8    | **0.2** | ▼ 89 % |
| Dev hrs lost / mo | 40.3   | **< 10** | ▼ 75 % |
| MTTR              | 2 h 45 m | **1 h 3 m** | ▼ 62 % |
| Deploy lead time  | 14 h   | **9 h** | ▼ 36 % |

---

## 8. A sustainable anti-flake checklist

| Theme        | Practice                        | Tooling                  |
|--------------|---------------------------------|--------------------------|
| **Env**      | Containerize browsers & DBs     | Playwright, Testcontainers |
|              | Pin dependencies               | `npm ci`, lockfiles      |
| **Async**    | Event-based waits              | `waitForSelector`, `waitForNavigation` |
| **Network**  | Stub 3rd-party APIs            | MSW, WireMock            |
| **Data**     | Idempotent fixtures            | Factories, `beforeEach`  |
| **CI**       | Retries ≤ 2 + quarantine       | Trunk                    |
| **Governance** | “Flake sheriff” rotation     | 1 dev / sprint           |

---

## 9. Lessons learned & next steps

1. Flakes are a **systems** problem — detect → quarantine → fix loop.  
2. Fast feedback > perfect tests; quarantine buys time.  
3. Instrument first, optimize second.  
4. Automate guardrails; humans miss nondeterminism.

**Next:** extend gating to mobile suite, pipe flake events to Slack, refactor helpers.

---

## 10. Appendix A – reference pipeline YAML

```yaml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci

      - name: Run tests
        env:
          PLAYWRIGHT_JUNIT_OUTPUT_NAME: results/pytest.xml
        run: |
          npx trunk run jest -- --ci --reporters=default --reporters=jest-junit

      - name: Upload traces
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-traces
          path: |
            **/trace.zip
            **/*.mp4
```

---

## 11. Appendix B – suggested reading

* **“Fighting Flaky Tests at Google”** — Google Testing Blog, 2022  
* **CITCON Talk:** _Contain the pain: quarantining vs. deleting flaky tests_  
* **Martin Fowler:** _Eradicating Non-Determinism in Tests_  
* **Trunk Docs:** _Flaky Test Insights & Governance_

---

### Takeaway 💡

Flaky tests are a **silent tax** on developer productivity.  
Detect ↔ isolate ↔ fix ↔ measure, powered by containerized tests plus Trunk-gated retries, and you reclaim engineering days every sprint for real product work.
