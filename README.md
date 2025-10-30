# Deploy your microservice application using Harness CD

This repository gives a quick overview about deploying a microservice (Guestbook UI) to Dev, QA, and Prod Kubernetes environments using a Harness CD pipeline with a Production approval gate.

Harness project: `default_project` in your trial account (`https://app.harness.io/ng/account/exqRN-uBSkKCxM4tS2ZLBg/all/orgs/default/projects/default_project/overview`).

Reference followed: `https://developer.harness.io/docs/continuous-delivery/get-started/cd-tutorials/ownapp?interface=ui`

## What’s in this repo

- `guestbook/`
  - `guestbook-ui-deployment.yaml`, `guestbook-ui-svc.yaml` — Kubernetes manifests deployed by the pipeline
- `deploy-own-app/cd-pipeline/`
  - `service.yml` — Harness service pointing to manifests in this repo
  - `environment.yml` — Development environment
  - `environment-qa.yml` — QA environment
  - `environment-prod.yml` — Production environment
  - `infrastructure-definition.yml` — K8s infrastructure for Development
  - `infrastructure-definition-qa.yml` — K8s infrastructure for QA
  - `infrastructure-definition-prod.yml` — K8s infrastructure for Production
  - `kubernetes-connector.yml` — K8s connector (inherit from delegate)
  - `github-connector.yml` — GitHub connector (account-level)
  - `ownapp-multi-env-pipeline.yml` — Multi-stage pipeline (Dev → QA → Approval → Prod)

## 1) Pipeline design

- Stages: Dev (Rolling) → QA (Rolling) → Approval (HarnessApproval) → Prod (Rolling)
- Service: `ownappservice` (Kubernetes) deploying `guestbook` manifests from this repo
- Environments: `ownappdevenv` (Dev), `ownappqaenv` (QA), `ownappprodenv` (Prod)
- Approval: requires a manual gate before Prod deployment

```yaml
pipeline:
  name: ownapp_multi_env_pipeline
  identifier: ownapp_multi_env_pipeline
  projectIdentifier: default_project
  orgIdentifier: default
  tags: {}
  stages:
    - stage:
        name: Deploy to Dev
        identifier: deploy_to_dev
        description: ""
        type: Deployment
        spec:
          deploymentType: Kubernetes
          service:
            serviceRef: ownappservice
          environment:
            environmentRef: ownappdevenv
            deployToAll: false
            infrastructureDefinitions:
              - identifier: ownappk8sinfra
          execution:
            steps:
              - step:
                  name: Rollout Deployment
                  identifier: rolloutDeployment
                  type: K8sRollingDeploy
                  timeout: 10m
                  spec:
                    skipDryRun: false
                    pruningEnabled: false
            rollbackSteps:
              - step:
                  name: Rollback Rollout Deployment
                  identifier: rollbackRolloutDeployment
                  type: K8sRollingRollback
                  timeout: 10m
                  spec:
                    pruningEnabled: false
        tags: {}
        failureStrategies:
          - onFailure:
              errors:
                - AllErrors
              action:
                type: StageRollback
    - stage:
        name: Deploy to QA
        identifier: deploy_to_qa
        description: ""
        type: Deployment
        spec:
          deploymentType: Kubernetes
          service:
            serviceRef: ownappservice
          environment:
            environmentRef: ownappqaenv
            deployToAll: false
            infrastructureDefinitions:
              - identifier: ownappk8sinfraqa
          execution:
            steps:
              - step:
                  name: Rollout Deployment
                  identifier: rolloutDeployment
                  type: K8sRollingDeploy
                  timeout: 10m
                  spec:
                    skipDryRun: false
                    pruningEnabled: false
            rollbackSteps:
              - step:
                  name: Rollback Rollout Deployment
                  identifier: rollbackRolloutDeployment
                  type: K8sRollingRollback
                  timeout: 10m
                  spec:
                    pruningEnabled: false
        tags: {}
        failureStrategies:
          - onFailure:
              errors:
                - AllErrors
              action:
                type: StageRollback
    - stage:
        name: Approve Prod
        identifier: approve_prod
        description: "Production gate approval"
        type: Approval
        spec:
          execution:
            steps:
              - step:
                  type: HarnessApproval
                  name: Prod Approval
                  identifier: prodApproval
                  timeout: 1d
                  spec:
                    approvalMessage: |-
                      Please review the deployment to Prod and approve.
                    includePipelineExecutionHistory: true
                    approvers:
                      minimumCount: 1
                      disallowPipelineExecutor: false
                      userGroups:
                        - account._account_all_users
                    approverInputs: []
    - stage:
        name: Deploy to Prod
        identifier: deploy_to_prod
        description: ""
        type: Deployment
        spec:
          deploymentType: Kubernetes
          service:
            serviceRef: ownappservice
          environment:
            environmentRef: ownappprodenv
            deployToAll: false
            infrastructureDefinitions:
              - identifier: ownappk8sinfraprod
          execution:
            steps:
              - step:
                  name: Rollout Deployment
                  identifier: rolloutDeployment
                  type: K8sRollingDeploy
                  timeout: 10m
                  spec:
                    skipDryRun: false
                    pruningEnabled: false
            rollbackSteps:
              - step:
                  name: Rollback Rollout Deployment
                  identifier: rollbackRolloutDeployment
                  type: K8sRollingRollback
                  timeout: 10m
                  spec:
                    pruningEnabled: false
        tags: {}
        failureStrategies:
          - onFailure:
              errors:
                - AllErrors
              action:
                type: StageRollback
```

## 2) Pipeline execution

Run the pipeline in Harness. Execution should show:
- Dev stage succeeds → QA stage succeeds → Approval waits → once approved → Prod stage succeeds

## 3) Verify the deployment

Run the following commands for each environment.

### 1. Development Namespace (`dev-ns`)

| Command | 
| :--- | 
| `kubectl get pods -n dev-ns` |
| `kubectl get svc -n dev-ns` |

---

### 2. QA Namespace (`qa-ns`)

| Command | 
| :--- |
| `kubectl get pods -n qa-ns` | 
| `kubectl get svc -n qa-ns` | 

---

### 3. Production Namespace (`prod-ns`)

| Command |
| :--- |
| `kubectl get pods -n prod-ns` |
| `kubectl get svc -n prod-ns` | 

---

## 4) Deployment strategy (what and why)

I recommend starting with the **Rolling Deployment** strategy. This is the simplest strategy for a standard Kubernetes Deployment object.

### How and Why:

| Aspect | Details |
| :--- | :--- |
| **How It's Implemented** | This strategy is implemented **by default** when deploying a standard K8S `Deployment` manifest. |
| **Why It's Chosen for this Exercise** | **Rolling Deployment** is the easiest to start with, requires no additional infrastructure (unlike Blue/Green), and offers a good balance of speed and safety for non-critical changes. |
| **Real-World Context** | For a real-world critical application, one might choose **Canary** (for finer-grained traffic splitting and automated verification) or **Blue/Green** (for near-zero downtime and instant rollback). The Rolling strategy is suitable for non-mission-critical updates where a gradual rollout is acceptable. |

## 5) Automating Harness setup

The process of creating Harness entities (Connectors, Services, Environments, Pipelines) should be automated using **Configuration as Code (CaC)** to ensure consistency, version control, and auditability across all environments.

### Primary Methods for Automating Harness Entity Creation

#### 1. Harness Git Experience (Recommended for Pipelines)

This is the preferred GitOps approach built directly into Harness.

* **How:** Store your Pipeline and entity YAML files (e.g., Service, Environment, Infrastructure Definition) directly in your Git repository. Harness is configured to read from and sync these entities automatically upon committing changes to Git.
* **Why:** Ensures that the source of truth is always Git. Changes are version-controlled, easily auditable, and deployments are consistent.

---

#### 2. Terraform/OpenTofu Provider (Recommended for Infrastructure)

This method integrates Harness management into existing Infrastructure as Code (IaC) workflows.

* **How:** Use the dedicated **Harness Terraform provider** (or OpenTofu, a fork of Terraform) to manage entities. Entities are defined using HashiCorp Configuration Language (HCL).
* **Why:** Excellent for provisioning cross-module entities like **Connectors** and **Projects**, as well as complex Pipelines. It allows leveraging standard IaC practices and state management.

---

#### 3. Harness API

For advanced or highly customized automation, direct API interaction provides maximum flexibility and access to the platform's core functionalities.

* **How:** Directly use the Harness API (GraphQL and REST) for creation and management, typically accomplished by scripting with tools like Python or custom CLI wrappers.
* **Interaction:** The automation scripts interact directly with the Harness API endpoints for managing and distributing all context, configuration, and state changes within the Harness platform.
* **Why:** Provides granular control over the entity creation process. Advanced scripting can be used for bulk operations, complex dependency management, or integrating with specialized internal tools by directly leveraging the platform's API layer.

## 6) Doc feedback

The following suggestions are aimed at improving the clarity, flow, and accessibility of the tutorial, particularly for users setting up a local development environment.

### 1. Clarity on Prerequisites

The tutorial could be significantly improved by having a more explicit **"Prerequisites"** section at the very top. This section should clearly outline the following requirements *before* starting the main deployment steps:

* **A live Kubernetes cluster:** (e.g., local Minikube, K3D, or a cloud-based cluster).
* **A publicly accessible Docker image** (the application artifact to be deployed).
* **A Git repository** (containing application manifests or for use as the Configuration as Code source).

---

### 2. K3D/Local Cluster Setup

When you want to try a fast local setup, including a dedicated section on local cluster installation would significantly lower the barrier to entry.

* **Recommendation:** Include a brief, high-level guide (or a link to a one-click setup script) for creating a local **K3D** or **Minikube** cluster. This should cover the minimum steps needed to get a functioning Kubernetes context ready for the Harness Delegate installation.
