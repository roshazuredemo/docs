# AKS Platform — Automation Pipeline Architecture

**Document type:** Engineering Reference
**Audience:** Platform engineers, DevOps, pipeline developers
**Classification:** Internal — Restricted

---

## 1. Overview

This document describes every CloudBees pipeline required to build and operate the AKS platform from scratch. Pipelines are organised into numbered layers. Each layer has a hard dependency on the layer beneath it and must be completed successfully before the next layer is triggered. No pipeline is auto-chained — every layer is triggered independently to allow individual rollback and clear audit separation.

**Starting assumption — what the Cloud Team pre-provisions for us:**

The following resources exist before any of our pipelines run. They come from the Azure Landing Zone and are outside our control.

| Pre-provisioned resource | Where | Notes |
|---|---|---|
| Spoke subscription | Per environment (Dev, Test, Prod) | Already created and linked to the correct Entra ID tenant |
| VNet with 3 subnets | `rg-network-{sub}` | `snet-vm`, `snet-services`, `snet-aks` — pre-allocated CIDRs |
| Hub VNet peering | Hub subscription | Already established — traffic flows via Hub Firewall |
| Private DNS zones | Hub or spoke (confirm with Cloud Team) | e.g. `privatelink.azurecr.io`, `privatelink.vaultcore.azure.net` |
| Master Managed Identity | `rg-common-{sub}` | Used to bootstrap SPN creation and initial Terraform runs |
| Storage encryption key (CMK) | Shared subscription Key Vault | Customer-managed key for storage account encryption |
| Managed Identity for CMK | Shared subscription | Has `Key Vault Crypto User` on the CMK — passed to storage resources |

**Pipeline execution model:**

All pipelines run on CloudBees on-premises agents. Agents connect to Azure via ExpressRoute through the Hub Firewall. All Azure endpoints (ACR, AKV, AKS API server) are private — no public internet traversal.

```
Stash (Bitbucket) → Webhook → CloudBees Controller → CloudBees Agent (on-prem)
                                                           │
                                          ┌────────────────┼────────────────┐
                                          ▼                ▼                ▼
                                    Terraform          Helm / kubectl    Azure CLI
                                    (via ARM)          (via AKS API)    (az commands)
                                          │                │
                                    ExpressRoute     ExpressRoute
                                          │                │
                                    Azure ARM        AKS Private
                                    Endpoint         API Server
```

---

## 2. Pipeline Layers at a Glance

| Layer | Pipeline name | Tool | Identity | Trigger | Deploys |
|---|---|---|---|---|---|
| L0 | subscription-config | Azure CLI + Terraform | `spn-infra-{sub}` | Manual — once per subscription | Resource providers, policy assignments, budget alerts, diagnostics |
| L1 | common-resources | Terraform (WNZL modules) | `spn-infra-{sub}` | Manual — on change | ACR, Key Vault, Private Endpoints, Managed Identities |
| L2 | aks-cluster-provision | Terraform (AVM + WNZL) | `spn-cluster-{sub}` | Manual — on change | AKS cluster, node pools, cluster Managed Identity |
| L2.1 | aks-acrpull-assignment | Azure CLI (Cloud Team runbook) | Cloud Team Owner account | Manual — post L2 | AcrPull role assignment on ACR for AKS MI |
| L3 | cluster-bootstrap | Helm + kubectl | `spn-k8s-baseline` | Manual — post L2.1 | CSI Driver, cert-manager, Kyverno |
| L4 | cluster-operators | Helm | `spn-k8s-baseline` | Manual — post L3 | Istio, Prometheus, Alertmanager, KEDA, RabbitMQ, Fluent Bit, Jaeger, OTel |
| L5 | cluster-baseline | Terraform + kubectl | `spn-k8s-baseline` | Manual — post L4 | System namespaces, ClusterRoles, ClusterRoleBindings, Entra ID group bindings |
| L6 | app-namespace-config | Terraform + kubectl + Helm | `spn-app-{env}` | Manual — once per environment | App namespace, RBAC, NetworkPolicy, ResourceQuota, Workload Identity, SecretProviderClass, PVC |
| L7 | image-build-push | Docker + Azure CLI | `spn-image-push-{sub}` | Manual (vendor ZIP) or webhook | Build vendor image, inject enterprise config, push to ACR |
| L8 | app-deploy | Helm | `spn-app-{env}` | Automatic (dev/test) or gated (prod) | Application Helm release per environment |

---

## 3. Layer 0 — Subscription Configuration

**Repository:** `platform-subscription-config`
**Purpose:** Configure the subscription itself before any resources are provisioned. This is a one-time pipeline per subscription but can be re-run safely at any time.

### What it does

| Step | Tool | Action |
|---|---|---|
| 1. Register resource providers | Azure CLI | Registers `Microsoft.ContainerService`, `Microsoft.ContainerRegistry`, `Microsoft.KeyVault`, `Microsoft.Storage`, `Microsoft.Network`, `Microsoft.ManagedIdentity`, `Microsoft.Authorization` |
| 2. Set subscription-level diagnostics | Azure CLI / Terraform | Sends Activity Log to Log Analytics workspace in Hub subscription |
| 3. Apply budget alert | Terraform (WNZL module) | Creates budget alert at subscription level — notifies platform team at 80% and 100% of monthly threshold |
| 4. Apply required tags | Azure CLI | Sets mandatory tags on the subscription — `environment`, `team`, `cost-centre`, `managed-by` |
| 5. Configure Microsoft Defender | Azure CLI | Enables Defender for Containers and Key Vault at subscription level |
| 6. Verify Landing Zone policy compliance | Azure CLI | Runs `az policy state summarize` and fails the pipeline if any non-compliant assignments exist before proceeding |

### Pipeline identity

| Item | Value |
|---|---|
| SPN | `spn-infra-{sub}` |
| Azure role required | Contributor on subscription + Security Admin (for Defender) |
| K8s access | None |
| Credentials stored in | CloudBees Credentials Store |

### Key parameters

| Parameter | Description | Example |
|---|---|---|
| `SUBSCRIPTION_ID` | Target subscription ID | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| `ENVIRONMENT` | dev / test / prod | `dev` |
| `BUDGET_THRESHOLD_NZD` | Monthly budget alert amount | `5000` |
| `LOG_ANALYTICS_WORKSPACE_ID` | Hub subscription workspace resource ID | `/subscriptions/.../workspaces/hub-law-001` |

### Outputs

Nothing is provisioned — this pipeline configures subscription-level settings only. A compliance report is archived as a CloudBees artefact.

### Idempotency note

All steps are safe to re-run. Resource provider registration is a no-op if already registered. Budget alerts are managed in Terraform state.

---

## 4. Layer 1 — Common Resources

**Repository:** `infra-common`
**Purpose:** Provision the shared Azure resources that every environment in the subscription depends on — ACR, Key Vault, Private Endpoints, and Managed Identities. These are created once per subscription and shared across all clusters and environments in that subscription.

### What it does

| Step | Tool | Module type | Action |
|---|---|---|---|
| 1. Create resource group | Terraform | WNZL internal module | Creates `rg-common-{sub}` with mandatory tags |
| 2. Provision ACR | Terraform | WNZL internal module | Creates ACR in `rg-common-{sub}`, Standard SKU (Dev/Test) or Premium (Prod), public access disabled, private endpoint enabled |
| 3. Provision Key Vault | Terraform | WNZL internal module | Creates AKV with RBAC authorisation model (not access policies), soft delete enabled, purge protection on, public access disabled |
| 4. Create Private Endpoint — ACR | Terraform | WNZL internal module | Projects ACR into `snet-services` as a private IP, links to Private DNS zone `privatelink.azurecr.io` |
| 5. Create Private Endpoint — Key Vault | Terraform | WNZL internal module | Projects AKV into `snet-services` as a private IP, links to Private DNS zone `privatelink.vaultcore.azure.net` |
| 6. Register Private DNS records | Terraform | WNZL internal module | Adds A records to Private DNS zones for ACR and AKV private IPs |
| 7. Create Managed Identities | Terraform | WNZL internal module | Creates one `mi-app-{env}` per environment in this subscription (e.g. `mi-app-sandbox`, `mi-app-dev`) |
| 8. Assign Key Vault roles to MIs | Terraform | WNZL internal module | Grants `Key Vault Secrets User` on the AKV to each `mi-app-{env}`, scoped to the env-prefixed secrets only via conditions |
| 9. Assign Storage roles to MIs | Terraform | WNZL internal module | Grants `Storage Blob Data Reader` on the respective storage account to each `mi-app-{env}` — storage accounts are created in L1 app-resources pipeline |
| 10. Store outputs in Key Vault | Terraform | Azure provider | Writes ACR login server, AKV URI, and MI client IDs as secrets in AKV for downstream pipelines to consume |

### Pipeline identity

| Item | Value |
|---|---|
| SPN | `spn-infra-{sub}` |
| Azure roles required | Contributor on `rg-common-{sub}`, Network Contributor on `rg-network-{sub}` (for Private Endpoint DNS) |
| K8s access | None |
| Credentials stored in | CloudBees Credentials Store |
| Terraform state | TF Enterprise — workspace `infra-common-{sub}` |

### Key parameters

| Parameter | Description |
|---|---|
| `SUBSCRIPTION_ID` | Target subscription |
| `ENVIRONMENT` | dev / test / prod |
| `VNET_RESOURCE_ID` | Pre-provisioned spoke VNet resource ID (from Cloud Team) |
| `SNET_SERVICES_ID` | Private Endpoints subnet resource ID |
| `PRIVATE_DNS_ZONE_ACR_ID` | ACR Private DNS zone resource ID (from Cloud Team) |
| `PRIVATE_DNS_ZONE_AKV_ID` | AKV Private DNS zone resource ID |
| `CMK_KEY_ID` | Storage encryption key URI from shared subscription AKV |
| `CMK_MI_ID` | Managed Identity resource ID that has access to the CMK |
| `ENVIRONMENTS_IN_SUB` | Comma-separated list of env names (e.g. `sandbox,dev` for Dev sub) |

### Outputs (written to TF state and AKV)

| Output | Written to |
|---|---|
| ACR login server URL | TF state output + AKV secret `common-acr-login-server` |
| ACR resource ID | TF state output |
| AKV URI | TF state output + AKV secret `common-akv-uri` |
| `mi-app-{env}` client ID (per env) | TF state output + AKV secret `mi-app-{env}-client-id` |

---

## 5. Layer 1.1 — Application Resources (Per Environment)

**Repository:** `infra-app-resources`
**Purpose:** Provision the environment-specific Azure resources — PostgreSQL VMs and Storage Accounts with SFTP. This pipeline is parameterised and run once per environment.

### What it does

| Step | Tool | Module type | Action |
|---|---|---|---|
| 1. Create resource group | Terraform | WNZL internal module | Creates `rg-app-{env}` with mandatory tags and environment label |
| 2. Provision Storage Account | Terraform | WNZL internal module | Creates Storage Account in `rg-app-{env}`, Standard_ZRS, SFTP enabled, public access disabled, encrypted with CMK from shared subscription |
| 3. Create NFS file share | Terraform | WNZL internal module | Creates file share `app-data-{env}` with NFS v4.1 protocol inside the Storage Account |
| 4. Create Private Endpoint — Storage | Terraform | WNZL internal module | Projects Storage Account into `snet-services`, links to Private DNS zone `privatelink.blob.core.windows.net` and `privatelink.file.core.windows.net` |
| 5. Create SFTP local user | Terraform | WNZL internal module | Creates SFTP local user for on-premises physical client file uploads |
| 6. Provision PostgreSQL VMs | Terraform | WNZL internal module | Creates PostgreSQL VM(s) in `rg-app-{env}`, placed in `snet-vm` subnet. NIC attached to VNet — no Public IP. |
| 7. Configure VM disks | Terraform | WNZL internal module | Attaches managed premium SSD data disks for PostgreSQL data and WAL volumes |
| 8. Write connection strings to Key Vault | Terraform | Azure provider | Writes PostgreSQL connection strings as AKV secrets using env-prefixed naming (e.g. `sandbox-db-connstr`) |
| 9. Write storage key to Key Vault | Terraform | Azure provider | Writes Storage Account key as AKV secret `{env}-storage-key` for CSI Driver to mount |

### Pipeline identity

| Item | Value |
|---|---|
| SPN | `spn-infra-{sub}` |
| Azure roles required | Contributor on `rg-app-{env}`, Network Contributor on `rg-network-{sub}`, Key Vault Secrets Officer on AKV (to write secrets) |
| K8s access | None |
| Terraform state | TF Enterprise — workspace `infra-app-{env}` |

### Key parameters

| Parameter | Description |
|---|---|
| `ENVIRONMENT` | Target environment (e.g. `sandbox`, `intgtest`) |
| `SUBSCRIPTION_ID` | Target subscription |
| `SNET_VM_ID` | VM subnet resource ID |
| `SNET_SERVICES_ID` | Services subnet resource ID |
| `AKV_RESOURCE_ID` | Common AKV resource ID (from L1 outputs) |
| `CMK_KEY_ID` | CMK URI for storage encryption |
| `CMK_MI_ID` | MI resource ID for CMK access |
| `POSTGRES_VM_SIZE` | VM SKU for PostgreSQL (e.g. `Standard_D4s_v5`) |
| `POSTGRES_VM_COUNT` | Number of PostgreSQL VMs (typically 1 for dev/test, 3 for prod) |

---

## 6. Layer 2 — AKS Cluster Provisioning

**Repository:** `infra-aks-clusters`
**Purpose:** Provision the AKS private cluster, configure node pools, and attach the cluster's Managed Identity. Uses AVM (Azure Verified Modules) for the AKS resource itself and WNZL internal modules for supporting resources.

### What it does

| Step | Tool | Module type | Action |
|---|---|---|---|
| 1. Create AKS resource group | Terraform | WNZL internal module | Creates `rg-aks-{sub}` with mandatory tags |
| 2. Create cluster Managed Identity | Terraform | WNZL internal module | Creates `mi-aks-{sub}` — used by the kubelet to pull images and manage node resources |
| 3. Assign MI to subnet | Terraform | WNZL internal module | Grants `Network Contributor` on `snet-aks` to `mi-aks-{sub}` — required for AKS to manage load balancer IPs and subnet delegation |
| 4. Provision AKS cluster | Terraform | **AVM** `avm-res-containerservice-managedcluster` | Creates private AKS cluster — private API server, no public FQDN, Entra ID integration enabled, Azure CNI networking, RBAC enabled |
| 5. Configure system node pool | Terraform | AVM (node pool submodule) | Tainted `CriticalAddonsOnly=true:NoSchedule`, VM SKU, OS disk type, single AZ (dev/test) or zone-spread (prod) |
| 6. Configure platform node pool | Terraform | AVM (node pool submodule) | Tainted `workload=platform:NoSchedule`, VM SKU, fixed node count, OS disk type |
| 7. Configure application node pool | Terraform | AVM (node pool submodule) | Tainted `workload=application:NoSchedule`, KEDA-compatible autoscale min/max, VM SKU |
| 8. Attach ACR to cluster | Terraform | WNZL internal module | **Note:** This step raises a Cloud Team request — the pipeline cannot self-assign the AcrPull role. The Terraform outputs include the MI object ID and ACR resource ID needed for the request. |
| 9. Configure diagnostic settings | Terraform | WNZL internal module | Sends AKS control plane logs and metrics to Hub Log Analytics workspace |
| 10. Export kubeconfig | Azure CLI | n/a | Retrieves kubeconfig for the private cluster and stores it as a CloudBees credential for downstream pipelines |

### Pipeline identity

| Item | Value |
|---|---|
| SPN | `spn-cluster-{sub}` |
| Azure roles required | Contributor on `rg-aks-{sub}`, Network Contributor on `rg-network-{sub}` |
| K8s access | None at this stage |
| Terraform state | TF Enterprise — workspace `aks-cluster-{sub}` |

### Cluster configuration flags (key AVM inputs)

| Flag | Value | Reason |
|---|---|---|
| `private_cluster_enabled` | `true` | No public API server endpoint |
| `azure_active_directory_role_based_access_control` | `enabled` | Entra ID integration for human kubectl access |
| `network_plugin` | `azure` | Azure CNI — pods get VNet IPs |
| `network_policy` | `azure` or `calico` | Enforces NetworkPolicy objects |
| `oidc_issuer_enabled` | `true` | Required for Workload Identity |
| `workload_identity_enabled` | `true` | Enables pod-level Managed Identity federation |
| `azure_policy_enabled` | `false` | We use Kyverno — not Azure Policy add-on |
| `http_application_routing_enabled` | `false` | No Azure-managed ingress — we use Istio |
| `oms_agent` | `false` | No Azure Monitor add-on — we use Fluent Bit + Prometheus |
| `key_vault_secrets_provider` | `false` | We install the CSI Driver ourselves via Helm (L3) |

### Outputs

| Output | Written to |
|---|---|
| Cluster name | TF state |
| Cluster resource ID | TF state |
| AKS Managed Identity object ID | TF state + AKV secret `aks-mi-object-id` |
| OIDC issuer URL | TF state + AKV secret `aks-oidc-issuer-url` |
| Private API server FQDN | TF state |
| Kubeconfig | CloudBees credential `kubeconfig-{sub}` |

---

## 7. Layer 2.1 — AcrPull Role Assignment (Cloud Team Step)

**Repository:** N/A — this is a Cloud Team runbook step, not a platform pipeline
**Purpose:** Assign the `AcrPull` role on the subscription ACR to the AKS cluster Managed Identity. Our SPNs cannot perform role assignments — only the Cloud Team's Owner-level account can do this.

### What the Cloud Team runs

```bash
# Values come from L2 Terraform outputs
AKS_MI_OBJECT_ID="<value from aks-mi-object-id AKV secret>"
ACR_RESOURCE_ID="<value from L1 TF state output>"

az role assignment create \
  --assignee-object-id $AKS_MI_OBJECT_ID \
  --assignee-principal-type ServicePrincipal \
  --role "AcrPull" \
  --scope $ACR_RESOURCE_ID
```

### Verification step (platform team confirms before proceeding to L3)

```bash
# Confirm assignment exists before triggering L3
az role assignment list \
  --assignee $AKS_MI_OBJECT_ID \
  --scope $ACR_RESOURCE_ID \
  --query "[?roleDefinitionName=='AcrPull']" \
  -o table
```

> **Do not trigger Layer 3 until this step is confirmed.** Cluster pods will fail to pull images if AcrPull is not assigned before the first workload is deployed.

---

## 8. Layer 3 — Cluster Bootstrap

**Repository:** `k8s-cluster-bootstrap`
**Purpose:** Install the foundational cluster components that everything else depends on — Kyverno (admission control), Secrets Store CSI Driver (AKV integration), and cert-manager (TLS). These must be in place before any application or operator workloads are deployed.

### What it does

| Step | Tool | Action |
|---|---|---|
| 1. Retrieve kubeconfig | Azure CLI | Fetches kubeconfig from CloudBees credential store and sets `KUBECONFIG` for the pipeline session |
| 2. Create system namespaces | kubectl | Creates `kyverno`, `cert-manager`, `secrets-store-csi` namespaces with standard labels |
| 3. Install Kyverno | Helm | `helm upgrade --install kyverno kyverno/kyverno -n kyverno` with HA values for prod (3 replicas, `failurePolicy: Fail`) |
| 4. Apply base Kyverno policies | kubectl | Applies ClusterPolicy manifests — pod security, resource limits required, image registry whitelist, default NetworkPolicy generation |
| 5. Install Secrets Store CSI Driver | Helm | `helm upgrade --install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver -n secrets-store-csi` with Azure Key Vault provider |
| 6. Configure Workload Identity webhook | Helm | Installs the Azure Workload Identity webhook (mutating webhook) that injects the projected token volume into pods using Workload Identity |
| 7. Install cert-manager | Helm | `helm upgrade --install cert-manager jetstack/cert-manager -n cert-manager --set installCRDs=true` |
| 8. Create ClusterIssuer | kubectl | Applies `ClusterIssuer` manifest pointing to on-premises Venafi CA endpoint |
| 9. Verify Kyverno webhook is live | kubectl | Polls `kubectl get validatingwebhookconfiguration` until Kyverno webhook is registered |

### Pipeline identity

| Item | Value |
|---|---|
| SPN | `spn-k8s-baseline` |
| Azure role | None |
| K8s role | `ClusterRoleBinding → cluster-admin` |
| Credentials | Kubeconfig from CloudBees credential `kubeconfig-{sub}` |

### Helm chart versions (pin in pipeline — do not use `latest`)

| Chart | Repo | Version variable |
|---|---|---|
| `kyverno` | `https://kyverno.github.io/kyverno/` | `KYVERNO_VERSION` |
| `csi-secrets-store-driver` | `https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts` | `CSI_DRIVER_VERSION` |
| `azurekeyvault-provider` | `https://azure.github.io/secrets-store-csi-driver-provider-azure/charts` | `AKV_PROVIDER_VERSION` |
| `cert-manager` | `https://charts.jetstack.io` | `CERT_MANAGER_VERSION` |

### Kyverno policies installed at this step

| Policy name | Type | Effect |
|---|---|---|
| `require-resource-limits` | ClusterPolicy — Validate | Blocks any pod without CPU and memory limits |
| `restrict-image-registries` | ClusterPolicy — Validate | Only allows images from `{sub}acr001.azurecr.io` |
| `pod-security-restricted` | ClusterPolicy — Validate | Blocks privileged, hostPath, hostNetwork, root user |
| `generate-default-networkpolicy` | ClusterPolicy — Generate | Auto-creates default-deny NetworkPolicy in every new namespace |
| `require-labels` | ClusterPolicy — Validate | All pods must have `environment`, `app`, and `team` labels |
| `mutate-add-labels` | ClusterPolicy — Mutate | Injects `managed-by: platform` label on all resources |

---

## 9. Layer 4 — Cluster Operators

**Repository:** `k8s-ops-components`
**Purpose:** Install all shared platform operators. Each operator is a separate pipeline stage so a failed Jaeger install does not block a Prometheus upgrade. All operators are installed into dedicated namespaces on the platform node pool.

### Operator install stages

Each stage follows the same pattern: `helm repo add` → `helm upgrade --install` → verify rollout.

| Stage | Namespace | Helm chart | Node pool | Notes |
|---|---|---|---|---|
| 4.1 Istio base | `istio-system` | `istio/base` | Platform | Installs Istio CRDs only — no control plane yet |
| 4.2 Istio control plane | `istio-system` | `istio/istiod` | Platform | Installs Istiod. mTLS in STRICT mode. Canary upgrade strategy — run old and new control plane concurrently during upgrades. |
| 4.3 Istio Ingress Gateway | `istio-system` | `istio/gateway` | Platform | Internal load balancer only — annotation `service.beta.kubernetes.io/azure-load-balancer-internal: "true"` |
| 4.4 Prometheus Operator | `monitoring` | `prometheus-community/kube-prometheus-stack` | Platform | Deploys Prometheus, Alertmanager, and default K8s dashboards. Remote-write to on-prem Dynatrace configured in values. |
| 4.5 KEDA | `keda` | `kedacore/keda` | Platform | Event-driven autoscaler. ScaledObject CRDs installed here. |
| 4.6 RabbitMQ Operator | `rabbitmq-system` | `bitnami/rabbitmq-cluster-operator` | Platform | Operator only at this stage — RabbitMQ cluster instances are provisioned in L5 baseline. |
| 4.7 Fluent Bit | `logging` | `fluent/fluent-bit` | All nodes (DaemonSet) | Configured to forward to on-prem ELK and Splunk. Tolerations set to run on all three node pools. |
| 4.8 OpenTelemetry Collector | `opentelemetry` | `open-telemetry/opentelemetry-collector` | Platform | Receives OTLP from application pods. Exports to Jaeger and Dynatrace. |
| 4.9 Jaeger | `jaeger` | `jaegertracing/jaeger` | Platform | Distributed tracing. Storage backend — in-memory for dev/test, Elasticsearch for prod. |

### Pipeline identity

| Item | Value |
|---|---|
| SPN | `spn-k8s-baseline` |
| K8s role | `ClusterRoleBinding → cluster-admin` |
| Notes | Each operator stage is a separate CloudBees stage block so any single failure can be retried without re-running all operators |

### Helm values files per operator

Each operator has a values file per subscription variant:

```
k8s-ops-components/
├── istio/
│   ├── values-dev.yaml
│   ├── values-test.yaml
│   └── values-prod.yaml
├── prometheus/
│   ├── values-dev.yaml
│   └── values-prod.yaml
├── keda/
│   └── values-common.yaml
...
```

---

## 10. Layer 5 — Cluster Baseline

**Repository:** `k8s-cluster-baseline`
**Purpose:** Configure the cluster-level Kubernetes governance objects — system namespace hardening, ClusterRoles, ClusterRoleBindings for Entra ID groups, and node pool labelling. This pipeline is run after operators are installed and is re-run whenever RBAC or node pool label changes are needed.

### What it does

| Step | Tool | Action |
|---|---|---|
| 1. Apply node pool labels and taints | kubectl | Labels nodes with `nodepool=system`, `nodepool=platform`, `nodepool=application` and applies taints. Ensures Kyverno affinity enforcement works. |
| 2. Harden system namespaces | kubectl | Applies `PodSecurity` labels to `kube-system`, `kyverno`, `istio-system` etc. — `enforce: restricted` where possible |
| 3. Create custom ClusterRoles | kubectl | Applies `platform-admin`, `cluster-readonly`, `audit-viewer`, `namespace-deployer`, `namespace-readonly` ClusterRole manifests |
| 4. Bind Entra ID groups to ClusterRoles | kubectl | Creates ClusterRoleBindings mapping Entra group object IDs to each ClusterRole |
| 5. Create platform namespaces | kubectl | Ensures all operator namespaces (`monitoring`, `logging`, `opentelemetry`, `jaeger`, `rabbitmq-system`) exist with correct labels |
| 6. Apply NetworkPolicies — platform namespaces | kubectl | Explicit allow-rules for inter-operator communication (e.g. Prometheus can scrape all namespaces, Fluent Bit can read pod logs) |
| 7. Apply ResourceQuotas — platform namespaces | kubectl | Prevents platform operators from unbounded resource consumption |
| 8. Create RabbitMQ cluster instance | kubectl / Helm | Deploys a `RabbitmqCluster` CR in `rabbitmq-system` — the operator (installed in L4) creates the actual pods |
| 9. Verify all ClusterRoleBindings | kubectl | `kubectl get clusterrolebindings` — pipeline fails if any expected binding is absent |

### Entra ID group → ClusterRole bindings applied

| Entra group name | Group object ID source | ClusterRole |
|---|---|---|
| `grp-platform-engineers` | Pipeline parameter `ENTRA_GROUP_PLATFORM_ENGINEERS` | `platform-admin` |
| `grp-ops-readonly` | Pipeline parameter `ENTRA_GROUP_OPS_READONLY` | `cluster-readonly` |
| `grp-security-audit` | Pipeline parameter `ENTRA_GROUP_SECURITY_AUDIT` | `audit-viewer` |
| `grp-breakglass` | Pipeline parameter `ENTRA_GROUP_BREAKGLASS` | `cluster-admin` |

> Entra group object IDs are passed as pipeline parameters — they are not hardcoded in manifests. This allows the same baseline pipeline to work across both tenants (Non-Prod and Prod) by changing the parameter set.

### Pipeline identity

| Item | Value |
|---|---|
| SPN | `spn-k8s-baseline` |
| K8s role | `ClusterRoleBinding → cluster-admin` |

---

## 11. Layer 6 — Application Namespace Configuration

**Repository:** `k8s-app-namespace`
**Purpose:** Create and fully configure a single application namespace. This pipeline is parameterised and run once per environment (sandbox, dev, intgtest, etc.). It wires the namespace to its specific Azure resources via Workload Identity federation.

### What it does

| Step | Tool | Action |
|---|---|---|
| 1. Create namespace | kubectl | `kubectl create namespace app-{env}` with labels `environment={env}`, `team={team}`, `istio-injection=enabled` |
| 2. Create Kubernetes ServiceAccount | kubectl | Creates `sa-app-{env}` ServiceAccount in the namespace — this is the identity that Workload Identity will federate |
| 3. Federate ServiceAccount to Managed Identity | Azure CLI + kubectl | Runs `az identity federated-credential create` to link `sa-app-{env}` to `mi-app-{env}`. Uses OIDC issuer URL from L2 output. Annotates the ServiceAccount with `azure.workload.identity/client-id` |
| 4. Apply NetworkPolicy — default deny | kubectl | Applies explicit default-deny NetworkPolicy (in addition to Kyverno auto-generated one — belt and braces) |
| 5. Apply NetworkPolicy — allow rules | kubectl | Opens specific allow rules: pods → PostgreSQL VMs on port 5432, pods → RabbitMQ on port 5672, pods → Key Vault Private Endpoint on port 443 |
| 6. Apply ResourceQuota | kubectl | Sets namespace-level CPU, memory, and PVC count limits appropriate to the environment tier |
| 7. Apply LimitRange | kubectl | Sets default requests and limits for containers that do not declare them — Kyverno will also block missing limits, but LimitRange provides a safety net |
| 8. Create SecretProviderClass | kubectl | Applies `SecretProviderClass` manifest referencing the env-prefixed secrets in AKV. Uses `mi-app-{env}` client ID from L1 output. |
| 9. Create PVC | kubectl | Creates `app-data-{env}-pvc` PersistentVolumeClaim referencing `StorageClass: azurefile-csi-nfs` |
| 10. Create RoleBinding — pipeline SPN | kubectl | Binds `spn-app-{env}` to `namespace-deployer` Role in this namespace |
| 11. Create RoleBinding — dev team | kubectl | Binds `grp-dev-team` Entra group to `namespace-readonly` Role in this namespace |
| 12. Verify namespace health | kubectl | Confirms ServiceAccount exists, SecretProviderClass is valid, PVC is bound, RoleBindings are present |

### Pipeline identity

| Item | Value |
|---|---|
| SPN | `spn-k8s-baseline` (for namespace setup steps) then `spn-app-{env}` (for verification) |
| Azure role | `spn-infra-{sub}` needed for the federated credential creation step (Azure CLI) |
| K8s role | `cluster-admin` (setup), `namespace-deployer` (verify) |

### Key parameters

| Parameter | Description |
|---|---|
| `ENVIRONMENT` | Target environment name (e.g. `sandbox`, `intgtest`) |
| `AKV_NAME` | Key Vault name from L1 output |
| `MI_APP_CLIENT_ID` | Managed Identity client ID for this env from L1 output |
| `OIDC_ISSUER_URL` | AKS OIDC issuer URL from L2 output |
| `POSTGRES_HOST_CONNSTR_SECRET` | AKV secret name for PostgreSQL connection string |
| `STORAGE_KEY_SECRET` | AKV secret name for Storage Account key |
| `RESOURCE_QUOTA_CPU` | CPU quota for namespace (e.g. `8`) |
| `RESOURCE_QUOTA_MEMORY` | Memory quota (e.g. `16Gi`) |
| `ENTRA_GROUP_DEV_TEAM_ID` | Entra group object ID for developer read access |

---

## 12. Layer 7 — Image Build and Push

**Repository:** `image-build-{app}`
**Purpose:** Build a container image from a vendor-supplied ZIP file, inject enterprise configuration, tag it, and push it to the subscription ACR. This pipeline runs independently of all Kubernetes pipelines — it only touches ACR.

### What it does

| Step | Tool | Action |
|---|---|---|
| 1. Receive vendor ZIP | Pipeline trigger / manual upload | Vendor delivers ZIP containing Maven build output and reference Helm charts to a pre-agreed Stash path |
| 2. Extract ZIP | Shell | Extracts ZIP contents to workspace |
| 3. Inject enterprise config | Shell + Docker | Copies enterprise CA certificates, proxy settings (`HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY`), and internal DNS resolver config into the Docker build context |
| 4. Build image | Docker | `docker build` using vendor-provided Dockerfile as base. Enterprise layer added via multi-stage build. |
| 5. Tag image | Shell | Tags as `{acr-login-server}/{repo}:{semver}-{git-sha}` |
| 6. Authenticate to ACR | Azure CLI | `az acr login` using `spn-image-push-{sub}` |
| 7. Push to ACR | Docker | `docker push` via Private Endpoint |
| 8. Scan image | Azure CLI | `az acr repository show-manifests` — confirm push succeeded. Optional: trigger Defender for Containers vulnerability scan. |
| 9. Archive image tag | CloudBees | Writes the full image tag to a CloudBees artefact for downstream pipelines to reference |
| 10. Trigger L8 (optional) | CloudBees | If `AUTO_DEPLOY=true` and target environment is dev/test, triggers the app-deploy pipeline with the new image tag |

### Pipeline identity

| Item | Value |
|---|---|
| SPN | `spn-image-push-{sub}` |
| Azure role | `AcrPush` on `{sub}acr001` (resource-level scope) |
| K8s access | None |

### Image tag format

```
{acr-login-server}/{repository-name}:{major}.{minor}.{patch}-{short-git-sha}

Example:
devacr001.azurecr.io/aci-upf/app:5.3.1-a4f9c2d
```

The same tag is preserved during promotion. A prod image has the same semver and SHA as the dev image it was promoted from.

---

## 13. Layer 8 — Application Deployment

**Repository:** `app-deploy-{app}`
**Purpose:** Deploy the application Helm release to a target environment namespace. The Helm chart is maintained by the platform team — vendor reference charts are never deployed directly.

### What it does

| Step | Tool | Action |
|---|---|---|
| 1. Retrieve kubeconfig | CloudBees credential | Loads `kubeconfig-{sub}` for the target cluster |
| 2. Set image tag | Shell | Reads image tag from pipeline parameter or artefact from L7 |
| 3. Lint Helm chart | Helm | `helm lint` with target values file — fails pipeline on any error |
| 4. Dry run | Helm | `helm upgrade --dry-run` — validates K8s manifest output before applying |
| 5. Deploy | Helm | `helm upgrade --install {release-name} ./chart -n app-{env} -f values-{env}.yaml --set image.tag={IMAGE_TAG} --atomic --timeout 8m` |
| 6. Verify rollout | kubectl | `kubectl rollout status deployment/{app} -n app-{env} --timeout=5m` |
| 7. Smoke test | Shell / curl | Runs a basic health check against the application readiness endpoint inside the cluster |
| 8. Record deployment | CloudBees | Archives Helm release manifest as a deployment artefact |

### Deployment trigger per environment

| Environment | Trigger | Approval required | SPN |
|---|---|---|---|
| `app-sandbox` | Manual or PR open | None | `spn-app-sandbox` |
| `app-dev` | Automatic — feature/* branch build | None | `spn-app-dev` |
| `app-intgtest` | Automatic — develop branch merge | None | `spn-app-intgtest` |
| `app-systest` | Automatic — develop branch merge | None | `spn-app-systest` |
| `app-uat` | Manual — release/* branch create | None | `spn-app-uat` |
| `app-perftest` | Manual — on demand | None | `spn-app-perftest` |
| `app-prod` | Manual — main branch | **Human approval required** | `spn-app-prod` |

### Rollback procedure

```bash
# Automatic rollback — triggered if --atomic flag detects a failed rollout
# Manual rollback if needed after a successful but bad deployment:

helm rollback {release-name} {previous-revision} -n app-{env}

# Verify
kubectl rollout status deployment/{app} -n app-{env}
```

### Helm values file structure

```
app-deploy-{app}/
├── chart/
│   ├── Chart.yaml
│   ├── templates/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── serviceaccount.yaml
│   │   ├── secretproviderclass.yaml
│   │   ├── pvc.yaml
│   │   └── networkpolicy.yaml
├── values-sandbox.yaml
├── values-dev.yaml
├── values-intgtest.yaml
├── values-systest.yaml
├── values-uat.yaml
├── values-perftest.yaml
└── values-prod.yaml
```

---

## 14. Image Promotion Pipeline

**Repository:** `image-promote`
**Purpose:** Promote a validated image from one subscription's ACR to the next. Runs after environment sign-off (UAT → Prod promotion requires explicit approval). No rebuild — the image is pulled, re-tagged for the destination registry, and pushed.

### What it does

| Step | Tool | Action |
|---|---|---|
| 1. Authenticate to source ACR | Azure CLI | `az acr login` using `spn-image-promote` (AcrPull on source) |
| 2. Pull image | Docker | Pull `{source-acr}/{repo}:{tag}` |
| 3. Re-tag for destination | Docker | `docker tag {source-acr}/{repo}:{tag} {dest-acr}/{repo}:{tag}` — same semver and SHA, different registry prefix |
| 4. Authenticate to destination ACR | Azure CLI | `az acr login` using `spn-image-promote` (AcrPush on dest) |
| 5. Push to destination | Docker | `docker push {dest-acr}/{repo}:{tag}` |
| 6. Verify | Azure CLI | `az acr repository show-tags` — confirms tag exists in destination |
| 7. Trigger downstream deploy | CloudBees | Triggers L8 app-deploy for the destination environment |

### Promotion path

```
Dev ACR  ──────────────────►  Test ACR  ──────────────────►  Prod ACR
devacr001.azurecr.io          testacr001.azurecr.io           prodacr001.azurecr.io
(after dev/intgtest pass)      (after UAT sign-off)            (after prod approval gate)
```

---

## 15. Pipeline Dependency Order — Full Stack

The table below shows the complete dependency chain for bringing up a new environment from scratch.

```
PRE-CONDITIONS (Cloud Team)
  └─► Spoke subscription exists
  └─► VNet + 3 subnets exist
  └─► Private DNS zones exist
  └─► Master Managed Identity exists
  └─► CMK + CMK Managed Identity in shared subscription
        │
        ▼
L0  subscription-config          (once per subscription)
        │
        ▼
L1  common-resources             (once per subscription)
L1.1 app-resources               (once per environment)
        │
        ▼
L2  aks-cluster-provision        (once per cluster)
        │
        ▼
L2.1 AcrPull assignment          (Cloud Team — once per cluster)
        │
        ▼
L3  cluster-bootstrap            (once per cluster)
        │
        ▼
L4  cluster-operators            (once per cluster, re-run per operator upgrade)
        │
        ▼
L5  cluster-baseline             (once per cluster, re-run on RBAC changes)
        │
        ▼
L6  app-namespace-config         (once per environment namespace)
        │
        ▼
L7  image-build-push             (per release — runs independently)
        │
        ▼
L8  app-deploy                   (per release per environment)
```

---

## 16. SPN Summary — Which Pipeline Uses Which

| Pipeline | SPN | Azure scope | K8s scope |
|---|---|---|---|
| L0 subscription-config | `spn-infra-{sub}` | Subscription-level settings | None |
| L1 common-resources | `spn-infra-{sub}` | `rg-common-{sub}` | None |
| L1.1 app-resources | `spn-infra-{sub}` | `rg-app-{env}` | None |
| L2 aks-cluster-provision | `spn-cluster-{sub}` | `rg-aks-{sub}`, `rg-network-{sub}` | None |
| L2.1 AcrPull assignment | Cloud Team Owner | ACR resource | None |
| L3 cluster-bootstrap | `spn-k8s-baseline` | None | `cluster-admin` |
| L4 cluster-operators | `spn-k8s-baseline` | None | `cluster-admin` |
| L5 cluster-baseline | `spn-k8s-baseline` | None | `cluster-admin` |
| L6 app-namespace-config | `spn-k8s-baseline` + `spn-infra-{sub}` (federated cred step) | Federated credential creation only | `cluster-admin` |
| L7 image-build-push | `spn-image-push-{sub}` | AcrPush on ACR | None |
| L8 app-deploy | `spn-app-{env}` | None | `namespace-deployer` in `app-{env}` |
| Image promotion | `spn-image-promote` | AcrPull (source) + AcrPush (dest) | None |

---

## 17. Terraform Module Strategy

| Resource | Module type | Rationale |
|---|---|---|
| Resource groups | WNZL internal module | Simple — internal module enforces mandatory tags and naming convention |
| ACR | WNZL internal module | Internal module applies organisation-specific defaults (no public access, diagnostic settings) |
| Key Vault | WNZL internal module | Internal module enforces RBAC model, soft delete, purge protection by default |
| Private Endpoints | WNZL internal module | Internal module handles DNS registration pattern consistently |
| Managed Identities | WNZL internal module | Straightforward resource — internal module adds tagging and output consistency |
| Storage Accounts (SFTP) | WNZL internal module | Internal module enforces CMK encryption, ZRS, public access disabled |
| Virtual Machines (PostgreSQL) | WNZL internal module | Internal module enforces no public IP, managed disk, OS hardening defaults |
| **AKS Cluster** | **AVM** `avm-res-containerservice-managedcluster` | AKS has significant configuration complexity. AVM is Microsoft-maintained, well-tested, and handles the private cluster, node pool, Workload Identity, and OIDC configuration cleanly. |
| Budget alerts | WNZL internal module | Enforces organisation budget alert naming and notification group |
| Diagnostic settings | WNZL internal module | Enforces consistent log category selection and retention policy |

---

## 18. Key Principles for Pipeline Authors

1. **Never hardcode secrets.** All credentials come from CloudBees Credentials Store or Azure Key Vault. No passwords, client secrets, or connection strings in Jenkinsfiles or Terraform files.

2. **Always pin versions.** Helm chart versions, Terraform module versions, and provider versions must be explicitly set in every pipeline. No `latest` tags.

3. **Fail fast, fail loud.** Pipelines should fail at the first sign of trouble. Use `--atomic` on Helm deploys, use `set -e` in shell steps, check TF plan output before apply.

4. **Dry run before you apply.** Every Terraform pipeline runs `terraform plan` and archives the plan file as an artefact before `terraform apply`. Every Helm pipeline runs `--dry-run` before the real deploy.

5. **Every pipeline is re-runnable.** Infrastructure pipelines use Terraform state — re-running them is safe. Helm pipelines use `upgrade --install` — re-running them is safe. Namespace config pipelines use `kubectl apply` — re-running them is safe.

6. **No pipeline bypasses the dependency order.** Do not trigger L8 if L6 has not completed. Do not trigger L3 if L2.1 has not been confirmed. The dependency order exists to prevent partial deployments.

---

*AKS Platform — Pipeline Architecture · v1.0 Draft · Internal — Restricted · Platform Engineering*
