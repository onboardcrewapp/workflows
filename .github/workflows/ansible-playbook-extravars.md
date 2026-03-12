<!-- markdownlint-disable -->
# Ansible Playbook with Extra Vars

This reusable workflow provides a standardized way to run an Ansible playbook from GitHub Actions.
It includes support for:

- Fetching an SSH private key from AWS SSM and configuring `ssh-agent` so Ansible can connect to
  remote hosts.
- Fetching an Ansible Vault password from AWS SSM and passing it to the `ansible-playbook`
  command.
- Supplying arbitrary `--extra-vars` to the playbook via workflow inputs.
- Posting Slack messages at start, success, and failure for visibility.

It is intended to be called with `workflow_call` from other workflows that orchestrate deployments
or infrastructure changes.

## Overview

The job named `ansible` executes on an `onpremise-ubuntu-general` runner (adjustable via the
caller). It performs:

1. Checkout of repository code.
2. Optionally send a "start" Slack notification describing the playbook and environment.
3. Create a temporary directory used for SSH and Vault artifacts.
4. If `ansible_ssh_enabled` is `true`, retrieve the private key from SSM, secure it, load it into
   `ssh-agent`, and keep the agent running for the duration of the playbook run.
5. If `ansible_vault_enabled` is `true`, fetch the Vault password from SSM and set
   `ANSIBLE_VAULT_PASSWORD_FILE` to point at it.
6. Run `ansible-playbook` with the provided inventory, playbook path, extra vars, and the vault
   password file (if required).
7. Clean up the SSH agent and remove the temporary directory.
8. Send Slack success/failure messages and reactions depending on job outcome.

The workflow exports AWS credentials and region into the environment so the AWS CLI can fetch
parameters from the SSM Parameter Store. SSH host key checking is disabled and Python warnings
are suppressed to reduce noise.

## Required Inputs

| Input | Description |
|-------|-------------|
| `repository_name` | Name of the repository (for log fields). |
| `environment` | Logical target environment (e.g. `development`, `staging`, `production`). |
| `ansible_playbook` | Path to the playbook file to execute. |
| `ansible_inventory_file` | Inventory file or host pattern. |

## Optional Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `ansible_ssh_enabled` | Whether to set up an SSH key from SSM. | `true` |
| `ansible_ssh_key_aws_ssm_parameter_store_path` | SSM path containing the private key. Required if SSH enabled. | - |
| `ansible_vault_enabled` | Whether to fetch a Vault password from SSM. | `false` |
| `ansible_vault_aws_ssm_parameter_store_path` | SSM path containing the Vault password. Required if Vault enabled. | - |
| `ansible_extra_vars` | Additional variables to pass (`KEY=VALUE` syntax). | - |
| `aws_account_id` | AWS account used when querying SSM (not used in the job directly). | - |
| `aws_region` | AWS region for SSM (defaults to `ap-southeast-1`). | `ap-southeast-1` |
| `slack_notify_enabled` | Enable Slack notifications. | `false` |
| `slack_channel_id` | Channel ID for Slack messages (required if notifications enabled). | - |
| `runner` | Runner label (configured in workflow but not exposed as input). | `onpremise-ubuntu-general` |

## Required Secrets

| Secret | Description |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | For the AWS CLI to access SSM. |
| `AWS_SECRET_ACCESS_KEY` | Associated secret key. |
| `SLACK_BOT_TOKEN` | Token for sending Slack messages (needed if notifications enabled). |

## Example Usage

A caller workflow might look like this:

```yaml
jobs:
  deploy-inventory:
    uses: your-org/your-repo/.github/workflows/ansible-playbook-extravars.yml@main
    with:
      repository_name: "infra"
      environment: "staging"
      ansible_playbook: "deploy/site.yml"
      ansible_inventory_file: "inventories/staging/hosts.ini"
      ansible_ssh_enabled: true
      ansible_ssh_key_aws_ssm_parameter_store_path: "/ssh/keys/deploy"
      ansible_vault_enabled: true
      ansible_vault_aws_ssm_parameter_store_path: "/ansible/vault/password"
      ansible_extra_vars: "version=${{ github.sha }}"
      aws_account_id: "123456789012"
      aws_region: "us-east-1"
      slack_notify_enabled: true
      slack_channel_id: "C1234567890"
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

## Best Practices

1. **Minimize privileges** – ensure the AWS credentials have only `ssm:GetParameter` privileges for
   the required paths.
2. **Protect SSH keys** – use a separate parameter for each environment and rotate regularly.
3. **Vault security** – store only passwords in SSM; avoid embedding secrets in the repository or logs.
4. **Use extra vars sparingly** – long strings or sensitive data should be avoided; consider using
   a separate Vault parameter if needed.
5. **Cleanup** – the workflow automatically kills `ssh-agent` and removes temp files; do not rely on
   leftover state between runs.

## Troubleshooting

- **SSH key fetch fails** – confirm the SSM parameter name is correct and the key has proper
  permissions (`--with-decryption` will decrypt encrypted values).
- **Vault password not found** – ensure the parameter exists and your AWS region/account are correct.
- **Playbook errors** – examine the action logs; the `ansible-playbook` command is echoed and any
  Ansible failures will surface as non-zero exit codes.
- **Slack messages missing** – verify the Slack bot token scopes and that notifications are enabled
  with a valid channel ID.

The workflow is flexible and can serve as the basis for most Ansible deployments executed from
GitHub Actions. Adjust runner labels, inventory locations, and notification details according to your
organization's needs.
