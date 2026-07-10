# LCNAF Agent Reconciliation Script

This script reconciles ArchivesSpace agent records with Library of Congress Name Authority File (LCNAF) records from `id.loc.gov`.

It is designed to help with authority control for ArchivesSpace **people**, **corporate entities**, and **families** by finding likely LCNAF matches, writing reviewable reports, and optionally updating records in ArchivesSpace.

## What the script does

For each ArchivesSpace agent record, the script:

1. retrieves the full agent record from ArchivesSpace,
2. skips the record if it already has an LCNAF authority ID,
3. builds one or more candidate name labels from the existing name,
4. searches LCNAF for a matching authority record,
5. compares the LCNAF result against the ArchivesSpace record,
6. updates the ArchivesSpace name with LC authority information if a match is found,
7. writes a JSONL report of what happened,
8. and, in apply mode, saves the change back to ArchivesSpace.

If the script encounters an LC authority ID conflict during update, it can also merge the current ArchivesSpace agent into the conflicting record.

## Requirements

You will need:

* Python 3
* `requests`
* `asnake`
* access to an ArchivesSpace instance
* a `secrets.json` file containing your ArchivesSpace credentials

Example `secrets.json`:

```json
{
  "baseurl": "https://your-archivesspace.example.edu",
  "username": "your-username",
  "password": "your-password"
}
```

## Installation

Install the Python dependencies in your environment:

```bash
pip install requests asnake
```

Make sure the script can read your `secrets.json` file and can reach both ArchivesSpace and `id.loc.gov`.

## Basic usage

### Dry run

By default, the script runs in dry-run mode. It looks for matches and writes reports, but does **not** update ArchivesSpace.

```bash
python3 reconcile_lcnaf_agents.py
```

### Apply changes

Use `--apply` to actually update ArchivesSpace records.

```bash
python3 reconcile_lcnaf_agents.py --apply
```

### Limit processing to one agent type

You can restrict processing to one ArchivesSpace agent type:

* `people`
* `corporate_entities`
* `families`

```bash
python3 reconcile_lcnaf_agents.py --agent-type people
```

### Limit the number of records

Use `--limit` to stop after a certain number of records per agent type.

```bash
python3 reconcile_lcnaf_agents.py --limit 25
```

## Review workflow

The script can generate a CSV review file during a dry run. That file is meant for human review.

```bash
python3 reconcile_lcnaf_agents.py --review-csv review.csv
```

In the review file:

* `apply` starts as `no`
* a reviewer can change it to `yes` for records they want to approve
* then the script can be run again using `--apply-review-csv`

To apply only approved rows:

```bash
python3 reconcile_lcnaf_agents.py --apply-review-csv review.csv
```

Only rows with an affirmative `apply` value are processed.

Accepted values include:

* `yes`
* `y`
* `true`
* `1`
* `update`

## Command-line options

### `--secrets PATH`

Path to the ArchivesSpace credentials file.

Default:

```bash
secrets.json
```

### `--agent-type TYPE`

Process only one ArchivesSpace agent type.

### `--apply`

Actually update ArchivesSpace instead of running in dry-run mode.

### `--apply-review-csv PATH`

Apply only the rows marked for update in the review CSV.

### `--limit N`

Process only the first `N` records for each agent type.

### `--delay SECONDS`

Pause briefly after each agent lookup.

Default:

```bash
0.2
```

### `--output PATH`

Write a JSONL report file.

Default:

```bash
lcnaf_agent_updates.jsonl
```

### `--review-csv PATH`

Write a CSV review file during a dry run.

## How matching works

The script uses two main LCNAF matching approaches:

1. **Known-label lookup**
   It tries a direct lookup using the label.

2. **LC search**
   It searches `id.loc.gov` for likely authority records.

The matching logic is intentionally forgiving. It ignores some punctuation and spacing differences and tries to identify the most likely LCNAF record from the available data.

### For people

The script builds several candidate forms from fields like:

* primary name
* rest of name
* title
* suffix
* dates
* qualifier
* sort name

It tries variants with and without dates and titles so that it can match both fuller and simpler name forms.

### For corporate entities

The script builds hierarchical forms from fields like:

* primary name
* subordinate names
* number
* dates
* qualifier
* sort name

### For families

The script uses the family name and may also try the same name followed by the word `family`.

## What gets updated

When a match is found, the script updates the ArchivesSpace name to include LC authority data.

It typically sets:

* `authority_id`
* `rules`
* `source`
* `authorized`
* `is_display_name`
* `sort_name_auto_generate`

For the matched name, it also rebuilds the parsed name fields from the LCNAF label.

For corporate entities, the script may preserve the old name as a deprecated local name if it matches an LC variant label.

## Reports and output files

### JSONL report

The script writes one JSON object per processed record to the output file.

Default:

```bash
lcnaf_agent_updates.jsonl
```

This is useful for auditing what happened during a run.

### Review CSV

If you use `--review-csv`, the script writes a human-reviewable CSV file containing:

* agent type
* record ID
* ArchivesSpace URI
* current name
* proposed LC name
* LC URI
* LC label
* attempted search labels
* variant labels
* status

## Important assumptions

This script makes several strong assumptions:

* The first display name or first available name is the right one to match.
* A single LCNAF search result of the expected type is probably the right match.
* LC labels can be safely converted into ArchivesSpace name fields.
* Existing name authorization flags can be cleared and replaced.
* If two ArchivesSpace agents end up with the same LC authority ID, they should be merged.

Because of those assumptions, dry-run mode and human review are strongly recommended before applying changes.

## Cautions

This script is helpful, but it is not a fully manual authority control workflow.

Be especially careful with:

* common names
* incomplete names
* names with dates
* corporate names with similar wording
* records that already have complex name structures
* records that may represent distinct entities with similar LC authority data

The script may also merge records automatically if ArchivesSpace reports an authority ID conflict.

## Example workflow

Run a dry run and create a review file:

```bash
python3 reconcile_lcnaf_agents.py --review-csv review.csv
```

Review the CSV and mark approved rows with `apply=yes`.

Then apply only the approved rows:

```bash
python3 reconcile_lcnaf_agents.py --apply-review-csv review.csv
```

If you want to apply changes directly without review, use:

```bash
python3 reconcile_lcnaf_agents.py --apply
```

## Notes

* The script talks directly to `id.loc.gov` during matching.
* The script expects a working ArchivesSpace API session.
* The default behavior is safe: it does not change records unless you explicitly use `--apply` or `--apply-review-csv`.
* Output files are overwritten each time the script runs.
