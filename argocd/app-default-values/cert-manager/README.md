# cert-manager

Deploys cert-manager into the `cert-manager` namespace for automated TLS certificate provisioning via Let's Encrypt. Uses DNS-01 ACME challenges — Route53 on AWS (IRSA) or Cloud DNS on GCP (Workload Identity). Synced at **wave 0** so CRDs and the webhook are ready before the wave-1 `cert-manager-issuers` app applies ClusterIssuers.

## Architecture

```
Gateway (cert-manager.io/cluster-issuer annotation)
  └── cert-manager controller
        └── ClusterIssuer (letsencrypt-staging or letsencrypt-prod)
              └── ACME DNS-01 challenge
                    AWS:  Route53 TXT record (via IRSA)
                    GCP:  Cloud DNS TXT record (via Workload Identity)
                      └── Let's Encrypt validates → issues cert
                            └── Stored as K8s Secret → TLS termination at Envoy Gateway
```

## Components

| Pod | Description |
|-----|-------------|
| cert-manager (controller) | Watches Certificate/Gateway resources, drives ACME flow |
| cert-manager-webhook | Validates/mutates cert-manager CRDs |
| cert-manager-cainjector | Injects CA bundles into webhook configurations |

## ClusterIssuers

Two ClusterIssuers are deployed per environment by the `cert-manager-issuers` ArgoCD app (wave 1):

| Name | ACME server | Use |
|------|-------------|-----|
| `letsencrypt-staging` | `acme-staging-v02.api.letsencrypt.org` | Testing — issues untrusted certs, no rate limits |
| `letsencrypt-prod` | `acme-v02.api.letsencrypt.org` | Production — browser-trusted certs |

Use `letsencrypt-staging` during initial setup to avoid Let's Encrypt rate limits. Switch to `letsencrypt-prod` only when the TLS flow is confirmed working end-to-end.

## Files

```
app-default-values/cert-manager/
└── values/
    └── base.yaml                         ← shared Helm values (all envs)

environments/<env>/cert-manager/
├── values.yaml                           ← IRSA role ARN (AWS) or Workload Identity GSA (GCP)
└── issuers/
    ├── clusterissuer-staging.yaml        ← letsencrypt-staging ClusterIssuer
    └── clusterissuer-prod.yaml           ← letsencrypt-prod ClusterIssuer
```

The `cert-manager-issuers` ArgoCD app (wave 1) deploys everything under `environments/<env>/cert-manager/issuers/` as raw manifests.

## Configuration

### Base values (`values/base.yaml`)

- `installCRDs: true` — CRDs installed with the chart, no separate CRD job needed
- Resource requests/limits set for controller, webhook, and cainjector
- `global.leaderElection.namespace: cert-manager` — scoped leader election

### Per-environment Helm values (`environments/<env>/cert-manager/values.yaml`)

**AWS (EKS):**
```yaml
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/cert-manager-<env>
```

**GCP (GKE):**
```yaml
serviceAccount:
  annotations:
    iam.gke.io/gcp-service-account: cert-manager-<env>@<GCP_PROJECT_ID>.iam.gserviceaccount.com
```

### ClusterIssuer manifests (`environments/<env>/cert-manager/issuers/`)

Each issuer file is pre-filled for AWS (Route53 solver) with the GCP alternative (Cloud DNS solver) shown as a comment. Edit in place when targeting GCP:

**AWS solver (default):**
```yaml
solvers:
  - dns01:
      route53:
        region: <AWS_REGION>
        hostedZoneID: <ROUTE53_HOSTED_ZONE_ID>
```

**GCP solver (replace the above):**
```yaml
solvers:
  - dns01:
      cloudDNS:
        project: <GCP_PROJECT_ID>
```

### Placeholders to fill

| Placeholder | File | Description |
|-------------|------|-------------|
| `<ACME_EMAIL>` | Both issuer files per env | Email for Let's Encrypt expiry notifications |
| `<AWS_REGION>` | Both issuer files per env | AWS region of the Route53 hosted zone |
| `<ROUTE53_HOSTED_ZONE_ID>` | Both issuer files per env | Route53 hosted zone ID (e.g. `Z1234ABCD`) |
| `<GCP_PROJECT_ID>` | Both issuer files per env (GCP only) | GCP project ID |

## IAM Pre-requisites

### AWS (EKS)

Create role `cert-manager-<env>` trusted by the `cert-manager/cert-manager` ServiceAccount via IRSA:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:GetChange",
        "route53:ChangeResourceRecordSets",
        "route53:ListHostedZones",
        "route53:ListResourceRecordSets",
        "route53:ListTagsForResource"
      ],
      "Resource": "*"
    }
  ]
}
```

### GCP (GKE)

Create GCP service account `cert-manager-<env>` with `roles/dns.admin`, then bind it to the `cert-manager/cert-manager` Kubernetes ServiceAccount via Workload Identity:

```bash
gcloud iam service-accounts add-iam-policy-binding \
  cert-manager-<env>@<GCP_PROJECT_ID>.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:<GCP_PROJECT_ID>.svc.id.goog[cert-manager/cert-manager]"
```

## Usage

### Annotate a Gateway for automatic TLS

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-app-gateway
  namespace: my-app
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod   # or letsencrypt-staging
spec:
  gatewayClassName: envoy
  listeners:
    - name: https
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - name: my-app-tls
```

cert-manager detects the annotation, creates a Certificate resource, runs the DNS-01 challenge, and stores the issued cert in the `my-app-tls` Secret.

### Verify a certificate

```bash
kubectl get certificate -n <namespace>
kubectl describe certificate my-app-tls -n <namespace>
```

`READY=True` means the cert is issued and stored in the referenced Secret.
