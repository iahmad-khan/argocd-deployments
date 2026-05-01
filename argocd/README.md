# ArgoCD App-of-Apps

GitOps configuration for a multi-environment Kubernetes platform using ArgoCD's App-of-Apps pattern. Manages Vault, External Secrets Operator, monitoring (kube-prometheus-stack + Thanos), Envoy Gateway (Gateway API), cert-manager, external-dns, Langfuse, and a sample workload across `dev`, `uat`, and `prod` clusters. Portable across **EKS (AWS)** and **GKE (GCP)** — cloud-specific config lives in `environments/<env>/` and is edited in place when switching clouds.

## Architecture

```
root-apps/{dev,uat,prod}.yaml          ← manually applied once per cluster
        │
        └── apps/ (Helm chart)         ← renders all child ArgoCD Applications
                │
                ├── vault
                ├── eso (External Secrets Operator)
                ├── vault-config
                ├── eso-secretstore
                ├── monitoring (kube-prometheus-stack)
                ├── thanos
                ├── vault-monitoring
                ├── eso-monitoring
                ├── k8s-monitoring
                ├── cert-manager
                ├── cert-manager-issuers
                ├── external-dns
                ├── envoy-gateway
                ├── envoy-gateway-config
                ├── langfuse
                └── hello-world
```

### Value resolution

Two source patterns are used depending on the app type:

**Helm apps** — base values merged with per-env overrides:
```
app-default-values/<app>/values/base.yaml   ← shared Helm values
environments/<env>/<app>/values.yaml        ← environment overrides (merged on top)
```

**Directory-source apps** — raw manifests deployed directly from a Git path:
```
app-default-values/<app>/config/            ← common manifests (e.g. GatewayClass)
environments/<env>/<app>/                   ← env-specific manifests (e.g. EnvoyProxy, ClusterIssuers)
```

---

## Sync Wave Order

| Wave | Applications |
|------|-------------|
| 0 | `vault`, `eso`, `monitoring`, `cert-manager`, `external-dns`, `envoy-gateway` |
| 1 | `vault-config`, `thanos`, `cert-manager-issuers`, `envoy-gateway-config` |
| 2 | `eso-secretstore` |
| 3 | `vault-monitoring`, `eso-monitoring`, `k8s-monitoring`, `langfuse` |
| 4 | `hello-world` |

- `monitoring` at wave 0: Prometheus Operator CRDs (PrometheusRule, ServiceMonitor, PodMonitor) must exist before the wave-3 monitoring apps apply them.
- `cert-manager` at wave 0: CRDs and webhook must exist before `cert-manager-issuers` (wave 1) applies ClusterIssuers.
- `envoy-gateway` at wave 0: Gateway API CRDs must exist before `envoy-gateway-config` (wave 1) applies the GatewayClass and EnvoyProxy.

---

## Applications

| App | Chart / Source | Namespace | Description |
|-----|---------------|-----------|-------------|
| vault | hashicorp/vault 0.30.1 | vault | HA Vault with Raft, AWS KMS auto-unseal, IRSA |
| eso | external-secrets 0.9.13 | external-secrets | Syncs Vault secrets to Kubernetes Secrets |
| vault-config | directory: `app-default-values/vault/manifests/<env>` | vault | Post-sync Job: init, auth, policies |
| eso-secretstore | directory: `app-default-values/eso/secretstore` | external-secrets | ClusterSecretStore pointing to Vault |
| monitoring | kube-prometheus-stack 67.5.0 | monitoring | Prometheus + Grafana + AlertManager + node-exporter + kube-state-metrics |
| thanos | bitnami/thanos 15.7.0 | monitoring | Long-term metrics storage via S3 (AWS) or GCS (GCP) |
| vault-monitoring | directory: `app-default-values/vault/monitoring` | vault | PodMonitor + PrometheusRules + Grafana dashboard |
| eso-monitoring | directory: `app-default-values/eso/monitoring` | external-secrets | ServiceMonitor + PrometheusRules + Grafana dashboard |
| k8s-monitoring | directory: `app-default-values/k8s-monitoring` | monitoring | Kubernetes + Prometheus self-monitoring rules, Grafana dashboard, AlertManager ExternalSecret |
| cert-manager | jetstack/cert-manager v1.14.5 | cert-manager | Automated TLS via Let's Encrypt DNS-01 (Route53 on AWS, Cloud DNS on GCP) |
| cert-manager-issuers | directory: `environments/<env>/cert-manager/issuers` | cert-manager | Per-env ClusterIssuers (`letsencrypt-staging`, `letsencrypt-prod`) |
| external-dns | kubernetes-sigs/external-dns 1.14.4 | external-dns | Automatic DNS records from HTTPRoute/Ingress/Service hostnames (Route53 or Cloud DNS) |
| envoy-gateway | gateway.envoyproxy.io/gateway-helm v1.2.3 | envoy-gateway-system | Gateway API controller — replaces Ingress for L7 routing |
| envoy-gateway-config | multi-source: common `GatewayClass` + env `EnvoyProxy` | envoy-gateway-system | GatewayClass (shared) + cloud-specific LoadBalancer annotations (NLB on AWS, External LB on GCP) |
| langfuse | langfuse 1.3.1 | langfuse | LLM observability platform |
| hello-world | local Helm chart | hello-world | Sample workload with Vault secret injection |

---

## Repository Layout

```
argocd/
├── root-apps/                          # One Application per environment — apply once to bootstrap
│   ├── dev.yaml
│   ├── uat.yaml
│   └── prod.yaml
│
├── apps/                               # Helm chart that renders all child ArgoCD Applications
│   ├── Chart.yaml
│   ├── values.yaml                     # App registry: enabled, syncWave, chartVersion, namespace
│   └── templates/                      # One Application template per child app
│
├── app-default-values/                 # Shared base config — no env-specific values here
│   ├── vault/
│   ├── eso/
│   ├── monitoring/                     # kube-prometheus-stack base Helm values
│   ├── thanos/                         # Thanos base Helm values
│   ├── k8s-monitoring/                 # PrometheusRules, Grafana dashboard, AlertManager ExternalSecret
│   ├── cert-manager/
│   │   └── values/base.yaml           # cert-manager Helm base values (no issuers here)
│   ├── external-dns/
│   │   └── values/base.yaml           # external-dns Helm base values (provider set per-env)
│   ├── envoy-gateway/
│   │   ├── values/base.yaml           # Envoy Gateway Helm base values
│   │   └── config/
│   │       └── gatewayclass.yaml      # Common GatewayClass (cloud-agnostic)
│   └── langfuse/
│
├── environments/                       # All env-specific and cloud-specific config
│   ├── dev/
│   │   ├── values.yaml                # env: dev
│   │   ├── vault/values.yaml
│   │   ├── monitoring/values.yaml
│   │   ├── thanos/values.yaml
│   │   ├── cert-manager/
│   │   │   ├── values.yaml            # IRSA ARN (AWS) or Workload Identity GSA (GCP)
│   │   │   └── issuers/               # ClusterIssuer manifests — edit solver for AWS vs GCP
│   │   │       ├── clusterissuer-staging.yaml
│   │   │       └── clusterissuer-prod.yaml
│   │   ├── external-dns/values.yaml   # provider, region/project, txtOwnerId, auth
│   │   └── envoy-gateway/
│   │       └── envoyproxy.yaml        # Cloud-specific LB annotations — edit for AWS vs GCP
│   ├── uat/                           # Same structure as dev/
│   └── prod/                          # Same structure as dev/
│
└── app-versions-{dev,uat,prod}.yaml   # Image tags per environment
```

---

## Bootstrap

### Prerequisites

- ArgoCD running in the target cluster
- Cluster registered in ArgoCD with a name matching the environment (`dev`, `uat`, or `prod`)
- **Vault**: AWS KMS key and IAM roles for auto-unseal and IRSA
- **Monitoring + Thanos**: S3 buckets and IAM roles — see [Monitoring README](app-default-values/monitoring/README.md)
- **AlertManager**: Slack incoming webhook URL and (prod only) PagerDuty Events API v2 routing key
- **cert-manager**: IAM role `cert-manager-<env>` with Route53 permissions (AWS) or GCP service account with `roles/dns.admin` (GCP) — see [cert-manager README](app-default-values/cert-manager/README.md)
- **external-dns**: IAM role `external-dns-<env>` with Route53 permissions (AWS) or GCP service account with `roles/dns.admin` (GCP) — see [external-dns README](app-default-values/external-dns/README.md)
- **Envoy Gateway on AWS**: AWS Load Balancer Controller deployed in the cluster — provisions NLBs from Service annotations. See [Envoy Gateway README](app-default-values/envoy-gateway/README.md)

---

### 1. Fill in placeholders

Replace all `<PLACEHOLDER>` values in `environments/` before committing:

| Placeholder | File(s) | Description |
|-------------|---------|-------------|
| `<YOUR_DOMAIN>` | All env values files | Base domain for hostnames |
| `<DEV/UAT/PROD_AWS_ACCOUNT_ID>` | All env values files (AWS) | AWS account ID per environment |
| `<AWS_REGION>` | monitoring, thanos, external-dns env values | AWS region |
| `<DEV/UAT/PROD_KMS_KEY_ARN>` | vault env values | KMS key ARN for Vault auto-unseal |
| `<GRAFANA_ADMIN_PASSWORD>` | monitoring env values | Grafana admin password |
| `<ACME_EMAIL>` | `environments/<env>/cert-manager/issuers/*.yaml` | Email for Let's Encrypt expiry notifications |
| `<ROUTE53_HOSTED_ZONE_ID>` | `environments/<env>/cert-manager/issuers/*.yaml` | Route53 hosted zone ID (e.g. `Z1234ABCD`) |

---

### 2. Configure for your cloud (AWS or GCP)

All cloud-specific config is in `environments/<env>/`. The files are pre-filled for AWS. When targeting GCP, edit these four files per environment:

| File | Change for GCP |
|------|----------------|
| `environments/<env>/cert-manager/values.yaml` | Replace IRSA ARN with `iam.gke.io/gcp-service-account` annotation |
| `environments/<env>/cert-manager/issuers/*.yaml` | Replace `route53` solver with `cloudDNS` (see comment in each file) |
| `environments/<env>/envoy-gateway/envoyproxy.yaml` | Replace AWS NLB annotations with `cloud.google.com/load-balancer-type: External` (see comment in the file) |
| `environments/<env>/external-dns/values.yaml` | Replace `provider: aws` + `aws.region` with `provider: google` + `google.project` |

---

### 3. Populate AlertManager secrets in Vault

The `k8s-monitoring` app creates a `alertmanager-credentials` K8s Secret via ESO from:

```
Vault path: secret/monitoring/alertmanager
  ├── slack_webhook_url
  └── pagerduty_routing_key
```

Populate before syncing:

```bash
vault kv put secret/monitoring/alertmanager \
  slack_webhook_url="https://hooks.slack.com/services/T.../B.../..." \
  pagerduty_routing_key="<KEY>"   # non-empty dummy is fine for dev/uat
```

| Environment | Slack channel | PagerDuty |
|-------------|--------------|-----------|
| dev | `#dev-alerts` | dummy value |
| uat | `#staging-alerts` | dummy value |
| prod | `#prod-alerts` | real Events API v2 key |

---

### 4. Apply the root Application

```bash
kubectl apply -f root-apps/dev.yaml    # dev cluster
kubectl apply -f root-apps/uat.yaml    # uat cluster
kubectl apply -f root-apps/prod.yaml   # prod cluster
```

ArgoCD creates all child Applications and syncs them in wave order.

---

### 5. Post-sync: Vault bootstrap

After wave-0 sync completes, create the bootstrap secret so the `vault-config` Job can run:

```bash
kubectl create secret generic vault-bootstrap-token \
  --from-literal=root-token=<VAULT_ROOT_TOKEN> \
  -n vault
```

---

## Enabling / Disabling Apps

Toggle an app for all environments in `apps/values.yaml`:

```yaml
apps:
  helloWorld:
    enabled: false
```

Override for a single environment in `environments/<env>/values.yaml`:

```yaml
apps:
  helloWorld:
    enabled: false
```

## Upgrading a Chart

Update `chartVersion` in `apps/values.yaml` (all environments) or in `environments/<env>/values.yaml` (one environment):

```yaml
apps:
  monitoring:
    chartVersion: "68.0.0"
```
