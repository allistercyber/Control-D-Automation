# Configuration guide

This document covers every value you need to change to make this workflow work for your own Control D account.

---

## Files to edit

| File | What to change |
|------|----------------|
| `scripts/controld_api_push.py` | `FILE_MAPPINGS` — which upstream files map to which of your profiles/folders |
| `scripts/controld_sync.py` | `TARGET_FILES` — which upstream files to download (should match `FILE_MAPPINGS` keys) |

---

## `FILE_MAPPINGS` in `controld_api_push.py`

This is the primary configuration. It tells Stage 2 which Control D folder to push each JSON file's domains into.

### Format

```python
FILE_MAPPINGS: Dict[str, List[Tuple[str, str]]] = {
    "upstream-filename.json": [
        ("Your Profile Name", "Your Folder Name"),
    ],
}
```

Each entry maps one upstream filename to a list of `(profile_name, folder_name)` pairs. One file can sync to multiple profiles simultaneously — just add more pairs to the list.

### How to find your profile and folder names

1. Log into the [Control D dashboard](https://controld.com/dashboard)
2. Go to **Profiles** in the left sidebar
3. The profile name is shown at the top of each profile — **copy it exactly, it is case-sensitive**
4. Inside a profile, open the folder list on the left — **copy the folder name exactly**

### Example

```python
FILE_MAPPINGS: Dict[str, List[Tuple[str, str]]] = {
    # Sync Apple Private Relay list to one profile
    "apple-private-relay-allow-folder.json": [
        ("Home", "Apple Private Relay Block"),
    ],

    # Sync spam TLDs to two profiles at once
    "spam-tlds-folder.json": [
        ("Home",   "Blocked TLDs"),
        ("Travel", "Blocked TLDs"),
    ],
}
```

### Available upstream files

These are the files currently published by hagezi in the `controld/` directory of their repo:

| Filename | What it covers |
|----------|----------------|
| `apple-private-relay-allow-folder.json` | Apple Private Relay exit nodes (allow-list) |
| `meta-tracker-allow-folder.json` | Meta tracker domains (allow-list) |
| `microsoft-allow-folder.json` | Microsoft service domains (allow-list) |
| `native-tracker-apple-folder.json` | Apple native tracker domains |
| `native-tracker-microsoft-folder.json` | Microsoft native tracker domains |
| `spam-tlds-folder.json` | Spam and high-abuse top-level domains |

Remove any file you don't want to track. If hagezi publishes new files in the future, add them here and to `TARGET_FILES`.

### Processing order and cross-folder deduplication

Within a single profile, domains claimed by an **earlier** entry in `FILE_MAPPINGS` are excluded from **later** entries. This prevents the same domain appearing in both an allow folder and a block folder at the same time.

**Practical rule:** put allow-folders before block-folders in `FILE_MAPPINGS`.

---

## `TARGET_FILES` in `controld_sync.py`

This controls which files Stage 1 downloads from the upstream repo. It should match the keys you have in `FILE_MAPPINGS` — no more, no less.

```python
TARGET_FILES: List[str] = [
    "apple-private-relay-allow-folder.json",
    "spam-tlds-folder.json",
    # add or remove filenames here
]
```

If a filename is in `FILE_MAPPINGS` but not in `TARGET_FILES`, Stage 2 will fail to find the file and skip it with an error.

---

## Schedule

The workflow runs at 05:00 and 17:00 UTC by default. To change this, edit `.github/workflows/sync-controld.yml`:

```yaml
on:
  schedule:
    - cron: '0 5  * * *'   # 05:00 UTC
    - cron: '0 17 * * *'   # 17:00 UTC
```

Standard cron syntax applies. [crontab.guru](https://crontab.guru) is useful for building expressions.

---

## Required GitHub secrets

Set these under **Settings → Secrets and variables → Actions**:

| Secret | Required | Notes |
|--------|:--------:|-------|
| `CTRLD_API_TOKEN` | ✅ | Control D dashboard → API |
| `EMAIL_SERVER` | Optional | e.g. `smtp.gmail.com` |
| `EMAIL_PORT` | Optional | `587` (STARTTLS) or `465` (TLS) |
| `EMAIL_USERNAME` | Optional | SMTP login |
| `EMAIL_PASSWORD` | Optional | Use an app password if your provider requires it |
| `EMAIL_FROM` | Optional | Sender address shown in the report email |
| `EMAIL_TO` | Optional | Where to deliver the report |

All email secrets are optional. If `EMAIL_SERVER`, `EMAIL_FROM`, or `EMAIL_TO` is missing, the email step is skipped silently.

---

## First run behaviour

On the very first run, the `controld/` directory does not exist yet. Stage 1 will create it and commit all downloaded files as new. Stage 2 will then push all domains in those files to the configured folders — treat this as an initial population, not an incremental diff.

If you already have domains in your Control D folders that are not in the upstream files, they will be removed during the first run (Stage 2 always reconciles to the exact desired state). Make sure your folder contents align with what you expect before the first run, or review the Stage 2 log output carefully.
