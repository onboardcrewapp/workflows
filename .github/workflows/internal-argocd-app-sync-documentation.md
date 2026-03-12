<!-- markdownlint-disable -->
# ArgoCD App Sync and Release (Internal)

> **Internal workflow** used by the OnBoard Crew App team. It is maintained for our managed
> infrastructure and release conventions. Other teams are encouraged to copy and adapt the
> logic rather than calling this workflow directly.

This reusable workflow automates the process of logging into an ArgoCD server, updating an
application with new parameters, syncing the application and waiting for health checks to
complete. It includes robust retry loops and optional Slack notifications so that release status
is visible to the team.

## Overview

Steps performed by the workflow:

1. Send a "start" message to Slack (if enabled).
2. Login to the ArgoCD server using the provided credentials, with retry logic in case of
temporary failures.
3. Populate a `RELEASE_ARGS` variable from the `argocd_release_args` input and append them
   to the `argocd app set` command (empty string is accepted and simply results in no
   additional parameters).
4. Perform application operations in sequence: `set` → `sync` → `wait` (sync, operation,
   health), each with a 10‑minute timeout and retry wrapper. The job also sets the GitHub
   `environment` attribute to the value of the `argocd_app_env` input so that deployments
   are properly scoped.
5. Attempt to fetch the last 60 lines of application logs (failure is tolerated).
6. Send a Slack notification on success or failure, along with a reaction emoji.

Because the workflow uses a pinned `argocd-2.11.2` binary path and assumes specific network
directives, it’s not suitable as a drop‑in for arbitrary projects; use as inspiration instead.

## Required Inputs

| Input | Description |
|-------|-------------|
| `repository_name` | Repository name (used for logging). |
| `argocd_server_url` | URL of the ArgoCD server to target. |
| `argocd_app_name` | Name of the ArgoCD application to operate on. |
| `argocd_app_env` | Logical environment name (e.g. `staging`, `production`). Used in field
values and assigned directly to the GitHub job `environment` property so that the run
appears under the correct environment in the UI. |
| `argocd_release_args` | Arguments to pass to `argocd app set` in `KEY=VALUE` format, one per line. |
| `tag` | Docker image tag or other release identifier. |
| `runner` | GitHub runner label (defaults to `ubuntu-latest`). |
| `slack_notify_enabled` | `true`/`false` to toggle Slack messages. |
| `slack_channel_id` | Target Slack channel ID (required if notifications enabled). |


## Required Secrets

| Secret | Description |
|--------|-------------|
| `ARGOCD_USERNAME` | Username for ArgoCD login. |
| `ARGOCD_PASSWORD` | Password for ArgoCD login. |
| `SLACK_BOT_TOKEN` | Slack bot token (needed if `slack_notify_enabled` is `true`). |

## Optional Inputs and Defaults

Most inputs besides those listed above are optional or have sensible defaults. In particular
`runner` defaults to `ubuntu-latest` and `slack_notify_enabled` defaults to `false`. The
`argocd_release_args` input is technically marked required by the workflow schema, but an
empty string is accepted; leaving it empty simply means no extra `-p key=value` arguments
will be passed to `argocd app set`.

## Example 1: Minimal Call (no Slack)

```yaml
jobs:
  release:
    uses: your-org/your-repo/.github/workflows/internal-argocd-app-sync.yml@main
    with:
      repository_name: "crew-app"
      argocd_server_url: "https://argocd.example.com"
      argocd_app_name: "crew-app-staging"
      argocd_app_env: "staging"
      argocd_release_args: |
        image.tag=latest
      tag: "latest"
      slack_notify_enabled: false
    secrets:
      ARGOCD_USERNAME: ${{ secrets.ARGOCD_USERNAME }}
      ARGOCD_PASSWORD: ${{ secrets.ARGOCD_PASSWORD }}
```

## Example 2: Slack Notifications & Multiple Arguments

```yaml
jobs:
  release:
    uses: your-org/your-repo/.github/workflows/internal-argocd-app-sync.yml@main
    with:
      repository_name: "crew-app"
      argocd_server_url: "https://argocd.example.com"
      argocd_app_name: "crew-app-prod"
      argocd_app_env: "production"
      argocd_release_args: |
        image.tag=${{ github.sha }}
        featureFlag=true
      tag: "${{ github.sha }}"
      slack_notify_enabled: true
      slack_channel_id: "C1234567890"
    secrets:
      ARGOCD_USERNAME: ${{ secrets.ARGOCD_USERNAME }}
      ARGOCD_PASSWORD: ${{ secrets.ARGOCD_PASSWORD }}
      SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

## Best Practices

1. **Scope credentials tightly** – give the ArgoCD account only the permissions needed for
   the targeted application(s).
2. **Keep release arguments simple** – treat `argocd_release_args` as a short key/value list
   (avoid very long or binary values).
3. **Environment-specific config** – maintain separate workflows or inputs for staging vs
   production if your ArgoCD configuration differs.
4. **Use Slack for visibility** – enabling notifications helps catch failed releases quickly.
5. **Pin the Argocd CLI** – the workflow uses a fixed binary path; update the workflow when
   upgrading the version.

## Troubleshooting

- **Login failures**: Ensure the server URL is reachable from the runner and credentials are
  correct; the workflow retries three times before aborting.
- **Sync errors**: Check that the `argocd_release_args` are syntactically valid and that the
  application exists; logs in the action will show each command invocation.
- **Timeouts**: The `app wait` commands have a 600‑second timeout per phase; if the cluster is
  slow, consider splitting the operation into multiple jobs with manual control.
- **Slack messages missing**: Verify the bot token has `chat:write` and `reactions:write` scopes
  and that `slack_notify_enabled` is `true` with a valid `slack_channel_id`.

## Notes for External Use

This document is deliberately detailed but the workflow may reference internal tooling and
is optimized for our infrastructure. Treat it as a blueprint rather than a drop‑in
solution and feel free to copy it into your own repository when building your own
ArgoCD release process.
