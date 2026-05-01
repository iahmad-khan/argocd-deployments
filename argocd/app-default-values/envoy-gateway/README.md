# Envoy Gateway

Deploys Envoy Gateway into the `envoy-gateway-system` namespace as the cluster's Gateway API controller. A companion app `envoy-gateway-config` (wave 1) applies the common `GatewayClass` and the env-specific `EnvoyProxy` CR once the CRDs are ready.

Envoy Gateway is chosen over other implementations (Contour, Traefik, nginx Gateway Fabric) because it is purpose-built for Gateway API — no legacy Ingress support to maintain, cleaner API surface, and strong upstream backing (Envoy project + Google/Microsoft/Tetrate).

## Architecture

```
HTTPRoute / GRPCRoute / TLSRoute
  │
  GatewayClass "envoy"
  │   └── parametersRef → EnvoyProxy CR (cloud-specific LB annotations)
  │
  Envoy Gateway controller (envoy-gateway-system)
        └── reconciles Gateway resources
              └── creates Envoy proxy Deployment + LoadBalancer Service
                    AWS:  NLB  (internet-facing, ip target type)
                    GCP:  External LB
```

## Wave Order

| Wave | App | Why |
|------|-----|-----|
| 0 | `envoy-gateway` | Installs Envoy Gateway controller + Gateway API CRDs |
| 1 | `envoy-gateway-config` | Applies GatewayClass + EnvoyProxy (needs CRDs from wave 0) |

## Files

```
app-default-values/envoy-gateway/
├── values/
│   └── base.yaml                         ← Helm values (controller config, CRD install)
└── config/
    └── gatewayclass.yaml                 ← common GatewayClass (cloud-agnostic)

environments/<env>/envoy-gateway/
└── envoyproxy.yaml                       ← env-specific EnvoyProxy (cloud-specific LB annotations)
```

The `envoy-gateway-config` ArgoCD app (wave 1) is a **multi-source** app: it merges manifests from `app-default-values/envoy-gateway/config/` (GatewayClass) and `environments/<env>/envoy-gateway/` (EnvoyProxy) into a single sync.

## Cloud-Specific Configuration

The `EnvoyProxy` CR in `environments/<env>/envoy-gateway/envoyproxy.yaml` controls what kind of LoadBalancer Service Envoy Gateway provisions. Edit this file directly when targeting a different cloud — the AWS annotations are active by default with the GCP alternative shown as a comment.

### AWS (EKS)

```yaml
annotations:
  service.beta.kubernetes.io/aws-load-balancer-type: "external"
  service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
  service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
```

Requires the **AWS Load Balancer Controller** to be running in the cluster. Without it, the Service will remain in `Pending` state.

### GCP (GKE)

Replace the annotations block in `envoyproxy.yaml` with:

```yaml
annotations:
  cloud.google.com/load-balancer-type: "External"
```

GKE's built-in cloud controller handles this natively — no additional controller needed.

## Deploying a Gateway

Once the GatewayClass is `ACCEPTED=True`, create a `Gateway` in your application namespace:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-app-gateway
  namespace: my-app
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  gatewayClassName: envoy
  listeners:
    - name: http
      port: 80
      protocol: HTTP
    - name: https
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - name: my-app-tls
```

## Routing Traffic with HTTPRoute

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app
  namespace: my-app
spec:
  parentRefs:
    - name: my-app-gateway
  hostnames:
    - my-app.prod.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: my-app-svc
          port: 8080
```

external-dns automatically creates a Route53/Cloud DNS A record for `my-app.prod.example.com` when it sees this HTTPRoute (via the `gateway-httproute` source in its base values).

cert-manager automatically issues a TLS certificate from the `cert-manager.io/cluster-issuer` annotation on the parent Gateway.

## Verification

```bash
# Controller running
kubectl get pods -n envoy-gateway-system

# GatewayClass accepted
kubectl get gatewayclass envoy

# After creating a Gateway, verify LB provisioned
kubectl get gateway -n <namespace>
kubectl get svc -n envoy-gateway-system   # look for the envoy-<namespace>-<gateway> service

# Check cert issued
kubectl get certificate -n <namespace>

# Check DNS record created (~30s after HTTPRoute applied)
aws route53 list-resource-record-sets --hosted-zone-id <ZONE_ID>  # AWS
gcloud dns record-sets list --zone=<ZONE_NAME>                    # GCP
```
