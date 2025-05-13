## Sample 3: Improving Developer Onboarding with Pre-Commit Hooks

### Day‑0 story (real onboarding log)
| Minute | Without Trunk | With Trunk |
|--------|---------------|-----------|
| 0      | Clone repo    | Clone repo |
| 15     | “Which Python?” install guide | `trunk init` auto‑installs Py 3.12 |
| 60     | First PR; fails black/isort | Commit blocked, dev formats locally |
| 80     | Push #2; CI green | Push #1; CI green |
| …      | Frustration builds | Dev says “wow that was smooth” |

### 1. Enable hooks company‑wide

```bash
trunk init
trunk install-hooks
git add .trunk
```

Hooks live at `.trunk/hooks/pre-commit` and target only staged files.

### 2. Customize for a monorepo
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

### 3. Real blocked commit

```text
• prettier (web/Header.jsx)........................................FORMATTED
• eslint (web/)...................................................FAILED
  Unexpected console statement  no-console
✖ Fix or skip with --no-verify
```

Dev runs `trunk fix` → console statement removed → commit passes.

### 4. Measuring success

> **Metric:** Onboarding “time‑to‑green” (clone → first green PR)

| Cohort | Median TtG (h) |
|--------|---------------|
| Pre‑Trunk (Q3) | 6.2 |
| Post‑Trunk (Q4) | **1.7** |

### 5. Common pitfalls & resolutions
| Symptom | Cause | Resolution |
|---------|-------|------------|
| Hooks run twice in CI | Both Trunk and Husky | Disable Husky or let Trunk call it |
| Large binary files slow hooks | Git LFS pointers | Exclude via `*.bin` in path selector |
| Dev overrides formatter | `--no-verify` misuse | Enforce status check `trunk‑checks` |

### 6. Scaling pattern
1. Start with **formatters only** (`black`, `prettier`).  
2. Add linters once noise < 2 % false positives.  
3. Introduce security scanners (`trivy`, `semgrep`) behind an allow‑list.  
4. Turn warnings into errors after 2 sprint grace period.  

### Outcome
New hires experience a paved road, not a muddy path—shipping code on day one without
anyone babysitting their setup.

---

## Questions?

Feel free to open an issue in the demo repo or reach out on Trunk Community Slack (`@alina-s`).


---

