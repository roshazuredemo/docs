# Vendor AKS Compatibility Questionnaire

**Purpose:** To understand how the vendor has built and configured the application's target AKS environment so we can provision a compatible platform and avoid deployment blockers.

**Instructions for vendor:** Please complete each table below. Where a value is not applicable, write N/A. Where a value is flexible, write *flexible* and we will align during onboarding. Attach any existing Helm values files, deployment manifests, or architecture documents where relevant.

---

## 1. Kubernetes Cluster Configuration

These questions confirm the base cluster setup the application was built and tested against.

| # | Question | Vendor Answer |
|---|---|---|
| 1.1 | What Kubernetes version was the application built and tested on? (e.g. 1.28.x) | |
| 1.2 | Is the target cluster a private cluster (no public API server endpoint)? | |
| 1.3 | Is Kubernetes RBAC enabled on the cluster? | |
| 1.4 | Is the cluster integrated with an identity provider (e.g. Entra ID / Azure AD) for authentication? | |
| 1.5 | What is the cluster's Kubernetes API server authorisation mode? (e.g. RBAC, ABAC, Webhook) | |
| 1.6 | Are any admission webhooks registered on the cluster? If yes, list them. | |
| 1.7 | Are any Pod Security Standards or Pod Security Admission policies enforced? If yes, at what level — privileged, baseline, or restricted? | |
| 1.8 | Does the application require any cluster-level resources to be created (e.g. CRDs, ClusterRoles, ClusterRoleBindings, MutatingWebhookConfigurations)? | |
| 1.9 | What Kubernetes feature gates, if any, are enabled or disabled on the cluster? | |
| 1.10 | Does the application require alpha or beta Kubernetes features? | |

---

## 2. Node Pool Configuration

These questions confirm the compute environment the application expects to run on.

| # | Question | Vendor Answer |
|---|---|---|
| 2.1 | How many node pools does the application require? What is the purpose of each? | |
| 2.2 | What VM SKU / instance type is recommended for the application node pool? | |
| 2.3 | What is the recommended minimum and maximum node count per pool? | |
| 2.4 | What OS is used on nodes — Linux (which distro) or Windows? | |
| 2.5 | What OS disk type and size are used? (e.g. managed premium SSD, 128 GB) | |
| 2.6 | Are there any required node labels the application pods rely on for scheduling? If yes, list them. | |
| 2.7 | Are there any required node taints configured? If yes, list the key, value, and effect for each. | |
| 2.8 | Do application pods declare tolerations to match those taints? | |
| 2.9 | Does the application use node affinity or pod affinity / anti-affinity rules? If yes, provide the rules. | |
| 2.10 | Does the application require any specific kernel parameters or sysctl settings on nodes? | |
| 2.11 | Does the application require GPU nodes or any specialised hardware? | |

---

## 3. Networking Configuration

Networking mismatches are a common source of deployment failures. These questions confirm the network model the application was tested on.

| # | Question | Vendor Answer |
|---|---|---|
| 3.1 | What CNI plugin was used in the test environment? (e.g. Azure CNI, Azure CNI Overlay, Kubenet, Cilium) | |
| 3.2 | What pod CIDR range was used? | |
| 3.3 | What service CIDR range was used? | |
| 3.4 | What DNS service IP was configured? | |
| 3.5 | Is a network policy engine required? If yes, which one — Azure Network Policy, Calico, or Cilium? | |
| 3.6 | Does the application ship with NetworkPolicy manifests? If yes, please attach them. | |
| 3.7 | What ports does the application expose per service? List service name, port, and protocol for each. | |
| 3.8 | Does any component require host networking (`hostNetwork: true`)? | |
| 3.9 | Does any component require host port mappings (`hostPort`)? | |
| 3.10 | Does the application require an Ingress controller? If yes, which one and what annotations does it rely on? | |
| 3.11 | Does the application use any LoadBalancer type services? If yes, are they internal or external load balancers? | |
| 3.12 | Does the application require specific DNS configuration beyond cluster default (e.g. custom search domains, external DNS entries)? | |
| 3.13 | Does the application make any egress calls to external systems? If yes, list the destinations, ports, and protocols. | |
| 3.14 | Are there any IP allowlisting requirements either inbound or outbound? | |

---

## 4. Service Mesh Configuration

| # | Question | Vendor Answer |
|---|---|---|
| 4.1 | Does the application require a service mesh? If yes, which one? (e.g. Istio, Linkerd, Consul) | |
| 4.2 | If Istio — what version was used in the test environment? | |
| 4.3 | What Istio installation profile was used? (e.g. default, minimal, demo) | |
| 4.4 | Is mTLS required between services? If yes, in what mode — STRICT or PERMISSIVE? | |
| 4.5 | Does the application require sidecar injection to be enabled on specific namespaces? If yes, which namespaces? | |
| 4.6 | Are there any PeerAuthentication, DestinationRule, or VirtualService manifests the application ships with? | |
| 4.7 | Does any component need to opt out of sidecar injection (`sidecar.istio.io/inject: "false"`)? | |
| 4.8 | Does the application expose any services through an Istio Ingress Gateway? If yes, provide the Gateway and VirtualService configuration. | |
| 4.9 | Are any Istio Egress Gateways required for external connectivity? | |
| 4.10 | Are there any known Istio version incompatibilities we should be aware of? | |

---

## 5. Storage Configuration

| # | Question | Vendor Answer |
|---|---|---|
| 5.1 | Does the application require persistent storage? If yes, describe the use case for each volume. | |
| 5.2 | What StorageClass names does the application reference in its PVC manifests? | |
| 5.3 | What access modes are required for each PVC? (ReadWriteOnce, ReadWriteMany, ReadOnlyMany) | |
| 5.4 | What storage capacity is requested for each PVC? | |
| 5.5 | What volume plugin or CSI driver is required? (e.g. azure-disk-csi, azure-file-csi, nfs) | |
| 5.6 | If file storage is used — what protocol is required? NFS v4.1 or SMB? | |
| 5.7 | Does the application write to shared storage from multiple pods simultaneously? (This determines if ReadWriteMany is needed.) | |
| 5.8 | Does the application require a specific mount path for each volume? | |
| 5.9 | Are there any file permission requirements on mounted volumes (e.g. specific UID/GID or chmod settings)? | |
| 5.10 | Does the application use any emptyDir, configMap, or secret volumes? If yes, describe them. | |
| 5.11 | Does the application use the Secrets Store CSI Driver to mount secrets from a vault? If yes, from which vault provider? | |

---

## 6. Security & Pod Configuration

These confirm whether the application can run under restricted security settings, which our platform enforces via Kyverno.

| # | Question | Vendor Answer |
|---|---|---|
| 6.1 | Do any containers run as root (UID 0)? | |
| 6.2 | Does any container require `privileged: true` in its security context? | |
| 6.3 | Does any container require `allowPrivilegeEscalation: true`? | |
| 6.4 | What UID and GID does each container run as? Can these be configured? | |
| 6.5 | Does any container require read-write access to the host filesystem (`hostPath` volumes)? | |
| 6.6 | Does any container require additional Linux capabilities beyond the default set? (e.g. `NET_ADMIN`, `SYS_PTRACE`) | |
| 6.7 | Is the root filesystem set to read-only (`readOnlyRootFilesystem: true`)? If not, is this something that can be enabled? | |
| 6.8 | Does any component use `hostPID: true` or `hostIPC: true`? | |
| 6.9 | Does any init container require elevated permissions? If yes, describe what it does and why. | |
| 6.10 | How are application secrets provided at runtime? (e.g. environment variables, mounted files, Kubernetes Secrets, vault integration) | |
| 6.11 | Does the application store any secrets in Kubernetes Secret objects directly? | |
| 6.12 | Does the application require any Kubernetes Secrets to be pre-created before deployment? If yes, list them. | |
| 6.13 | Are TLS certificates required by the application? If yes, how are they provisioned and rotated? | |
| 6.14 | Does the application use Workload Identity or pod-managed identity to access cloud resources? | |

---

## 7. Container Images

| # | Question | Vendor Answer |
|---|---|---|
| 7.1 | Where are the container images hosted? (e.g. Docker Hub, private registry, vendor registry) | |
| 7.2 | What is the base OS image used for each component? (e.g. Ubuntu 22.04, Alpine 3.18, Red Hat UBI 9) | |
| 7.3 | Does the application pull any images from public registries at runtime (not just at build time)? | |
| 7.4 | Are there any sidecar or init container images that are pulled from a different registry? | |
| 7.5 | Are the images provided as a tarball / OCI archive that can be loaded into a private registry, or only available via pull from vendor registry? | |
| 7.6 | Do images support multi-architecture? (e.g. amd64 and arm64) What architecture is the application tested on? | |
| 7.7 | Are there any image pull secrets required for any component? | |
| 7.8 | Do images contain any hardcoded registry references (e.g. in Helm charts, operator bundles) that would need to be overridden to use our private ACR? | |
| 7.9 | What is the image tag strategy? Are tags immutable once published? | |
| 7.10 | Are image vulnerability scans available for the provided images? | |

---

## 8. Resource Requirements

| # | Question | Vendor Answer |
|---|---|---|
| 8.1 | What CPU and memory requests are configured for each component? | |
| 8.2 | What CPU and memory limits are configured for each component? | |
| 8.3 | Are resource requests and limits mandatory for all containers, or are any left unset? | |
| 8.4 | Does the application use Horizontal Pod Autoscaler (HPA)? If yes, what metric and thresholds are configured? | |
| 8.5 | Does the application use KEDA for event-driven autoscaling? If yes, what ScaledObject configuration is used? | |
| 8.6 | What is the minimum and maximum replica count for each component? | |
| 8.7 | Are PodDisruptionBudgets (PDBs) configured? If yes, what are the minAvailable or maxUnavailable values? | |
| 8.8 | Does the application have any specific Quality of Service (QoS) class requirements? (Guaranteed, Burstable, BestEffort) | |

---

## 9. Helm Chart & Deployment Configuration

| # | Question | Vendor Answer |
|---|---|---|
| 9.1 | Is the application packaged as a Helm chart? If yes, what chart version is current? | |
| 9.2 | Can the Helm chart be hosted in our private Helm repository, or must it be pulled from a vendor-hosted repository? | |
| 9.3 | What are the mandatory Helm values that must be set for a minimal working deployment? | |
| 9.4 | Are there any values that reference specific namespace names, cluster names, or domain names that we will need to override? | |
| 9.5 | Does the Helm chart create any ClusterRole or ClusterRoleBinding resources? If yes, list them. | |
| 9.6 | Does the Helm chart include any ServiceAccount resources? If yes, are they configurable? | |
| 9.7 | Does the chart include pre-install or post-install Helm hooks? If yes, describe what they do. | |
| 9.8 | Does the chart rely on sub-charts (chart dependencies)? If yes, list them with versions. | |
| 9.9 | Is there a recommended deployment order for components? (e.g. database before app server) | |
| 9.10 | Does the application support blue-green or canary deployment strategies, or is rolling update the only supported strategy? | |

---

## 10. Operator & CRD Dependencies

| # | Question | Vendor Answer |
|---|---|---|
| 10.1 | Does the application require any Kubernetes operators to be pre-installed on the cluster? If yes, list each operator, version, and namespace. | |
| 10.2 | Does the application install its own operator? If yes, what CRDs does it register? | |
| 10.3 | Are any of the required operators available as community operators (OperatorHub) or are they vendor-proprietary? | |
| 10.4 | Does the application depend on cert-manager? If yes, what version and what Issuer / ClusterIssuer type is expected? | |
| 10.5 | Does the application depend on a specific RabbitMQ version or RabbitMQ Operator version? | |
| 10.6 | Does the application depend on any external secret management operators (e.g. External Secrets Operator)? | |
| 10.7 | Does the application ship or depend on any Custom Resource Definitions (CRDs) that must exist before the Helm chart is installed? | |
| 10.8 | Are there any known CRD version conflicts with commonly used operators (e.g. cert-manager, Prometheus Operator)? | |

---

## 11. Application Health & Probes

| # | Question | Vendor Answer |
|---|---|---|
| 11.1 | Is a readinessProbe configured for each component? If yes, what endpoint, port, and initial delay are used? | |
| 11.2 | Is a livenessProbe configured? If yes, what endpoint, port, and thresholds are used? | |
| 11.3 | Is a startupProbe configured for slow-starting components? | |
| 11.4 | What is the typical pod startup time from container start to ready state under normal conditions? | |
| 11.5 | How does the application handle graceful shutdown? Is a `terminationGracePeriodSeconds` value specified? | |
| 11.6 | Are there any known scenarios that cause pods to enter a CrashLoopBackOff? What is the recommended troubleshooting step? | |

---

## 12. Observability & Logging

| # | Question | Vendor Answer |
|---|---|---|
| 12.1 | Does the application expose a Prometheus metrics endpoint? If yes, on what path and port? | |
| 12.2 | Does the application include a ServiceMonitor or PodMonitor resource for Prometheus Operator? | |
| 12.3 | What log format does the application write to stdout / stderr? (e.g. JSON structured, plain text) | |
| 12.4 | Does the application write logs to files on disk rather than stdout? If yes, where and can this be changed? | |
| 12.5 | Does the application support OpenTelemetry for traces and metrics? If yes, what exporter protocol is used? (OTLP gRPC, OTLP HTTP, Jaeger) | |
| 12.6 | Are there any application-specific dashboards (e.g. Grafana dashboards as ConfigMaps) provided? | |
| 12.7 | Are there any application-specific Prometheus alerting rules provided? | |
| 12.8 | Does the application log any sensitive data (credentials, PII) that would need to be masked before forwarding to a log aggregator? | |

---

## 13. Configuration & Environment Variables

| # | Question | Vendor Answer |
|---|---|---|
| 13.1 | What environment variables are required for the application to start? List each variable, its purpose, and whether it is mandatory or optional. | |
| 13.2 | Are environment variable values provided via Helm values, Kubernetes ConfigMaps, Kubernetes Secrets, or a combination? | |
| 13.3 | Are there any environment-specific configuration files the application reads from disk? If yes, how are they provided — ConfigMap mount, init container, or baked into the image? | |
| 13.4 | Does the application support a configuration reload without pod restart (e.g. watching a mounted ConfigMap for changes)? | |
| 13.5 | Are there any database migration steps that must run before the application starts? How are they triggered? | |
| 13.6 | Does the application require any pre-seeded data in the database before first start? If yes, how is that data loaded? | |

---

## 14. Multi-Environment & Namespace Compatibility

| # | Question | Vendor Answer |
|---|---|---|
| 14.1 | Is the application designed to run in multiple namespaces within the same cluster simultaneously? (We run sandbox, dev, intgtest, systest etc. in the same cluster.) | |
| 14.2 | Are there any cluster-scoped resources (ClusterRole, CRD, webhook) that would conflict if the application is installed in more than one namespace? | |
| 14.3 | Does the application use any hardcoded namespace references in its code or configuration? | |
| 14.4 | Can the application be configured to connect to environment-specific databases and file shares via Helm values, or is the connection target hardcoded? | |
| 14.5 | Is it possible to run a scaled-down version of the application (e.g. single replica, smaller DB) in lower environments while keeping the same Helm chart? | |
| 14.6 | Have you tested the application behind a corporate proxy? Are proxy settings (HTTP_PROXY, HTTPS_PROXY, NO_PROXY) configurable? | |

---

## 15. Known Constraints & Gotchas

These questions help us avoid surprises during onboarding.

| # | Question | Vendor Answer |
|---|---|---|
| 15.1 | Are there any known issues or limitations when running the application on AKS specifically (as opposed to other Kubernetes distributions)? | |
| 15.2 | Are there any Azure-specific configurations or Azure services the application depends on that are not covered above? | |
| 15.3 | Have you tested the application with Azure CNI specifically? Any known CNI compatibility issues? | |
| 15.4 | Does the application have any known compatibility issues with the versions of Istio, Kyverno, or cert-manager we are running? | |
| 15.5 | Are there any Kyverno or OPA / Gatekeeper policies that are known to block the application? (e.g. does it require privileged mode or a specific UID?) | |
| 15.6 | Are there any minimum or maximum node count requirements below which the application becomes unstable or fails? | |
| 15.7 | Are there any known resource contention issues under peak load? What is the recommended cluster headroom? | |
| 15.8 | Are there any licensing considerations that affect how many replicas or instances can run simultaneously? | |
| 15.9 | Is there a recommended deployment runbook or installation guide we should follow? Please attach if available. | |
| 15.10 | Who is the technical contact on your side for deployment support during our onboarding? | |

---

*AKS Platform — Vendor Compatibility Questionnaire · v1.0 · Platform Engineering*
