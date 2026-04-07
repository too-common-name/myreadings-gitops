# myreadings-gitops

GitOps repository for deploying the MyReadings application on OpenShift using Argo CD and ACM.

MyReadings is a Quarkus-based modular monolith being incrementally decomposed into microservices. The first extracted service is the user-service. This repo manages the full deployment lifecycle: operator installation via ACM policies, infrastructure provisioning (PostgreSQL, Keycloak, RabbitMQ), application workloads, and CI pipelines.

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
│   └── myreadings-appset.yaml      # ApplicationSet using ACM clusterDecisionResource
├── acm-policies/
│   ├── kustomization.yaml
│   └── myreadings-operators.yaml   # ACM Policy installing operators on labeled clusters
├── base/                           # Shared manifests (Kustomize base)
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── postgres/
│   │   └── postgrescluster.yaml
│   ├── keycloak/
│   │   ├── keycloak.yaml
│   │   ├── keycloak-route.yaml
│   │   └── keycloak-realm-import.yaml
│   ├── rabbitmq/
│   │   ├── rabbitmqcluster.yaml
│   │   └── rabbitmq-config-job.yaml
│   ├── monolith/
│   │   ├── configmap.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── route.yaml
│   ├── user-service/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── ui/
│   │   ├── configmap-nginx.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── route.yaml
│   └── tekton/
│       └── pipeline.yaml
├── overlays/
│   ├── dev/
│   │   ├── kustomization.yaml
│   │   └── patches/
│   │       └── resource-limits.yaml
│   └── prod/
│       ├── kustomization.yaml
│       └── patches/
│           └── resource-limits.yaml
└── tekton/                         # Pipeline tasks, RBAC, triggers (WIP)
    ├── rbac/
    ├── tasks/
    └── triggers/
```

## Prerequisites

These must be installed and configured on the cluster before deploying MyReadings:

| Component | Notes |
|-----------|-------|
| **OpenShift** | Tested on OCP 4.x |
| **Red Hat ACM** | Hub cluster with managed clusters in the `default` ClusterSet |
| **OpenShift GitOps** (Argo CD) | Installed cluster-wide |
| **Ansible Automation Platform** | Operator installed, instance created, subscription activated |

A couple of RBAC tweaks are also needed on the hub (applied once, not managed by this repo):

- The `openshift-gitops-applicationset-controller` ServiceAccount needs `cluster-admin` to read ACM `PlacementDecision` resources (a scoped ClusterRole on `placementdecisions` alone was not sufficient):

```bash
oc adm policy add-cluster-role-to-user cluster-admin \
  system:serviceaccount:openshift-gitops:openshift-gitops-applicationset-controller
```

- AAP connection secret for the Resource Operator CRs (Project, JobTemplates). Create an OAuth2 token in the AAP UI (Access Management > Users > your user (admin) > API Tokens > Create API token), then:

```bash
oc create secret generic aap-connection \
  --from-literal=token=<oauth2-token> \
  --from-literal=host=https://<aap-route> \
  -n myreadings-dev
```

- Register the custom Execution Environment in AAP (Automation Execution > Infrastructure > Execution Environments > Create):
  - Name: `myreadings-ee`
  - Image: `quay.io/rh-ee-drossi/myreadings-ee:latest`

Managed clusters need these labels:

- `myreadings: "true"` — triggers operator installation via ACM policies
- `environment: dev` or `environment: prod` — determines which overlay the ApplicationSet deploys

## How it works

### Operator installation (ACM-driven)

The `acm-policies/myreadings-operators.yaml` policy installs the required operators on any managed cluster labeled `myreadings: "true"`:

- CrunchyData Postgres Operator (`certified-operators`, channel `v5`)
- Red Hat Build of Keycloak (`redhat-operators`, channel `stable-v26.4`)
- OpenShift Pipelines (`redhat-operators`, channel `latest`)
- RabbitMQ Cluster Operator (`community-operators`, channel `stable`)

The policy uses `OperatorPolicy` resources with `remediationAction: enforce`, so operators are installed automatically.

### Application deployment (Argo CD + ACM)

A single `bootstrap.yaml` is all you apply manually. It creates an app-of-apps that, through sync-waves, sets up:

1. **wave -4**: ACM-Argo CD integration (GitOpsCluster, ManagedClusterSetBinding, broad Placement)
2. **wave -2**: AppProjects (`myreadings-platform` for policies, `myreadings` for workloads)
3. **wave -1**: Environment-specific Placements (dev, prod)
4. **wave 0**: ACM policies Application + ApplicationSet for workloads

The ApplicationSet uses the `clusterDecisionResource` generator to dynamically discover clusters from ACM PlacementDecisions. When a cluster matches the `dev-placement`, Argo CD creates an Application deploying `overlays/dev` to that cluster.

### Post-deploy configuration (AAP)

Keycloak realm/client setup and RabbitMQ vhost/user/queue configuration are managed by Ansible Automation Platform job templates, reusing the existing Ansible roles from `myreadings_deploy/ansible/`.

### CI (Tekton)

Container images are built using a parameterized Tekton Pipeline (`git-clone` + `buildah`) and pushed to the OpenShift internal registry. Pipeline runs are triggered per component (monolith, user-service, UI).

## Getting started

### 1. Label your managed cluster

```bash
oc label managedcluster <cluster-name> myreadings="true" environment="dev"
```

### 2. Apply the bootstrap

```bash
oc apply -f bootstrap.yaml
```

This kicks off the full chain: projects, placements, ACM policies, and the ApplicationSet. Within a few minutes, operators will be installed and applications deployed.

### 3. Activate post-deploy configuration

Run the AAP job templates to configure Keycloak (realm, client, roles) and RabbitMQ (vhost, user, permissions, queue bindings).

### 4. Build container images

Trigger Tekton PipelineRuns for each component to build and push images to the internal registry.

## Container images

| Component | Image |
|-----------|-------|
| Monolith | `image-registry.openshift-image-registry.svc:5000/myreadings-dev/myreadings-app:latest` |
| User Service | `image-registry.openshift-image-registry.svc:5000/myreadings-dev/myreadings-user-service:latest` |
| UI | `image-registry.openshift-image-registry.svc:5000/myreadings-dev/myreadings-ui:latest` |
| Keycloak (custom) | `quay.io/rh-ee-drossi/myreadings-keycloak:latest` |

The custom Keycloak image extends `registry.redhat.io/rhbk/keycloak-rhel9:26.4` with a custom theme JAR and a RabbitMQ event listener JAR.
