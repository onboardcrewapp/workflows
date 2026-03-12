<!-- markdownlint-disable -->
# Debezium Connectors Operations

A minimal reusable workflow for performing basic operations on Debezium connectors via the
Debezium REST API.

It is intentionally lightweight and only supports two actions: pause and resume. The workflow is
meant to be called by other workflows that need to temporarily suspend a connector (for schema
changes, maintenance, etc.) and then resume it later.

## Overview

The job named `debezium_operation` runs on the specified GitHub runner and executes a `curl`
command against the provided Debezium API URL. Depending on the boolean input
`suspend_enabled`, it either:

- Issues a `PUT /connectors/{name}/pause` request to stop the connector, or
- Issues a `PUT /connectors/{name}/resume` request to restart it.

No authentication or additional headers are included; callers must ensure the runner has
network access to the Debezium server and that the service allows unauthenticated requests or
they handle auth themselves by surrounding the call with environment variables or wrapper
dispatches.

## Required Inputs

| Input | Description |
|-------|-------------|
| `connector_name` | Name of the Debezium connector to operate on. |
| `debezium_url` | Base URL of the Debezium REST API (e.g. `http://debezium.example.com:8083`). |
| `suspend_enabled` | `true` to pause the connector, `false` to resume it. |

## Optional Input

| Input | Description | Default |
|-------|-------------|---------|
| `runner` | GitHub runner label to execute on. | `ubuntu-latest` |

## Example Usage

```yaml
jobs:
  toggle-connector:
    uses: your-org/your-repo/.github/workflows/debezium-connectors-operations.yml@main
    with:
      connector_name: "inventory-connector"
      debezium_url: "http://debezium.local:8083"
      suspend_enabled: true
      runner: "self-hosted-asia"
```

Calling again with `suspend_enabled: false` will resume the connector.

## Notes and Limitations

- There is no built-in retry logic; if the `curl` command fails the job will error. Redundant
  calls or surrounding logic should handle failures if needed.
- Authentication must be managed externally (e.g. via an additional `Authorization` header set
  through environment variables or the runner's configuration).
- The workflow makes no assumptions about connector configuration; it simply delegates to the
  Debezium API.

Feel free to copy or extend this workflow for more complex connector management tasks such as
status checks, configuration updates, or error handling.
