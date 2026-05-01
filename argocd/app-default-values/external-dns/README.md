# external-dns

Deploys external-dns into the `external-dns` namespace. Watches Ingress, Service, and **HTTPRoute** resources and automatically creates/updates DNS A/CNAME records. Supports Route53 (AWS/EKS via IRSA) and Cloud DNS (GCP/GKE via Workload Identity). Synced at **wave 0** alongside cert-manager and envoy-gateway.

## Architecture

```
HTTPRoute (hostname: my-app.prod.example.com)
  └── external-dns watches → creates DNS A record → my-app.prod.example.com
        └── TXT ownership record → prevents other clusters overwriting the record
```

## Key Configuration

| Setting | Where | Why |
|---------|-------|-----|
| `provider` | per-env values | `aws` for Route53, `google` for Cloud DNS — set per environment |
| `sources` | base values | `ingress`, `service`, `gateway-httproute` — picks up all hostname sources |
| `policy` | base values | `sync` — deletes stale records when resources are removed |
| `txtOwnerId` | per-env values | Unique per cluster, e.g. `dev-cluster` — prevents cross-cluster record conflicts |
| `domainFilters` | per-env values | Restricts external-dns to only the env's domain — avoids touching unrelated zones |

## Files

```
app-default-values/external-dns/
└── values/
    └── base.yaml                         ← shared Helm values (sources, policy, resource limits)

environments/<env>/external-dns/values.yaml   ← provider, region/project, txtOwnerId, domainFilter, auth
```

## Per-environment overrides

### AWS (EKS)

```yaml
provider: aws

aws:
  region: <AWS_REGION>

txtOwnerId: <env>-cluster

domainFilters:
  - <env>.<YOUR_DOMAIN>    # prod: <YOUR_DOMAIN>

serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/external-dns-<env>
```

### GCP (GKE)

Replace the env values file with:

```yaml
provider: google

google:
  project: <GCP_PROJECT_ID>

txtOwnerId: <env>-cluster

domainFilters:
  - <env>.<YOUR_DOMAIN>    # prod: <YOUR_DOMAIN>

serviceAccount:
  annotations:
    iam.gke.io/gcp-service-account: external-dns-<env>@<GCP_PROJECT_ID>.iam.gserviceaccount.com
```

### Domain filter per environment

| Environment | domainFilters |
|-------------|---------------|
| dev | `dev.<YOUR_DOMAIN>` |
| uat | `uat.<YOUR_DOMAIN>` |
| prod | `<YOUR_DOMAIN>` (apex, covers all subdomains) |

## IAM Pre-requisites

### AWS (EKS)

Create role `external-dns-<env>` trusted by the `external-dns/external-dns` ServiceAccount via IRSA:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["route53:ChangeResourceRecordSets"],
      "Resource": "arn:aws:route53:::hostedzone/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:ListResourceRecordSets",
        "route53:ListTagsForResources"
      ],
      "Resource": "*"
    }
  ]
}
```

### GCP (GKE)

Create GCP service account `external-dns-<env>` with `roles/dns.admin`, then bind it to the `external-dns/external-dns` Kubernetes ServiceAccount via Workload Identity:

```bash
gcloud iam service-accounts add-iam-policy-binding \
  external-dns-<env>@<GCP_PROJECT_ID>.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:<GCP_PROJECT_ID>.svc.id.goog[external-dns/external-dns]"
```

## Usage

### HTTPRoute (Gateway API) — primary

external-dns automatically picks up hostnames from HTTPRoute resources via the `gateway-httproute` source:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app
  namespace: my-app
spec:
  hostnames:
    - my-app.prod.example.com
  ...
```

A DNS A record pointing to the Envoy Gateway LoadBalancer IP is created within ~30 seconds.

### LoadBalancer Service — direct annotation

```yaml
metadata:
  annotations:
    external-dns.alpha.kubernetes.io/hostname: my-svc.prod.example.com
```

## Placeholders to Fill

| Placeholder | File | Description |
|-------------|------|-------------|
| `<AWS_REGION>` | AWS env values | AWS region |
| `<YOUR_DOMAIN>` | All env values | Base domain |
| `<DEV/UAT/PROD_AWS_ACCOUNT_ID>` | AWS env values | AWS account ID per environment |
| `<GCP_PROJECT_ID>` | GCP env values | GCP project ID |
