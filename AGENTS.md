# Repository Guidelines

## Project Structure & Module Organization
Top-level entry points (`vcli.py`) orchestrate backups by instantiating `wb_backup2s3.BackupWB2S3`. Core logic lives in `wb_backup2s3/core.py`, which handles Tableau API calls, retry logic, and S3 uploads. Configuration templates (`config.toml`) sit beside the CLI for quick swapping between environments; avoid editing them in place for secrets—copy to a local override. Logs (`wb-backup2s3.log*`) remain in the repo root during development but rotate in production. Keep any future tests under `tests/` to separate fixtures from executable code.

## Build, Test, and Development Commands
- `python -m venv venv && source venv/bin/activate`: create and enter a dedicated virtualenv.
- `pip install -r requirements.txt`: install Tableau Server Client, boto3, Sentry, Zabbix helpers, and Vault client pinned for compatibility.
- `python vcli.py -c config.toml --debug --zs`: run a dry backup using the sample config, verbose logging, and stubbed Zabbix sender (`--zs`); remove `--debug` for production parity.
- `PYTHONPATH=. python -m wb_backup2s3.core`: quick module-level experiments; prefer this mode for library-style scripts.

## Coding Style & Naming Conventions
Use Python 3.10+ with 4-space indents, type hints (see `BackupWB2S3.__init__`), and descriptive logger names (`main.<module>`). Keep pure functions near their helpers (e.g., decorators at the top of `core.py`) and place dataclasses above consumers. Follow snake_case for functions/variables, PascalCase for dataclasses, and uppercase for constants such as `ZAB_KEY_*`. Run `ruff` or `flake8` before committing if available; otherwise ensure imports are grouped stdlib/third-party/local.

## Testing Guidelines
There is no automated suite yet; start by adding pytest-based cases inside `tests/` that mock Tableau and S3 clients. Name test files `test_<module>.py`, and use fixtures to provide fake workbooks plus temp directories for `_download_workbook` flows. Always run `pytest -q` before opening a PR. For functional checks, execute `python cli.py --zs --debug` against a staging config to verify upload state persistence.

## Commit & Pull Request Guidelines
Recent history favors short, imperative summaries (`module: intent`). Reference the touched module up front (e.g., `core: log upload retries`). Each PR should include: purpose, config or infra changes, manual test notes (command + result), and links to Jira tickets. Attach anonymized log snippets when touching retry/Zabbix/S3 code paths. If UI or config output changes, include screenshots or sample TOML diffs.

## Security & Configuration Tips
Never commit live credentials; `config.toml` shows structure only. Store real secrets in Vault (see `[vault]` block) and inject via environment or CI. When sharing logs, scrub workbooks and site names unless already public. Validate that `SENTRY_DENYLIST` covers new sensitive fields whenever you add metadata.
