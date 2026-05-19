## Tableau Workbook and Datasource Backup to S3

`tableau-content-backup-to-s3` automates incremental backups of Tableau Server workbooks and datasources into versioned folders on S3. It signs in to Tableau, downloads content (with extracts when possible), pushes it to AWS, and streams telemetry to Sentry and Zabbix so operations teams can monitor drift.

### Features
- **Incremental sync with state tracking:** `upload_state.json` is stored per site in S3, letting the script skip unchanged workbooks and detect removals without scanning every object's tags; reading a single JSON blob is cheaper than issuing `GetObjectTagging` across thousands of keys.
- **Automatic S3 retagging:** workbook metadata (owner, timestamps, descriptions) is converted into S3-compliant tags, with retry logic guarding uploads.
- **Observability baked in:** Zabbix heartbeats/exit codes and rich Sentry breadcrumbs (including failed workbook details) make postmortems quick.
- **Selective restores:** limit runs to explicit sites/projects or target a single `TS_SITE_NAME` via env, perfect for triaging.
- **Self-healing objects:** the `_s3_update_outdated_last_modified` task refreshes LastModified timestamps for stale objects so lifecycle policies keep working.

## Requirements
- Python 3.14+ (as declared in `pyproject.toml`)
- AWS IAM user/key with `s3:ListBucket`, `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject`
- Tableau Server account with download rights on desired sites/projects
- Optional: Vault access for secrets (`hvac`), outbound network to Sentry/Zabbix

Install dependencies inside a virtual environment:

```bash
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

## Configuration
Copy `config.toml` and update:
- `[main]`: `workdir` temporary directory and `max_workers` for the thread pool.
- `[backup.sites]`: `excluded_sites` and default S3 bucket for full backups.
- `[[backup.projects]]`: site-specific project filters with their own buckets.
- `[vault]`: Vault url/role/paths if secrets should be resolved dynamically.

At runtime you may override the config file (`-c custom.toml`) and specify `TS_SITE_NAME` to limit processing to a single site.

### Example `config.toml`

```toml
[main]
# Temporary directory for downloaded workbooks before upload
workdir = "/tmp/tableau-backup"
# Number of parallel download/upload workers
max_workers = 6

# Full backup of all sites into one bucket.
# Use this OR [[backup.projects]], or both.
[backup.sites]
s3_bucket_name = "my-tableau-backup"
# Sites to skip (optional)
excluded_sites = ["Personal", "Sandbox"]

# Per-site project-scoped backups into separate buckets.
# Repeat this block for each site that needs project filtering.
[[backup.projects]]
site = "Sales"
projects = ["Finance/Revenue", "Marketing"]
bucket = "my-tableau-sales-backup"

[[backup.projects]]
site = "Engineering"
projects = ["Data Platform"]
bucket = "my-tableau-eng-backup"

# HashiCorp Vault – used to fetch Tableau credentials and script params.
# Remove this section and set credentials directly in the environment
# (AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY, etc.) if Vault is not used.
[vault]
url = "https://vault.example.com"
role_id = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
secret_id = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

[vault.paths]
# Secret must contain: username, password, url
ts_creds = "secret/tableau-server/prod"
# Secret must contain: sentry_dsn
params   = "secret/tableau-content-backup/params"
```

**Vault secrets format expected by the script:**

| Path key   | Required fields                    |
|------------|------------------------------------|
| `ts_creds` | `username`, `password`, `url`      |
| `params`   | `sentry_dsn`                       |

## Running a Backup

```bash
python cli.py -c config.toml --debug --zs
```

- `--debug` increases log verbosity and writes to `wb-backup2s3.log`.
- `--zs` stubs the Zabbix sender for local tests.

When the script finishes it reports successes/failures, updates Zabbix metrics, uploads new workbooks, and refreshes the S3 state file.

## Development & Testing
- Core backup logic lives in `core/core.py`; keep entry point logic in `cli.py` thin.
- Current test file in this repo is `test_cli.py`; for broader coverage, add new tests under `tests/`.
- Use `pytest -q` with mocked Tableau/S3 clients to cover new flows.
- Before opening a PR, run a dry job against a staging Tableau site with `--debug --zs` and attach the anonymized log snippet showing Zabbix/S3 updates.
