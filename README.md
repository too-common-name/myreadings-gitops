# myreadings-gitops

GitOps repository for deploying the MyReadings application on OpenShift using Argo CD and ACM.

MyReadings is a Quarkus-based modular monolith being incrementally decomposed into microservices. The first extracted service is the user-service. This repo manages the full deployment lifecycle: operator installation via ACM policies, infrastructure provisioning (PostgreSQL, Keycloak, RabbitMQ), post-deploy configuration via Ansible K8s Jobs, application workloads, and CI pipelines.

## Repository structure

```
.
├── bootstrap.yaml                  # App-of-apps entry point — apply this first
├── argocd/
│   ├── kustomization.yaml          # Orchestrates sync-waves for all ArgoCD resources
│   ├── projects/
│   │   ├── myreadings-platform-project.yaml   # AppProject for ACM policies
│   │   └── myreadings-project.yaml            # AppProject for application workloads
│   ├── placements/
│   │   ├── acm-argocd-integration.yaml        # GitOpsCluster + ManagedClusterSetBinding
│   │   ├── dev-placement.yaml                 # Targets clusters labeled environment=dev
│   │   └── prod-placement.yaml                # Targets clusters labeled environment=prod
│   ├── acm-policies-app.yaml       # Argo CD Application for ACM policies
│   ├── myreadings-appset.yaml      # ApplicationSet using ACM clusterDecisionResource
│   └── myreadings-tekton-app.yaml  # Argo CD Application for Tekton CI resources
├── acm-policies/
│   └── myreadings-operators.yaml   # ACM Policy installing operators on labeled clusters
├── base/                           # Shared manifests (Kustomize base)
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── postgres/postgrescluster.yaml
│   ├── keycloak/keycloak.yaml
│   ├── rabbitmq/rabbitmqcluster.yaml
│   ├── config-jobs/                # Ansible K8s Jobs for post-deploy configuration
│   │   ├── rbac.yaml               #   ServiceAccount + Role for reading Secrets
│   │   ├── configure-keycloak.yaml  #   Job: realm, client, roles
│   │   └── configure-rabbitmq.yaml  #   Job: vhost, user, queue, binding
│   ├── monolith/
│   │   ├── configmap.yaml, deployment.yaml, service.yaml, route.yaml
│   └── ui/
│       ├── configmap-nginx.yaml, deployment.yaml, service.yaml, route.yaml
├── overlays/
│   └── dev/
│       ├── kustomization.yaml      # Namespace, patches, image overrides (updated by CI)
│       └── patches/
├── ansible/                        # Ansible playbooks + roles baked into EE image
│   ├── Containerfile               # Custom EE image build
│   ├── requirements.yml
│   ├── configure-keycloak.yml
│   ├── configure-rabbitmq.yml
│   └── roles/
│       ├── ocp_keycloak_config/
│       └── ocp_rabbitmq_config/
└── tekton/                         # CI pipeline, tasks, RBAC
    ├── kustomization.yaml
    ├── rbac.yaml                   # ServiceAccount + image-builder RoleBinding
    ├── task-maven-test.yaml        # Maven test task (unit + integration)
    ├── task-acs-image-scan.yaml    # ACS image scan (graceful if not configured)
    └── pipeline.yaml               # Reusable parameterized pipeline
```

## Prerequisites

These must be installed and configured on the cluster before deploying MyReadings:

| Component | Notes |
|-----------|-------|
| **OpenShift** | Tested on OCP 4.x |
| **Red Hat ACM** | Hub cluster with managed clusters in the `default` ClusterSet |
| **OpenShift GitOps** (Argo CD) | Installed cluster-wide |
| **OpenShift Pipelines** (Tekton) | Installed via ACM policy (included in this repo) |

### Manual RBAC setup (one-time, on the hub)

The `openshift-gitops-applicationset-controller` ServiceAccount needs `cluster-admin` to read ACM `PlacementDecision` resources:

```bash
oc adm policy add-cluster-role-to-user cluster-admin \
  system:serviceaccount:openshift-gitops:openshift-gitops-applicationset-controller
```

### Secrets (created manually, not in Git)

**GitHub token** — needed by the Tekton pipeline to push image tag updates to this gitops repo. Create a fine-grained PAT with `Contents: Read and Write` on `myreadings-gitops` only:

```bash
oc create secret generic github-token \
  --from-literal=username=<github-username> \
  --from-literal=password=<fine-grained-PAT> \
  --type=kubernetes.io/basic-auth \
  -n myreadings-dev

oc annotate secret github-token \
  "tekton.dev/git-0=https://github.com" \
  -n myreadings-dev
```

**ACS API token** (optional) — needed for image security scanning. If not present, the scan step is skipped gracefully:

```bash
oc create secret generic acs-api-token \
  --from-literal=token=<acs-api-token> \
  -n myreadings-dev
```

### Cluster labels

Managed clusters need these labels:

- `myreadings: "true"` — registers the cluster in Argo CD and triggers operator installation via ACM policies
- `environment: dev` or `environment: prod` — determines which Kustomize overlay the ApplicationSet deploys

## How it works

### Operator installation (ACM-driven)

The ACM policy installs operators on any managed cluster labeled `myreadings: "true"`:

- CrunchyData Postgres Operator (`certified-operators`, channel `v5`)
- Red Hat Build of Keycloak (`redhat-operators`, channel `stable-v26.4`)
- OpenShift Pipelines (`redhat-operators`, channel `latest`)
- RabbitMQ Cluster Operator (`community-operators`, channel `stable`)

### Application deployment (Argo CD + ACM)

A single `bootstrap.yaml` creates an app-of-apps that sets up everything through sync-waves:

1. **wave -4**: ACM-Argo CD integration (GitOpsCluster, ManagedClusterSetBinding, Placement for clusters with `myreadings: "true"`)
2. **wave -2**: AppProjects (`myreadings-platform` for policies, `myreadings` for workloads)
3. **wave -1**: Environment-specific Placements (dev, prod)
4. **wave 0**: ACM policies Application, ApplicationSet for workloads, Tekton CI Application

The ApplicationSet uses the `clusterDecisionResource` generator to discover clusters from ACM PlacementDecisions.

### Application sync-waves

Within each application deployment, resources are ordered by sync-waves:

| Wave | Resources | Purpose |
|------|-----------|---------|
| 0 (default) | Namespace, SA, RBAC, ConfigMaps, Services, Routes | Foundation |
| 1 | PostgresCluster, Keycloak, RabbitmqCluster | Infrastructure (waits for healthy) |
| 2 | K8s Jobs: configure-keycloak, configure-rabbitmq | Post-deploy config (must complete) |
| 3 | Deployments: myreadings-app, myreadings-ui | Application (after config is done) |

### Post-deploy configuration (Ansible K8s Jobs)

Keycloak and RabbitMQ configuration runs as Kubernetes Jobs using the custom Ansible Execution Environment image (`quay.io/rh-ee-drossi/myreadings-ee:latest`). The playbooks and roles are baked into the image, mirroring the same pattern used locally with Docker Compose (`myreadings_deploy/ansible/`).

### CI (Tekton)

A single reusable Pipeline (`myreadings-build`) handles all components:

```
clone -> unit tests -> build+push -> integration tests  \
                                  -> image scan (ACS)    -> update gitops
```

- **Unit tests** fail fast before any image is built
- **Integration tests** and **image scan** run in parallel after the build, both gating promotion
- **Update gitops** patches the Kustomize overlay with the new image SHA, triggering Argo CD deployment
- Images are tagged with the commit SHA (no `:latest`)

## Getting started

### 1. Label your managed cluster

```bash
oc label managedcluster <cluster-name> myreadings="true" environment="dev"
```

### 2. Apply the bootstrap

```bash
oc apply -f bootstrap.yaml
```

### 3. Create required secrets

See the Secrets section in Prerequisites above.

### 4. Build container images

Trigger a Tekton PipelineRun for each component. Example for the monolith:

```bash
oc create -f tekton/pipelineruns/run-app.yaml -n myreadings-dev
```

## Container images

| Component | Base image ref | Built by |
|-----------|---------------|----------|
| Monolith | `quay.io/too-common-name/myreadings-app` | Tekton pipeline |
| UI | `quay.io/too-common-name/myreadings-ui` | Tekton pipeline |
| Keycloak (custom) | `quay.io/rh-ee-drossi/myreadings-keycloak:latest` | Manual build |
| Ansible EE | `quay.io/rh-ee-drossi/myreadings-ee:latest` | Manual build |

The Tekton pipeline pushes images to the internal registry with commit SHA tags. The Kustomize overlay maps the base image refs to the internal registry images.
