# ADM Silver Layer — Full CI/CD Security Automation Guide
**Scope:** `apps-azure` (React/TS) · `apis-azure` (Python/FastAPI) · `adm-foundation-layer-agents-azure` (Python/Azure Functions)  
**Platform:** GitLab (self-hosted, closed-source) 
**Last updated:** 2026-05-05

---

## Table of Contents

1. [Strategy Overview](#1-strategy-overview)
2. [Layer 1 — Pre-commit Hooks (Local Dev)](#2-layer-1--pre-commit-hooks-local-dev)
3. [Layer 2 — GitLab CI Pipeline (Server-side Gates)](#3-layer-2--gitlab-ci-pipeline-server-side-gates)
4. [Per-repo Setup](#4-per-repo-setup)
   - [apis-azure (FastAPI backend)](#41-apis-azure-fastapi-backend)
   - [adm-foundation-layer-agents-azure (Azure Functions)](#42-adm-foundation-layer-agents-azure-azure-functions)
   - [apps-azure (React/TypeScript frontend)](#43-apps-azure-reacttypescript-frontend)
5. [Secret Scanning Deep Dive](#5-secret-scanning-deep-dive)
6. [SAST Deep Dive](#6-sast-deep-dive)
7. [Merge Request Gate Policy](#7-merge-request-gate-policy)
8. [GitLab License Note](#8-gitlab-license-note)
9. [Rollout Plan for Your Team](#9-rollout-plan-for-your-team)
10. [Tool Reference Summary](#10-tool-reference-summary)

---

## 1. Strategy Overview

Two layers work together. Neither is optional:

```
Developer machine                  GitLab server
─────────────────                  ──────────────────────────────────────
pre-commit hooks          →  git push  →  .gitlab-ci.yml pipeline
  - secret scan (gitleaks)              - SAST (bandit / semgrep)
  - lint (ruff / eslint)                - secret scan (gitleaks CI)
  - type check (mypy / tsc)             - dependency audit (pip-audit / npm audit)
  - format check (black / prettier)     - lint + type check
  - no verify=False check               - coverage gate
  - no localStorage check               - merge blocked on any failure
```

**Why both layers?**
- Pre-commit catches issues at the source — before they ever reach GitLab. Fast, free, instant feedback.
- CI pipeline is the hard enforcement gate. Devs cannot bypass it. Even if someone skips pre-commit, the pipeline fails the merge.

---

## 2. Layer 1 — Pre-commit Hooks (Local Dev)

### 2.1 Install pre-commit framework

Every developer runs this once after cloning any repo:

```bash
pip install pre-commit
```

Add `pre-commit` to your `requirements-dev.txt` or `pyproject.toml` dev dependencies so it's part of the standard dev setup.

### 2.2 Root `.pre-commit-config.yaml`

Create this file at the root of **each** repo. Full config for Python repos (apis-azure and agents-azure) — frontend section covered separately.

```yaml
# .pre-commit-config.yaml
# Run: pre-commit install  (once per developer, per repo)
# Run all manually: pre-commit run --all-files

repos:

  # ── 1. SECRET SCANNING ───────────────────────────────────────────────────
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.4
    hooks:
      - id: gitleaks
        name: "🔑 Secret scan (gitleaks)"
        # Blocks: hardcoded passwords, API keys, tokens, connection strings

  # ── 2. GENERAL FILE HYGIENE ──────────────────────────────────────────────
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-toml
      - id: check-merge-conflict
      - id: check-added-large-files
        args: ['--maxkb=500']
      - id: detect-private-key          # catches PEM/RSA keys
      - id: no-commit-to-branch
        args: ['--branch', 'main', '--branch', 'master']

  # ── 3. PYTHON FORMATTING (Black) ─────────────────────────────────────────
  - repo: https://github.com/psf/black
    rev: 24.4.2
    hooks:
      - id: black
        name: "🎨 Format check (black)"
        language_version: python3

  # ── 4. PYTHON LINT + STYLE (Ruff — replaces flake8/isort/pyflakes) ───────
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.7
    hooks:
      - id: ruff
        name: "🔍 Lint (ruff)"
        args: [--fix]
      - id: ruff-format

  # ── 5. PYTHON TYPE CHECK (mypy) ──────────────────────────────────────────
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.10.0
    hooks:
      - id: mypy
        name: "🧠 Type check (mypy)"
        additional_dependencies: [types-requests, types-PyYAML]
        args: [--ignore-missing-imports, --no-strict-optional]

  # ── 6. CUSTOM SECURITY PATTERN CHECKS ───────────────────────────────────
  - repo: local
    hooks:

      - id: no-verify-false
        name: "🚫 Block verify=False"
        language: pygrep
        entry: 'verify\s*=\s*False'
        types: [python]
        files: '\.(py)$'

      - id: no-fstring-sql
        name: "🚫 Block f-string SQL"
        language: pygrep
        # Catches: f"SELECT", f'INSERT', f"UPDATE", f"DELETE", f"FROM {
        entry: 'f["\''](SELECT|INSERT|UPDATE|DELETE|DROP|CREATE|ALTER|FROM\s)'
        types: [python]
        files: '\.(py)$'

      - id: no-hardcoded-passwords
        name: "🚫 Block hardcoded password patterns"
        language: pygrep
        entry: '(password|passwd|pwd|secret|api_key|token)\s*=\s*["\'][^"\']{4,}["\']'
        types: [python]
        files: '\.(py)$'
        exclude: '(test_|_test|\.example|conftest)'

      - id: no-real-env-commit
        name: "🚫 Block committing real .env files"
        language: fail
        entry: "Real .env file detected — commit .env.example instead"
        files: '^\.env$'

      - id: no-localstorage
        name: "🚫 Block localStorage usage"
        language: pygrep
        entry: 'localStorage\.(setItem|getItem)'
        types_or: [javascript, jsx, ts, tsx]
        files: '\.(js|jsx|ts|tsx)$'
        exclude: '(\.example|\.test\.|\.spec\.)'
```

### 2.3 Developer onboarding command (one-time)

```bash
# After cloning the repo
pip install pre-commit
pre-commit install
pre-commit install --hook-type commit-msg   # optional: enforce commit message format
```

After this, every `git commit` will auto-run all the checks above before the commit goes through.

### 2.4 Running manually

```bash
# Run against all files (good for first-time setup verification)
pre-commit run --all-files

# Run one specific hook
pre-commit run no-verify-false --all-files
pre-commit run gitleaks --all-files
```

### 2.5 gitleaks config (custom rules for ADM)

Create `.gitleaks.toml` at the root of each repo to extend default gitleaks rules:

```toml
# .gitleaks.toml
title = "ADM Silver Layer Gitleaks Config"

[extend]
# Extend default ruleset
useDefault = true

[[rules]]
id = "adm-db-connection-string"
description = "ADM database connection string"
regex = '''(mssql|postgresql|mongodb|mysql):\/\/[^:]+:[^@]+@'''
severity = "CRITICAL"
tags = ["database", "credentials"]

[[rules]]
id = "azure-connection-string"
description = "Azure Storage/Service connection string"
regex = '''DefaultEndpointsProtocol=https;AccountName=[^;]+;AccountKey=[^;]+'''
severity = "CRITICAL"
tags = ["azure", "credentials"]

[[rules]]
id = "hardcoded-localhost-with-port"
description = "Hardcoded localhost URL with port in non-config files"
regex = '''(http://localhost:\d{4,5}|http://127\.0\.0\.1:\d{4,5})'''
severity = "MEDIUM"
tags = ["config-drift"]
[rules.allowlist]
files = ['''(\.example|vite\.config|README|test_|\.spec\.)''']

[allowlist]
description = "Global allowlist for ADM repo"
files = [
  '''\.env\.example$''',
  '''README\.md$''',
  '''CHANGELOG\.md$''',
  '''test_.*\.py$''',
  '''.*_test\.py$''',
]
```

---

## 3. Layer 2 — GitLab CI Pipeline (Server-side Gates)

### 3.1 Pipeline design principles

- All security gates run on **every merge request**
- Pipeline must **fail** (not warn) on: hardcoded secrets, `verify=False`, banned `localStorage`, f-string SQL
- Lint/type checks fail the pipeline — no cosmetic-only passes
- Dependency audit warns on medium CVEs, fails on critical/high

### 3.2 Shared pipeline template (`.gitlab-ci-security-template.yml`)

Create this in a shared project or in each repo. This defines reusable job templates:

```yaml
# .gitlab-ci-security-template.yml
# Include this in each repo's .gitlab-ci.yml

.python_security_base:
  image: python:3.11-slim
  before_script:
    - pip install --upgrade pip
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_BRANCH == "develop"'

.node_security_base:
  image: node:20-slim
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_BRANCH == "develop"'
```

---

## 4. Per-repo Setup

---

### 4.1 `apis-azure` (FastAPI backend)

#### File: `apis-azure/.gitlab-ci.yml`

```yaml
stages:
  - lint
  - security
  - test
  - audit

variables:
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.pip-cache"
  PYTHON_VERSION: "3.11"

default:
  image: python:3.11-slim
  cache:
    key: "$CI_COMMIT_REF_SLUG-pip"
    paths:
      - .pip-cache/

# ── STAGE 1: LINT ────────────────────────────────────────────────────────────

ruff-lint:
  stage: lint
  script:
    - pip install ruff
    - ruff check . --output-format=gitlab
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH =~ /^(main|develop)$/'
  allow_failure: false    # Hard fail — lint errors block merge

black-format-check:
  stage: lint
  script:
    - pip install black
    - black --check --diff .
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
  allow_failure: false

mypy-type-check:
  stage: lint
  script:
    - pip install mypy types-requests types-PyYAML
    - pip install -r requirements.txt --quiet
    - mypy . --ignore-missing-imports --no-strict-optional --junit-xml=mypy-report.xml
  artifacts:
    reports:
      junit: mypy-report.xml
    when: always
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
  allow_failure: false

# ── STAGE 2: SECURITY ────────────────────────────────────────────────────────

secret-scan-gitleaks:
  stage: security
  image:
    name: zricethezav/gitleaks:latest
    entrypoint: [""]
  script:
    - gitleaks detect
        --source .
        --config .gitleaks.toml
        --report-format sarif
        --report-path gitleaks-report.sarif
        --exit-code 1
  artifacts:
    when: always
    paths:
      - gitleaks-report.sarif
    reports:
      sast: gitleaks-report.sarif
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH =~ /^(main|develop)$/'
  allow_failure: false    # HARD FAIL — secrets block everything

bandit-sast:
  stage: security
  script:
    - pip install bandit[toml]
    - bandit -r . -f json -o bandit-report.json -ll
        --exclude ./tests,./venv,./.venv
  after_script:
    - pip install bandit[toml]
    - bandit -r . -f txt -ll --exclude ./tests,./venv,./.venv || true
  artifacts:
    when: always
    paths:
      - bandit-report.json
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
  allow_failure: false    # High/critical severity = fail

semgrep-sast:
  stage: security
  image: returntocorp/semgrep:latest
  script:
    - semgrep ci
        --config=p/python
        --config=p/fastapi
        --config=p/owasp-top-ten
        --config=p/secrets
        --sarif
        --output=semgrep-report.sarif
  artifacts:
    when: always
    paths:
      - semgrep-report.sarif
    reports:
      sast: semgrep-report.sarif
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
  allow_failure: false

custom-pattern-check:
  stage: security
  script:
    - pip install grep-ast || true
    - echo "=== Checking for verify=False ==="
    - |
      if grep -rn "verify\s*=\s*False" --include="*.py" .; then
        echo "❌ FAIL: verify=False found. CWE-295 violation."
        exit 1
      fi
      echo "✅ No verify=False found"
    - echo "=== Checking for f-string SQL ==="
    - |
      if grep -Prn 'f["\x27](SELECT|INSERT|UPDATE|DELETE|DROP|ALTER)' --include="*.py" .; then
        echo "❌ FAIL: f-string SQL detected. CWE-89 violation."
        exit 1
      fi
      echo "✅ No f-string SQL found"
    - echo "=== Checking for hardcoded connection strings ==="
    - |
      if grep -Prn '(password|passwd|pwd)\s*=\s*["\x27][^"\x27]{4,}["\x27]' \
          --include="*.py" \
          --exclude-dir=tests \
          --exclude-dir=venv .; then
        echo "❌ FAIL: Possible hardcoded credential."
        exit 1
      fi
      echo "✅ No hardcoded passwords found"
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
  allow_failure: false

# ── STAGE 3: TEST ────────────────────────────────────────────────────────────

pytest:
  stage: test
  script:
    - pip install pytest pytest-cov pytest-asyncio httpx
    - pip install -r requirements.txt --quiet
    - pytest tests/
        --cov=.
        --cov-report=xml:coverage.xml
        --cov-report=term-missing
        --cov-fail-under=60
        --junit-xml=pytest-report.xml
  coverage: '/TOTAL.*\s+(\d+%)$/'
  artifacts:
    when: always
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
      junit: pytest-report.xml
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
  allow_failure: false

# ── STAGE 4: DEPENDENCY AUDIT ────────────────────────────────────────────────

pip-audit:
  stage: audit
  script:
    - pip install pip-audit
    - pip-audit -r requirements.txt
        --format json
        --output pip-audit-report.json
        --vulnerability-service pypi
        --fail-on-severity high
  artifacts:
    when: always
    paths:
      - pip-audit-report.json
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'
  allow_failure: false    # High/Critical CVEs = hard fail
```

#### File: `apis-azure/pyproject.toml` (tool configuration)

```toml
[tool.ruff]
target-version = "py311"
line-length = 100
select = [
  "E",    # pycodestyle errors
  "W",    # pycodestyle warnings
  "F",    # pyflakes
  "I",    # isort
  "B",    # flake8-bugbear
  "S",    # flake8-bandit (security)
  "UP",   # pyupgrade
]
ignore = [
  "S101", # assert used (ok in tests)
  "B008", # function calls in defaults (FastAPI Depends pattern)
]
exclude = [".venv", "venv", "__pycache__", "migrations"]

[tool.ruff.per-file-ignores]
"tests/*" = ["S", "B"]

[tool.black]
line-length = 100
target-version = ["py311"]

[tool.mypy]
python_version = "3.11"
ignore_missing_imports = true
warn_return_any = true
warn_unused_configs = true
exclude = ["venv", ".venv", "tests"]

[tool.bandit]
exclude_dirs = ["tests", "venv", ".venv"]
skips = ["B101"]          # skip assert warnings in test files
severity = "medium"       # flag medium+ severity
confidence = "medium"
```

---

### 4.2 `adm-foundation-layer-agents-azure` (Azure Functions)

This repo has the same Python checks plus a specific focus on committed `.env` files and connector duplication.

#### File: `adm-foundation-layer-agents-azure/.gitlab-ci.yml`

```yaml
stages:
  - lint
  - security
  - test
  - audit

default:
  image: python:3.11-slim
  cache:
    key: "$CI_COMMIT_REF_SLUG-pip-agents"
    paths:
      - .pip-cache/

# ── LINT ─────────────────────────────────────────────────────────────────────

ruff-lint:
  stage: lint
  script:
    - pip install ruff
    - ruff check . --output-format=gitlab
  allow_failure: false

mypy-type-check:
  stage: lint
  script:
    - pip install mypy types-requests azure-functions
    - pip install -r requirements.txt --quiet
    - mypy . --ignore-missing-imports
  allow_failure: false

# ── SECURITY ─────────────────────────────────────────────────────────────────

secret-scan-gitleaks:
  stage: security
  image:
    name: zricethezav/gitleaks:latest
    entrypoint: [""]
  script:
    - gitleaks detect --source . --config .gitleaks.toml --exit-code 1
  allow_failure: false

bandit-sast:
  stage: security
  script:
    - pip install bandit
    - bandit -r . -ll --exclude ./tests,./venv -f json -o bandit-report.json
  artifacts:
    paths:
      - bandit-report.json
    when: always
  allow_failure: false

custom-agent-checks:
  stage: security
  script:
    - echo "=== Check: verify=False ==="
    - |
      if grep -rn "verify=False" --include="*.py" .; then
        echo "❌ verify=False found"; exit 1
      fi
    - echo "=== Check: No real .env committed ==="
    - |
      if git diff --name-only HEAD~1 HEAD 2>/dev/null | grep -E '^\.env$'; then
        echo "❌ Real .env file in commit!"; exit 1
      fi
      if [ -f ".env" ]; then
        echo "⚠️  .env file exists in repo root — should not be committed"
        if git ls-files --error-unmatch .env 2>/dev/null; then
          echo "❌ .env is tracked by git!"; exit 1
        fi
      fi
    - echo "=== Check: Duplicate connector files ==="
    - |
      CONNECTOR_COUNT=$(find . -name "VectorDatabaseConnectors.py" | wc -l)
      if [ "$CONNECTOR_COUNT" -gt 1 ]; then
        echo "❌ $CONNECTOR_COUNT copies of VectorDatabaseConnectors.py found."
        echo "Move to shared package. Duplication is banned."
        find . -name "VectorDatabaseConnectors.py"
        exit 1
      fi
      echo "✅ No connector duplication"
  allow_failure: false

semgrep-sast:
  stage: security
  image: returntocorp/semgrep:latest
  script:
    - semgrep ci --config=p/python --config=p/secrets --sarif --output=semgrep.sarif
  artifacts:
    paths:
      - semgrep.sarif
    when: always
  allow_failure: false

# ── TEST ─────────────────────────────────────────────────────────────────────

pytest:
  stage: test
  script:
    - pip install pytest pytest-cov
    - pip install -r requirements.txt --quiet
    - pytest tests/ --cov=. --cov-report=xml --cov-fail-under=50
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
  allow_failure: false

# ── AUDIT ─────────────────────────────────────────────────────────────────────

pip-audit:
  stage: audit
  script:
    - pip install pip-audit
    - pip-audit -r requirements.txt --fail-on-severity high
  allow_failure: false
```

---

### 4.3 `apps-azure` (React/TypeScript frontend)

#### File: `apps-azure/.gitlab-ci.yml`

```yaml
stages:
  - lint
  - security
  - test
  - audit

default:
  image: node:20-slim
  cache:
    key: "$CI_COMMIT_REF_SLUG-npm"
    paths:
      - node_modules/
      - .npm/

# ── LINT ─────────────────────────────────────────────────────────────────────

eslint:
  stage: lint
  script:
    - npm ci --cache .npm --prefer-offline
    - npx eslint . --ext .ts,.tsx,.js,.jsx
        --format gitlab
        --max-warnings 0
  allow_failure: false

typescript-check:
  stage: lint
  script:
    - npm ci --cache .npm --prefer-offline
    - npx tsc --noEmit
  allow_failure: false

prettier-check:
  stage: lint
  script:
    - npm ci --cache .npm --prefer-offline
    - npx prettier --check "src/**/*.{ts,tsx,js,jsx,css}"
  allow_failure: false

# ── SECURITY ─────────────────────────────────────────────────────────────────

secret-scan-gitleaks:
  stage: security
  image:
    name: zricethezav/gitleaks:latest
    entrypoint: [""]
  script:
    - gitleaks detect --source . --config .gitleaks.toml --exit-code 1
  allow_failure: false

custom-frontend-checks:
  stage: security
  image: node:20-slim
  script:
    - echo "=== Check: No localStorage for sensitive state ==="
    - |
      LOCALSTORAGE_HITS=$(grep -rn "localStorage\.setItem" src/ \
        --include="*.ts" --include="*.tsx" \
        | grep -v "non-sensitive-ui-state" \
        | grep -v "\.test\." \
        | grep -v "\.spec\." || true)
      if [ -n "$LOCALSTORAGE_HITS" ]; then
        echo "❌ Banned localStorage.setItem usage found:"
        echo "$LOCALSTORAGE_HITS"
        echo "localStorage is banned unless annotated // non-sensitive-ui-state"
        exit 1
      fi
      echo "✅ No banned localStorage usage"
    - echo "=== Check: No hardcoded API base URLs ==="
    - |
      if grep -rn "http://localhost:" src/ \
          --include="*.ts" --include="*.tsx" \
          | grep -v "\.test\." \
          | grep -v "\.spec\." \
          | grep -v "\.example"; then
        echo "❌ Hardcoded localhost URL in source. Use VITE_* env vars."
        exit 1
      fi
      echo "✅ No hardcoded localhost URLs"
    - echo "=== Check: No hardcoded production URLs ==="
    - |
      if grep -Prn 'https?://[a-zA-Z0-9\-]+\.(azure(websites|functions)\.net|azurewebsites\.net)' \
          src/ --include="*.ts" --include="*.tsx" \
          | grep -v "\.example\." | grep -v "\.test\."; then
        echo "❌ Hardcoded Azure production URL found in source."
        echo "Use VITE_API_BASE_URL environment variable instead."
        exit 1
      fi
      echo "✅ No hardcoded Azure URLs"
  allow_failure: false

semgrep-frontend:
  stage: security
  image: returntocorp/semgrep:latest
  script:
    - semgrep ci
        --config=p/typescript
        --config=p/react
        --config=p/secrets
        --config=p/owasp-top-ten
        --sarif
        --output=semgrep-frontend.sarif
  artifacts:
    paths:
      - semgrep-frontend.sarif
    when: always
  allow_failure: false

# ── TEST ─────────────────────────────────────────────────────────────────────

vitest:
  stage: test
  script:
    - npm ci --cache .npm --prefer-offline
    - npx vitest run --coverage
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
  allow_failure: false

# ── AUDIT ─────────────────────────────────────────────────────────────────────

npm-audit:
  stage: audit
  script:
    - npm ci --cache .npm --prefer-offline
    - npm audit --audit-level=high
  allow_failure: false
```

#### ESLint config: `apps-azure/.eslintrc.json`

```json
{
  "extends": [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:react-hooks/recommended"
  ],
  "parser": "@typescript-eslint/parser",
  "plugins": ["@typescript-eslint", "react-hooks", "no-secrets"],
  "rules": {
    "@typescript-eslint/no-explicit-any": "error",
    "@typescript-eslint/no-unused-vars": "error",
    "no-console": "warn",
    "no-secrets/no-secrets": ["error", { "tolerance": 4.2 }],
    "no-restricted-globals": [
      "error",
      {
        "name": "localStorage",
        "message": "localStorage is banned for sensitive data. Use backend persistence or in-memory state."
      }
    ]
  }
}
```

**Install ESLint plugins:**
```bash
npm install --save-dev \
  eslint \
  @typescript-eslint/eslint-plugin \
  @typescript-eslint/parser \
  eslint-plugin-react-hooks \
  eslint-plugin-no-secrets
```

---

## 5. Secret Scanning Deep Dive

### 5.1 How gitleaks works

Gitleaks scans the **git history**, not just the current state. So even if someone removes a secret in a later commit, gitleaks will still catch it in the original commit. This is important — secrets must be rotated even after removal.

### 5.2 What it catches by default

- AWS keys, GCP keys, Azure credentials
- GitHub/GitLab tokens
- Private keys (RSA, EC, PEM)
- Generic high-entropy strings that look like secrets
- Database connection strings (via your custom `.gitleaks.toml`)
- Hardcoded passwords matching common patterns

### 5.3 Handling false positives

If a legitimate string triggers gitleaks (e.g., in a test fixture):

```python
# gitleaks:allow
DUMMY_KEY_FOR_TESTING = "not-a-real-key-abc123"  # gitleaks:allow
```

Or in `.gitleaks.toml`:
```toml
[allowlist]
regexes = [
  '''DUMMY_KEY_FOR_TESTING''',
  '''example\.com''',
]
```

---

## 6. SAST Deep Dive

### 6.1 Bandit — Python-specific security checks

Bandit checks for Python anti-patterns mapped to CWE:

| Bandit Rule | What it catches | Maps to |
|---|---|---|
| B105, B106, B107 | Hardcoded password strings | CWE-798 |
| B201, B608 | SQL injection patterns | CWE-89 |
| B501, B502 | TLS/SSL issues (`verify=False`) | CWE-295 |
| B301–B303 | Pickle deserialization | CWE-502 |
| B401–B412 | Dangerous imports | Various |

### 6.2 Semgrep rules used

- `p/python` — general Python security
- `p/fastapi` — FastAPI-specific patterns
- `p/owasp-top-ten` — OWASP Top 10 coverage
- `p/secrets` — secret patterns
- `p/typescript` + `p/react` — frontend

### 6.3 GitLab built-in SAST (if Ultimate license)

If you have GitLab Ultimate, you get built-in SAST for free via:

```yaml
include:
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/Secret-Detection.gitlab-ci.yml
  - template: Security/Dependency-Scanning.gitlab-ci.yml
```

This gives you a Security Dashboard in GitLab UI. Check your GitLab instance at:  
`Settings > General > Visibility, project features, permissions > Security & compliance`

If you don't have Ultimate, everything in this guide (bandit, semgrep, gitleaks, pip-audit) covers the same ground with open-source tools.

---

## 7. Merge Request Gate Policy

### 7.1 GitLab branch protection settings

Go to each repo: `Settings > Repository > Protected Branches`

| Setting | Value |
|---|---|
| Branch | `main` |
| Allowed to merge | Maintainers |
| Allowed to push | No one (force pipeline) |
| Require approval | 1 approval minimum |
| Code owner approval | Yes (if CODEOWNERS defined) |

### 7.2 GitLab merge request settings

Go to: `Settings > Merge requests`

- ✅ Enable "Pipelines must succeed"
- ✅ Enable "All discussions must be resolved"
- ✅ Enable "Delete source branch by default"
- Set merge method: **Merge commit with semi-linear history** (prevents force pushes bypassing CI)

### 7.3 CODEOWNERS file

Create `.gitlab/CODEOWNERS` in each repo:

```
# Require senior engineer review for security-sensitive files
/apis-azure/backend/api/routes/     @senior-backend-engineer @your-gitlab-username
/apis-azure/backend/core/config.py  @your-gitlab-username
/.env.example                        @your-gitlab-username
/.gitlab-ci.yml                      @your-gitlab-username
/.pre-commit-config.yaml             @your-gitlab-username
```

---

## 8. GitLab License Note

### If you're on Free/Premium:
All tools in this guide (bandit, semgrep, gitleaks, pip-audit, ruff, mypy, eslint) are **fully open source and free**. You're not missing anything by not having Ultimate — you're just doing it yourself rather than using GitLab's wrappers.

### If you have Ultimate:
Add these to your `.gitlab-ci.yml` to also get the GitLab Security Dashboard UI:

```yaml
include:
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/Secret-Detection.gitlab-ci.yml
  - template: Security/Dependency-Scanning.gitlab-ci.yml
  - template: Security/Container-Scanning.gitlab-ci.yml  # if using Docker
```

To check your license: `Admin Area > License` or ask your GitLab admin.

---

## 9. Rollout Plan for Your Team

### Week 1 — Foundation (you do this)
- [ ] Add `.pre-commit-config.yaml` and `.gitleaks.toml` to all 3 repos
- [ ] Add `pyproject.toml` with ruff/black/mypy/bandit config to Python repos
- [ ] Add `.eslintrc.json` to apps-azure
- [ ] Set up protected branches in GitLab for `main`
- [ ] Enable "Pipelines must succeed" in MR settings

### Week 2 — Pipeline (you + DevOps)
- [ ] Add `.gitlab-ci.yml` to all 3 repos (start with `allow_failure: true` on tests if coverage is 0)
- [ ] Verify pipeline runs on a test MR
- [ ] Fix any pre-existing violations that cause CI noise (gitleaks history scan may surface old secrets — rotate those)
- [ ] Move `allow_failure: false` on lint and security gates

### Week 3 — Team Onboarding
- [ ] Send team a one-pager: "run `pre-commit install` after cloning, here's why"
- [ ] Code review session: walk through what each gate catches and the CWE it maps to
- [ ] Add pre-commit install step to the repo README onboarding section

### Week 4 — Tighten
- [ ] Set coverage thresholds (start at 40%, raise to 60% over a month)
- [ ] Enable CODEOWNERS for sensitive paths
- [ ] Schedule weekly `pip-audit` run on `main` even without MRs (cron job in CI)

### Handling the "existing code is broken" problem
When you first run these checks on legacy code, there will be violations. Don't let that block the team. Strategy:

```yaml
# Temporarily allow failures on legacy violations while you fix them
bandit-sast:
  allow_failure: true   # ← set to false once existing code is clean
```

Create a GitLab issue for each category of violation, assign them as P0/P1 per your remediation plan, and flip `allow_failure: false` once each category is resolved.

---

## 10. Tool Reference Summary

| Tool | What it checks | Lang | How used |
|---|---|---|---|
| **gitleaks** | Secrets, credentials, keys | All | pre-commit + CI |
| **bandit** | Python SAST, CWE mappings | Python | CI |
| **semgrep** | SAST, OWASP, custom rules | Python + TS | CI |
| **ruff** | Lint, style, security rules | Python | pre-commit + CI |
| **black** | Code formatting | Python | pre-commit + CI |
| **mypy** | Static type checking | Python | pre-commit + CI |
| **pip-audit** | CVE scan on dependencies | Python | CI |
| **eslint** | Lint + security rules | TypeScript/React | pre-commit + CI |
| **prettier** | Code formatting | TS/JS/CSS | pre-commit + CI |
| **tsc** | TypeScript type check | TypeScript | CI |
| **npm audit** | CVE scan on dependencies | Node | CI |
| **custom grep checks** | verify=False, f-string SQL, localStorage, hardcoded URLs | Mixed | pre-commit + CI |

---

## Quick Reference Card (send to your team)

```
PRE-COMMIT SETUP (run once after cloning):
  pip install pre-commit
  pre-commit install

WHAT WILL BLOCK YOUR COMMIT:
  ❌ Any secret / password / token in code
  ❌ verify=False in any Python file
  ❌ f-string SQL (f"SELECT {var}...")
  ❌ localStorage.setItem for non-UI state
  ❌ Hardcoded API URLs (use VITE_* vars)
  ❌ Committing real .env files

WHAT WILL BLOCK YOUR MERGE REQUEST:
  Everything above, PLUS:
  ❌ Lint errors (ruff / eslint)
  ❌ Type errors (mypy / tsc)
  ❌ High/Critical CVEs in dependencies
  ❌ Failing tests / coverage below threshold
  ❌ Duplicate connector files

TO FIX BEFORE COMMITTING:
  black .            ← auto-format Python
  ruff check --fix . ← auto-fix lint
  npx prettier --write src/  ← auto-format TS
```
