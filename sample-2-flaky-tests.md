## Sample 2: The Hidden Cost of Flaky Tests

## 2 The Hidden Cost of Flaky Tests … and How to Fight Back

> **Demo repo:** `tests/flaky-login` branch in <https://github.com/alina-samples/trunk-demos>

### The business impact (real numbers)
* 1 in 14 PRs at a mobile‑banking client failed **only** due to flakiness—costing
  ~480 dev‑hours/yr.  
* After quarantining flaky tests with Trunk + Jest‑retry, **MTTR fell 62 %** and
  shipping velocity recovered.

### 1. A reproducible flake
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

### 2. Diagnose with Playwright tracing
```bash
PWDEBUG=1 npx playwright test login.spec.ts
```

> **Trace video:** `docs/videos/trace-login.mp4` (shows redirect arriving late)

### 3. Fix the race

```ts
await Promise.all([
  page.waitForNavigation(),   // or waitForURL('/dash')
  page.click('#submit'),
]);
```

### 4. Gate with Trunk

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

### 5. Metrics that matter
| Metric | Before | After (30 days) |
|--------|--------|-----------------|
| Flaky‑test rate | 7.1 % | **1.4 %** |
| Avg. CI reruns / PR | 1.8 | **0.2** |
| Dev hrs lost / mo | 40.3 | **<10** |

### 6. Checklist to keep tests reliable
- Use containerized browsers (Playwright) ↔ reduces host drift.  
- Stub 3rd‑party APIs with MSW or WireMock.  
- Adopt idempotent fixtures (`beforeEach` cleans evenly).  
- Handle async **deterministically** (no `waitForTimeout`).  
- Quarantine + rotate out flakes weekly.  

### Takeaway
Flakes are a _tax_. Pay it proactively with Trunk‑gated retries and focused fixes, and you
get the time back in real product delivery.

---
