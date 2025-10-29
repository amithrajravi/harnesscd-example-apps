# Exercise for DevRel Role — Multi‑Env Harness CD Pipeline

This repository is trimmed to only the artifacts required to deploy a microservice (Guestbook UI) to Dev, QA, and Prod Kubernetes environments using a Harness pipeline with a Production approval gate.

Harness project: `default_project` in your trial account (`https://app.harness.io/ng/account/exqRN-uBSkKCxM4tS2ZLBg/all/orgs/default/projects/default_project/overview`).

Reference followed: `https://developer.harness.io/docs/continuous-delivery/get-started/cd-tutorials/ownapp?interface=ui`

## What’s in this repo

- `guestbook/`
  - `guestbook-ui-deployment.yaml`, `guestbook-ui-svc.yaml` — Kubernetes manifests deployed by the pipeline
- `deploy-own-app/cd-pipeline/`
  - `service.yml` — Harness Service pointing to manifests in this repo
  - `environment.yml` — Dev environment (PreProduction)
  - `environment-qa.yml` — QA environment (PreProduction)
  - `environment-prod.yml` — Prod environment (Production)
  - `infrastructure-definition*.yml` — K8s infrastructure for Dev/QA/Prod
  - `kubernetes-connector.yml` — K8s connector (inherit from delegate)
  - `github-connector.yml` — GitHub connector (account-level)
  - `ownapp-multi-env-pipeline.yml` — Multi-stage pipeline (Dev → QA → Approval → Prod)

## 1) Pipeline design

- Stages: Dev (Rolling) → QA (Rolling) → Approval (HarnessApproval) → Prod (Rolling)
- Service: `ownappservice` (Kubernetes) deploying `guestbook` manifests from this repo
- Environments: `ownappdevenv` (Dev), `ownappqaenv` (QA), `ownappprodenv` (Prod)
- Infra: per-environment K8s Direct with `connectorRef: ownappk8sconnector`
- Approval: requires a manual gate before Prod deployment

## 2) Pipeline execution (what to expect)

Run the pipeline in Harness. Execution should show:
- Dev stage succeeds → QA stage succeeds → Approval waits → once approved → Prod stage succeeds

## 3) Verify the deployment (simple checks)

Run these commands against the namespace used by each stage (Dev/QA/Prod):

```bash
# What’s running and exposed
kubectl get deploy,rs,po,svc -n <dev|qa|prod-namespace>

# Health and rollout details
kubectl describe deploy <guestbook-deployment> -n <namespace>

# Recent app logs
kubectl logs deploy/<guestbook-deployment> -n <namespace> --tail=100
```

## 4) Deployment strategy (what and why)

- Strategy: Kubernetes Rolling update in all environments.
- Why this strategy:
  - Keeps the app available during updates (pods roll gradually)
  - Uses native readiness checks for safe rollout
  - Easy rollback to the previous ReplicaSet if needed
- Notes: Canary and Blue‑Green are valid alternatives but were omitted here to keep the exercise focused.

## 5) Automating Harness setup (your options)

- GitOps with YAML (recommended here): store Services, Environments, Infrastructures, Connectors, and Pipelines in Git (this repo) and import/sync in Harness.
- Harness APIs: create/update entities via REST.
- Terraform Provider for Harness: manage entities declaratively and promote across orgs/projects.
- Pipelines as Code: keep pipelines remote in Git with PR reviews.

## 6) Doc feedback (how it could be clearer)

- Include a complete Dev→QA→Prod example with an explicit Prod approval gate.
- Clarify connector scopes and how to reference them (`account.`, `org.`, or project‑level `connectorRef`).
- Add troubleshooting for common issues: delegate→cluster connectivity (localhost:8080), namespace RBAC, and wrong repo/branch.

## Setup notes (before you run it)

- Make sure your Delegate is connected and can reach the Kubernetes API.
- In `deploy-own-app/cd-pipeline/kubernetes-connector.yml`, set the delegate selector to match your delegate.
- Ensure a GitHub PAT secret exists (e.g., `ownappgitpat`) and that `github-connector.yml` points to your account/repo/branch.
- Namespaces in infra definitions: Dev `dev-ns`, QA `qa-ns`, Prod `prod-ns` — change if your cluster uses different ones.

