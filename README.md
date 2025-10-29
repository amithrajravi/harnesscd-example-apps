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

## 3) Verifying the running pod

After Dev/QA/Prod deploys, verify on the cluster:

```bash
kubectl get deploy,rs,po,svc -n <dev|qa|prod-namespace>
kubectl describe deploy <guestbook-deployment> -n <namespace>
kubectl logs deploy/<guestbook-deployment> -n <namespace> --tail=100
```

## 4) Deployment strategy and rationale

- Strategy: Kubernetes Rolling update across all environments.
- Why: Simple, low‑risk for stateless services, leverages K8s readiness probes, easy rollback to prior ReplicaSet, and aligns with the referenced tutorial baseline.
- Alternatives available (included previously for reference but removed for focus): Canary / Blue‑Green.

## 5) How to automate Harness entities

- GitOps via YAML (this repo): store Services, Environments, Infrastructures, Connectors, and Pipelines as YAML and import/sync in Harness.
- Harness APIs: create/update entities programmatically.
- Terraform Provider for Harness: manage entities declaratively and promote across orgs/projects.
- Pipelines as Code: keep pipelines remote in Git with PR workflows.

## 6) Documentation feedback

- Provide a full multi‑environment example with an explicit Production approval stage.
- Call out common connector scopes and `connectorRef` formats (account./org./project).
- Add troubleshooting for delegate‑to‑cluster connectivity (localhost:8080), namespace RBAC, and branch/repo mismatches.

## Setup notes (pre‑flight)

- Ensure your Docker or K8s Delegate is connected and has access to the cluster.
- Update `deploy-own-app/cd-pipeline/kubernetes-connector.yml` delegate selector to match your delegate.
- Ensure GitHub PAT secret (`ownappgitpat`) exists; update `github-connector.yml` username/account URL if needed.
- Adjust namespaces in infra definitions: Dev default `harness-delegate-ng`, QA `qa-ns`, Prod `prod-ns` (edit as needed).

