<!-- markdownlint-disable -->
# Provision EC2 Ephemeral Runner for EAS (Internal)

> **Internal workflow** used by the OnBoard Crew App team. It is maintained for our managed
> infrastructure and release conventions. Other teams are encouraged to copy and adapt the
> logic rather than calling this workflow directly.

This reusable workflow automates the lifecycle (provisioning and destroying) of an EC2 ephemeral runner specifically designed for Expo Application Services (EAS) Android builds. It uses Terraform to manage the AWS EC2 infrastructure and Ansible to configure and register the instance as a GitHub self-hosted runner.

## Overview

Depending on the `kind` input (`provision` or `destroy`), the workflow performs the following operations:

### Provisioning (`kind: provision`)

1. **Checkout**: Clones the specified Bifrost infrastructure repository containing Terraform and Ansible code.
2. **Input Validation**: Ensures required paths and configurations are provided and validates that the `terraform_workdir` targets an `eas-github` scope for security.
3. **Environment Setup**: Installs the specified version of Terraform, configures AWS credentials, and sets up a local provider mirror for caching.
4. **Pre-cleanup**: Best-effort destruction of existing resources and cleanup of any offline runners in the GitHub organization sharing the same label.
5. **Terraform Apply**: Initializes and applies the Terraform configuration to provision the EC2 instance.
6. **Ansible Configuration**: Fetches SSH keys and Ansible Vault passwords securely from AWS SSM Parameter Store, then runs the provided Ansible playbook to configure the instance and register it as a runner.
7. **Wait for Runner**: Polls the GitHub API (up to 10 minutes) until the newly provisioned runner is reported as `online`.

### Destruction (`kind: destroy`)

1. **Environment Setup**: Clones the Bifrost repository, installs Terraform, and configures AWS credentials.
2. **Terraform Destroy**: Runs `terraform destroy` with automatic retries to tear down the EC2 infrastructure.
3. **Runner Cleanup**: Removes the offline runner entry from the GitHub organization using the provided label.

## Required Inputs

| Input | Description |
|-------|-------------|
| `kind` | Action to perform: `"provision"` or `"destroy"`. |
| `bifrost_repository` | Name of the infrastructure repository to check out. |
| `terraform_workdir` | Path to the Terraform directory (must contain `eas-github`). |
| `ephemeral_runner_label` | The label assigned to the runner (used for wait/cleanup logic). |
| `runner` | GitHub runner label to execute this provisioning workflow. |
| `ansible_ssh_key_aws_ssm_parameter_store_path` | SSM Parameter Store path containing the SSH private key (required even for destroy, though only used in provision). |
| `ansible_vault_aws_ssm_parameter_store_path` | SSM Parameter Store path containing the Ansible Vault password (required). |

*Note: For `provision`, the inputs `ansible_workdir`, `ansible_config`, and `ansible_playbook` are practically required by validation logic.*

## Required Secrets

| Secret | Description |
|--------|-------------|
| `SELF_HOSTED_RUNNER_ACCESS_TOKEN` | Personal Access Token (PAT) used for cross-repo checkout and GitHub API calls (runner registration/cleanup). |
| `AWS_ROLE_TO_ASSUME` | IAM Role ARN to assume via OIDC for Terraform provisioning and SSM access. |

## Optional Inputs and Defaults

- `bifrost_repository_branch_ref` (default: `"main"`): Branch to clone from the Bifrost repository.
- `terraform_version` (default: `"1.13.1"`): Version of the Terraform CLI to install.
- `aws_region` (default: `"ap-southeast-1"`): AWS Region for AWS credentials.
- `ansible_workdir` (default: `""`): Path to the Ansible working directory.
- `ansible_playbook` (default: `""`): Name or relative path of the Ansible playbook.
- `ansible_config` (default: `""`): Name or relative path of the Ansible config (`ansible.cfg`).
- `ansible_ssh_enabled` (default: `true`): Whether to fetch and configure the SSH key from SSM.
- `ansible_vault_enabled` (default: `true`): Whether to fetch and configure the Ansible Vault password from SSM.

## Example 1: Provisioning a Runner

```yaml
jobs:
  provision-runner:
    permissions:
      id-token: write
      contents: read
    uses: your-org/your-repo/.github/workflows/internal-provision-eas-android-ec2-ephemeral-runner.yml@main
    with:
      kind: "provision"
      runner: "ubuntu-latest"
      bifrost_repository: "your-org/bifrost-infra"
      terraform_workdir: "terraform/eas-github-runner"
      ansible_workdir: "ansible/eas-github-runner"
      ansible_config: "ansible.cfg"
      ansible_playbook: "playbook.yml"
      ephemeral_runner_label: "eas-android-ephemeral-1"
      ansible_ssh_key_aws_ssm_parameter_store_path: "/bifrost/ansible/ssh_key"
      ansible_vault_aws_ssm_parameter_store_path: "/bifrost/ansible/vault_pass"
    secrets:
      SELF_HOSTED_RUNNER_ACCESS_TOKEN: ${{ secrets.PAT_TOKEN }}
      AWS_ROLE_TO_ASSUME: ${{ secrets.AWS_ROLE_TO_ASSUME }}
```

## Example 2: Destroying a Runner

```yaml
jobs:
  destroy-runner:
    permissions:
      id-token: write
      contents: read
    uses: your-org/your-repo/.github/workflows/internal-provision-eas-android-ec2-ephemeral-runner.yml@main
    with:
      kind: "destroy"
      runner: "ubuntu-latest"
      bifrost_repository: "your-org/bifrost-infra"
      terraform_workdir: "terraform/eas-github-runner"
      ephemeral_runner_label: "eas-android-ephemeral-1"
      # These must be provided to satisfy the schema, even if unused during destroy
      ansible_ssh_key_aws_ssm_parameter_store_path: "/bifrost/ansible/ssh_key"
      ansible_vault_aws_ssm_parameter_store_path: "/bifrost/ansible/vault_pass"
    secrets:
      SELF_HOSTED_RUNNER_ACCESS_TOKEN: ${{ secrets.PAT_TOKEN }}
      AWS_ROLE_TO_ASSUME: ${{ secrets.AWS_ROLE_TO_ASSUME }}
```

## Best Practices

1. **Concurrency and State**: The workflow handles concurrency by grouping runs based on the repository, terraform working directory, and the action kind. Ensure your Terraform backend securely supports state locking (e.g., S3 + DynamoDB).
2. **Security**: The `terraform_workdir` is strictly validated to contain `eas-github` to prevent arbitrary infrastructure from being executed by this specific workflow.
3. **Resilience**: The workflow implements retry logic around Terraform initialization and destruction steps to handle transient network or AWS API errors.
4. **Cleanup**: Even during a `provision` action, the workflow does a best-effort `terraform destroy` and GitHub API cleanup of offline runners sharing the same label to ensure a clean slate.

## Troubleshooting

- **Validation Failure (`eas-github`)**: Ensure your `terraform_workdir` contains the string `eas-github`. This is a hardcoded safety check.
- **Ansible SSM Parameter Errors**: If the SSH key or Vault password fetch fails during provisioning, ensure the IAM role assumed (or the credentials provided) has `ssm:GetParameter` permissions for the specified paths.
- **Timeout Waiting for Runner**: If the job times out waiting for the runner to come online (10 minutes), the EC2 instance might have failed to register. Check the Ansible output step for configuration errors or SSH connectivity issues.
- **Destroy Fallback**: If the initial `terraform destroy` fails, the workflow will fallback to a `destroy` without state refresh (`-refresh=false`). Ensure your AWS resources don't have deletion protection enabled.
