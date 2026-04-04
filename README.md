# controld-hagezi-sync

Automatically syncs DNS blocklist folders from [hagezi/dns-blocklists](https://github.com/hagezi/dns-blocklists) into your [Control D](https://controld.com) account via the Control D API.

Runs on a GitHub Actions schedule — no server required.

---

## How it works

The workflow runs twice daily (05:00 and 17:00 UTC) in two stages:

**Stage 1 — File sync** (`scripts/controld_sync.py`)
Downloads the target JSON files from the hagezi upstream repository, diffs them against the local copies, and commits any changes back to this repo. Sets a `changed` flag for the next stage.

**Stage 2 — API push** (`scripts/controld_api_push.py`)
Runs only when Stage 1 detected changes. Reads the updated JSON files and reconciles each mapped Control D folder against the desired state — adding new domains and removing stale ones. The reconciliation is always against the **live API state**, so the script is idempotent and self-healing if a previous run was interrupted.

An email report is sent after Stage 2 summarising every domain added, removed, or skipped per profile and folder.

---

## Quick start

### 1. Fork this repo

### 2. Set the required GitHub secret

Go to **Settings → Secrets and variables → Actions → New repository secret**:

| Secret | Required | Description |
|--------|:--------:|-------------|
| `CTRLD_API_TOKEN` | ✅ | Your Control D API token. Found in the Control D dashboard under **API**. |

### 3. Configure your profile and folder mappings

Edit `scripts/controld_api_push.py` and update `FILE_MAPPINGS` to match your Control D profile and folder names. See [CONFIGURATION.md](CONFIGURATION.md) for full details.

Also update `TARGET_FILES` in `scripts/controld_sync.py` if you want to track a different subset of files.

### 4. Run manually to verify

Go to **Actions → Sync Control D folders from upstream → Run workflow** to trigger an immediate run and verify everything is working before waiting for the schedule.

---

## Optional: email notifications

When changes are detected, the workflow can send an email report. Configure these additional secrets (all optional — omit any to skip email):

| Secret | Description |
|--------|-------------|
| `EMAIL_SERVER` | SMTP server hostname (e.g. `smtp.gmail.com`) |
| `EMAIL_PORT` | `587` for STARTTLS, `465` for implicit TLS |
| `EMAIL_USERNAME` | SMTP login username |
| `EMAIL_PASSWORD` | SMTP password or app password |
| `EMAIL_FROM` | Sender address |
| `EMAIL_TO` | Recipient address |

Email is only sent when Stage 2 runs (i.e. files actually changed).

---

## Repository structure

```
.github/
  workflows/
    sync-controld.yml       # workflow orchestrator
scripts/
  controld_sync.py          # Stage 1: file sync
  controld_api_push.py      # Stage 2: Control D API push
requirements.txt            # pinned Python dependencies
controld/                   # synced JSON files (created on first run)
```

---

## Upstream source

All blocklist JSON files come from [hagezi/dns-blocklists](https://github.com/hagezi/dns-blocklists/tree/main/controld). Hat tip to hagezi for maintaining these lists.
