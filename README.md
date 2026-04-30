# RHCL Gateway Pipeline GitOps

GitOps configuration for an Istio ingress gateway managed by [Kuadrant](https://kuadrant.io), deployed via OpenShift GitOps (ArgoCD).

---

## Prerequisites

### Operators

The following operators must be installed on the cluster before deploying:

- **OpenShift GitOps** — ArgoCD-based GitOps operator
- **Red Hat Connectivity Link (RHCL)** — installs Kuadrant and its components (`kuadrant-system` namespace must be healthy)

### Credentials

Two Secrets must be created manually before ArgoCD syncs — they contain AWS IAM credentials and must **not** be committed to git.

**Kuadrant DNS integration** (`ingress-gateway` namespace):


**cert-manager DNS-01 challenge** (`cert-manager` namespace):


```bash
oc create secret generic aws-credentials \
  --from-literal=AWS_ACCESS_KEY_ID=$KUADRANT_AWS_ACCESS_KEY_ID \
  --from-literal=AWS_SECRET_ACCESS_KEY=$KUADRANT_AWS_SECRET_ACCESS_KEY \
  -n secret-store
```

A `letsencrypt` ClusterIssuer must also exist in the cluster before deploying.

---

## Deployment

Apply the ArgoCD Applications from the `argocd/` directory:

```bash
oc apply -f argocd/application-gw.yaml      # Gateway, TLS, DNS, auth/rate-limit policies
oc apply -f argocd/application-route.yaml   # Bookinfo test app + HTTPRoute
```

ArgoCD will sync both applications automatically. The gateway app uses sync waves to enforce order:

| Wave | Resources |
|------|-----------|
| 0 | Namespace, RBAC |
| 1 | Gateway, Telemetry, PodMonitor, TLSPolicy |
| 3 | DNSPolicy |
| 4 | AuthPolicy (deny-all default) |
| 5 | RateLimitPolicy |

---

## Verification

### 1. Gateway reachability

Get the gateway's external IP and confirm it responds:

```bash
export GATEWAY_IP=$(oc -n ingress-gateway get gateway prod-gateway \
  -o jsonpath='{.status.addresses[*].value}')

curl -v -H "Host: demo.leonlevy.lol" http://$GATEWAY_IP
```

A `404 Not Found` from `istio-envoy` confirms the gateway is reachable (no routes attached yet is expected at this stage):

```
< HTTP/1.1 404 Not Found
< server: istio-envoy
```

### 2. DNS resolution

Confirm Route 53 records have been created:

```bash
dig demo.leonlevy.lol
```

If DNS is not resolving, check the DNSPolicy and DNSRecord status:

```bash
oc get dnspolicy prod-gateway-dnspolicy -n ingress-gateway \
  -o jsonpath='{.status.conditions}' | jq .

oc get dnsrecord -n ingress-gateway
```

### 3. Test the bookinfo HTTPRoute with auth and rate limiting

Get the hostname from the HTTPRoute:

```bash
export GATEWAY_URL=$(oc -n bookinfo get httproutes.gateway.networking.k8s.io bookinfo \
  -o jsonpath='{.spec.hostnames[0]}')

echo "Testing: https://${GATEWAY_URL}/api/v1/products/0/ratings"
```

**Valid user (alice) — authenticated, rate-limited to 2 req/10s:**
```bash
curl -k -so - -w "\n%{http_code}\n" \
  "https://${GATEWAY_URL}/api/v1/products/0/ratings" \
  -H 'Authorization: APIKEY IAMALICE'
```

**Valid user (bob) — authenticated, higher rate limit of 20 req/10s:**
```bash
curl -k -so - -w "\n%{http_code}\n" \
  "https://${GATEWAY_URL}/api/v1/products/0/ratings" \
  -H 'Authorization: APIKEY IAMBOB'
```

**Invalid key — expect `401 Unauthorized`:**
```bash
curl -k -so - -w "\n%{http_code}\n" \
  "https://${GATEWAY_URL}/api/v1/products/0/ratings" \
  -H 'Authorization: APIKEY NOONE'
```


If things get stuck
kubectl patch namespace ingress-gateway -p '{"metadata":{"finalizers":null}}' --type merge

oc patch app rhcl-gateway -n openshift-gitops -p '{"metadata":{"finalizers":null}}' --type merge