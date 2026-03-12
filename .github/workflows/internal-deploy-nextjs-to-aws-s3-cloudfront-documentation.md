<!-- markdownlint-disable -->
# Internal Next.js Static Site Build & AWS S3 Deployment

> **Internal workflow** maintained by the OnBoard Crew App team. It contains logic that
> depends on our self-hosted runners and caching strategy. Use it as a reference when
> building your own workflows rather than invoking it directly from external projects.

This workflow compiles a Next.js application into a static site, caches dependencies and
build artifacts to speed up successive runs, deploys the resulting files to an AWS S3 bucket,
and invalidates a CloudFront distribution. It also supports loading environment variables
from the AWS SSM Parameter Store and sending Slack notifications to track progress.

## Key Features

- Node.js environment setup with version input
- Intelligent dependency caching based on lockfile checksums
- Optional `.env` generation from SSM parameters (prefix, custom filename/directory)
- Build and deploy commands that accept custom scripts with retry loops
- Pre‑ and post‑deployment CloudFront invalidation with retries
- Detailed Slack messages for start, success, and failure
- Build caches stored on the self-hosted runner filesystem for faster incremental builds

Because the caching path is hard-coded to `/tmp/bifrost-cache` and the job uses
`onpremise` or other self-hosted runners, this workflow is unsuitable for GitHub-hosted
runners. Instead, use it as a blueprint for workflows running in similar environments.

## Required Inputs

| Input | Description |
|-------|-------------|
| `repository_name` | Repository name used in logs and cache paths. |
| `node_version` | Node.js version to install (e.g. `16.x`). |
| `aws_account_id` | AWS account ID (for SSM query context). |
| `aws_region` | AWS region for SSM, S3, and CloudFront (default `ap-southeast-1`). |
| `aws_ssm_generate_env_file_enabled` | Boolean to turn on/off `.env` generation. |
| `aws_s3_bucket` | Target S3 bucket for the static site. |
| `aws_cloudfront_distribution_id` | CloudFront distribution ID to invalidate. |
| `slack_notify_enabled` | Enable Slack notifications (`true`/`false`). |

## Optional Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `node_package_manager` | `auto`, `yarn`, or `npm` (auto-detects lockfile). | `auto` |
| `node_frozen_lockfile` | Whether to use frozen lockfile/`npm ci`. | `true` |
| `node_env` | Value for `NODE_ENV` during build. | `production` |
| `runner` | Runner label for the job. | `ubuntu-latest` |
| `aws_ssm_parameter_prefix` | SSM prefix path for env vars. | - |
| `aws_ssm_parameter_prefix_custom_env_filename` | Filename for generated env file. | `.env` |
| `aws_ssm_parameter_prefix_env_directory` | Directory for generated env file. | `.` |
| `slack_channel_id` | Slack channel ID (if notifications enabled). | - |
| `build_command` | Custom shell command for building site. | Next.js default build (`npm run build && npm run export`) |
| `deploy_command` | Custom deploy command. | Sync `./out` to S3 and invalidate CloudFront |
| `cache_base_directory` | Base path for caches. | `/tmp/bifrost-cache` |

## Required Secrets

| Secret | Description |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | AWS credentials for SSM, S3, CloudFront. |
| `AWS_SECRET_ACCESS_KEY` | Corresponding secret key. |
| `SLACK_BOT_TOKEN` | Slack bot token if notifications are used. |
| `FIREBASE_TOKEN` | Optional token used by custom build commands. |
| `CERT_PASSWORD` | Optional password used by custom build commands. |

## Examples

### Basic invocation (no Slack, no custom commands)

```yaml
jobs:
  build:
    uses: your-org/your-repo/.github/workflows/internal-deploy-nextjs-to-aws-s3-cloudfront.yml@main
    with:
      repository_name: "crew-web"
      node_version: "18.x"
      aws_account_id: "123456789012"
      aws_region: "ap-southeast-1"
      aws_ssm_generate_env_file_enabled: false
      aws_s3_bucket: "crew-web-static"
      aws_cloudfront_distribution_id: "E2EXAMPLE12345"
      slack_notify_enabled: false
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

### Full-featured run with caching, env file and Slack

```yaml
jobs:
  build:
    uses: your-org/your-repo/.github/workflows/internal-deploy-nextjs-to-aws-s3-cloudfront.yml@main
    with:
      repository_name: "crew-web"
      node_version: "18.x"
      aws_account_id: "123456789012"
      aws_region: "us-east-1"
      aws_ssm_generate_env_file_enabled: true
      aws_ssm_parameter_prefix: "/crew-web/${{ github.ref_name }}"
      aws_ssm_parameter_prefix_custom_env_filename: ".env.local"
      aws_ssm_parameter_prefix_env_directory: "app"
      aws_s3_bucket: "crew-web-${{ github.ref_name }}"
      aws_cloudfront_distribution_id: "E3EXAMPLE67890"
      slack_notify_enabled: true
      slack_channel_id: "C1234567890"
      build_command: |
        npm install && npm run lint && npm run build && npm run export
      deploy_command: |
        aws s3 sync ./out s3://$AWS_S3_BUCKET/ --delete --cache-control "public, max-age=31536000, immutable"
        aws s3 cp ./out s3://$AWS_S3_BUCKET/ --recursive --exclude "*" --include "*.html" \
          --cache-control "public, max-age=0, must-revalidate" --content-type "text/html; charset=utf-8"
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
      FIREBASE_TOKEN: ${{ secrets.FIREBASE_TOKEN }}
      CERT_PASSWORD: ${{ secrets.CERT_PASSWORD }}
```

### Custom runner and caching directory

```yaml
jobs:
  build:
    uses: your-org/your-repo/.github/workflows/internal-deploy-nextjs-to-aws-s3-cloudfront.yml@main
    with:
      repository_name: "crew-web"
      node_version: "18.x"
      runner: "self-hosted-linux-cache"
      cache_base_directory: "/var/cache/bifrost"
      # ...other inputs...
```

## Best Practices

1. **Run on self-hosted runners** – the workflow's caching strategy depends on a writable
   filesystem that persists between jobs. GitHub-hosted runners are ephemeral and therefore
   not suitable.
2. **Use lockfile hashes for cache keys** – the `build cache` logic already handles this; keep
   your lockfiles committed to benefit from speedups.
3. **Separate environment prefixes by branch** – using `${{ github.ref_name }}` in your SSM
   paths helps avoid leaking staging variables into production builds.
4. **Leverage custom commands for complex builds** – you can run tests, linting, or other
   preparatory steps in `build_command` before the static export.
5. **Invalidate CloudFront twice when needed** – the workflow handles pre‑ and post‑deployment
   invalidations with retry loops to ensure caches are fresh.

## Troubleshooting

- **Caches not used**: Verify that the `cache_base_directory` is writable and not cleaned between
  runs by other processes. Also confirm that the hash outputs from the `Generate Cache Keys`
  step change when dependencies change.
- **SSM env file missing or empty**: Make sure `aws_ssm_parameter_prefix` is correct and accessible
  using the same AWS credentials configured in the job.
- **Build timeouts or failures**: The build step retries up to three times; inspect the logs for
  the underlying error. Consider adding additional diagnostics in a custom `build_command`.
- **Deploy fails or no `out` directory**: Ensure the previous build step succeeded and produced an
  `out/` folder. The deploy logic exits with an error message if the directory is absent.
- **Slack messages don't appear**: Check the bot token scopes and confirm `slack_notify_enabled`
  is `true` with a valid channel ID.

## Internal Notes

This workflow is optimized for our infrastructure and may reference environment variables or
tools not available outside the OnBoard Crew App ecosystem. Reuse with caution and feel free to
clone it into your own repository where you can modify caching paths, runner labels, or other
defaults without impacting the central copy.
