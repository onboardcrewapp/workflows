<!-- markdownlint-disable -->
# PostgreSQL Dump and Restore

This reusable workflow performs a logical dump of a PostgreSQL "source" database and
restores it into a specified "destination" database. It exposes a variety of options to
control the dump/restore behaviour, including format, object ownership, privileges, and
table exclusions. Slack notifications may be sent at the start, success, and failure of the
operation.

The workflow is designed for use via `workflow_call` from parent workflows that manage
operational tasks such as migrations, environment clones, or backups.

## Overview

The single job (`dump_and_restore`) runs on a configurable runner and executes the
following sequence:

1. (Optional) Post a start message to Slack with details of source and destination.
2. Construct `pg_dump` options based on inputs (format, blobs, owners, privileges, ACLs,
   table exclusions, etc.).
3. Perform the dump to a temporary directory unique to the run.
4. Check whether the destination database exists:
   - If not, create it.
   - If yes, terminate active connections using a retry loop, then optionally drop and
     recreate it unless `skip_database_drop` is `true`.
5. Run `pg_restore` with the computed options and `--clean --if-exists` to ensure objects
   are overwritten.
6. Clean up the dump files and directory.
7. Send Slack success or failure notifications with a reaction emoji.

The script logs commands with `set -x` and will exit with a non-zero code on any failure,
causing the job to fail.

## Required Inputs

| Input | Description |
|-------|-------------|
| `runner` | GitHub runner label to execute on. |
| `source_db_host` | Hostname or IP of the source database. |
| `source_db_port` | TCP port of the source database (e.g. `5432`). |
| `source_db_name` | Name of the source database to dump. |
| `destination_db_host` | Hostname or IP of the destination database. |
| `destination_db_port` | TCP port of the destination database. |
| `destination_db_name` | Name of the destination database to restore into. |
| `slack_notify_enabled` | `true`/`false` to toggle Slack notifications. |
| `slack_channel_id` | Slack channel ID (required if notifications are enabled). |

## Optional Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `format` | Dump format (`custom`/`plain`/`directory`/`tar`). | `c` (custom) |
| `blobs_enabled` | Include large objects in the dump. | `true` |
| `no_owner_enabled` | Add `--no-owner` to dump/restore. | `true` |
| `no_privileges_enabled` | Add `--no-privileges`. | `true` |
| `no_acl_enabled` | Add `--no-acl`. | `true` |
| `exclude_tables_from_dump` | Space-separated list of tables to skip. | (empty) |
| `skip_database_drop` | If `true`, restore without dropping the existing database. | `false` |

## Required Secrets

| Secret | Description |
|--------|-------------|
| `SOURCE_DB_USERNAME` | Username for source database connection. |
| `SOURCE_DB_PASSWORD` | Password for source database connection. |
| `DESTINATION_DB_USERNAME` | Username for destination database. |
| `DESTINATION_DB_PASSWORD` | Password for destination database. |
| `SLACK_BOT_TOKEN` | Slack bot token if notifications are used. |

## Examples

### Simple copy

```yaml
jobs:
  copy-db:
    uses: your-org/your-repo/.github/workflows/postgresql-dump-and-restore.yml@main
    with:
      runner: "ubuntu-latest"
      source_db_host: "db1.example.com"
      source_db_port: "5432"
      source_db_name: "app_prod"
      destination_db_host: "db2.example.com"
      destination_db_port: "5432"
      destination_db_name: "app_staging"
      slack_notify_enabled: false
    secrets:
      SOURCE_DB_USERNAME: ${{ secrets.SOURCE_DB_USERNAME }}
      SOURCE_DB_PASSWORD: ${{ secrets.SOURCE_DB_PASSWORD }}
      DESTINATION_DB_USERNAME: ${{ secrets.DESTINATION_DB_USERNAME }}
      DESTINATION_DB_PASSWORD: ${{ secrets.DESTINATION_DB_PASSWORD }}
```

### Excluding audit tables and preserving destination

```yaml
jobs:
  copy-db:
    uses: your-org/your-repo/.github/workflows/postgresql-dump-and-restore.yml@main
    with:
      runner: "self-hosted-linux"
      source_db_host: "db1.internal"
      source_db_port: "5432"
      source_db_name: "app_prod"
      destination_db_host: "db2.internal"
      destination_db_port: "5432"
      destination_db_name: "app_dev"
      exclude_tables_from_dump: "logs audit_trail"
      skip_database_drop: true
      slack_notify_enabled: true
      slack_channel_id: "C9876543210"
    secrets:
      SOURCE_DB_USERNAME: ${{ secrets.SOURCE_DB_USERNAME }}
      SOURCE_DB_PASSWORD: ${{ secrets.SOURCE_DB_PASSWORD }}
      DESTINATION_DB_USERNAME: ${{ secrets.DESTINATION_DB_USERNAME }}
      DESTINATION_DB_PASSWORD: ${{ secrets.DESTINATION_DB_PASSWORD }}
      SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

## Best Practices

1. **Test on non‑production** databases first – dumps can be large and restores destructive.
2. **Use network proximity** – run the workflow on a runner with fast access to both
   source and destination servers to minimise transfer time.
3. **Regularly rotate credentials** stored in secrets.
4. **Verify compatibility** – ensure both Postgres servers are compatible versions, especially
   when using custom dump format.
5. **Exclude unnecessary tables** (e.g. audit logs) to reduce dump size and time.

## Troubleshooting

- **Dump command fails**: check that `pg_dump` exists on the runner and that the source
  credentials/host/port are correct.
- **Restore errors**: review the logs for permission errors or missing schemas; the script
  uses `--clean --if-exists` which may cause harmless object-not-found messages.
- **Connections cannot be terminated**: ensure the user has sufficient privileges to run
  `pg_terminate_backend`; the job retries three times and aborts on failure.
- **Destination DB not dropped when expected**: confirm `skip_database_drop` is not set to
  `true` and that the workflow has rights to `DROP DATABASE`.

This workflow is generic and can be adapted for other RDBMS tools with minimal modifications. It
serves well for platform migrations, environment cloning, or emergency recoveries.

