# Build and Push Docker Image

This document provides examples and guidance for using the reusable GitHub Actions workflow defined in `build-docker.yml`. The workflow is designed to build a Docker image from your repository, optionally hydrate environment files from AWS SSM, authenticate with AWS ECR, push the built image, and send Slack notifications about progress.

## Overview

The `build-docker.yml` workflow automates the following tasks:

- Checking out the repository code
- Preparing a Slack message (optional)
- Generating a `.env` file from AWS SSM Parameter Store (optional)
- Generating .NET appsettings from AWS SSM (optional)
- Authenticating with AWS ECR with retry logic
- Building the Docker image with cache tagging
- Tagging and pushing the image to AWS ECR with retry logic
- Sending Slack notifications for start, success, and failure

It is intended to be called from another workflow using `workflow_call` so that you can centralise your build logic and reuse it across projects or environments.

## Required Parameters

These parameters must be provided when calling the workflow:

| Parameter | Description |
|-----------|-------------|
| `repository_name` | Name of the repository to build the image from |
| `build_env` | Target environment for the build (e.g. `development`, `staging`, `production`) |
| `dockerfile_path` | Path to the `Dockerfile` relative to the repository root |
| `tag` | Docker image tag to apply to the built image (e.g. `latest`, `v1.2.3`) |
| `image_name` | Name of the Docker image to be built and pushed |
| `aws_account_id` | AWS account ID where the ECR repository exists |
| `aws_region` | AWS region where the ECR repository is located (defaults to `ap-southeast-1`) |
| `aws_ssm_generate_env_file_enabled` | Enable/disable generation of a `.env` file from SSM parameters |
| `aws_ssm_generate_dotnet_app_settings_enabled` | Enable/disable generation of .NET app settings from SSM |
| `slack_notify_enabled` | Enable/disable Slack notifications for build events |

> The workflow also accepts `runner` (defaults to `ubuntu-latest`) and `dockerfile_context_path` (defaults to `.`) which are used internally but typically don't need to be changed.

## Required Secrets

| Secret | Description |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | AWS Access Key ID with permissions for SSM and ECR |
| `AWS_SECRET_ACCESS_KEY` | AWS Secret Access Key |
| `SLACK_BOT_TOKEN` | Slack Bot Token (required if `slack_notify_enabled` is set to `true`) |

## Optional Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `dockerfile_context_path` | Build context path for Docker | `.` |
| `build_args` | Build arguments in `KEY=VALUE` format, one per line | (none) |
| `runner` | GitHub runner label | `ubuntu-latest` |
| `aws_ssm_parameter_prefix` | Prefix path for SSM parameters used to create the `.env` file | - |
| `aws_ssm_dotnet_app_settings_parameter_prefix` | Name of the SSM parameter containing the .NET appsettings JSON/text | - |
| `aws_ssm_dotnet_app_settings_path_destination` | Destination path for generated .NET settings file | - |
| `aws_ssm_dotnet_app_pregenerate_custom_command` | Custom command to run before generating .NET settings (e.g. to create directories) | - |
| `slack_channel_id` | Slack channel to send build notifications to | - |

## .NET Extras

The workflow includes specialized support for retrieving application settings formatted for .NET applications. These extras are useful when your container needs an `appsettings.json` or equivalent configuration file generated at build time from AWS SSM.

- **`aws_ssm_generate_dotnet_app_settings_enabled`** – set to `true` to enable the feature. When disabled, no .NET-specific steps run.
- **`aws_ssm_dotnet_app_settings_parameter_prefix`** – the full name of the SSM parameter containing the settings. The workflow uses `aws ssm get-parameter` (not the path query) so the value can be a JSON string or any text that your application expects. Remember to call the parameter with `--with-decryption` if it is encrypted.
- **`aws_ssm_dotnet_app_settings_path_destination`** – where the parameter value will be written inside the build context. The path may be relative (e.g. `app/appsettings.json`) or absolute, depending on your Docker build. The step will create parent directories using `mkdir -p` if they don't already exist (either automatically or via the pre-generate command).
- **`aws_ssm_dotnet_app_pregenerate_custom_command`** – optionally run arbitrary shell commands before fetching the parameter. Typical uses include:
  - creating directories (`mkdir -p app/config`)
  - installing helper tools (`dotnet tool install ...`)
  - preparing environment for conditional logic

> 📌 The pre-generate command is executed with `set -e` style behavior: if it fails, the build stops with an error.

### How it works

When the feature is enabled the job executes the following logic:

1. If a pre-generate command is defined, run it. If it exits non-zero, the workflow fails immediately.
2. Ensure the destination directory exists (`mkdir -p` is run unconditionally for the parent path).
3. Attempt up to three times to fetch the value using `aws ssm get-parameter --name $PREFIX --with-decryption --query "Parameter.Value" --output text`.
4. Write the output to the destination path.
5. If the resulting file is empty, retry or fail after the third attempt.

The retry loop mirrors other AWS calls in the workflow, protecting against transient API errors.

### Edge cases and tips

- **Large settings** – SSM parameters are limited to 4 KB. For larger JSON blobs split them into multiple parameters and concatenate them in a `build_args` or `pregenerate` command before writing the final file.
- **Binary data** – the parameter value is treated as UTF-8 text. If you need binary data, base64‑encode it and decode in a custom command.
- **Multiple environments** – combine this feature with the environment helpers from Example 3. Use different parameter names per environment and ensure the destination path matches your Dockerfile's `COPY` instructions.
- **Secrecy** – the settings file is written to the workspace and then included in the Docker build. Ensure your `.dockerignore` prevents accidental inclusion in source control, or generate settings inside the container instead if secrecy is a concern.

These details make the .NET extras more transparent and easier to adapt to unusual scenarios.

## Example 1: Basic Usage

Minimal configuration for a straightforward Docker build and push:

```yaml
name: "Build Docker Image"

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  build:
    uses: your-org/your-repo/.github/workflows/build-docker.yml@main
    with:
      repository_name: "my-service"
      build_env: "production"
      dockerfile_path: "./Dockerfile"
      tag: "latest"
      image_name: "my-service"
      aws_account_id: "123456789012"
      aws_region: "us-east-1"
      aws_ssm_generate_env_file_enabled: false
      aws_ssm_generate_dotnet_app_settings_enabled: false
      slack_notify_enabled: false
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

## Example 2: Comprehensive Configuration

A full example showing all capabilities, including generating environment files, custom build arguments and Slack notifications:

```yaml
name: "Build and Push Docker Image (Staging)"

on:
  push:
    branches: [develop]
  pull_request:
    types: [closed]
    branches: [develop]
  workflow_dispatch:
    inputs:
      environment:
        description: "Environment for this build"
        required: true
        default: "staging"
        type: choice
        options:
          - staging
          - production

jobs:
  build:
    uses: your-org/your-repo/.github/workflows/build-docker.yml@main
    with:
      repository_name: "my-service"
      build_env: ${{ github.event.inputs.environment || 'staging' }}
      dockerfile_path: "services/my-service/Dockerfile"
      dockerfile_context_path: "services/my-service"
      build_args: |
        NODE_ENV=production
        API_URL=https://api.example.com
      tag: "${{ github.sha }}"
      image_name: "my-service"
      aws_account_id: "123456789012"
      aws_region: "us-west-2"
      aws_ssm_generate_env_file_enabled: true
      aws_ssm_parameter_prefix: "/my-service/${{ github.event.inputs.environment || 'staging' }}/"
      aws_ssm_generate_dotnet_app_settings_enabled: false
      slack_notify_enabled: true
      slack_channel_id: "C1234567890"
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

## Example 3: Multi-Environment Pipeline with .NET Settings

Demonstrates using a helper job to choose environment and generate both `.env` and .NET appsettings, plus a pre‑generate command for directory creation:

```yaml
name: "Docker Build Multi-Environment"

on:
  push:
    branches: [main, develop]
  workflow_dispatch:
    inputs:
      environment:
        description: "Environment to build for"
        required: true
        default: "development"
        type: choice
        options:
          - production
          - staging
          - development

jobs:
  set-env:
    runs-on: ubuntu-latest
    outputs:
      env: ${{ steps.set.outputs.env }}
    steps:
      - id: set
        run: |
          if [[ "${{ github.event_name }}" == "workflow_dispatch" ]]; then
            echo "env=${{ github.event.inputs.environment }}" >> $GITHUB_OUTPUT
          elif [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
            echo "env=production" >> $GITHUB_OUTPUT
          elif [[ "${{ github.ref }}" == "refs/heads/develop" ]]; then
            echo "env=staging" >> $GITHUB_OUTPUT
          else
            echo "env=development" >> $GITHUB_OUTPUT
          fi

  build:
    needs: set-env
    uses: your-org/your-repo/.github/workflows/build-docker.yml@main
    with:
      repository_name: "my-dotnet-service"
      build_env: ${{ needs.set-env.outputs.env }}
      dockerfile_path: "Dockerfile"
      dockerfile_context_path: "."
      tag: "${{ github.sha }}"
      image_name: "my-dotnet-service"
      aws_account_id: "123456789012"
      aws_region: "eu-central-1"
      aws_ssm_generate_env_file_enabled: true
      aws_ssm_parameter_prefix: "/my-dotnet-service/${{ needs.set-env.outputs.env }}/"
      aws_ssm_generate_dotnet_app_settings_enabled: true
      aws_ssm_dotnet_app_settings_parameter_prefix: "/my-dotnet-service/${{ needs.set-env.outputs.env }}/appsettings"
      aws_ssm_dotnet_app_settings_path_destination: "app/appsettings.json"
      aws_ssm_dotnet_app_pregenerate_custom_command: "mkdir -p app"
      slack_notify_enabled: true
      slack_channel_id: "C1234567890"
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

## Best Practices

1. **Sanitise branch names** – the workflow converts slashes, underscores and uppercase letters in branch names when creating cache tags. Be aware of the final tag format.
2. **Lock down AWS permissions** – grant the workflow the minimum necessary rights to read SSM parameters and push to ECR.
3. **Use SSM prefixes** – store per‑environment variables under distinct prefixes and enable `aws_ssm_generate_env_file_enabled` to keep builds identical to runtime.
4. **Leverage custom commands** – the `.NET pre‑generate` hook can create directories, install tools, or perform any preparatory work before retrieving settings.
5. **Notification channels** – set `slack_channel_id` and enable Slack notifications to track builds in real time. The workflow posts start, success, failure messages and reactions.

## Advanced Usage

### Custom Build Arguments

You can pass multiple `build_args` lines to inject build‑time variables into the Dockerfile:

```yaml
build_args: |
  API_URL=https://api.example.com
  FEATURE_FLAG=true
```

The workflow loops over the lines and appends `--build-arg` flags to the `docker build` command.

### Pre-populated .NET Settings

For .NET projects that need an `appsettings.json`, set:

```yaml
aws_ssm_generate_dotnet_app_settings_enabled: true
aws_ssm_dotnet_app_settings_parameter_prefix: "/path/to/parameter"
aws_ssm_dotnet_app_settings_path_destination: "src/app/appsettings.json"
```

Optionally provide a `aws_ssm_dotnet_app_pregenerate_custom_command` to create directories or run helper scripts.

### Retry Logic

All network operations – SSM retrieval, ECR login, Docker push – include built‑in retries to make CI more resilient to transient failures. You do not need to implement your own loops unless you have special requirements.

## Troubleshooting

### `.env` File Not Generated

1. Verify the AWS credentials have permission to call `ssm:GetParametersByPath`.
2. Ensure the `aws_ssm_parameter_prefix` ends with a slash (`/`) and matches the parameter names.
3. Check the action logs for the retry attempts and any error messages from the AWS CLI.

### ECR Authentication Fails

1. Confirm `aws_account_id`, `aws_region`, and credentials are correct.
2. Make sure the IAM user/role has `ecr:GetAuthorizationToken` permission.
3. Review the log lines that indicate each login attempt and the reason for failure.

### Docker Build or Push Errors

1. Ensure the `dockerfile_path` and `dockerfile_context_path` exist and are accessible.
2. Validate your build arguments; malformed lines may cause the build to error.
3. For push issues, make sure the ECR repository exists and the account has push rights.

### Slack Messages Not Sending

1. Confirm `SLACK_BOT_TOKEN` is set and has `chat:write` and `reactions:write` scopes.
2. Check that `slack_notify_enabled` is `true` and a valid `slack_channel_id` is specified.

---

Feel free to copy this file into any public repository; it contains only generic instructions and example usage without any proprietary details.
