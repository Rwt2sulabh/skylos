# fix(policy): prevent false positive SKY-R101 and SKY-R102 on non-Python repositories

## What does this PR do?

- Passes discovered source files (`source_files=files`) and folder exclusions (`exclude_folders=exclude_folders`) from `Analyzer.analyze()` to `analyze_repo_policy()`.
- Updates `_has_python_sources()` in `skylos/rules/quality/policy.py` to inspect the discovered `source_files` directly. If the analyzed codebase contains no Python files (e.g. 100% TypeScript/Node.js, Go, Rust, Java), `_has_python_sources()` immediately returns `False`, suppressing `SKY-R101` (*python-type-check-policy*) and `SKY-R102` (*python-lint-policy*).
- Expands `_iter_repo_files()` and `SKIP_DIR_NAMES` to incorporate `DEFAULT_EXCLUDE_FOLDERS`, user-defined `[tool.skylos.exclude]`, and common virtual environment and package manager cache folders (`env`, `.env`, `.yarn`, `.pnpm`, `.pnpm-store`, `.cache`, `.tox`, `vendor`).
- Adds regression unit tests in `test/test_good_practices_rules.py` covering TypeScript projects with `pyproject.toml`, source file filtering, and excluded folders.

## Why?

- When users run `skylos init` in a non-Python repository (such as a 100% TypeScript/Node.js project), Skylos initializes `pyproject.toml` solely to store `[tool.skylos]` configuration.
- Previously, `analyze_repo_policy` was decoupled from the analyzer's source discovery and exclusion lists. It performed an independent `os.walk()` using a hardcoded 10-folder skip list that ignored:
  - `DEFAULT_EXCLUDE_FOLDERS` (`.vscode`, `.idea`, `.next`, `.turbo`, `vendor`, `.tox`, etc.)
  - Standard virtual environment paths like `env/` (e.g. `python -m venv env`) and `.env/`
  - Package manager caches like `.yarn/`, `.pnpm/`
  - User-configured `[tool.skylos.exclude]` and CLI `--exclude`
- If any `.py` file existed in those folders or scripts, `_has_python_sources()` returned `True`, incorrectly requiring `mypy`/`pyright` (`SKY-R101`) and `Ruff` (`SKY-R102`) on pure TypeScript repositories.

## How to test

1. Run the policy regression test suite:
   ```bash
   pytest -q test/test_good_practices_rules.py
   ```
2. Verify against a minimal TypeScript reproduction:
   - Create a directory with `package.json`, `index.ts`, and `pyproject.toml` (containing `[tool.skylos]`).
   - Run:
     ```bash
     skylos . -a --format json
     ```
   - Confirm `SKY-R101` and `SKY-R102` are absent from `quality` findings.

## Precision Impact

- Eliminates false positives for `SKY-R101` (*python-type-check-policy*) and `SKY-R102` (*python-lint-policy*) on non-Python repositories that use `pyproject.toml` exclusively for Skylos tool configuration.
- Prevents spurious policy violations triggered by excluded directories (e.g., `env/`, `.yarn/`, `scripts/`).
- Preserves expected policy enforcement for genuine Python repositories.

## Checklist

- [x] Tests pass (`python3 -m pytest test/test_good_practices_rules.py`)
- [x] No new false positives introduced (if modifying analysis logic)
- [ ] Added or updated a corpus case for any confirmed precision regression or false positive fix
- [ ] Ran the corpus guard (`python3 scripts/corpus_ci.py --manifest corpus/manifest.json`) if analysis logic changed
- [ ] CHANGELOG updated (if user-facing change)
