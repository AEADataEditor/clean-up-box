# Box Folder Cleanup Script

Automated cleanup of Box folders for completed Jira cases in the AEA Data Editor workflow.

## Overview

This script automates the process of cleaning up Box folders for cases that have completed their workflow:

1. **Scans** the Box root folder for case folders (e.g., `aearep-1234`)
2. **Queries** `jira_purge_query.py` to check if each case is ready for purging
3. For ready cases:
   - **Deletes** data files (CSV, DTA, ZIP, etc.)
   - **Keeps** documents (PDF, DOCX, TXT, etc.)
   - **Moves** the folder to the `1Completed` subfolder

## Prerequisites

### Python Dependencies

```bash
pip install 'boxsdk[jwt]'
```

and [`jira_purge_query.py`](https://github.com/AEADataEditor/editor-scripts) must be in your path.

### Environment Variables

#### Box Authentication (required)

```bash
export BOX_FOLDER_PRIVATE="your_root_folder_id"
export BOX_PRIVATE_KEY_ID="your_key_id"
export BOX_ENTERPRISE_ID="your_enterprise_id"
export BOX_CONFIG_PATH="/path/to/config/directory"
```

**Alternative:** Use base64-encoded config
```bash
export BOX_PRIVATE_JSON="base64_encoded_config_json"
```

#### Jira Authentication (required for jira_purge_query.py)

```bash
export JIRA_USERNAME="your_email@example.com"
export JIRA_API_KEY="your_jira_api_token"
```

Get your Jira API token at: https://id.atlassian.com/manage-profile/security/api-tokens

## Usage

### Clone this repository

```bash
git clone https://github.com/AEADataEditor/clean-up-box.git
cd clean-up-box
```


### Test Mode (Recommended First!)

**Always test first** to see what would be done without making any changes:

```bash
python3 clean_box_folders.py --test
```

This will:
- Query Jira to check which cases are ready
- Show which files would be deleted
- Show which folders would be moved
- Make NO actual changes to Box

### Process All Ready Cases

```bash
python3 clean_box_folders.py
```

You'll be prompted to confirm if multiple folders will be processed.

### Process Specific Case

```bash
python3 clean_box_folders.py --case 1234
```

This processes only `aearep-1234`.

### Skip Confirmation Prompt

```bash
python3 clean_box_folders.py --yes
```

### List Cases and Status

To see which cases are in Box and check their Jira status without making any changes:

```bash
python3 clean_box_folders.py --list
```

This will:
- Scan Box for all case folders
- Query Jira for each case's purge status
- Display the full output from `jira_purge_query.py` for each case
- Show a summary of how many cases are ready

Example output:
```
✗ [FAIL] AEAREP-6645: Neither this issue nor linked revisions passed through required statuses (Current MCstatus: Done; Current MCRecommendationV2: N/A)
✓ [OK] AEAREP-2124: Ready for purge
  - Current status: Done; Current MCRecommendationV2: Accept
```

You can also check a specific case:
```bash
python3 clean_box_folders.py --list --case 6645
```

### Advanced Options

```bash
# Skip Jira checks (process all folders found) - TESTING ONLY
python3 clean_box_folders.py --skip-jira-check --test
```

## Command-Line Help

Full command-line options available:

```
usage: clean_box_folders.py [-h] [--test] [--list] [--case NUMBER] [--yes]
                            [--skip-jira-check]

Clean up Box folders for completed Jira cases

options:
  -h, --help         show this help message and exit
  --test             Test mode: show what would be done without making changes
  --list             List all cases and their Jira status without making any
                     changes
  --case NUMBER      Process only this specific case number (e.g., 1234 for
                     aearep-1234)
  --yes, -y          Skip confirmation prompt
  --skip-jira-check  Skip Jira status checks (process all folders found) - for
                     testing only

Examples:
  # Test mode (dry run - no modifications)
  clean_box_folders.py --test

  # Process all ready cases
  clean_box_folders.py

  # Process specific case
  clean_box_folders.py --case 1234

  # Skip confirmation prompt
  clean_box_folders.py --yes

Environment Variables Required:
  Box Authentication:
    BOX_FOLDER_PRIVATE - Root Box folder ID
    BOX_PRIVATE_KEY_ID - JWT public key ID
    BOX_ENTERPRISE_ID - Enterprise ID
    BOX_CONFIG_PATH - Directory containing config JSON file
    (or BOX_PRIVATE_JSON - Base64 encoded config)
    
  Jira Authentication:
    JIRA_USERNAME - Your Jira email address
    JIRA_API_KEY - API token
```

## File Classification

### Data Files (Will be DELETED)

- Statistical data: `.csv`, `.dta`, `.sas7bdat`, `.rds`, `.rdata`, `.mat`, `.sav`
- Compressed: `.zip`, `.gz`, `.tar`, `.7z`, `.rar`
- Databases: `.db`, `.sqlite`, `.sql`
- Modern formats: `.parquet`, `.feather`, `.hdf5`
- Other: `.json`, `.xml`, `.xlsx`

### Documents (Will be KEPT)

- Documents: `.pdf`, `.docx`, `.doc`, `.txt`, `.md`, `.rtf`
- LaTeX: `.tex`, `.bib`, `.aux`, `.log`
- Presentations: `.pptx`, `.ppt`
- Unknown file types (kept to be safe)

## Workflow Details

For each case folder found:

1. **Jira Check**: Queries `jira_purge_query.py` to verify the case has passed through required workflow statuses:
   - "Pending openICPSR"
   - "Assess openICPSR"  
   - "Pending Publication"

2. **File Classification**: Recursively scans the folder and all subfolders to:
   - Identify data files (to delete)
   - Identify documents (to keep)
   - Handle unknown file types conservatively (keep them)

3. **Data Deletion**: Deletes all identified data files

4. **Folder Move**: Moves the entire folder (with remaining contents) to `1Completed`

## Logging

Each run creates a detailed log file: `box_cleanup_YYYYMMDD_HHMMSS.log`

The log includes:
- All actions taken (or would be taken in test mode)
- File paths and sizes
- Any errors encountered
- Summary statistics

Console output shows:
- Progress through each case
- Files being deleted
- Folders being moved
- Final summary

## Safety Features

- **Test mode**: Complete dry-run capability
- **Confirmation prompts**: Required for batch operations (>1 folder)
- **Conservative file handling**: Unknown file types are preserved
- **Error recovery**: Continues processing remaining cases if one fails
- **Detailed logging**: Full audit trail of all operations
- **Exit detection**: Folder name conflicts in 1Completed are handled gracefully

## Examples

### Example 1: Check Status First

```bash
# List all cases and check which are ready for purge
python3 clean_box_folders.py --list

# Check specific case
python3 clean_box_folders.py --list --case 1234
```

### Example 2: Initial Exploration
```bash
# See what cases exist and which are ready
python3 clean_box_folders.py --test

# Review the log file
cat box_cleanup_*.log
```

### Example 3: Process Single Case
```bash
# Test a specific case first
python3 clean_box_folders.py --case 1234 --test

# If it looks good, run for real
python3 clean_box_folders.py --case 1234
```

### Example 4: Batch Processing
```bash
# Test all ready cases
python3 clean_box_folders.py --test

# Review what will happen, then execute
python3 clean_box_folders.py
# (You'll be prompted to confirm)
```

### Example 5: Automated Batch (Use with Caution!)
```bash
# Skip confirmation - useful for scheduled jobs
python3 clean_box_folders.py --yes
```

## Output

### Console Output

```
2026-02-19 10:30:00 - INFO - Box Cleanup Script Started
2026-02-19 10:30:00 - INFO - Log file: box_cleanup_20260219_103000.log
2026-02-19 10:30:01 - INFO - Authenticating to Box...
2026-02-19 10:30:02 - INFO - ✓ Authenticated as: John Doe
2026-02-19 10:30:02 - INFO - Scanning root folder 123456789 for case folders...
2026-02-19 10:30:03 - INFO - Found 5 case folder(s)

============================================================
Processing: aearep-1234
============================================================
2026-02-19 10:30:04 - INFO -   ✓ Ready for purge
2026-02-19 10:30:04 - INFO -   Scanning folder contents...
2026-02-19 10:30:05 - INFO -   Found 23 data file(s) to delete
2026-02-19 10:30:05 - INFO -   Found 5 document(s) to keep
2026-02-19 10:30:05 - INFO -   Deleting data files...
2026-02-19 10:30:08 - INFO -   ✓ Deleted 23/23 data files (145.67 MB)
2026-02-19 10:30:08 - INFO -   Moving folder to '1Completed'...
2026-02-19 10:30:09 - INFO -   ✓ Moved folder 'aearep-1234' to '1Completed'

============================================================
SUMMARY
============================================================
Folders found:          5
Folders checked:        5
Folders ready to purge: 3
Folders moved:          3
Data files deleted:     67
Total bytes deleted:    423.45 MB
Errors:                 0
```

## Troubleshooting

### "BOX_FOLDER_PRIVATE environment variable not set"

Set the required Box environment variables. Check your `.bashrc` or `.zshrc`.

### "jira_purge_query.py not found"

Ensure `jira_purge_query.py` exists and is in your PATH. Typically in `$HOME/bin` or `$HOME/bin/editor-scripts`.

### "Error: boxsdk not installed"

```bash
pip install 'boxsdk[jwt]'
```

### "Failed to authenticate with BOX_PRIVATE_JSON"

Check that:
1. The base64 encoding is correct
2. The JSON contains valid credentials
3. Your service account has access to the folder

### "Folder already exists in '1Completed'"

The folder was already moved previously. The script will skip it and continue.

## Development Notes

The script is based on the authentication patterns from `download_box_private.py` and integrates with the existing `jira_purge_query.py` workflow tool.

### Testing Without Box/Jira Access

```bash
# Test the script structure without actually calling APIs
python3 -c "import clean_box_folders; print('Syntax OK')"
```

## Security Considerations

- **Never commit credentials** to version control
- Store Box config files in secure locations with restricted permissions
- Use environment variables or secure secret management
- Review log files before sharing (may contain folder/file names)
- Test mode logs may reveal folder structure

## Related Scripts

- `download_box_private.py` - Downloads content from Box folders (not required; present in each [template repository](https://github.com/AEADataEditor/replication-template))
- `jira_purge_query.py` - Checks if Jira cases are ready for purging (REQUIRED)
