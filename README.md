<!-- markdownlint-disable -->
# Workflows
Collections of GitHub Actions Workflow (reusable).

## Available Workflows

- [build-docker.yml](.github/workflows/build-docker-documentation.md) – Build and push Docker images with AWS ECR, SSM helpers, and optional Slack notifications.
- [ansible-playbook-extravars.yml](.github/workflows/ansible-playbook-extravars-documentation.md) – Run an Ansible playbook with optional SSM‑provided SSH key, Vault password, and extra vars, plus Slack notifications.
- [debezium-connectors-operations.yml](.github/workflows/debezium-connectors-operations-documentation.md) – Pause or resume a Debezium connector via its REST API using a reusable workflow call.

## Internal Workflows

These workflows are specifically tailored to our internal infrastructure and runner environment. They may include assumptions or caching strategies that are not suitable for public reuse, but they serve as useful reference examples when building similar workflows.

- [internal-argocd-app-sync.yml](.github/workflows/internal-argocd-app-sync-documentation.md) – **Internal** workflow for syncing ArgoCD applications; tailored for the OnBoard Crew App team and provided as a reference only.
- [internal-deploy-nextjs-to-aws-s3-cloudfront.yml](.github/workflows/internal-deploy-nextjs-to-aws-s3-cloudfront-documentation.md) – **Internal** Next.js static site build and S3/CloudFront deployment with caching and Slack support.
