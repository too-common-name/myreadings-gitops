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
│   ├── keycloak/
│   │   ├── keycloak.yaml              #   Keycloak CR (RHBK operator)
│   │   └── route.yaml                 #   Explicit Route (operator Ingress lacks host)
│   ├── monolith/
│   │   ├── configmap.yaml, deployment.yaml, service.yaml
│   └── ui/
│       ├── deployment.yaml, service.yaml, route.yaml
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
    ├── task-maven-test.yaml        # Maven test/lint (no Docker needed)
    ├── task-maven-test-dind.yaml   # Maven test with DinD sidecar (Testcontainers)
    ├── task-acs-image-scan.yaml    # ACS image scan (graceful skip if not configured)
    ├── pipeline.yaml               # Reusable parameterized pipeline
    └── pipelineruns/
        ├── run-app.yaml            # PipelineRun for the monolith
        └── run-ui.yaml             # PipelineRun for the UI (uses Dockerfile.ocp)
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

**GitHub token** — needed by the Tekton pipeline to push image tag updates to this gitops repo. Create a fine-grained PAT with `Contents: Read and Write` on `myreadings-gitops` only.

The `git-cli` ClusterTask expects `.git-credentials` and `.gitconfig` keys:

```bash
oc create secret generic github-token \
  --from-literal=".git-credentials=https://<github-username>:<fine-grained-PAT>@github.com" \
  --from-literal=".gitconfig=[credential \"https://github.com\"]
  helper = store" \
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

A single reusable Pipeline (`myreadings-build`) handles all components. The pipeline has seven stages:

```
clone ─┬─ lint (checkstyle)  ─┬─ build+push ─┬─ integration tests ─┬─ update gitops
       └─ unit tests ─────────┘              └─ image scan (ACS)   ─┘
```

- **Lint** runs `checkstyle:check` against the project's `checkstyle.xml` (Sun conventions). Currently non-blocking (`failOnViolation=false`) due to pre-existing violations; switch to `true` to enforce
- **Unit tests** run in parallel with lint for fast feedback. Pure Mockito/JUnit tests only — no Docker required
- **Build** uses `buildah` with the app's multi-stage Dockerfile, pushes to the OpenShift internal registry tagged with the commit SHA
- **Integration tests** and **image scan** run in parallel after the build, both gating the promotion step
- **Update gitops** clones this repo, appends the new image reference to the Kustomize overlay's `images` section, commits and pushes — which triggers Argo CD to deploy

Images are always tagged with the full commit SHA (no `:latest`).

#### Testcontainers / DinD limitation

Tests that use `@QuarkusTest` with Testcontainers (JPA repository tests, controller integration tests) are excluded from the pipeline because they need a Docker daemon. A `maven-test-dind` task with a Docker-in-Docker sidecar is included and ready, but OpenShift Pipelines globally enforces `pipelines-scc` via `TektonConfig`, which blocks privileged containers. To enable DinD tests, change the SCC in TektonConfig:

```yaml
# oc edit tektonconfig config
spec:
  platforms:
    openshift:
      scc:
        default: privileged   # was: pipelines-scc
```

Until then, Testcontainers-based tests should be run locally or in an environment with Docker available.

## Cluster-specific configuration

The `overlays/dev/` directory contains values that depend on the target cluster. When deploying to a new cluster, update these:

### Keycloak hostname

The Keycloak OIDC issuer must match the external Route URL so that tokens issued via the browser are accepted by the backend. The hostname follows the OCP pattern `{route-name}-{namespace}.apps.{cluster-domain}`.

Update `overlays/dev/patches/keycloak-hostname.yaml`:

```yaml
spec:
  hostname:
    hostname: myreadings-keycloak-myreadings-dev.apps.<your-cluster-domain>
```

This is the only value that needs changing per cluster. The UI server derives all other URLs dynamically at runtime from the incoming request's `Host` header.

### UI architecture (Node server, no nginx)

On OCP the UI runs a Node.js Express server (`Dockerfile.ocp` + `server.js`) instead of nginx:

- **API proxy**: `/api/*` requests are proxied to backend services via internal K8s DNS -- no backend Route needed, no CORS
- **Dynamic config**: `GET /config.js` is generated at runtime by deriving the Keycloak Route URL from the request's `Host` header and the pod's `NAMESPACE` env var (Kubernetes Downward API)
- **SPA fallback**: all other requests serve the Vue app's `index.html`

For local development (`docker-compose`), the original `Dockerfile` with nginx is used unchanged.

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

Trigger a Tekton PipelineRun for each component:

```bash
oc create -f tekton/pipelineruns/run-app.yaml -n myreadings-dev
oc create -f tekton/pipelineruns/run-ui.yaml -n myreadings-dev
```

The pipeline will build, test, scan, and push the image, then automatically update the Kustomize overlay to trigger an Argo CD deployment. The UI pipeline uses `Dockerfile.ocp` (Node server) and skips Maven tests.

## Container images

| Component | Base image ref | Built by |
|-----------|---------------|----------|
| Monolith | `quay.io/too-common-name/myreadings-app` | Tekton pipeline |
| UI | `quay.io/too-common-name/myreadings-ui` | Tekton pipeline |
| Keycloak (custom) | `quay.io/rh-ee-drossi/myreadings-keycloak:latest` | Manual build |
| Ansible EE | `quay.io/rh-ee-drossi/myreadings-ee:latest` | Manual build |

The Tekton pipeline pushes images to the internal registry with commit SHA tags. The Kustomize overlay maps the base image refs to the internal registry images.

## Known limitations and tech debt

- **Testcontainers in CI**: JPA and controller integration tests are excluded from the pipeline due to the `pipelines-scc` restriction. The `maven-test-dind` task is ready for when the cluster allows privileged containers.
- **Checkstyle violations**: The project has pre-existing checkstyle violations. The lint step currently reports but does not fail the build. Flip `failOnViolation` to `true` once violations are cleaned up.
- **Dockerfiles use fully qualified image names**: Required by buildah on OpenShift, which enforces short-name resolution and cannot prompt for a registry without a TTY (e.g. `docker.io/library/maven:3.9-eclipse-temurin-21`).
- **Keycloak hostname is per-cluster**: The OIDC issuer must match the external Route URL. When moving to a new cluster, update `overlays/dev/patches/keycloak-hostname.yaml` with the new apps domain.
- **RHBK operator Ingress has no host**: The RHBK operator creates a Kubernetes Ingress without a `host` field, so OpenShift doesn't auto-generate a usable Route. An explicit Route is included in `base/keycloak/route.yaml`.
