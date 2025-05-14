## Sample 3: Improving Developer Onboarding with Pre-Commit Hooks

### The Problem

New engineers often spend their first hours fighting inconsistent tooling:

* *“Which Python/node/go version do I need?”*  
* Mis-configured editors that butcher formatting  
* Flaky CI red builds for issues that could have been caught locally  

The result is a **high “time-to-green” (TtG)** — the elapsed time from `git clone` to the first pull-request that passes CI. Our goal: pave that road so a new hire ships code **on day one**.

---

### Table of Contents
1. [Day-0 story (before vs. after)](#day-0-story)  
2. [Enabling hooks company-wide](#2-enabling-hooks-company-wide)  
3. [Customising hooks for a monorepo](#3-customising-hooks-for-a-monorepo)  
4. [Developer workflow in action](#4-developer-workflow-in-action)  
5. [Measuring success](#5-measuring-success)  
6. [Common pitfalls & resolutions](#6-common-pitfalls--resolutions)  
7. [Scaling pattern](#7-scaling-pattern)  
8. [Outcomes & lessons learned](#8-outcomes--lessons-learned)  
9. [Appendix A – full `trunk.yaml`](#appendix-a--full-trunkyaml)  
10. [Appendix B – suggested reading](#appendix-b--suggested-reading)

---

## Day-0 story

A **real onboarding log** for a junior backend dev (Python) joining the team last quarter.

| Minute | Without Trunk | With Trunk |
|:------:|---------------|-----------|
| 0 m    | `git clone`   | `git clone` |
| 15 m   | “Which Python?” README spelunking | `trunk init` auto-installs Python 3.12 |
| 40 m   | Local tests fail — missing `black` | Hooks auto-install formatters |
| 60 m   | First PR; fails `black` + `isort` in CI | Commit blocked, dev formats locally |
| 80 m   | Push #2; finally green | Push #1; CI green |
| 90 m   | **Frustration builds** | Dev says “wow, that was smooth” |

Median TtG dropped from **6.2 h → 1.7 h** after rolling out hooks (see §5).

---

## 2. Enabling hooks company-wide

```bash
# initialise Trunk in the repo (creates .trunk/)
trunk init

# install git hooks under .git/hooks
trunk install-hooks

# commit the Trunk config so everyone gets it
git add .trunk
git commit -m "chore: enable Trunk pre-commit hooks"
```

Hooks reside in `.trunk/hooks/pre-commit` and run **only on staged files**, keeping feedback fast.

---

## 3. Customising hooks for a monorepo

Below is a trimmed `trunk.yaml`.  
Key ideas:

* **Path selectors** divide the repo into logical areas (frontend vs. backend).  
* **Tool enable list** keeps noise low; add linters gradually.  
* **`run_in_ci:true`** guarantees parity — same checks locally & in CI.

```yaml
tools:
  enabled:
    - eslint
    - prettier
    - go-vet
    - black
    - isort

path_selectors:
  frontend:
    - "web/**"
  backend:
    - "api/**"

git:
  hooks:
    enable: true
    run_in_ci: true
```

> **Tip:** Use `trunk check --select frontend` to lint only the React app on demand.

---

## 4. Developer workflow in action

A **real blocked commit** captured from the React team:

```
• prettier (web/Header.jsx)................................FORMATTED
• eslint (web/)...........................................FAILED
  Unexpected console statement  no-console
✖ Fix or skip with --no-verify
```

The dev runs:

```bash
trunk fix                 # auto-applies eslint --fix
git add web/Header.jsx
git commit -m "feat(ui): remove debug log"
```

Commit now passes; the push is green on the first try.

---

## 5. Measuring success

### KPI: Onboarding “time-to-green” (clone → first green PR)

| Cohort            | Median TtG (hours) | Δ |
|-------------------|--------------------|--:|
| **Pre-Trunk (Q3)**| 6.2                | — |
| **Post-Trunk (Q4)**| **1.7**           | ↓ 72 % |

Additional signals (30-day window):

| Metric                       | Before | After | Δ |
|------------------------------|--------|-------|---:|
| % PRs failing only style     | 23 %   | **4 %** | ↓ 83 % |
| Avg. CI minutes / PR         | 14.1   | **8.3** | ↓ 41 % |
| New-hire satisfaction (CSAT) | 3.2 / 5| **4.6 / 5** | +1.4 |

---

## 6. Common pitfalls & resolutions

| Symptom                                     | Root Cause                   | Resolution |
|---------------------------------------------|------------------------------|------------|
| Hooks run twice in CI                       | Husky + Trunk both active    | Disable Husky or let Trunk invoke it (`git.husky_proxy:true`) |
| Large binary files make hooks slow          | Git LFS pointers still staged| Exclude via `*.bin`, `*.png` in `path_selectors` |
| Dev bypasses hooks with `--no-verify`       | Lack of gate in CI           | Require status check `trunk-checks` before merge |
| Node/Go/Py versions drift per machine       | Unpinned toolchain           | Use `.tool-versions` or asdf via Trunk’s `tools.autoinstall` |

---

## 7. Scaling pattern

1. **Start with formatters only** (`black`, `prettier`) → fast, zero-false-positive.  
2. **Add linters** once noisy warnings < 2 % (eslint, `go vet`).  
3. **Introduce security scanners** (`trivy`, `semgrep`) behind an allow-list.  
4. **Escalate severities**: warnings → errors after a two-sprint grace period.  
5. **Automate fixes** (`trunk fix`, `eslint --fix`, `goimports`) to maintain flow.

---

## 8. Outcomes & lessons learned

### Outcomes

* **New hires ship code on day-one** — no more “install-fest”.  
* CI noise evaporates; senior devs review logic, not whitespace.  
* Toolchain parity fosters a culture of *“green means green”*.

### Lessons

1. **Small, fast feedback wins hearts.** Even 200 ms hooks feel snappy.  
2. **Local ≅ CI parity** prevents “works on my laptop” bugs.  
3. **Path selectors** are critical in polyglot repos — lint only what changed.  
4. Roll-outs succeed when **noise is < 5 %**; audit false-positives early.

---

## Appendix A – full `trunk.yaml`

<details>
<summary>Click to expand</summary>

```yaml
# Full config used in production
version: 0.1

tools:
  enabled:
    - eslint
    - prettier
    - black
    - isort
    - go-vet
    - trivy     # container / vuln scanning
    - semgrep

path_selectors:
  frontend:
    - "web/**"
  backend:
    - "api/**"
  terraform:
    - "infra/**.tf"

git:
  hooks:
    enable: true
    run_in_ci: true
    parallel: 4          # speed up on multi-core laptops

ci:
  skip_patterns:
    - "docs/**"
    - "*.md"

notifications:
  slack:
    channel: "#trunk-alerts"
    on_fail: true
```
</details>

---

## Appendix B – suggested reading

* **“Monorepo Pre-Commit Strategies”** — Trunk Blog, 2024  
* **Google EngProd:** _Standardised Toolchains at Scale_ (video)  
* **Fowler:** _Continuous Integration_ — section on developer ergonomics  
* **OWASP Cheat Sheet:** _Pre-Commit Security Scanning_  

---

### Questions?

Open an issue in the [demo repo](https://github.com/alina-samples/trunk-demos) or ping me on Trunk Community Slack (`@alina-s`).

---

### TL;DR 💡

Pre-commit hooks turn onboarding from a muddy path into a paved highway.  
With Trunk:

* `git clone` → `trunk init`  
* First commit is green  
* Engineers focus on features, not formatting.
