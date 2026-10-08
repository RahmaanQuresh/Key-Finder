# keyfinder

**Find leaked secrets in your build output and source tree before they ship.**

keyfinder scans local files for API keys, tokens and private keys, prints
*redacted* findings (so the report never becomes a second leak), and exits
non-zero so CI can block the build.

[![CI](https://github.com/YOUR-USERNAME/keyfinder/actions/workflows/ci.yml/badge.svg)](https://github.com/YOUR-USERNAME/keyfinder/actions)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
![Python 3.9+](https://img.shields.io/badge/python-3.9%2B-blue.svg)

- Zero dependencies, pure Python
- Local only: no network access, nothing leaves your machine
- Redacted output by default
- Works in CI, as a pre-commit hook, or as a library

## Install

```bash
pip install keyfinder
```

From source:

```bash
git clone https://github.com/YOUR-USERNAME/keyfinder
cd keyfinder
pip install -e .
```

## Quick start

```bash
keyfinder ./build            # scan a directory
keyfinder src/app.py .env    # scan specific files
keyfinder                    # scan the current directory
```

Example output:

```
build/config.js:12  AWS access key ID  AKIA...******

1 potential secret(s) found.
Rotate/revoke each one first, then remove it from the code and git history.
```

## Options

| Flag | Description |
|---|---|
| `paths...` | Files or directories to scan (default: `.`) |
| `--json` | Machine-readable JSON output |
| `--exit-zero` | Always exit 0 (report only) |
| `--version` | Print the version |

**Exit codes:** `0` nothing found (or `--exit-zero`), `1` potential secrets found.

## What it detects

| Rule | Examples |
|---|---|
| AWS access key ID | `AKIA...`, `ASIA...` |
| GitHub token | `ghp_`, `gho_`, `ghu_`, `ghs_`, `ghr_` |
| Slack token | `xoxb-`, `xoxp-`, ... |
| Google API key | `AIza...` |
| Stripe live key | `sk_live_...`, `rk_live_...` |
| Private key block | `-----BEGIN ... PRIVATE KEY-----` |
| Generic assignment | `password = "..."`, `api_key: "..."`, `secret`, `token` |

Generic assignments are entropy-checked, so obvious placeholders like
`password = "aaaaaaaaaaaaaaaa"` are skipped. Binary files, files over 2 MB, and
directories such as `.git` and `node_modules` are skipped.

## Handling false positives

**Inline:** add `keyfinder:ignore` in a comment on the same line.

```python
EXAMPLE_KEY = "..."  # keyfinder:ignore
```

**File-level:** create a `.keyfinderignore` in the directory you scan, one glob
per line. Patterns match the relative path or the file name. Lines starting
with `#` are comments.

```
# test fixtures and docs
tests/fixtures/*
docs/*.md
```

## Use it in CI

GitHub Actions:

```yaml
- uses: actions/checkout@v4
- uses: actions/setup-python@v5
  with:
    python-version: "3.12"
- run: pip install keyfinder
- run: keyfinder ./build
```

Any other CI works the same way: install, run `keyfinder <dir>`, and let the
non-zero exit code fail the job.

## Use it as a pre-commit hook

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/YOUR-USERNAME/keyfinder
    rev: v0.1.0
    hooks:
      - id: keyfinder
```

## JSON output

```bash
keyfinder ./build --json
```

```json
[
  {
    "path": "build/config.js",
    "line": 12,
    "rule": "AWS access key ID",
    "redacted": "AKIA...******"
  }
]
```

## Use it as a library

```python
from keyfinder import scan_path

for f in scan_path("./build"):
    print(f.path, f.line, f.rule, f.redacted)
```

## If keyfinder finds a real secret

1. **Rotate or revoke it first.** Treat anything that was ever committed,
   pushed or published as compromised, even after you delete it.
2. Remove it from the code, then from git history if it was committed.
3. Move the secret to environment variables or a secrets manager.

## Limitations

keyfinder is intentionally small and regex-based.

- It does not scan git history.
- It will miss secret formats it has no rule for, and it can produce false
  positives.
- For broader coverage, run it alongside tools like
  [gitleaks](https://github.com/gitleaks/gitleaks) or
  [trufflehog](https://github.com/trufflesecurity/trufflehog).

## Contributing

Issues and pull requests are welcome, especially new detection rules.

```bash
pip install -e ".[dev]"
pytest
```

When adding a rule, add it to `PATTERNS` in `src/keyfinder/scanner.py` and add a
test. Build fake test keys by string concatenation so the test file doesn't
trigger scanners itself. Never commit real credentials.

## Reporting a security issue

If you find a bug in keyfinder that could cause it to leak or miss secrets in a
harmful way, please open a private security advisory on GitHub instead of a
public issue.

## License

[MIT](LICENSE)
