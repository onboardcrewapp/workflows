<!-- markdownlint-disable -->
# Build EAS Android Local with EC2 Ephemeral Runner (Internal)

> **Internal workflow** used by the OnBoard Crew App team. It is maintained for our managed
> infrastructure and release conventions. Other teams are encouraged to copy and adapt the
> logic rather than calling this workflow directly.

This reusable workflow automates the process of building an Android application locally using Expo Application Services (EAS) on an EC2 ephemeral runner. It handles dependency installation (Node.js, Java, Android SDK/NDK), caching, EAS configuration, the actual build process, artifact uploading to AWS S3, and optional distribution via Firebase App Distribution. It includes optional Slack notifications to keep the team updated on the build progress.

## Overview

Steps performed by the workflow:

1. **Input Validation**: Ensures all conditionally required inputs are provided (e.g., Slack channel ID if notifications are enabled).
2. **Slack Notification (Start)**: Sends a start message to a designated Slack channel with build context.
3. **Environment Setup**: Installs required tooling including Node.js, Java, Android SDK, and Android NDK.
4. **Dependency Installation**: Generates cache keys based on source and lock files, caches node modules, and installs dependencies via NPM or Yarn.
5. **EAS and Gradle Setup**: Configures Expo (EAS CLI) using the provided token, handles Hermes compiler configuration for ARM hosts, and caches Gradle.
6. **Build Execution**: Runs the provided local EAS build command with strict error handling, verifying the output of `.apk` or `.aab` artifacts.
7. **Artifact Upload (Optional)**: Uploads the generated artifacts to an AWS S3 bucket with configurable retry logic.
8. **Firebase App Distribution (Optional)**: Installs the Firebase CLI and distributes the built APK to configured tester groups.
9. **Slack Notification (Success)**: Updates the initial Slack message with a success status, artifacts location, and a direct download link if S3 upload was enabled.

## Required Inputs

| Input | Description |
|-------|-------------|
| `repository_name` | Name of the repository (used for logging and S3 path). |
| `node_version` | Node.js version to use (e.g., `"20"`). |
| `eas_version` | Version of EAS CLI to install. |
| `runner` | GitHub runner label to execute on (e.g., `self-hosted-android`). |
| `aws_region` | AWS Region where the S3 bucket is located. |
| `slack_notify_enabled` | `true`/`false` to toggle Slack messages. |
| `build_commands` | EAS build command to execute (e.g., `eas build --local --platform android`). |

## Required Secrets

| Secret | Description |
|--------|-------------|
| `AWS_ROLE_TO_ASSUME` | IAM Role ARN to assume via OIDC for AWS operations. |
| `EXPO_TOKEN` | Token for authenticating with Expo/EAS. |

*Note: Depending on optional features enabled, `SLACK_BOT_TOKEN` and `FIREBASE_TOKEN` may also be required.*

## Optional Inputs and Defaults

The workflow provides extensive customization through optional inputs:

### Build & Environment
- `node_package_manager` (default: `"auto"`): Package manager to use (`auto`, `yarn`, `npm`).
- `node_frozen_lockfile` (default: `true`): Use frozen lockfile during install.
- `java_version` (default: `"21"`): Java version for the Android build.
- `android_ndk_version` (default: `"23.1.7779620"`): Android NDK version.
- `eas_build_profile` (default: `"preview"`): EAS build profile used for logging context.

### S3 Upload
- `s3_upload_artifacts_enabled` (default: `false`): Toggle S3 artifact upload.
- `s3_build_artifacts_bucket`: S3 bucket to store artifacts (required if S3 upload is enabled).

### Firebase Distribution
- `firebase_app_distribution_enabled` (default: `false`): Toggle Firebase App Distribution.
- `firebase_app_id`: Firebase App ID (required if enabled).
- `firebase_groups` / `firebase_testers`: Targeting for the distribution.

### Slack Notifications
- `slack_channel_id`: Target Slack channel ID (required if `slack_notify_enabled` is `true`).

## Example 1: Basic Build with S3 Upload

```yaml
jobs:
  build:
    permissions:
      id-token: write
      contents: read
    uses: your-org/your-repo/.github/workflows/internal-build-eas-android-ec2-ephemeral-runner.yml@main
    with:
      repository_name: "crew-app"
      node_version: "20"
      eas_version: "latest"
      runner: "self-hosted-android"
      aws_region: "ap-southeast-1"
      build_commands: "npx eas-cli build --local --platform android --profile preview"
      slack_notify_enabled: false
      s3_upload_artifacts_enabled: true
      s3_build_artifacts_bucket: "my-build-artifacts-bucket"
    secrets:
      AWS_ROLE_TO_ASSUME: ${{ secrets.AWS_ROLE_TO_ASSUME }}
      EXPO_TOKEN: ${{ secrets.EXPO_TOKEN }}
```

## Example 2: Full Build with Firebase & Slack

```yaml
jobs:
  build_and_distribute:
    permissions:
      id-token: write
      contents: read
    uses: your-org/your-repo/.github/workflows/internal-build-eas-android-ec2-ephemeral-runner.yml@main
    with:
      repository_name: "crew-app"
      node_version: "20"
      eas_version: "latest"
      runner: "self-hosted-android"
      aws_region: "ap-southeast-1"
      build_commands: "npx eas-cli build --local --platform android --profile development"
      
      # Slack
      slack_notify_enabled: true
      slack_channel_id: "C1234567890"
      
      # S3 Upload
      s3_upload_artifacts_enabled: true
      s3_build_artifacts_bucket: "my-build-artifacts-bucket"
      
      # Firebase Distribution
      firebase_app_distribution_enabled: true
      firebase_app_id: "1:1234567890:android:0a1b2c3d4e5f67890"
      firebase_groups: "qa-team,internal-testers"
      firebase_release_notes: "New staging build ${{ github.sha }}"
    secrets:
      AWS_ROLE_TO_ASSUME: ${{ secrets.AWS_ROLE_TO_ASSUME }}
      EXPO_TOKEN: ${{ secrets.EXPO_TOKEN }}
      SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
      FIREBASE_TOKEN: ${{ secrets.FIREBASE_TOKEN }}
```

## Best Practices

1. **Caching Efficiency**: Ensure your repository structure correctly utilizes the caching setup (checking `yarn.lock`, `package-lock.json`, and tracking source files) to drastically reduce build times.
2. **Resource Management**: Since this targets an EC2 ephemeral runner, keep build commands optimized and monitor resource usage (especially memory on Android builds).
3. **Artifact Handling**: Use the S3 artifact upload for storing build history and obtaining a direct download link.
4. **Hermes Support**: The workflow includes automatic configuration for the Hermes JS engine when running on ARM architectures.

## Troubleshooting

- **Validation Errors**: If the workflow fails immediately, check the "Validate Required Inputs" step to ensure you provided all necessary parameters for enabled features (like `slack_channel_id` or `firebase_app_id`).
- **Dependency Issues**: If NPM/Yarn fails, you can toggle `node_frozen_lockfile` to `false` or explicitly set the `node_package_manager`.
- **Hermes/ARM Build Errors**: If running on an ARM-based EC2 runner and facing build errors related to `hermesc`, check the "Configure Hermes for ARM hosts" step logs.
- **Missing Artifacts**: The build step explicitly looks for `.apk` and `.aab` files after the build command completes. If your build outputs to a custom directory, ensure the files are discoverable from the workspace root.
