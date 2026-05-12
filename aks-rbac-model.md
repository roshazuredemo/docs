# RBAC Model — Azure & Kubernetes

**Document type:** Architecture Reference
**Audience:** Platform engineers, DevOps, security team
**Classification:** Internal — Restricted

---

## What this document covers

This document explains how access control works across two layers — the **Azure layer** (who can provision and manage Azure resources) and the **Kubernetes layer** (who can deploy and operate workloads inside clusters). The two layers are separate but connected, and both follow the same principle: every identity — human or automated — gets only the access it needs for its specific job, nothing more.

---

## The two-layer model at a glance

Think of it in two tiers:

**Tier 1 — Azure layer**
Controls who can create and manage Azure resources (clusters, databases, storage, container registries, key vaults). Identities here are SPNs (for pipelines) and Managed Identities (for pods connecting to Azure services). Humans get Azure RBAC roles via Entra ID groups.

**Tier 2 — Kubernetes layer**
Controls who can deploy and operate workloads inside a cluster. Identities here are the same SPNs (bound via kubeconfig) and the same Entra ID groups (integrated via AKS Entra ID integration). Kubernetes Roles and ClusterRoles define what each identity can do.

```
On-premises / Pipeline          Azure                       Kubernetes
─────────────────────           ─────────────────────────   ──────────────────────────
CloudBees (SPN) ──────────────► Azure RBAC role ──────────► K8s RoleBinding
Humans (Entra group) ─────────► Azure RBAC role ──────────► K8s ClusterRoleBinding
Pods (Managed Identity) ──────► AKV / ACR / Storage         (no K8s role needed)
```

---

## Part 1 — Azure Layer

### 1.1 Service Principals (Pipeline Automation)

SPNs are used exclusively by pipelines. No human logs in as an SPN. Each SPN is a separate App Registration in the relevant tenant, scoped to exactly the resources it needs to touch.

#### Infrastructure SPNs

These create and manage Azure resources.

| SPN Name | Tenant | Azure Scope | Role | What the pipeline does with it |
|---|---|---|---|---|
| `spn-infra-dev` | Non-Prod | `rg-common-dev`, `rg-app-sandbox`, `rg-app-dev` | Contributor | Provisions ACR, Key Vault, Storage Accounts, PostgreSQL VMs in the Dev subscription |
| `spn-infra-test` | Non-Prod | `rg-common-test`, `rg-app-intgtest`, `rg-app-systest`, `rg-app-uat`, `rg-app-perftest` | Contributor | Same as above but for the Test subscription |
| `spn-infra-prod` | Prod | `rg-common-prod`, `rg-app-prod` | Contributor | Same but for the Prod subscription. Separate tenant, separate credentials. |
| `spn-cluster-dev` | Non-Prod | `rg-aks-dev`, `rg-network-dev` | Contributor + Network Contributor | Creates and configures the Dev AKS cluster and subnet delegation |
| `spn-cluster-test` | Non-Prod | `rg-aks-test`, `rg-network-test` | Contributor + Network Contributor | Same for Test cluster |
| `spn-cluster-prod` | Prod | `rg-aks-prod`, `rg-network-prod` | Contributor + Network Contributor | Same for Prod cluster. Prod tenant only. |

#### Image SPNs

These move container images between registries.

| SPN Name | Tenant | Azure Scope | Role | What it does |
|---|---|---|---|---|
| `spn-image-push-dev` | Non-Prod | `devacr001` (resource level) | AcrPush | CloudBees agents push newly built images here after the build step |
| `spn-image-push-test` | Non-Prod | `testacr001` (resource level) | AcrPush | Images promoted from Dev ACR land here |
| `spn-image-push-prod` | Prod | `prodacr001` (resource level) | AcrPush | Final promotion to Prod ACR after UAT sign-off |
| `spn-image-promote` | Both | Source ACR (AcrPull) + Dest ACR (AcrPush) | AcrPull + AcrPush | Runs the promotion pipeline — pulls from source, re-tags, pushes to destination. Never rebuilds. |

> **Important:** No SPN has Owner or User Access Administrator. This means no SPN can self-assign roles. The AcrPull role on each cluster's Managed Identity must be assigned manually by the Cloud Team after cluster provisioning. Build this into the provisioning runbook.

---

### 1.2 Managed Identities (Pod-to-Azure Connectivity)

Pods never use SPN credentials. Instead, each environment namespace has a Kubernetes ServiceAccount that is federated to an Azure Managed Identity via Workload Identity. When a pod presents its ServiceAccount token, Azure exchanges it for a short-lived access token scoped to that Managed Identity. No secrets are involved.

**How it works in practice:**

```
Pod starts
  → Kubernetes projects a ServiceAccount token into the pod
    → CSI Driver or SDK presents the token to Azure AD
      → Azure issues a short-lived access token for the Managed Identity
        → Pod accesses Key Vault / Storage using that token
          → Token expires — next request gets a fresh one
```

#### Managed Identity reference

| Managed Identity | Subscription | Azure Role Assignments | Used by |
|---|---|---|---|
| `mi-aks-dev` | Dev | AcrPull on `devacr001` | AKS Dev cluster — pulls images for all pods. Assigned by Cloud Team. |
| `mi-aks-test` | Test | AcrPull on `testacr001` | AKS Test cluster — pulls images for all pods |
| `mi-aks-prod` | Prod | AcrPull on `prodacr001` | AKS Prod cluster — pulls images for all pods |
| `mi-app-sandbox` | Dev | Key Vault Secrets User (sandbox-* secrets), Storage Blob Data Reader on `sandboxsa001` | Pods in `app-sandbox` namespace |
| `mi-app-dev` | Dev | Key Vault Secrets User (dev-* secrets), Storage Blob Data Reader on `devsa001` | Pods in `app-dev` namespace |
| `mi-app-intgtest` | Test | Key Vault Secrets User (intgtest-* secrets), Storage Blob Data Reader on `intgtestsa001` | Pods in `app-intgtest` namespace |
| `mi-app-systest` | Test | Key Vault Secrets User (systest-* secrets), Storage Blob Data Reader on `systestsa001` | Pods in `app-systest` namespace |
| `mi-app-uat` | Test | Key Vault Secrets User (uat-* secrets), Storage Blob Data Reader on `uatsa001` | Pods in `app-uat` namespace |
| `mi-app-perftest` | Test | Key Vault Secrets User (perftest-* secrets), Storage Blob Data Reader on `perftestsa001` | Pods in `app-perftest` namespace |
| `mi-app-prod` | Prod | Key Vault Secrets User (prod-* secrets), Storage Blob Data Reader on `prodsa001` | Pods in `app-prod` namespace |

> Each `mi-app-{env}` is scoped to only its own environment's secrets and storage. A misconfigured pod in `app-dev` cannot reach `prod-*` secrets even if it tries — the Managed Identity simply does not have permission.

---

### 1.3 Entra ID Groups — Human Access to Azure

Humans access Azure through their Entra ID account. Role assignments are on the group, not the individual — adding a person to the right group is all that is needed to grant or revoke access.

| Entra Group | Azure RBAC Role | Scope | Who is in this group |
|---|---|---|---|
| `grp-platform-engineers` | Contributor | All subscriptions (both tenants) | Platform Engineering team. Can provision and manage all Azure resources. Cannot assign roles. |
| `grp-dev-team` | Reader | Dev and Test subscriptions | Application developers. Can see resource configuration, costs, and metrics but cannot change anything. |
| `grp-ops-readonly` | Reader | All subscriptions | Operations and support staff. Read-only view of all resources. |
| `grp-security-audit` | Security Reader | All subscriptions | Security and compliance team. Can view security policies, Defender alerts, and diagnostic settings. |
| `grp-breakglass` | Owner | All subscriptions | Platform Lead + Security Lead only. PAM-gated, dual approval, session recorded. Used for emergencies only — not day to day. |

---

## Part 2 — Kubernetes Layer

AKS is integrated with Entra ID. When a human runs `kubectl`, their Entra ID identity is used for authentication. Kubernetes then checks its own RBAC to decide what they are allowed to do. Pipeline SPNs authenticate with a kubeconfig generated per cluster.

### 2.1 ClusterRoles — What Permissions Look Like

A ClusterRole defines a set of permissions. It is not assigned to anyone by itself — it becomes meaningful only when bound to an identity via a ClusterRoleBinding or RoleBinding.

The platform defines these custom ClusterRoles on every cluster:

---

#### `platform-admin`

Intended for: Platform engineers who need to operate and troubleshoot the cluster.

```yaml
# Can read everything across the entire cluster
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]

# Can write to system and platform namespaces
- apiGroups: ["", "apps", "batch"]
  resources: ["deployments", "pods", "services", "configmaps", "daemonsets"]
  verbs: ["create", "update", "patch", "delete"]

# Can manage RBAC objects (but not ClusterRoles themselves)
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["roles", "rolebindings"]
  verbs: ["get", "list", "watch", "create", "update", "delete"]
```

---

#### `cluster-readonly`

Intended for: Operations staff and developers who need visibility but no write access.

```yaml
# Read everything — no exceptions
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
```

---

#### `audit-viewer`

Intended for: Security and compliance team. Focused on policy and audit data only.

```yaml
# Kyverno policy reports
- apiGroups: ["wgpolicyk8s.io", "kyverno.io"]
  resources: ["policyreports", "clusterpolicyreports", "admissionreports"]
  verbs: ["get", "list", "watch"]

# Events and audit logs
- apiGroups: [""]
  resources: ["events"]
  verbs: ["get", "list", "watch"]

# Pod logs (for audit trail)
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
```

---

#### `namespace-deployer` (used as a Role, not ClusterRole)

Intended for: Pipeline SPNs deploying application workloads. Applied per namespace — not cluster-wide.

```yaml
# Core workload resources
- apiGroups: ["", "apps", "batch", "autoscaling"]
  resources: ["deployments", "replicasets", "statefulsets", "pods",
              "services", "configmaps", "persistentvolumeclaims",
              "serviceaccounts", "horizontalpodautoscalers"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

# Secrets Store CSI — so the pipeline can create SecretProviderClass
- apiGroups: ["secrets-store.csi.x-k8s.io"]
  resources: ["secretproviderclasses"]
  verbs: ["get", "list", "create", "update", "patch"]

# Helm release tracking (if using Flux HelmRelease CRD)
- apiGroups: ["helm.toolkit.fluxcd.io"]
  resources: ["helmreleases"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
```

> `namespace-deployer` is defined as a **Role** (not ClusterRole) because it should never apply cluster-wide. A pipeline SPN for the `app-dev` namespace cannot affect `app-prod` — the Role only exists inside the namespace it is created in.

---

#### `namespace-readonly`

Intended for: Developers who need to read what is running in their environment namespace.

```yaml
# Everything in the namespace — read only
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
```

---

### 2.2 How Humans Get Access — ClusterRoleBindings

Humans authenticate via Entra ID. AKS maps their Entra group membership to a Kubernetes ClusterRole via a ClusterRoleBinding.

| Entra Group | Kubernetes Binding | ClusterRole | Effect |
|---|---|---|---|
| `grp-platform-engineers` | ClusterRoleBinding | `platform-admin` | Can read all namespaces, write to system/platform namespaces, manage RBAC |
| `grp-ops-readonly` | ClusterRoleBinding | `cluster-readonly` | Can read everything across all namespaces — no writes |
| `grp-security-audit` | ClusterRoleBinding | `audit-viewer` | Can read policy reports, events, and pod logs only |
| `grp-breakglass` | ClusterRoleBinding | `cluster-admin` (built-in) | Full cluster access. PAM-gated, time-limited, session recorded. |

> Developers (`grp-dev-team`) do not get a ClusterRoleBinding. They get RoleBindings in specific namespaces only — see below.

---

### 2.3 How Humans Get Access — Namespace RoleBindings

Developers only need visibility into the namespaces relevant to their work. They get a RoleBinding per namespace rather than a cluster-wide binding. This means a dev team member cannot see what is running in `app-prod` or any system namespace.

| Entra Group | Namespace | Kubernetes Binding | Role |
|---|---|---|---|
| `grp-dev-team` | `app-sandbox` | RoleBinding | `namespace-readonly` |
| `grp-dev-team` | `app-dev` | RoleBinding | `namespace-readonly` |
| `grp-dev-team` | `app-intgtest` | RoleBinding | `namespace-readonly` |
| `grp-dev-team` | `app-systest` | RoleBinding | `namespace-readonly` |
| `grp-dev-team` | `app-uat` | RoleBinding | `namespace-readonly` |
| `grp-dev-team` | `app-perftest` | RoleBinding | `namespace-readonly` |

> Dev team members cannot read `app-prod`. If they need to investigate a production issue they must request it through the break-glass process or ask an ops-readonly member to pull the information for them.

---

### 2.4 How Pipelines Get Access — SPN Bindings

Pipelines authenticate to Kubernetes using a kubeconfig generated per cluster per SPN. The kubeconfig is stored in CloudBees Credentials Store and rotated every 90 days.

#### Cluster-level SPN bindings

| SPN | Kubernetes Binding | Role/ClusterRole | Scope | Why |
|---|---|---|---|---|
| `spn-k8s-baseline` | ClusterRoleBinding | `cluster-admin` (built-in) | All namespaces | Installs system operators (Istio, Kyverno, Prometheus, etc.) via Helm. Pipeline use only — never interactive. |

> `spn-k8s-baseline` has `cluster-admin` because operator installation touches multiple namespaces and CRDs. Once baseline is installed, this SPN is not used again until the next operator upgrade cycle.

#### Namespace-level SPN bindings

Each environment namespace gets its own SPN. The SPN can only deploy to its own namespace.

| SPN | Kubernetes Binding | Role | Namespace |
|---|---|---|---|
| `spn-app-sandbox` | RoleBinding | `namespace-deployer` | `app-sandbox` |
| `spn-app-dev` | RoleBinding | `namespace-deployer` | `app-dev` |
| `spn-app-intgtest` | RoleBinding | `namespace-deployer` | `app-intgtest` |
| `spn-app-systest` | RoleBinding | `namespace-deployer` | `app-systest` |
| `spn-app-uat` | RoleBinding | `namespace-deployer` | `app-uat` |
| `spn-app-perftest` | RoleBinding | `namespace-deployer` | `app-perftest` |
| `spn-app-prod` | RoleBinding | `namespace-deployer` | `app-prod` |

> `spn-app-prod` additionally requires a human approval gate in the pipeline before it can execute. The SPN has the permission — the pipeline step is the gate, not the RBAC.

---

## Part 3 — How the Two Layers Connect

This is where it comes together. The table below shows how each identity type spans from the human or system doing the work, through the Azure layer, into the Kubernetes layer.

| Who | Azure identity | Azure permission | Kubernetes identity | Kubernetes permission |
|---|---|---|---|---|
| CloudBees build pipeline | `spn-image-push-{sub}` | AcrPush on ACR | None — does not touch K8s | None |
| CloudBees infra pipeline | `spn-infra-{sub}` | Contributor on RGs | None — does not touch K8s | None |
| CloudBees cluster pipeline | `spn-cluster-{sub}` | Contributor + Network Contributor | None — does not touch K8s | None |
| CloudBees baseline pipeline | `spn-k8s-baseline` | None — does not touch Azure resources | kubeconfig → Entra SPN | ClusterRoleBinding → `cluster-admin` |
| CloudBees app deploy pipeline | `spn-app-{env}` | None — does not touch Azure resources | kubeconfig → Entra SPN | RoleBinding → `namespace-deployer` in env namespace |
| Application pod | `mi-app-{env}` (Workload Identity) | Key Vault Secrets User, Storage Reader | ServiceAccount (federated) | None — pod accesses Azure, not K8s API |
| AKS cluster (pulling images) | `mi-aks-{sub}` | AcrPull on ACR | None — kubelet credential, not user-facing | None |
| Platform engineer | Entra account in `grp-platform-engineers` | Contributor on subscriptions | Entra group → AKS integration | ClusterRoleBinding → `platform-admin` |
| Developer | Entra account in `grp-dev-team` | Reader on Dev/Test subscriptions | Entra group → AKS integration | RoleBinding → `namespace-readonly` in app-* namespaces |
| Ops staff | Entra account in `grp-ops-readonly` | Reader on all subscriptions | Entra group → AKS integration | ClusterRoleBinding → `cluster-readonly` |
| Security team | Entra account in `grp-security-audit` | Security Reader on all subscriptions | Entra group → AKS integration | ClusterRoleBinding → `audit-viewer` |
| Emergency access | Entra account in `grp-breakglass` | Owner (PAM-gated) | Entra group → AKS integration | ClusterRoleBinding → `cluster-admin` (time-limited) |

---

## Part 4 — Quick Reference Summary

### Azure RBAC roles used on this platform

| Role | What it allows |
|---|---|
| Contributor | Create, update, delete resources. Cannot assign roles. |
| Network Contributor | Manage VNets, subnets, NSGs. Needed for AKS subnet delegation. |
| Reader | Read-only view of all resources. Cannot see secrets. |
| Security Reader | Read Defender alerts, security policies, diagnostic settings. |
| Owner | Everything including role assignment. Break-glass only. |
| AcrPush | Push images to a Container Registry. |
| AcrPull | Pull images from a Container Registry. |
| Key Vault Secrets User | Read secret values from Key Vault. Cannot manage or delete secrets. |
| Storage Blob Data Reader | Read files from a Storage Account. Cannot write or delete. |

---

### Kubernetes RBAC objects used on this platform

| Object | Scope | What it is |
|---|---|---|
| ClusterRole | Cluster-wide | A named set of permissions that applies across all namespaces |
| Role | Namespace | A named set of permissions that applies only within one namespace |
| ClusterRoleBinding | Cluster-wide | Binds a ClusterRole to an identity (Entra group or SPN) for the whole cluster |
| RoleBinding | Namespace | Binds a ClusterRole or Role to an identity within a single namespace |

---

### Golden rules

1. **No identity has more access than its job requires.** Infrastructure SPNs cannot deploy to Kubernetes. Application deploy SPNs cannot touch Azure resources. Pod identities cannot see other environments.

2. **Humans never use SPN credentials interactively.** SPNs are pipeline identities only. Human access always goes through Entra ID and personal accounts.

3. **Pods never store secrets.** Secrets come from Key Vault, mounted at runtime via the CSI Driver. They are never in environment variables, ConfigMaps, or container images.

4. **Role assignments to Managed Identities are done by the Cloud Team.** No pipeline SPN has the Azure permission to assign roles. After cluster provisioning, the AcrPull assignment is a manual step — it must be in the runbook.

5. **Break-glass is logged and time-limited.** Both the Azure Owner role and the Kubernetes cluster-admin ClusterRoleBinding for the break-glass group generate audit events in Splunk and are automatically revoked after four hours.

---

*AKS Platform · RBAC Model · v1.0 Draft · Internal — Restricted*
