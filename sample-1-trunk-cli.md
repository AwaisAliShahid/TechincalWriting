
## Sample 1: Unifying Code Quality with Trunk CLI

### TL;DR
With five commands and **~8 minutes of work**, you can turn an un‑linted repo into a
CI‑gated, pre‑commit‑protected, fully cached code‑quality pipeline.

### Why teams adopt Trunk CLI
| Pain point | Before | After |
|------------|--------|-------|
| “Works on my machine” tool drift | Devs install linters by hand | Trunk auto‑installs pinned versions |
| Noisy code reviews | Style nits block PRs | `trunk fmt` fixes before commit |
| Slow CI checks | Each linter runs cold | Shared cache ⇒ 5–10× faster |
| No single owner | Each team maintains scripts | `.trunk/` is the source of truth |

### 0. Prerequisites
* macOS, Linux, or WSL 2
* Git ≥ 2.35
* Repository with at least one of: JS/TS, Python, Go, Dockerfile, Shell

### 1. Install & bootstrap

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

### 2. Enable your first tools
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

### 3. Run locally

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

### 4. Wire up GitHub Actions
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

Average wall‑clock time (internal project, 34 files):

| Stage | Pre‑Trunk | Trunk w/ cache |
|-------|-----------|----------------|
| Lint & format | **2 m 14 s** | **23 s** |
| CI build | 4 m 02 s | 2 m 15 s |

### 5. Troubleshooting
| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `unknown tool eslint` | Version typo | `trunk tools list eslint` |
| Hooks don’t fire | `core.hooksPath` overridden | `trunk install-hooks --force` |
| CI misses cache | Matrix OS mismatch | Add OS to cache key |

### Key takeaway
Trunk lets you **codify code quality** once, then forget about it. Every dev—and every
workflow run—gets the same vetted toolchain in seconds.



---
