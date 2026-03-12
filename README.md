<!-- markdownlint-disable -->
# Workflows
Collections of GitHub Actions Workflow (reusable).

## Available Workflows

- [build-docker.yml](.github/workflows/build-docker-documentation.md) – Build and push Docker images with AWS ECR, SSM helpers, and optional Slack notifications.
- [internal-argocd-app-sync.yml](.github/workflows/internal-argocd-app-sync.md) – **Internal** workflow for syncing ArgoCD applications; tailored for the OnBoard Crew App team and provided as a reference only.
- [ansible-playbook-extravars.yml](.github/workflows/ansible-playbook-extravars.md) – Run an Ansible playbook with optional SSM‑provided SSH key, Vault password, and extra vars, plus Slack notifications.
- [debezium-connectors-operations.yml](.github/workflows/debezium-connectors-operations.md) – Pause or resume a Debezium connector via its REST API using a reusable workflow call.
